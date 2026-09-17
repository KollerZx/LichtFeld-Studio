# Training NaN/Inf loss and viewer deadlock at storage growth — investigation notes

Field notes from 2026-09-17 (Ubuntu 22.04, RTX 3060 12 GB, CUDA 12.8, MRNF
strategy, 164-image COLMAP dataset of RGBA PNGs where **every image has a
slightly different resolution**). Two independent bugs showed up as
"training stops at ~5 % and the app must be force-quit". Both were reproduced
without touching the GUI (see *Reproduction method*), one is fixed, the other
is mitigated with the root cause narrowed down to the CUDA memory pool.

## TL;DR

| Symptom | Root cause | Status |
|---|---|---|
| App freezes (GNOME "not responding") right after `SplatExportableStorage grew` | Deadlock: a refining training step holds `render_mutex_` exclusively and waits for the **viewer thread** to bind new Vulkan chunks, while the viewer thread is blocked on a plain `shared_lock` of the same mutex (click-to-set-pivot, previews, captures, selection queries). | **Fixed** — `TrainerManager::acquireLiveModelReadLock()` pumps posted work while contended. |
| `Training failed: NaN/Inf loss at iteration N` at a random early iteration (280, 410, 415, 1240…) | The float32 ground-truth image buffer is corrupted mid-step (thousands of NaN/Inf values) while the model and the rendered image are still finite. Only seen with the viewer running; never in `--headless`. Points to a stream-ordering hole around the GT buffer (memory-pool reuse or loader hand-off) involving viewer-side CUDA work. | **Diagnosed, not fixed** — per-step attribution tooling added (`LFS_NAN_TRACE`); a persistent GT ring outside the pool (`LFS_GT_RING`, default on) is a low-cost mitigation whose effect is not yet statistically proven (intermittent ~40 % failure rate). |
| `init_opacity must be finite and within (0, 1) (got 1)` on resume / autosave | UI ranges allowed 1.0 while `validate()` requires (0,1) (value feeds `logit()`); plus project resume overwrote checkpoint params. | **Fixed** (UI max 0.99, `restorePendingProjectState` wired, resume guard). |
| After "Recover" of a crashed session: `makeSplatExportableInteropAllocator: shape for 'SplatData.means' needs 120000 bytes but region only holds 12` | Recovered autosave (from a killed test run) installs a trainer whose exportable region has capacity 1. | **Open**, observed once, not investigated. |

## Reproduction method (no clicking required)

- Copy the user's project (`~/.lichtfeld/projects/*.licht`) to a scratch
  folder; never resume the original. Remove `*.autosave`, `*.lock` and
  `*.recovery-session.*.tmp.licht` between runs or the GUI shows a recovery prompt.
- Launch under gdb so a hang can be captured with `kill -INT <app pid>`
  (`kernel.yama.ptrace_scope=1` prevents attaching later):
  `gdb -batch -ex run -ex "thread apply all bt" --args ./build/LichtFeld-Studio --resume copy.licht`
- `--resume` does **not** auto-start training. The running app serves MCP at
  `http://127.0.0.1:45677/mcp`: call `initialize`, `notifications/initialized`,
  then `tools/call training_start`. `render_capture` takes the blocking
  viewer-thread read lock and is a good way to hammer the deadlock path.
- `~/.lichtfeld/logs/lichtfeld.log` is appended across sessions; grep only the
  lines after the line count recorded at launch.
- `--headless -d <dataset> -o <out> --iter N` isolates the viewer.

Scripts used live in the session scratchpad (`repro5.sh`, `nan_hunt.sh`) and are
easy to recreate from the description above.

## Bug 1 — deadlock at exportable storage growth

Chain (all confirmed by code reading, the hang itself reproduced only on the
user's interactive sessions):

1. `Trainer::train_step` takes `render_mutex_` **exclusively** for refining steps
   (`trainer.cpp`, "need_exclusive").
2. `MRNF::grow_and_split → append_child_rows → preflight_grow_capacity →
   TrainerManager::growExportableForDensify` → `post_work_and_wait` posts
   `bindNewExportableChunks` to the viewer thread and waits forever.
3. If the viewer thread is inside any *blocking* `std::shared_lock(render_mutex_)`
   (`renderExpectedDepthAtPixel` from click-to-set-pivot, `renderPreviewImage*`,
   `captureViewportImage`, VkSplat selection query, cold-start fallback), it can
   never drain the work queue → both threads wait on each other.
   The per-frame render path already used `try_to_lock`, which is why the hang
   needs user interaction (or another blocking caller) to coincide with a growth.

Fix: `TrainerManager::acquireLiveModelReadLock()` (`training_manager.hpp/.cpp`)
try-locks in a loop and, on the viewer thread, pumps the posted-work queue
(`pumpPostedWorkForProjectWrite`) between attempts, logging after 5 s. All
blocking viewer-side acquisitions now go through it
(`rendering_manager_viewport.cpp`, `rendering_manager_vulkan.cpp`,
`selection_service.cpp`). Verified: 12 growth events across four 7-minute GUI
runs with ~430 blocking captures each, no stall.

## Bug 2 — NaN/Inf loss

### Evidence

Per-step attribution was added to the trainer (env `LFS_NAN_TRACE=1`): a small
CUDA kernel counts non-finite values of the ground truth, the rendered image and
the loss every iteration, rides the existing async loss-readback ring, and on
the first bad sample logs the view name, the last 48 views and a non-finite scan
of every model tensor. Reliable sample:

```
NaN trace (async): first non-finite step iter=415 view='00049.png'
gt_non_finite=4477 render_non_finite=0 loss_non_finite=1 loss=nan;
model non-finite counts: means=11625 scaling=11625 rotation=15500 opacity=3875 sh0=3875 (size=69225)
```

- GT corrupted (4 477 NaN/Inf), render finite, model rows corrupted = exactly the
  Gaussians visible in that view (3 875 rows × {3,3,4,1,1}) — i.e. the NaN
  loss's backward poisoned the visible set. Earlier crashes showed the same
  "N full rows" pattern (10 153 rows at iteration ~280).
- The GT is decoded as uint8 and converted to float32 on the training stream;
  uint8→float cannot produce NaN, so the float32 buffer is overwritten after the
  conversion by someone else.
- Statistics (each GUI run ≈ 7 min ≈ 9 000 iterations unless it failed): NaN in
  3 of 7 GUI runs without the ring and without per-step host sync, always before
  iteration 450 except one at 1 240; 0 of 3 GUI runs when the trace synchronized
  the stream every step; 0 of 1 headless run (12 000 iterations); 0 of 3 GUI runs
  with the GT ring; **but also 0 of 2 control runs with `LFS_GT_RING=0`** run
  right afterwards. The failure is intermittent (~40 %), so the ring is a
  plausible, low-cost mitigation (6 × 32 MiB VRAM), **not** a proven fix — keep
  collecting runs before drawing conclusions.
- All observed failures happened in the first ~450 iterations (epochs 2–3 of a
  164-image dataset), i.e. while the pipelined loader is still switching from the
  cold PNG path to its compressed hot cache. Worth checking that path first.
- Dataset property worth keeping in mind: every image has its own resolution, so
  per-image buffers change size every step (more pool churn than usual datasets).

### Mitigation shipped

`Trainer` keeps a 6-slot ring of raw `cudaMalloc` buffers (sized to the largest
training image) and wraps the current slot with `Tensor::from_blob(...,
training_stream_)` for the float32 GT (`trainer.cpp`, search `LFS_GT_RING`).
`LFS_GT_RING=0` restores the pool allocation. Freed in `~Trainer`.

### Root cause still open (for the pool maintainers)

The corrupted block is a ~25 MiB bucketed-pool allocation homed on
`training_stream_`. Something running only with the viewer alive obtains the same
block (or writes through a stale pointer) while the loss of the current step
still reads it. Candidates examined but not confirmed: cross-stream reuse edge in
`SizeBucketedPool::try_allocate_cached` / `free_routed` (`memory_pool.hpp`),
tensors homed on the legacy stream from viewer/loader threads, `from_blob` views
without owner tracking. The MCP `render_capture` hammering did **not** change the
failure rate, so ordinary viewer preview traffic is enough to trigger it.
Suggested next step: log cross-stream reuse of ≥16 MiB buckets with thread ids
during a GUI run with `LFS_GT_RING=0`, and correlate with the GT block address
logged per iteration.

## Other changes made during the session

- MRNF split path: `launch_sanitize_gaussians_inplace` neutralizes non-finite
  parent/child rows after the long-axis split and logs a warning
  (`densification_kernels.cu`, `mrnf.cpp`). It never fired in any run — the NaN
  does not originate in densification — but it is a cheap safety net.
- Loss readback ring depth 4 → 8 and each slot carries the view name.
- `describeNonFiniteLossState()` is also logged by the normal (non-trace) NaN
  detection path, so future reports carry the view name and a model scan.

## Open issues for the next session

1. **NaN/Inf loss root cause** (Bug 2). Reproduce with `LFS_GT_RING=0
   LFS_NAN_TRACE=1` in GUI mode (≈40 % of runs fail before iteration 450).
   Suggested instrumentation: log the GT block address per iteration and every
   cross-stream reuse of ≥16 MiB buckets (`SizeBucketedPool::try_allocate_cached`)
   with thread ids; also check the pipelined loader's cold→hot cache transition
   (`src/io/pipelined_image_loader.cpp`, decode ring leases) and viewer readers of
   `RenderOutput::target_image`. Decide afterwards whether the GT ring stays.
2. **"Recover" of a crashed session fails to start training**:
   `makeSplatExportableInteropAllocator: shape for 'SplatData.means' needs 120000
   bytes but region only holds 12` (`project_lifecycle.cpp` recovery path →
   `training_manager.cpp` interop allocator). Seen once after killing a run that
   had an autosave; reproduce by killing the app mid-training and choosing
   Recover on relaunch.
3. **Training panel shows empty fields after a project is restored** even though
   the parameters are applied correctly (reported by the user after
   `restorePendingProjectState` was wired into project hydration in
   `project_lifecycle.cpp`). Likely the UI/RmlUI data model is not refreshed after
   `ParameterManager` is repopulated; check whether switching strategy tabs
   repaints the values.
4. **MRNF split sanitizer** (`launch_sanitize_gaussians_inplace`) never triggered
   in any run. Keep as safety net or drop once Bug 2 is fixed.
5. **`clang-format` is not installed on this workstation**, so the pre-commit hook
   could not format the commit `fix/training-deadlock-nan-diagnostics`. Install it
   (`sudo apt install clang-format`) and run it over the touched files before
   opening a PR.
6. **Automated tests not run** in this environment (`tests/test_parameter_manager.cpp`,
   `tests/test_project_document.cpp` need the LibTorch SDK). Run them on a
   machine that has it before merging the parameter-persistence changes.

## Environment switches added

| Variable | Default | Effect |
|---|---|---|
| `LFS_GT_RING` | on | `0` disables the persistent GT ring (pool allocation as before). |
| `LFS_NAN_TRACE` | off | `1` enables per-step gt/render/loss non-finite counting and attribution. Small per-step cost, no host sync. |
