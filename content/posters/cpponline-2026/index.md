# MayaFlux: Multi-Domain Real-Time Scheduling with C++20

**Ranjith Hegde** · [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux) · [mayaflux.org](https://mayaflux.org)

C++Online 2026 · Poster

---

## The Problem

Real-time multimedia systems must run independent execution contexts simultaneously: audio callbacks at 48 kHz, graphics render loops at 60 Hz, async input polling, and user-defined coroutines with arbitrary timing. Existing approaches either collapse these into a single scheduler (losing domain autonomy) or isolate them entirely (losing coordination).

MayaFlux is an open-source C++20/23 framework that lets these contexts coexist in a single process without a unifying abstraction. Each domain keeps its own timing, thread model, and processing semantics. Coordination happens through lock-free patterns, compile-time type unification, and coroutine-based temporal weaving — not shared state behind a mutex.

---

## 1 · Lock-Free Coordination Across Thread Boundaries

### The Pattern: CAS-Based Pending Operations

The central challenge: nodes, buffers, and processors must be added and removed dynamically while real-time threads are actively processing. MayaFlux solves this with a fixed-size pending operation array scanned via `compare_exchange_strong`:

```cpp
// RootNode::register_node — called from any thread
void RootNode::register_node(const std::shared_ptr<Node>& node)
{
    if (m_is_processing.load(std::memory_order_acquire)) {

        // Try to claim a pending slot via CAS
        for (auto& m_pending_op : m_pending_ops) {
            bool expected = false;
            if (m_pending_op.active.compare_exchange_strong(
                    expected, true,
                    std::memory_order_acquire,
                    std::memory_order_relaxed)) {
                m_pending_op.node = node;
                atomic_add_flag(node->m_state, NodeState::INACTIVE);
                m_pending_count.fetch_add(1, std::memory_order_relaxed);
                return;
            }
        }

        // All slots full — wait for processing to finish
        while (m_is_processing.load(std::memory_order_acquire)) {
            m_is_processing.wait(true, std::memory_order_acquire);
        }
    }

    m_Nodes.push_back(node);
    atomic_add_flag(node->m_state, NodeState::ACTIVE);
}
```

This pattern appears identically in `RootBuffer::add_child_buffer` and `BufferProcessingChain::queue_pending_processor_op` — the same CAS-claim → deferred-apply structure is used for nodes, buffers, and processors. No mutexes anywhere in the processing path.

### Atomic State Flags

Node state is managed through atomic bitfield operations using CAS loops. Multiple state transitions (ACTIVE, INACTIVE, PROCESSED, PENDING_REMOVAL) coexist without locking:

```cpp
void atomic_add_flag(std::atomic<NodeState>& state, NodeState flag)
{
    auto current = state.load();
    NodeState desired;
    do {
        desired = static_cast<NodeState>(current | flag);
    } while (!state.compare_exchange_weak(current, desired,
        std::memory_order_acq_rel,
        std::memory_order_acquire));
}
```

### Multi-Buffer Snapshot Coordination

When multiple buffers reference the same node, only one should trigger `save_state` per processing cycle. This is handled by atomic context claiming:

```cpp
bool Node::try_claim_snapshot_context(uint64_t context_id)
{
    uint64_t expected = 0;
    return m_snapshot_context_id.compare_exchange_strong(
        expected, context_id,
        std::memory_order_acq_rel,
        std::memory_order_acquire);
}
```

The first buffer to claim the context saves/restores state; subsequent buffers detect the active snapshot and use the already-computed output. Zero contention.

### Processing Cycle Fence

The processing cycle itself is guarded by a single atomic bool with `wait`/`notify_all` (C++20):

```cpp
bool RootNode::preprocess()
{
    bool expected = false;
    if (!m_is_processing.compare_exchange_strong(expected, true,
            std::memory_order_acquire, std::memory_order_relaxed)) {
        return false;
    }

    if (m_pending_count.load(std::memory_order_relaxed) > 0) {
        process_pending_operations();
    }
    return true;
}

void RootNode::postprocess()
{
    for (auto& node : m_Nodes) {
        node->request_reset_from_channel(m_channel);
    }
    m_is_processing.store(false, std::memory_order_release);
    m_is_processing.notify_all();
}
```

The entire node graph processes inside a `preprocess` → compute → `postprocess` window. Pending operations are drained at the boundary. External threads that attempt registration during processing either claim a pending slot (fast path) or `wait` on the atomic (slow path, rare).

---

## 2 · C++20 Coroutines Weaving Temporal Intent

### The Architecture: Vruta (Infrastructure) + Kriya (Patterns)

MayaFlux splits coroutine concerns into two namespaces. `Vruta` provides the scheduling infrastructure — `TaskScheduler`, `SoundRoutine` (the coroutine type), domain clocks. `Kriya` provides the creative temporal patterns — awaiters that encode timing intent.

### Sample-Accurate Delay

The fundamental awaiter is `Kriya::SampleDelay`. It advances the coroutine's resumption point by an exact number of discrete time units:

```cpp
struct SampleDelay {
    using promise_handle = Vruta::audio_promise;
    uint64_t samples_to_wait;

    [[nodiscard]] bool await_ready() const { return samples_to_wait == 0; }
    void await_resume() { }

    void await_suspend(std::coroutine_handle<Vruta::promise_type> h)
    {
        auto& promise = h.promise();
        promise.next_sample += samples_to_wait;
    }
};
```

Because `next_sample` is updated inside the coroutine frame and the `TaskScheduler` compares it against the current clock position, timing is deterministic: it does not depend on wall-clock jitter, thread scheduling, or processing load. One `co_await SampleDelay{48000}` is exactly one second at 48 kHz, every time.

### Clock Domains and Token-Based Routing

`TaskScheduler` maintains separate clocks for each `ProcessingToken`:

```cpp
scheduler->process_token(Vruta::ProcessingToken::SAMPLE_ACCURATE, 1024);
scheduler->process_token(Vruta::ProcessingToken::FRAME_ACCURATE, 1);
```

Each call advances the corresponding clock and resumes all coroutines whose `next_sample` or `next_frame` has been reached. Different coroutines can exist in different temporal domains — an audio-rate coroutine ticking at sample resolution, a graphics-rate coroutine at frame resolution — and they never interfere because domain advancement is explicit and token-routed.

### Composable Temporal Patterns

Beyond raw delays, `Kriya` provides higher-level awaiters that compose into complex temporal structures:

```cpp
auto routine = [](Vruta::TaskScheduler& scheduler) -> Vruta::SoundRoutine {
    auto& promise = co_await Kriya::GetAudioPromise{};

    while (true) {
        if (promise.should_terminate) break;

        // Gate: suspend until a logic node outputs true
        co_await Kriya::Gate{scheduler, []() {
            schedule_another_clock();
        }, logic_node, true};

        // Dynamic timing from musical state
        float wait_time = calculate_musical_timing();
        co_await Kriya::SampleDelay{scheduler.seconds_to_samples(wait_time)};

        // Trigger: fire callback and synchronize
        co_await Kriya::Trigger{scheduler, true, []() {
            sync_frame_clock();
        }, sync_node};
    }
};
```

`Gate`, `Trigger`, `SampleDelay`, `MultiRateDelay` — each is a standalone awaiter. The coroutine body reads as a sequential description of temporal intent. The scheduler handles the mechanics of when to resume. State is stored in the coroutine frame itself and accessible externally via `promise.get_state<T>()` / `promise.set_state()`.

### Convenience API and Fluent Integration

For common patterns, convenience wrappers eliminate boilerplate:

```cpp
MayaFlux::schedule_metro(2.0, []() { modulate_filter_cutoff(); }, "main_clock");

MayaFlux::schedule_pattern([](uint64_t beat) {
    return beat % 8 == 0;
}, []() { change_distribution(); }, 1.0, "pattern_trigger");

auto shape = vega.Polynomial({0.1, 0.5, 2.f});
shape >> Time(2.f);  // Creates coroutine, registers with TaskScheduler
```

Each of these constructs a `SoundRoutine` internally. `schedule_metro` builds a `SampleDelay` loop. `schedule_pattern` builds a conditional awaiter. The fluent `>>` operator on nodes creates `NodeTimer` coroutines that are automatically registered with the appropriate domain.

---

## 3 · Compile-Time Data Unification

### The Foundation: Universal Concepts in pch.h

MayaFlux defines concepts at the lowest level of the type hierarchy — in the precompiled header — so they're available everywhere without include chains:

```cpp
template <typename T>
concept ArithmeticData = std::is_integral_v<T> || std::is_floating_point_v<T>;

template <typename T>
concept ComplexData = requires {
    typename T::value_type;
    std::is_same_v<T, std::complex<float>> || std::is_same_v<T, std::complex<double>>;
};

template <typename T>
concept ContiguousContainer = requires(T t) {
    { t.data() } -> std::convertible_to<typename T::value_type*>;
    { t.size() } -> std::convertible_to<std::size_t>;
    typename T::value_type;
};

template <typename From, typename To>
concept SafeArithmeticConversion = ArithmeticData<From> && ArithmeticData<To>
    && (SafeIntegerConversion<From, To>
     || SafeDecimalConversion<From, To>
     || (IntegerData<From> && DecimalData<To>));
```

These are not audio concepts or graphics concepts. They describe data. Any function constrained by `ArithmeticData` works identically whether the data represents audio samples, pixel intensities, control voltages, or physics simulation parameters.

### DataVariant: The Universal Container

`DataVariant` is a `std::variant` over every numeric storage type the framework supports:

```cpp
using DataVariant = std::variant<
    std::vector<double>,
    std::vector<float>,
    std::vector<uint8_t>,
    std::vector<uint16_t>,
    std::vector<uint32_t>,
    std::vector<std::complex<float>>,
    std::vector<std::complex<double>>,
    std::vector<glm::vec2>,
    std::vector<glm::vec3>,
    std::vector<glm::vec4>,
    std::vector<glm::mat4>
>;
```

This is the type-erased boundary between domains. Audio data arrives as `vector<double>`, image data as `vector<uint8_t>` or `vector<float>`, vertex positions as `vector<glm::vec3>` — all held in the same container type. The framework doesn't care about the origin domain; it cares about the data shape.

### Semantic Access Without Templates on Containers

`DataAccess` and `DataInsertion` provide type-erased read/write over `DataVariant` with semantic dimension metadata:

```cpp
DataAccess access(variant, dimensions, DataModality::AUDIO_1D);

auto audio_view = access.view<double>();      // span<double>
auto gpu_info   = access.gpu_buffer();        // (void*, byte_count, format_hint)
auto elems      = access.element_count();
auto comps      = access.component_count();   // 1 for scalar, 3 for vec3, etc.
```

Containers stay template-free. The template parameter lives on the `view<T>()` call — the user selects their access type, and `DataAccess` handles conversion if needed, caching the result for the lifetime of the accessor.

### Extraction Traits: Compile-Time Data Routing

`extraction_traits_d` is a traits struct specialized per input type that tells the extraction system how to handle it at compile time:

```cpp
template <> struct extraction_traits_d<Kakshya::DataVariant> {
    static constexpr bool is_multi_variant = false;
    static constexpr bool requires_container = false;
    static constexpr bool is_region_like = false;
    using result_type = std::span<double>;
};

template <> struct extraction_traits_d<Kakshya::Region> {
    static constexpr bool is_multi_variant = true;
    static constexpr bool requires_container = true;
    static constexpr bool is_region_like = true;
    using result_type = std::vector<std::span<double>>;
};
```

The `ComputeData` concept (in `DataSpec.hpp`) then unifies everything that can flow through the computation pipeline:

```cpp
template <typename T>
concept ComputeData =
    std::same_as<T, Kakshya::DataVariant> ||
    std::same_as<T, std::vector<Kakshya::DataVariant>> ||
    std::same_as<T, std::shared_ptr<Kakshya::SignalSourceContainer>> ||
    std::same_as<T, Kakshya::Region> ||
    std::same_as<T, Kakshya::RegionGroup> ||
    std::same_as<T, std::vector<Kakshya::RegionSegment>> ||
    std::is_base_of_v<Eigen::MatrixBase<T>, T> ||
    VariantVector<T> ||
    std::constructible_from<Kakshya::DataVariant, T>;
```

Any `ComputeOperation<InputType, OutputType>` is constrained by `ComputeData`. Extractors, transformers, and analyzers are all templated on these types. The same `UniversalExtractor<DataVariant, Eigen::MatrixXd>` can extract a matrix from audio spectral data or from a point cloud — the operation doesn't know or care.

### Type-Safe Conversion with Auditable Safety

The `try_convert<To>(from)` system returns a `CastResult<T>` that carries the converted value, any error, and a `precision_loss` flag:

```cpp
template <typename To, typename From>
CastResult<To> try_convert(const From& value)
{
    CastResult<To> result;
    if constexpr (SafeArithmeticConversion<From, To>) {
        result.value = static_cast<To>(value);
        result.precision_loss = (sizeof(From) > sizeof(To));
    } else if constexpr (ComplexData<From> && ArithmeticData<To>) {
        result.value = static_cast<To>(std::abs(value));
        result.precision_loss = (std::imag(value) != 0);
    }
    // ...
    return result;
}
```

No silent truncation. No unchecked casts. The conversion safety is expressed through concepts at compile time and diagnosed through `CastResult` at runtime.

---

## 4 · Multi-Domain Coexistence: A Working Example

### Processing Tokens: Independent Execution Contexts

Each subsystem (Nodes, Buffers, Coroutines) has its own `ProcessingToken` enum. These compose into unified `Domain` values via bitfield composition:

```cpp
enum Domain : uint64_t {
    AUDIO = (uint64_t(Nodes::ProcessingToken::AUDIO_RATE) << 32)
          | (uint64_t(Buffers::ProcessingToken::AUDIO_BACKEND) << 16)
          | (uint64_t(Vruta::ProcessingToken::SAMPLE_ACCURATE)),

    GRAPHICS = (uint64_t(Nodes::ProcessingToken::VISUAL_RATE) << 32)
             | (uint64_t(Buffers::ProcessingToken::GRAPHICS_BACKEND) << 16)
             | (uint64_t(Vruta::ProcessingToken::FRAME_ACCURATE)),
};
```

Each domain is decomposable: you can extract the node token, buffer token, or scheduler token independently and compose custom domains from individual tokens.

### Unified RingBuffer: Policy-Driven, Zero-Cost Dispatch

The `Memory::RingBuffer` is a single template parameterized by three orthogonal policies:

```cpp
template <typename T, typename StoragePolicy,
          typename ConcurrencyPolicy = SingleThreadedPolicy,
          typename AccessPattern = QueueAccess>
class RingBuffer;

// Lock-free input queue: real-time thread → worker thread
using InputQueue = RingBuffer<InputValue,
    FixedStorage<InputValue, 4096>, LockFreePolicy, QueueAccess>;

// Audio delay line: single-threaded DSP
using AudioDelay = RingBuffer<double,
    DynamicStorage<double>, SingleThreadedPolicy, HistoryBufferAccess>;

// Real-time logging: audio thread → disk writer
using LogBuffer = RingBuffer<RealtimeEntry,
    FixedStorage<RealtimeEntry, 8192>, LockFreePolicy, QueueAccess>;
```

All dispatch is compile-time via `if constexpr`. `LockFreePolicy` uses `alignas(64)` atomic indices with acquire/release ordering. `SingleThreadedPolicy` uses plain `size_t`. `HistoryBufferAccess` reverses the push direction so `operator[]` with index 0 returns the newest sample — natural indexing for difference equations (`y[n-1]`, `y[n-2]`).

### Audio + Graphics in a Single Composition

```cpp
void compose() {
    // Audio domain: modal synthesis driven by logic
    auto bell = vega.ModalNetwork(12, 220.0,
        ModalNetwork::Spectrum::INHARMONIC)[0] | Audio;

    auto osc = vega.Sine(0.2, 1.0f);
    auto logic = vega.Logic([](double input) {
        static double last = 0.0;
        bool crossed = (last < 0.0) && (input >= 0.0);
        last = input;
        return crossed;
    });
    osc >> logic;

    logic->on_change_to(true, [bell](auto& ctx) {
        bell->excite(get_uniform_random(0.5f, 0.9f));
        bell->set_fundamental(get_uniform_random(220.0f, 1000.0f));
    });

    // Graphics domain: Vulkan point collection
    auto window = MayaFlux::create_window({"Bell Visualization", 1280, 720});
    auto points = vega.PointCollectionNode(500) | Graphics;
    auto geom = vega.GeometryBuffer(points) | Graphics;
    geom->setup_rendering({.target_window = window});
    window->show();

    // Cross-domain coordination via metro coroutine
    MayaFlux::schedule_metro(0.016, [points]() {
        // Runs at ~60Hz, reads audio state, writes to graphics node
        float x = std::cos(angle) * radius;
        float y = std::sin(angle) * radius;
        points->add_point(Nodes::GpuSync::PointVertex{
            .position = glm::vec3(x, y, 0.0f),
            .color = glm::vec3(brightness, brightness * 0.8f, 1.0f),
            .size = 8.0f + radius * 4.0f
        });
    });
}
```

Audio processing runs at 48 kHz via `AUDIO_BACKEND` token. Graphics rendering runs at display refresh via `GRAPHICS_BACKEND` token. The metro coroutine bridges the two — its callback reads audio-domain state and writes to a graphics-domain node. The `| Audio` and `| Graphics` operators route nodes and buffers to their respective `RootNode` and `RootBuffer` hierarchies. No shared locks. Each domain processes its own graph independently; the coroutine provides temporal coordination.

---

## 5 · Thread Architecture: Where the Boundaries Actually Are

### Engine: Composition, Not Unification

`Engine::Init` constructs five independent managers — `NodeGraphManager`, `BufferManager`, `TaskScheduler`, `WindowManager`, `InputManager` — then hands them to a `SubsystemManager` that creates typed subsystems, each with its own thread model:

```cpp
void Engine::Init(const GlobalStreamInfo& streamInfo,
                  const GlobalGraphicsConfig& graphics_config,
                  const GlobalInputConfig& input_config)
{
    m_scheduler = std::make_shared<Vruta::TaskScheduler>(streamInfo.sample_rate);
    m_buffer_manager = std::make_shared<Buffers::BufferManager>(
        streamInfo.output.channels, /* ... */ streamInfo.buffer_size);
    m_node_graph_manager = std::make_shared<Nodes::NodeGraphManager>(
        streamInfo.sample_rate, streamInfo.buffer_size);
    m_window_manager = std::make_shared<WindowManager>(graphics_config);
    m_input_manager = std::make_shared<InputManager>();

    m_subsystem_manager = std::make_shared<SubsystemManager>(
        m_node_graph_manager, m_buffer_manager, m_scheduler,
        m_window_manager, m_input_manager);

    m_subsystem_manager->create_audio_subsystem(streamInfo);
    m_subsystem_manager->create_graphics_subsystem(graphics_config);
    m_subsystem_manager->create_input_subsystem(input_config);
}
```

Each subsystem receives a `SubsystemProcessingHandle` — a scoped interface containing token-typed handles for buffers, nodes, and the scheduler. The handle constrains what a subsystem can touch: the audio subsystem's handle routes through `AUDIO_BACKEND` / `AUDIO_RATE` / `SAMPLE_ACCURATE` tokens; the graphics subsystem's handle routes through `GRAPHICS_BACKEND` / `VISUAL_RATE` / `FRAME_ACCURATE`. Same managers, different views.

### AudioSubsystem: Hardware-Driven Clock

The audio callback is the only externally-driven timing source. RtAudio fires it at hardware interrupt rate. Inside, the processing follows a strict sequence:

```cpp
int AudioSubsystem::process_output(double* output_buffer, unsigned int num_frames)
{
    m_callback_active.fetch_add(1, std::memory_order_acquire);

    // 1. Buffer-cycle tasks (metro callbacks, scheduled events)
    m_handle->tasks.process_buffer_cycle();

    // 2. Per-channel: process buffer chains, then collect network outputs
    for (uint32_t channel = 0; channel < num_channels; channel++) {
        m_handle->buffers.process_channel(channel, num_frames);
        all_network_outputs[channel] =
            m_handle->nodes.process_audio_networks(num_frames, channel);
        buffer_data[channel] = m_handle->buffers.read_channel_data(channel);
    }

    // 3. Per-sample: advance scheduler, sum node + buffer + network outputs
    for (size_t i = 0; i < num_frames; ++i) {
        m_handle->tasks.process(1);  // Advance coroutines by 1 sample

        for (size_t j = 0; j < num_channels; ++j) {
            double sample = m_handle->nodes.process_sample(j)
                          + buffer_data[j][i];

            for (const auto& net_buf : all_network_outputs[j]) {
                if (i < net_buf.size()) sample += net_buf[i];
            }

            output_span[i * num_channels + j] = std::clamp(sample, -1., 1.);
        }
    }

    // 4. Drain deferred operations (CAS pending slots)
    m_handle->nodes.cleanup_completed_routing();
    m_handle->buffers.cleanup_completed_routing();

    m_callback_active.fetch_sub(1, std::memory_order_release);
    return 0;
}
```

This is where the lock-free patterns from Section 1 pay off. Step 3 calls `process_sample()` on every node — which internally hits `RootNode::preprocess()` / `postprocess()` and its CAS-based pending operation drain. Step 1 calls `process_buffer_cycle()` which advances buffer-cycle-scoped coroutines. The scheduler's `process(1)` in Step 3 advances sample-accurate coroutines by exactly one sample unit per iteration. No thread synchronization during the inner loop — all coordination was handled by the pending-op mechanism at cycle boundaries.

### GraphicsSubsystem: Self-Driven Clock, Parallel Structure

The graphics subsystem spawns its own thread and drives its own timing — a fundamentally different model from AudioSubsystem, but the same handle structure:

```cpp
void GraphicsSubsystem::start()
{
    m_running.store(true);
    m_frame_clock->reset();

    m_graphics_thread = std::thread([this]() {
        m_graphics_thread_id = std::this_thread::get_id();
        graphics_thread_loop();
    });
}

void GraphicsSubsystem::graphics_thread_loop()
{
    while (m_running.load(std::memory_order_acquire)) {
        if (m_paused.load(std::memory_order_acquire)) {
            std::this_thread::sleep_for(std::chrono::milliseconds(16));
            continue;
        }

        m_frame_clock->tick();       // Wall-clock driven, not hardware-driven

        process();                   // Same handle pattern as AudioSubsystem

        m_frame_clock->wait_for_next_frame();

        if (m_frame_clock->is_frame_late()) {
            uint64_t lag = m_frame_clock->get_frame_lag();
            if (lag > 2) {
                MF_RT_WARN(/* ... */ "Frame lag: {} frames behind (FPS: {:.1f})",
                    lag, m_frame_clock->get_measured_fps());
            }
        }
    }
}
```

The `process()` method is structurally parallel to the audio callback — pre-process hooks, task/node/buffer processing through the same handle interface, then Vulkan rendering and window management:

```cpp
void GraphicsSubsystem::process()
{
    for (auto& [name, hook] : m_handle->pre_process_hooks) { hook(1); }

    m_handle->tasks.process(1);      // FRAME_ACCURATE coroutines
    m_handle->nodes.process(1);      // VISUAL_RATE nodes
    m_handle->buffers.process(1);    // GRAPHICS_BACKEND buffers

    register_windows_for_processing();
    m_backend->handle_window_resize();
    render_all_windows();            // Vulkan: acquire → record → submit → present
    m_handle->windows.process();
    cleanup_closed_windows();

    for (auto& [name, hook] : m_handle->post_process_hooks) { hook(1); }
}
```

The key architectural insight: audio timing is externally driven (hardware interrupt requests N frames), graphics timing is self-driven (`FrameClock` measures elapsed wall-clock time and ticks). Both subsystems process their respective `RootNode` and `RootBuffer` hierarchies through identical CAS-based patterns, but with different `ProcessingToken` values routing to different node/buffer graphs. The `GraphicsSubsystem` also registers a custom `FRAME_ACCURATE` processor with the scheduler so that frame-rate coroutines are resumed based on the self-driven `FrameClock` position rather than an external tick.

### InputSubsystem: Event-Driven, Backend-Threaded

The input subsystem uses a third timing model — neither hardware-driven nor self-timed, but event-driven with backend-owned polling threads:

```cpp
InputSubsystem::InputSubsystem(GlobalInputConfig& config)
    : m_config(config)
    , m_tokens {
        .Buffer = Buffers::ProcessingToken::INPUT_BACKEND,
        .Node = Nodes::ProcessingToken::EVENT_RATE,
        .Task = Vruta::ProcessingToken::EVENT_DRIVEN
    }
{}

void InputSubsystem::register_callbacks()
{
    // Input subsystem doesn't register timing callbacks like audio/graphics.
    // Backends push to InputManager's queue, which has its own thread.
}
```

Each backend (HID, MIDI, OSC, Serial) manages its own polling. HIDBackend runs a dedicated poll thread that reads raw reports and pushes `InputValue` events:

```cpp
void HIDBackend::poll_thread_func()
{
    while (!m_stop_requested.load()) {
        {
            std::lock_guard lock(m_devices_mutex);
            for (auto& [id, state] : m_open_devices) {
                if (state->active.load() && state->handle) {
                    poll_device(id, *state);
                }
            }
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }
}
```

MIDIBackend uses RtMidi's callback mechanism — `rtmidi_callback` fires on RtMidi's internal thread, parses the MIDI message, and pushes an `InputValue` through the same path. Both backends converge on `InputManager`, which dispatches to registered `InputNode` instances. `InputNode` is a regular node in the processing graph — it reads from atomic target values written by the input thread and applies smoothing (linear, exponential, slew-limited) per sample during audio-rate processing:

```cpp
double InputNode::apply_smoothing(double target, double current) const
{
    switch (m_config.smoothing) {
    case SmoothingMode::NONE:       return target;
    case SmoothingMode::LINEAR:     return current + (target - current) * m_config.smoothing_factor;
    case SmoothingMode::EXPONENTIAL: return m_config.smoothing_factor * target
                                          + (1.0 - m_config.smoothing_factor) * current;
    case SmoothingMode::SLEW: {
        double diff = target - current;
        return (std::abs(diff) <= m_config.slew_rate)
            ? target : current + (diff > 0 ? m_config.slew_rate : -m_config.slew_rate);
    }
    }
}
```

The boundary between input threads and processing threads is an atomic double — the input thread writes `m_target_value`, the audio thread reads it and applies smoothing. `InputNode` also supports callback-driven events (`on_value_change`, `on_threshold_rising`, `on_button_press`, `while_in_range`) that fire within the processing cycle.

### Three Subsystems, Three Timing Models, One Pattern

```
AudioSubsystem       │ Hardware-driven (RtAudio callback)
                     │ Tokens: AUDIO_BACKEND, AUDIO_RATE, SAMPLE_ACCURATE
                     │ Clock: sample counter advanced by hardware request
                     │
GraphicsSubsystem    │ Self-driven (FrameClock + dedicated thread)
                     │ Tokens: GRAPHICS_BACKEND, VISUAL_RATE, FRAME_ACCURATE
                     │ Clock: wall-clock elapsed time, adaptive frame pacing
                     │
InputSubsystem       │ Event-driven (backend polling threads)
                     │ Tokens: INPUT_BACKEND, EVENT_RATE, EVENT_DRIVEN
                     │ Clock: none — events arrive asynchronously,
                     │        consumed by audio/graphics threads via atomics
```

Each subsystem constructs its `SubsystemTokens` at initialization and receives a `SubsystemProcessingHandle` scoped to those tokens. The handle provides the same `tasks` / `nodes` / `buffers` interface regardless of which subsystem holds it. The subsystem decides when and how to call `process()` — from a hardware callback, a self-timed loop, or not at all (input backends push to queues that other subsystems consume).

Every boundary between these threads is crossed via one of the mechanisms from Sections 1–4: CAS-claimed pending operations for graph mutation, lock-free ring buffers for event queues, atomic values for input→processing bridging, coroutines for temporal coordination, and processing tokens for domain routing. No mutexes in any processing path.

---

## Architecture Summary

| Layer              | C++20/23 Feature                              | Role                                              |
| ------------------ | --------------------------------------------- | ------------------------------------------------- |
| Type System        | Concepts, `constexpr`, variant                | Domain-agnostic data unification                  |
| Concurrency        | `atomic`, `compare_exchange`, `wait`/`notify` | Lock-free graph mutation during processing        |
| Scheduling         | Coroutines, `co_await`                        | Temporal coordination across timing domains       |
| Data Structures    | Templates, `if constexpr`, policy design      | Zero-cost dispatch for storage/concurrency/access |
| Domain Composition | Bitfield enums, token routing                 | Independent execution contexts in one process     |

---

## Links

- **Source**: [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux)
- **Website**: [mayaflux.org](https://mayaflux.org)
- **Docs**: [mayaflux.github.io/MayaFlux](https://mayaflux.github.io/MayaFlux)
- **License**: GPLv3
