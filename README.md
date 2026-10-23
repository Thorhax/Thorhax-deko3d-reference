# Thorhax deko3d reference

What it took to bring a statically recompiled Xbox 360 game to the Nintendo Switch with Renut's renderer on
**deko3d**, the Switch's own graphics API, at a locked 60 fps, written down for the next port that wants the same
result. The source of everything here is [usf4-nx](https://github.com/Thorhax/usf4-nx), Ultra Street Fighter IV on
the [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk), whose v0.8.0 (October 2026) moved from a Vulkan renderer
on Mesa's NVK to Renut on deko3d.

This repository holds documentation only: no code, no game data.

## Results

Ultra Street Fighter IV, a Training fight, a V1 Switch, docked, the same build with only the renderer changed:

| | Vulkan renderer (NVK), 540p | NVK, 720p | Renut on deko3d, 720p |
|---|---|---|---|
| Frame rate | 59.9 | 53.5 | **59.9** |
| GPU per frame | 6.9 ms | 9.3 ms | 8.4 ms |
| Render / ring thread per frame | 7.4 ms | 18.6 ms | **2.8 ms** |
| CPU total (of 400 %) | 167 % | 155 % | **110 %** |

With the GPU at 768 MHz, Renut at 720p holds 59.2-59.9 fps across fights on four stages. The release NRO dropped
from 66.7 MB to 56.3 MB (no Vulkan driver inside) and carries no translated game shader.

## Where to start

The step-by-step recipe for a Renut-on-deko3d port is ChanseyIsTheBest's
[sa2-nx `docs/deko3d-renderer-guide.md`](https://github.com/ChanseyIsTheBest/sa2-nx/blob/main/docs/deko3d-renderer-guide.md).
Read it first. The documents here add a second game's experience on top of it:

| Document | What it covers |
|---|---|
| [01 - deko3d case study](docs/01-deko3d-case-study.md) | The USF4 port end to end: what was taken from where, the census, the console-only bugs (a depth target smaller than the colour target clips everything on Maxwell; a log socket that froze the game), the render-thread work (stable linear textures: 6.4 -> 2.8 ms a frame), compile priority, late input from the game's frame limiter, the renderer comparison and how it first went wrong, release shaders made in the browser, Switch button icons, traps, open items, a checklist |
| [02 - console test rig](docs/02-console-test-rig.md) | Working on a console with nobody at it: hbmenu's netloader and nxlink, sys-botbase, an FTP sysmodule, live logs, the profiler, Atmosphère's fatal reports, clock overrides (never the memory clock), auto-sleep |
| [03 - game-side CPU lessons](docs/03-game-side-cpu-lessons.md) | Fixes in the game's own threads that any renderer needs: spinning threads starving others on Horizon, sleeping waits and frame limiter, native memcpy, SD card file-system costs, one tile instead of three |
| [04 - shipping without game data](docs/04-shipping-without-game-data.md) | Publishing only the NRO; an installer page that makes the game files and the shaders from the player's own disc; pinning by SHA-256; browser pitfalls; telling players how to update |
| [References](docs/references.md) | Every public repository involved |

## The short version

1. Follow sa2-nx's guide: census the game's Direct3D, bring Renut up on the PC behind a boot key, build the deko3d
   shader chain, test in Ryujinx, drop the Vulkan driver.
2. Get onto the real console early and test it unattended (document 02). Emulators hide console-only behaviour
   (Maxwell clips to the smallest bound target; Ryujinx does not) and their timings are not the console's.
3. Profile the render thread on the console before optimizing; fix what the profile shows (document 01, section 6).
4. Make the game's own threads behave on Horizon (document 03).
5. Compare renderers only after proving each one draws the scene, with one build and the same clocks.
6. Ship without game data: the NRO holds the shader table, the installer page makes `deko3d_glsl.bin` from the player's
   disc, every result pinned by SHA-256 and tested on the console (document 04).

## Credits

This work stands on other people's work:

- **masterspike52** for [reNut](https://github.com/masterspike52/reNut), the Direct3D-level renderer, used with the
  author's permission.
- **ChanseyIsTheBest** for [sa2-nx](https://github.com/ChanseyIsTheBest/sa2-nx), its installer, its deko3d guide and
  the sibling ports ([peggle2-nx](https://github.com/ChanseyIsTheBest/peggle2-nx),
  [peggle2-nx-installer](https://github.com/ChanseyIsTheBest/peggle2-nx-installer),
  [futari-nx](https://github.com/ChanseyIsTheBest/futari-nx)) whose public work this port uses: the deko3d back end,
  the shader chain and the render-thread tuning as sa2-nx carries them, the release shaders made in the browser
  (peggle2-nx) and the frame latency probe (futari-nx).
- **NebadasSwifty** for [masseffect-nx](https://github.com/nebadasSwifty/masseffect-nx), **StevensND** for
  [nfsmw-nx](https://github.com/StevensND/nfsmw-nx) and its installer page, and **madelrandel-blip** for
  [NFSMW Recompiled](https://github.com/madelrandel-blip/NFSMW-Recompiled).
- **Tom Clay** and the Xenia contributors for the [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk).
- **fincs** and devkitPro for [deko3d](https://github.com/devkitPro/deko3d) and [uam](https://github.com/devkitPro/uam);
  renderbag for [plume](https://github.com/renderbag/plume); hedge-dev for
  [XenosRecomp](https://github.com/hedge-dev/XenosRecomp); the Khronos Group and Microsoft for SPIRV-Cross and DXC.
- **ITotalJustice** ([ftpsrv](https://github.com/ITotalJustice/ftpsrv)), **olliz0r**
  ([sys-botbase](https://github.com/olliz0r/sys-botbase)), the Atmosphère, switchbrew and Horizon OC developers, for
  the tools that made unattended console testing possible.

**Thank you** as well to the developers who shared knowledge and work from projects that are not public. Their
private material is not linked or reproduced here; where their results reached this work, it was through what their
public repositories carry.

## Licence

The documents in this repository are under the Creative Commons Attribution 4.0 International licence
([LICENSE](LICENSE)). The projects they describe keep their own licences (see each repository). Ultra Street Fighter
IV is a trademark of Capcom; this work is not affiliated with or endorsed by Capcom, Nintendo or Microsoft.
