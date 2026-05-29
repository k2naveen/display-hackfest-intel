# Pipe Joiner as a Power Optimization Mechanism

[← 2026](2026)

## 1. Background

Display hardware may include features like **Pipe/CRTC Joiner**, where two display CRTCs are combined to drive a single output.

For example In Intel display hardware, pipe/CRTC joiner is primarily used when:

- Display resolution exceeds single-CRTC limits (e.g., 5K+ panels)
- Bandwidth constraints require splitting the workload
- High pixel rate configurations cannot be supported by a single CRTC

## 2. Problem Statement

In many real-world scenarios:

- Displays run at high resolution but moderate refresh rates (e.g., 4K@60Hz)
- The system is in battery-powered or low-power modes
- Workloads are not bandwidth intensive (desktop, media playback)

In these cases, a single CRTC configuration often leads to:

- Higher required Display Clock
- Increased voltage/frequency operating points
- Suboptimal power consumption

## 3. Solution via Joiner

Pipe joiner can be **utilized for power optimization**.

By distributing the display workload across two CRTCs:

- Per-CRTC throughput requirement is reduced
- Required Display Clocks may be lowered
- The system can operate at a more power-efficient point

| Configuration | CRTC Used | Display Clocks | Power Characteristics                     |
|---------------|----------|----------------|-------------------------------------------|
| Single Pipe   | 1        | Higher         | Lower CRTC overhead, higher clock         |
| Pipe Joiner   | 2        | Lower          | Higher CRTC usage, lower clock            |

The goal is to optimize for **net power savings**, not just minimize hardware usage.

## 4. Open Questions and Design Directions

- Should userspace be made aware of **multi-CRTC / joiner capabilities** as part of display configuration?

- Is there value in exposing **explicit pipeline composition controls**
  (e.g., joiner-like concepts), or should such mechanisms remain fully **driver-managed**?

---

### Combinatorial Nature of Joiners

- Theoretically, for **n CRTCs**, a join can involve any subset of size ≥ 2:
  - 2-CRTC, 3-CRTC, … up to n-CRTC joiners

- Number of combinations:
  - For each size *k*: `nCk`
  - Total combinations: `∑ nCk` (from `k = 2` to `n`) = `2^n - n - 1`

- However, **real hardware may support only a constrained subset** due to:
  - topology (e.g., contiguous CRTCs only)
  - routing limitations
  - internal resource constraints

---

### Capability Representation (Discussion Point)

If drivers need to expose joiner capabilities to userspace, one possible model is:

```c
struct drm_join_group {
    __u32 crtc_mask;   // bitmask of CRTCs participating in the join
};

struct drm_joiner_caps {
    __u32 num_crtcs;
    __u32 num_groups;
    struct drm_join_group groups[];
};
```
Drivers may expose:
```c

caps.num_crtcs = 4;
caps.num_groups = 5;

groups = {
    { .crtc_mask = 0b0011 }, // A+B
    { .crtc_mask = 0b0110 }, // B+C
    { .crtc_mask = 0b1100 }, // C+D
    { .crtc_mask = 0b0111 }, // A+B+C
    { .crtc_mask = 0b1110 }, // B+C+D
};
```
