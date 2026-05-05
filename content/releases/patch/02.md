---
title: "Patch Notes 0.2.1"
summary: "v0.2.1: Correctness fixes in physical modelling audio buffer synchronisation, descriptor binding source dispatch, and SoundFileBridge initialisation."
---

<div class="card wide">

## MayaFlux 0.2.1: Physical Modelling and Descriptor Binding Fixes

**20th March 2026**

MayaFlux 0.2.1 is a patch release targeting two correctness bugs in the physical modelling and descriptor binding subsystems, plus a missing initialisation call in `SoundFileBridge`.

No API changes. Drop-in replacement for 0.2.0.

</div>

<div class="card group">

<div class="card">

## Descriptor Binding Source Dispatch Crash

A missing `break` after the `NODE` source type case in `DescriptorBindingsProcessor::update_descriptor_from_node` caused fallthrough into the `AUDIO_BUFFER` branch. Any binding with a node source would attempt a `dynamic_pointer_cast` on a null buffer and crash.

The function has been refactored into a dispatcher with three focused handlers: `update_from_node`, `update_from_buffer`, and `update_from_network`. Each handler validates its input on entry before proceeding.

</div>

<div class="card">

## NodeNetwork Audio Buffer Data Race

`m_last_audio_buffer` in `ModalNetwork`, `WaveguideNetwork`, and `ResonatorNetwork` was filled incrementally via `push_back` with no synchronisation between the audio thread writing and the graphics thread reading via `get_audio_buffer`. The read window was proportional to fill duration, making torn reads probable under normal async operation.

Each `process_batch` override now fills into a `thread_local` scratch vector, then acquires an `atomic_flag` spinlock to `assign` into `m_last_audio_buffer` and run `apply_output_scale` before releasing. `get_audio_buffer` acquires the same lock before copying. The lock is held only for the assign and scale pass, not during the fill. The `clear` + `reserve` + `push_back` pattern is replaced with `assign` throughout, eliminating reallocation risk on the shared buffer.

</div>

<div class="card">

## SoundFileBridge Incomplete Initialisation

`SoundFileBridge::setup_processors` was not calling `initialize()` before constructing `SoundStreamWriter`, leaving the bridge in an incomplete state. The missing call is inserted.

</div>

<div class="card">

## BufferOperation ROUTE Semantics

`route_to_buffer` and `route_to_container` now document that `ROUTE` is a batch operation for post-accumulation output, not live per-cycle routing. Live routing is handled by `supply`, `clone`, and per-buffer processing chains.

</div>

<div class="card">

## Availability

- Platforms: Windows, macOS, Linux
- Distribution: GitHub releases, Launchpad PPA, COPR, AUR
- Version: 0.2.1

No migration steps required.

</div>

<div class="card wide">

## Commit Summary

<p>See full diff: <a href="https://github.com/MayaFlux/MayaFlux/compare/v0.2.0...v0.2.1">Github</a></p>

<pre><code>a7130a00 (HEAD -> support/v0.2, origin/support/v0.2) prepare for 0.2.1 patch release
f37f9a13 fix(kriya,buffers): call initialize() in SoundFileBridge::setup_processors; clarify ROUTE batch semantics
2c41437d fix(NodeNetwork): spinlock-guard m_last_audio_buffer for async graphics reads
7e104b9c fix(DescriptorBindingsProcessor): add missing break in NODE source type dispatch
</code></pre>

</div>

</div>
