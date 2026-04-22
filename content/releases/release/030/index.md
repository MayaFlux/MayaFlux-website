---
title: "MayaFlux 0.3.0: The Full Stack"
summary: "MayaFlux 0.3.0 is the first release to cover the full substrate: spatial entity lifecycles, offline compute with a production grammar, polyphonic multi-cursor playback, mesh as addressable data, text as texture, GPU readback, and networking. Developed in parallel with the TOPLAP Bengaluru live set on a Steam Deck."
---

**April 2026**

MayaFlux started as an argument. Not a roadmap, not a plan: a working implementation of a specific claim, which is that audio, visual, and control data do not need to be separate domains managed by separate systems. They are all numbers. Routing is a scheduling annotation. The domain is decided last, not first.

That argument shipped in its first public form on December 31, 2025. By then the tutorials were already written, the visual artist persona page was already teaching Lorenz attractors driven by audio-rate chaotic maps, and Lila was already compiling and executing arbitrary C++23 in under one buffer cycle.

0.2 shipped in March 2026, coinciding with a 60-minute CppOnline talk with live demonstration. The ISRO 3-day intensive workshop ran on 0.2: physical modelling networks on day 1, rhythm machines driving structural GPU visuals on day 2, shader programming with node-to-push-constant bindings on day 3.

0.3 was developed in parallel with the TOPLAP Bengaluru performance: a 20-minute extended live coding set on a Steam Deck running Arch Linux in a Distrobox container. Four pieces. Additive drone with particle field. Waveguide physical modelling with bounce-to-bow transition. Granular reconstruction. A live-coded closer built from scratch in Lila during the performance. The framework and the performance were simultaneous. 0.3 is what made that set possible and what the set stress-tested.

The companion document, [The Shape of the Material](/releases/news/the_shape_of_material/), covers every major feature with production code and video from the performance. What follows is the architectural summary.

---

<div class="card-grid columns breakout">

<div class="card vertical">
<h2>Nexus: Spatial Entity Lifecycle</h2>

`Fabric`, `Wiring`, `Emitter`, `Sensor`, `Agent`. A plain object, not a subsystem, not registered with the engine. `Fabric` owns a `SpatialIndex3D` with lock-free snapshot publication. `Wiring` is a fluent builder connecting entities to the scheduler: fire every N seconds, on a key event, on an OSC message, via choreographed position steps through `EventChain`. `Agent` runs two phases per `commit()`: perception first, reading the spatial snapshot into a `PerceptionContext`, then influence, acting on render processors or audio sinks via `InfluenceContext`. `Sensor` queries without acting. `Emitter` acts without querying. `HitTest` ray casting and `SpatialIndex::all()` complete the spatial query surface.

This is not a scene graph. There is no update loop, no component system, no hierarchy. There are entities with positions, functions, and output sinks. `Fabric` coordinates when those functions run and what spatial context they receive when they do. An entity that hears its neighbours and simultaneously renders its influence on them is a few lines of `Wiring`.
</div>

<div class="card vertical">
<h2>SamplingPipeline: Post-Analog Playback</h2>

`StreamSlice` as a region descriptor. `CursorAccessProcessor` owning cursor position, loop region, a fractional frame speed accumulator, loop count, and `on_end` callback. `StreamSliceProcessor` owning a pool of these, each independently allocated on the same `DynamicSoundStream` via `allocate_dynamic_slot()`, accumulating into a single `AudioBuffer` per cycle. `SamplingPipeline` wrapping everything: `play(index, slice)`, `play_continuous`, `stop`, `build_for(ms)`, `rebuild_for(ms)`.

Speed is fractional frame accumulation across buffer boundaries, not resampling. Multiple simultaneous playback heads on the same container, each at its own speed, each in its own sub-region, mixed polyphonically. There is no analog equivalent for this architecture. The data does not move. The cursors do.
</div>

</div>

<hr/>

<div class="card-grid columns breakout">

<div class="card vertical">
<h2>GranularWorkflow: The Grammar Model Becomes Concrete</h2>

`ComputationGrammar` with named rule executors has been in Yantra's architecture since the beginning. `GranularWorkflow` is the first time it is populated with real, production content: `segment_grains`, `attribute_grains`, `sort_grains`, `reconstruct_grains`, `reconstruct_grains_additive` as named executors that the matrix selects and sequences based on data characteristics and `ExecutionContext` metadata. `STREAM` and `STREAM_ADDITIVE` output modes write reconstructed audio directly into a running `DynamicSoundStream` asynchronously. `GrainTaper` is a span lambda applied in-place per grain before accumulation.

The TOPLAP granular reconstruction piece ran on this. Not a demonstration pipeline: the actual performance code.
</div>

<div class="card vertical">
<h2>FieldOperator and Tendency: Fields into Geometry</h2>

`Tendency<D,R>` is a composable field primitive: a function over space with typed domain and range. Force field factories build on it. `FieldOperator` evaluates bound fields against each vertex, in strict order: position, color, normal, tangent, scalar, UV. `ABSOLUTE` mode restores the reference frame each evaluation, deterministic and frame-rate independent. `ACCUMULATE` mode stacks displacements, producing drift. `MeshFieldOperator` applies per-slot `FieldOperator` instances to `MeshNetwork` slots, writing results back through `set_mesh_vertices`. The UV path feeds through `UVFieldProcessor` and `uv_field.comp` into a compute shader. A field defined in C++ is the shader input, routed automatically. `NetworkTextureBuffer` makes a particle system's UV coordinates a texture.
</div>

</div>

<hr/>

<div class="card-grid columns breakout">

<div class="card vertical">
<h2>TextureContainer and WindowContainer: The GPU-CPU Boundary Disappears</h2>

`TextureContainer` is a `SignalSourceContainer` for GPU texture pixel data, making it a first-class input to the same Yantra compute infrastructure that processes audio containers. `WindowContainer` reads back live GLFW window pixels via `WindowAccessProcessor`, making rendered output a processable data source. `TextureExecutionContext` handles image binding staging and layout transitions for compute shaders operating on textures directly. GPU output becomes CPU input becomes GPU input, in whatever order the work requires.
</div>

<div class="card vertical">
<h2>Portal::Text: Text as Texture</h2>

`InkPress` with `FreeTypeContext`, `FontFace`, `GlyphAtlas`. `press` allocates a `TextBuffer` at content dimensions. `repress` replaces content, reallocating under `Fit` policy or not under `Clip`. `impress` appends incrementally into a pre-allocated budget, tracks cursor position, wraps at `render_bounds_w`, returns `Overflow` when height is exhausted, grows the budget automatically under `Grow` policy. `ink_quads` and `rasterize_quads` exposed for caller-driven rasterization when the cursor model is not wanted. `TextBuffer` is a `TextureBuffer` subclass: streaming mode and alpha blending enabled at construction, entering `setup_rendering` identically to any other texture buffer. Text driven by the same node graph that drives audio is not a separate system. It is a texture. The routing is identical.
</div>

</div>

<hr/>

<div class="card-grid columns breakout">

<div class="card vertical">
<h2>ViewTransform to UBO, Push Constants Freed</h2>

`ViewTransform` moved from push constant to UBO at `set=0, binding=0`. `set=0` is now permanently engine-reserved. User descriptors start at `set=1`. Push constants are fully free for user shader parameters. `UNIFORM_BDA` and `STORAGE_BDA` buffer usage types and `get_buffer_device_address` complete the buffer device address infrastructure.
</div>

<div class="card vertical">
<h2>`DomainSpec` and `Audio[ch]` Syntax</h2>

`DomainSpec` replaces bare `constexpr Domain` constants with `operator[]` for channel binding. `object | Audio[0]` is now the canonical syntax for channel-specific routing. `CreationProxy` is the transition path for 0.3.1.
</div>

</div>

<hr/>

<div class="card">
<h2>What Comes Next</h2>

0.3.1 is a single-purpose breaking patch: `CreationProxy` removal. Deprecated with `[[deprecated]]` and `MF_WARN` notices throughout 0.3.

0.4 is a dedicated agent and Nexus namespace build-out on top of the 0.3 foundations.

Ongoing: AUR package, Flatpak targets for `mayaflux` base and `mayaflux+lila` with LLVM 21 for the Steam Deck and Bazzite ecosystem. Rosetta Stone documentation series for users arriving from Pure Data, Max/MSP, SuperCollider, and p5.js. Institutional outreach at Sonology, CCRMA, IEM Graz, and UPF Barcelona.
</div>


<div class="card">
<h2> Commit Summary </h2>

<p>See full diff: <a href="https://github.com/MayaFlux/MayaFlux/compare/v0.2.1...v0.3.0">Github</a></p>

<pre><code>
63e7db7a (HEAD -> main, origin/main) create 0.3 release
5c722a34 feat(api): add Nexus, SamplingPipeline, Taper, and WindowContainer to umbrella header
deaa8e2c fix(text): unload default typeface before FreeType shutdown
3d6d4889 fix(nexus): skip UBO upload in invoke when no influence target is set
f3452280 fix(PhysicsOperator): add missing attraction strength setter
7ca1ef3d feat(memory): add make_persistent / make_persistent_shared to Persist.hpp
bdb80259 fix(buffers): propagate preferred buffer size through unit manager
fb72f9d3 docs(readme): update for 0.3.0 release
30671540 docs(contributing): require human authorship for all contributions
8f95154c docs: improve Doxygen config (add tagfile + better graphs & exclusions)
53efdb0a docs: archive outdated paradigm files and promote new architecture doc
8f13e408 chore(deps): update 0.3 dependencies into stable release builders
1e6e7c38 docs: update Digital_Architecture.md post-baseline corrections
5890eb17 docs: add Digital_Architecture.md as system-wide conceptual baseline
7f80cb36 (origin/0.3/feature_freeze) feat(nodes): add Counter generator node
d9fac5ee refactor(nodes): introduce TypedHook<ContextT> for typed callback dispatch
85d34213 feat(nodes): propagate frame rate through node graph for visual-rate timing
fa4e9bbb docs(Kakshya): update WindowContainer, WindowAccessProcessor and SpatialRegionProcessor
593aa0b4 fix(Kakshya): correct BGRA swapchain format mapping in WindowContainer
cd98d0eb feat(Kakshya): WindowContainer GPU bridge and interface completeness
58601966 fix(Kakshya): remove uint8_t assumption from surface processor chain
dc8c6b2b feat(Buffers): add TextureBuffer::set_gpu_texture
f4a85f7c feat(portal/text): add create_layout to public Text API
2529e221 feat(portal/text): add ink_quads for caller-driven quad rasterization
833031cb feat(portal/text): promote composite_into as public rasterize_quads
1a4da048 feat(portal/text): add create_layout to public Text API
34f9194d feat(portal/text): add codepoint field to GlyphQuad
6050d1a0 feat(portal/text): rename get_atlas to get_default_atlas, return reference
dd94d368 feat(journal): add MF_ASSERT macro
0f6901e3 feat(kakshya): add source_path and source_format to FileContainer
3e2c4364 fix(yantra/granular): correct SegmentOp frame count for interleaved containers
034fec01 feat(yantra/granular): add process_to_stream_async overloads
88078171 feat(yantra/granular): add STREAM and STREAM_ADDITIVE output paths to GranularWorkflow
7b44b7ae fix(io): normalise backslash separators in resolve_path
ae1426da fix(io): normalise cross-platform file path handling
89ac8ca5 fix(io): resolve relative file paths against cwd and source root
2fe49ce5 docs: migrate object[ch] | Audio to object | Audio[ch] syntax
5a4c94cd fix(api): infer AudioBuffer channel from get_channel_id() when context omits channel
57d3c13d docs(api): update CreationProxy removal target to 0.3.1
bfa7dda5 (origin/feature/printing_press) fix(portal/text): refactor InkPress arg surface, fix vertical layout bugs
815f1298 feat(portal/text): impress() with automatic wrapping, budget growth, and overflow
3dd53d42 fix(portal/text): handle \n in lay_out and fix impress cursor tracking
23e17f98 feat(portal): integrate Portal::Text into GraphicsSubsystem
32ef556d update(deps): Add fontconfig to linux builds
53432921 fix(windows): resolve build errors exposed by Windows CI
c1e397fe feat(portal/text): add impress() for incremental append into pre-allocated TextBuffer
1c79d7e8 feat(portal/text): add growing budget to press/repress, prep for impress
20083097 refactor(portal/text): collapse RedrawPolicy to Clip/Fit, drop Strict
1ce694fb fix(buffers): runtime texture reallocation for TextureBuffer Grow policy
22ad3452 feat(text): add press/repress API with RedrawPolicy to InkPress
880f1c2a refactor(text): move default font state into TypeFaceFoundry singleton
f998f88b feat(buffers): add TextBuffer as TextureBuffer subclass for glyph textures
4a68e303 refactor(portal/text): rename text subsystem files to match Portal vocabulary
c3cf0058 feat(text): add set_default_font overload with platform font discovery
6a2dea20 feat(platform): add find_font for system font discovery
5683de86 feat(portal/text): add default font/atlas convenience API
7e1bbd11 feat(portal/text): add RenderText
6211141d feat(portal/text): add TextLayout
48e04e73 feat(portal/text): add FreeTypeContext, FontFace, and GlyphAtlas
6352cf40 feat(deps): add FreeType and utf8proc as dependencies
9c0b0690 feat(kakshya): make TextureContainer layer-aware
34a6009b refactor(yantra): loosen Gpu op constructors to accept GpuExecutionContext
20c0328b feat(yantra): add TextureExecutionContext
8dd4a35d fix(yantra): fix image binding path in GpuDispatchCore
fa7c3079 refactor(yantra): make GpuExecutionContext input extraction overridable
f5c14eaa feat(kakshya): introduce TextureContainer as SignalSourceContainer for GPU texture pixel data
63611bc1 refactor(yantra): extract GpuDispatchCore from GpuExecutionContext
e83f1171 feat(yantra): extend GpuExecutionContext with image binding staging and output retrieval
b4131a52 feat(yantra): extend GpuResourceManager with image binding and layout transition support
b3b95367 feat(portal/graphics): add storage image creation and layout transition to TextureLoom
d16519f7 feat(transitive): add persistent store and safe teardown
68f94530 fix(linux packaging): set /usr install prefix, update Dest dir to pkg
83492940 fix(api): correct multi-channel buffer registration via pipe and deprecated subscript path
7ab78614 refactor(api): replace CreationHandle with CreationProxy for 0.3 transition
659079c5 feat(api): add CreationContext overload for shared_ptr pipe operator
f2994fe7 refactor(API): add DomainSpec for Audio[ch] channel binding syntax
9d203674 (origin/feature/sampling) fix(macos): Pedantic min/max type implicit conversion failures
48d0637a feat(kriya): add create_samplers for multichannel shared-stream construction
942bbf91 feat(kriya): add create_sampler_from_stream for multichannel shared-stream samplers
24f1cd7b feat(kriya): add bounded duration build and sampler rig duration parameter
7f49ec9f feat(kriya): add BufferPipeline::on_complete callback and pipeline timing infrastructure
31d2798b feat(kakshya): add loop_count to StreamSlice and CursorAccessProcessor
fe60c0f5 refactor(kriya): remove template parameter from StreamSliceProcessor and SamplingPipeline
aa955cbd feat(api): add Rigs.hpp with create_sampler factory
6dcc6842 feat(kakshya,buffers,kriya): polyphonic mixing, slice fluent config, play reload
306e90d1 fix(kakshya,kriya): fix CursorAccessProcessor tail read overrun and SamplingPipeline looping
dea10226 feat(kriya): add SamplingPipeline with polyphonic slice playback
e77fb2f1 fix(Buffers): add force param to supply_buffer_to to bypass channel id guard
6d8a4e5d fix(Vruta): remove double-write of next_buffer_cycle in try_resume_with_context
97d6b7bb feat(IOManager): add load_audio_bounded for single-step DynamicSoundStream loading
0a10b9d5 refactor(StreamSliceProcessor): replace peek_sequential cursor arithmetic with per-slot CursorAccessProcessor
0e1672c0 refactor(CursorAccessProcessor): isolate to dynamic slot writes, remove container state side effects
b23ee3d2 feat(CursorAccessProcessor): add variable speed playback via fractional frame accumulator
718b0f97 refactor: extract channel data extraction to ContainerUtils free function
eb4a1f05 refactor(Kakshya): replace StreamSlice cursor state with Region descriptor
76e14f9b feat(Kakshya): refactor CursorAccessProcessor to use dynamic data slots
e3ca8410 feat(Kakshya): add dynamic processed data slots to DynamicSoundStream
ada1894e fix(IO): add truncation support and correct layout handling in load_bounded
61d07702 feat(Buffers): add load overloads to StreamSliceProcessor
df7ae937 feat(Buffers): add StreamSliceProcessor BufferProcessor
b6c55468 feat(Kakshya): add CursorAccessProcessor for independent cursor-based stream reads
6933ef72 feat(Kakshya): add StreamSlice plain data struct
646864b1 feat(IO): add SoundFileReader::load_bounded for size-bounded stream loading
7f80369c feat(render): add view transform getters to RenderProcessor
4b63a267 fix(render): resolve influence UBO binding collision and descriptor phantom
789e4878 feat(shaders): add line_lit shader variant, consolidate line fallback frags
76724a49 feat(nexus,shaders): influence UBO binding and lit shader variants
3a3fb183 feat(nexus): bind influence UBO to target RenderProcessor
644f2bbe feat(nexus): add influence properties and target to Emitter and Agent
bd60985a feat(nexus): expand InfluenceContext with intensity, radius, color, size, render_proc
f4d90b74 feat(nexus): replace set_geometry with typed set_vertices API
35acbd0a feat(geometry): add raw vertex upload path to GeometryWriteProcessor
51a4435f feat(geometry): add MESH write mode to GeometryWriteProcessor
d33126b6 feat(nexus): accept RenderConfig in render sinks, push geometry on set_position
b193a3c6 feat(nexus): add output sink capability to Emitter and Agent
be575695 feat(nexus): entity lifecycle layer : Fabric, Wiring, Emitter, Sensor, Agent
5081329f feat(nexus): spatial entity layer with Fabric orchestrator and Wiring builder
7211294e chore(cleanup): Clang tidy updates to Kriya namespace
28557e96 refactor(Granular): consolidate pipeline API around GranularConfig
7c85f41c feat(ComputeMatrix): add with_async for deferred chain execution
9eacf8b3 fix(lila): replace async_read_until with async_read_some to preserve multi-line blocks
6ae3c470 update(API): Add remove processor shorthands
315c25a2 feat(api): add prepare_audio_buffers for deferred playback registration
6d2aeb83 Update(API): create_line to use 'retain' flag instead of 'loop'
2f46984d feat(api): add ViewportPresetMode enum to bind_viewport_preset
f76331dd fix(Nodes): initialize FieldOperator on reset in Particle and PointCloud networks
73397ebe fix(Vruta): prevent unbounded event queue growth under filtered waiters
09fd169b build: add MAYAFLUX_SHIP_DEV and MAYAFLUX_SHIP_REL packaging modes
8d3b606f feat(kinesis): add HitTest ray casting and SpatialIndex::all()
6f763c09 feat(kinesis): add SpatialIndex with lock-free snapshot publication
ff644ff5 refactor(kinesis): move ProximityGraphs into Kinesis/Spatial subfolder
eb03e08c docs(rendering): document m_engine_owns_set_zero ownership and subclass contract
bf5e49a7 refactor(buffers): replace engine_internal flag with EngineContext on VKBuffer
45f9f303 fix(rendering): complete bind_texture path for engine-owned set=0 bindings
5756806c fix(macos): enable bufferDeviceAddress via vk::StructureChain in VKDevice
74d3091d refactor(rendering): consolidate set=0 as engine-owned; user descriptors at set=1+
752e9673 feat(registry): add get_buffer_device_address to BufferService and BackendResourceManager
230f978f feat(buffers): add UNIFORM_BDA and STORAGE_BDA usage types to VKBuffer
fa8a7536 feat(core): enable bufferDeviceAddress via vk::StructureChain in VKDevice
7e4286bd refactor(rendering): move ViewTransform from push constant to UBO at set=0, binding=0; reserve set=0 for engine, user descriptors at set=1+
fe0bb48b feat(buffers/shaders): per-slot texture array path for MeshNetworkBuffer
48428513 feat(shaders): add descriptor array count support to ShaderBinding and RenderProcessor
4bf583d2 feat(buffers): add per-slot SSBO transform path to MeshNetworkProcessor
1b79b433 feat(shaders): add mesh_network vert/frag pair for MeshNetwork SSBO transform path
0bbd4361 feat(nodes): add MeshOperator base, MeshTransformOperator, MeshFieldOperator; update MeshNetwork::process_batch
2113e886 feat(API): add read_mesh_network to Creator/vega
f2cde765 register MeshNetwork and MeshNetworkBuffer in proxy and public header
289c8e14 add TextureResolver to ModelReader; add load_mesh_network to IOManager
17698036 feat(Mesh): MeshNetworkProcessor and MeshNetworkBuffer
ba8b4f98 feat(Network): MeshSlot and MeshNetwork
7197352b add OperatorChain to NodeNetwork
6afb4d96 fix(WaveguideNetwork): route sample output through scratch buffer
1fa08bdf refactor(shaders): unify view and non-view shader variants
cd754ab6 feat(RenderProcessor): auto-detect ViewTransform push constant via reflection
b8118a3e refactor(Buffers): add VKBufferProcessor::ensure_initialized
3928cea9 refactor(Buffers): replace open-coded grow+upload with upload_resizing
5f40b12f quickfix(ubuntu): wrong deps syntax
66eb3de9 feat(api): ViewportPreset -> window-level fly-navigation binding
0ff8e78c fix(kinesis): frame-rate independent movement in compute_view_transform
9ba041b1 refactor(api): Windowing.cpp delegates screen-space math to Kinesis
e194dcb9 feat(kinesis): add screen-space math to ViewTransform.hpp
ee4da059 feat(kinesis): NavigationState : first-person fly-navigation primitives
40026396 fix(buffers): guard ViewTransform push via m_has_view_transform_slot
dee98b83 refactor(buffers): virtual get_render_processor() on VKBuffer base
43674ce9 feat(api): vega.read_mesh() returning MeshGroupHandle
c92d32f7 update(io): ModelReader::create_mesh_buffers() and IOManager::load_mesh()
dadaf33a feat(API): Expose MeshBuffer to Registry and Creator/vega
6618e189 update(io/kakshya): extract diffuse texture path into submesh Region
41de0756 update(buffers/shaders): texture support to MeshBuffer
48914160 feat(io): ModelReader with Assimp backend
fa55baaa build(deps): add assimp to all scripts and linker targets
cb861acc feat(buffers): MeshBuffer and MeshProcessor
a2eb7e34 feat(kakshya): add MeshData owning struct to NDData layer
6f03d75e feat(kakshya): MeshAccess and MeshInsertion to NDData layer
2692d270 feat(api): expose MeshWriterNode in public API and headers
9f2750a8 feat(buffers): GeometryBuffer::setup_rendering inherits topology from node
771838c2 feat(shaders): add triangle.vert and triangle.frag for MeshVertex pipeline
664c38c2 feat(buffers): indexed draw path in RenderProcessor
dee92239 fix(portal): RenderFlow::bind_index_buffer reads index handle from VKBufferResources
c389167d feat(buffers): index buffer upload in GeometryBindingsProcessor
eefd808e feat(buffers): add index buffer resources to VKBufferResources
b9601524 feat(nodes): add MeshWriterNode for indexed triangle mesh geometry
9f5585a8 feat(nodes): add index buffer storage to GeometryWriterNode
6b8b64c0 feat(kakshya,kinesis): add MeshVertex support to VertexAccess and VertexSampler
727ca2b0 feat(Geometry): MeshVertex type with universal 60-byte layout
882dd909 quickfix(windows): use vcpkg names instead of pkgconfig
ef9f30d5 refactor(lila): replace hand-rolled socket server with asio
c4eb8a52 build: define _WIN32_WINNT=0x0A00 globally on Windows
83ca28d3 build(lila): add direct asio dependency for Server TCP rewrite
60388f64 build: replace global GLOB_RECURSE with per-namespace source lists
4b903fe7 remove(nodes): delete TextureFieldOperator, superseded by FieldOperator UV target
06af77bc fix(ci): install asio separately instead of cache
6ad4e5a5 feat(nodes): implement UV target in FieldOperator
ca37e628 feat(api): expose NetworkTextureBuffer in public API and umbrella header
c8bc67aa fix(shaders): rewrite uv_field.comp SSBO as flat float[] with scalar layout
456a62e8 fix(buffers): create sampler in on_descriptors_created, not in set_texture
ba54c3c4 fix(buffers): call on_before_execute before push constant submission in ComputeProcessor
ee4e1a22 refactor(buffers): NetworkTextureBuffer as NetworkGeometryBuffer subclass
f439a1bb update(buffers): expose NetworkGeometryBuffer members as protected
2624c91b update(nodes): register TextureFieldOperator in ParticleNetwork and PointCloudNetwork
dc951a79 feat(buffers): add NetworkTextureBuffer
daa7424a feat(nodes): TextureFieldOperator for CPU-side UV generation
b8392865 feat(buffers): UVFieldProcessor and uv_field.comp
b7e6817c feat(kinesis): add UVField tendency alias and UV projection factories
feb5f764 feat(Nodes): implement TANGENT target in FieldOperator
7fa1a1ab feat(Nodes): implement COLOR and NORMAL targets in FieldOperator
4fbbc948 feat(Nodes): add FieldOperator for Tendency-driven vertex manipulation
00735370 feat(Kinesis): add Tendency force field factories, integrate into PhysicsOperator
88fb35ce feat(Kinesis): add Tendency<D,R> composable field primitive
56cd3e90 feat(vertex): add uv, normal, tangent to PointVertex and LineVertex
b1669072 fix(atomic): remove clang bound initialization
d749bc3b update(API): Expose all transfer processors to global scope
67c5989b fix(Kriya): route-to-buffer via AudioWriteProcessor, remove direct write_to_buffer in pipeline
b0b09d08 feat(Buffers): add GeometryWriteProcessor
dcf196c3 update(Kakshya): add as_point_vertex_access and as_line_vertex_access
f4e5b8ea feat(Kakshya): add VertexAccess: DataVariant to vertex buffer view
1c9b4b80 fix(Kakshya): improve DataUtils inference correctness
20703b34 fix(TextureProcessor): re-bind texture to RenderProcessor after on_attach
d913f208 feat(TextureWriteProcessor): TextureProcessor subclass fed by external DataVariant
33a8f9ff refactor(TextureProcessor): add PixelSource enum for extensible pixel supply
099aa02b update(TextureLoom): create_2d(DataVariant) overload
d11b7ba4 feat(TextureAccess) NDData layer: DataVariant → texel memory description
f0602f26 feat(AudioWriteProcessor): unconditional external data write into AudioBuffer
ad988c60 Portal::Network: add MessageUtils (as_osc, serialize_osc)
ddf04902 OSCNode: replace extract_from_arg with OSCMessage::get_float/get_int
41e88bd4 InputBinding: safe typed accessors on InputValue and OSCMessage
349217a8 refactor(OscParser): delegate byte primitives to Transitive::Protocol::BinaryBuffer
96f3fa68 feat(NetworkSubsystem): wire Portal::Network lifecycle
4e4c0ef8 draft(Portal::Network): establish namespace skeleton and NetworkSink
5c15691e fix(NetworkSource): deliver to waiters OR queue, never both
8a097dcd fix(UDPBackend): exclude SEND endpoints from receive broadcast
2c06a1e2 feat(kriya,api): add NetworkEvents coroutine factories and Chronie network handlers
0bf88e13 update(GetEventPromise): NetworkSource lifetime bound to coroutine frame
125a30a4 feat(NetworkSource/NetworkAwaiter): broadcast coroutine receive
91e54290 move NetworkMessage to Core::GlobalNetworkConfig
1a9affd0 fix(UDP): broadcast datagrams to all endpoints on matching local port
cfeaec9a OSC bridge wiring: InputSubsystem calls setup_osc_bridge on init
ef7dbb2e feat(OSCNode): InputNode for OSC message argument extraction
24c68dfc jthread platform detection: centralize in config.h.in
3862ffac update(build): Add copr asio-standalone to fedora deps and ci
55e4ccf8 feat(OSC bridge): InputManager routes UDP via NetworkService
1fcb07f2 update(Network subsystem): engine integration
f5fb87b7 feat(Network subsystem): service registration and endpoint routing
7968ec77 feat(Network): transport backends and configuration
22efa626 update(build): integrate standalone Asio and enforce No-Boost policy
4109258c fix(kriya,buffers): call initialize() in SoundFileBridge::setup_processors; clarify ROUTE batch semantics
4ab442af fix(NodeNetwork): spinlock-guard m_last_audio_buffer for async graphics reads
9ecca14e fix(DescriptorBindingsProcessor): add missing break in NODE source type dispatch
102197f0 update(arch script): Do not target non -dev releases
880dcdf4 feat(workflows): add workflow opt-in system, move GranularWorkflow to Yantra/Workflows/Granular/
f267edaf refactor(granular): extract shared utilities to Kakshya and Yantra layers
966be4bb update(granular): phase 4: additive reconstruction with per-grain tapering
dc064536 feat(kinesis): add Taper: window coefficient generation and in-place application
886ca99a update(yantra/granular): phase 3: layout-aware extraction and single fetch per grain
b6eac3c5 Yantra/Granular: offline container output workflow: first full grain-to-audio path
df865843 feat(Yantra): introduce GranularWorkflow grammar (phase 1)
8c6c591d feat(Yantra): add named rule overload to execute_with_grammar
e3f7beaf fix(Yantra): guard StatisticalAnalyzer::analyze_implementation empty() check
d8424b53 promote analyze/extract methods to accept Datum directly as primary
e379cdd2 refactor(ComputeMatrix): remove raw-T convenience overloads
689cea54 feat(Yantra): add to_io_batch free function to DataIO.hpp
0ce6db51 update(ComputeOperation): Drop struct qualification warn to info
5e947c35 refactor(tests/yantra): migrate all Yantra tests to Datum-primary API
440d4310 refactor(yantra): migrate execute_with_grammar and FluentExecutor to Datum-primary API
e2fb1ec9 refactor(yantra): migrate ComputeMatrix and FluentExecutor to Datum-primary API
bbd4f3b0 refactor(API): migrate gain/normalize/reverse to Kinesis::Discrete
964fb88c refactor(yantra, kakshya): replace std::any_cast with safe_any_cast system
2193118b update(Grammar): Add type alias for grammar rule executor
ce61fc43 fix(ShaderFoundry): Destroy vkshader object when destroying shader source.
828b9f9e feat(yantra): add ergonomic helpers to ExecutionContext
55d73f89 test(Yantra): add ComputationPipeline and ComputationGrammar unit tests
4dedd823 refactor(API): replace per-call operator allocations with static locals
bad29d5d refactor(yantra): remove GpuComputeOperation and ShaderOperation
401bfe94 fix(yantra/executors): handle INPUT_OUTPUT readback when inputs are pre-staged
9bfef519 feat(yantra): add GpuTransformer, GpuAnalyzer, GpuExtractor, GpuSorter
8e539e66 feat(yantra/executors): add ergonomic API to ShaderExecutionContext
72e9e631 feat(yantra): add get_operation_type() to ComputeOperation hierarchy
d9d3d0a0 feat(yantra): add ShaderExecutionContext as concrete GpuExecutionContext
60d35518 feat(yantra): add GpuExecutionContext backend slot to ComputeOperation
48e53ca7 refactor(yantra): introduce GpuExecutionContext as standalone GPU executor
2fae9c76 refactor(yantra): extract GpuResourceManager to Executors/GpuResourceManager.hpp/.cpp
e7b5b7ac feat(gpu): add CHAINED execution mode for multi-pass batched dispatch
2e00c9df refactor(GpuComputeOperation): add per-binding data staging and independent output sizing
d1c75c57 feat(Yantra/shaders): add example compute shaders for ShaderOperation
a969babf feat(Yantra): add ShaderOperation concrete GpuComputeOperation
0290ece1 feat(Yantra): add GpuComputeOperation and GpuResourceManager
bec040e4 update(ShaderFoundry): Expose logical and physical device query
887073aa refactor(Eigen): rotation/scaling matrix factories to Kinesis
57af8497 refactor(Yantra): Phase II: Data conversion helpers
2cfe6df8 refactor(Transform)par_unseq to interpolate_linear and interpolate_cubic
faad982e Refactor ConvolutionTransformer to use Kinesis::Discrete primitives
30c6e88f fix(Yantra): Reintroduce a unified apply_per_channel helper.
c3e2ddb4 refactor(Yantra):Mathematical+Temporal: replace helpers with Kinesis::Discrete::Transform
a2e36a9e refactor(Yantra): replace SpectralHelper with Kinesis::Discrete::Spectral
083f9273 feat(Kinesis): add STFT primitive layer
2f02fa69 kinesis/discrete: add Transform primitive layer
31e9cece yantra/extractors: rewrite ExtractionHelper on Kinesis::Discrete primitives
a8345828 kinesis/discrete: add Extract primitive layer
027f79cc refactor(Yantra): migrate Sort math to Kinesis::Discrete, collapse SortingHelper
ff3d94ba fix(macos): Apple still does not support std::ranges::iota
1a86f03c feat(Kinesis): introduce Kinesis::Discrete::Analysis and port analyzer math
e552d24c start v0.3.0-dev tag
</code></pre>

</div>

---

**Source**: [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux)  
**Docs**: [mayaflux.org](https://mayaflux.org)  
**License**: GPLv3
