# Pipe Joiner as a Power Optimization Mechanism in Intel Display Pipeline

[← 2026](2026)

## 1. Background

Modern Intel display hardware includes the concept of a **Pipe Joiner**, where two display pipes are combined to drive a single output.

Traditionally, pipe joiner is treated as a capability-driven feature, primarily used when:

- Display resolution exceeds single-pipe limits (e.g., 5K+ panels)
- Bandwidth constraints require splitting the workload
- High pixel rate configurations cannot be supported by a single pipe

## 2. Problem Statement

In many real-world scenarios:

- Displays run at high resolution but moderate refresh rates (e.g., 4K@60Hz)
- The system is in battery-powered or low-power modes
- Workloads are not bandwidth intensive (desktop, media playback)

In these cases, a single pipe configuration often leads to:

- Higher required CDCLK (Core Display Clock)
- Increased voltage/frequency operating points
- Suboptimal power consumption

## 3. Key Insight

Pipe joiner can be **repurposed for power optimization**, not just a capability fallback.

By distributing the display workload across two pipes:

- Per-pipe throughput requirement is reduced
- Required CDCLK may be lowered
- The system can operate at a more power-efficient point

This introduces a new tradeoff dimension:

| Configuration | Pipes Used | CDCLK  | Power Characteristics                       |
| ------------- | ---------- | ------ | ------------------------------------------- |
| Single Pipe   | 1          | Higher | Lower pipe overhead, higher clock           |
| Pipe Joiner   | 2          | Lower  | Higher pipe usage, lower clock              |

The goal is to optimize for **net power savings**, not just minimize hardware usage.

## 4. Proposed Approach

### Option 1: Joiner Property

- Expose control: `pipe_joiner_mode = auto | force_enable | force_disable`
- I915 driver has exposed debugfs capability for experimentation.
- Enables direct A/B comparison (joiner vs non-joiner)
- Expose a connector property where user can choose pipe joiner for it.

### Option 2: Connector Power Policy

- Expose intent: `power_profile = performance | balanced | low_power`
- Driver decides internally when to use joiner
- Abstract, policy-based approach (no hardware exposure)
- Extensible to other optimizations:
  - BPC adjustment
  - Link training
  - DSC / PSR / DRRS?
