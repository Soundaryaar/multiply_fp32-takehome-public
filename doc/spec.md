# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)

Stage 6 is **one clock cycle** containing **three sub-steps that execute in
strict sequence** on the same cycle. Sub-steps A and B are mutually exclusive
(at most one of them fires), but **sub-step C (rounding) is executed on every
operation, unconditionally**, using the values *produced by* A or B.

> **Implementation note.** Because each sub-step consumes the result of the
> previous one within the same cycle, write Stage 6 with *blocking* temporaries
> (`logic [23:0] m; logic signed [9:0] e; logic g, r, s;`) initialised from the
> registered `z_m / z_e / guard_bit / round_bit / sticky`, apply A/B/C to the
> temporaries, and assign the temporaries back to the registers at the end.
> **Do not** structure this stage as
> `if (underflow) ... else if (normalize) ... else round ...` — that silently
> skips rounding whenever a shift occurs and produces results that are off by
> one LSB.

**Sub-step A — Underflow alignment (only when `z_e < -126`)**
- `sh = -126 - z_e` (number of right-shift positions, may exceed 24).
- `sticky |= OR(bits of m shifted out) | g | r` — everything that falls off the
  right end, *including the old guard and round bits*, is folded into sticky.
- `m = m >> sh`, `g = 0`, `r = 0`, `e = -126`.
- If `sh >= 24` the mantissa becomes 0 (sticky still set).

**Sub-step B — Left normalization (only when A did not fire and `m[23] == 0`)**
- `m = {m[22:0], g}` (shift left by one, pulling the guard bit into the LSB).
- `g = r`, `r = 0`, `e = e - 1`.
- Only a single left shift is ever required, because the product of two
  24-bit mantissas with hidden ones is always in `[1.0, 4.0)`.

**Sub-step C — RNE rounding (ALWAYS executed, after A or B)**
- `if (g && (r || s || m[0]))`  →  `m = m + 1`.
- If the increment carries out (`m` becomes `24'h000000` after being
  `24'hFFFFFF`), set `m = 24'h800000` and `e = e + 1`.
- Write `m → z_m`, `e → z_e` (guard/round/sticky are no longer needed).

### Stage 7 — Pack
- `biased = z_e + 127`.
- **Overflow:** if `z_e > 127` (i.e. `biased >= 255`) → `z = {z_s, 8'hFF, 23'h0}`
  (±INF). Note the comparison is *strictly greater than* 127; `z_e == 127`
  (biased 254) is the largest finite normal and must be packed normally.
- **Underflow / denormal boundary:** if `z_e == -126 && z_m[23] == 0` the result
  has no hidden one and is below the smallest normal number. In that case the
  exponent field **must be 0**: `z = {z_s, 8'h00, z_m[22:0]}`.
  - This is the *only* path that may produce an exponent field of 0.
  - A result that is exactly ±0 (mantissa rounded away completely) is
    therefore `{z_s, 31'h0}`.
  - Never emit exponent field `8'h01` together with `z_m[23] == 0` — that is
    not a valid encoding and will not match the reference.
- **Normal:** otherwise `z = {z_s, biased[7:0], z_m[22:0]}`.
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)
- Outputs **are** checked for overflow (→ ±INF) and underflow (→ exponent
  field 0). Products of two normal inputs can easily fall outside the normal
  range (e.g. `1e-20 * 1e-20`, `1e30 * 1e30`), so Stage 6-A and the Stage 7
  overflow / underflow rules **are exercised by the hidden tests**.
- The result must be a **bit-exact** match to IEEE-754 binary32 multiplication
  with round-to-nearest-even (e.g. `numpy.float32(a) * numpy.float32(b)` or C
  `float` multiplication with default rounding).

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands.

### Self-check checklist (strongly recommended before finishing)
Simple hand-picked values such as `1.0 * 2.0` or `0.5 * 0.5` have all-zero
rounding bits and never exercise Stage 6; passing them proves very little.
Before declaring the design complete, compare against a software reference
(e.g. Python `struct` + `numpy.float32`) on **random bit patterns** and make
sure the following categories are covered:

| Category | How to generate | What it checks |
|----------|-----------------|----------------|
| Random normals | random `exp ∈ [1..254]`, random mantissas | general RNE correctness |
| Left-normalize **and** round | operands whose mantissa product is `< 2.0` with non-zero low bits | rounding after sub-step B |
| Round with carry-out | mantissas near all-ones | `m` wraps to `0x800000`, `e+1` |
| Underflow → 0 | both operands with very small exponents | sub-step A + exp field 0 |
| Underflow boundary | products whose exponent lands exactly at `-126` | `e == -126 && !m[23]` rule |
| Overflow → INF | both operands with very large exponents | `e > 127` test |
| Largest finite | `FLT_MAX * 1.0` | `e == 127` must *not* be INF |

A design that passes `1.0 * 2.0` but fails any row above is **not** complete.

---