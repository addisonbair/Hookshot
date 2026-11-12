# Steam Deck branch

This branch (`deck` on [addisonbair/Hookshot](https://github.com/addisonbair/Hookshot/tree/deck)) is upstream **v2.0.14**
plus fixes measured on a Steam Deck. It is not an official Hookshot release. It exists because v2.0.14 is the fastest
Hookshot on the Deck today: 30-40 % faster than current upstream main at the same scene (below). Deck tuning is paused for now. A change goes on this
branch only when alternating A/B runs on a Deck show it faster, and a check shows it correct (screenshots, a flicker check
over 200+ frames, or a byte check of what the game reads back). Findings and wins also go to
[#1000](https://github.com/NotRexed/Hookshot/issues/1000), and small general fixes go upstream as PRs.

What is on it now, on top of v2.0.14:

| Change | Why |
|---|---|
| Task and mesh programs built with the driver's optimizer on RADV (upstream #1965, merged) | Without it the GPU resets while the save loads at the Great Deku Tree. Same effect as `RX_VK_MESH_OPT=1` on stock v2.0.14 |

## Settings for the Deck

On this branch or stock v2.0.14, in `hookshot_env.txt` (one per line):

```
RX_VK_PER_QUEUE_FENCES=0
RX_VK_WAIT_GPU=0
RX_VK_BATCH_MAX=4
RADV_PERFTEST=cswave32
```

Add `RX_VK_MESH_OPT=1` on stock v2.0.14 (this branch has it built in). In the Insert menu: Upscaler **AMD FSR 3.1**,
Render resolution **Performance**, Frame generation **2x**, Descriptor reuse **on**, Sampler cache **off**. In the
Deck's Quick Access menu → Performance: **Manual GPU clock 1200 MHz**. The automatic clock starves the GPU under the
15 W limit. For an even 24 fps base with frame generation, set the refresh rate to 48 Hz and the framerate limit to 48.
This gives ~24 fps base (~48 shown) in normal play, and ~20 in the heaviest spots of Kokiri Forest.

On upstream v2.1.0 or main, things differ:

- `RX_VK_MESH_OPT=1` until a release includes #1965.
- Upscaler: **the game's own**. On main, AMD runs a cheap bilinear stand-in in place of the game's network, and FSR 3.1
  costs 4-8 ms more GPU on the Deck (measured, #2110). On v2.0.14 the network still runs (~12 ms), so FSR 3.1 wins there.
- Anisotropic filtering: the game's own. 16x costs ~1.4 ms of GPU on the Deck.
- Don't set `RX_VK_BATCH_IDLE=0` alone on main: it saves CPU but makes readback waits longer (net −0.8 fps). #2422
  pairs it with a fix.

## Measurements

All in one deterministic scene: standing in Kokiri Forest next to Saria (~2,400 draws), headless, GPU at 1200 MHz, from
the same save. Each line is one run, and lines with two values are two passes.

| Steam Deck LCD (Van Gogh, RADV, SteamOS) | fps | GPU ms |
|---|---|---|
| v2.0.14, the settings above | 19.4-20.9 | 28.2 |
| main eaef1460, default switches | 13.5 / 13.8 | 30.0 |
| main + #2422 (no idle send + readback send) | 14.3 / 14.4 | 29.9 |
| main + #2422, #2125's readback catch-up off | 14.6 | 29.7 |

Where main's time goes on the Deck (perf context switches with call stacks, per thread):

- **The game's main thread looks twice as busy on main (~35 vs ~16 ms of CPU a frame), but most of that is spinning.**
  The job system sleep-polls (`sched_yield`) for workers stuck in readback waits: on main ~1 a frame of 10-14 ms, on the
  newest submission; on v2.0.14 ~6 a frame of ~0.6 ms. The prime suspect is #1133 (direct-path write-back pages on GPUs
  without a VRAM mirror, so every iGPU). It is not confirmed on a Deck; `RX_VK_DIRECT_WB_PAGES=0` is the A/B, and #1126's
  foliage glow is the risk to watch for.
- #2125's readback catch-up is a small part: turning it off gives ~+1.8 fps.

The Deck's own ceiling: even on v2.0.14 the frame is ~37 ms of replay that runs in order under the order lock (held
93-98 % of the time on a P16 stand-in). The replay thread's ~20 ms of waits in `present()` are mostly that replay running
on the helper thread, not idle handoff. Faster single-thread CPUs are what move this.

Measured and not helping on the Deck: async replay (`RX_VK_ASYNC=1`: slower, and exposure flicker), the recorder thread,
parallel replay, per-queue fences, batch sizes, mirrors, DCC (broke lighting), thread pinning, a cross-frame
descriptor-set cache (replay 12.2 -> 13.7 ms on the P16), deferring the present so the game doesn't wait for it (flat), and
skipping the memory snapshots (faster, but they exist for culling correctness). Hyrule Field is CPU-bound (~10 fps): a
lower GPU clock (800-1000 MHz) leaves more of the 15 W for the CPU.

## Using this branch

The changes here are in the renderer only, so a v2.0.14 game folder (built with the v2.0.14 AppImage) only needs a new
`bin/hookshot_vk.so`. From a clone of this branch, with the tools of a source build ([readme](../readme.md#build-from-source)):

```sh
RX_VK_ONLY=1 RX_RELEASE=1 RX_MARCH=x86-64-v3 python3 -I nx2/build.py --standalone
cp <game folder>/bin/hookshot_vk.so <game folder>/bin/hookshot_vk.so.v2.0.14   # keep the old one
cp work/sa/build/hookshot_vk.so <game folder>/bin/
```

`RX_CPM` must point at the tools' `cpm` folder (the Vulkan headers), or the renderer is left out without a word. To go
back, copy `hookshot_vk.so.v2.0.14` over it.

## Measuring on your own Deck

The setup behind these numbers, so other Deck results line up:

- **Headless:** `HOS_HEADLESS=1 HOS_AUDIO=0 HOS_INPUT_FRAME_CLOCK=1 HOS_REALTIME_STEP=0 HOOKSHOT_NO_UPDATE_CHECK=1`. Pass
  A presses through `HOS_INPUT_LIVE=<file>`, one per new frame line, only until a line shows 100+ draws, then none. A
  later press starts the Saria conversation, another scene with ~600 fewer draws.
- **GPU clock fixed with no root:**
  `busctl --user set-property com.steampowered.SteamOSManager1 /com/steampowered/SteamOSManager1 com.steampowered.SteamOSManager1.GpuPerformanceLevel1 GpuPerformanceLevel s manual`,
  then the same with `ManualGpuClock u 1200`.
- **Readings:** the last ~10 `presents, N fps; a frame: N draws` lines, the `replay time (ms)` line, and `GPU time`.
  Alternate A and B at least twice.
- **Per-thread waits:**
  `perf record --switch-events -e context-switches -c 1 --call-graph dwarf,16384 -t <tid> -- sleep 5`. User-level perf
  works on SteamOS; tracepoints don't.
