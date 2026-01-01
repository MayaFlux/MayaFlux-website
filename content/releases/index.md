---
title: "MayaFlux 0.1.0: Infrastructure for Digital Creativity"
---

**January 1, 2026**

MayaFlux 0.1.0 is now available. This is not another creative coding framework. This is computational infrastructure built from 15 years of interdisciplinary performance practice, pedagogy, research and production dsp engineering.
It treats audio, visual, and control data as unified numerical streams processed through lock-free computation graphs, C++20 coroutines for temporal coordination, and complete LLVM JIT compilation for live coding.

<div class="card-grid columns breakout">

<div class="card vertical">
<h2> What MayaFlux Is </h2>

MayaFlux is a C++20/23 multimedia computation framework that rejects some fundamental assumptions of existing tools.
Three design principles distinguish it from existing tools:

- **Analog synthesis metaphors have no place in digital systems.** Oscillators, patch cables, and envelope generators are pedagogical crutches borrowed from hardware that never constrained digital computation. MayaFlux embraces recursion, look-ahead processing, arbitrary precision, cross-domain data sharing, and computational patterns that have no analog equivalent. Polynomials sculpt data. Logic gates make creative decisions. Coroutines coordinate time itself.

- **Audio, visual, and control processing are artificially separated.** These boundaries exist for commercial tool design, not computational necessity. In MayaFlux, a single unit outputs to audio channels, triggers GPU compute shaders, and coordinates temporal events simultaneously, in the same processing callback. Data flows between domains without conversion overhead because samples, pixels, and parameters are all double-precision floating point.

- **Tools that hide complexity also hide possibility.** MayaFlux provides hooks everywhere. Replace the audio callback. Intercept buffer processing. Override channel coordination. Substitute backends. Every layer is replaceable, every system is overridable. If you understand the implications, you can modify anything. If you don't, the documentation teaches you through working code examples that produce real sound within minutes.
</div>

<div class="card vertical">
<h2>What MayaFlux Is Not</h2>

MayaFlux is infrastructure, not application software:

**Not a DAW.** No timeline editor, no MIDI piano roll, no plugin hosting. MayaFlux provides computational substrate. Build your own sequencing logic.

**Not a node-based UI.** No visual patching interface. Everything is C++ code. Text is more precise for complex logic. Your patches are version-controlled source files, not opaque binaries.

**Not consumption-oriented software**
MayaFlux assumes an active relationship with computation. It rewards curiosity, modification, and experimentation rather than menu navigation or preset selection.
Users do not need deep systems programming knowledge, but they do need a willingness to think in terms of data, processes, and structure. The framework teaches these ideas through fluent APIs and runnable examples.

**Not a replacement for Max/P5.js yet.** Those tools excel at rapid prototyping and visual exploration. MayaFlux excels at computational depth and architectural control. Eventually, yes. Right now, different purposes.

</div>
</div>

<hr/>

<div class="card vertical">
<h2> Who Should Use This </h2>

<details>
<summary><strong>Creative technologists hitting tool limits.</strong></summary>
            If you've prototyped in Processing but need real-time audio, mastered Max/MSP but want programmatic control, or built installations in openFrameworks then watched Apple deprecate OpenGL, MayaFlux is infrastructure built from frustration with those limitations.
</details>

<details>
<summary><strong>Visual artists and installation makers needing computational depth.</strong></summary>
            If you've built generative visuals in Processing or TouchDesigner but want low-level GPU control without OpenGL's deprecated patterns, or need audio and visuals truly synchronized rather than awkwardly bridged, MayaFlux treats graphics processing with the same architectural rigor as audio DSP.
</details>

<details>
<summary><strong>Researchers needing genuine flexibility.</strong></summary>
            Academic audio research shouldn't require fighting commercial tools to implement novel algorithms. MayaFlux provides direct buffer access, arbitrary processing rates, cross-domain coordination. Research shouldn't require reverse-engineering closed systems.
</details>

<details>
<summary><strong>Musicians and composers outgrowing presets.</strong></summary>
            If you've exhausted existing tools and want instruments matching your musical imagination rather than vendor roadmaps, MayaFlux treats synthesis as data transformation you control at every sample.
</details>

<details>
<summary><strong>Developers escaping framework constraints.</strong></summary>
            Game audio middleware, creative coding libraries, visual programming environments all impose architectural boundaries. MayaFlux removes them while maintaining performance guarantees through lock-free coordination and deterministic processing.
</details>

</div>

<hr/>

<div class="card">
<h2>The Technical Reality</h2>

<div class="card">
<h3>Lock-Free Concurrent Processing</h3>

Every node, buffer, and network uses lock-free atomic coordination through C++20's `atomic_ref`, compare-exchange operations, and explicit memory ordering. You add oscillators, connect filters, restructure graphs while audio plays with no glitches, no dropouts, no mutex contention. Maximum latency for any modification: one buffer cycle (typically 10-20ms).

This isn't careful, sparse or scoped locking. It's no locks in the processing path. Pending operations queue atomically. Channel coordination uses bitmask CAS patterns. Cross-domain transfers happen through processing handles with token validation.

</div>

<hr/>
<div class="card">
<h3>The State Promise</h3>

Computational units process exactly once per cycle regardless of consumers. A spectral transform feeding both granular synthesis and texture generation processes once, both domains receive synchronized output. Atomic state flags prevent reprocessing. Reference counting coordinates reset. Channel bitmasks handle multi-output scenarios.

This eliminates the traditional separation between audio rate and control rate. Rate is just a processing token (`AUDIO_RATE`, `VISUAL_RATE`, `CUSTOM_RATE`) that tells the engine calling frequency.

</div>

<hr/>
<div class="card">
<h3>Unified Data Primitives</h3>

Audio samples are numbers. Pixel values are numbers. Control parameters are numbers. Time is numbers. No conversion overhead. No semantic boundaries. A visual analysis routine directly modulates synthesis parameters. A recursive audio filter drives texture coordinates. The same `RootBuffer` pattern works for `RootAudioBuffer` and `RootGraphicsBuffer`.

</div>

<hr/>
<div class="card">
<h3>Graphics as First-Class Computation</h3>

Vulkan integration isn't an afterthought or "audio visualization". The graphics pipeline runs on identical infrastructure: lock-free buffer coordination, token-based domain composition, unified data flow. Particle systems, geometry generation, shader bindings, all use the same Node/Buffer/Processor architecture as audio. A polynomial node can drive vertex displacement as naturally as it drives waveshaping. This is computation substrate, not an audio library with graphics bolted on.

</div>

<hr/>
<div class="card">
<h3>C++20 Coroutines as Temporal Material</h3>

Time becomes compositional material through first-class coroutine support:

```cpp
auto sync_routine = [](Vruta::TaskScheduler& scheduler) -> Vruta::SoundRoutine {
    while (true) {
        co_await Kriya::SampleDelay{scheduler.seconds_to_samples(0.02)};
        process_audio_frame();

        co_await Kriya::MultiRateDelay{
            .samples_to_wait = scheduler.seconds_to_samples(0.1),
            .frames_to_wait = 6
        };
        bind_push_constants(some_audio_matrix)
    }
};
```

Coroutines suspend on sample counts, buffer boundaries, or arbitrary predicates. Temporal logic reads like the musical idea. No callback hell. No message-passing complexity.

</div>

<hr/>
<div class="card">
<h3>The Lila JIT System</h3>

MayaFlux includes complete LLVM-based JIT compilation. Write C++ code, hit evaluate, hear/see results within one buffer cycle. No compilation step. No application restart. No workflow interruption. Full C++20 syntax, template metaprogramming, compile-time evaluation. Live coding shouldn't mean switching to a simpler language.

</div>

</div>

<hr/>

<div class="card">
<h2>The Architecture</h2>

Processing happens through **Domains** combining Node tokens (rate), Buffer tokens (backend), and Task tokens (coordination):

```cpp
Domain::AUDIO = Nodes::ProcessingToken::AUDIO_RATE +
                Buffers::ProcessingToken::AUDIO_BACKEND +
                Vruta::ProcessingToken::SAMPLE_ACCURATE

Domain::GRAPHICS = Nodes::ProcessingToken::VISUAL_RATE +
                   Buffers::ProcessingToken::GRAPHICS_BACKEND +
                   Vruta::ProcessingToken::FRAME_ACCURATE

// Custom user example
Domain::PARALLEL = Nodes::ProcessingToken::AUDIO_RATE +
                   Buffers::ProcessingToken::AUDIO_PARALLEL + // Executes on the GPU
                   Vruta::ProcessingToken::SAMPLE_ACCURATE
```

Custom domains compose individual tokens for specialized requirements. Cross-modal coordination happens naturally through token compatibility rules enforced at registration, not during hot path execution.

Buffers own processing chains. Chains execute processors sequentially. Processors transform data through mathematical expressions, logic operations, or custom functions. Everything composes:

```cpp
void compose() {
    auto sine = vega.Sine(440.0);
    auto node_buffer = vega.NodeBuffer(0, 512, sine)[0] | Audio;

    auto distortion = vega.Polynomial([](double x) { return std::tanh(x * 2.0); });
    MayaFlux::create_processor<PolynomialProcessor>(node_buffer, distortion);
}
```

The substrate doesn't change. Your access to it deepens.

</div>

<h2>What's Available Now</h2>

<div class="card-grid columns breakout">
<div class="card vertical">
<h3>Platform support</h3>:
Windows (MSVC/Clang), macOS (Clang), Linux (GCC/Clang). Distributed through GitHub releases, Launchpad PPA (Ubuntu/Debian), COPR (Fedora/RHEL), AUR (Arch).

</div>

<div class="card vertical">
<h3>Project management</h3>:
Weave command-line tool handles automated dependency management, MayaFlux version acquisition and installation, and C++ project generation with autogenerated CMake configuration loading MayaFlux library and all necessary includes.

</div>
<div class="card vertical">
<h3>Audio backend</h3>:
RtAudio with WASAPI (Windows), CoreAudio (macOS), ALSA/PulseAudio/JACK (Linux).

</div>

<div class="card vertical">
<h3>Graphics backend</h3>:
Vulkan 1.3 with complete pipeline from initialization dynamic rendering, command buffer management, swapchain presentation. Currently supports 2D particle systems, geometry networks, shader bindings with node data via push constants and descriptors. Foundation for procedural animation, generative visuals, and computational geometry

</div>
<div class="card vertical">
<h3>Live coding</h3>:
Lila JIT system with LLVM 21+ supporting full C++ syntax including templates, constexpr evaluation, and incremental compilation.
</div>

<div class="card vertical">
<h3>Temporal coordination</h3>:
Complete coroutine infrastructure with sample-accurate scheduling, frame-accurate synchronization, multi-rate adaptation, and event-driven execution.
</div>

<div class="card vertical">
<h3>Documentation</h3>:
Comprehensive tutorials starting from "load a file" building to complete pipeline architectures. Technical blog series covering lock-free architecture and state coordination patterns. Persona(musician, visual artist, etc...) based onboarding addressing mental-model transitions from Pure Data, Max/MSP, SuperCollider, p5.js, openFrameworks, Processing.

</div>

<div class="card vertical">
<h3>Testing</h3>:
Over 700 component tests validating lock-free patterns, buffer processing, node coordination, graphics pipeline integration.
</div>
</div>

<hr/>

<div class="card">
<h2>Working Examples</h2>

**Load and process audio:**

```cpp
void compose() {
    auto sound = vega.read("path/to/file.wav") | Audio;
    auto buffers = MayaFlux::get_last_created_container_buffers();

    auto poly = vega.Polynomial([](double x) { return x * x; });
    MayaFlux::create_processor<PolynomialProcessor>(buffers[0], poly);
}
```

**Build processing chains:**

```cpp
void compose() {
    auto sound = vega.read("drums.wav") | Audio;
    auto buffers = MayaFlux::get_last_created_container_buffers();

    auto bitcrush = vega.Logic(LogicOperator::THRESHOLD, 0.0);
    auto crush_proc = MayaFlux::create_processor<LogicProcessor>(buffers[0], bitcrush);
    crush_proc->set_modulation_type(LogicProcessor::ModulationType::REPLACE);

    auto clock = vega.Sine(4.0);
    auto freeze_logic = vega.Logic(LogicOperator::THRESHOLD, 0.0);
    freeze_logic->set_input_node(clock);
    auto freeze_proc = MayaFlux::create_processor<LogicProcessor>(buffers[0], freeze_logic);
    freeze_proc->set_modulation_type(LogicProcessor::ModulationType::HOLD_ON_FALSE);
}
```

**Recursive filters:**

```cpp
auto string = vega.Polynomial(
    [](const std::deque<double>& history) {
        return 0.996 * (history[0] + history[1]) / 2.0;
    },
    PolynomialMode::RECURSIVE,
    100
);
string->set_initial_conditions(std::vector<double>(100, vega.Random(-1.0, 1.0)));
```

**Audio-visual coordination:**

```cpp
auto control = vega.Phasor(0.15) | Audio;
control->enable_mock_process(true);

auto particles = vega.ParticleNetwork(
    600,
    glm::vec3(-2.0f, -1.5f, -0.5f),
    glm::vec3(2.0f, 1.5f, 0.5f),
    ParticleNetwork::InitializationMode::GRID
) | Graphics;

particles->map_parameter("turbulence", control, NodeNetwork::MappingMode::BROADCAST);
```

</div>

<div class="card">
<h2>What Comes Next</h2>

This release establishes the foundation. Future development includes:

- Expanded graphics capabilities moving toward 3D rendering, input handling systems, networking components for distributed processing, hardware acceleration through CUDA and FPGA implementations. WebAssembly builds enabling interactive web demos running actual MayaFlux C++ code in browsers.
- Additional backends: JACK audio, multiple Vulkan extensions, custom backend interfaces for embedded systems or specialized hardware.
- Educational content: video walkthroughs, interactive examples, pattern libraries demonstrating specific creative techniques.
- Institutional partnerships exploring funding for full-time development, hardware integration research, academic collaboration on novel algorithms.
</div>

## Get Started

Visit https://mayaflux.org for documentation, tutorials, and downloads.

Installation takes minutes. First working audio: under five minutes following Sculpting Data tutorial.

The framework provides default automation for common workflows while enabling complete override for specialized requirements. Start with simple patterns. Progress to architectural customization when needed. The documentation meets you where you are.

Source: https://github.com/mayaflux/MayaFlux  
License: GPL-3.0  
Contact: [mayafluxcollective@proton.me]

MayaFlux exists because computational substrate evolved while creative tools maintained backward compatibility with 1980s architectures. New paradigms become possible when you're not constrained by analog metaphors, disciplinary separation, or protective abstraction layers.

The substrate is ready. Build what you imagine.
