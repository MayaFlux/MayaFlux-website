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

<pre><code class="language-cpp">
void compose() {
    auto wave = vega.Sine(440.0f, 0.2f)[0] | Audio;
}
</code></pre>

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

<pre><code class="language-cpp">
void compose() {
    auto clock = vega.Sine(2.0f, 0.3f)[0] | Audio;

    clock->on_tick_if(
        [](NodeContext ctx) { return ctx.value > 0.0; },
        [](NodeContext ctx) {
            std::cout << "Beat at " << ctx.value << "\n";
        }
    );
}
</code></pre>

<p>
<code>on_tick_if</code> attaches to every sample evaluation (48,000 times per second) but only fires the callback when <code>ctx.value > 0.0</code> becomes true. Sample-accurate rhythm tied to mathematical conditions, not external clocks.
</p>

<p>You can make time operations from any mathematical relationship:</p>

<pre><code class="language-cpp">
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
</code></pre>

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

<pre><code class="language-cpp">
void compose() {
    auto synth = vega.Sine(220.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    auto gate_lfo = vega.Sine(0.5f, 1.0f);
    auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.3);
    gate_lfo >> gate;

    auto processor = create_processor<LogicProcessor>(buffer, gate);
    processor->set_modulation_type(LogicProcessor::ModulationType::HOLD_ON_FALSE);
}
</code></pre>

<p>
<code>LogicProcessor</code> doesn't just multiply audio by 0 or 1 (traditional gate).
<code>HOLD_ON_FALSE</code> freezes the buffer contents when logic is false.
Result: granular stuttering effect. The audio repeats frozen moments.
</p>

<p>Other logical transformations:</p>

<pre><code class="language-cpp">
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
</code></pre>

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

<pre><code class="language-cpp">
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f);
    auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

    auto envelope = vega.Polynomial(std::vector{0.1, 0.9, -0.3, 0.05});
    auto processor = create_processor<PolynomialProcessor>(buffer, envelope);
}
</code></pre>

<p>
<code>PolynomialProcessor</code> applies the polynomial to every sample in the buffer.
Each buffer cycle (512 samples typically) gets reshaped by <code>0.1 + 0.9x - 0.3x² + 0.05x³</code>.
No separate "envelope follower" or "modulation routing." The data is the shape.
</p>

<p>Variations:</p>

<pre><code class="language-cpp">
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
    //     [](const std::deque<double>& history) {
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
    //     [](const std::deque<double>& inputs) {
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
</code></pre>

<p>
You're not "applying an envelope to a sound." You're transforming data according to a mathematical shape.
The same polynomial creates wildly different results depending on the processing mode.
</p>
</div>

---

<div class="card">
<h2>Time as Compositional Material</h2>

<p>
Traditional DAWs treat time as a fixed grid. MayaFlux treats time as something you compose with.
</p>

<pre><code class="language-cpp">
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
</code></pre>

<p>
<code>metro</code> creates a persistent temporal pulse at exact intervals (0.5 seconds = 2Hz).
<code>sequence</code> choreographs events at precise time offsets.
Both run indefinitely until you terminate them. Sample-accurate timing coordinated by the scheduler's clock system.
</p>

<p>More complex temporal structures:</p>

<pre><code class="language-cpp">
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
</code></pre>

<code>pattern</code> generates events based on beat-conditional logic.
The scheduler manages all temporal coordination, you just describe the relationships.

EventChains for temporal choreography:

<pre><code class="language-cpp">
void compose() {
    auto synth = vega.Sine(440.0f, 0.3f)[0] | Audio;

    auto chain = Kriya::EventChain{}
        .then([synth]() { synth->set_frequency(220.0f); }, 0.0)
        .then([synth]() { synth->set_frequency(440.0f); }, 0.5)
        .then([synth]() { synth->set_frequency(880.0f); }, 1.0)
        .then([synth]() { synth->set_frequency(220.0f); }, 1.5);

    chain.start();
}
</code></pre>

Each <code>.then()</code> schedules an action at a specific time offset from the previous event.
The entire sequence runs once, executing actions at precise moments. You're composing with temporal relationships.

</div>

---

<div class="card">
<h2>Buffer Pipelines: Data Flow as Process</h2>

<p>
Buffers aren't just storage. They're temporal gatherers that accumulate moments, transform them, then release.
<code>BufferPipeline</code> lets you compose complex data flow patterns declaratively.
</p>

<p>Simple capture from file:</p>

<pre><code class="language-cpp">
void compose() {
    auto capture_buffer = vega.AudioBuffer()[0] | Audio;
    auto pipeline = create_buffer_pipeline();

    pipeline
        >> BufferOperation::capture_file_from("res/audio.wav", 0)
            .for_cycles(1)
        >> BufferOperation::route_to_buffer(capture_buffer);

    pipeline->execute_buffer_rate();
}
</code></pre>

<p>
<code>capture_file_from</code> reads from audio file.
<code>route_to_buffer</code> sends it to a buffer that plays.
<code>execute_buffer_rate</code> runs the pipeline synchronized to audio hardware boundaries.
</p>

<p>Accumulation and batch processing:</p>

<pre><code class="language-cpp">
void compose() {
    Config::get_global_stream_info().input.enabled = true;
    Config::get_global_stream_info().input.channels = 1;
}

void compose() {
    auto output = vega.AudioBuffer()[0] | Audio;
    auto pipeline = create_buffer_pipeline();
    pipeline->with_strategy(ExecutionStrategy::PHASED);

    pipeline
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
</code></pre>

<p>
<code>.for_cycles(20)</code> captures 20 times, concatenating into a single buffer.  
The <code>transform</code> sees all accumulated samples at once.  
<code>PHASED</code> means capture happens first, then all processing.
</p>

<p>Windowed analysis:</p>

<pre><code class="language-cpp">
void compose() {
    auto pipeline = create_buffer_pipeline();

    pipeline
        >> BufferOperation::capture_input_from(get_buffer_manager(), 0)
            .for_cycles(10)
            .with_window(2048, 0.5f)
            .on_data_ready([](const auto& data, uint32_t cycle) {
                const auto& windowed = std::get<std::vector<double>>(data);
                std::cout << "Cycle " << cycle << ": " << windowed.size() << " samples\n";
            });

    pipeline->execute_buffer_rate();
}
</code></pre>

<p>
<code>with_window</code> creates overlapping capture windows —
use it for spectral analyzers, feature extraction, and sliding-window transforms.
</p>
</div>

---

<div class="card">
<h2>Cross-Domain Coordination</h2>

<p>
This is where MayaFlux diverges completely from analog paradigms. Audio data can directly control visual processes.
</p>

<pre><code class="language-cpp">
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

            points->add_point(Nodes::GpuSync::PointVertex {
                .position = glm::vec3(x, y, 0.0f),
                .color = glm::vec3(1.0f, 0.8f, 0.2f),
                .size = 10.0f });
        });
}
</code></pre>

<p>
Audio peaks trigger visual particle spawns. Sample-accurate coordination between domains. 
You can't do this in analog hardware. There's no physical equivalent to "audio amplitude directly spawns GPU geometry."
</p>

<p>
This is digital-native thinking: all data is just numbers, so audio streams, 
visual transforms, and control logic are the same substrate.
</p>
</div>

---

<div class="card">
<h2>Temporal Accumulation</h2>

<p>
Traditional: Audio flows continuously through effects.
MayaFlux: Audio accumulates in temporal chunks, gets transformed, then releases.
</p>

<pre><code class="language-cpp">
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
</code></pre>

<p>
Buffers aren't just "storage." They're moments of accumulated time. 
Each cycle, 512 samples gather, get transformed by the chain, then release.
Order matters: noise → shaping → feedback creates different results than feedback → shaping → noise.

Buffer size (512, 1024, 2048 samples) becomes a rhythmic parameter.
Larger buffers = slower transformation update rate = more "smeared" or "granular" results.

</p>
</div>

---

<div class="card">
<h2>Modal Synthesis</h2>

<p>
Traditional synthesis uses banks of oscillators. 
MayaFlux treats resonant modes as parallel data transformations.
</p>

<pre><code class="language-cpp">
void compose() {
    auto bell = vega.ModalNetwork(
        12,
        220.0,
        ModalNetwork::Spectrum::INHARMONIC
    )[0] | Audio;

    schedule_metro(2.0, [bell]() {
        bell->excite(0.8);
        bell->set_fundamental(get_uniform_random(220.0f, 880.0f));
    }, "bell_strikes");
}
</code></pre>

<p>
<code>ModalNetwork</code> creates 12 resonant modes with inharmonic frequency ratios (bell-like). 
Each `excite()` call strikes all modes simultaneously. `set_fundamental()` changes the base frequency, all modes adjust proportionally.

The same network with different spectrum types produces completely different timbres:

</p>

<pre><code class="language-cpp">
void compose() {
    auto string = vega.ModalNetwork(
        8,
        440.0,
        ModalNetwork::Spectrum::HARMONIC
    )[0] | Audio;

    auto sifi_synth = vega.ModalNetwork(
        16,
        220.0,
        ModalNetwork::Spectrum::STRETCHED
    )[1] | Audio;
}
</code></pre>

<p>
<code>HARMONIC</code> creates integer frequency ratios (string-like). 
<code>STRETCHE</code> creates slightly sharp harmonics (piano-like stiffness). 
Same computational structure, different mathematical relationships produce different musical results.
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
<li>cross-domain coordination</li>
<li>temporal accumulation patterns</li>
<li>modal networks instead of oscillators</li>
</ul>

<p>
This is digital-first thinking — not “how do I recreate analog workflows” but “what becomes possible when sound is computational data?”
This is digital-first thinking. Not "how do I make MayaFlux behave like my analog synth?" but "what becomes possible when I treat sound as computational data?"
</p>

<p>
Everything you know about music still applies.  
But the foundation shifts from signal flow to data transformation.
</p>
</div>

<div class="card wide">
<h2>Immediate Next Steps</h2>

<p>Immediate next steps:</p>

<ol>
<li>Live coding: modify these transformations while audio is running (Lila JIT system)</li>
<li>Recursive structures: feedback not as "delay line" but "previous computational state informing current decision"</li>
<li>Grammar-driven processing: defining transformation rules as formal grammars</li>
</ol>

<p>Deeper paradigm shifts:</p>

<ol>
<li>Coroutine-based temporal coordination: suspending/resuming computations based on musical conditions</li>
<li>Container-based composition: treating entire audio files as multi-dimensional data to slice/transform/recombine</li>
<li>Network topologies: spatial relationships between synthesis elements creating emergent behaviors</li>
</ol>

<p>Cross-domain expansion:</p>

<ol>
<li>Audio analysis → particle physics parameters</li>
<li>Logic gates → shader compute triggers</li>
<li>Polynomial transforms → texture coordinate warping</li>
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
