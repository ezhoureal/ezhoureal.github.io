---
layout: post
title: "Pipeline Parallelism in Rosen"
date: 2026-10-09 17:30:00 +0200
categories: HarmonyOS
---

In Part 3 we looked at how Rosen avoids doing work it doesn't need to do. This part is about the work that *does* need to be done, and how we accelerate that via parallelism.

The idea is simple. Every frame Render Service does three very different kinds of work:

1. **Process the node tree.** Apply app transactions, run animations, compute dirty regions and occlusion, prune whatever is invisible.
2. **Build drawing ops and submit them to the GPU.** Walk the surviving tree, record draw commands, flush them.
3. **Hardware composition.** Hand the final layers to the display hardware (HWC) and get buffers back.

Originally these ran mostly back to back inside one VSync period. We split them into a three-stage pipeline: a **main thread**, a **render thread**, and a **composer thread** (historically called the *hardware thread*). Each stage works on a different frame at the same moment.

The headline result: **tail latency dropped sharply and the jank rate fell by about 80%.** Most of this post is about *how* that number moved so much, and what we had to build so that three threads could share one render tree without tripping over each other.

## 1. The Problem: A Frame Is Only As Fast As Its Sum

At 120Hz you get about 8.3ms per frame. In a serial design, the time a frame takes is:

```text
T_frame = T_prepare + T_draw + T_compose
```

A frame is janky when `T_frame` goes over the budget. The catch is that each stage has its own **independent** spikes:

- **Prepare** spikes when an app sends a huge transaction, or when a window animation dirties a large subtree.
- **Draw** spikes on heavy blur, a cold shader cache, or a big texture upload.
- **Compose** spikes when HWC validation fails and falls back to GPU redraw, or when the display driver stalls.

In a serial design, *any* of those spikes going over the budget drops a frame, and two medium spikes landing together drop one too. The tail of `T_frame` is the tail of a **sum** of random variables, which is much fatter than the tail of any one of them.

Pipelining changes the question. Once each stage runs on its own thread and works on a different frame, throughput depends on the slowest stage, not the total:

```text
Throughput limit ≈ max(T_prepare, T_draw, T_compose)
```

Every stage now gets close to a full VSync period to itself. Take a 6ms frame (2ms per stage) whose draw spikes to 5ms. Serially, that is a 9ms frame and a dropped frame. Pipelined, it is just a 5ms draw, well within budget. **This is where the tail-latency improvement comes from.** The average frame barely changed. What changed is that the 99th-percentile frame stopped going over the deadline.

![Serial vs. pipelined execution: a 5ms draw spike blows the 8.3ms budget when stages run back to back, but fits when each stage has its own thread]({{ "/assets/post4_001_serial_vs_pipelined.svg" | relative_url }})

## 2. The Three Stages

Here is what each thread owns in the current code:

```text
VSync
  │
  ▼
┌──────────────────────────────────────────────────────────────────┐
│ RSMainThread  (stage 1: node tree processing & pruning)          │
│   ConsumeAndUpdateAllNodes   ← latch app buffers                 │
│   ProcessCommand             ← apply RSTransactionData           │
│   Animate                                                        │
│   CollectInfoForHardwareComposer  ← pick HWC candidate layers    │
│   Render() → QuickPrepare    ← dirty region, occlusion, pruning  │
│   OnUniRenderDraw() → drawFrame_.PostAndWait()   ── handoff ──┐  │
└───────────────────────────────────────────────────────────────┼──┘
                                                                │
┌───────────────────────────────────────────────────────────────▼──┐
│ RSUniRenderThread  (stage 2: draw op construction & GPU submit)  │
│   Sync()                 ← staging → render-side snapshot        │
│   UnblockMainThread()    ← main thread is free from here on      │
│   Render()                                                       │
│     rootNodeDrawable_->OnDraw(canvas)   ← build draw ops         │
│     RequestFrame / renderFrame->Flush() ← submit to GPU          │
│     CreateLayer…  → CommitLayers()                ── handoff ──┐ │
└────────────────────────────────────────────────────────────────┼─┘
                                                                 │
┌────────────────────────────────────────────────────────────────▼─┐
│ RSRenderComposer  "CompThread_<screenId>"  (stage 3: HW compose) │
│   ComposerProcess                                                │
│     hdiOutput_->SetRSLayers(layers)                              │
│     hdiOutput_->Repaint()        ← HWC commit                    │
│     hdiOutput_->ReleaseLayers()  → buffers back to render thread │
└──────────────────────────────────────────────────────────────────┘
```

When all three are busy, the timeline looks like this:

```text
VSync:      │   V0    │   V1    │   V2    │   V3    │   V4    │
────────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
Main        │ prep F0 │ prep F1 │ prep F2 │ prep F3 │ prep F4 │
Render      │         │ draw F0 │ draw F1 │ draw F2 │ draw F3 │
Composer    │         │         │ comp F0 │ comp F1 │ comp F2 │
```

Three frames are in flight at once. The diagram is idealized, though: stages don't line up exactly on VSync boundaries. The render thread starts as soon as the main thread hands off, and the composer commit is timed on purpose (Section 5).

The rest of this post covers the two handoffs, because both are hard in their own way.

## 3. Handoff #1: Main → Render (Sharing a Tree Without Sharing a Tree)

The main thread mutates the render tree on every frame. The render thread needs to *read* that tree while drawing the previous frame. A global lock around the tree would put us right back to serial execution. Rosen avoids this by **never letting the two threads touch the same data**.

### 3.1 Node vs. Drawable

Every `RSRenderNode`, which is owned and mutated by the main thread, has a matching `RSRenderNodeDrawable`, which is owned and read by the render thread. Each piece of render state exists twice, once on each side:

```text
RSRenderNode (main thread)              RSRenderNodeDrawable (render thread)
──────────────────────────              ────────────────────────────────────
stagingRenderParams_        ──Sync──▶   renderParams_
stagingDrawCmdList_         ──Sync──▶   drawCmdList_
drawable slots (staging)    ──Sync──▶   drawable slots (render)
```

During prepare, the main thread writes only to the `staging*` side. During draw, the render thread reads only the render side. The two sides meet at exactly one point: `Sync()`.

### 3.2 Sync: Swap, Don't Copy

`Sync()` is the only moment both threads look at the same state, so it has to be short. Three design choices keep it that way.

**Only dirty nodes are synced.** While a node is being prepared, if anything about it changed, it adds itself to a pending list:

```cpp
void RSRenderNode::AddToPendingSyncList()
{
    if (addedToPendingSyncList_) {
        return;
    }
    if (auto context = GetContext().lock()) {
        context->AddPendingSyncNode(shared_from_this());
        addedToPendingSyncList_ = true;
    }
    ...
}
```

`RSDrawFrame::Sync()` walks only `pendingSyncNodes_`, not the whole tree. Thanks to Part 3's dirty tracking, that list is usually tiny.

**Containers are swapped, not copied.** Draw command lists move across with `std::swap`:

```cpp
// RSRenderNode::OnSync (simplified)
if (drawCmdListNeedSync_) {
    std::swap(stagingDrawCmdList_, renderDrawable_->drawCmdList_);
    stagingDrawCmdList_.clear();
    drawCmdListNeedSync_ = false;
}
```

**Params carry dirty bits.** `RSRenderParams::OnSync` checks a `dirtyType_` bitset, so expensive fields such as the matrix are only swapped when they actually changed:

```cpp
if (dirtyType_.test(RSRenderParamsDirtyType::MATRIX_DIRTY)) {
    targetPtr->matrix_.Swap(matrix_);
    dirtyType_.reset(RSRenderParamsDirtyType::MATRIX_DIRTY);
}
```

Frame-global state such as screen info, the global dirty region and the list of HWC-enabled drawables travels in a single `RSRenderThreadParams` object. It is `std::move`d into the render thread, so ownership changes hands and nothing gets copied.

### 3.3 The Main Thread Only Waits for Sync

This is the key detail. In `RSDrawFrame::PostAndWait()`, the main thread waits **only until `Sync()` is done**, then moves on to the next frame:

```cpp
// RSDrawFrame::PostAndWait, ASYNC (simplified)
std::unique_lock<std::mutex> frameLock(frameMutex_);
canUnblockMainThread = false;
unirenderInstance_.PostTask([this]() {
    RenderFrame();          // Sync() → UnblockMainThread() → Render() → ...
});
frameCV_.wait(frameLock, [this] { return canUnblockMainThread; });
```

And inside `RenderFrame()` on the render thread:

```cpp
Sync();                 // staging → render side
...
UnblockMainThread();    // notify frameCV_; main thread starts frame N+1
Render();               // draw frame N, in parallel with prepare of N+1
```

The main thread blocks only for the length of a sync, which is usually a small fraction of a millisecond. Draw-op construction and GPU submission for frame N then run on the render thread while the main thread is already preparing frame N+1.

### 3.4 Catching Violations

A pipeline like this has one invariant: *never sync a drawable while it is being drawn.* Rosen checks this at runtime instead of hoping for it:

```cpp
// RSRenderNode::OnSync
DrawableV2::RSRenderNodeSingleDrawableLocker singleLocker(renderDrawable_.get());
if (UNLIKELY(!singleLocker.IsLocked())) {
    HILOG_COMM_ERROR("Drawable try to Sync when node %{public}" PRIu64 " onDraw!!!", GetId());
    if (RSSystemProperties::GetSingleDrawableLockerEnabled()) {
        return;   // skip this sync rather than corrupt the draw
    }
}
```

This matters because the render thread isn't the only thing drawing. UIFirst sub-threads draw window caches off the main path too, and `RSUifirstManager::CollectSkipSyncNode` lets their nodes skip a sync while they're busy. A try-lock that logs and backs off turns a rare memory-corruption crash into a log line plus a one-frame delay. That is a very good trade.

## 4. Handoff #2: Render → Composer (Don't Wait for the GPU)

Once the render thread has built the draw ops, it calls `renderFrame->Flush()`, and the GPU commands are submitted. The obvious next step would be to wait for the GPU to finish and then compose. We don't wait.

```cpp
// RSScreenRenderNodeDrawable::OnDraw (simplified)
renderFrame->Flush();
bufferGuard.SetAcquireFence(renderFrame->GetAcquireFence());
...
for (auto& drawable : hardwareDrawables) {
    processor->CreateLayerForRenderThread(*drawable);   // HWC overlay layers
}
processor->ProcessScreenSurfaceForRenderThread(*this); // the GPU-composited layer
processor->PostProcess();                              // → CommitLayers()
```

The GPU-composited screen buffer is handed to the composer together with an **acquire fence**, a kernel sync object that signals when the GPU finishes writing that buffer. The CPU never blocks on the GPU. The display hardware waits on the fence itself. From the render thread's point of view, the frame is finished as soon as the commands are flushed and the layers are committed, so it can start on the next frame while the GPU is still rasterizing this one.

This means the GPU acts as a fourth pipeline stage that we don't need a thread for. Fences handle the synchronization.

![The render thread flushes to the GPU and commits layers with an acquire fence, then moves on; the display hardware waits on the fence, and released buffers flow back to the render thread]({{ "/assets/post4_003_gpu_fences.svg" | relative_url }})

### 4.1 The Composer Thread

Layers travel to `RSRenderComposerAgent::ComposerProcess` as an `RSLayerTransactionData`, and the work is posted to a per-screen thread called `CompThread_<screenId>`:

```cpp
rsRenderComposer_->ComposerPrepare(currentRate, delayTime, pipelineParam);
rsRenderComposer_->PostDelayTask([...]() {
    renderComposerAgent->rsRenderComposer_->ComposerProcess(currentRate, transactionData);
}, delayTime);
```

`ProcessComposerFrame` then calls `SetRSLayers` → `Repaint()` (the HWC commit) → `ReleaseLayers()`, and sends the released buffers and their release fences back to the render thread.

There is one composer thread **per screen**. On a foldable, or with an external display attached, a slow commit on one panel can't stall the other.

## 5. Keeping the Pipeline Honest: Timing and Backpressure

A pipeline with no flow control just converts jank into latency, or into memory blowups. Rosen has three mechanisms that keep the stages in step.

### 5.1 Delayed Commit

The composer doesn't always commit right away. `CalculateDelayTime` works out when the frame *should* reach the display, based on the VSync timestamp, the pipeline offset (on non-LTPO panels this is `2 * period`, one period for draw and one for composition) and the expected composition time. It then delays the commit to land on that time:

```cpp
expectCommitTime = frameTimestamp - preTime + frameOffset - compositionTime - RESERVE_TIME;
delayTime = round((expectCommitTime - currTime) / NS_MS_UNIT_CONVERSION);
```

This sounds backwards: why slow down a frame that's ready early? The answer is that consistent frame **pacing** is what the eye perceives as smoothness. A frame shown early followed by a frame shown on time reads as a stutter, even if no frame was dropped. `UpdateDelayTime` also guarantees ordering: commit `n+1` is always at least `COMMIT_DELTA_TIME` after commit `n`, so frames can never overtake each other.

### 5.2 Buffer Backpressure

The render thread can only run ahead as far as there are free buffers. Before it requests a new frame buffer, it checks:

```cpp
// RSScreenRenderNodeDrawable::OnDraw
RSUniRenderThread::Instance().WaitUntilScreenNodeBufferReleased(*this);
auto renderFrame = RequestFrame(*params, processor);
```

If the screen's `BufferQueue` has no free buffer, the render thread waits on a condition variable, with a timeout, until the composer's `ReleaseLayerBuffers` call notifies it. This is the brake that stops the render thread from overwriting a buffer the display is still scanning out.

### 5.3 Telling VSync How Deep the Queue Is

`ComposerPrepare` increments `unExecuteTaskNum_`, and `ProcessComposerFrame` decrements it. That count is reported through `SetHardwareTaskNum` all the way up to the VSync distributor (DVSync). The VSync scheduler therefore knows how many composition tasks are queued, and it can hold back when the back of the pipeline is full instead of piling more frames into it.

Put together, the three mechanisms close the loop around the pipeline:

![The forward pipeline with its three feedback loops: delayed commit on the composer, buffer release waking the render thread, and queue depth reported to DVSync]({{ "/assets/post4_004_flow_control.svg" | relative_url }})

## 6. The Shortcut: Skip the Middle Stage

Sometimes stage 2 has nothing to do. A typical example is full-screen video, where the only thing changing is a buffer that HWC can display directly as an overlay. For these frames the main thread takes the **DirectComposition** path:

```cpp
needPostAndWait_ = !doDirectComposition_ && needDrawFrame_;
if (needPostAndWait_) {
    drawFrame_.PostAndWait();
}
```

When `DoDirectComposition` succeeds, the main thread sends layers straight to the composer, the render thread is never woken, and the GPU isn't touched. The conditions for this path are strict: no animation, no dirty nodes apart from HWC buffers, no capture tasks and a single display, among others. They are re-checked at several points during the frame, and any doubt sends the frame back to the full pipeline. This path is the main reason video playback stays at very low power.

![DirectComposition: when only HWC buffers changed, the main thread sends layers straight to the composer, skipping the render thread and the GPU]({{ "/assets/post4_005_direct_composition.svg" | relative_url }})

## 7. What It Costs

Pipelining isn't free.

- **Two VSync periods of added pipeline depth.** A frame now passes through each stage in turn, one period apiece, so it spends up to two extra periods between "VSync arrived" and "photons on the panel". This is the `2 * period` offset that delayed commit targets in Section 5.1. For scrolling and animations, which are the cases where jank is visible, a steady frame stream matters much more than a few milliseconds of input-to-photon latency. The delayed-commit logic and DirectComposition both reduce the cost where it matters.
- **Memory.** Every node carries a staging copy of its params and draw commands as well as the render-side copy. The swap-based sync limits the cost to *one* extra copy and never more.
- **Complexity.** Every field that the main thread writes and the render thread reads has to go through staging. Forget once, and you get a data race that shows up as a one-in-a-million flicker. The single-drawable locker and the `SYNC`/`SINGLE_THREAD` fallback modes exist because we needed them.

## 8. Results

Running performance benchmarks on our test devices, the pipeline cut the **jank rate by about 80%**. Tail frame time improved far more than average frame time, which is what the max-vs-sum argument in Section 1 predicts.

## Summary

```text
Stage 1 — RSMainThread
    Transactions, animation, dirty regions, occlusion, pruning, HWC selection.
    Writes only to staging state.

        ── Sync(): swap dirty nodes' staging → render side; move RSRenderThreadParams
        ── main thread unblocks immediately after Sync

Stage 2 — RSUniRenderThread
    Walk the drawable tree, build draw ops, Flush() to GPU.
    Never waits for the GPU: acquire fence travels with the buffer.

        ── CommitLayers(): RSLayerTransactionData to the per-screen composer

Stage 3 — RSRenderComposer (CompThread_<screenId>)
    Delay to target present time, SetRSLayers → Repaint → ReleaseLayers.
    Released buffers wake the render thread; queue depth feeds back to VSync.
```

> Serial rendering fails when the *sum* of the stages goes over budget. Pipelined rendering fails only when one stage *on its own* goes over budget.

## Appendix I, Design Choices

- **Why a separate Drawable tree instead of locking the RenderNode tree?** Locks on a shared tree serialize the threads again, and fine-grained per-node locks cost a lot when they're taken thousands of times per frame. A mirrored tree with a single, short sync point keeps the hot paths lock-free and puts every hazard into one function you can audit.

- **Why does the main thread wait for `Sync()` at all?** Because the main thread's next prepare would overwrite the staging state that the sync is reading. Waiting for sync, rather than for the whole render, is the smallest critical section that keeps that safe.

- **Why swap instead of copy?** Draw command lists can be large. A swap is O(1) no matter how big they are, and the old list, which is now in staging, is cleared or reused instead of reallocated.

- **Why sync only pending nodes?** The pending list follows the dirty tracking as described in the last [blog post](./2026-07-26-efficient-redraws-in-harmonyos-pt-3.md), so the cost of sync grows with what changed, not with the size of the tree. On a static screen, sync is close to free.

- **Why fences instead of `glFinish`/`vkQueueWaitIdle`?** Blocking the CPU on the GPU takes away the overlap that pipelining exists to create. Fences let the display hardware wait for the GPU, which it has to do anyway.

- **Why one composer thread per screen?** HWC commits for different panels are independent, and a stalled external display shouldn't drop frames on the built-in one.

- **Why delay the composer commit?** Even frame pacing matters more than raw latency for perceived smoothness. Presenting early and then on time looks like stutter.
