# Technical notes

Target: Astral Chain `01007300020FA000`, update v1.0.1, `main` NSO build ID
`4B159F0F7A360669738390EE8657B4F0`. All addresses are offsets in `main` (module base 0). The
`.pchtxt` files use `@flag offset_shift 0x100`, so the address on each line is the `main`
offset.

## Frame timer (original game)

Function `0x91DBB0`. Timer object `0xDCBFD0` (GOT `0xDB89A0`). The main loop calls it once per
iteration at `0x3E39CC` with `s0 = 1.0` and `w1 = 1 & ~(flags >> 30)`; `flags = [0xDCBA30]`.

| Field | Meaning |
|---|---|
| `+0xC8` | previous tick (`0xA0D630` reads the system tick) |
| `+0xD0` | **step**, in 30 FPS frames. About 120 sites read it. |
| `+0xD4` | carry (original) / fractional whole-tick remainder (this mod) |
| `+0xD8` | dt accumulator (original) / whole 30 FPS ticks this frame (this mod) |
| `+0xDC` | `30 / 1000` (set at `0x91D90C`) |
| `+0xE0` | measured frame time in ms |

Original behaviour:
- `w1 == 0` (flag bit 30 set): step = 1.0 (fixed).
- Otherwise: if the override float `0xDADC5C` (read by `0x8A9570`, default `-1`) is positive,
  step = override. Else step = `dt_ms × 0.03 − carry`, clamped to `[1.0, 2.0]`. Below 1.0
  the step stays 1.0 and the difference becomes carry.

Bit 30 is set when cutscene playback starts (`0x8B7428 → 0x42FE60`). It is cleared with a
reference count (`0x8B842C → 0x42FEA0`). Cutscene progress (`0x8C2490..0x8C24A4`) is
`fraction += authoredFps/30 × step × speed`. So a fixed step of 1.0 at 60 FPS plays cutscenes
2 times too fast. Playback audio sync writes the override (`0x8B9F0C`, `0x8C2F5C`).

The per-frame delta in seconds (`step / 30`) is also stored at `main+0xCCA6764`
(`0x6E5180 → 0x905720`).

## Frame cap (present interval)

- Window start-up: `0x9E8D74..0x9E8D84` maps mode `0→1`, `1→2`, other`→0`, then
  `nvnWindowBuilderSetPresentInterval`.
- Run time: graphics object `0xCCB7AF0` (`+0x420` requested mode, `+0x424` current mode).
  `0x9DB690..0x9DB6C0` calls `nvnWindowSetPresentInterval` when they differ. The main loop
  requests mode 1 (30 FPS) every frame through `0x9DFE40` (call at `0x3E39D4`).
- The community cutscene cheat writes `[[[main+0xCCB7B08]+0x70]+0xF14]`. That is the
  `android::Surface` swap interval: 2 when step = 1.0, else 1.

## This mod

| Address | Original | New | Purpose |
|---|---|---|---|
| `0x9E8D84` | `csinc w1,w8,wzr,ne` | `mov w1,#0` / `#1` | start-up frame cap |
| `0x9DB6BC` | `csinc w1,w9,wzr,ne` | `mov w1,#0` / `#1` | run-time frame cap |
| `0x91DBDC` | `cbz x8,0x91DCB4` | `cbz x8,0xA0E828` | first sample: clear remainder, step 1 |
| `0x91DC10` | `cbz w21,0x91DC48` | `b 0xA0E7D0` | measured path into the code cave |
| `0x48BE14` | `fcvtzs w8,s0` | `ldr w8,[x8,#0xD8]` | counter to 30: add whole ticks |
| `0x57E7DC` | `ldr s0,[x24,#0xD0]` | `ldr w8,[x24,#0xD8]` | parity counter: whole ticks |
| `0x57E7E0` | `fcvtzu w8,s0` | `cbz w8,0x57E810` | skip the update when no whole tick |
| `0xA0E7D0..0xA0E82C` | zero padding | 24 instructions | new step calculation |

Code cave (`.text` padding `0xA0E7D0..0xA0EFFF`, 0x830 bytes, all zero). Eden pads the
program image to the segment layout before it applies patches, and the page is executable.

```
step = dt_ms × [+0xDC]
if w21 != 0 and override > 0: step *= override      ; keep audio-sync correction (relative)
step = clamp(step, 1/16, 2)                          ; no carry
[+0xD0] = step
r = [+0xD4] + step ; ticks = trunc(r) ; [+0xD4] = r − ticks ; [+0xD8] = ticks
[+0xE4] = 0 ; branch to 0x91DCB4 (store tick, epilogue)
```

The cave never sets the step from the fixed path. Bit 30 now only skips the override.

Design notes:
- Floor 1/16 instead of 0: the steps derived from it are used as divisors at `0x6BBDC`,
  `0x225270` and `0x37BD68`. Those have guards, but a full proof was not possible.
- `0x48BE14` and `0x57E7E0` converted the step to an integer. Fractional steps then gave 0,
  so those counters stopped.
- `0x430DD4` compares time-scale channel 3 with 1.0 to choose a clock. It was not changed.
- The staff-roll scheduler (`0x6A3F28`, counter `0xA282930`) counts integer frames. It was not
  changed. A fix exists (hook `0x6A23D0`/`0x6A1180`) if testing shows the credits run fast.

## Method

1. Decrypted the update NCA with hactool, using the title key from the ticket. Built a flat
   memory image from the uncompressed NSO segment table.
2. Disassembled with capstone. Resolved GOT slots from `R_AARCH64_RELATIVE` relocations.
3. Assembled with keystone. Every original instruction was checked against the binary before
   it was replaced.
4. Review: an independent second model (Codex, gpt-6.1-sol, extra-high reasoning) verified the
   facts and proposed the cave design. The final design came from two debate rounds.
