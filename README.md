I’ve incorporated the existing 5-D versus 8-D interpolation numerical-equivalence results, the small-scene measurements, and the latest full-scene benchmark. I’ve kept the numerical-equivalence claim scoped to the tested interpolation points.

# ACOLITE PACE OCI Optimisation

A performance-optimised version of [ACOLITE](https://github.com/acolite/acolite), focused on reducing processing time for NASA PACE Ocean Color Instrument (OCI) data.

The primary objective is to improve computational efficiency while retaining the original ACOLITE scientific methodology, aerosol retrieval settings, scientific LUTs, and processing configuration.

## Key improvements

- Replaced the expensive general 8-dimensional `RegularGridInterpolator` operation used in AOT LUT interpolation with a specialised 5-dimensional multilinear interpolation.
- Vectorised wavelength and pixel calculations to reduce Python-level iteration.
- Reused interpolation coordinates, neighbouring grid indices, and interpolation weights.
- Introduced caching of modelled radiance arrays to avoid repeated LUT calculations across spectral bands.
- Preserved the existing 6 × 6 pixel tile configuration.

## 1. AOT LUT interpolation optimisation

### Original implementation

The original AOT estimation used an 8-dimensional `RegularGridInterpolator` (RGI), with the following dimensions:

1. Pressure
2. Parameter
3. Wavelength
4. Relative azimuth angle (RAA)
5. View zenith angle (VZA)
6. Solar zenith angle (SZA)
7. Wind speed
8. Aerosol optical thickness (AOT)

During the AOT calculation, parameter, wavelength, and AOT were already located at exact LUT grid nodes. Interpolation was therefore required only along the other five dimensions.

### Optimised implementation

A specialised 5-D multilinear interpolation was implemented:

- `np.searchsorted` identifies neighbouring grid indices.
- Interpolation weights are calculated for the five varying dimensions.
- The 32 interpolation corners ((2^5)) are evaluated.
- Interpolation along the three exact-grid dimensions is avoided.
- Existing RGI LUT values are used directly.

This removes unnecessary interpolation operations while retaining the original multilinear interpolation formulation.

## 2. Vectorisation

The initial specialised interpolation still contained loops over AOT and wavelength.

These calculations were subsequently vectorised to process wavelength and pixel arrays together, reducing Python-level iteration and improving NumPy utilisation.

## 3. Reuse of interpolation coordinates and weights

The five varying interpolation coordinates are:

- Pressure
- RAA
- VZA
- SZA
- Wind speed

Their neighbouring LUT indices and interpolation weights are calculated once and reused across AOT and wavelength calculations.

This avoids repeatedly searching the LUT grids for identical interpolation coordinates.

## 4. Modelled-radiance caching

A `rhot_aot_cache` was introduced to avoid recalculating the complete modelled-radiance array for every spectral band.

The cache stores arrays indexed by LUT identifier. Subsequent bands using the same LUT reuse the stored array.

This is applicable to the tested PACE OCI processing case, where the relevant geometry is common across spectral bands.

A cache-handling issue involving entries containing `None` was also addressed. Such entries are treated as cache misses, allowing the modelled radiance to be recalculated rather than causing a `NoneType` error.

The subsequent test successfully generated valid arrays for both MOD1 and MOD2.

## 5. Numerical equivalence

The specialised 5-D interpolation was compared against the original 8-D `RegularGridInterpolator` implementation.

| LUT  | Maximum absolute difference | Mean absolute difference | RMSE                  |
| ---- | --------------------------- | ------------------------ | --------------------- |
| MOD1 | (1.49 \times 10^{-7})       | (4.20 \times 10^{-9})    | (9.29 \times 10^{-9}) |
| MOD2 | (2.38 \times 10^{-7})       | (5.54 \times 10^{-9})    | (1.17 \times 10^{-8}) |

The differences are extremely small and demonstrate close numerical agreement between the specialised 5-D implementation and the original 8-D RGI implementation at the tested interpolation points.

**Boundary handling qualification:** The original RGI uses `fill_value=np.nan` for coordinates outside LUT boundaries. The saved source snapshot clips interpolation indices but does not explicitly mask out-of-range coordinates. This behaviour should be checked before claiming equivalence for all possible interpolation coordinates.

## 6. Performance benchmarks

### 6.1 Small-scene benchmark

Test scene:

| Parameter        | Value        |
| ---------------- | ------------ |
| Image dimensions | 372 × 279    |
| Number of tiles  | 2,914        |
| Tile size        | 6 × 6 pixels |

Observed timings:

| Operation                               | Processing time     |
| --------------------------------------- | ------------------- |
| LUT loading                             | 42.0 s              |
| First surface-reflectance band (346 nm) | 35.4 s              |
| Next band (348 nm)                      | 0.1 s               |
| Subsequent bands                        | Approximately 0.1 s |

The L2W NetCDF output was successfully generated.

### 6.2 Tile-count scaling observation

A larger scene contained 60,420 tiles. Its recorded first-band processing time was 771.8 seconds.

| Metric                     | Small scene | Larger scene |
| -------------------------- | ----------- | ------------ |
| Number of tiles            | 2,914       | 60,420       |
| First-band processing time | 35.4 s      | 771.8 s      |

The tile-count ratio is approximately 20.72, while the processing-time ratio is approximately 21.8.

This indicates broadly similar scaling with tile count for this operation. It does not establish that increasing tile size would improve overall performance.

### 6.3 Full-scene PACE OCI benchmark

A full-scene comparison was performed using the same PACE OCI L1B input: `PACE_OCI.20250719T105436.L1B.V3.nc`

| Parameter        | Value        |
| ---------------- | ------------ |
| Image dimensions | 1287 × 1173  |
| Number of tiles  | 42,140       |
| Tile size        | 6 × 6 pixels |

Processing-time comparison:
            
| Processing stage                    | Original ACOLITE | Optimised ACOLITE |
| ----------------------------------- | ---------------- | ----------------- |
| AOT estimation                      | 7 h 27 min 32 s  | 7 min 20 s        |
| End-to-end processing to L2W NetCDF | 7 h 53 min 38 s  | 26 min 05 s       |

Performance improvement:

| Metric                        | Improvement         |
| ----------------------------- | ------------------- |
| AOT estimation speedup        | Approximately 61.0× |
| AOT processing-time reduction | Approximately 98.4% |
| End-to-end speedup            | Approximately 18.2× |

Both runs generated L2R and L2W NetCDF products. The latest optimised run successfully generated Rrs datasets and corresponding quicklook images.

These measurements demonstrate a substantial reduction in PACE OCI processing time, particularly during AOT estimation.

The benchmark timings describe processing performance. Numerical equivalence of the complete full-scene retrieval products is a separate validation requirement.

## 7. Components retained unchanged

The optimisation focuses on computational implementation rather than changes to the scientific retrieval methodology.

| Component                               | Status                        |
| --------------------------------------- | ----------------------------- |
| Scientific LUTs                         | Unchanged                     |
| Aerosol retrieval methodology           | Unchanged                     |
| Retrieval thresholds and settings       | Unchanged                     |
| Tile size                               | Retained at 6 × 6 pixels      |
| AOT interpolation implementation        | Specialised 5-D interpolation |
| Repeated modelled-radiance calculations | Cached and reused             |
| Surface-reflectance computation         | Further profiling required    |

## 8. Remaining work

The following items remain relevant to finalising the implementation:

- Validate out-of-range interpolation behaviour against the original RGI implementation.
- Confirm the final working source contains the vectorised implementation and cache safeguard.
- Complete pixel-wise comparison of full-scene AOT and Rrs products.
- Profile the first-band surface-reflectance calculation to identify further computational bottlenecks.
- Reduce excessive diagnostic logging. The latest test log contained approximately 25,425 lines, largely from repeated debugging messages.

The existing 6 × 6 tile size should be retained during these investigations to avoid introducing additional variables into performance and numerical comparisons.

## 9. Acknowledgement and licensing

This work is based on the original [ACOLITE repository](https://github.com/acolite/acolite).

The original ACOLITE project and its contributors are acknowledged. This repository focuses on computational optimisation for PACE OCI processing and is not intended to replace or redefine the original scientific methodology.

Refer to the repository's `LICENSE` file for licensing information. Original copyright notices and applicable licence terms should be preserved.

## Summary

The principal achievement is the acceleration of AOT LUT interpolation through a specialised 5-D implementation, vectorised calculations, reuse of interpolation coordinates and weights, and caching of modelled radiance.

The specialised interpolation demonstrates very close numerical agreement with the original 8-D RGI implementation at the tested interpolation points. Full-scene PACE OCI processing time was reduced from approximately 7 hours 54 minutes to 26 minutes, corresponding to an end-to-end speedup of approximately 18.2×.

The optimisation retains the original scientific LUTs, aerosol retrieval methodology, retrieval settings, and tile configuration. Further work will focus on boundary-condition validation, full-scene product comparison, and profiling the remaining processing bottlenecks.

Clarify the benchmark timing basisSeparate demonstrated results from pending work
