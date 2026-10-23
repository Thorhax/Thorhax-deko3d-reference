# References

Public repositories this work used, learned from, or builds on.

## Ports and guides

| Repository | What it is | Used for |
|---|---|---|
| [Thorhax/usf4-nx](https://github.com/Thorhax/usf4-nx) | Ultra Street Fighter IV for PC and Switch (this work) | The code every document here refers to; `docs/usf4-notes.md` is the full record, `docs/renut-census.md` the Renut census, `docs/deko3d-renderer-usf4.md` the case study |
| [Thorhax/usf4-nx-installer](https://github.com/Thorhax/usf4-nx-installer) | usf4-nx's installer page | Shaders and game files made in the browser; resumable download |
| [ChanseyIsTheBest/sa2-nx](https://github.com/ChanseyIsTheBest/sa2-nx) | Sonic Adventure 2 for PC and Switch | The port usf4-nx was made from; its `docs/porting-a-360-game-prompt.md` (the method), `docs/deko3d-renderer-guide.md` (the deko3d recipe) and `docs/renut-renderer.md` (the record) |
| [ChanseyIsTheBest/sa2-nx-installer](https://github.com/ChanseyIsTheBest/sa2-nx-installer) | sa2-nx's installer page | The base of usf4-nx's page and of its deko3d shader step |
| [ChanseyIsTheBest/peggle2-nx-installer](https://github.com/ChanseyIsTheBest/peggle2-nx-installer) | Peggle 2's installer page | Where the browser deko3d shader chain was first written |
| [ChanseyIsTheBest/peggle2-nx](https://github.com/ChanseyIsTheBest/peggle2-nx) | Peggle 2 for the Switch | Release shaders made by the installer page |
| [ChanseyIsTheBest/futari-nx](https://github.com/ChanseyIsTheBest/futari-nx) | A sibling port | The frame latency probe and the deferred chain, as sa2-nx carries them |
| [nebadasSwifty/masseffect-nx](https://github.com/nebadasSwifty/masseffect-nx) | Mass Effect for the Switch | The native PM4 renderer, the profiler, the CPU enhancement framework |
| [StevensND/nfsmw-nx](https://github.com/StevensND/nfsmw-nx) | Need for Speed: Most Wanted for the Switch | The Horizon layer of the SDK, the NVK driver build, profiling and PGO |
| [StevensND/nfsmw-nx-installer](https://github.com/StevensND/nfsmw-nx-installer) | nfsmw-nx's installer page | The installer pages' structure, style and service worker; the DXC WebAssembly build |
| [madelrandel-blip/NFSMW-Recompiled](https://github.com/madelrandel-blip/NFSMW-Recompiled) | NFSMW recompilation | The project the ports above started from |

## Renderer and shaders

| Repository | Used for |
|---|---|
| [masterspike52/reNut](https://github.com/masterspike52/reNut) | Renut, the Direct3D-level renderer (no licence file; used with its author's permission) |
| [renderbag/plume](https://github.com/renderbag/plume) | The graphics API layer Renut draws through |
| [devkitPro/deko3d](https://github.com/devkitPro/deko3d) | The Switch's graphics API |
| [devkitPro/uam](https://github.com/devkitPro/uam) | deko3d's shader compiler (GLSL to DKSH), on the console and as a PC tool |
| [hedge-dev/XenosRecomp](https://github.com/hedge-dev/XenosRecomp) | Xenos microcode to HLSL (the reblue fork as reNut carries it) |
| [microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler) | HLSL to SPIR-V |
| [KhronosGroup/SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) | SPIR-V to GLSL (commit `22b22f56`) |
| [rexglue/rexglue-sdk](https://github.com/rexglue/rexglue-sdk) | The recompiler, kernel and runtime |

## Console tools

| Repository | Used for |
|---|---|
| [Atmosphere-NX/Atmosphere](https://github.com/Atmosphere-NX/Atmosphere) | Custom firmware; fatal reports |
| [switchbrew/nx-hbmenu](https://github.com/switchbrew/nx-hbmenu) | The homebrew menu and its netloader (with devkitPro's `nxlink`) |
| [olliz0r/sys-botbase](https://github.com/olliz0r/sys-botbase) | Buttons, sticks and screenshots over the network |
| [ITotalJustice/ftpsrv](https://github.com/ITotalJustice/ftpsrv) | FTP server sysmodule |
| [Horizon-OC/sys-clk-plus](https://github.com/Horizon-OC/sys-clk-plus) | Clock overrides |
| [emscripten-core/emsdk](https://github.com/emscripten-core/emsdk) | Building the installer page's WebAssembly tools (the `emscripten/emsdk` Docker image) |
