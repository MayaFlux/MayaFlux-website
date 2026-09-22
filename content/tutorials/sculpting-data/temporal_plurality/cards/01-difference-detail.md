---
build:
  render: never
  list: never
---

#### Tutorial: Metro

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    auto points = vega.PointCollectionNode() | Graphics;
    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    auto& angle       = make_persistent(0.0f);
    auto& point_count = make_persistent(0U);

    MayaFlux::schedule_metro(0.016, [points, &angle, &point_count]() {
        float spiral_offset = 0.25f * point_count;
        float r = 0.6f + 0.3f * std::sin((angle + spiral_offset) * 0.4f);
        float x = r * std::cos(angle + spiral_offset);
        float y = r * std::sin(angle + spiral_offset);
        float hue = (angle + spiral_offset) / (2.0f * M_PI);
        glm::vec3 color(
            std::abs(std::sin(hue * 3.14f)),
            std::abs(std::sin(hue * 3.14f + 2.09f)),
            std::abs(std::sin(hue * 3.14f + 4.19f)));
        points->add_point({ glm::vec3(x, y, 0.0f), color, 6.0f });
        angle += 0.18f;
        ++point_count;
    });
}
```

Run this. A spiral of colored points grows across the window at a fixed pace. One point every 16ms, regardless of anything else in the system. The rate is a wall-time promise measured in samples internally, but from the outside it behaves like a fixed interval.

#### Tutorial: Impulse node with `on_impulse`

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    auto points = vega.PointCollectionNode() | Graphics;
    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    auto clock    = vega.Impulse(2.0f) | Graphics;
    auto rate_mod = vega.Sine(0.1f) | Graphics;
    clock->set_frequency_modulator(rate_mod);

    auto& angle = make_persistent(0.0f);
    auto& count = make_persistent(0U);

    clock->on_impulse([points, clock, &angle, &count](const Nodes::NodeContext& ctx) {
        float r = 0.6f + 0.3f * std::sin(angle * 0.4f);
        float x = r * std::cos(angle);
        float y = r * std::sin(angle);
        float hue = angle / (2.0f * M_PI);
        glm::vec3 color(
            std::abs(std::sin(hue * 3.14f)),
            std::abs(std::sin(hue * 3.14f + 2.09f)),
            std::abs(std::sin(hue * 3.14f + 4.19f)));
        points->add_point({ glm::vec3(x, y, 0.0f), color, 6.0f });
        angle += 0.18f;

        ++count;
        if (count == 200) {
            clock->set_frequency(6.0f);
        }
    });
}
```

Run this. The same spiral grows, but the pace breathes. It slows down, speeds up, slows again. After 200 points the frequency jumps from 2 Hz to 6 Hz and stays there, triple the original rate, with no teardown and no new callback. The `clock` capture lets the callback reach back into the node that fired it and change its own future rate on the fly.

Change `0.1f` to `0.5f` on the Sine. The breathing is faster and more dramatic. Change the Impulse base frequency from `2.0f` to `6.0f`. More points, still breathing. The modulator and the base rate are independent parameters, both live, both modifiable without touching the callback.


{{< tutorial-detail title="Deep dive" >}}
Metro does not know about frequency. It knows about seconds. You give it `0.016` and it fires every 0.016 seconds, period. It has no concept of modulation. To change its rate you would have to cancel and reschedule it, or manage the timing yourself inside the callback.

`on_impulse` fires when the Impulse node fires. The Impulse node is a signal. Its frequency is a parameter that can be modulated by any other node, changed at any time with `set_frequency()`, or driven by an external input. The callback rate is a consequence of that signal, not a separately managed thing.

Metro is a scheduled timer. `on_impulse` is a side effect of computation that was already happening. The Impulse node processes samples regardless of whether anything is listening. `on_impulse` attaches to it.


{{< tutorial-detail title="Expansion 1: What metro actually is" >}}


Metro is a coroutine. When you call `schedule_metro(0.016, callback)`, MayaFlux creates a `Vruta::SoundRoutine` that runs on the `TaskScheduler`:

``` cpp
Vruta::SoundRoutine metro_body(Vruta::TaskScheduler& scheduler,
                               double interval_seconds,
                               std::function<void()> callback) {
    uint64_t interval_samples = scheduler.seconds_to_samples(interval_seconds);
    auto& promise = co_await Kriya::GetAudioPromise{};

    while (true) {
        if (promise.should_terminate) break;
        callback();
        co_await Kriya::SampleDelay{ interval_samples };
    }
}
```

`seconds_to_samples(0.016)` at 48000 Hz gives 768 samples. The coroutine fires, suspends for exactly 768 samples, fires again. Time is counted in samples because the audio engine is the timing substrate. The scheduler advances a `SampleClock` every buffer cycle, and the coroutine resumes when the clock passes its target.

This is why metro is sample-accurate but not frequency-aware. It sleeps for N samples. That N does not change unless you cancel and reschedule.

`angle` and `point_count` are declared with `make_persistent` rather than as plain locals. The metro callback outlives `compose()` - the function returns immediately after scheduling the coroutine, destroying all stack variables. `make_persistent` allocates the value in a store with process lifetime and returns a reference the callback can safely hold across that boundary.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: What `on_impulse` actually is" >}}


`Impulse::on_impulse` registers a callback that fires inside `notify_tick` only when `m_impulse_occurred` is true. That flag is set in `process_sample` when `m_phase < m_phase_inc`, which is the sample where the phase accumulator crosses the cycle boundary.

The node is processed every sample at audio rate by the buffer and scheduler infrastructure. Your callback fires on the precise sample where the impulse occurs. That sample is determined by `m_phase_inc = frequency / sample_rate`. When frequency is modulated, `m_phase_inc` changes on each sample, so the interval between impulses changes continuously.

`set_frequency()` writes directly to `m_frequency` and recomputes `m_phase_inc` in the same call. This takes effect on the next call to `process_sample`. No coroutine is involved. No teardown. The node's next cycle boundary simply arrives sooner or later depending on the new `m_phase_inc`.

Metro lives in scheduler time. `on_impulse` lives in signal time. Signal time can be warped by any node, modulated by any source, driven by external input, or driven from inside the callback itself. Scheduler time cannot.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: Reading the context" >}}


The `NodeHook` type is declared against the base `NodeContext`, so the callback signature is:

``` cpp
clock->on_impulse([](const Nodes::NodeContext& ctx) {
    // ctx is available here
});
```

The actual object passed is always a `GeneratorContext` for any node that derives from `Generator`, including `Impulse`. `GeneratorContext` extends `NodeContext` with three additional fields: `frequency`, `amplitude`, and `phase`. These reflect the node's state at the exact sample the impulse fired, after any modulation has been applied for that sample.

`ctx.value` is the output at impulse time - for `Impulse` this is the amplitude, typically `1.0`. This is enough for most callbacks.

When you need the generator fields, use `as<GeneratorContext>()`:

``` cpp
clock->on_impulse([](const Nodes::NodeContext& ctx) {
    if (auto* gen = ctx.as<Nodes::Generator::GeneratorContext>()) {
        float current_freq  = gen->frequency;
        double current_phase = gen->phase;
    }
});
```

`phase` at an impulse is always near zero - the accumulator has just crossed the cycle boundary and been reset. `frequency` is the effective frequency after modulation, not the base parameter. If a Sine modulator is pushing the frequency around, `gen->frequency` gives you the actual Hz value at the moment that impulse fired, not what you passed to the constructor.

`as<T>()` returns `nullptr` if the type does not match. For `Impulse`, the concrete type is always `GeneratorContext`, so the null check is a safety idiom rather than a branch you expect to take.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: When to use which" >}}


Metro is appropriate when the interval is a fixed design decision: "update this visualization every 16ms," "poll this resource every second." The interval is a system parameter, not a signal.

`on_impulse` is appropriate when the rate should be a function of other computation: rhythms derived from data, tempos that drift, event densities that respond to material. It is also appropriate when you want a single node to serve as a clock for multiple listeners, each attached independently without coordination overhead.

The two mechanisms are not mutually exclusive. Metro and `on_impulse` can run simultaneously in the same compose, writing into the same geometry, at their own independent rates.

A useful heuristic: if you find yourself computing a new interval inside a metro callback and wishing you could just pass a node, switch to `on_impulse` with a modulator.

## Try It: Both at once

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });
    auto points = vega.PointCollectionNode() | Graphics;
    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    auto& angle       = make_persistent(0.0f);
    auto& imp_count   = make_persistent(0U);
    auto& point_count = make_persistent(0U);

    MayaFlux::schedule_metro(0.016, [points, &angle, &point_count]() {
        float spiral_offset = 0.25f * point_count;
        float r = 0.6f + 0.3f * std::sin((angle + spiral_offset) * 0.4f);
        float x = r * std::cos(angle + spiral_offset);
        float y = r * std::sin(angle + spiral_offset);
        float hue = (angle + spiral_offset) / (2.0f * M_PI);
        glm::vec3 color(
            std::abs(std::sin(hue * 3.14f)),
            std::abs(std::sin(hue * 3.14f + 2.09f)),
            std::abs(std::sin(hue * 3.14f + 4.19f)));
        points->add_point({ glm::vec3(x, y, 0.0f), color, 6.0f });
        angle += 0.18f;
        ++point_count;
    });

    auto clock    = vega.Impulse(20.0f) | Audio[0];
    auto rate_mod = vega.Sine(0.01f, 80) | Graphics;
    clock->set_frequency_modulator(rate_mod);

    clock->on_impulse([points, clock, &angle, &imp_count](const Nodes::NodeContext& ctx) {
        float r = 0.6f + 0.3f * std::sin(angle * 0.4f);
        float x = r * std::cos(angle);
        float y = r * std::sin(angle);
        float hue = angle / (2.0f * M_PI);
        glm::vec3 color(
            std::abs(std::sin(hue * 3.14f)),
            std::abs(std::sin(hue * 3.14f + 2.09f)),
            std::abs(std::sin(hue * 3.14f + 4.19f)));
        points->add_point({ glm::vec3(x, y, 0.0f), color, 12.0f });
        angle += 0.78f;
        ++imp_count;
        if (imp_count == 200) {
            clock->set_frequency(6.0f);
        }
    });
}
```

Metro lays down small points at a fixed 16ms grid. The Impulse fires at 20 Hz, modulated by a slow Sine, writing larger points at a different angular step. Both share `angle` and `points` - the spiral they build is the product of two clocks running at different rates into the same collection. At 200 impulses the Impulse rate drops to 6 Hz; the metro does not notice and does not change.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}




