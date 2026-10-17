# Astral Chain — Dynamic FPS

> **Experimental.** Tested only on **Astral Chain v1.0.1** (build ID `4B159F0F7A360669`) with
> the [Eden](https://eden-emu.dev) emulator on Linux, with the **Dynamic-Unlocked** variant and the
> 1062p resolution patch. Gameplay, menus and cutscenes run at the correct speed in that test.
> v1.1 was tested on Eden on Linux with Dynamic-Unlocked (without the 1062p patch): move timings,
> camera and cutscene audio sync were correct. The Dynamic-60fps variant of v1.1 was not tested.
> v1.1.1 removes the v1.1 cutscene change (see below).
> It was not tested on a real Switch, on other emulators, or through a full play-through.
> Please report problems in the [issues](https://github.com/yogesh1239/AstralChain-Dynamic-FPS/issues).

## What's new in v1.1.1

v1.1.1 removes fix 3 (cutscene audio sync) from v1.1. Its 30 FPS cutscene steps made characters
look smeared in in-engine cutscenes. Cutscene motion is smooth again. The v1.0 audio drift in
some cutscenes (picture slightly ahead of the voices) comes back. Fixes 1 and 2 are unchanged.

## What's new in v1.1

v1.0 fixed the game speed, but some systems still counted frames. At high frame rates they ran
too fast. v1.1 fixes three of them ([issue #1](https://github.com/yogesh1239/AstralChain-Dynamic-FPS/issues/1)):

1. **Move and input timing.** Legion summon, recall, Sync Attacks, switches and other moves
   remember a button press for a number of frames. At 60 FPS that window was half as long, so
   moves failed. These timers now count whole 30 FPS frames, as in the original game. The same
   applies to action delays, menu repeat speed and controller rumble length.
2. **Camera smoothing.** The camera followed the player by a fixed amount per frame, so it
   felt stiff and shaky at high frame rates. It now uses the real frame time.
3. **Cutscene audio sync.** At each scene change inside a cutscene, the game dropped a small part
   of a frame. With v1.0 the picture ran up to about 0.5 seconds ahead of the voices. Cutscene
   playback now moves in whole 30 FPS steps, so it stays in time. **Side effect:** movement in
   cutscenes updates 30 times per second and can look less smooth. Gameplay is not affected.

Astral Chain runs at **30 FPS** (frames per second). The known 60 FPS patches tell the game that
**every frame lasts exactly 1/60 second**. When your PC drops below 60 FPS, the whole game
goes into slow motion. Cutscenes also need a separate cheat.

This mod uses the **real time of each frame** instead. The game then runs at the correct speed
at any frame rate from 15 FPS to 480 FPS, including in cutscenes. No cheat is needed.

## Terms

| Term | Meaning |
|---|---|
| Step | How much game time one frame moves forward. The game counts it in "30 FPS frames": 1.0 = 1/30 second. |
| Frame cap | The highest frame rate that the game allows. |
| `.pchtxt` | A text patch file that Eden (and yuzu-based emulators) applies to the game code when the game starts. |

## How it works

The game already measures the real time of each frame. Then it changes the result in three ways:

1. It never lets the step go below 1.0. This holds the game at 30 FPS or slower.
2. During cutscenes it turns on a "fixed step" switch, so every frame counts as exactly 1/30 second.
3. It asks the display for a 30 FPS cap.

This mod removes all three limits. The game keeps using its own measurement. The mod also
changes two counters that count only whole 30 FPS frames. Without that change, those counters
stop at high frame rates.

The full technical analysis of v1.0, with every address, is in [docs/TECHNICAL.md](docs/TECHNICAL.md).
The v1.1 changes are described by the comments in the patch files.

## Download

| Folder | Frame cap | Use it when |
|---|---|---|
| [`mods/Dynamic-Unlocked`](mods/Dynamic-Unlocked) | None | Your screen runs above 60 Hz, or you set a limit in the emulator. **Tested.** |
| [`mods/Dynamic-60fps`](mods/Dynamic-60fps) | 60 FPS | Your screen runs at 60 Hz. It differs from Dynamic-Unlocked only in the cap value (2 lines). |

Use **one** of the two folders, not both.

## Install (Eden, and other yuzu-based emulators)

1. In Eden, right-click Astral Chain and select **Open Mod Data Location**.
   On Linux this is `~/.local/share/eden/load/01007300020FA000/`.
2. Copy **one** folder from `mods/` into that location. Keep the inner `exefs` folder.
3. Right-click the game and select **Properties**. In the add-ons list, turn on the new folder.
4. Turn **off** all other frame-rate patches and cheats. For example: Static 60 FPS, 90 FPS,
   and the "Cutscenes Speed Fix For 60 FPS Patch" cheat. They change the same code.
5. Start the game. The game must have **update v1.0.1** installed.

Resolution patches, such as KeatonTheBot's 1062p patch, work together with this mod.

## Limits

- **Below 15 FPS** the game still goes into slow motion. The original game does the same.
- **Above 480 FPS** the game runs slightly fast. This mod uses 480 FPS as the shortest step, to
  protect game code that divides by the step.
- **Cutscene audio:** in some cutscenes the picture runs up to about 0.5 seconds ahead of the voices.
- **Camera:** some smaller camera timers still count frames. v1.1 does not change them.
- **Timers not found:** v1.1 changes the frame counters that a code search found. Others may
  still exist. Please report any move or effect that still feels too fast.
- **End credits:** one credits counter was not changed. It may run fast above 30 FPS. Not tested.
- **Real Switch hardware:** not tested. The mod puts new code in empty space at the end of the
  game code. Eden accepts this. Atmosphere was not checked.

## Credits

- **theboy181** ([Twitter](https://twitter.com/theboy181), [Patreon](https://www.patreon.com/user?u=3127108)):
  the original Static 60 FPS patch for Astral Chain.
  It identified the game's frame timer and frame cap code. This mod starts from that work.
- **KeatonTheBot**: the 1062p resolution patch, used together with this mod during testing
  ([linktr.ee/keatonthebot](https://linktr.ee/keatonthebot)). It is not included here.
- The authors of the community **"Cutscenes Speed Fix For 60 FPS Patch"** cheat. Its addresses
  helped to find the frame cap that the game uses at run time.
- Analysis and patch design: **Claude** (Anthropic) and **Codex / gpt-6.1-sol** (OpenAI), as
  author and reviewer. Testing: **yogesh1239**.

## License

[MIT](LICENSE). This repository contains no game code or game data, only patch instructions.
You need your own legal copy of Astral Chain.
