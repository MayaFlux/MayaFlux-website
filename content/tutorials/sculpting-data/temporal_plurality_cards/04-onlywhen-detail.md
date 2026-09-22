---
build:
  list: never
---

#### Tutorial: TimedAction : bracketed state

``` cpp
void compose() {
    const std::vector<float> freqs = { 220.0f, 277.0f, 330.0f, 370.0f, 440.0f, 554.0f, 660.0f };

    std::vector<std::shared_ptr<Sine>> sines;
    for (auto f : freqs) {
        auto s = vega.Sine(f) | Audio[0];
        s->set_amplitude(0.08);
        sines.push_back(s);
    }

    auto action = std::make_shared<Kriya::TimedAction>(*get_scheduler());
    store(action);

    auto clock = vega.Impulse(0.15f) | Audio[0];
    clock->on_impulse([sines, freqs, action](const Nodes::NodeContext&) {
        action->execute(
            [sines]() {
                for (auto& s : sines)
                    s->set_frequency(220.0f);
            },
            [sines, freqs]() {
                for (int i = 0; i < (int)sines.size(); ++i)
                    sines[i]->set_frequency(freqs[i]);
            },
            get_uniform_random(0.5, 2.0)
        );
    });
}
```

Seven sines tuned to a major seventh chord run continuously. Every ~6.5 seconds the Impulse fires and the chord collapses: all voices snap to 220Hz. After a random 0.5 to 2 second window they restore to their original frequencies. The collapse and restore are sample-accurate. Rapid impulses do not stack brackets; a new `execute` call cancels the pending restore and restarts the duration from the new trigger point.

Try: change the Impulse to `0.08f` and the duration to `get_uniform_random(4.0, 8.0)`. The bracket now outlasts the impulse period. The next impulse fires while still collapsed, restarting from unison again. The chord barely has time to breathe before the next collapse.

#### Tutorial: `>> Time(N)` — rotating partials

``` cpp
void compose() {
    auto base = vega.Sine(110.0f) | Audio[0];
    base->set_amplitude(0.3);

    const std::vector<float> partials = { 330.0f, 550.0f, 770.0f, 990.0f, 1320.0f, 1650.0f, 2200.0f };

    auto clock = vega.Impulse(1.2f) | Audio[0];
    clock->set_frequency_modulator(vega.Sine(0.05f));

    clock->on_impulse([partials](auto&) {
        float dur = get_uniform_random(0.3, 1.5);
        auto idx = (int)get_uniform_random(0, 6);

        auto s0 = vega.Sine(partials[idx]);
        auto s1 = vega.Sine(partials[(idx + 2) % 7]);
        auto s2 = vega.Sine(partials[(idx + 4) % 7]);

        s0->set_amplitude(0.10);
        s1->set_amplitude(0.07);
        s2->set_amplitude(0.05);

        s0 >> Time(dur)         | Audio[0];
        s1 >> Time(dur * 0.6f)  | Audio[1];
        s2 >> Time(dur * 0.35f) | Audio[0];
    });
}
```

A 110Hz base sustains permanently. Each impulse fires three partials from the array, selected by stepping two positions apart so they always form a consistent intervallic relationship. Each partial has its own duration: the longest on channel 0, a mid-length on channel 1, the shortest back on channel 0. They enter together and leave at different times, so the gesture decays in layers rather than cutting simultaneously.

The slow Sine modulating the clock breathes the impulse density over roughly 20 second cycles. At peak density, partial clusters overlap: several are still alive when the next impulse fires its own set. Change `(idx + 2) % 7` and `(idx + 4) % 7` to `(idx + 1) % 7` and `(idx + 6) % 7` for a tighter cluster plus a distant partial instead of even thirds.


{{< tutorial-detail title="Deep dive" >}}
Everything in the previous cards repeats or runs indefinitely. Metro fires until cancelled. `on_impulse` fires as long as the node processes. EventChain has a fixed arc but no handle on individual moments within it. None of them express "do this, then undo it, after exactly N seconds" or "this node exists in the graph for this duration and no longer."

`TimedAction` is a start/end pair with a sample-accurate duration between them. `>> Time(N)` is graph membership for a bounded duration. Both are about computation that is conditionally present in time, not continuously running.


{{< tutorial-detail title="Expansion 1: What `TimedAction::execute` actually does" >}}


`execute(start, end, duration)` calls `start` immediately, then schedules a `Timer` to call `end` after `duration` seconds. The timer is a coroutine that suspends for exactly that many samples, so the end fires at a sample-accurate position regardless of buffer size or scheduling jitter.

One `TimedAction` holds one bracket at a time. Calling `execute` while a bracket is active cancels the pending end and starts a new one immediately; the previous end function never fires. Rapid re-triggers therefore reset the duration from the new trigger point rather than queuing multiple restores. To get stacked brackets that each fire their own end independently, use separate `TimedAction` instances.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: Why not EventChain for this" >}}


EventChain is declared once with a fixed shape and runs from a known start point. It cannot be re-fired mid-execution in response to a signal, and its duration is set at construction time rather than at trigger time.

`TimedAction` has no predetermined choreography. The same object can be re-executed from different callbacks with different durations computed at trigger time. Here the duration is `get_uniform_random(0.5, 2.0)`: a different value on every impulse, determined by when the impulse fires, not by anything declared in advance.

Use EventChain when the sequence is known in advance and has multiple steps. Use `TimedAction` when you have a single reversible state change that needs to be triggered from anywhere, at any time, with a duration determined by the trigger context.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: What `>> Time(N) | Audio[ch]` actually does" >}}


The expression creates a `TemporalActivation` that registers the node with the audio graph on the specified channel immediately, then schedules a timer to unregister it after N seconds. The node is held alive by the graph registration for that duration with no explicit `store` needed. When the timer fires, the node leaves the graph and is released.

The node has no awareness of its bounded lifetime. It processes normally for the duration, and any modulators or hooks attached to it continue to work. The graph simply stops calling it after N seconds.

This is not the same as setting amplitude to zero. A node at zero amplitude still runs `process_sample` on every sample. A node outside the graph does not. At large counts of simultaneous transient nodes the CPU difference is meaningful.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: The layered decay shape" >}}


Three partials enter the graph simultaneously with durations `dur`, `dur * 0.6`, and `dur * 0.35`. The shortest expires first, leaving a pair. Then the middle one expires, leaving only the longest. Then silence until the next impulse.

This decay shape (full chord, then subset, then single, then silence) emerges from the duration arithmetic alone. No envelope node, no amplitude ramp, no explicit sequencing. The shape is determined entirely by which nodes are alive in the graph at each moment. Change the scale factors to `1.0`, `0.98`, `0.96` and the three partials expire almost simultaneously: a tight staccato cut rather than a layered decay.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: Gate vs Trigger vs Toggle" >}}


`Gate` fires its callback on every sample where the Logic output is currently true (level-sensitive). `Trigger` fires once per rising edge: the single sample where output transitions from false to true. `Toggle` fires on any state change, rising or falling.

All three are coroutines built on the same loop: drive the Logic node one sample at a time, check its output, fire the registered hook when the condition matches. The difference is which hook type is registered (`while_true`, `on_change_to`, or `on_change`).

Gate needs decimation inside the callback when the intended event rate is coarser than sample rate. Trigger does not, because it fires at most once per gate opening by definition. Replacing Gate with Trigger in the bowl example below would fire one partial per Phasor cycle rather than a burst of them over the open window.

## Try It: Singing bowl

``` cpp
void compose() {
    auto fundamental = vega.Sine(196.0f) | Audio[0];
    fundamental->set_amplitude(0.25);
    auto wobble = vega.Sine(0.13f);
    auto wobble_depth = vega.Polynomial([](double x) { return x * 2.8; });
    wobble >> wobble_depth;
    fundamental->set_frequency_modulator(wobble_depth);

    const std::vector<float> bowl_harmonics = {
        392.3f,   // ~2x + asymmetry
        588.7f,   // ~3x
        789.1f,   // ~4x - more inharmonic
        1179.4f,  // ~6x
    };
    for (int i = 0; i < 4; ++i) {
        auto h = vega.Sine(bowl_harmonics[i]) | Audio[0];
        h->set_amplitude(0.12f / (i + 1));
        auto beat = vega.Sine(get_uniform_random(0.03, 0.19));
        auto bdep = vega.Polynomial([i](double x) { return x * (3.0 - i * 0.5); });
        beat >> bdep;
        h->set_frequency_modulator(bdep);
        h->set_amplitude_modulator(
            vega.Sine(static_cast<float>(get_uniform_random(0.07, 0.31))));
    }

    auto rotor     = vega.Phasor(0.11f) | Audio[0];
    auto gate_logic = std::make_shared<Nodes::Generator::Logic>(0.5);
    gate_logic->set_input_node(rotor);

    const std::vector<float> spiral = { 980.0f, 1372.0f, 1568.0f, 2156.0f, 2940.0f };

    auto gate = std::make_shared<Vruta::SoundRoutine>(
        Kriya::Gate(*get_scheduler(), [spiral]() {
            static uint64_t n = 0;
            if (++n % 24000 != 0) return;
            auto s = vega.Sine(spiral[(int)get_uniform_random(0, 4)]);
            s->set_amplitude(get_uniform_random(0.04, 0.09));
            s >> Time(get_uniform_random(0.4, 1.2)) | Audio[0];
        }, gate_logic));
    get_scheduler()->add_task(gate, "bowl_gate");
}
```

A fundamental at 196Hz sustains with a slow 2.8Hz frequency wobble. Four harmonics at slightly inharmonic ratios (mimicking a real bowl's asymmetric modes) each have independent slow amplitude and frequency modulators at incommensurate rates; their beating pattern never locks into a repeating cycle.

A Phasor at 0.11Hz (roughly a 9 second cycle) feeds a threshold Logic node. The Gate coroutine drives that Logic node sample-by-sample. While the Phasor is above 0.5 (roughly half of each cycle) the gate is open. Every 24000 samples (0.5 seconds at 48kHz) a partial from the `spiral` array enters the graph for 0.4 to 1.2 seconds. These upper partials overlap with each other and with the bowl harmonics, creating a texture that waxes and wanes with the Phasor rotation.

Change the Phasor frequency to `0.04f` for a slower rotation (25 second cycles, longer open and closed windows). Change the threshold from `0.5` to `0.8` and the gate is open for only 20% of each cycle: brief dense bursts of spiral partials separated by long silences.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}




