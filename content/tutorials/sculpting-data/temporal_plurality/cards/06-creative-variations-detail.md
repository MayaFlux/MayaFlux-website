---
build:
  list: never
---

The previous cards all react: to clocks, to conditions, to external events. This card generates. `schedule_pattern` and `line` produce values according to computational rules, not in response to anything. Everything downstream reads from them.

#### Tutorial: Pattern as a generative sequencer

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1400, 900 });
    window->show();

    // Kick: pitched envelope on a sine
    auto kick_env   = vega.Phasor(0.0f) | Audio[0];
    auto kick_shape = vega.Polynomial([](double x) {
        return std::exp(-x * 18.0);
    }) | Audio[0];
    kick_shape->set_input_node(kick_env);
    auto kick = vega.Sine(55.0f) | Audio[0];
    kick->set_amplitude_modulator(kick_shape);

    // Snare: filtered noise with exponential decay
    auto noise = vega.Random();
    noise->set_amplitude(0.08);
    std::vector<double> b = { 0.0675, 0.0, -0.0675 };
    std::vector<double> a = { 1.0, -1.1430, 0.4128 };
    auto snare_iir = vega.IIR(noise, b, a);
    snare_iir->set_gain(0.15);
    auto snare_shape = vega.Polynomial([](double x) { return std::exp(-x * 30.0); });
    snare_shape->set_input_node(snare_iir);
    auto snare_env = vega.Phasor(0.0f) | Audio[1];
    snare_env->set_amplitude_modulator(snare_shape);
    snare_env->set_frequency_modulator(snare_shape);

    // Hat: high-frequency sine, very short decay
    auto hat_env   = vega.Phasor(0.0f) | Audio[0];
    auto hat_shape = vega.Polynomial([](double x) { return std::exp(-x * 60.0); }) | Audio[0];
    hat_shape->set_input_node(hat_env);
    auto hat = vega.Sine(6000.0f) | Audio[0];
    hat->set_amplitude_modulator(hat_shape);

    // Visual: three geometry layers in one composite buffer
    auto hits       = vega.PointCollectionNode() | Graphics;
    auto connector  = vega.PathGeneratorNode(Kinesis::InterpolationMode::CATMULL_ROM, 16, 64) | Graphics;
    auto sweep_mesh = vega.MeshWriterNode(64) | Graphics;

    {
        std::vector<MeshVertex> v;
        std::vector<uint32_t> idx;
        const int SEGS = 16;
        v.push_back({ {0.f,0.f,0.f}, {0.5f,0.5f,1.f}, 1.f, {}, {0,0,1}, {1,0,0} });
        for (int i = 0; i <= SEGS; ++i) {
            float a = float(i) / SEGS * 2.f * M_PI;
            v.push_back({ {0.35f*std::cos(a), 0.35f*std::sin(a), 0.f},
                          {0.2f,0.6f,0.9f}, 0.f, {}, {0,0,1}, {1,0,0} });
            if (i < SEGS)
                idx.insert(idx.end(), { 0u, uint32_t(i+1), uint32_t(i+2) });
        }
        sweep_mesh->set_mesh(v, idx);
    }

    auto composite = vega.CompositeGeometryBuffer() | Graphics;
    composite->add_geometry("hits",  hits,
        Portal::Graphics::PrimitiveTopology::POINT_LIST,    window);
    composite->add_geometry("path",  connector,
        Portal::Graphics::PrimitiveTopology::LINE_STRIP,    window);
    composite->add_geometry("sweep", sweep_mesh,
        Portal::Graphics::PrimitiveTopology::TRIANGLE_LIST, window);

    struct StepData {
        bool kick, snare, hat;
        float x, y;
        glm::vec3 color;
    };

    float orbit_angle = 0.f;

    MayaFlux::schedule_pattern(
        [](uint64_t step) -> std::any {
            const uint64_t s = step % 8;
            StepData d;
            d.kick  = (s == 0 || s == 3 || s == 6);
            d.snare = (s == 2 || s == 6);
            d.hat   = (s % 2 == 1);
            const float angle = float(step) * 0.37f;
            const float r = 0.5f + 0.2f * std::sin(float(step) * 0.13f);
            d.x = r * std::cos(angle);
            d.y = r * std::sin(angle);
            d.color = {
                std::abs(std::sin(float(step) * 0.11f)),
                std::abs(std::sin(float(step) * 0.07f + 1.f)),
                std::abs(std::sin(float(step) * 0.05f + 2.f)),
            };
            return d;
        },
        [kick_env, snare_env, hat_env,
         hits, connector, sweep_mesh, &orbit_angle](std::any val) {
            const auto& d = std::any_cast<const StepData&>(val);

            if (d.kick)  kick_env->reset();
            if (d.snare) snare_env->reset();
            if (d.hat)   hat_env->reset();

            hits->add_point({ glm::vec3(d.x, d.y, 0.f), d.color, 8.f });
            connector->add_control_point({ glm::vec3(d.x, d.y, 0.f), d.color, 1.5f });

            orbit_angle += d.kick ? 0.4f : 0.08f;
            if (d.kick) {
                auto verts = sweep_mesh->get_mesh_vertices();
                verts[0].color = d.color;
                sweep_mesh->set_mesh_vertices(verts);
            }
        },
        0.125
    );
}
```

Run this. You hear a Euclidean rhythm: kick on beats 0, 3, 6 of 8; snare on 2 and 6; hat on odd steps. Simultaneously, colored points accumulate at positions computed from the same step index, a Catmull-Rom path connects them, and a triangle fan rotates faster on kick steps. The rhythm and the visual share one source: the pattern function.

Change the step period from `0.125` to `0.0833`. Both audio and visual accelerate together. Change `% 8` to `% 12`. The rhythm changes shape and so does the visual orbit. The function is the single point of compositional control.

#### Tutorial: `line` as a parametric parameter

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });

    auto points = vega.PointCollectionNode() | Graphics;
    auto buf    = vega.GeometryBuffer(points) | Graphics;
    buf->setup_rendering({ .target_window = window });
    window->show();

    auto phasor = vega.Phasor(0.0f) | Audio[0];
    auto env    = vega.Polynomial([](double x) { return std::exp(-x * 20.0); }) | Audio[0];
    env->set_input_node(phasor);
    auto drum   = vega.Sine(110.0f) | Audio[0];
    drum->set_amplitude_modulator(env);

    // Threshold rises from 0.0 to 1.0 over 8 seconds
    MayaFlux::schedule_task("reveal",
        MayaFlux::create_line(0.0f, 1.0f, 8.0f, 128, true));

    MayaFlux::schedule_pattern(
        [](uint64_t step) -> std::any {
            auto is_prime = [](uint64_t n) {
                if (n < 2) return false;
                for (uint64_t i = 2; i * i <= n; ++i)
                    if (n % i == 0) return false;
                return true;
            };
            const float density = is_prime(step % 32) ? 0.9f : 0.3f;
            const float angle   = float(step) * 0.41f;
            const float r       = 0.4f + 0.35f * std::abs(std::sin(float(step) * 0.17f));
            return std::make_tuple(density, r * std::cos(angle), r * std::sin(angle));
        },
        [points, phasor](std::any val) {
            auto [density, x, y] =
                std::any_cast<std::tuple<float, float, float>>(val);

            phasor->reset();

            float* threshold = MayaFlux::get_line_value("reveal");
            if (threshold && density > *threshold) {
                points->add_point({
                    glm::vec3(x, y, 0.f),
                    glm::vec3(density, 1.f - density, 0.3f),
                    6.f + density * 8.f
                });
            }
        },
        0.1
    );
}
```

Run this. You hear a steady rhythm. The window starts empty. Over eight seconds, points appear progressively: first only the sparse non-prime steps (low density, revealed early), then gradually the prime steps emerge (high density, revealed later). Audio and visual were always computing the same thing; only the threshold determined visibility.

The `line` is not an event. It does not call anything. `get_line_value("reveal")` is a pointer into the coroutine frame advancing on each step. No push, no subscription, no coordination. The parametric value is ambient state.


{{< tutorial-detail title="Deep dive" >}}
`schedule_pattern` separates the question of *what a step produces* from the question of *what happens when it fires*. The generator function is pure: it takes a step index and returns a value. The callback is the side effect: it reads that value and acts on it. Because the generator has no side effects, it can be reasoned about in isolation, tested, or replaced without touching the callback.

`line` is orthogonal to all of this: it is not a callback mechanism, not a signal, not a hook. It is a continuously-advancing scalar that ambient code can poll. Anything that can call `get_line_value` can read it, from any thread that the scheduler owns.


{{< tutorial-detail title="Expansion 1: `std::any` return type" >}}


The pattern function returns `std::any`, which accepts any copyable type: a plain `float`, a struct, a `std::tuple`, a `std::vector`. The callback receives the same `std::any` and casts it with `std::any_cast`. The cast must match the returned type exactly; a mismatch throws at runtime.

The idiomatic form is to define a local struct for step data, return it, and cast with `std::any_cast<const YourStruct&>(val)`. The reference cast avoids a copy. For structs that both the generator lambda and the callback need to name, define at file scope rather than inside `compose()`.

For simple cases, `std::tuple` works without defining a struct:

``` cpp
// Generator
return std::make_tuple(frequency, amplitude, duration);

// Callback
auto [freq, amp, dur] = std::any_cast<std::tuple<float,float,float>>(val);
```


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: Step index as the only state" >}}


The generator receives a `uint64_t` that increments by one on each step and never resets. All sequence structure must be derived from this single value. Periodicity comes from modulo: `step % 8` gives an 8-step cycle. Variation over longer timescales comes from secondary modulo or from direct arithmetic on the raw index.

This has a useful property: the sequence is deterministic and replayable. Given any step index, you can compute exactly what that step produced without running all previous steps. There is no hidden state that drifts over time.

Layering two modulos produces a phrase structure: steps with a large-period cycle can change the mode or register while the small-period cycle handles beat content.

``` cpp
[](uint64_t step) -> std::any {
    const uint64_t beat   = step % 8;   // 8-step rhythm cycle
    const uint64_t phrase = step % 32;  // 32-step phrase cycle

    float root_hz = (phrase < 16) ? 220.f : 330.f; // phrase changes the root
    bool on_beat  = (beat == 0 || beat == 4);
    // ...
}
```


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: `line` is a pointer into a coroutine frame" >}}


`get_line_value("reveal")` returns `float*`, pointing directly into the coroutine frame of the named line task. The coroutine writes to this float on each scheduler step; the callback reads it with a pointer dereference. There is no synchronisation cost because both the coroutine and the metro callback run on the scheduler thread.

If you read `get_line_value` from a thread outside the scheduler (e.g. an event callback), take a copy rather than holding the pointer across a yield point.

The pointer is valid until the task is cancelled or the scheduler is destroyed. After the line reaches its end value and `restartable` is false, the coroutine suspends permanently but the frame is not destroyed while the scheduler holds the shared_ptr. The pointer remains valid and readable; it just stops changing.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: `CompositeGeometryBuffer` draw order and topology" >}}


Geometry collections inside a `CompositeGeometryBuffer` are drawn in insertion order: the first `add_geometry` call renders first, the last renders on top. Points drawn last appear above lines drawn earlier. For the audio-visual examples here, hits on top of path is the correct order.

Each collection has its own `RenderProcessor` with independent topology and shader. `POINT_LIST`, `LINE_STRIP`, and `TRIANGLE_LIST` can coexist in the same composite buffer because each processor has its own pipeline. They share one `VKBuffer` for upload efficiency but issue separate draw calls.

The nodes passed to `add_geometry` must already be registered via `| Graphics` before the call. The composite buffer does not own their registration; it only aggregates their draw calls. This means the same node can appear in multiple composite buffers targeting different windows without duplication of upload work.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: Pattern vs metro for generative work" >}}


Metro fires a void callback at a fixed interval. It has no concept of sequence position. If you need to know "which step is this" inside a metro callback, you have to maintain that counter yourself, and it becomes part of the callback's captured state.

`schedule_pattern` externalises that counter. The step index is provided by the infrastructure, not managed by the callback. The separation means the generator can be stateless: given the same index it always returns the same value. Callbacks that modify shared state (adding points, triggering audio) stay in the callback where they belong.

Metro is appropriate when the work is ongoing and uniform and has no positional identity. Pattern is appropriate when each firing is a distinct position in a sequence, even if that sequence is infinite.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 6: Combining `line` with pattern for reveal and collapse arcs" >}}


A rising line used as a threshold produces a reveal: early steps are sparse, later steps dense. A falling line produces a collapse: all content is initially visible, then progressively filtered out. A line that oscillates (via a `restartable` ping-pong) produces a breathing density that rises and falls without any explicit control mechanism.

The key insight is that the line and the pattern are not coordinated. The line advances on its own schedule; the pattern fires on its own schedule. The callback is the only point of contact, and it reads the line value at the moment it fires. Any phase relationship between them emerges from the ratio of their rates, not from explicit synchronisation.

``` cpp
// 6-second oscillating threshold, restarts indefinitely
MayaFlux::schedule_task("breath",
    MayaFlux::create_line(0.0f, 1.0f, 6.0f, 256, true));

// Callback polls it at each pattern step
float* thr = MayaFlux::get_line_value("breath");
if (thr && density > *thr) { /* draw */ }
```

Try It: Multi-dependency composition

Pattern, line, and Logic-as-clock from Card 2 acting on the same `CompositeGeometryBuffer` simultaneously. Three independent temporal mechanisms, no coordination between them.

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1400, 900 });

    auto hits = vega.PointCollectionNode() | Graphics;
    auto path = vega.PathGeneratorNode(Kinesis::InterpolationMode::CATMULL_ROM, 20, 96) | Graphics;

    auto composite = vega.CompositeGeometryBuffer() | Graphics;
    composite->add_geometry("hits", hits,
        Portal::Graphics::PrimitiveTopology::POINT_LIST,  window);
    composite->add_geometry("path", path,
        Portal::Graphics::PrimitiveTopology::LINE_STRIP,  window);
    window->show();

    auto phasor_a = vega.Phasor(0.0f) | Audio[0];
    auto env_a    = vega.Polynomial([](double x) { return std::exp(-x * 15.0); }) | Audio[0];
    env_a->set_input_node(phasor_a);
    auto tone_a   = vega.Sine(220.0f) | Audio[0];
    tone_a->set_amplitude_modulator(env_a);

    auto phasor_b = vega.Phasor(0.0f) | Audio[1];
    auto env_b    = vega.Polynomial([](double x) { return std::exp(-x * 25.0); }) | Audio[1];
    env_b->set_input_node(phasor_b);
    auto tone_b   = vega.Sine(330.0f) | Audio[1];
    tone_b->set_amplitude_modulator(env_b);

    struct Hit {
        float x, y, size;
        glm::vec3 color;
        bool secondary;
    };

    // Mechanism 1: pattern drives hit positions and audio triggers
    MayaFlux::schedule_pattern(
        [](uint64_t step) -> std::any {
            const uint64_t s = step % 16;
            const bool primary   = (s == 0 || s == 5 || s == 8 || s == 13);
            const bool secondary = (s == 3 || s == 10);
            const float angle    = float(step) * 0.29f;
            const float r        = primary ? 0.6f : secondary ? 0.4f : 0.25f;
            return Hit {
                r * std::cos(angle), r * std::sin(angle),
                primary ? 14.f : secondary ? 9.f : 5.f,
                { std::abs(std::sin(float(step) * 0.09f)),
                  std::abs(std::sin(float(step) * 0.13f + 1.f)),
                  std::abs(std::sin(float(step) * 0.07f + 2.f)) },
                secondary
            };
        },
        [phasor_a, phasor_b, hits](std::any val) {
            const auto& h = std::any_cast<const Hit&>(val);
            hits->add_point({ glm::vec3(h.x, h.y, 0.f), h.color, h.size });
            if (h.secondary) phasor_b->reset();
            else             phasor_a->reset();
        },
        1.0 / 12.0
    );

    // Mechanism 2: line controls path color saturation over 12 seconds
    MayaFlux::schedule_task("saturation",
        MayaFlux::create_line(0.1f, 1.0f, 12.0f, 256, false));

    // Mechanism 3: Logic with sequential criterion gates path writing
    auto lfo  = vega.Sine(0.11f) | Graphics;
    auto gate = std::make_shared<Nodes::Generator::Logic>(
        [](std::span<bool> history) -> bool {
            if (history.size() < 6) return false;
            int alt = 0;
            for (size_t i = 1; i < 6; ++i)
                if (history[i] != history[i-1]) ++alt;
            return alt >= 4;
        }, 6) | Audio[0];
    lfo >> gate;

    gate->while_true([path, lfo](const Nodes::NodeContext&) {
        const float v = float(lfo->get_last_output());
        const float r = 0.5f + 0.3f * v;
        const float a = float(std::chrono::duration<double>(
            std::chrono::steady_clock::now().time_since_epoch()).count()) * 0.8f;

        float* sat = MayaFlux::get_line_value("saturation");
        const float s = sat ? *sat : 0.5f;

        path->add_control_point({ glm::vec3(r * std::cos(a), r * std::sin(a), 0.f),
            glm::vec3(s, 0.4f + 0.4f * v * s, 1.f - s * 0.6f), 1.8f });
    });

    gate->on_change_to(false, [path](const Nodes::NodeContext&) {
        path->clear_path();
    });
}
```

Run this. Two tones fire at different pitches in a sparse Euclidean pattern. Points accumulate in orbiting positions. When the LFO's recent history satisfies the zigzag criterion, a path grows alongside the points; its color shifts from desaturated toward fully saturated over twelve seconds via the line. When the criterion fails the path clears.

Three mechanisms act on the same scene without coordinating with each other: the pattern determines when and where; the line determines the color envelope; the Logic determines whether path writing is currently happening. None of them knows the others exist.

The composability here is structural, not incidental. Pattern, line, and Logic hooks are all designed to have no awareness of each other. They share targets (the same geometry nodes, the same audio phasors) but share no state and no coordination mechanism. The scene is the product of their independent action.

This is the digital counterpart to the analog notion of independent voices: each mechanism is a voice, and the result is their superposition. The difference is that here, the voices operate on different temporal substrates (scheduler steps, coroutine frames, signal samples) and interact only through the shared values they write into.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 1: Why the three mechanisms do not need to be aware of each other" >}}


The pattern writes hit positions and triggers audio. The line advances a float. The Logic gate opens and closes path writing. Each of these is a unidirectional write into a shared target. None of them reads from the others.

This is only possible because the targets (geometry nodes, phasors) are designed to accept concurrent writes. `add_point` and `add_control_point` are safe to call from multiple scheduling contexts because geometry nodes use internal dirty flags and deferred upload rather than immediate GPU mutation. The scheduler thread owns all three mechanisms, so there are no actual concurrent writes here, but the architecture permits it.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: The `while_true` decimation pattern" >}}


The Logic gate runs at audio rate (48000 Hz). `while_true` fires on every sample where the output is true. `add_control_point` called 48000 times per second would produce 800 control points per frame against a 60Hz renderer: nearly all redundant.

The example avoids this by using `std::chrono::steady_clock` to compute an angle that advances smoothly over time rather than per sample. The path grows at the rate meaningful to the visual, not at audio rate. An alternative decimation is a sample counter inside the callback, as used in the Card 2 gate example:

``` cpp
gate->while_true([path, &n](const Nodes::NodeContext&) {
    if (++n % 800 != 0) return; // ~60Hz at 48kHz
    path->add_control_point(/* ... */);
});
```


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: Pattern step rate and visual density" >}}


The pattern fires at `1.0 / 12.0` seconds per step, which is 12 steps per second. At this rate, the 16-step cycle completes roughly every 1.3 seconds. Point density on screen is a function of both the rate and the geometry's ring buffer capacity.

Increasing the rate to `1.0 / 24.0` doubles the point density without changing the rhythmic structure. Decreasing it to `0.25` spreads the same pattern over 4 seconds per cycle. The audio and visual both change because both are driven by the same step rate.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}



