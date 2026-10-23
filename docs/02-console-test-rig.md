# An unattended console test rig

Every console measurement and fix for usf4-nx 0.8.0 was made with nobody at the console: launching new builds,
playing menus and fights, reading logs and profiles, changing clocks, and recovering from hangs. This is the setup,
the commands, and what went wrong on the way.

**Safety first.** sys-botbase has no authentication: keep the console on your own network and never forward its port.
The SD card's `/switch` folder can hold the console's keys (`prod.keys`, `title.keys`) from other homebrew; a test
script must never read or copy them. Never change the memory clock remotely (section 6).

## 1. The pieces

| Piece | Project | Role |
|---|---|---|
| Atmosphère (emuMMC) | [Atmosphere-NX/Atmosphere](https://github.com/Atmosphere-NX/Atmosphere) | The console's custom firmware; its fatal reports explain crashes |
| hbmenu + nxlink | [switchbrew/nx-hbmenu](https://github.com/switchbrew/nx-hbmenu), devkitPro's `nxlink` | Sends and starts an NRO over the network, with arguments |
| sys-botbase | [olliz0r/sys-botbase](https://github.com/olliz0r/sys-botbase) | Buttons, sticks and screenshots over TCP (port 6000) |
| ftpsrv (sysmodule) | [ITotalJustice/ftpsrv](https://github.com/ITotalJustice/ftpsrv) | Files to and from the SD card |
| Horizon OC's sys-clk | [Horizon-OC/sys-clk-plus](https://github.com/Horizon-OC/sys-clk-plus) | CPU and GPU clocks, set from the game for tests |
| The game's own log stream and profiler | ReXGlue SDK, as usf4-nx carries it | What happened, per thread, every 10 s |

## 2. Launching builds: hbmenu's netloader and nxlink

1. Start hbmenu in **title takeover** (hold R while starting an installed game, and keep holding through the user
   picker): the album applet does not give a ReXGlue game enough memory.
2. Press Y in hbmenu: the netloader waits on port 28280.
3. From the PC:
   ```sh
   nxlink -a <console ip> -p usf4-nx/usf4_test.nro out/nx-release/usf4.nro -- \
       --log_network=<pc ip>:28772 --some_cvar=value
   ```
   - `-p` is relative to `sdmc:/switch/`, and that folder must already exist (the NRO finds its data beside itself).
   - Arguments after `--` reach the game as command-line settings.
   - `nxlink -s` (its stdio server) takes port 28771; give the game's log stream another port.
4. If the console's network came up after the netloader opened, nxlink cannot connect: close the netloader (B) and
   open it again (Y).
5. Do not probe port 28280 with a bare TCP connection to see whether it is open: hbmenu then shows "Error getting name
   length" and waits for a button.

With sys-botbase, all of this is scriptable: closing the running title is HOME, X, A; starting hbmenu again is
holding R, A on the title, A on the user, releasing R, then Y for the netloader.

## 3. Driving the game: sys-botbase

Commands are text lines on TCP port 6000:

| Command | Use |
|---|---|
| `click A`, `press R`, `release R` | Buttons (`DUP`, `DDOWN`, `PLUS`, `MINUS`, `HOME`, `ZL`, ...) |
| `setStick LEFT 0x7FFF 0x0` | Stick positions |
| `pixelPeek` | A screenshot: one line of hex, a JPEG |
| `getTitleID` | Which title is running |

Things learned:

- **Buttons follow the game's layout.** usf4-nx maps buttons by position by default (the Switch's B is the Xbox's A),
  so B confirms and A goes back in the menus.
- **A black screenshot is evidence, not a capture problem.** usf4-nx's native renderer took black screenshots for a
  day because it was drawing nothing (its shader library was missing from the card). Once it drew, its screenshots
  worked like any other.
- **Hands-free coverage.** Left at the title screen, the game plays attract-mode demo fights on random stages; a
  random-input player (directions, attacks, never the button that backs out or pauses) covers Arcade.
- **Training mode's stage is fixed** in USF4: the world map shown after character select is a display only.

## 4. Files: ftpsrv as a sysmodule

ITotalJustice's ftpsrv, built from source as a sysmodule (`cmake --preset switch` in the `devkitpro/devkita64`
Docker image; title id `420000000000011B`; its configuration in `/config/ftpsrv/config.ini`: user, password, port,
LED off, timeout). Measured: 20 of 20 sessions and 30 of 30 listings in one session, 64 MB at 2.2 MB/s up and
8.6 MB/s down. Python's `ftplib` is enough as a client.

- **sys-ftpd-light was dropped:** on the Switch its `poll()` missed about half of the passive-mode data connections, so
  transfers hung; a non-blocking accept did not fix it. (It also had a button combination that ends in a fatal error,
  and a GCC 15 build fix to make.)
- **A file the game holds open cannot be read** ("Device or resource busy"): close the game first. The game's log and
  shader caches are such files.
- Everything a test changes on the card (a settings file, caches moved aside) is put back afterwards.

## 5. Logs and profiles

- **Live log:** `log_network=<pc ip>:<port>` streams the log over TCP. The PC listener must serve each connection on
  its own thread with TCP keepalive: a listener still blocked on the dead connection of a crashed run left the new
  one unread, the TCP window filled, and the game froze inside the log sink (the socket is now non-blocking).
- **Files on the card:** `logs/usf4_NNN.log` and `logs/rex/rex_profile.log` beside the NRO.
- **The profiler** (ReXGlue SDK `src/ui/switch_perf.cpp`, from masseffect-nx): every 10 s, CPU per thread, priority
  and core mask, the clocks the console really ran at, and with `switch_profile_stacks = true` stack samples of every
  thread, including blocked ones (it costs about a core; leave it off for timing runs). Symbolize the addresses with
  `aarch64-none-elf-addr2line -f -C` on the **unstripped** ELF of the same build.
- **Atmosphère's fatal reports** (`/atmosphere/fatal_errors/report_*.bin`, an "AFE2" record): the program id, the
  registers and the stack. A whole-console crash during usf4-nx's first run was program `0x010041544D530000`
  (ams.mitm) aborting on a kernel result, not the game; it went away when an aggressive memory overclock and undervolt
  were lowered. Decode the report before debugging the game.

## 6. Clocks

Horizon OC's sys-clk accepts a manual override per module (CPU, GPU, memory), the same mechanism its overlay's
sliders use. usf4-nx sets CPU and GPU from the game for tests (`usf4_test_clock_override = "CPU_MHZ,GPU_MHZ"`,
`SetCpuGpuOverride` in the SDK's `switch_sysclk.cpp`):

- **Only CPU and GPU, never memory.** A memory clock change can need a reboot, and on a console set up like this one
  the reboot lands in hekate with nobody there.
- The override stays after the game exits. Read the clocks the console runs at first (the profiler's `clocks:` line),
  and set exactly those back afterwards.
- If the game crashes during a test, the clocks stay at the test values; prefer tests that lower clocks.
- The profiler's `clocks:` line is the proof of what a run used.

## 7. Keeping the console awake

Set **System Settings > Sleep Mode > Auto-Sleep** to Never for both the TV and the console screen. A sleeping
console drops its network and cannot be woken remotely. (Virtual button presses may reset the idle timer, but builds
and analysis leave long gaps without any.)

## 8. A test's checklist

1. Console reachable, the netloader open, the log listener running (threaded).
2. Send the build with its settings; check the log's first lines: the renderer, the resolution, the shader package,
   and the settings you meant to change (the player's settings file can override the command line).
3. Drive to the scene; take screenshots at fixed times; compare two screenshots to detect a frozen picture.
4. Wait the measured time; then pull the profile and the log over FTP (after closing the game if a file is busy).
5. Read: frame rate per 5 s, the longest frames, the renderer's 10 s lines, CPU per thread, clocks.
6. Put back anything the test changed (settings files, clocks, caches).
