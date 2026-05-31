---
layout: post
title: "Uncovering the UI Framework of HarmonyOS, Part 2: ArkUI Declarative UI Engine"
date: 2026-05-15 10:30:00 +0800
categories: harmonyos arkui declarative-ui
---

After exploring the Rosen Render Service in [Part 1](/2026-05-14-uncovering-ui-framework-of-harmonyos-pt-1), we now move one layer up the stack to the declarative UI framework that developers actually use every day: **ArkUI**.

The engine behind it lives in the [arkui_ace_engine](https://gitcode.com/openharmony/arkui_ace_engine) repository (also mirrored on GitHub under openharmony).

## What makes it special

The declarative layer in ArkUI is deliberately **very thin**. High-level ArkTS (or TypeScript/JavaScript) syntax is transpiled at build time into imperative code that drives a small set of native hooks. The overwhelming majority of work—component creation, tree management, layout, rendering, property updates, and event handling—executes in highly optimized C++.

This is a conscious performance decision. Even with modern JavaScript engines and JIT/AOT compilation, staying in native code for the hot paths (especially per-frame work) is dramatically more efficient. In cross-platform scenarios on iOS, ArkUI's component creation and layout times have measured around 10% of the cost of equivalent native SwiftUI work in some benchmarks. The gap comes from avoiding JS object churn, direct bridging into C++, and aggressive dirty-flag-based incremental updates.

### Before transpilation (what you write)

```arkts
Column() {
    Button("Click me") {
        // child content
    }
    .onClick(() => { ... })
    .width('100%')
    .aspectRatio(2.0)
}
```

### After transpilation (what actually runs)

The ArkTS compiler transforms the declarative syntax into imperative calls wrapped in `observeComponentCreation` (or the optimized `observeComponentCreation2`) closures. These closures interact with `ViewStackProcessor` for ID allocation, dependency tracking, and stack-based tree construction.

Here's a simplified illustration of the generated shape:

```ts
this.observeComponentCreation((elmtId, isInitialRender) => {
    ViewStackProcessor.StartGetAccessRecordingFor(elmtId);

    Column.create();
    // width / aspectRatio applied via ModelNG or ModifierWithKey path

    this.observeComponentCreation((childElmtId) => {
        ViewStackProcessor.StartGetAccessRecordingFor(childElmtId);
        Button.createWithLabel("Click me");
        Button.onClick(() => { ... });
        Button.pop();
        ViewStackProcessor.StopGetAccessRecording();
    }, Button);

    Column.pop();
    ViewStackProcessor.StopGetAccessRecording();
}, Column);
```

![ArkUI declarative transpilation pipeline and ViewStackProcessor bridge]({{ "/assets/transpilation_diagram.png" | relative_url }})

*Figure 1: From author-written ArkTS to transpiled `observeComponentCreation` code that drives `ViewStackProcessor` and builds the native `FrameNode` tree.*

There is no runtime interpretation of the nice declarative syntax you wrote. Every component creation flows through a tiny, well-defined bridge into C++.

### Tree formation with ViewStackProcessor

`ViewStackProcessor` (see `frameworks/core/components_ng/base/view_stack_processor.h`) is the central singleton that owns the construction stack while declarative code executes:

- `GetInstance()`
- `StartGetAccessRecordingFor(elmtId)` / `StopGetAccessRecording()` — records reads of state variables against the current element ID for fine-grained reactivity.
- `Push(node)`, `Pop()`, `PopContainer()`, `Finish()` — builds the parent-child relationships.
- `GetMainFrameNode()` — the node that property setters (via `ViewAbstract` and the various `ModelNG` classes) target.
- Support for element IDs, keys (used by ForEach), visual states, implicit animations, and prebuild commands.

![ViewStackProcessor stack-based tree construction]({{ "/assets/view_stack_processor.png" | relative_url }})

*Figure 2: ViewStackProcessor maintains a construction stack that mirrors the nesting in your declarative code.*

When your component's `build()` (or equivalent) runs—whether on initial render or a targeted re-execution after a state change—it pushes nodes onto this stack. The pop/finish operations mount them into the real `UINode` / `FrameNode` tree and mark the work complete.

This mechanism is the direct counterpart to Jetpack Compose's slot table / Composer or SwiftUI's view builder internals, but wired straight into ArkUI's retained C++ node tree.

### Modules that remain (mostly) in the JS/ArkTS world

While the heavy lifting lives in C++, a few pieces stay in the higher-level language for developer experience and reactivity:

- **Decorators and state** (`@Component`, `@State`, `@Link`, `@Builder`, `@Prop`, etc.)

  The syntactic transformation and reactivity system are primarily the responsibility of the ArkTS compiler and the state management runtime. At execution time the engine only sees the `observeComponentCreation*` hooks plus access recording through `ViewStackProcessor`.

  When a `@State` variable mutates, the state layer re-invokes only the affected observation closures. Inside those closures the element IDs and `ViewStackProcessor` allow the framework to reuse existing `FrameNode` instances instead of throwing everything away.

  `JSDataChangeListener` and `ElementRegister` provide the native-side hooks that make data-driven updates (ForEach, LazyForEach, etc.) efficient.

- **Example: `@Component` + `@State` (conceptual)**

  ```arkts
  @Component
  struct MyCard {
      @State count: number = 0;

      build() {
          Column() {
              Text(`Count: ${this.count}`)
              Button("Inc").onClick(() => this.count++)
          }
      }
  }
  ```

  The compiler turns the body of `build()` into an `observeComponentCreation` closure. Reads of `this.count` during the recording phase register the dependency against the current `elmtId`. Later mutation triggers a minimal re-execution of just that closure.

![@State reactivity and targeted re-execution flow in ArkUI]({{ "/assets/state_variable_flow.png" | relative_url }})

*Figure 3: When a @State variable mutates, only the affected component closures are re-executed using element IDs for efficient node reuse.*

- **LazyForEach** (virtualized / lazy lists)

  This has first-class support on both sides of the bridge (`js_lazy_foreach.cpp` in the declarative frontend and `LazyForEachNode` + `LazyForEachBuilder` in `frameworks/core/components_ng/syntax/`).

  It uses the `FrameProxy` virtualization mechanism inside `FrameNode` for on-demand creation, caching, and eviction of children based on the visible range. The ArkTS side supplies the data and item builder; the C++ side owns the recycling and diffing. This is a great example of deliberately keeping generation logic close to the developer while the performance-critical tree management stays native.

Another important piece that stays lightweight in JavaScript is the `ModifierWithKey` system (implemented in `arkComponent.js` and bridged via `arkUINativeModule`):

```ts
class WidthModifier extends ModifierWithKey {
  applyPeer(node, reset) {
    if (reset) {
      getUINativeModule().common.resetWidth(node);
    } else {
      getUINativeModule().common.setWidth(node, this.value);
    }
  }
  checkObjectDiff() { /* ... */ }
}
```

This design allows cheap structural diffing on the JS side for state-driven updates before any native call is made.

### Cross platform by design

The architecture was built with multiple frontends and backends in mind:

- **Frontends**: The main declarative path (ArkTS/TS/JS via the `declarative_frontend/` bridge and JSI), experimental ArkTS variants, Cangjie, and a full **C/C++ Native API** surface (`OH_ArkUI_*` functions + the Arkoala API).
- **Rendering backends**: The primary backend is Rosen (`RosenRenderContext` wrapping `RSNode`). The design keeps a clean abstraction layer (there are also references to Skia in the broader ecosystem).
- **Platform adapters**: `adapter/ohos/`, a previewer mode, and the cross-platform ports that allow the same core to run on Android and iOS.

All paths—whether you write ArkTS with `@Component` or call the native C API directly—converge on the same `FrameNode` + `Pattern` + `LayoutProperty`/`PaintProperty` core, and ultimately the same `PipelineContext` and render backend.

This is why ArkUI can deliver a consistent, high-performance declarative model across HarmonyOS devices, Android, and iOS from a single C++ implementation.

## Summary

ArkUI's declarative engine achieves its performance by:

- Making the JS/ArkTS layer a thin transpiled bridge.
- Driving almost everything through `ViewStackProcessor` + `ModelNG` into native `FrameNode` / `Pattern` objects.
- Using element IDs + access recording for precise, incremental state updates.
- Keeping only the developer-facing ergonomics (decorators, builders, lazy item generation) in the higher-level language.
- Maintaining a clean multi-frontend / multi-backend architecture from day one.

The result is a system that feels like a modern declarative UI framework while executing with the efficiency of a retained-mode native engine.

In future parts we'll go deeper into the `FrameNode` + `Pattern` composition model, the dirty propagation and layout/paint pipeline, how LazyForEach and virtual scrolling actually work at the node level, and the native C API surface.

---

*Sources for the details in this post include the arkui_ace_engine repository (especially `view_stack_processor.h`, `frame_node.cpp`, `pattern.h`, `arkComponent.js`, the declarative frontend bridges, and `LazyForEach*` implementations) together with the excellent auto-generated architecture documentation on DeepWiki.*