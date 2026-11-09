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

## Stick gestures (r6)

### Gesture parser (original game)

Function `0x322BE0`, called once per frame by the player input update `0x282870` (call at
`0x283030`), with `x19` = the player input object. The stick's squared length is `[+0x940]`, its
angle `[+0x948]` (sector test), and `[+0x94C]` is the direction the push angles are taken from. A push is a stick length above `640000`
(`[0xB5CEC4]`).

| Field | Set to | Meaning |
|---|---|---|
| `+0x1508..+0x1524` | 12 | one timer per direction sector (8 sectors), stamped at `0x322D6C` while the stick points there |
| `+0x1528` | 5 | rotation window for the spin attack (360 spin): 6 or more live sectors (`0x322DE8`) |
| `+0x1530` | 5 / 10 | press window: 5 after a first push, 10 after a second push inside it |
| `+0x1534` | angle | angle of the last push |
| `+0x1538` | 5 | **reversal**: a push inside the press window that turns more than 135° (`2.356`, `0x323010`) |
| `+0x153C` | 5 | same-direction push: a push inside the window that turns less than 45° (`0.785`, `0x322FF4`) |
| `+0x1544`, `+0x155C` | 5 | push windows |
| `+0x1554` | 1 | pushed this frame (also when a held stick turns more than 135°, `0x322EF8`) |
| `+0x1568`, `+0x156C` | 5 | right-stick push windows |

All windows are decremented by 1 per frame. A rotation clears `+0x1538` and `+0x153C`
(`str xzr,[x19,#0x1538]` at `0x322DFC`). The attack selector reads the flags when the attack button
is pressed: `0x2867C0` picks attack `0x41` while `+0x1538 > 0` and attack `0x43` while
`+0x1528 > 0`, so an open rotation window wins. Other readers of `+0x1538`: `0x2861C4`, `0x287010`,
`0x325748`.

Two problems at 60 FPS:
1. **Window length.** The press window lasted 83 ms instead of 167 ms. A controller stick needs
   more time than that to go from back through the centre to forward, so the reversal was never
   set. A key changes direction at once, so a keyboard still worked.
2. **Sampling.** A back-to-forward sweep along the edge of the stick passes about 5 sectors. At
   30 FPS a fast sweep is sampled about every 33 ms and skips some of them. At 60 FPS more sectors
   are stamped, the rotation test reaches 6 more often, and the rotation clears the reversal and
   takes over the attack. Scaling the windows alone left the assault attack (back-to-forward)
   window tighter than at 30 FPS in play.

### Changes

| Address | Original | New | Purpose |
|---|---|---|---|
| `0x322BF8`, `0x322C1C..0x322C6C` (6) | `subs w8,w8,#1` | `bl hs_w8` | sector timers |
| `0x322C10` | `sub w8,w8,#1` | `bl hs_w8` | sector timer `+0x150C` |
| `0x322D74` | `subs w8,w8,#1` | `bl hs_w8` | rotation window |
| `0x322E04`, `0x322E3C` | `subs w8,w8,#1` | `bl hs_w8` | same-direction window |
| `0x322E4C` | `subs w8,w8,#1` | `bl hs_w8` | reversal window |
| `0x322E5C..0x322E8C` (4) | `subs w8,w8,#1` | `bl hs_w8` | push and right-stick windows |
| `0x322F0C`, `0x322F24` | `subs w9,w8,#1` | `bl hs_w8` | right-stick push window `+0x1568` |
| `0x322F30` | `mov w8,w9` | `mov w9,w8` | its store takes the scaled `w8` |
| `0x322FB0` | `subs w8,w8,#1` | `bl hs_w8` | window `+0x46B438` |
| `0x322FC8` | `sub w8,w8,#1` | `bl hs_w8` | press window `+0x1530` |
| `0x322D6C` | `str w9,[x8,#0x1508]` | `b 0xA0EFD4` | sector stamp only with a whole tick |
| `0x282A48`, `0x282A64` | `subs w8,w8,#1` | `bl hs_w8` | stick push windows `+0x46AE80`, `+0x46AE84` (3 frames) |
| `0x282FD0`, `0x282FEC` | `subs w9,w9,#1` | `bl hs_w9` | windows `+0x46B570`, `+0x46B590` |
| `0x2832E4` | `subs w9,w9,#1` | `bl hs_w9` | press-repeat window `+0x46AE5C` (5 frames) |
| `0x2833A0` | `fmov s1,#-1.0` | `b 0xA0EFEC` | countdown `+0x46ADB4`: subtract the step |

Code cave, the last 11 free words:

```
0xA0EFD4  adrp x10,0xDB8000 ; ldr x10,[x10,#0x9A0] ; ldr w10,[x10,#0xD8]   ; whole ticks
          cbz w10,+8 ; str w9,[x8,#0x1508] ; b 0x322D70                   ; stamp only with a tick
0xA0EFEC  adrp x9,0xDB8000 ; ldr x9,[x9,#0x9A0] ; ldr s1,[x9,#0xD0]       ; step
          fneg s1,s1 ; b 0x2833A4                                          ; count down by the step
```

Notes:
- `hs_w8` / `hs_w9` (`0xA0ED80`, `0xA0EDBC`) are the scaled decrement of v1.1: subtract the whole
  ticks, stop at 0 from a positive value, and set the flags the original `subs` gave for "expired",
  "reaches 0 now" and "still running". Both functions save the link register in their prologue,
  so `bl` is safe at every site, as at the v1.1 sites in `0x282870`.
- The two `sub` sites run only after a `> 0` test, so `hs_w8` gives the same result. The flags it
  sets are rewritten before any read.
- `subs w9,w8,#1` has no shared routine. Both sites scale `w8` with `hs_w8`; the shared store path
  at `0x322F30` then copies `w8` to `w9`. On the other path `w8` holds the same "expired" value
  (`<= 0`) that the original left in `w8`, and the next reader (`cmp w8,#1` at `0x322F70`) branches
  the same way.
- Scratch registers: `x10` at `0x322D6C` is rewritten at `0x322D80` before any read. `x9` at
  `0x2833A0` is dead: the path goes on to a call (`bl 0x323140`) without reading it.
- The code cave is now full. A further cave needs new space, for example the relocated regions at
  `0x5763E4` and `0x93073C` that v1.1 already uses.

## Method

1. Decrypted the update NCA with hactool, using the title key from the ticket. Built a flat
   memory image from the uncompressed NSO segment table.
2. Disassembled with capstone. Resolved GOT slots from `R_AARCH64_RELATIVE` relocations.
3. Assembled with keystone. Every original instruction was checked against the binary before
   it was replaced.
4. Review: an independent second model (Codex, gpt-6.1-sol, extra-high reasoning) verified the
   facts and proposed the cave design. The final design came from two debate rounds.
