# Astral Chain — Dynamic FPS

> **Experimental.** Tested only on **Astral Chain v1.0.1** (build ID `4B159F0F7A360669`) with
> the [Eden](https://eden-emu.dev) emulator on Linux, with the **Dynamic-Unlocked** variant and the
> 1062p resolution patch. Gameplay, menus and cutscenes run at the correct speed in that test.
> It was not tested on a real Switch, on other emulators, or through a full play-through.
> Please report problems in the [issues](https://github.com/yogesh1239/AstralChain-Dynamic-FPS/issues).

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

The full technical analysis, with every address, is in [docs/TECHNICAL.md](docs/TECHNICAL.md).

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
