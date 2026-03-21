---
title: "MayaFlux: Multi-Domain Real-Time Scheduling with C++20"
layout: "single"
---

**Ranjith Hegde** · [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux) · [mayaflux.org](https://mayaflux.org)

C++Online 2026 · Poster

---

<div class="two-column-grid breakout">
  <div class="card section-narrow">
<h2> Architectural Constraints of Heterogeneous Real-Time Systems </h2>

Real-time multimedia systems do not run in a single loop. They run several incompatible ones at once: audio callbacks at 48kHz (~20μs per sample) with hard real-time deadlines, graphics render loops at 60–144 FPS (~7–16ms per frame) with soft frame pacing, asynchronous input from MIDI/HID/OSC devices with backend-defined threading, and user-defined coroutines that may demand either sample accuracy or frame-rate granularity depending on intent. These contexts cannot be collapsed into a single abstraction without erasing the constraints that make them correct.

Historically, creative computing frameworks addressed this by separation. Pure Data distinguishes control rate from audio rate. Max/MSP sandboxes gen~ from its message scheduler. openFrameworks treats audio as an addon to an OpenGL draw loop. SuperCollider splits synthesis and language into separate processes. These were sound engineering decisions for their time: isolate domains to preserve guarantees.

But isolation introduces friction. Data fragments at boundaries. Scheduling becomes glue code. Coordination requires mutexes, queues, or ad-hoc bridges. Some loops cannot legally block; others can. Some require deterministic timing; others tolerate jitter. When independent execution contexts must cooperate without violating each other’s safety constraints, the architectural problem becomes deeper than “how to schedule.”

The question is not how to unify everything under one scheduler. The question is how independent real-time execution contexts can coexist without collapsing into a single abstraction.

This implies several simultaneous requirements:

- Independent execution loops (audio, graphics, input backends, user code) operating concurrently.
- Lock-free coordination, because some contexts cannot block or yield.
- Temporal intent that survives movement across threads and rate domains.
- Compile-time data unification, so computation does not fragment into domain-specific representations at every boundary.

Without lock-free mutation patterns, independent loops collide. Without coroutines, cross-domain temporal coordination becomes brittle or centralized. Without type-level data abstraction, every boundary degenerates into copying, conversion, and special-case glue.

When these constraints are treated as fundamental rather than incidental, a different class of system emerges: a single computation can exist simultaneously at audio-rate precision, graphics-rate updates, input-driven events, and user-defined timing, without rewriting it per domain and without compromising real-time correctness.

This is not an isolated problem to any single domain programming. Game engines coordinate rendering, physics, and input at mismatched rates. Robotics systems coordinate sensors, actuators, and planners. Any real-time C++ system with heterogeneous execution models faces the same architectural tension.

The problem is coexistence without collapse.

</div>

  <div class="card section-narrow">
<h2> A Lock-Free, Coroutine-Centric Coordination Architecture </h2>

MayaFlux approaches this problem by designing for coexistence from the outset.

It is an open-source C++20/23 framework (GPL-3.0) built around the idea that independent execution contexts should remain independent. Audio callbacks, graphics threads, input backends, and user-defined coroutines operate within a single process, but they do not share a forced abstraction or centralized scheduler. Each domain retains its own clock, threading model, and evaluation semantics.

The coordination model is architectural, not incidental.

Lock-free mutation patterns allow graph structures to evolve without blocking real-time threads. C++20 coroutines provide a weaving layer through which temporal intent can move across threads and rate domains without inventing a global scheduler. Compile-time data unification treats audio, visual, and control signals as structured numerical data rather than domain-specific types, allowing computation to exist identically across contexts.

The question MayaFlux explores is not how to merge domains, but what coordination patterns become possible when coexistence is treated as a first principle, and when modern C++ facilities make that coexistence structurally expressible.

The result is a system in which heterogeneous real-time loops can coordinate without collapsing into a single timing model, without fragmenting into glue code, and without sacrificing the guarantees that make each domain correct.

</div>
</div>

---

<div class="card wide">

<h2>In Practice</h2>

<p>
MayaFlux is not about producing a particular audiovisual result that cannot be replicated elsewhere.
Such novelties are neither the point, nor interesting even if true.
</p>

<p>
It's about making the structure of computation fluid enough that small changes open entirely different creative possibilities.
The ideas, their ontology, and how they compose drive the work. Not mastering the API. Not fighting it.
</p>

<p>
The two examples below are the same sound engine. The second changes roughly 20 lines.
What those 20 lines unlock is not a variation on the first, it's a different compositional universe.
</p>

<a href="https://youtu.be/OecKzGCxpRM" target="_blank">
<img src="https://img.youtube.com/vi/OecKzGCxpRM/maxresdefault.jpg?v=1"
alt="Example #1" width="100%" />
</a>

</div>

</br>

<div class="two-column-grid breakout">

<div class="card collapsible">

<div class="collapsible-header">
<h3>Example 1: Bouncing Bell</h3>
<p class="hint">Click to expand code</p>
</div>

<p>
A modal resonator body with dynamic spatial identity. A sine oscillator provides continuous motion that becomes a structural timing source via zero-crossing detection.
Each crossing excites the resonator, randomises its fundamental, and alternates its stereo position.
</p>

<p>
Three visual modes share the same audio engine and switch at runtime. Mouse position maps directly to strike position and pitch.
</p>

<div class="collapsible-body">

```cpp
void bouncing_bell()
{
    // Unified audiovisual playground with multiple representation modes
    auto window = create_window({ .title = "Bouncing Bell", .width = 1920, .height = 1080 });

    // System memory: timing, spatial alternation, visual accumulation, representation mode
    struct State {
        double last_wobble = 0.0; // previous oscillator value for transition detection
        uint32_t side = 0; // stereo side toggle
        float angle = 0.0F; // spiral phase memory
        float radius = 0.0F; // spiral growth memory
        bool explode = false; // reserved structural flag (future extension)
        int visual = 0; // 0=spiral, 1=burst, 2=field
    };
    auto state = std::make_shared<State>();

    // Single resonant body — spatial identity is dynamic
    auto bell = vega.ModalNetwork(12, 220.0, ModalNetwork::Spectrum::INHARMONIC);

    // Slow continuous motion becomes structural timing source
    auto wobble = vega.Sine(0.3, 1.0);

    // Continuous → discrete event (structural bounce)
    auto swing = vega.Logic([state](double x) {
        bool crossed = (state->last_wobble < 0.0) && (x >= 0.0);
        state->last_wobble = x;
        return crossed;
    });
    wobble >> swing;

    // Each bounce excites the body and flips spatial polarity
    swing->on_change_to(true, [bell, state](auto&) {
        bell->excite(get_uniform_random(0.5F, 0.9F));
        bell->set_fundamental(get_uniform_random(200.0F, 800.0F));

        route_network(bell, { state->side }, 0.15);
        state->side = 1 - state->side; // alternate left/right
    });

    // Oscillator also shapes decay — timing signal influences physical response
    bell->map_parameter("decay", wobble, MappingMode::BROADCAST);

    // High-capacity visual field for multiple accumulation strategies
    auto points = vega.PointCollectionNode(2000) | Graphics;
    auto geom = vega.GeometryBuffer(points) | Graphics;
    geom->setup_rendering({ .target_window = window });
    window->show();

    // Representation layer: same sound, multiple visual ontologies
    schedule_metro(0.016, [points, bell, wobble, state]() {
        float energy = bell->get_audio_buffer().has_value()
            ? (float)rms(bell->get_audio_buffer().value()) * 5.0F
            : 0.0F;

        auto wobble_val = (float)wobble->get_last_output(); // continuous state influences color and motion
        float hue = (wobble_val + 1.0F) * 0.5F;
        float x_pos = (state->side == 0) ? -0.5F : 0.5F;

        if (state->visual == 0) {
            // Mode 0: spiral accumulation (memory + growth)
            if (energy > 0.1F) {
                state->angle += 0.5F + wobble_val * 0.3F;
                state->radius += 0.002F;
            } else {
                state->angle += 0.01F;
                state->radius += 0.0001F;
            }

            if (state->radius > 1.0F) {
                state->radius = 0.0F;
                points->clear_points(); // visual cycle reset
            }

            points->add_point({ .position = glm::vec3(
                                    std::cos(state->angle) * state->radius,
                                    std::sin(state->angle) * state->radius * (16.0F / 9.0F),
                                    0.0F),
                .color = glm::vec3(hue, 0.8F, 1.0F - hue),
                .size = 8.0F + energy * 15.0F });

        } else if (state->visual == 1) {
            // Mode 1: localized burst (energy → multiplicity)
            for (int i = 0; i < (int)(energy * 80); i++) {
                auto a = (float)get_uniform_random(0.0F, 6.28F);
                auto r = (float)get_uniform_random(0.01F, 0.05F);

                points->add_point({ .position = glm::vec3(x_pos + std::cos(a) * r, std::sin(a) * r, 0.0F),
                    .color = glm::vec3(energy, 0.5F, 1.0F - energy),
                    .size = (float)get_uniform_random(5.0F, 20.0F) });
            }

            if (points->get_point_count() > 2000) {
                points->clear_points(); // density control
            }

        } else {
            // Mode 2: distributed field (energy → spatial diffusion)
            for (int i = 0; i < (int)(energy * 60); i++) {
                points->add_point({ .position = glm::vec3(
                                        (float)get_uniform_random(-0.8F, 0.8F),
                                        (float)get_uniform_random(-0.8F, 0.8F),
                                        0.0F),
                    .color = glm::vec3(hue, energy, 1.0F - hue),
                    .size = 3.0F + energy * 5.0F });
            }

            if (points->get_point_count() > 2000) {
                points->clear_points(); // prevent runaway accumulation
            }
        }
    });

    // Direct physical interaction: position maps to pitch and intensity
    on_mouse_pressed(window, IO::MouseButtons::Left, [window, bell](double x, double y) {
        glm::vec2 pos = normalize_coords(x, y, window);
        float pitch = 200.0F + ((float)pos.y + 1.0F) * 300.0F;
        float intensity = 0.3F + std::abs((float)pos.x) * 0.6F;
        bell->set_fundamental(pitch);
        bell->excite(intensity);
    });

    // Switch representation modes (same sound engine, different interpretation)
    on_key_pressed(window, IO::Keys::N1, [points, state]() { state->visual = 0; points->clear_points(); });
    on_key_pressed(window, IO::Keys::N2, [points, state]() { state->visual = 1; points->clear_points(); });
    on_key_pressed(window, IO::Keys::N3, [points, state]() { state->visual = 2; points->clear_points(); });

    // Collapse spatial polarity
    on_key_pressed(window, IO::Keys::M, [bell]() { route_network(bell, { 0, 1 }, 0.3); });
}
```

</div>
<!-- </div>  -->
</div>

<div class="card collapsible">

<div class="collapsible-header">
<h3>Example 1.2: Bouncing Bell Extended</h3>
<p class="hint">Click to expand code</p>
</div>

<p>
The resonator gains modal coupling between adjacent modes, a slow pitch drift oscillator mapped directly into the network's frequency parameter and into visual hue simultaneously, and a runtime-swappable rhythm source (slow sine, fast sine, random noise) wired through the same zero-crossing logic node. Mouse now maps to strike position on the resonator body rather than pitch. Space cycles the rhythm source at runtime without stopping anything.

The audio engine is structurally identical. The relationships between signals changed.

</p>

<div class="collapsible-body">

```cpp
void bouncing_v2()
{
    // Unified audiovisual playground with multiple representation modes
    auto window = create_window({ .title = "Bouncing Bell", .width = 1920, .height = 1080 });

    // System memory: timing, spatial alternation, visual accumulation, representation mode
    struct State {
        double last_wobble = 0.0; // previous oscillator value for transition detection
        uint32_t side = 0; // stereo side toggle
        float angle = 0.0F; // spiral phase memory
        float radius = 0.0F; // spiral growth memory
        bool explode = false; // reserved structural flag (future extension)
        int visual = 0; // 0=spiral, 1=burst, 2=field
    };
    auto state = std::make_shared<State>();

    // Single resonant body — spatial identity is dynamic
    auto bell = vega.ModalNetwork(12, 220.0, ModalNetwork::Spectrum::INHARMONIC);

    // Slow continuous motion becomes structural timing source
    auto wobble = vega.Sine(0.3, 1.0);

    // Continuous → discrete event (structural bounce)
    auto swing = vega.Logic([state](double x) {
        bool crossed = (state->last_wobble < 0.0) && (x >= 0.0);
        state->last_wobble = x;
        return crossed;
    });
    wobble >> swing;

    // Each bounce excites the body and flips spatial polarity
    swing->on_change_to(true, [bell, state](auto&) {
        bell->excite(get_uniform_random(0.5F, 0.9F));
        bell->set_fundamental(get_uniform_random(200.0F, 800.0F));

        route_network(bell, { state->side }, 0.15);
        state->side = 1 - state->side; // alternate left/right
    });

    // Oscillator also shapes decay — timing signal influences physical response
    bell->map_parameter("decay", wobble, MappingMode::BROADCAST);

    // High-capacity visual field for multiple accumulation strategies
    auto points = vega.PointCollectionNode(2000) | Graphics;
    auto geom = vega.GeometryBuffer(points) | Graphics;
    geom->setup_rendering({ .target_window = window });
    window->show();

    // Representation layer: same sound, multiple visual ontologies
    schedule_metro(0.016, [points, bell, wobble, state]() {
        float energy = bell->get_audio_buffer().has_value()
            ? (float)rms(bell->get_audio_buffer().value()) * 5.0F
            : 0.0F;

        auto wobble_val = (float)wobble->get_last_output(); // continuous state influences color and motion
        float hue = (wobble_val + 1.0F) * 0.5F;
        float x_pos = (state->side == 0) ? -0.5F : 0.5F;

        if (state->visual == 0) {
            // Mode 0: spiral accumulation (memory + growth)
            if (energy > 0.1F) {
                state->angle += 0.5F + wobble_val * 0.3F;
                state->radius += 0.002F;
            } else {
                state->angle += 0.01F;
                state->radius += 0.0001F;
            }

            if (state->radius > 1.0F) {
                state->radius = 0.0F;
                points->clear_points(); // visual cycle reset
            }

            points->add_point({ .position = glm::vec3(
                                    std::cos(state->angle) * state->radius,
                                    std::sin(state->angle) * state->radius * (16.0F / 9.0F),
                                    0.0F),
                .color = glm::vec3(hue, 0.8F, 1.0F - hue),
                .size = 8.0F + energy * 15.0F });

        } else if (state->visual == 1) {
            // Mode 1: localized burst (energy → multiplicity)
            for (int i = 0; i < (int)(energy * 80); i++) {
                auto a = (float)get_uniform_random(0.0F, 6.28F);
                auto r = (float)get_uniform_random(0.01F, 0.05F);

                points->add_point({ .position = glm::vec3(x_pos + std::cos(a) * r, std::sin(a) * r, 0.0F),
                    .color = glm::vec3(energy, 0.5F, 1.0F - energy),
                    .size = (float)get_uniform_random(5.0F, 20.0F) });
            }

            if (points->get_point_count() > 2000) {
                points->clear_points(); // density control
            }

        } else {
            // Mode 2: distributed field (energy → spatial diffusion)
            for (int i = 0; i < (int)(energy * 60); i++) {
                points->add_point({ .position = glm::vec3(
                                        (float)get_uniform_random(-0.8F, 0.8F),
                                        (float)get_uniform_random(-0.8F, 0.8F),
                                        0.0F),
                    .color = glm::vec3(hue, energy, 1.0F - hue),
                    .size = 3.0F + energy * 5.0F });
            }

            if (points->get_point_count() > 2000) {
                points->clear_points(); // prevent runaway accumulation
            }
        }
    });

    // Direct physical interaction: position maps to pitch and intensity
    on_mouse_pressed(window, IO::MouseButtons::Left, [window, bell](double x, double y) {
        glm::vec2 pos = normalize_coords(x, y, window);
        float pitch = 200.0F + ((float)pos.y + 1.0F) * 300.0F;
        float intensity = 0.3F + std::abs((float)pos.x) * 0.6F;
        bell->set_fundamental(pitch);
        bell->excite(intensity);
    });

    // Switch representation modes (same sound engine, different interpretation)
    on_key_pressed(window, IO::Keys::N1, [points, state]() { state->visual = 0; points->clear_points(); });
    on_key_pressed(window, IO::Keys::N2, [points, state]() { state->visual = 1; points->clear_points(); });
    on_key_pressed(window, IO::Keys::N3, [points, state]() { state->visual = 2; points->clear_points(); });

    // Collapse spatial polarity
    on_key_pressed(window, IO::Keys::M, [bell]() { route_network(bell, { 0, 1 }, 0.3); });
}
```

</div> </div> </div>

---

# MayaFlux Architecture

<div class="card collapsible">

<div class="collapsible-header">
<h2>1 · Lock-Free Coordination Across Thread Boundaries</h2>
<p class="hint">Click to expand</p>
</div>

<div class="section-preview">
<p>
Real-time threads cannot block. But processing graphs must evolve:  nodes added, buffers removed, processors swapped; often while audio callbacks are mid-flight at 48 kHz. MayaFlux solves this with a CAS-based pending operation pattern: any thread can mutate the graph at any time, real-time threads never block, and pending changes drain at cycle boundaries. The same pattern drives nodes, buffers, and processors identically.
</p>
</div>

<div class="collapsible-body">

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

</div>
</div>

---

<div class="card collapsible">

<div class="collapsible-header">
<h2>2 · C++20 Coroutines Weaving Temporal Intent</h2>
<p class="hint">Click to expand</p>
</div>

<div class="section-preview">
<p>
Traditional real-time systems fragment temporal logic across callbacks, timers, and state machines. Coroutines invert this: a single function body describes a complete temporal narrative: gates, triggers, delays, patterns, as sequential code. The scheduler determines <em>when</em> to resume; the coroutine describes <em>what</em> and <em>how long</em>. Timing is deterministic and jitter-free: one <code>co_await SampleDelay{48000}</code> is exactly one second at 48 kHz, every time.
</p>
</div>

<div class="collapsible-body">

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

</div>
</div>

---

<div class="card collapsible">

<div class="collapsible-header">
<h2>3 · Compile-Time Data Unification</h2>
<p class="hint">Click to expand</p>
</div>

<div class="section-preview">
<p>
Audio samples, pixel values, vertex positions, control voltages, at the machine level these are all just numbers. MayaFlux's type system treats them that way. Concepts like <code>ArithmeticData</code> and <code>ComputeData</code> constrain computation at compile time without encoding domain assumptions. <code>DataVariant</code> holds any numeric storage type. The same extractor, transformer, or analyzer works identically whether it's processing spectral data or a point cloud.
</p>
</div>

<div class="collapsible-body">

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

</div>
</div>

---

<div class="card collapsible">

<div class="collapsible-header">
<h2>4 · Multi-Domain Coexistence: A Working Example</h2>
<p class="hint">Click to expand</p>
</div>

<div class="section-preview">
<p>
Each subsystem {audio, graphics, input} carries its own <code>ProcessingToken</code> that routes through independent node graphs, buffer hierarchies, and scheduler clocks. These tokens compose into <code>Domain</code> values via bitfield operations, making domain membership decomposable and extensible. A single <code>compose()</code> function can declare audio synthesis, Vulkan rendering, and cross-domain coordination without shared locks or unified timing.
</p>
</div>

<div class="collapsible-body">

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
        points->add_point(Nodes::PointVertex{
            .position = glm::vec3(x, y, 0.0f),
            .color = glm::vec3(brightness, brightness * 0.8f, 1.0f),
            .size = 8.0f + radius * 4.0f
        });
    });
}
```

Audio processing runs at 48 kHz via `AUDIO_BACKEND` token. Graphics rendering runs at display refresh via `GRAPHICS_BACKEND` token. The metro coroutine bridges the two — its callback reads audio-domain state and writes to a graphics-domain node. The `| Audio` and `| Graphics` operators route nodes and buffers to their respective `RootNode` and `RootBuffer` hierarchies. No shared locks. Each domain processes its own graph independently; the coroutine provides temporal coordination.

</div>
</div>

---

<div class="card collapsible">

<div class="collapsible-header">
<h2>5 · Thread Architecture: Where the Boundaries Actually Are</h2>
<p class="hint">Click to expand</p>
</div>

<div class="section-preview">
<p>
Three subsystems, three timing models, one coordination pattern. Audio is hardware-driven: RtAudio fires the callback, the sample counter advances. Graphics is self-driven: a dedicated thread ticks a <code>FrameClock</code> and paces its own frames. Input is event-driven: backend polling threads push to lock-free queues consumed by other subsystems. All three use the same <code>SubsystemProcessingHandle</code> interface -> same managers, different token-scoped views. Every thread boundary is crossed via CAS operations, atomic values, or coroutine-based coordination. No mutexes in any processing path.
</p>
</div>

<div class="collapsible-body">

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

</div>
</div>

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

<div class="card wide">

<h2>In Practice: Rhythm as Topology</h2>

<p>
The previous examples showed how a single modal resonator could become an entirely different compositional instrument by changing 20 lines.
The same principle applies at larger scale.
</p>

<p>
The three examples below share the same audio engine: a four-voice rhythm section built from phasors, envelopes, and noise filters.
What changes is the visual substrate and the relationships between rhythmic events and spatial form.
</p>

<p>
Each step adds a layer. Each layer opens a different creative universe.
</p>

<a href="https://youtu.be/SHWebquQbZs" target="_blank">
<img src="https://img.youtube.com/vi/SHWebquQbZs/maxresdefault.jpg"
alt="Rhythm as Topology" width="100%">

</a>

</div>

<div class="card-group columns breakout">

<div class="card collapsible">

<div class="collapsible-header">
<h3>Example 2: Living Topology</h3>
<p class="hint">Click to expand code</p>
</div>

<p>
A four-voice rhythm engine drives a point cloud connected by proximity topology. Kick expands the field radially. Snare triggers topology regeneration and applies rotational shear. Hat cycles between proximity algorithms (minimum spanning tree, k-nearest, nearest neighbor, sequential) every 16 hits. Clap injects positional jitter into individual points.
</p>

<p>
The topology IS the content. Rhythm becomes spatial relationship.
</p>

<div class="collapsible-body">

```cpp
void rhythm_topology_live()
{
    auto window = create_window({ .title = "Living Topology",
        .width = 1920,
        .height = 1080 });

    constexpr size_t N = 34;

    struct State {
        bool sequencing {};
        bool chaos_mode {};
        float expansion {};
        float shear {};
        uint32_t hat_count {};
        uint32_t mode_idx {};
        std::vector<glm::vec3> home = std::vector<glm::vec3>(N);
        std::vector<glm::vec3> jitter = std::vector<glm::vec3>(N, glm::vec3(0.0F));
    };
    auto state = std::make_shared<State>();

    // === AUDIO ===

    auto kick_phasor = vega.Phasor(20.0, 1.0);
    auto kick_env = vega.Polynomial([](double x) { return std::exp(-x * 15.0); });
    kick_env->set_input_node(kick_phasor);
    auto kick = vega.Sine(55.0)[0] | Audio;
    kick->set_amplitude_modulator(kick_env);

    auto snare_phasor = vega.Phasor(30.0, 1.0);
    auto snare_env = vega.Polynomial([](double x) { return std::exp(-x * 25.0); })[1] | Audio;
    snare_env->set_input_node(snare_phasor);
    auto snare_noise = vega.Random();
    auto snare = (vega.FIR(snare_noise, std::vector { 0.3, 0.4, 0.3 })) * snare_env;
    register_audio_node(snare, 1);

    auto hat_phasor = vega.Phasor(50.0, 1.0);
    auto hat_env = vega.Polynomial([](double x) { return std::exp(-x * 50.0); });
    hat_env->set_input_node(hat_phasor);
    auto hat = vega.Sine(8000.0)[0] | Audio;
    hat->set_amplitude_modulator(hat_env);

    auto clap_phasor = vega.Phasor(40.0, 1.0);
    auto clap_env = vega.Polynomial([](double x) {
        return std::exp(-x * 35.0) * (1.0 + 0.3 * std::sin(x * 200.0));
    });
    clap_env->set_input_node(clap_phasor);
    auto clap = (vega.FIR(vega.Random(), std::vector { 0.1, 0.2, 0.4, 0.2, 0.1 })) * clap_env;
    register_audio_node(clap, 0);

    auto bass = vega.Sine(42.0)[{ 0, 1 }] | Audio;
    bass->set_amplitude_modulator(vega.Sine(0.15, 0.12));

    // === TOPOLOGY ===

    auto topo = vega.TopologyGeneratorNode(
                    Kinesis::ProximityMode::MINIMUM_SPANNING_TREE,
                    false,
                    N)
        | Graphics;

    {
        Kinesis::Stochastic::Stochastic rng;
        Kinesis::SamplerBounds bounds { glm::vec3(-0.65F), glm::vec3(0.65F) };
        auto samples = Kinesis::generate_samples(
            Kinesis::SpatialDistribution::LISSAJOUS, N, bounds, rng);
        for (size_t i = 0; i < N; ++i) {
            state->home[i] = samples[i].position;
            topo->add_point({ .position = samples[i].position,
                .color = samples[i].color,
                .thickness = 1.5F });
        }
        topo->regenerate_topology();
    }

    auto buffer = vega.GeometryBuffer(topo) | Graphics;
    buffer->setup_rendering({ .target_window = window,
        .topology = Portal::Graphics::PrimitiveTopology::LINE_LIST });

    window->show();

    // === SEQUENCING ===

    schedule_metro(0.5, [kick_phasor, state]() {
        if (!state->sequencing) return;
        kick_phasor->reset();
        state->expansion = std::min(state->expansion + 0.25F, 1.2F);
    }, "kick_layer");

    /// @brief Snare: regenerate topology from current deformed positions
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.6;
            return (step % 4 == 2);
        },
        [snare_phasor, topo, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                snare_phasor->reset();
                state->shear += 0.2F;
                topo->regenerate_topology();
            }
        },
        0.25, "snare_pattern");

    /// @brief Hat: cycle proximity mode every 16 hits
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.5;
            return true;
        },
        [hat_phasor, topo, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                hat_phasor->reset();
                state->hat_count++;
                if (state->hat_count >= 16) {
                    state->hat_count = 0;
                    static constexpr std::array modes = {
                        Kinesis::ProximityMode::MINIMUM_SPANNING_TREE,
                        Kinesis::ProximityMode::K_NEAREST,
                        Kinesis::ProximityMode::NEAREST_NEIGHBOR,
                        Kinesis::ProximityMode::SEQUENTIAL,
                    };
                    state->mode_idx = (state->mode_idx + 1) % modes.size();
                    topo->set_connection_mode(modes[state->mode_idx]);
                }
            }
        },
        0.125, "hat_pattern");

    /// @brief Clap: jitter burst
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.8;
            return (step % 8 == 5);
        },
        [clap_phasor, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                clap_phasor->reset();
                for (auto& j : state->jitter)
                    j = glm::vec3(
                        static_cast<float>(get_uniform_random(-0.1, 0.1)),
                        static_cast<float>(get_uniform_random(-0.1, 0.1)),
                        0.0F);
            }
        },
        0.25, "clap_pattern");

    // === DEFORMATION ===

    schedule_metro(0.016, [topo, kick, bass, state]() {
        float kick_e = static_cast<float>(std::abs(kick->get_last_output()));
        float bass_e = static_cast<float>(std::abs(bass->get_last_output()));

        state->expansion *= 0.97F;
        state->shear *= 0.985F;

        for (size_t i = 0; i < N; ++i) {
            glm::vec3 home = state->home[i];
            glm::vec3 pos = home;

            float dist = glm::length(glm::vec2(home));
            if (dist > 0.001F) {
                glm::vec3 radial = glm::normalize(glm::vec3(home.x, home.y, 0.0F));
                pos += radial * state->expansion * 0.25F;
            }

            float sign = (home.y > 0.0F) ? 1.0F : -1.0F;
            float a = state->shear * sign;
            pos = glm::vec3(
                pos.x * std::cos(a) - pos.y * std::sin(a),
                pos.x * std::sin(a) + pos.y * std::cos(a),
                pos.z);

            state->jitter[i] *= 0.93F;
            pos += state->jitter[i];

            float brightness = 0.3F + kick_e * 2.0F;
            float pct = i / static_cast<float>(N);
            brightness *= pct;
            topo->update_point(i, { .position = pos,
                .color = glm::vec3(brightness * 0.6F, brightness * 0.8F,
                    std::min(1.0F, brightness)),
                .thickness = 1.0F + (bass_e * 1.2F) * pct });
        }
    });

    // === INTERACTION ===

    on_key_pressed(window, IO::Keys::Space, [state]() {
        state->sequencing = !state->sequencing;
    });

    on_key_pressed(window, IO::Keys::C, [state]() {
        state->chaos_mode = !state->chaos_mode;
    });

    auto regen_homes = [topo, state](Kinesis::SpatialDistribution dist) {
        Kinesis::Stochastic::Stochastic rng;
        Kinesis::SamplerBounds bounds { glm::vec3(-0.65F), glm::vec3(0.65F) };
        auto samples = Kinesis::generate_samples(dist, N, bounds, rng);
        for (size_t i = 0; i < N; ++i)
            state->home[i] = samples[i].position;
        topo->regenerate_topology();
    };

    on_key_pressed(window, IO::Keys::Q, [regen_homes]() {
        regen_homes(Kinesis::SpatialDistribution::LISSAJOUS);
    });
    on_key_pressed(window, IO::Keys::W, [regen_homes]() {
        regen_homes(Kinesis::SpatialDistribution::FIBONACCI_SPHERE);
    });
    on_key_pressed(window, IO::Keys::E, [regen_homes]() {
        regen_homes(Kinesis::SpatialDistribution::TORUS);
    });

    on_mouse_pressed(window, IO::MouseButtons::Left,
        [window, topo](double x, double y) {
            glm::vec2 pos = normalize_coords(x, y, window);
            topo->add_point({ .position = glm::vec3(pos, 0.0F),
                .color = glm::vec3(1.0F, 0.9F, 0.4F),
                .thickness = 2.5F });
            topo->regenerate_topology();
        });
}
```

</div>
<!-- </div>  -->
</div>

<div class="card collapsible">

<div class="collapsible-header">
<h3>Example 2.1: Living Curve </h3>
<p class="hint">Click to expand code</p>
</div>

<p>
The audio engine is identical. The visual substrate changes from discrete topology to continuous path.

`TopologyGeneratorNode` becomes `PathGeneratorNode`. Proximity algorithms become interpolation modes (Catmull-Rom, B-spline, linear). Points no longer connect through geometric relationship; they define a parametric curve that flows through space.

What changed:

- **Snare** no longer regenerates topology. It toggles curve tension between tight (0.8) and loose (0.15), snapping the curve between rigid and fluid states.
- **Hat** cycles interpolation mode every 12 hits instead of proximity mode every 16. The visual character of the curve itself transforms.
- **Clap** injects angular jitter into orbital phases rather than positional jitter into coordinates. The perturbation is rotational, not translational.
- **Deformation** drives points along Lissajous orbits with per-point phase accumulation. Points are no longer displaced from fixed home positions; they travel continuous paths.

Same rhythm. Same timing. Different spatial ontology entirely.

</p>

<div class="collapsible-body">

```cpp
void rhythm_path_live()
{
    auto window = create_window({ .title = "Living Curve",
        .width = 1920,
        .height = 1080 });

    constexpr size_t N = 18;

    struct State {
        bool sequencing {};
        bool chaos_mode {};
        float expansion {};
        float tension { 0.5F };
        bool tension_tight { true };
        uint32_t hat_count {};
        uint32_t mode_idx {};
        std::array<float, N> phases {};
        std::array<float, N> jitter {};
    };
    auto state = std::make_shared<State>();

    for (size_t i = 0; i < N; ++i) {
        state->phases[i] = static_cast<float>(i) / static_cast<float>(N) * 6.2832F;
    }

    // === AUDIO (identical engine) ===

    auto kick_phasor = vega.Phasor(20.0, 1.0);
    auto kick_env = vega.Polynomial([](double x) { return std::exp(-x * 15.0); });
    kick_env->set_input_node(kick_phasor);
    auto kick = vega.Sine(55.0)[0] | Audio;
    kick->set_amplitude_modulator(kick_env);

    auto snare_phasor = vega.Phasor(30.0, 1.0);
    auto snare_env = vega.Polynomial([](double x) { return std::exp(-x * 25.0); })[1] | Audio;
    snare_env->set_input_node(snare_phasor);
    auto snare = (vega.FIR(vega.Random(), std::vector { 0.3, 0.4, 0.3 })) * snare_env;
    register_audio_node(snare, 1);

    auto hat_phasor = vega.Phasor(50.0, 1.0);
    auto hat_env = vega.Polynomial([](double x) { return std::exp(-x * 50.0); });
    hat_env->set_input_node(hat_phasor);
    auto hat = vega.Sine(8000.0)[0] | Audio;
    hat->set_amplitude_modulator(hat_env);

    auto clap_phasor = vega.Phasor(40.0, 1.0);
    auto clap_env = vega.Polynomial([](double x) {
        return std::exp(-x * 35.0) * (1.0 + 0.3 * std::sin(x * 200.0));
    });
    clap_env->set_input_node(clap_phasor);
    auto clap = (vega.FIR(vega.Random(), std::vector { 0.1, 0.2, 0.4, 0.2, 0.1 })) * clap_env;
    register_audio_node(clap, 0);

    auto bass = vega.Sine(42.0)[{ 0, 1 }] | Audio;
    bass->set_amplitude_modulator(vega.Sine(0.15, 0.12));

    // === PATH (replaces topology) ===

    auto path = vega.PathGeneratorNode(
                    Kinesis::InterpolationMode::CATMULL_ROM,
                    24, N, 0.5)
        | Graphics;

    for (size_t i = 0; i < N; ++i) {
        float phase = state->phases[i];
        float x = std::sin(phase) * 0.5F;
        float y = std::sin(phase * 1.5F) * 0.4F;
        float hue = static_cast<float>(i) / static_cast<float>(N);
        path->add_control_point({ .position = glm::vec3(x, y, 0.0F),
            .color = glm::vec3(0.4F + hue * 0.5F, 0.6F, 1.0F - hue * 0.4F),
            .thickness = 2.0F });
    }

    path->set_path_color(glm::vec3(0.5F, 0.7F, 1.0F), false);
    path->set_path_thickness(2.0F, false);

    auto buffer = vega.GeometryBuffer(path) | Graphics;
    buffer->setup_rendering({ .target_window = window,
        .topology = Portal::Graphics::PrimitiveTopology::LINE_LIST });

    window->show();

    // === SEQUENCING (same timing, different targets) ===

    schedule_metro(0.5, [kick_phasor, state]() {
        if (!state->sequencing) return;
        kick_phasor->reset();
        state->expansion = std::min(state->expansion + 0.2F, 0.8F);
    }, "kick_layer");

    /// @brief Snare toggles tension between tight and loose
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.6;
            return (step % 4 == 2);
        },
        [snare_phasor, path, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                snare_phasor->reset();
                state->tension_tight = !state->tension_tight;
                state->tension = state->tension_tight ? 0.8F : 0.15F;
                path->set_tension(static_cast<double>(state->tension));
            }
        },
        0.25, "snare_pattern");

    /// @brief Hat cycles interpolation mode every 12 hits
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.5;
            return true;
        },
        [hat_phasor, path, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                hat_phasor->reset();
                state->hat_count++;
                if (state->hat_count >= 12) {
                    state->hat_count = 0;
                    static constexpr std::array modes = {
                        Kinesis::InterpolationMode::CATMULL_ROM,
                        Kinesis::InterpolationMode::BSPLINE,
                        Kinesis::InterpolationMode::LINEAR,
                    };
                    state->mode_idx = (state->mode_idx + 1) % modes.size();
                    path->set_interpolation_mode(modes[state->mode_idx]);
                }
            }
        },
        0.125, "hat_pattern");

    /// @brief Clap injects angular jitter into orbital phases
    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.8;
            return (step % 8 == 5);
        },
        [clap_phasor, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                clap_phasor->reset();
                for (auto& j : state->jitter)
                    j = static_cast<float>(get_uniform_random(-0.6, 0.6));
            }
        },
        0.25, "clap_pattern");

    // === DEFORMATION (orbital, not displacement) ===

    schedule_metro(0.016, [path, kick, bass, state]() {
        float kick_e = static_cast<float>(std::abs(kick->get_last_output()));
        float bass_e = static_cast<float>(std::abs(bass->get_last_output()));

        state->expansion *= 0.97F;

        for (size_t i = 0; i < N; ++i) {
            state->phases[i] += 0.008F + static_cast<float>(i) * 0.001F;
            state->jitter[i] *= 0.94F;

            float phase = state->phases[i] + state->jitter[i];
            float base_x = std::sin(phase) * 0.5F;
            float base_y = std::sin(phase * 1.5F) * 0.4F;

            float dist = std::sqrt(base_x * base_x + base_y * base_y);
            float expand = (dist > 0.001F) ? state->expansion * 0.3F / dist : 0.0F;
            float x = base_x * (1.0F + expand);
            float y = base_y * (1.0F + expand);

            float hue = static_cast<float>(i) / static_cast<float>(N);
            float brightness = 0.4F + kick_e * 1.5F;

            path->update_control_point(i, { .position = glm::vec3(x, y, 0.0F),
                .color = glm::vec3(
                    brightness * (0.4F + hue * 0.5F),
                    brightness * 0.7F,
                    brightness * (1.0F - hue * 0.3F)),
                .thickness = 1.5F + bass_e * 4.0F });
        }
    });

    // === INTERACTION ===

    on_key_pressed(window, IO::Keys::Space, [state]() {
        state->sequencing = !state->sequencing;
    });

    on_key_pressed(window, IO::Keys::C, [state]() {
        state->chaos_mode = !state->chaos_mode;
    });

    on_mouse_pressed(window, IO::MouseButtons::Left,
        [window, path](double x, double y) {
            glm::vec2 pos = normalize_coords(x, y, window);
            path->add_control_point({ .position = glm::vec3(pos, 0.0F),
                .color = glm::vec3(1.0F, 0.9F, 0.4F),
                .thickness = 3.0F });
        });
}

```

</div> </div>

<div class="card collapsible">

<div class="collapsible-header">
<h3>Example 2.2: Curve over texture</h3>
<p class="hint">Click to expand code</p>
</div>

<p>
The audio engine is still identical. The visual substrate gains a second layer: a textured backdrop driven by audio through custom fragment shaders and node-to-push-constant bindings.

What changed from Example 2:

- **Layer 1 (new): Textured backdrop.** `vega.read_image()` loads a texture. `setup_rendering()` receives a custom fragment shader (`polar_warp.frag`). Alpha blending is enabled on the render processor. Three `Polynomial` nodes derive shader parameters from audio envelopes: kick drives radial distortion scale, snare drives angular velocity, bass drives chromatic aberration split. A `NodeBindingsProcessor` binds these nodes to push constant offsets, injecting audio-reactive values directly into the GPU shader every frame. This is the same node architecture used everywhere else in MayaFlux. The `Polynomial` that scales the kick envelope doesn't know it's feeding a shader; it just outputs a number. The GPU doesn't know the number came from an audio envelope. The binding is structural, not conceptual.
- **Layer 2: The living curve** from Example 2, rendered on top with its own custom fragment shader (`line_glow.frag`). The curve deformation logic is unchanged.
- **Stereo routing controls** added: keys 1/2/3 route the entire rhythm section hard left, hard right, or center stereo.
- **Commented camera alternative** shows the same textured layer could source from a live camera device instead of a static image, with identical shader pipeline. The swap is one block of code.

The audio engine drives both layers simultaneously. The curve deforms the same way. The texture warps from the same envelopes. Two visual ontologies layered from one rhythmic source.

</p>

<div class="collapsible-body">

```cpp
void rhythm_path_textured()
{
    auto window = create_window({ .title = "Curve Over Texture",
        .width = 3840,
        .height = 2160 });

    constexpr size_t N = 18;

    struct State {
        bool sequencing { true };
        bool chaos_mode {};
        float expansion {};
        float tension { 0.5F };
        bool tension_tight { true };
        uint32_t hat_count {};
        uint32_t mode_idx {};
        std::array<float, N> phases {};
        std::array<float, N> jitter {};
    };
    auto state = std::make_shared<State>();

    for (size_t i = 0; i < N; ++i) {
        state->phases[i] = static_cast<float>(i) / static_cast<float>(N) * 6.2832F;
    }

    // === AUDIO (identical engine) ===

    auto kick_phasor = vega.Phasor(20.0, 1.0);
    auto kick_env = vega.Polynomial([](double x) { return std::exp(-x * 15.0); });
    kick_env->set_input_node(kick_phasor);
    auto kick = vega.Sine(55.0)[0] | Audio;
    kick->set_amplitude_modulator(kick_env);

    auto snare_phasor = vega.Phasor(30.0, 1.0);
    auto snare_env = vega.Polynomial([](double x) { return std::exp(-x * 25.0); })[1] | Audio;
    snare_env->set_input_node(snare_phasor);
    auto snare = (vega.FIR(vega.Random(), std::vector { 0.3, 0.4, 0.3 })) * snare_env;
    register_audio_node(snare, 1);

    auto hat_phasor = vega.Phasor(50.0, 1.0);
    auto hat_env = vega.Polynomial([](double x) { return std::exp(-x * 50.0); });
    hat_env->set_input_node(hat_phasor);
    auto hat = vega.Sine(8000.0)[0] | Audio;
    hat->set_amplitude_modulator(hat_env);

    auto clap_phasor = vega.Phasor(40.0, 1.0);
    auto clap_env = vega.Polynomial([](double x) {
        return std::exp(-x * 35.0) * (1.0 + 0.3 * std::sin(x * 200.0));
    });
    clap_env->set_input_node(clap_phasor);
    auto clap = (vega.FIR(vega.Random(), std::vector { 0.1, 0.2, 0.4, 0.2, 0.1 })) * clap_env;
    register_audio_node(clap, 0);

    auto bass = vega.Sine(42.0)[{ 0, 1 }] | Audio;
    bass->set_amplitude_modulator(vega.Sine(0.15, 0.12));

    // === LAYER 1: TEXTURED BACKDROP ===

    auto tex = vega.read_image("res/texture.png") | Graphics;

    // Alternative: live camera source with identical shader pipeline
    // auto manager = get_io_manager();
    // auto container = manager->open_camera({
    //     .device_name = "/dev/video0",
    //     .target_width = 1920, .target_height = 1080, .target_fps = 30.0
    // });
    // auto tex = manager->hook_camera_to_buffer(container);

    tex->setup_rendering({
        .target_window = window,
        .fragment_shader = "polar_warp.frag",
    });

    tex->get_render_processor()->enable_alpha_blending();

    window->show();

    struct Params {
        float radial_scale = 0.0F;
        float angular_velocity = 0.0F;
        float chroma_split = 0.0F;
    };

    auto radial_node = vega.Polynomial([](double x) {
        return std::abs(x) * 0.5;
    }) | Graphics;
    radial_node->set_input_node(kick_env);

    auto angular_node = vega.Polynomial([](double x) {
        return std::abs(x) * 2.5;
    }) | Graphics;
    angular_node->set_input_node(snare_env);

    auto chroma_node = vega.Polynomial([](double x) {
        return std::abs(x) * 0.15;
    }) | Graphics;
    chroma_node->set_input_node(bass);

    auto shader_config = Buffers::ShaderConfig { "polar_warp.frag" };
    shader_config.push_constant_size = sizeof(Params);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<Params>();
    node_bindings->bind_node("radial", radial_node,
        offsetof(Params, radial_scale), sizeof(float));
    node_bindings->bind_node("angular", angular_node,
        offsetof(Params, angular_velocity), sizeof(float));
    node_bindings->bind_node("chroma", chroma_node,
        offsetof(Params, chroma_split), sizeof(float));

    add_processor(node_bindings, tex, Buffers::ProcessingToken::GRAPHICS_BACKEND);

    // === LAYER 2: LIVING CURVE ===

    auto path = vega.PathGeneratorNode(
                    Kinesis::InterpolationMode::CATMULL_ROM,
                    24, N, 0.5)
        | Graphics;

    for (size_t i = 0; i < N; ++i) {
        float phase = state->phases[i];
        path->add_control_point({ .position = glm::vec3(
                                      std::sin(phase) * 0.5F,
                                      std::sin(phase * 1.5F) * 0.4F, 0.0F),
            .color = glm::vec3(1.0F, 0.85F, 0.6F),
            .thickness = 2.5F });
    }

    path->set_path_color(glm::vec3(1.0F, 0.85F, 0.6F), false);

    auto line_buf = vega.GeometryBuffer(path) | Graphics;
    line_buf->setup_rendering({ .target_window = window,
        .fragment_shader = "line_glow.frag",
        .topology = Portal::Graphics::PrimitiveTopology::LINE_LIST });

    // === SEQUENCING (identical timing) ===

    schedule_metro(0.5, [kick_phasor, state]() {
        if (!state->sequencing) return;
        kick_phasor->reset();
        state->expansion = std::min(state->expansion + 0.2F, 0.8F);
    }, "kick_layer");

    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.6;
            return (step % 4 == 2);
        },
        [snare_phasor, path, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                snare_phasor->reset();
                state->tension_tight = !state->tension_tight;
                state->tension = state->tension_tight ? 0.8F : 0.15F;
                path->set_tension(static_cast<double>(state->tension));
            }
        },
        0.25, "snare_pattern");

    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.5;
            return true;
        },
        [hat_phasor, path, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                hat_phasor->reset();
                state->hat_count++;
                if (state->hat_count >= 12) {
                    state->hat_count = 0;
                    static constexpr std::array modes = {
                        Kinesis::InterpolationMode::CATMULL_ROM,
                        Kinesis::InterpolationMode::BSPLINE,
                        Kinesis::InterpolationMode::LINEAR,
                    };
                    state->mode_idx = (state->mode_idx + 1) % modes.size();
                    path->set_interpolation_mode(modes[state->mode_idx]);
                }
            }
        },
        0.125, "hat_pattern");

    schedule_pattern(
        [state](uint64_t step) {
            if (state->chaos_mode)
                return get_uniform_random(0.0, 1.0) > 0.8;
            return (step % 8 == 5);
        },
        [clap_phasor, state](std::any hit) {
            if (!state->sequencing)
                return;
            if (std::any_cast<bool>(hit)) {
                clap_phasor->reset();
                for (auto& j : state->jitter)
                    j = static_cast<float>(get_uniform_random(-0.6, 0.6));
            }
        },
        0.25, "clap_pattern");

    // === PATH DEFORMATION (unchanged from Example 2) ===

    schedule_metro(0.016, [path, kick, bass, state]() {
        float kick_e = static_cast<float>(std::abs(kick->get_last_output()));
        float bass_e = static_cast<float>(std::abs(bass->get_last_output()));

        state->expansion *= 0.97F;

        for (size_t i = 0; i < N; ++i) {
            state->phases[i] += 0.008F + static_cast<float>(i) * 0.001F;
            state->jitter[i] *= 0.94F;

            float phase = state->phases[i] + state->jitter[i];
            float base_x = std::sin(phase) * 0.5F;
            float base_y = std::sin(phase * 1.5F) * 0.4F;

            float dist = std::sqrt(base_x * base_x + base_y * base_y);
            float expand = (dist > 0.001F) ? state->expansion * 0.3F / dist : 0.0F;

            float brightness = 0.6F + kick_e * 1.0F;

            path->update_control_point(i, { .position = glm::vec3(
                                                base_x * (1.0F + expand),
                                                base_y * (1.0F + expand), 0.0F),
                .color = glm::vec3(brightness, brightness * 0.85F,
                    brightness * 0.6F),
                .thickness = 2.0F + bass_e * 3.0F });
        }
    });

    // === INTERACTION ===

    on_key_pressed(window, IO::Keys::Space, [state]() {
        state->sequencing = !state->sequencing;
    });

    on_key_pressed(window, IO::Keys::C, [state]() {
        state->chaos_mode = !state->chaos_mode;
    });

    on_key_pressed(window, IO::Keys::N1, [kick, snare, hat, clap]() {
        route_node(kick, { 0 }, 1.5);
        route_node(snare, { 0 }, 1.5);
        route_node(hat, { 0 }, 1.5);
        route_node(clap, { 0 }, 1.5);
    });

    on_key_pressed(window, IO::Keys::N2, [kick, snare, hat, clap]() {
        route_node(kick, { 1 }, 1.5);
        route_node(snare, { 1 }, 1.5);
        route_node(hat, { 1 }, 1.5);
        route_node(clap, { 1 }, 1.5);
    });

    on_key_pressed(window, IO::Keys::N3, [kick, snare, hat, clap]() {
        route_node(kick, { 0, 1 }, 2.0);
        route_node(snare, { 0, 1 }, 2.0);
        route_node(hat, { 0, 1 }, 2.0);
        route_node(clap, { 0, 1 }, 2.0);
    });

    on_mouse_pressed(window, IO::MouseButtons::Left,
        [window, path](double x, double y) {
            glm::vec2 pos = normalize_coords(x, y, window);
            path->add_control_point({ .position = glm::vec3(pos, 0.0F),
                .color = glm::vec3(1.0F, 0.9F, 0.4F),
                .thickness = 3.0F });
        });
}
```

</div> </div> </div>

## Links

- **Source**: [github.com/MayaFlux/MayaFlux](https://github.com/MayaFlux/MayaFlux)
- **Website**: [mayaflux.org](https://mayaflux.org)
- **Docs**: [mayaflux.github.io/MayaFlux](https://mayaflux.github.io/MayaFlux)
- **License**: GPLv3
