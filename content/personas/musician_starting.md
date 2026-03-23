---
title: "Starting as a Musician"
---

_[Back to Personas](../)_

<div class="card wide">
<p>
You've been working with oscillators, envelopes, filters. Units that simulate analog signal flow.
MayaFlux asks you to think differently: not as signal paths, but as data transformation moments.
</p>

<p>
This isn't about making MayaFlux behave like Max/MSP or SuperCollider. It's about discovering what becomes possible
when you treat sound as numbers flowing through precise computational decisions rather than electricity through circuits.
</p>
</div>

---

<div class="card">
<h2>Your First Sound</h2>

```cpp
void compose() {
    auto wave = vega.Sine(440.0f, 0.2f)[0] | Audio;
}
```

<p>
<code>vega.Sine(440.0f, 0.2f)</code> doesn't create an oscillator object.
It creates a mathematical decision applied to each sample point.
<code>[0]</code> routes to channel 0.
<code>| Audio</code> means "evaluate this at sample rate" (48kHz typically). The "sine wave" doesn't exist as a continuous signal.
It's 48,000 discrete computational moments per second, each one a fresh calculation.
</p>
</div>

---

<div class="card">
<h2>Rhythm from Mathematical Truth</h2>

<p>
In traditional systems, you use a clock or metronome to trigger events. In MayaFlux, rhythm emerges from mathematical conditions.
</p>

```cpp
void compose() {
    auto clock = vega.Sine(2.0f, 0.3f)[0] | Audio;

    clock->on_tick_if(
        [](NodeContext ctx) { return ctx.value > 0.0; },
        [](NodeContext ctx) {
            std::cout << "Beat at " << ctx.value << "\n";
        }
    );
}
```

<p>
<code>on_tick_if</code> attaches to every sample evaluation (48,000 times per second) but only fires the callback when <code>ctx.value > 0.0</code> becomes true. Sample-accurate rhythm tied to mathematical conditions, not external clocks.
</p>

<p>You can make time operations from any mathematical relationship:</p>

```cpp
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f)[0] | Audio;

    // Updates from polynomial peaks
    auto envelope = vega.Polynomial(std::vector{1.0, -0.8, 0.2})[1] | Audio;
    envelope->on_tick_if(
        [](NodeContext ctx) { return ctx.value > 0.5; },
        [synth](NodeContext ctx) {
            synth->set_frequency(get_uniform_random(220.0f, 880.0f));
        }
    );
}
```

<p>
Traditional systems force you to separate "rhythm generation" from "sound generation." Any data source can become a temporal trigger.
</p>
</div>

---

<div class="card">
<h2>Gates as Data Transformation</h2>

<p>
You know gates as "open/closed" switches on audio signals. MayaFlux treats gates as logical decisions about data flow.
</p>

```cpp
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    auto gate_lfo = vega.Sine(0.5f, 1.0f);
    auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.3);
    gate_lfo >> gate;

    auto processor = create_processor<LogicProcessor>(buffer, gate);
    processor->set_modulation_type(LogicProcessor::ModulationType::HOLD_ON_FALSE);
}
```

<p>
<code>LogicProcessor</code> doesn't just multiply audio by 0 or 1 (traditional gate).
<code>HOLD_ON_FALSE</code> freezes the buffer contents when logic is false.
Result: granular stuttering effect. The audio repeats frozen moments.
</p>

<p>Other logical transformations:</p>

```cpp
void compose() {
    auto synth = vega.Sine(120.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.0);
    auto processor = create_processor<LogicProcessor>(buffer, gate);

    // Bit crushing
    processor->set_modulation_type(LogicProcessor::ModulationType::REPLACE);
    // Audio becomes pure 0.0 or 1.0 (1-bit audio)

    // Or phase inversion on trigger
    // processor->set_modulation_type(LogicProcessor::ModulationType::INVERT_ON_TRUE);

    // Or granular freeze + crossfade
    // auto tick = vega.Impulse(4.0f)[0] | Audio;
    // gate->set_input_node(tick);
    // processor->set_modulation_type(LogicProcessor::ModulationType::CROSSFADE);
}
```

<p>
Gates aren't on/off switches. They're data transformation strategies.
The same logical condition creates different musical results depending on how you apply it.
</p>
</div>

---

<div class="card">
<h2>Envelopes as Data Shapes</h2>

<p>
Traditional: Envelope generates control signal, modulates something else.
MayaFlux: Envelope is a data transformation function applied directly.
</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    auto envelope = vega.Polynomial(std::vector{0.1, 0.9, -0.3, 0.05});
    auto processor = create_processor<PolynomialProcessor>(buffer, envelope);
}
```

<p>
<code>PolynomialProcessor</code> applies the polynomial to every sample in the buffer.
Each buffer cycle (512 samples typically) gets reshaped by <code>0.1 + 0.9x - 0.3x² + 0.05x³</code>.
No separate "envelope follower" or "modulation routing." The data is the shape.
</p>

<p>Variations:</p>

```cpp
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    // Create polynomial in DIRECT mode (default)
    // auto curve = vega.Polynomial(std::vector { 0.0, 1.2, -0.4 });
    // auto processor = create_processor<PolynomialProcessor>(buffer, curve);

    // Waveshaping distortion (polynomial transforms amplitude directly)
    // curve is already in DIRECT mode by default

    // Or RECURSIVE mode: bitcrushing through truncation feedback
    // auto crush = vega.Polynomial(
    //     [](std::span<double> history) {
    //         // Quantize previous outputs, creating harmonic distortion
    //         double quantized = std::floor(history[0] * 8.0) / 8.0;
    //         return quantized * 0.7 + history[20] * 0.3;
    //     },
    //     PolynomialMode::RECURSIVE,
    //     64
    // );
    // auto crush_proc = create_processor<PolynomialProcessor>(buffer, crush);

    // Or FEEDFORWARD mode: asymmetric distortion based on trajectory
    // auto trajectory = vega.Polynomial(
    //     [](std::span<double> inputs) {
    //         // Different distortion based on whether signal is rising or falling
    //         double slope = inputs[0] - inputs[10];
    //         double curve = (slope > 0) ? inputs[0] * inputs[0] * inputs[0] : std::tanh(inputs[0] * 5.0);
    //         return std::clamp(curve * (1.0 + std::abs(inputs[20] - inputs[40]) * 10.0), -0.5, 0.5);
    //     },
    //     PolynomialMode::FEEDFORWARD,
    //     64);
    // auto trajectory_proc = create_processor<PolynomialProcessor>(buffer, trajectory);

    // Optional: adjust how processor iterates through buffer
    // processor->set_process_mode(PolynomialProcessor::ProcessMode::BATCH);
}
```

<p>
You're not "applying an envelope to a sound." You're transforming data according to a mathematical shape.
The same polynomial creates wildly different results depending on the processing mode.
</p>
</div>

---

<div class="card">
<h2>Node Chains and Composition</h2>

<p>
Nodes can be chained in series using the <code>>></code> operator.
The output of each node becomes the input to the next.
</p>

```cpp
void compose() {
    auto osc = vega.Sine(440.0f, 0.3f);
    auto filter = vega.Filter(FilterType::LOWPASS, 2000.0);
    auto shaper = vega.Polynomial(std::vector{0.0, 1.5, -0.5});

    // Chain: oscillator -> filter -> waveshaper
    auto chain = osc >> filter >> shaper;
    auto buffer = vega.NodeBuffer(0, 512, chain)[0] | Audio;
}
```

<p>
The <code>>></code> operator creates a <code>ChainNode</code>: a sequential processing pipeline of arbitrary length.
This is not "routing" in the Max/MSP sense. It's function composition: <code>shaper(filter(osc(x)))</code> evaluated at sample rate.
</p>

<p>For combining multiple signals, <code>CompositeOpNode</code> takes N inputs and a combine function:</p>

```cpp
void compose() {
    auto osc_a = vega.Sine(220.0f, 0.3f);
    auto osc_b = vega.Sine(221.0f, 0.3f);
    auto osc_c = vega.Sine(330.0f, 0.2f);

    auto mix = vega.CompositeOpNode(
        {osc_a, osc_b, osc_c},
        [](std::span<const double> inputs) {
            return (inputs[0] + inputs[1]) * 0.5 + inputs[2] * 0.3;
        }
    )[0] | Audio;
}
```

<p>
No mixer object. No bus. Just a function that receives all inputs and returns a number.
The combine logic is whatever you write.
</p>
</div>

---

<div class="card">
<h2>Time as Compositional Material</h2>

<p>
Traditional DAWs treat time as a fixed grid. MayaFlux treats time as something you compose with.
</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;

    // Regular pulse
    schedule_metro(0.5, [synth]() {
        synth->set_frequency(get_uniform_random(220.0f, 880.0f));
    }, "pitch_changes");

    // Timed event sequence
    schedule_sequence({
        {0.0, [synth]() { synth->set_frequency(220.0f); }},
        {0.5, [synth]() { synth->set_frequency(440.0f); }},
        {1.0, [synth]() { synth->set_frequency(880.0f); }}
    }, "pitch_sequence");
}
```

<p>
<code>metro</code> creates a persistent temporal pulse at exact intervals (0.5 seconds = 2Hz).
<code>sequence</code> choreographs events at precise time offsets.
Both run indefinitely until you terminate them. Sample-accurate timing coordinated by the scheduler's clock system.
</p>

<p>More complex temporal structures:</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f);

    schedule_pattern(
        [](uint64_t beat) -> std::any {
            return (beat % 8 == 0) ? get_uniform_random(220.0f, 880.0f) : 0.0;
        },
        [synth](std::any value) {
            double freq = safe_any_cast<double>(value);
            if (freq > 0.0) {
                synth->set_frequency(freq);
                synth >> Time(1);
            }
        },
        0.2,
        "every_eighth");
}
```

<code>pattern</code> generates events based on beat-conditional logic.
The scheduler manages all temporal coordination, you just describe the relationships.

<p>EventChain for temporal choreography:</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;

    auto chain = Kriya::EventChain(*get_scheduler())
        .then([synth]() { synth->set_frequency(220.0f); }, 0.0)
        .then([synth]() { synth->set_frequency(440.0f); }, 0.5)
        .then([synth]() { synth->set_frequency(880.0f); }, 1.0)
        .then([synth]() { synth->set_frequency(220.0f); }, 1.5)
        .repeat(4);

    chain.start();
}
```

<p>
Each <code>.then()</code> schedules an action at a specific time offset from the previous event.
<code>.repeat(4)</code> runs the entire sequence four times.
EventChain also supports <code>.times()</code>, <code>.wait()</code>, and <code>.every()</code> for richer temporal patterns.
You're composing with temporal relationships, not programming a clock.
</p>

</div>

---

<div class="card">
<h2>Physical Modelling: Networks, Not Oscillators</h2>

<p>
Traditional synthesis uses banks of oscillators.
MayaFlux treats physical models as networks of parallel data transformations,
with three distinct network types covering different physical domains.
</p>

<h3>Modal: Resonant Modes (Bells, Bars, Membranes)</h3>

```cpp
void compose() {
    auto bell = vega.ModalNetwork(
        12,
        220.0,
        ModalNetwork::Spectrum::INHARMONIC
    )[0] | Audio;

    // Spatial excitation: strike position affects which modes ring
    schedule_metro(2.0, [bell]() {
        double position = get_uniform_random(0.0, 1.0);
        bell->excite_at_position(position, 0.8);
        bell->set_fundamental(get_uniform_random(220.0f, 880.0f));
    }, "bell_strikes");

    // Modal coupling: energy transfers between adjacent modes
    bell->set_coupling_enabled(true);
    bell->set_mode_coupling(0, 1, 0.3);
    bell->set_mode_coupling(1, 2, 0.2);
}
```

<p>
<code>ModalNetwork</code> creates 12 resonant modes with inharmonic frequency ratios (bell-like).
<code>excite_at_position()</code> uses spatial mode shapes: striking at the center excites all modes,
striking at the edge excites odd modes only. This is physical geometry controlling spectral content.
</p>

<p>
Modal coupling transfers energy between modes over time: a struck bell's partials shift and bleed
into each other, creating the evolving shimmer that makes physical sounds alive.
No analog equivalent exists for this. It's computational physics.
</p>

<h3>Waveguide: Wave Propagation (Strings, Tubes)</h3>

```cpp
void compose() {
    auto string = vega.WaveguideNetwork(
        WaveguideNetwork::WaveguideType::STRING, 220.0
    )[0] | Audio;

    auto tube = vega.WaveguideNetwork(
        WaveguideNetwork::WaveguideType::TUBE, 146.8
    )[1] | Audio;

    // Pluck the string: position and strength control the attack spectrum
    schedule_metro(1.5, [string]() {
        string->pluck(
            get_uniform_random(0.1, 0.9),
            get_uniform_random(0.5, 1.0)
        );
    }, "pluck");

    // Strike the tube: Gaussian noise burst excitation
    schedule_metro(2.0, [tube]() {
        tube->strike(0.1, 0.9);
    }, "blow");
}
```

<p>
Where ModalNetwork decomposes resonance into independent modes (frequency domain),
WaveguideNetwork simulates wave propagation through delay lines (time domain).
<code>STRING</code> models Karplus-Strong plucked strings.
<code>TUBE</code> models cylindrical bores (clarinet-like) with bidirectional wave propagation
and boundary reflections that produce the characteristic odd-harmonic spectrum.
</p>

<h3>Resonator: Spectral Shaping (Voice, Formants)</h3>

```cpp
void compose() {
    auto voice = vega.ResonatorNetwork(
        5,
        ResonatorNetwork::FormantPreset::VOWEL_A
    )[0] | Audio;

    // Feed it a pitched exciter (glottal pulse simulation)
    auto pulse = vega.Phasor(120.0f);
    voice->set_exciter(pulse);

    // Switch vowels with keyboard
    auto window = create_window({ "Vowel Synth", 800, 600 });
    window->show();

    on_key_pressed(window, IO::Keys::A, [voice]() {
        voice->apply_preset(ResonatorNetwork::FormantPreset::VOWEL_A);
    });

    on_key_pressed(window, IO::Keys::E, [voice]() {
        voice->apply_preset(ResonatorNetwork::FormantPreset::VOWEL_E);
    });

    on_key_pressed(window, IO::Keys::O, [voice]() {
        voice->apply_preset(ResonatorNetwork::FormantPreset::VOWEL_O);
    });
}
```

<p>
ResonatorNetwork operates purely in the time domain: IIR biquad bandpass filters shape
whatever signal you inject. Feed white noise for breathy formants, a pitched pulse for vowels,
or an arbitrary signal for spectral morphing toward the target formant profile.
</p>

</div>

---

<div class="card">
<h2>Networks Feeding Networks</h2>

<p>
Physical models are not isolated instruments. They're data sources.
One network's output can excite another network, creating compound physical structures
that have no analog equivalent.
</p>

```cpp
void compose() {
    // A waveguide string produces the raw excitation signal
    auto string = vega.WaveguideNetwork(
        WaveguideNetwork::WaveguideType::STRING, 220.0
    )[0] | Audio;
    string->set_output_mode(OutputMode::AUDIO_COMPUTE);
    string->set_output_scale(2);

    // A modal network adds bell-like resonance
    auto modes = vega.ModalNetwork(
        9, 220.0, ModalNetwork::Spectrum::INHARMONIC
    )[0] | Audio;
    modes->set_output_mode(OutputMode::AUDIO_COMPUTE);
    modes->set_output_scale(2);

    // A resonator network shapes the combined result as formants
    auto formants = vega.ResonatorNetwork(
        5, ResonatorNetwork::FormantPreset::VOWEL_A
    )[0] | Audio;
    formants->add_channel_usage(1);

    // Default exciter: a simple phasor
    auto pulse = vega.Phasor(120.0f);
    formants->set_exciter(pulse);

    // Switch excitation source at runtime
    on_key_pressed(window, IO::Keys::N1, [formants, pulse]() {
        formants->clear_network_exciter();
        formants->set_exciter(pulse);
    });

    // Feed the waveguide output into the resonator
    on_key_pressed(window, IO::Keys::N2, [formants, string]() {
        formants->clear_exciter();
        formants->set_network_exciter(string);
    });

    // Feed the modal output into the resonator
    on_key_pressed(window, IO::Keys::N3, [formants, modes]() {
        formants->clear_exciter();
        formants->set_network_exciter(modes);
    });

    // Pluck the string
    on_key_pressed(window, IO::Keys::Enter, [string]() {
        string->pluck(0.9, 0.9);
    });

    // Strike the bell
    on_key_pressed(window, IO::Keys::Tab, [modes]() {
        modes->excite(0.6);
    });
}
```

<p>
<code>set_network_exciter()</code> feeds one network's audio output as excitation into another.
<code>AUDIO_COMPUTE</code> mode means the network processes audio internally but doesn't route to hardware directly:
it exists as a computational data source for other networks.
</p>

<p>
A plucked string's output shaped by vowel formants. A bell's partials feeding a tube resonator.
These are compound physical structures assembled from modular computational components.
The same network architecture that models a guitar string also models a clarinet bore,
and you can pipe one into the other because they're all just numbers.
</p>
</div>

---

<div class="card">
<h2>Channels and Routing</h2>

<p>
<code>[0]</code> in <code>vega.Sine(440.0f, 0.3f)[0] | Audio</code> assigns the node to channel 0.
But channels are not fixed. Nodes, networks, and buffers can all be moved between channels at runtime
with smooth crossfade transitions. This is not "panning." It's data routing with temporal safety.
</p>

<p>Three routing functions cover three entity types:</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;

    // Route a node to channels 0 and 1 with 0.5s crossfade
    route_node(synth, {0, 1}, 0.5);

    // Route a network to multiple channels
    auto bell = vega.ModalNetwork(12, 220.0,
        ModalNetwork::Spectrum::INHARMONIC)[0] | Audio;
    route_network(bell, {0, 1}, 1.0);

    // Route a buffer to a different channel
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;
    route_buffer(buffer, 1, 0.5);
}
```

<p>
<code>route_node()</code> moves a node from its current channel(s) to the target channel(s).
<code>route_network()</code> does the same for entire physical model networks.
<code>route_buffer()</code> moves a buffer between channels.
All three accept either seconds or raw sample counts for the crossfade duration.
The routing system manages fade-in/fade-out at block boundaries to prevent clicks.
</p>

<p>
For static multi-channel registration (no fade needed), networks can declare channel presence directly:
</p>

```cpp
void compose() {
    auto formants = vega.ResonatorNetwork(
        5, ResonatorNetwork::FormantPreset::VOWEL_A
    )[0] | Audio;

    // Output to both channels from the start
    formants->add_channel_usage(1);
}
```

<p>
<code>add_channel_usage(1)</code> registers the network on channel 1 in addition to channel 0, immediately.
</p>

<p>
Routing is compositional. You can schedule routing changes over time:
</p>

```cpp
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;

    auto chain = Kriya::EventChain(*get_scheduler())
        .then([synth]() { route_node(synth, {0}, 0.5); }, 0.0)
        .then([synth]() { route_node(synth, {1}, 0.5); }, 2.0)
        .then([synth]() { route_node(synth, {0, 1}, 0.5); }, 2.0)
        .repeat(0);

    chain.start();
}
```

<p>
A node that migrates between left, right, and stereo on a timed cycle.
Routing becomes another dimension of compositional control, not a static mixer setting.
</p>
</div>

---

<div class="card">
<h2>Cross-Domain Coordination</h2>

<p>
This is where MayaFlux diverges completely from analog paradigms. Audio data can directly control visual processes.
</p>

```cpp
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f);

    auto peaks = vega.Logic(LogicOperator::THRESHOLD, 0.2)[0] | Audio;
    peaks->enable_mock_process(true);
    peaks->set_input_node(synth);

    auto window = create_window({ "Audio-Visual", 1920, 1080 });
    auto points = vega.PointCollectionNode(500) | Graphics;
    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window });
    window->show();

    peaks->on_change_to(true,
        [points](NodeContext ctx) {
            float x = get_uniform_random(-1.0f, 1.0f);
            float y = get_uniform_random(-1.0f, 1.0f);

            points->add_point(Nodes::PointVertex {
                .position = glm::vec3(x, y, 0.0f),
                .color = glm::vec3(1.0f, 0.8f, 0.2f),
                .size = 10.0f });
        });
}
```

<p>
Audio peaks trigger visual particle spawns. Sample-accurate coordination between domains.
You can't do this in analog hardware. There's no physical equivalent to "audio amplitude directly spawns GPU geometry."
</p>

<p>
Per-window keyboard and mouse input adds interactive control:
</p>

```cpp
void compose() {
    auto window = create_window({ "Interactive", 1920, 1080 });
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;
    window->show();

    on_key_pressed(window, IO::Keys::Space, [synth]() {
        synth->set_frequency(get_uniform_random(220.0f, 880.0f));
    });

    on_mouse_pressed(window, IO::MouseButtons::Left,
        [synth, window](double x, double y) {
            auto coords = normalize_coords(x, y, window);
            synth->set_frequency(220.0f + coords.y * 660.0f);
        });
}
```

<p>
This is digital-native thinking: all data is just numbers, so audio streams,
visual transforms, keyboard events, and control logic are the same substrate.
</p>
</div>

---

<div class="card">
<h2>Temporal Accumulation</h2>

<p>
Traditional: Audio flows continuously through effects.
MayaFlux: Audio accumulates in temporal chunks, gets transformed, then releases.
</p>

```cpp
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    // First transformation: add noise
    auto noise = vega.Random();
    auto noise_proc = create_processor<NodeSourceProcessor>(buffer, noise);

    // Second transformation: polynomial shaping
    auto curve = vega.Polynomial(std::vector { 0.0, 1.2, -0.4 });
    auto shape_proc = create_processor<PolynomialProcessor>(buffer, curve);

    // Third transformation: feedback delay
    auto feedback_proc = create_processor<FeedbackProcessor>(buffer);
    feedback_proc->set_feedback(0.3f);

    // Chain executes in order each buffer cycle
    auto chain = create_processing_chain();
    chain->add_processor(noise_proc, buffer);
    chain->add_processor(shape_proc, buffer);
    chain->add_processor(feedback_proc, buffer);

    buffer->set_processing_chain(chain);
}
```

<p>
Buffers aren't just "storage." They're moments of accumulated time.
Each cycle, 512 samples gather, get transformed by the chain, then release.
Order matters: noise -> shaping -> feedback creates different results than feedback -> shaping -> noise.

Buffer size (512, 1024, 2048 samples) becomes a rhythmic parameter.
Larger buffers = slower transformation update rate = more "smeared" or "granular" results.

</p>
</div>

---

<div class="card">
<h2>Buffer Pipelines: Data Flow as Process</h2>

<p>
Buffers aren't just storage. They're temporal gatherers that accumulate moments, transform them, then release.
<code>BufferPipeline</code> lets you compose complex data flow patterns declaratively.
</p>

<p>Simple capture from file:</p>

```cpp
void compose() {
    auto capture_buffer = vega.AudioBuffer()[0] | Audio;
    auto pipeline = create_buffer_pipeline();

    *pipeline
        >> BufferOperation::capture_file_from("res/audio.wav", 0)
            .for_cycles(1)
        >> BufferOperation::route_to_buffer(capture_buffer);

    pipeline->execute_buffer_rate();
}
```

<p>
<code>capture_file_from</code> reads from audio file.
<code>route_to_buffer</code> sends it to a buffer that plays.
<code>execute_buffer_rate</code> runs the pipeline synchronized to audio hardware boundaries.
</p>

<p>Accumulation and batch processing:</p>

```cpp
void compose() {
    Config::get_global_stream_info().input.enabled = true;
    Config::get_global_stream_info().input.channels = 1;
}

void compose() {
    auto output = vega.AudioBuffer()[0] | Audio;
    auto pipeline = create_buffer_pipeline();
    pipeline->with_strategy(ExecutionStrategy::PHASED);

    *pipeline
        >> BufferOperation::capture_input_from(get_buffer_manager(), 0)
            .for_cycles(20)
        >> BufferOperation::transform([](const auto& data, uint32_t cycle) {
            auto samples = std::get<std::vector<double>>(data);
            // Process all 20 buffer cycles as one batch
            for (auto& sample : samples) {
                sample *= 0.5;
            }
            return samples;
        })
        >> BufferOperation::route_to_buffer(output);

    pipeline->execute_buffer_rate();
}
```

<p>
<code>.for_cycles(20)</code> captures 20 times, concatenating into a single buffer.
The <code>transform</code> sees all accumulated samples at once.
<code>PHASED</code> means capture happens first, then all processing.
<code>STREAMING</code> strategy is also available: each capture flows through immediately for lower latency.
</p>

<p>Windowed analysis:</p>

```cpp
void compose() {
    auto pipeline = create_buffer_pipeline();

    *pipeline
        >> BufferOperation::capture_input_from(get_buffer_manager(), 0)
            .for_cycles(10)
            .with_window(2048, 0.5f)
            .on_data_ready([](const auto& data, uint32_t cycle) {
                const auto& windowed = std::get<std::vector<double>>(data);
                std::cout << "Cycle " << cycle << ": " << windowed.size() << " samples\n";
            });

    pipeline->execute_buffer_rate();
}
```

<p>
<code>with_window</code> creates overlapping capture windows:
use it for spectral analyzers, feature extraction, and sliding-window transforms.
</p>
</div>

---

<div class="card wide">
<h2>Where This Goes</h2>

<p>You didn't learn "how to use MayaFlux." You learned to think about sound as:</p>

<ul>
<li>mathematical decisions evaluated at sample points</li>
<li>truth conditions generating rhythm</li>
<li>data transformations instead of control signals</li>
<li>compositional temporal relationships</li>
<li>physical models as composable networks (modal, waveguide, resonator)</li>
<li>networks feeding networks: compound physical structures</li>
<li>cross-domain coordination with interactive input</li>
<li>temporal accumulation patterns</li>
<li>channel routing as compositional control: nodes, networks, and buffers all routable with crossfades</li>
</ul>

<p>
This is digital-first thinking. Not "how do I make MayaFlux behave like my analog synth?" but "what becomes possible when I treat sound as computational data?"
</p>

<p>
Everything you know about music still applies.
But the foundation shifts from signal flow to data transformation.
</p>
</div>

<div class="card wide">
<h2>Continue Learning</h2>

<p>
This page introduced concepts through compact examples.
The <strong>Sculpting Data</strong> tutorial series builds each idea systematically with runnable code:
</p>

<ol>
<li><strong><a href="../../tutorials/sculpting-data/foundations/">Foundations of Form</a></strong>: File IO, containers, buffers, how data is organized and flows through the system</li>
<li><strong><a href="../../tutorials/sculpting-data/processing_expression/">Processing Expression</a></strong>: Processors, chains, supply, copy, logic, polynomial transforms, and the full buffer operation vocabulary</li>
<li><strong><a href="../../tutorials/sculpting-data/visual_materiality_i/">Visual Materiality</a></strong>: Graphics basics, windows, point clouds, geometry buffers, and rendering setup</li>
</ol>

<p>Deeper paradigm shifts:</p>

<ol>
<li>Live coding: modify these transformations while audio is running (Lila JIT system)</li>
<li>Recursive structures: feedback not as "delay line" but "previous computational state informing current decision"</li>
<li>Grammar-driven processing: defining transformation rules as formal grammars (Yantra compute engine)</li>
<li>Container-based composition: treating entire audio files as multi-dimensional data to slice/transform/recombine</li>
<li>GPU compute pipelines: audio analysis driving shader parameters, network history buffers as descriptor bindings</li>
</ol>

<p>Cross-domain expansion:</p>

<ol>
<li>Audio network history -> GPU storage buffer -> fragment shader visualization</li>
<li>Logic gates -> shader compute triggers</li>
<li>Polynomial transforms -> texture coordinate warping</li>
<li>MIDI/HID input nodes as first-class signal sources (<code>vega.read_midi()</code>, <code>vega.read_hid()</code>)</li>
<li>Camera input through IOManager into the same buffer architecture as audio</li>
</ol>

<p>
MayaFlux isn't "better" than Max or SuperCollider or Ableton. It's asking a different question:
what if we stopped pretending digital audio is electricity flowing through wires, and embraced it
as discrete computational events we can control with arbitrary precision?
</p>

<p>
Everything you know about music production still applies: envelope shapes, filter responses,
rhythmic structure. But the conceptual foundation shifts from "signal flow" to "data transformation."
</p>
</div>
<p>
That shift unlocks possibilities that don't exist in analog paradigms. Not "better." Different.
</p>
