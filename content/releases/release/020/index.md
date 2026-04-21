---
title: "MayaFlux 0.2.0: Live Signal Matrix, Physical Modelling, and Input as Nodes"
summary: "MayaFlux 0.2.0 delivers the first major expansion pass on the 0.1 foundation. Two primary focus areas: IO completion and physical modelling synthesis, plus operator-driven geometry networks, EventChain as the primary temporal primitive, and hardware input routed directly into the node graph."
---

**March 2026** — Commit `994f774`

0.1.0 established the substrate: lock-free computation graphs, Vulkan 1.3 dynamic rendering, the Lila JIT environment, and the unified numerical domain model. 0.2.0 builds the first major layer on top of it.

Two areas drive this release. The live signal matrix is now complete: audio files, video files, live camera devices, and image assets all route through a single `IOManager` into the same buffer and processor architecture that has always handled synthesis. The physical modelling suite adds three distinct network types covering the main time-domain and frequency-domain approaches to acoustic modelling. Alongside those, operator-driven geometry networks, a redesigned temporal API, and MIDI/HID as first-class node graph signal sources arrive in this release.

---

<div class="card-grid columns breakout">

<div class="card vertical">
<h2>IO: The Live Signal Matrix</h2>

Every media type now enters MayaFlux through a single interface and exits into the same buffer architecture that handles synthesis nodes.

`IOManager` is the orchestration layer: load a sound file, open a video file, open a camera device, load an image asset. In each case, the result is a container wired to the same `SoundContainerBuffer`, `VideoContainerBuffer`, or `TextureBuffer` infrastructure that processes everything else. The calling code does not change based on the source type.

Camera input runs a dedicated background decode thread. The graphics thread is never blocked by device I/O. FFmpeg handles demux on all three platforms with platform-appropriate backends: V4L2 on Linux, AVFoundation on macOS, DirectShow on Windows.

Video files with embedded audio tracks surface both streams simultaneously through `VideoLoadResult`, extracting the audio into a `SoundFileContainer` alongside the video container. HEVC frame rate detection and sustained decode correctness are addressed.

`WindowContainer` and `WindowAccessProcessor` complete the loop in the other direction, providing pixel readback from GLFW windows back into the buffer graph.
</div>

<div class="card vertical">
<h2>Physical Modelling: Three Network Types</h2>

Physical modelling synthesis in MayaFlux arrives as three `NodeNetwork` subclasses, each addressing a distinct modelling approach. All three expose the standard `map_parameter` interface for external node control and integrate with the existing polyphonic buffer architecture via `NetworkAudioBuffer` and `NetworkAudioProcessor`.

**ModalNetwork** decomposes resonance into independent modes: each mode is a decaying sinusoidal oscillator with its own frequency ratio, decay time, and amplitude. Four predefined spectra are available: harmonic (integer multiples), inharmonic (bell-like: 2.76f, 5.40f, 8.93f...), stretched (piano-like stiffness), and custom ratio vectors. Spatial excitation distributes strike energy across modes by position. Mode coupling transfers energy between adjacent modes per sample, modelling the energy exchange of coupled physical resonators.

**WaveguideNetwork** moves to the time domain, propagating waves through delay lines. Two propagation modes reflect two physical structures. Unidirectional (string): a single loop, extended Karplus-Strong, frequency-dependent loop filter. Bidirectional (tube): two rails with signed reflection at each termination, producing odd-only or full harmonic series by configuration. `pluck()` seeds with shaped triangle displacement; `strike()` seeds with a Gaussian-windowed noise burst. A pickup position parameter controls spectral content by tapping at different points along the delay line.

**ResonatorNetwork** operates in the spectral shaping register: N second-order IIR biquad bandpass sections, each tuned to an independent centre frequency and Q factor. Feed it noise and it functions as a formant synthesiser; feed it a pitched source and it voices vowels; feed it an arbitrary signal and it morphs toward the target resonance profile. Coefficients follow the RBJ Audio EQ Cookbook bilinear transform. A single shared exciter or per-resonator independent exciters both supported.
</div>

</div>

<hr/>

<div class="card">
<h2>Operator-Driven Geometry Networks</h2>

The graphics pipeline gains a new structural layer: operators that drive geometry network topology and motion independently from the network itself.

`PhysicsOperator` runs N-body simulation, computing inter-particle forces and updating vertex positions per frame. `TopologyOperator` generates proximity graph connectivity from current vertex positions: minimum spanning tree, k-nearest, nearest neighbor, and sequential modes. `PathOperator` interprets a vertex set as a parametric curve using Catmull-Rom, B-spline, or linear interpolation.

`PointCloudNetwork` uses the operator architecture as its foundation. `CompositeGeometryBuffer` and `CompositeGeometryProcessor` handle multi-network rendering into a single pipeline. Vertex layouts carry per-vertex colour and thickness, making line weight and colour data carried in the same stream as position.

`Kinesis` grows substantially: `VertexSampler` generates spatial distributions (Lissajous, Fibonacci sphere, torus), proximity graph generation moves here as a reusable utility, and `Stochastic` provides unified stochastic generation infrastructure used across node types including a refactored `Random` node.

`ViewTransform` arrives as a push-constant helper for 3D view/projection matrices, with depth-enabled pipeline support and per-window depth image management. macOS receives a CPU-side quad expansion fallback for line rendering while mesh shader support awaits the upstream MoltenVK merge.
</div>

<hr/>

<div class="card">
<h2>EventChain: Temporal Sequencing</h2>

The coroutine scheduler gains `EventChain` as its primary temporal sequencing primitive. The old `Sequence` API is removed; `EventChain` supersedes it with a cleaner fluent interface.

```cpp
create_event_chain(scheduler)
    .then([]() { modal_bell->excite(1.0); })
    .then([]() { modal_bell->excite(0.6); }, 0.75)
    .then([]() { modal_bell->excite(0.3); }, 0.75)
    .times(4)
    .start();
```

`.then()` adds an action with a delay from the previous event. `.repeat(n)` repeats the last event n additional times. `.times(n)` repeats the entire chain. `.wait(s)` inserts a silent delay. `.every(interval, action)` is a convenience form for periodic actions. `.on_complete()` fires after the final event regardless of how the chain stopped. Timing is sample-accurate, backed by the same `SoundRoutine` coroutine infrastructure used throughout.

`NodeTimer` is replaced by `TemporalActivation`. `NodeTimeSpec` is replaced by the `Temporal` proxy DSL. `Gate` and `Trigger` awaiters handle condition-driven and signal-driven suspension. `MultiRateDelay` coordinates cross-domain suspension at both sample and frame boundaries.
</div>

<hr/>

<div class="card">
<h2>Input as Node Graph Signal Sources</h2>

MIDI and HID input are now first-class node graph signal sources. An `InputNode` behaves like any other node: it has a scalar output, it participates in parameter mapping, and it drives downstream processors through the same interfaces as synthesis nodes.

`MIDINode` filters by message type, channel, note number, and velocity. Note on/off, CC, pitch bend, and program change all map to normalized scalar outputs. A configurable velocity curve transforms raw MIDI velocity before it enters the graph.

`HIDNode` parses raw HID report bytes into scalar values via configurable byte offset, byte size, and signedness. Three parse modes: axis (normalized with optional deadzone), button (bitmask extraction), and custom (user-provided parsing function).

Both node types support per-value smoothing: none, linear, exponential, and slew-limited modes. Event-driven callbacks attach directly to the node: `on_value_change`, `on_threshold_rising`, `on_threshold_falling`, `on_button_press`, `on_button_release`, `while_in_range`.

The `InputSubsystem` coordinates `HIDBackend` (HIDAPI) and `MIDIBackend` (RtMidi) with lock-free registration. `InputManager` handles async dispatch to `InputNode` instances. The full path from hardware event to node graph value involves no allocations on the hot path.
</div>

<hr/>

<div class="card">
<h2>Working Examples</h2>

**Rhythm machine: phasor envelopes, step sequencer, topology**

Each drum is a phasor envelope driving a waveform. `schedule_pattern` gives per-step
probability control. Kick hits deposit spatial points; `TopologyGeneratorNode` rebuilds
connectivity from them each frame, proximity threshold modulated by kick energy.

```cpp
void rhythm_topology()
{
    auto window = create_window({ .title = "Rhythm Topology", .width = 1920, .height = 1080 });

    auto kick_phasor = vega.Phasor(20.0, 1.0);
    auto kick_env    = vega.Polynomial([](double x) { return std::exp(-x * 15.0); });
    kick_env->set_input_node(kick_phasor);
    auto kick_tone   = vega.Sine(55.0)[0] | Audio;
    kick_tone->set_amplitude_modulator(kick_env);

    auto snare_phasor = vega.Phasor(30.0, 1.0);
    auto snare_env    = vega.Polynomial([](double x) { return std::exp(-x * 25.0); });
    snare_env->set_input_node(snare_phasor);
    auto snare_noise  = vega.Random();
    auto snare_fir    = vega.FIR(snare_noise, std::vector{ 0.3, 0.4, 0.3 });
    auto snare        = snare_fir * snare_env;
    register_audio_node(snare, 1);

    auto state = std::make_shared<struct { bool chaos = false; }>();

    schedule_metro(0.5, [kick_phasor]()   { kick_phasor->reset();  }, "kick");
    schedule_pattern(
        [](uint64_t s)  { return (s % 4 == 2); },
        [snare_phasor](std::any hit) { if (std::any_cast<bool>(hit)) snare_phasor->reset(); },
        0.25, "snare");
    schedule_pattern(
        [state](uint64_t) { return state->chaos ? get_uniform_random(0.0, 1.0) > 0.6 : true; },
        [](std::any) {},
        0.125, "hat");

    auto topo  = vega.TopologyGeneratorNode(64, TopologyMode::K_NEAREST, 4) | Graphics;
    auto geom  = vega.GeometryBuffer(topo) | Graphics;
    geom->setup_rendering({ .target_window = window });
    window->show();

    schedule_metro(0.016, [topo, kick_tone, state]() {
        float energy = rms(kick_tone->get_audio_buffer().value()) * 8.0f;
        if (energy > 0.1f) {
            topo->add_point({ glm::vec3(get_uniform_random(-1.5f, 1.5f),
                                        get_uniform_random(-1.0f, 1.0f), 0.0f),
                              glm::vec3(0.9f, 0.4f + energy, 0.2f), 4.0f });
        }
        topo->set_proximity_threshold(0.4f + energy * 0.6f);
        topo->rebuild();
    });

    on_key_pressed(window, Keys::C, [state]() { state->chaos = !state->chaos; });
}
```

**ResonatorNetwork: noise as formant source**

```cpp
void vowel_body()
{
    auto noise    = vega.Random(-1.0, 1.0);
    auto formants = vega.ResonatorNetwork(5)[0] | Audio;
    formants->set_exciter(noise);
    formants->set_resonator(0,  730.0, 12.0);
    formants->set_resonator(1, 1090.0, 10.0);
    formants->set_resonator(2, 2440.0,  8.0);
    formants->set_resonator(3, 3000.0,  6.0);
    formants->set_resonator(4,  400.0, 14.0);

    auto lfo = vega.Sine(0.08, 1.0);
    formants->map_parameter("frequency", lfo, MappingMode::BROADCAST);
}
```

**Live camera into the graphics pipeline**

```cpp
void camera_feed()
{
    auto cam        = io.open_camera({ .device_name = "/dev/video0" });
    auto cam_buffer = io.hook_camera_to_buffer(cam) | Graphics;
}
```

**WaveguideNetwork: cylindrical bore, pickup modulation**

```cpp
void tube_resonance()
{
    auto tube = vega.WaveguideNetwork(
        WaveguideNetwork::WaveguideType::TUBE, 110.0)[0] | Audio;
    tube->strike(0.1, 0.9);

    auto mod = vega.Sine(0.05, 1.0);
    tube->map_parameter("position", mod, MappingMode::BROADCAST);
    tube->map_parameter("damping",  mod, MappingMode::BROADCAST);
}
```

**MIDI CC driving resonator bandwidth**

```cpp
void midi_formants()
{
    auto cc_mod   = vega.read_midi(MIDIConfig::cc(1), InputBinding::midi("UM-ONE"));
    auto formants = vega.ResonatorNetwork(4)[0] | Audio;
    formants->set_exciter(vega.Random(-1.0, 1.0));
    formants->map_parameter("q", cc_mod, MappingMode::BROADCAST);
}
```

**EventChain: deterministic multi-event sequence**

```cpp
void waveguide_sequence()
{
    auto str = vega.WaveguideNetwork(
        WaveguideNetwork::WaveguideType::STRING, 220.0)[0] | Audio;

    EventChain chain(get_scheduler(), "pluck_seq");
    chain.then([str]() { str->pluck(0.3, 1.0); })
         .then([str]() { str->pluck(0.7, 0.6); }, 0.8)
         .then([str]() { str->pluck(0.5, 0.3); }, 0.8)
         .times(4)
         .start();
}
```
</div>

<hr/>

<div class="card">
<h2>Breaking Changes</h2>

0.2.0 includes several renames and removals. Upgrade paths are direct one-to-one replacements unless noted.

- `ContainerBuffer` renamed to `SoundContainerBuffer`
- `ContainerToBufferAdapter` renamed to `SoundStreamReader`
- `StreamWriteProcessor` renamed to `SoundStreamWriter`
- `FileBridgeBuffer` removed: replaced by `SoundFileBridge`, a simplified single-buffer design with no inherited specialized processors
- `NodeTimer` replaced by `TemporalActivation`
- `NodeTimeSpec` replaced by the `Temporal` proxy DSL
- `OutputTerminal` removed
- `Sequence` API removed: `EventChain` is the replacement
- `set_target_window` now requires a buffer parameter
- Processing chains gain explicit `preprocess` and `postprocess` slots
- `Yantra::IO<T>` renamed to `Datum<T>` throughout

</div>

<hr/>

<div class="card">
<h2>What Comes Next</h2>

0.3.0 focus: Nexus spatial entity lifecycle system, Yantra offline compute workflows, and the `GranularWorkflow` grammar as the first public instantiation of the compute grammar model. `SamplingPipeline` for polyphonic audio infrastructure. `Portal::Text` rendering subsystem. The `DomainSpec`/`operator[]` syntax replacing `CreationProxy`.

Ongoing: AUR package, Flatpak targets for `mayaflux` base and `mayaflux+lila` with LLVM 21 for the Steam Deck/Bazzite ecosystem. Rosetta Stone documentation series for users arriving from Pure Data, Max/MSP, SuperCollider, and p5.js.

</div>


<div class="card">
<h2> Commit Summary </h2>

<p>See full diff: <a href="https://github.com/MayaFlux/MayaFlux/compare/v0.1.2...v0.2.0">Github</a></p>

<pre><code>
994f774c (HEAD -> main, origin/main) chore(release): prepare 0.2.0
a1b3dd36 (origin/develop) refactor(buffers/kinesis): centralise texture quad geometry in Kinesis
5b59e243 refactor(buffers): replace vk::DescriptorType with DescriptorRole in public API
880bf3f3 (origin/ci/020_cleanup) ci(release): append auto-generated changelog to stable releases
429fa880 fix(ci): Gate all distribution steps behind ci flag
8cd4b0b0 ci: Simplify release body, create per release files
4f9269e9 ci(release): remove per-OS distribution READMEs and section templates
b42432ec remove dormant woodpecker ci
424ad24b docs: consolidate tutorials to website, remove stale doc references
de82228d (origin/update/descriptor_bindings) feat(buffers): add NodeNetwork source support to DescriptorBindingsProcessor
da0d360e refactor(buffers): extract network GPU/audio data helpers into NetworkUtils
6f64116c feat(buffers): add AudioBuffer and host VKBuffer sources to DescriptorBindingsProcessor
c42157fb feat(nodes): add NodeCapability bitmask for descriptor binding introspection
f945a6a7 fix(buffers): fix AggregateBindingsProcessor fallback and add ProcessingMode
ae5baf3f fix(nodes): populate FilterContextGpu gpu_float_buffer from input history
c83d7f9d fix(nodes): populate gpu_float_buffer in GPU context update path
f55ba0ef ci(copr): replace push trigger with daily schedule, skip if no changes
e2c71f3e ci: split monolithic workflow into pr-build, dev-release, and release
fcef4bb9 update(README.md): add more examples
f1751351 compute processor is fully validate: Include in global header!
9e1eb873 fix(warnings): Silence msvc type noise for SoundFileBridge
08a186e4 feat(Vulkan): add dedicated compute command pool and command buffer path
a441f690 Buffers: fix device-local download staging and command buffer handling
003c25d4 fix(API): ensure graphics processors created via create_processor use GRAPHICS_BACKEND token
ef2d0c8c Graphics: remove fallback renderer debug log for empty renderables
9a2c1848 update(RenderProcessor): Add back cull mode to view transform
1c63f555 feat(shaders): add ViewTransform-compatible default shader variants
88835cb8 feat(renderprocessor): add ViewTransform support and depth-enabled pipelines
4b9cb36a feat(kinesis): add ViewTransform push-constant helper
8411c5c0 feat(rendering): propagate depth requirements from buffers to presenter
66bddc6d feat(renderflow): add optional depth attachment to dynamic rendering
22495110 feat(graphics): expose depth attachments via DisplayService
70347de6 feat(graphics): add per-window depth image management
ae89b998 feat(RenderProcessor): add configurable blend and depth-stencil state
40066f10 fix(RootNode): eliminate register/unregister race against m_Nodes iteration
789821b9 README: Update with examples, to 0.2 scheme
5f63710e fix(ubuntu): add libavdevice-dev to control file
d3b092f9 Introduce CompositeOpNode for N-ary node combination
cd0a91d1 refactor(Nodes): BinaryOpNode into NodeCombine conduit
39a84221 Introduce ChainNode pipeline and move >> operator to Graph API
5b6e0fd8 Remove API dependency from NodeUtils and introduce MAX_CHANNEL_COUNT
f5adc7ae Move NodeConfig ownership to Engine and propagate to NodeGraphManager
2b5d8bd8 Add NetworkAudioBuffer and NetworkAudioProcessor for NodeNetwork audio capture
180bb981 Add StreamReaderNode and NodeFeedProcessor for buffer-to-node streaming
b9680843 refactor(Nodes): Move Constant node to Conduit folder
23bd3b93 fix(io): resolve Windows FFmpeg dshow dimension and pix_fmt failures
6fefeddc feat(io): add camera support to IOManager
08882927 feat(io): add CameraReader for live camera device input
dd91c06c feat(kakshya): add CameraContainer for single-slot live camera input
f9ce56fd feat(io): extend FFmpegDemuxContext and IOService for device input
1878a5cf build: add libavdevice to FFmpeg dependencies across all platforms
b628a989 refactor(API): centralise file IO through IOManager, remove Depot wrappers
83ff6a2d feat(IO): introduce IOManager for centralised media loading and buffer wiring
1a36fd68 Refactor: Rename Yantra::IO<T> to Datum to resolve namespace conflicts
bdf9f4af fix(video): correct HEVC frame rate detection and decode loop error handling
ca94da71 fix(video): eliminate black frames and VK_ERROR_DEVICE_LOST on sustained playback
41ebe64d fix(io): Audio-visual sync handling correctness
5c50ee29 wip(io): ring-buffer demand-decode via IOService [DO NOT USE]
c9fad223 WIP(IO): Streaming mode instead of file loading DO NOT USE
a9870a7d WIP(IO): Temporary fix, DO NOT MERGE/USE
8050df6b feat(SoundFileReader): Allow reading directly from dmux context
7323a0c8 perf(kakshya): bypass region extraction for sequential frame access
b7f27372 feat(buffers): add VideoStreamReader and VideoContainerBuffer
90dafda8 feat(kakshya): add FrameAccessProcessor as default video container processor
d58b7dd6 feat(IO/Kakshya): add VideoFileReader and video container hierarchy
1f4e7e81 feat(Kakshya): add VideoStreamContainer and VideoFileContainer
94f79f64 feat(IO): add VideoStreamContext: RAII FFmpeg video codec + swscale owner
cca82f2e refactor(io): port SoundFileReader to FFmpegDemuxContext/AudioStreamContext
2459607e feat(io): introduce FFmpegDemuxContext and AudioStreamContext
3d7dc5bb feat(NodeNetwork): add output scale for uniformly scaling output audio buffer
fb8c8ff7 feat(Network): add per-node audio buffer interface across NodeNetwork hierarchy
c5ae7251 refactor(nodes): promote OutputMode, Topology, MappingMode to Nodes::Network namespace
ebf2864c feat(nodes): add OutputMode::AUDIO_COMPUTE for silent network processing
e5f51580 feat(nodes): add ResonatorNetwork: IIR biquad bandpass filter network
142fa777 feat(nodes): add Constant generator node
33b73b68 fix(processor): Use regiongroup header instead of region
5d2edf8f refactor(graphics): route swapchain readback through BackendResourceManager
aa62f2f1 refactor(vulkan): centralize format traits + make window readback format-aware
891dbe38 feat(Kakshya): add SpatialRegionProcessor for parallel spatial region extraction
58a1b10f refactor(kakshya): migrate WindowContainer to RegionGroup-based extraction model
25efae24 refactor(Region): Offload region extraction from container to Utils
b83288c9 feat(Graphics): add WindowContainer and WindowAccessProcessor
ebe81882 feat(Kakshya): implement SurfaceUtils pixel readback via DisplayService/BufferService
76f79802 update(display service): Prepare for accessing images from windows
60c2456c update(shaders): Provide more defaults, allow user dir auto detection
744c4eb3 build: add example shader discovery for dev builds
ef5e47c1 tests(WaveguideNetwork): extend coverage for bidirectional mode and boundary behavior
3101d423 update(WaveguideNetwork): explicit boundary reflections and measurement mode refinement
eaa7cefe update(WaveguideNetwork): introduce bidirectional propagation mode (Phase 2 – Part 1)
e893bccb test(WaveguideNetwork): unit and integration tests
d6acabdb update(Network): Waveguide registration, NodeNetwork simplification
44904d83 feat(WaveguideNetwork): initial unidirectional propagation implementation
87c95735 tests(FeedbackBuffer): HistoryBuffer migration and new feedback behavior
a89e1908 refactor(FeedbackBuffer): use HistoryBuffer; simplify processor logic
86e9760b chore(updates): Minor cleanups
1c66e51e feat(Graphics networks): Add reinitialize support
50e76e6b feat(kinesis): extract spatial generation into VertexSampler
d8601a8e feat(PointCloudNetwork): Expand initialization modes
9b0fa8fe Fix Windows JIT window responsiveness via main-thread execution
b87d3432 feat(buffers): add CompositeGeometryBuffer and CompositeGeometryProcessor
5716f427 feat(nddata): introduce SCALAR_F32 modality and refine vertex layouts
2bfac1ea feat(render): add vertex range drawing and per-buffer layout override
1a4401b4 move AggregateBindingsProcessor from Textures/ to Nodes/
a3366a85 feat(kinesis): add geometric primitive generation and transformation utilities
874ba0d9 feat(windowing): y-axis orientation and add window coordinate utilities
ddcdb983 test(EventChain): expand EventChain coverage for repeat, times, wait and every
2bbca3c2 Kriya: extend EventChain with repeat, times, wait and every; decouple coroutine from instance
b629f17a chore(refactor): Reduce dependency on global API PART II
38f43da9 feat(api): add buffer routing API with time-based conversion utilities
ea89280f feat(buffers): implement channel routing with crossfade transitions
78d7c78d chore(refactor various): Reduce dependency on global API
78bb201d refactor(core): inject stream configuration into NodeGraphManager and DSP units
e414018e refactor(routing): finalize routing lifecycle and block-based fade handling
718f7357 feat(nodes): introduce routing transition infrastructure for nodes and networks
07275d83 docs(OS versions): Clarify macos 15 requirement.
e0779bce ci(Macos): Bump macos for apple silicon to 15.6
da8b6cf8 fix(macos): Make mesh shaders optional until PR merge on MoltenVK
fbc4992f feat(vulkan): add mesh shading pipeline support
a57f99f5 feat(vulkan): enable dynamic dispatch loader for extension functions
6401a1c0 update(Geometry buffers): Make macos fix more dynamic
5144694d fix(graphics): Add macOS line rendering fallback via CPU-side quad expansion
a4761364 fix(jit): implement runtime discovery for Eigen include paths
53b1f317 fix(windows): use references for system includes
b376dfca refactor: decouple platform introspection from CMake-generated config
0597c22b refactor(EventChain): strengthens as the primary temporal sequencing primitive
697d2a84 BREAKING CHANGE: Remove Sequence API and consolidate temporal sequencing around EventChain
6ed2b788 chore(cleanup): Docs and leftovers after previous refactor
aaa5da42 feat(API): introduce Temporal proxy DSL and remove NodeTimeSpec
da874fe8 BREAKING CHANGE: replace NodeTimer with domain-agnostic TemporalActivation
66cead06 chore(tests): Disable specific task/timer tests until API refactor is complete
18f0fac7 BREAKING CHANGE: OutputTerminal gets the axe
7be2072d test(network/modal): add unit and integration tests for exciter, spatial excitation, and coupling
37cd55b3 feat(network/modal): add exciter system, spatial excitation, and modal coupling
e696f483 feat(nodes/network): add PointCloudNetwork as operator-driven structural spatial network
41034695 chore(documentation): Update doxygen banners to new Vertex format
d697e7a6 refactor(Operators): unify initialization around semantic vertex types
8a494d61 refactor(Nodes): Remove raw glm:: helper API, part I
ffadff63 update(Geometry Nodes): Make primitive topology configurable.
f5a4d0f0 refactor: simplify PathOperator and TopologyOperator by removing wrapper structs
b2bbcc9a refactor: consolidate RenderConfig to Portal::Graphics and VKBuffer
5418adf8 refactor(Nodes): Move vertex types from GpuSync to Nodes namespace
bceefe0a feat(Nodes/Graphics): Add per-vertex color/thickness support to topology and path generators
e931143b refactor(NetworkGeometryBuffer and NetworkGeometryProcessor): decoupling
afc350b5 refactor(NodeNetwork): Operator support and layout decoupling
059e86ee refactor(PhysicsOperator): Own bounds mode and rng generator
4ab85a89 refactor(Nodes/GpuSync): eliminate redundant vertex storage in PathGeneratorNode and TopologyGeneratorNode
9813d01b update(NetworkGeometryProcessor): use GraphicsOperator vertex interface
6e9d2113 refactor(ParticleNetwork): operator-based architecture
5d8e2e23 update(NetworkOperators): Feature parity with existing networks
8cdc1f36 update(PhysicsOperator): Port more physics operation from network
161adc42 feat(TopologyOperator): proximity-based network generation
04408cbb feat(PathOperator): curve-based network interpretation
99a88cbb feat(PhysicsOperator): N-body simulation via GraphicsOperator
6895b772 feat(NetworkOperator): base and abstract GraphicsOperator
a98c4440 feat(nodes): add TopologyGeneratorNode for dynamic mesh topology generation
0d7b58cd feat(Kinesis): Topology Generator - Proximity Graphs
5fbbdf6b refactor(VertexLayout): Static factory constructors
d07830ff refactor(PathGeneratorNode): Make generation logic reusable
18bd2bc2 feat(free draw): Add temp points + delayed resolution to path generator node
400bc247 update(PathGeneratorNode): Dont recalculate every frame
e2577ef5 feat(Graphics): Connected points
e97f72d6 feat(shaders): Line drawing support
a764a4bd fix(macos): clang does not resolve cpp file templates
94bc8ca6 Add basis matrices and motion curve primitives
cedc9a4e feat(EigenInsertion): semantic Eigen-to-NDData conversion
caa16646 feat(EigenAccess): NDData to Eigen conversion and views
ed07fd06 refactor(Nodes): Migrate Random node to Kinesis::Stochastic infrastructure
2498343e BREAKING CHANGE: Bikeshed Random node namespace
e64b3831 refactor(Random): Use the new Stochastic class in Engine
a68d81f3 feat(Kinesis): Add unified stochastic generation infrastructure
d8a88312 feat{Unify ring buffer}: implementations with policy-based architecture
5a7b62ba update(scripts): Various window setup improvements
0281a949 refactor(Parallel): Split Parallel module into Dispatch and Execution
c5022908 refactor(Utils): Move EnumUtils to Transitive/Reflect as EnumReflect
c5dcede0 refactor(Utils): Move utility functions to ChronUtils.hpp
0d42eb02 refactor(utils): Move distribution functions to Kinesis
353c2e8f feat(Nodes): Introduce NodeSpec for enhanced node metadata
dfe1f8d3 refactor(utils): Context aware offload Part 1
f90e7301 fix(midi): backend lifecycle and locking issues.
016ca1da Resolve build and linkage errors surfaced by CI and local test builds.
6ee47465 feat(API): Input representation in Creator interface
2d2f4f8f feat(Input): MIDI integration via RtMidi backend
d3e90284 deps(input): Add rtmidi, and make hid non optional
3dace847 chore(cleanup): Windows build system improvements
ee74433d fix(windows): workaround for MSVC warnings
b335645e fix(Windows): Handle path sep in local file
74f7a56b refactor(windows): Move some deps to vcpkg
04869585 update(windows): Bump llvm to 21.1.8
d5cb5ca0 fix(windows): Updated cmake project hid detection
baca0fcb update(cmake): Add HIDAPI config to MayaFluxConfig.cmake
dba4c3e7 Core/Input: finalize input binding + service-backed device resolution (HID-first)
f41fe707 refactor(Input): Offload all structs to GlobalInputConfig
6ee4c943 feat(API/Input): add user-facing input helpers and event-driven InputNode callbacks
55a8b227 feat(Nodes/Input): add HIDNode for parsing HID reports into node signals
69cc57ad fix(macos): Workarounds for the braindead hostility of macos towards cpp
c943518c update(Windows): Add HIDAPI to ci and scripts
dc55ed65 build(Unix): Add hidapi deps to Linux and macOS
c54ab636 refactor(Engine): Input subsystem/manager registry and API cleanup
99fa9df9 feat(Input): Add InputManager to SubsystemProcessingHandle
1c8e6ed6 feat(InputSubsystem): add input subsystem coordinating backends and InputManager
6f13c811 feat(Input): add InputManager for async input dispatch to InputNodes
465b2497 feat(Input): introduce InputNode infrastructure
18bf7ee2 refactor(Input): unify input typing
a001ff9a feat(Input): add HIDBackend using HIDAPI
566d6b3f feat(HID): Initial build system deps for HIDAPI
5332168e feat(InputBackend): Initial implementation of InputBackend and GlobalInputConfig
a3af1457 refactor(Buffers)!: Remove DescriptorBuffer wrapper class
747f781d feat(Buffers): Add INTERNAL/EXTERNAL processing modes to binding processors
2e3c51e1 feat(audio): Add JACK/PipeWire buffer size handling and API preference
4062fbf4 fix(ImageReader): inspect and convert 3 channel image to RGBA
32c164cb refactor(main loop): Modularize windows run loop
3ef1d925 fix(windows): glfw window events to main thread
53eac290 feat(API): Add helpers for key/mouse events
1bf1fe61 feat(kriya): add coroutine-based input event helpers
838e4648 refactor(EventSource): Use EventFilter for all source signals
e118ef50 chore(updates): Minor fixes for Event Source refactor prep
01874b0f refactor(EventAwaiter): Use new EventFilter.
5284eb0f feat(EventFilter): Granular filtering of Windowing events
261b1ae2 refactor(Awaiters): Split DelayAwaiters and add GetPromise awaiter
a4aacc33 feat(clear color): BackendWindowHandler supports empty window rendering
4c887448 BREAKING CHANGE: set_target_window now requires buffer param
2c4c5617 feat(Windowing): New method for clear color setting
12f9a80a chore(updates): Various build system and lib exports
cd19d7b1 refactor(cmake): Separate examples dir for project_launcher
5776d637 feat(Keys): Configure enums and string mappings for GLFW backend
eb7b6313 refactor(Containers): const refs to shared ptrs
03988856 refactor(DataProcessingChain): const refs to containers and functors
dcd53ee5 refactor(DataProcessor): const references and cleanups
d8a19a7b refactor(Buffers): More const refs and cleanups
200e089a refactor(Buffers): const ref pointers in processing_function
f113b342 refactor(Buffers): is_compatible_with and other improvements
05756cac refactor(BufferProcessor): attach, detach and process methods
38615b89 BREAKING CHANGE: FileBridgeBuffer removed, SoundFileBridge is simplified redesign
fb78473a BREAKING CHANGE: ContainerBuffer is now SoundContainerBuffer
9d8d7c86 BREAKING CHANGE: ContainerToBufferAdapter is now SoundStreamReader
878a7d29 BREAKING: StreamWriteProcessor is now SoundStreamWriter
ac0f03e6 refactor(buffers): introduce explicit pre/post stages and unified process_complete path
3fb0f958 bump dev version to v0.2.0-dev
</code></pre>

</div>

---

**Source**: [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux)  
**Docs**: [mayaflux.org](https://mayaflux.org)  
**License**: GPLv3
