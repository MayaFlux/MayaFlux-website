---
build:
  render: never
  list: never
---

**Important:** Quick explaination of timing models

``` cpp
void compose() {
    auto sine = vega.Sine(2.0f) | Audio[0];

    sine->on_tick([](auto& ctx) {
        // fires 48000 times per second
        // (void)ctx;
        std::cout << ctx.value << " at audio rate\n";
    });

    auto point_node = vega.PointNode() | Graphics;

    point_node->on_tick([](auto& ctx) {
        // fires 60 times per second
        // (void)ctx;
        std::cout << ctx.value << " at graphics rate\n";
    });
}
```

The Audio node ticks at sample rate. The Graphics node ticks at frame rate. Same method, two orders of magnitude apart in frequency. `| Audio[0]` and `| Graphics` do not just register the node with different subsystems; they determine the entire temporal contract of every hook attached to that node. `on_tick` on a Graphics node is frame-rate code. `on_tick` on an Audio node is DSP-rate code. Putting expensive work into an audio-rate `on_tick` stalls the audio thread. This is why `on_impulse`, `on_count`, and `on_increment` exist: they give you audio-rate precision with call frequency you can reason about.

The previous example with the metro and `on_impulse` did not explore the side effects of time. For metro, there is none as time is fixed. But for nodes, while `on_tick` itself does not beyond rate of the backend/subsystem responsible for ticking, `on_impulse` and similar hooks are dependent on the logic of the signal. For instance, the impulse rate is not constant if it were frequency modulated. Or if the node in question were inherently dependent on other signals, input sources, non deterministic logic, or any other factor that could affect the timing of the used hook callback.

#### Tutorial: Counter driving a path

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    window->show();

    auto path = vega.PathGeneratorNode(
        Kinesis::InterpolationMode::CATMULL_ROM, 32, 128
    ) | Graphics;
    path->set_path_color(glm::vec3(0.4f, 0.8f, 1.0f));
    path->set_path_thickness(2.0f);

    auto buffer = vega.GeometryBuffer(path) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    auto clock = vega.Impulse(0.5f);
    clock->set_frequency_modulator(vega.Sine(0.08f));

    auto counter = vega.Counter(64) | Audio[0];
    counter->set_reset_trigger(clock);

    counter->on_increment([path](auto& ctx) {
        float t     = static_cast<float>(ctx.phase);
        float angle = t * 2.0f * M_PI / 64.0f;
        float r     = 0.3f + 0.5f * std::sin(t * 0.4f);
        glm::vec3 pos(r * std::cos(angle), r * std::sin(angle), 0.0f);
        path->add_control_point({ pos, glm::vec3(1.0f - t / 64.0f, t / 64.0f, 0.6f), 2.0f });
    });

    counter->on_wrap([path](auto&) {
        path->set_path_color(glm::vec3(
            get_uniform_random(0.3f, 1.0f),
            get_uniform_random(0.3f, 1.0f),
            get_uniform_random(0.3f, 1.0f)
        ));
    });
}
```

Run this. A curved path grows across the window, tracing a slowly shifting orbit. The `PathGeneratorNode` ring holds 128 points; old ones age out as new ones arrive. On each wrap the color randomizes - a structural moment with no geometry destroyed.

The Impulse and its Sine modulator need no domain registration - they are processed because the Counter registers the Impulse as its reset trigger, and the Impulse registers the Sine as its frequency modulator. Only the Counter needs `| Audio[0]`.

`on_increment` fires once per counter tick. `on_wrap` fires exactly once at the modulo boundary. Neither is audio-rate in practice: the Impulse fires at 0.5Hz modulated, so `on_increment` fires at that rate regardless of the 48000Hz substrate underneath.

Example: Counter with `on_count` for structural moments

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    window->show();

    auto path = vega.PathGeneratorNode(
        Kinesis::InterpolationMode::CATMULL_ROM, 32, 128
    ) | Graphics;
    path->set_path_color(glm::vec3(1.0f, 0.5f, 0.2f));
    path->set_path_thickness(2.5f);

    auto buffer = vega.GeometryBuffer(path) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    auto clock   = vega.Impulse(0.5f);
    auto counter = vega.Counter(32) | Audio[0];
    counter->set_reset_trigger(clock);

    counter->on_increment([path](auto& ctx) {
        float t     = static_cast<float>(ctx.phase);
        float angle = t * 2.0f * M_PI / 32.0f;
        float r     = 0.5f + 0.2f * std::cos(t * 1.3f);
        path->add_control_point({
            glm::vec3(r * std::cos(angle), r * std::sin(angle), 0.0f),
            glm::vec3(0.9f, 0.5f, 0.2f),
            2.0f
        });
    });

    counter->on_count(8,  [path](auto&) { path->set_path_color(glm::vec3(0.2f, 0.9f, 0.5f)); });
    counter->on_count(16, [path](auto&) { path->set_path_color(glm::vec3(0.5f, 0.2f, 0.9f)); });
    counter->on_count(24, [path](auto&) { path->set_path_color(glm::vec3(0.9f, 0.2f, 0.2f)); });

    counter->on_wrap([path](auto&) {
        path->set_path_color(glm::vec3(1.0f, 0.5f, 0.2f));
    });
}
```

The path grows in four color phases: orange through 8 steps, then green, then purple, then red, then wraps and resets the color. Change the Impulse frequency and the phase structure stays identical - only the wall-time duration of each phase changes. Sequence structure is decoupled from speed.

Example: Temporal Logic as a gate

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    window->show();

    auto path = vega.PathGeneratorNode(
        Kinesis::InterpolationMode::CATMULL_ROM, 20, 256
    ) | Graphics;
    auto gbuf = vega.GeometryBuffer(path) | Graphics;
    gbuf->setup_rendering({ .target_window = window });

    auto lfo = vega.Sine(0.3f) | Audio[0];

    auto gate = vega.Logic(
        [](double input, double elapsed) -> bool {
            double wn = std::fmod(elapsed, 6.0);
            return wn > 1.0 && wn < 4.0;
        }
    ) | Audio[1];
    gate->set_input_node(lfo);

    auto& n     = make_persistent(0U);
    auto& x_pos = make_persistent(-0.8f);

    gate->while_true([&n, path, lfo, &x_pos](auto&) {
        if (++n % 800 != 0) return;
        float v = static_cast<float>(lfo->get_last_output());
        float x = std::fmod(static_cast<float>(n) / 800.0f, 533.0f) / 533.0f * 1.6f - 0.8f;
        path->add_control_point({
            .position  = glm::vec3((x * x_pos) + get_uniform_random(-0.05f, 0.5f),
                                    v * 0.6f + get_uniform_random(-0.05f, 0.05f), 0.0f),
            .color     = glm::vec3(0.9f, get_exponential_random(), 0.5f + 0.4f * std::abs(v)),
            .thickness = (float)get_uniform_random(0.02f, 1.6f)
        });
        x_pos += 0.006f;
    });

    gate->on_change_to(false, [path, &n, &x_pos](auto&) {
        auto pts = path->get_control_points();
        for (auto& pt : pts) {
            pt.color = glm::vec3(
                get_uniform_random(0.3f, 1.0f),
                get_uniform_random(0.3f, 1.0f),
                get_uniform_random(0.3f, 1.0f));
        }
        path->set_control_points(pts);
        n     = 0;
        x_pos = -0.8f;
    });
}
```

For the first second of every six nothing draws. From second 1 to 4 the gate opens and the path accumulates. At second 4 the gate closes: existing points are recolored in place and cursors reset. The LFO is not the criterion - elapsed time is. The signal is only the material being drawn.

`gate` needs `| Audio[1]` because Logic is a Generator the scheduler owns independently. It pulls its source via `set_input_node`. The `elapsed` parameter is `m_temporal_time`, which increments by `1.0 / sample_rate` on every sample - sample-counted elapsed time, not wall-clock.

------------------------------------------------------------------------


{{< tutorial-detail title="Deep dive" >}}
Metro is a timer. `on_impulse` is a side effect of a signal. `on_increment` fires at sequence positions. `while_true` fires for as long as a condition holds. `on_change_to` fires exactly once at the transition. These are not variations on a theme - they are different shapes of time: periodic, positional, conditional, durational.

The Logic node makes the condition explicit and composable. The criterion can be a threshold, a pattern across a history window, a conjunctive test across multiple signals, or purely temporal. The signal fed into the Logic node is the material; the lambda is the gate.


{{< tutorial-detail title="Expansion 1: What `ctx.phase` and `ctx.value` carry in Counter" >}}


For Counter, `ctx.phase` carries the raw integer count cast to `double`: 0 at the first increment, 1 at the second, up to `modulo - 1` before the wrap. Use it for positional logic inside `on_increment` - angle calculations, array indexing, color interpolation keyed to position.

`ctx.value` carries the normalized output: `count / modulo` when modulo is nonzero, raw count as double when modulo is zero. This is what the Counter outputs as a signal to downstream nodes. The two fields serve different consumers: `ctx.phase` for callback positional logic, `ctx.value` for signal routing downstream.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: Reset trigger edge detection" >}}


The reset trigger is edge-sensitive: Counter watches for a rising edge on the trigger node's output, not a sustained nonzero value. An Impulse node produces exactly one nonzero sample per cycle, so it is a natural trigger source.

A Logic node with sustained `while_true` output would re-trigger every sample it is high, effectively freezing the counter at zero. If you want a reset tied to a Logic condition, use `on_change_to(true, ...)` to fire a one-shot event rather than connecting sustained output directly as a trigger.

Any node whose output crosses zero on the event you care about serves as a valid trigger: a threshold-crossing on an envelope, a Logic node detecting a specific condition, an InputNode receiving a MIDI note.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: `on_change_to` vs `while_true` vs `on_tick`" >}}


`on_tick` fires on every sample the node processes. For a Logic node at Audio rate that is 48000 times per second. Use it when you need the raw boolean stream as data.

`on_change_to(true, ...)` fires exactly once at the sample where the output transitions from false to true. It is edge-sensitive: sustained true output does not re-fire it. Use it for "begin a new thing": start a new path, change a color, record a timestamp.

`while_true(...)` fires on every sample where the output is currently true. It is level-sensitive: it fires continuously for as long as the condition holds. Use it for "continue doing a thing while the condition holds": accumulating geometry, writing to a buffer, advancing a cursor.

The temporal Logic tutorial uses both: `on_change_to(false, ...)` recolors at the moment the gate closes, `while_true` accumulates during the open window. One fires once at the edge, the other fires continuously until the edge closes.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: Why not `on_tick` for path control?" >}}


`on_tick` on an Audio node fires 48000 times per second. `add_control_point` on a `PathGeneratorNode` pushes to a ring buffer and sets dirty flags. Calling it 48000 times per second to build a path that renders at 60Hz produces 800 control points per frame, almost all redundant. The path geometry regenerates from those control points each frame; excess control points increase CPU work for no visual benefit.

`on_increment` driven by a 0.5Hz Impulse or `while_true` decimated by a sample counter produces only the points that are visually meaningful. `on_tick` is appropriate when the node's per-sample value is itself the material: waveshaping, sample-by-sample analysis, building a data array for FFT input. For anything that interacts with visual geometry, a coarser hook is almost always correct.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: The four Logic modes" >}}


`DIRECT` mode evaluates each sample independently. No memory. Use it when the criterion is purely about the current value: amplitude gates, polarity checks, range membership.

`SEQUENTIAL` mode receives a sliding window of past boolean states derived by thresholding the input against `m_threshold`. The lambda receives a `std::span<bool>` of already-binarized values - it pattern-matches on those, it does not control what enters the history. Use it for run detection, alternation patterns, specific boolean sequences.

`TEMPORAL` mode receives both the current input and elapsed time in seconds. Use it when the criterion involves duration or periodic scheduling: "only during the second half of each N-second period," "true if enough time has elapsed."

`MULTI_INPUT` mode receives a vector of simultaneous inputs from multiple sources. Use it when the condition requires agreement across signals: two signals both above threshold, majority voting, cross-signal correlation.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}




