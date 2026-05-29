# KMS Color Pipeline - 1D LUT

[← 2026](2026)

## Discover capabilities of 1D LUT HW block

The driver exposes a blob property (HW_CAPS) which allows user-space to
parse and extract the hardware capabilities of 1D LUT. These include
number of LUT segments, number of LUT samples per segment, start and end
point of respective segments and the precision of the LUT sample along
with the normalization factor. This is how the capability structure looks like:

```c
struct drm_color_lut_range {
       __u32 flags;
       __u16 count;
       __s32 start, end;
       __u32 norm_factor;

       struct {
               __u16 intp;
               __u16 fracp;
       } precision;
};
```

If a hardware has multiple segments in 1D LUT, each segment will be represented
by one instance of the above structure and the whole 1D LUT block will be represented
by an array of `drm_color_lut_range`.

Here,

- **flags**: Indicates LUT characteristics like linearly increasing, negative reflect or any other property of the LUT.
- **count**: Number of samples in the respective segment.
- **start, end**: Indicates the starting point and ending point of the segment respectively. This represents a point in 0 to 1.0 color space curve and the value depends on the normalization factor chosen.
- **norm_factor**: This factor helps define a scale to represent LUT sample with the smallest step size in case of uniform or non-uniform LUT sample distribution.
- **precision**: Indicates the fixed point precision of HW LUT including integer and fractional component.

## Usage examples

### 1. Conventional 1D LUT with just one segment

```
|---|---|------------------------------------|
0   1   2                                   1024
```

- Hardware Description: A color block with a LUT linearly interpolating and covering range from 0 to 1.0
  - Number of segments: 1
  - Number of samples in LUT: 1024
  - Precision of LUT samples in HW: 0.10
  - Normalization Factor: Max value to represent 1.0 in terms of smallest step size which is 1024.

In this case, it will be represented by the following structure:

```c
struct drm_color_lut_range lut_1024[] = {
        .start = 0, .end = (1 << 10),
        .normalization_factor = 1024,
        .count = 1024,
        .precision = {
                .int_comp = 0,
                .fractional_comp = 10,
        }
}
```

### 2. Piece Wise Linear 1D LUT

```
        |---|---|------------------------------------|
        0   1   2                                   32
        |    \
        |       \
        |          \
        |             \
        |                \
        0                   \
        |---|---|--...-------|
        0   1   2            8
```

- Hardware Description: A color block with a LUT linearly interpolating and covering range from 0 to 1.0
  - Number of segments: 2
  - Number of samples:
    - segment 1: 9 (covers range from 0 to 1/32)
    - segment 2: 30 (covers range from 2/32 to 1.0)
  - Precision of LUT samples in HW: 0.24
  - Normalization Factor: Max value to represent 1.0 in terms of smallest step size which is 8*32.

```c
struct drm_color_lut_range lut_pwl[] = {
        /* segment 1 */
        {
                .count = 9,
                .start = 0, .end = 8,
                .norm_factor = 8*32,
                .precision = {
                        .intp = 0,
                        .fracp = 24,
                },
        },
        /* segment 2 */
        {
                .count = 30,
                .start = 8*2, .end = 8*32,
                .norm_factor = 8*32,
                .precision = {
                        .intp = 0,
                        .fracp = 24,
                },
        },
}
```

**Note:** In case HW supports overlapping LUTs, the expectation from uAPI is that the respective HW vendor driver exposes it as a linearly increasing LUT and internally handles the programming of the overlapping sections.

## Programming 1D LUT HW block

In order to compute the LUT samples, userspace will parse the `drm_color_lut_range` structure to
get the LUT distribution of the underlying HW block.

It needs to compute the normalized value of the LUT sample using the normalization factor provided
by the driver. The normalized value can then be scaled to the LUT precision of the HW. The computed
LUT samples will be packed in a blob and passed to the driver to be programmed in HW.

The pseudo code for calculating the LUT samples for a linear LUT:

```c
for (i = 0; i < sample_count; i++) {
                          start                         end - start
        normalized_value = ---------------------- + ----------------------------------------- * i
                            normalization_factor    (sample_count - 1) * normalization_factor

        lut[i] = normalized_value * lut_precision /* (1 << precision.fracp) */
}
```

**Note:** The same logic can be extended for any color space transfer function implementation.
