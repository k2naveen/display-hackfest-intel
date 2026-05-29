# CMRR (Content Match Refresh Rate) — Design & Policy Document

**Status:** RFC — Seeking community input on userspace interface  
**Panel Requirement:** Adaptive Sync capable panel

---

## 1. Summary

### The Problem

Media content uses NTSC-derived fractional frame rates (23.976, 29.97, 59.94 Hz)
expressed as N × (1000/1001). Panel timing is defined by integer parameters
(pixel_clock, htotal, vtotal), making it impossible to achieve these rates
exactly with a single fixed vtotal. The resulting mismatch causes cumulative
Audio/Video drift, forcing compositors to drop or repeat frames.

### How CMRR Solves This

CMRR (Content Match Refresh Rate) is an Intel hardware feature that uses the
Adaptive Sync Timing Generator in fixed-mode to **dither vtotal** across frames.
By alternating between vtotal and (vtotal + 1) in a precisely controlled ratio
(defined by M/N parameters), the hardware produces an *average* refresh rate
that exactly equals the fractional target — without any single-frame timing
violation visible to the panel.

**Key benefit:** Eliminates audio/video sync drift on media playback
(e.g., 23.976 Hz, 29.97 Hz, 59.94 Hz content).

---

### The Refresh Rate Mismatch Problem

```
Panel refresh:   60.000000 Hz
Content rate:    59.940060 Hz (60000/1001)
Mismatch:        0.059940 Hz

Per second:      0.059940 frames of drift
Per minute:      3.596 frames of drift  
Per hour:        215.8 frames of drift

Result: Compositor must drop ~216 frames per hour to stay in sync
        = ~1 frame drop every 16.7 seconds
        = visible stutter on slow camera pans
```

### Why Existing Solutions Fall Short

| Approach | Limitation |
|----------|-----------|
| Standard fixed RR | Integer vtotal → cannot hit fractional Hz exactly |
| Software frame pacing | Drops/repeats frames → visual stutter |

---

## 3. CMRR Solution Architecture

### 3.1 Hardware Mechanism

The CMRR hardware dithers vtotal on a per-frame basis:
- Some frames use `vtotal`
- Some frames use `vtotal + 1`

The ratio of short-to-long frames is controlled by the M/N parameters such that
the *average* refresh rate over N frames equals the exact fractional target.

```
Average refresh = pixel_clock × M / N

Where:
  N = desired_refresh_rate × htotal × multiplier_n
  M = pixel_clock × 1000 × multiplier_m / N  (remainder from division)
```

For 59.94 Hz: multiplier_m=1000, multiplier_n=1001
→ effective rate = 60 × 1000/1001 = 59.94005994… Hz ✓

---

## 4. Expectation from Userspace

The kernel driver computes and programs the CMRR M/N registers, but it needs
userspace (compositor or media framework) to communicate the desired target
refresh rate including its fractional component. Two approaches are proposed:

### 4.1 Approach 1: Enumerated Scaling Levels

**Concept:** A single CRTC property with predefined scaling levels.

```c
/* DRM CRTC Property: "content_match_refresh_rate" */
enum drm_cmrr_level {
    DRM_CMRR_DEFAULT,  /* No fractional scaling — standard fixed refresh */
    DRM_CMRR_LOW,      /* Downscale: rate × 1000/1001 (e.g., 60 → 59.94) */
    DRM_CMRR_HIGH,     /* Upscale: rate × 1001/1000 (e.g., 59.94 → 60.06) */
};
```

**How it works:**

| Level | Scale Factor | Example |
|-------|-------------|---------|
| `DRM_CMRR_DEFAULT` | 1000/1000 (= 1.0) | 60 Hz → 60.00 Hz (no CMRR) |
| `DRM_CMRR_LOW` | 1000/1001 | 60 Hz → 59.94 Hz |
| `DRM_CMRR_HIGH` | 1001/1000 | 60 Hz → 60.06 Hz |

---

### 4.2 Approach 2: Numerator and Denominator

**Concept:** Two CRTC properties that together express the exact target
refresh rate as a rational number.

```c
/* DRM CRTC Properties */
"CMRR_REFRESH_RATE_NUMERATOR"    /* u32: e.g., 60000 */
"CMRR_REFRESH_RATE_DENOMINATOR"  /* u32: e.g., 1001  */
```

**How it works:**

The target refresh rate = NUMERATOR / DENOMINATOR (in Hz).
The kernel computes appropriate M/N values from this ratio.

| Target Rate | Numerator | Denominator | Result |
|-------------|-----------|-------------|--------|
| 59.94 Hz | 60000 | 1001 | 59.94005… Hz |
| 23.976 Hz | 24000 | 1001 | 23.97602… Hz |
| 60.00 Hz | 60 | 1 | No CMRR needed |
| 29.97 Hz | 30000 | 1001 | 29.97002… Hz |
| 47.952 Hz | 48000 | 1001 | 47.95204… Hz |
| 119.88 Hz | 120000 | 1001 | 119.88011… Hz |
