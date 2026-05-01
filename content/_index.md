---
---

{{< rawhtml >}}

<!-- ====================================== -->
<!--  CARD 1 — Digital Paradigm             -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Beyond Analog Metaphors</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">
        <p>
        Most creative coding tools simulate analog hardware: virtual knobs, cables, circuit boards.
        This imposes constraints that don't exist in computation.
        </p>
        </br>
        <p><strong>MayaFlux embraces true digital paradigms.</strong> Oscillators, patch cables, and envelope generators are pedagogical crutches borrowed from hardware that never constrained digital computation. MayaFlux embraces recursion, look-ahead processing, arbitrary precision, cross-domain data sharing, and computational patterns that have no analog equivalent.
        </p>
        </br>
        <p>
    Polynomials sculpt data. Logic gates make creative decisions. Coroutines coordinate time itself.</p>
    </div>

</div>

<!-- ====================================== -->
<!--  CARD 2 — What Makes MayaFlux Different -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>What Makes MayaFlux Different</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <div class="feature-grid">
            <div class="feature-card">
                <h3>Unified Data Streams</h3>
                <p>Audio, visual, and control signals share the same numerical substrate. A node output routes to an RtAudio callback and a Vulkan push constant simultaneously, with no translation layer.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Nexus: Spatial Entity Lifecycle</h3>
                <p>Fabric, Wiring, Emitter, Sensor, Agent. A spatial computation layer where entities perceive and influence audio and graphics simultaneously. Not a scene graph. No update loop. No component system.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Granular Synthesis as Data Analysis</h3>
                <p>Recordings decompose into populations of named, attributed grains. Sort by spectral centroid. Filter by variance. Reorder by any function you can express in code. The composition is an explicit analytical argument about the material, not a phasor and a scatter parameter.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Portal::Text</h3>
                <p>A glyph is a quad. A quad is four numbers. Four numbers can go anywhere. Text renders as a texture driven by the same node graph as audio and geometry: per-glyph oscillators, physics, GPU bindings.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Physical Modelling Networks</h3>
                <p>ModalNetwork, WaveguideNetwork, and ResonatorNetwork implement physical modelling synthesis as first-class node graph citizens. Excitation, coupling, boundary conditions, and spatial routing are all live-configurable parameters.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Cross-Modal Node Bindings</h3>
                <p>Node outputs bind directly to GPU shader parameters. Audio envelopes, spectral data, and control signals reach the GPU through the same node API, with no bridging code.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Live Coding with Lila</h3>
                <p>An embedded Clang interpreter evaluates arbitrary C++23 at runtime via LLVM ORC JIT. Latency is one buffer cycle. Algorithms change without stopping audio or tearing down the graphics context. Used in production during a 20-minute live set on a Steam Deck.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Coroutine Temporal Control</h3>
                <p>C++20 coroutines are the scheduling primitive. SampleDelay, FrameDelay, and MultiRateDelay awaiters let temporal intent cross domain boundaries. Time is compositional material, not a callback interval.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Lock-Free Architecture</h3>
                <p>No mutexes in the real-time path. atomic_ref, CAS-based dispatch, and lock-free registration lists coordinate all four execution contexts: RtAudio callbacks, Vulkan render threads, async input backends, and user coroutines.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Live Signal Matrix</h3>
                <p>IOManager provides a unified interface across the full signal matrix: audio files, video files, live camera devices, and image assets. SamplingPipeline adds polyphonic multi-cursor playback with independent speed and looping per voice.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>MIDI and HID Input</h3>
                <p>RtMidi and HIDAPI backends feed an async InputManager that dispatches to InputNode instances in the node graph. Controllers, sensors, and custom HID devices become signal sources with the same routing API as any other node.</p>
            </div>
        </div>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 3 — Philosophy                   -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Philosophy</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <div class="philosophy-points">

            <div class="philosophy-point">
                <h3>Domain is decided last</h3>
                <p>Sound, visuals, control signals are all numbers. A camera is a position and two matrices. A light is a position and a falloff function. A voice is a cursor into a buffer. Domain vocabulary describes what something is <em>used for</em>, not what it <em>is</em> computationally. MayaFlux does not enshrine that vocabulary in its type system.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Same structure, different outputs</h3>
                <p>A gravitational attractor and a reverb send are the same computational structure pointed at different outputs. MayaFlux has no light type, no force type, no camera type. It has positioned entities, functions that fire, and outputs that route to audio, geometry, or GPU descriptors. What the entity <em>is</em> in domain terms is entirely the user's imagination applied to pure math.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Don't name what doesn't need a name</h3>
                <p>Vulkan does not have a camera. It has a push constant slot. If you put a matrix there, the vertex shader reads it. MayaFlux exposes this as-is. Not naming it is not a missing feature. It is a deliberate refusal to add abstraction that adds nothing but the illusion of safety.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Code as Creative Material</h3>
                <p>Data transformation is the creative act. Programming is compositional structure, not plumbing. When tools protect you from complexity, they also protect you from possibility.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Time as Structure</h3>
                <p>Temporal relationships are part of artistic expression, not implementation detail. Coroutines are the scheduling primitive. Time is material.</p>
            </div>

        </div>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 4 — Built From Necessity         -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Built From Necessity</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p>
        MayaFlux wasn't built to improve on existing tools. It was built because they could not support the work already happening.
        </p>
        <ul>
            <li><strong>15+ years of interdisciplinary performance</strong> across Chennai, Delhi, and the Netherlands</li>
            <li><strong>Production audio engineering</strong> for Unreal Engine 5 and Metro: Awakening VR</li>
            <li><strong>Experimental creative computing education</strong></li>
            <li><strong>Live performance under pressure:</strong> 0.3 was developed in parallel with a 20-minute live coding set on a Steam Deck at TOPLAP Bengaluru. Four pieces. The framework and the performance were simultaneous.</li>
        </ul>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 5 — Current Status               -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Current Status</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p><strong>Version 0.3.0, April 2026. Alpha. Stable core.</strong></p>
        <hr>
        <p><strong>Stable:</strong> Audio processing, lock-free node graphs, coroutine scheduler, channel routing with crossfade transitions.</p>
        <hr>
        <p><strong>Stable:</strong> Vulkan 1.3 dynamic rendering, multi-window, geometry nodes, texture pipeline, compute shaders, GPU readback.</p>
        <hr>
        <p><strong>Stable:</strong> Nexus spatial entity layer: Fabric, Wiring, Emitter, Sensor, Agent, SpatialIndex3D with lock-free snapshot publication.</p>
        <hr>
        <p><strong>Stable:</strong> Granular synthesis with offline grain attribution, SamplingPipeline for polyphonic multi-cursor playback.</p>
        <hr>
        <p><strong>Stable:</strong> Portal::Text: FreeType glyph rendering as GPU texture, driven by the same node graph as audio and geometry.</p>
        <hr>
        <p><strong>Stable:</strong> Live camera input (Linux, macOS, Windows), video file playback, image loading via IOManager.</p>
        <hr>
        <p><strong>Stable:</strong> MIDI and HID input backends, async dispatch, InputNode infrastructure.</p>
        <hr>
        <p><strong>Stable:</strong> Lila, LLVM ORC JIT for live C++23 evaluation at sub-buffer latency.</p>
        <hr>
        <p><strong>Next (0.4):</strong> Nexus and agent build-out. Kinesis as first-class analysis namespace: Eigen-level linear algebra, FluCoMa-level audio analysis primitives.</p>
        <hr>
        <p><strong>Planned (0.5):</strong> Native UI framework, building on existing Region and WindowContainer infrastructure.</p>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 6 — Call to Action               -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Ready to Explore?</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p>
        MayaFlux is for creators who've outgrown callback-driven thinking and want unified streams across audio, visual, and algorithmic composition.
        </p>

        <ul>
        <li><a href="/download/">Download Weave</a></li>
        <li><a href="https://mayaflux.github.io/MayaFlux/">Documentation</a></li>
        <li><a href="https://github.com/MayaFlux/MayaFlux">Source Code</a></li>
        <li><a href="/releases/">Release Notes</a></li>
        <li><a href="/posters/cpponline-2026/">C++ Online conference poster</a></li>
        </ul>

    </div>

</div>

{{< /rawhtml >}}
