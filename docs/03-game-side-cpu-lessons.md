# Game-side CPU lessons (any renderer)

A new renderer only shows its speed if the game's own threads leave it room. These fixes were made for usf4-nx while
it still drew with the native Vulkan renderer (0.5-0.7.x); they live in the game's code paths, not the renderer, so
they carried over to deko3d unchanged and are part of why 0.8.0 holds 60 fps with the CPU at 1785 MHz. Each one is a
setting in [usf4-nx](https://github.com/Thorhax/usf4-nx) (files under `app/src/`), checked against the guest code's
hash before it patches anything. The full record with every measurement is usf4-nx's
[docs/usf4-notes.md](https://github.com/Thorhax/usf4-nx/blob/main/docs/usf4-notes.md).

## 1. Horizon does not time-slice equal priorities: spinners starve

**What happened (0.5.6):** menus ran at about 58 fps but fights fell to 10-12 fps with all three cores at 100 % and the
GPU at 17 %. The four job worker threads spent 80-90 % of their samples in the Xbox 360 C library's job-list pop
(`sub_82966BA8`), spinning while the batch flag was set; the main thread, which publishes the batch, was 92 % waiting.

**Why:** all game threads run at Horizon priority 0x3B. Horizon switches between threads of equal priority only when
one yields (or at its 10 ms rotation), so a woken main thread waited for a core that a spinning worker held. Until
then, a global recursive mutex the code generator took around every interrupt-off section (`mfmsr`/`mtmsrd`) slept
under contention and hid the problem; the optimised codegen's `skip_msr` removed it.

**Fix:** `usf4_cpu_job_pop_yield` (`src/usf4_job_pop_yield.cpp`): an empty pop yields with core migration
(`svcSleepThread(YieldType_WithCoreMigration)`). Console: locked 60 fps again.

**Lesson:** the PC (Linux time-slices) cannot show this class of bug. Any guest spin loop is a starvation risk on
Horizon; find them in the console profile (threads at high CPU in a tiny function) and make them yield or sleep.

## 2. Make the game's waits sleep instead of spinning

| Setting | Guest code | What it does | Console result |
|---|---|---|---|
| `usf4_cpu_gpu_wait_sleep` | Direct3D's wait iteration (`sub_821F54E0`, from sa2-nx's `sa2_gpu_wait.cpp`) | Sleeps until the renderer reports progress (at most 1 ms) when nothing can have changed since the previous iteration | The main thread sleeps ~8.5 s per 10 s instead of spinning (PC) |
| `usf4_cpu_job_wait_block` | The main thread's "wait for my job batch" loop (`sub_82966C90`'s `Sleep(0)`) | Returns at once when a job finished, else yields 20 µs, then sleeps on a condition variable bumped by every job pop | Yields 0.74-0.98 M/s -> 0.45-0.64 M/s, main thread 88 -> 81 % of a core |
| `usf4_cpu_frame_limiter_sleep` | The game's frame limiter (`sub_82756DE8`: `Sleep(0)` + clock read until 16.667 ms) | Sleeps for the missing time minus 1 ms, then spins as before, so the frame starts at the same moment | Yields 0.45 -> 0.15 M/s, clock reads 0.5-1.0 M/s -> 0.13 M/s, main thread 81 -> 54 %, CPU total 235 -> ~205 % |

The frame limiter's wait is also what late input can use (the deko3d case study, section 7): the time the limiter
still has to wait at its first look is time the controller read could have waited.

## 3. Run the game's hot C library routines natively

The main game thread spent about 13 % in the Xbox 360 C library's copy routines (`sub_8220FFE8` with its VMX helper,
and `sub_82514750`). `usf4_cpu_native_memcpy` (`src/usf4_native_copy.cpp`) copies with the host's `memcpy` when the
ranges neither overlap nor wrap, otherwise the original runs.

- An audit mode compared about 14.5 million copies after the original ran: 0 differences.
- The call sites are direct calls after code generation; `tools/direct_calls.py --redo` makes them call the hooks.
- The host `memcpy` uses only instructions the SDK's fault handler decodes (LDP/STP/LDR/STR/LDUR/STUR).
- Console: the copies' share of the main thread fell from about 13 % to 3.5 %.

## 4. File system: the SD card is slow per operation

- **Save mounts:** USF4 opens its save (314 files, about 16 MB) before and after every fight. Each mount listed and
  stat'ed every file (about 0.7 ms each on the SD: 216-570 ms per mount, plus the same again for the log line).
  Writable folders are now indexed from the directory listing alone (`vfs_light_scan_writable`; `fsDirRead` gives
  name, type and size, 64 entries per call): 5-6 ms a mount, no hitch at the save opens.
- **Save writes:** `FILE_OVERWRITE_IF` was stat + delete + create + open (18-31 ms a file). An existing file is now
  truncated in place, without the stat: 9-23 ms, the FAT truncation itself.
- **Game data:** read straight from the extracted disc folder, with an index of the folder kept in `cache/` so the
  tree is not walked at every start (1,049 entries mounted in 344 ms on the console, 31 ms of it reading the index and
  building the tree).

## 5. Less GPU work from the game's own choices

- **One tile instead of three** (`usf4_single_tile`): USF4 draws 720p as three predicated strips with 2x MSAA, which
  costs every draw three times on a renderer without EDRAM. Three mid-function hooks choose the game's own one-tile
  preset with no MSAA: draws per frame -49 %. (A memory model that wrapped past 2048 EDRAM tiles cost more than it
  saved until it was given 4096 tiles: check the work counters, not only the draw counts.)
- **Video clip recording off** (`switch_video_recording = false`, `appletSetGamePlayRecordingState`): holding Capture
  to record a clip crashed the game with application memory full.

## 6. What was left as it is

Single 43-50 ms frames at specific moments of a fight (round start, KO, some pop-ups), about 30 s apart: the main
thread's samples were in the game's own job work (game logic), not a wait on the renderer. They were measured with a
per-frame `[hitch]` line (the frame split into GPU waits, job waits, the frame limiter and "the rest") and left alone,
since fixing them would mean rewriting game code natively.

## 7. How these were found

- The console profile with stack sampling (the console test rig document, section 5), symbolized on the build's
  unstripped ELF.
- A per-frame hitch line and 10 s summaries of yields, sleeps and clock reads per caller.
- Every patch of guest code checks an FNV-1a hash of the function's bytes first and stays off if the code differs.
- Each change has its own setting, so a console test can turn one off from the settings file.
