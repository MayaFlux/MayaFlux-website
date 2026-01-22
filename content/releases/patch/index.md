---
title: "Patch Notes 0.1.1 and 0.1.2"
summary: "v0.1.1: Maintenance and corrective releases focusing on stability, determinism, and API correctness. v0.1.2: Breaking change to buffer semantics, stochastic performance improvements, and shader pipeline fixes."
---

<div class="card wide">

## MayaFlux 0.1.1: Stability, Determinism, and Platform Hardening

**18th January 2026**

<p>
MayaFlux 0.1.1 is a maintenance release focused on stability, correctness, and platform robustness.
There are no new user-facing features and no API changes.
</p>

<p>
This release tightens behavior introduced in 0.1.0, with particular attention to shutdown safety, live coding reliability, and cross-platform edge cases.
</p>
</div>

<div class="card group">
<div class="card">

## Scope

<ul>
<li>Bug fixes</li>
<li>Internal refactors</li>
<li>Performance and allocation improvements</li>
<li>Platform-specific robustness</li>
<li>Build and packaging cleanup</li>
</ul>

<p>
Existing projects built against 0.1.0 should continue to compile and run without modification.
However, the new internal API around <code>::Await()</code> is not included in the old <code>main.cpp</code>.
Add that single line manually or simply create a new project from the 0.1.1 template
and move your existing classes, files and resources over.
</p>

</div>

<div class="card">

## Shutdown and Lifecycle Safety

<p>
Engine shutdown has been made deterministic and race-safe across audio, graphics, node networks, and coroutines.
</p>

<ul>
<li>Audio callbacks now terminate in a defined order</li>
<li>Node processing and coroutine execution are guarded against late execution during teardown</li>
<li>Graphics resources are released in a strictly ordered sequence</li>
<li>Cross-domain coordination is prevented from executing after shutdown begins</li>
</ul>

<p>
These changes address rare but real issues observed during live coding, repeated start/stop cycles, and application exit under load.
</p>

</div>

<div class="card">

## macOS Stability Improvements

<p>
Several fixes address macOS-specific lifecycle and threading constraints:
</p>

<ul>
<li>All GLFW and Cocoa interactions are dispatched on the main thread</li>
<li>Lila live coding server and Clang interpreter execution are enforced on the main thread</li>
<li>Vulkan and MoltenVK initialization and teardown paths are hardened</li>
<li>Temporary Homebrew runtime environment handling is formalized</li>
<li>Window teardown and resizing edge cases are stabilized</li>
</ul>

<p>
These changes improve reliability during live coding and repeated execution on macOS.
</p>

</div>

<div class="card">

## Performance and Allocation Cleanup

<p>
Internal refactors reduce unnecessary work and improve predictability:
</p>

<ul>
<li>Persistent <code>NodeContext</code> reuse removes per-frame context allocation</li>
<li>GPU sync and output nodes now use persistent graphics contexts</li>
<li>Event loop dispatch is eliminated for <code>NodeNetwork</code> nodes where unnecessary</li>
<li>Buffer management and stochastic generation are tightened</li>
</ul>

<p>
No observable behavior changes are intended.
</p>

</div>

<div class="card">

## Execution and Dispatch

<ul>
<li>Introduction of a new parallel dispatch executor</li>
<li>Cleaner separation between execution coordination and domain logic</li>
<li>Improved scheduling behavior under mixed audio and graphics workloads</li>
</ul>

</div>

<div class="card">

## Build and Packaging

<ul>
<li>Windows build script and versioning fixes</li>
<li>Debian and Ubuntu packaging cleanup</li>
<li>Minor cross-platform build improvements</li>
</ul>

</div>

<div class="card">

## Availability

<ul>
<li>Platforms: Windows, macOS, Linux</li>
<li>Distribution: GitHub releases, Launchpad PPA, COPR, AUR, Homebrew</li>
<li>Version: 0.1.1</li>
</ul>

<p>
No migration steps are required.
</p>

</div>

<div class="card wide">

## Commit Summary

<p>
See full diff: <a href="https://github.com/MayaFlux/MayaFlux/compare/v0.1.0...v0.1.1">Github</a>
</p>

<pre><code>
56d505b41 (github/develop, develop) bump version for release
3053209e0 update(windows): Build scripts and versioning
ba55ef5d0 (origin/develop, codeberg/develop) fix(build): Various minor build improvements
f486e35ba fix(debian): Simplify changelog entry for Ubuntu packaging
3e867ccde (github/backport/clean_shutdown, backport/clean_shutdown) fix(macos): avoid main-loop teardown issues during window shutdown
a703c865e fix(shutdown): make audio callbacks and engine termination fully race-safe
326ea7b8b fix(shutdown): tighten node and coroutine termination ordering and guards
2b094b675 fix(audio): make engine shutdown audio-safe and deterministic
813588ebf fix(graphics): harden shutdown order and resource teardown across graphics stack
4033ddfe9 (backport/performance) fix(Various): Improve buffer management and stochastic generation
a6a3ee9e5 refactor(graphics): add persistent contexts for GPU sync/output nodes
3c8e97b15 refactor(nodes): eliminate context allocations and reuse persistent NodeContext
eedb7cf38 refactor(Nodes): Do not fire event loop for NodeNetwork nodes
84f1a0d48 (backport/cocoa_live_coding) fix(macos): run Lila server and Clang interpreter on main thread
802d20d3f update(macos): Temporary workaround for homebrew runtime dirs
40136e3d0 (backport/moltenvk_cocoa) fix(macos): Use homebrew like env file
f8a5a5291 refactor(macos): dev script simplification
b6601f8cb refactor(macos): move Await into Engine-owned main thread loop
95c145daa chore(updates): Minor mac compatibility improvements
22a47e3d3 feat(Executor): New Parallel dispatch class
0274a57d6 fix(macos): Temporarily disable resizing
024c5841b refactor(macos): dispatch all glfw calls to main - Pass II
576b100a8 fix(macos): Minor improvements to Vulkan backend on macOS
4f8798726 feat(API): Add Await function for main thread blocking
c54375e4d fix(glfw): Try main thread diversion on macOS
8c10cc0ed fix(macos): Update required Vulkan extensions for MoltenVK
d729ef4e2 chore: bump version to 0.1.1
</code></pre>

</div>
</div>

---

<div class="card wide">

## MayaFlux 0.1.2: Buffer Semantics Correction, Deterministic Stochastic Processing, and Shader Fixes

**Released shortly after 0.1.1**

MayaFlux 0.1.2 is a corrective release addressing a necessary API break, a critical Vulkan pipeline bug, and several internal performance and determinism improvements.

This release intentionally lands early to establish correct long-term semantics before downstream code solidifies around incorrect assumptions.

</div>

<div class="card group">
<div class="card">

## Breaking Change: Explicit Temporal Buffer Semantics (`std::span`)

**Affects:**

- `Polynomial` lambdas (recursive and feedforward modes)
- `Logic` sequential mode lambdas
- Related context accessors, tests, and tutorials

**What changed**

- Lambda parameters previously typed as `std::deque<T>` are now `std::span<T>`
- Internal history storage is now implemented as explicit ring buffers
- Lambdas receive a linearized temporal view rather than a mutable container

**Why**

- Removes implicit allocation and push/pop semantics
- Makes temporal ordering explicit and deterministic
- Decouples user logic from internal storage representation
- Enables zero-allocation, real-time–safe execution
- Aligns CPU and GPU contexts under a unified data contract

This is a deliberate semantic correction, not a cosmetic refactor.

</div>

<div class="card">

## Logic & Polynomial Internals

- Introduced explicit ring buffers with linearized views
- Removed implicit buffer construction paths
- Unified context propagation across CPU and GPU execution
- Preserved observable behavior while making time semantics explicit

These changes make recursive systems, filters, and sequential logic predictable and inspectable.

</div>

<div class="card">

## Stochastic Generator Performance & Determinism

- Added fast xorshift-based uniform RNG for hot paths
- Cached normal and exponential distributions
- Introduced lazy rebuilds via dirty flags on parameter changes
- Eliminated per-sample `std::distribution` reconstruction
- Made `set_normal_spread()` safely invalidate cached distributions

These changes significantly reduce overhead in stochastic-heavy graphs while preserving statistical correctness.

</div>

<div class="card">

## Shader Pipeline Fix: Push Constant Size Calculation

**Fixed**

- Pipelines could be created with zero-byte push constant ranges when push constants were set via `set_push_constant_data()` instead of staging buffers

**Resolution**

- Pipeline initialization now:
  1. Uses staging buffer size when available
  2. Falls back to the processor’s internal push constant data size
- Ensures correct allocation regardless of data source

**Affected**

- `RenderProcessor`
- `ComputeProcessor`

This fixes a silent but critical Vulkan correctness issue.

</div>

<div class="card">

## ShaderProcessor Consistency

- `set_push_constant_data()` now updates push constant size when required
- Prevents mismatches between stored data and pipeline layout
- Explicitly enforces Vulkan push constant limits

</div>

<div class="card">

## Build & Portability

- Added external GLSL → SPIR-V compilation fallback when Shaderc is not enabled
- Improves portability and minimal build configurations

</div>

<div class="card">

## Documentation

- README updated to reflect correct shutdown and main-thread flow using `MayaFlux::Await()`

</div>

<div class="card wide">

## Migration Notes (Important)

If you have lambdas like:

```cpp
[](const std::deque<double>& buffer) { ... }
```

Update them to:

```cpp
[](std::span<double> buffer) { ... }
```

No other logic changes should be required.

</div>

<div class="card wide">

## Commit Summary

<pre><code>
3c08747d5 bump release version to 0.1.2
5683d9caf build: introduce external GLSL→SPIR-V compiler fallback
08a94cdf6 perf: optimize stochastic generator with cached distributions and fast RNG
72871a380 refactor: replace deque-based history with span-backed ring buffers
66c29a52e fix(shaders): update push constant size when setting data
7dccd8216 fix(shaders): correct push constant size calculation
</code></pre>

</div>
</div>
