---
title: "From Pure Data to MayaFlux: Unlearning Analog Constraints"
---

<div class="card wide">
<p>
Pure Data taught you to think in patches: objects connected by wires, control rate separated from audio rate, bang messages triggering discrete events. These patterns made sense for simulating modular synthesizers, and they were brilliant solutions calibrated to the context in which PD emerged.
</p>

<p>
MayaFlux asks: what if those boundaries were simply one possible interpretation of computation rather than permanent constraints?
</p>

<p>
<strong>This isn't about "Pure Data but better." It is about seeing which patterns were PD-specific design commitments and which ones become fluid when you work directly with computational substrate.</strong>
</p>
</div>

---

## What Pure Data Made Invisible

<div class="card">

Every tool embeds assumptions that become "just how things work." In PD these assumptions were deliberate and effective simplifications for its architecture and era.

- **`[metro]` ticks at control rate**  
  You learned: Time is discrete bangs at ~64Hz intervals  
  Actually: PD emphasized discrete timing for clarity. MayaFlux treats time as a resolution you choose.

- **`~` objects run at 48kHz, regular objects at control rate**  
  You learned: Audio and control belong to separate domains  
  Actually: PD separated domains for pedagogical and performance clarity. In MayaFlux, they are simply different evaluation precisions applied to the same kind of data.

- **`[osc~ 440]` is a black box**  
  You learned: Oscillators generate sound and their internals are abstracted  
  Actually: PD hid internals intentionally. MayaFlux allows internal access when creative practice benefits from it.

- **`[fexpr~]` is "advanced" recursive processing**  
  You learned: Recursive expressions require special syntax  
  Actually: PD used a compact syntax to make recursion predictable. MayaFlux exposes recursion as one mode among several equally natural choices.

- **`[table]` stores data you write/read**  
  You learned: Arrays are passive storage  
  Actually: PD made this explicit and visible. MayaFlux treats buffers as temporal gatherers that can be shaped before being released.

- **Patching is visual connection**  
  You learned: Creativity happens through dragging wires  
  Actually: PD emphasized visibility. MayaFlux expresses flow as code because modern practice often benefits from explicit, versioned logic.

These were not limitations in skill or imagination. They were elegant design decisions shaped by PD’s moment in technological history.

**Today you simply have more choices available.**

</div>

---

## Time Isn't Control Rate Ticks

<div class="card">

In PD: `[metro 500]` bangs every 500ms. That is "how metronomes work" inside the PD model.

In MayaFlux: Time is just a number you choose.

```cpp
// Periodic evaluation at 0.5 second intervals
schedule_metro(0.5, []() {
    trigger_something();
}, "my_timer");

// OR Want sample-accurate? Just declare it
auto tick = vega.Impulse(2.0)[0] | Audio;  // 2Hz = every 0.5 seconds

tick->on_impulse([](auto& ctx) {
    trigger_something();
});

// Advanced: Custom coroutine-based timing
schedule_task("Sample_task", []() -> Vruta::SoundRoutine {
    while (true) {
        co_await SampleDelay{48000};  // 1 second at 48kHz
        trigger_something();
    }
}, true);
```

`[metro]` was not a conceptual limit. It was a clear, efficient implementation aligned with PD’s architecture.

Decoupling the concept (periodic evaluation) from the implementation detail (control rate) lets you choose whatever precision your idea needs.

</div>

---

## Audio Rate vs Control Rate: An Artificial Boundary

<div class="card">

PD expresses `[osc~]` at audio rate and `[metro]` at control rate to keep the mental model stable and the engine predictable.

MayaFlux expresses everything as data whose evaluation frequency is adjustable.

```cpp
// Rhythm generator evaluated at 48kHz
auto rhythm = vega.Impulse(4.0)[0] | Audio;

// Its output can control ANYTHING
rhythm->on_tick([](NodeContext ctx) {
    // Spawn visual particles at exact sample moments
    spawn_particle_at(ctx.value);

    // Change polynomial coefficients
    shaper->set_coefficients({ctx.value, 0.5, -0.2});

    // Reschedule another metro
    if (ctx.value > 0.9) {
        modify_metro_timing("bass_trigger", 0.125);
    }
});
```

PD’s separation of control and audio was a sensible design choice for its goals. In MayaFlux these categories collapse because modern processors allow a unified evaluation model.

A node's domain token (`| Audio`, `| Graphics`, `| Time(10)`) simply states how often it is evaluated. The data itself is agnostic.

</div>

---

## Every Sample Is Accessible

<div class="card">

Pure Data’s `[osc~ 440]` sensibly abstracts away details so you can reason about behavior rather than implementation.

MayaFlux lets you reach into those details when that becomes part of the creative gesture.

```cpp
auto wave = vega.Sine(440.0, 0.3)[0] | Audio;

// Hook into processing—48,000 times per second
wave->on_tick([](NodeContext ctx) {
    if (ctx.value > 0.95) {
        // React to wave peaks
        trigger_visual_flash();
    }
});

// Or conditional hooks
wave->on_tick_if(
    [](NodeContext ctx) { return ctx.value > 0.8; },
    [](NodeContext ctx) {
        modulate_something(ctx.value);
    }
);
```

The pattern you internalized was: "Oscillators generate sound and their internals are not meant to be inspected." This was completely valid for PD’s goals.

The computational truth: Every processing step is just a transformation. Transformations can be observed.

<h3>Complete State Access</h3>

Callbacks receive the node's complete internal state:

```cpp
auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.3);

gate->on_change([](auto& ctx) {
    auto logic_ctx = ctx.template as<LogicContext>();

    // Access everything:
    auto history = logic_ctx->get_history(); // Past states
    auto threshold = logic_ctx->get_threshold(); // Current threshold
    bool edge = logic_ctx->is_edge_detected(); // Edge detection
    auto mode = logic_ctx->get_mode(); // Processing mode

        // Make creative decisions from state
        if (history.size() > 10 && history > 7) {
            trigger_dense_event(); // Pattern detected
        }
});
```

```cpp
auto shaper = vega.Polynomial(std::vector { 0.1, 0.8, -0.3 });

shaper->on_tick([](auto& ctx) {
    auto poly_ctx = ctx.template as<PolynomialContext>();

    auto coeffs = poly_ctx->get_coefficients();
    auto input_history = poly_ctx->get_input_buffer();
    auto output_history = poly_ctx->get_output_buffer();

    // Envelope follower from output variance
    double variance = calculate_variance(output_history);
    if (variance > 0.5) {
        increase_resonance(); // Signal getting chaotic
    }
});
```

PD abstracted internal state to reduce cognitive load. MayaFlux exposes it because current creative practices often treat computation itself as material.

</div>

---

## fexpr~ → Polynomial Nodes: Making Recursion Natural

<div class="card">

PD's `[fexpr~]` was a breakthrough for its time, bringing recursive audio-rate expressions into an efficient, predictable syntax. The constraints it imposed were intentional.

<h3>fexpr PD Patch Example</h3>

```cpp
[fexpr~ a = fmod(a + (accumValues[i]*accumValues[i]*$f2+1)/48000, 1) ;
        if ((a-$y1[$i3])<0, i = (i+1)%40, 0)]
```

The pattern PD encouraged was: "Recursion lives inside a special syntax."

<h3>MayaFlux Polynomial Nodes</h3>

```cpp
// Direct mode: Simple polynomial (like fexpr without history)
auto waveshaper = vega.Polynomial({0.0, 1.2, -0.4});  // 1.2x - 0.4x²
```

```cpp
// Recursive mode: Access previous OUTPUTS (like $y1, $y2)
auto feedback = vega.Polynomial(
    [](const std::deque<double>& history) {
        // history[0] = last output, history[1] = output before that...
        return history[0] * 0.7;  // Simple feedback delay
    },
    PolynomialMode::RECURSIVE,
    100  // 100-sample history
);
```

```cpp
// Feedforward mode: Access previous INPUTS (like $x1, $x2)
auto moving_avg = vega.Polynomial(
    [](const std::deque<double>& history) {
        // history[0] = current input, history[1] = previous...
        double sum = 0.0;
        for (auto val : history) sum += val;
        return sum / history.size();
    },
    PolynomialMode::FEEDFORWARD,
    64  // 64-sample window
);
```

```cpp
auto phase_accum = vega.Polynomial(
    [&](const std::deque<double>& history) {
        static int index = 0;
        static std::vector<double> accum_values(40);

        double increment = (accum_values[index] * accum_values[index] * Config::get_sample_rate() + 1.0)
            / Config::get_sample_rate();
        double new_phase = std::fmod(history[0] + increment, 1.0);

        // Zero-crossing detection
        if (new_phase - history[1] < 0.0) {
            index = (index + 1) % 40;
        }

        return new_phase;
    },
    PolynomialMode::RECURSIVE,
    2 // Need current and previous
);
```

PD offered a predictable window of history for efficiency. MayaFlux expands this because modern architectures make larger or dynamic history sizes practical.

<h3>Polynomial Processors on Buffers</h3>

```cpp
// Karplus-Strong string via recursive polynomial on buffer
auto synth = vega.Sine(220.0, 0.3);
auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

auto string = vega.Polynomial(
    [](const std::deque<double>& history) {
        return 0.996 * (history[0] + history[1]) / 2.0;  // Lowpass + feedback
    },
    PolynomialMode::RECURSIVE,
    100  // Determines pitch
);

auto processor = create_processor<PolynomialProcessor>(buffer, string);

// Choose iteration strategy
processor->set_process_mode(PolynomialProcessor::ProcessMode::SAMPLE_BY_SAMPLE);
// processor->set_process_mode(PolynomialProcessor::ProcessMode::BATCH);
// processor->set_process_mode(PolynomialProcessor::ProcessMode::WINDOWED);
```

Pure Data’s architecture did not expose iteration strategies to preserve performance guarantees. MayaFlux makes them adjustable because it is built for a different computational moment.

</div>

---

## [table] → Buffers: From Storage to Temporal Gathering

<div class="card">

PD: `[table]` explicitly exposes memory, which is one of its strengths. You can see exactly where data is written and read.

MayaFlux: Buffers gather temporal slices that can be transformed before release.

```cpp
auto synth = vega.Sine(220.0, 0.3);
auto buffer = vega.NodeBuffer(0, 512, synth)[0] | Audio;

// Processors transform accumulated data
auto feedback = create_processor<FeedbackProcessor>(buffer);
feedback->set_feedback(0.3);

auto shaper = create_processor<PolynomialProcessor>(buffer,
    vega.Polynomial({0.0, 1.2, -0.4})
);

// Chain them (order matters)
auto chain = create_processing_chain();
chain->add_processor(shaper, buffer);   // Shape first
chain->add_processor(feedback, buffer); // Then feedback

buffer->set_processing_chain(chain);
```

Every 512 samples (adjustable):

1. Buffer gathers input
2. Processing chain runs
3. Buffer releases output
4. Cycle repeats

Buffer size becomes compositional parameter. Larger = slower update rate = more granular/smeared results.

<h3>Buffer Pipelines</h3>

```cpp
auto capture = vega.AudioBuffer()[0] | Audio;
auto analysis = vega.AudioBuffer()[1] | Audio;

auto pipeline = create_buffer_pipeline();
pipeline->with_strategy(ExecutionStrategy::PHASED);

pipeline
    // Capture 40 cycles (like 40-element array)
    >> BufferOperation::capture_input_from(get_buffer_manager(), 0)
        .for_cycles(40)

    // Transform accumulated data
    >> BufferOperation::transform([](const auto& data, uint32_t cycle) {
        auto samples = std::get<std::vector<double>>(data);
        for (auto& s : samples) {
            s = s * s;
        }
        return samples;
    })

    >> BufferOperation::route_to_buffer(analysis);

pipeline->execute_buffer_rate();
```

Pure Data's explicit `[tabwrite~]` and `[tabread~]` objects made data flow visible, letting you see precisely when arrays were being accessed and written. This clarity was one of PD’s strengths, especially for teaching and debugging. MayaFlux's declarative pipelines explore a complementary trade-off: you express the what of the operation (accumulate → transform → route) and the system handles the when, made possible by an architecture that manages buffer synchronization at the engine level.

</div>

---

## Modal Synthesis: Beyond Oscillator Banks

<div class="card">

```cpp
// Bell-like inharmonic spectrum
auto bell = vega.ModalNetwork(
    12,                                      // 12 modes
    220.0,                                   // Fundamental
    ModalNetwork::Spectrum::INHARMONIC       // Non-integer ratios
)[0] | Audio;

schedule_metro(2.0, [bell]() {
    bell->excite(0.8);
    bell->set_fundamental(get_uniform_random(220.0, 880.0));
}, "bell_strikes");
```

```cpp
// String-like harmonic spectrum
auto string = vega.ModalNetwork(
    8, 440.0, ModalNetwork::Spectrum::HARMONIC
)[0] | Audio;

// Piano-like stretched harmonics
auto piano = vega.ModalNetwork(
    16, 220.0, ModalNetwork::Spectrum::STRETCHED
)[1] | Audio;
```

Pure Data's one-oscillator-per-partial approach directly reflected modular synthesis practice, giving each mode a visible, manipulable identity. This explicitness made the technique intuitive and pedagogically strong, because you understood modal synthesis by assembling it yourself. MayaFlux's `ModalNetwork` offers a complementary approach: the mathematical structure of the modes becomes a first-class parameter, so you shape relationships between modes rather than wiring each one individually. Both approaches illuminate different aspects of the same idea.

</div>

---

## Cross-Domain Coordination

<div class="card">

```cpp
auto onset = vega.Logic(LogicOperator::THRESHOLD, 0.5);
onset->set_input_node(vega.Sine(2.0));

auto window = create_window({"Audio-Visual", 1920, 1080});
auto particles = vega.PointCollectionNode(500) | Graphics;

auto geom = vega.GeometryBuffer(points) | Graphics;

geom->setup_rendering({ .target_window = window });
window->show();

onset->on_change_to(true, [particles](NodeContext ctx) {
    float x = get_uniform_random(-1.0, 1.0);
    float y = get_uniform_random(-1.0, 1.0);
    particles->add_point({
        .position = glm::vec3(x, y, 0.0),
        .color = glm::vec3(1.0, 0.8, 0.2),
        .size = 10.0
    });
});
```

Pure Data handled graphics through GEM, a separate environment that made perfect sense when OpenGL was the available rendering model. This separation kept the system clear and modular. MayaFlux benefits from modern unified memory and scheduling architectures where audio and visual processes inhabit the same computational space, allowing sample-accurate cross-domain interactions that were not technically accessible in PD’s era. Different technical contexts, different expressive possibilities.

</div>

---

## Recursive Time

<div class="card">

```cpp
// Metro that recalculates its own interval
auto adaptive = [](Vruta::TaskScheduler& s) -> Vruta::SoundRoutine {
    while (true) {
        float complexity = analyze_spectral_density();
        float interval = map(complexity, 0.0, 1.0, 0.1, 2.0);

        co_await SampleDelay{s.seconds_to_samples(interval)};
        trigger_event();
    }
};
```

Pure Data's `[metro]` provides stable, deterministic timing suited for rhythmic structures, performances, and pedagogical clarity. MayaFlux’s coroutine-based timing sits alongside that idea and explores a different space. Temporal processes can observe their own computational context and adapt their timing dynamically. This is enabled by language-level coroutine support that simply did not exist when PD defined its timing model.

</div>

---

## Grammar-Defined Processing

<div class="card">

```cpp
auto grammar = std::make_shared<ComputationGrammar>();

grammar->create_rule("compress_loud")
    .matches_type<std::vector<double>>()
    .when([](const auto& data) { return peak_level(data) > 0.8; })
    .executes([](auto& data) { apply_compression(data, 4.0); });

grammar->create_rule("expand_quiet")
    .matches_type<std::vector<double>>()
    .when([](const auto& data) { return peak_level(data) < 0.2; })
    .executes([](auto& data) { apply_expansion(data, 1.5); });

auto pipeline = std::make_shared<ComputePipeline<InputType, OutputType>>(grammar);

// Apply to buffer processor
MayaFlux::attac_quick_process([grammar, pipeline](auto& buf) {
    buf->get_data() = pipeline->process(input_data);
}, your_audio_buffer);
```

You are not writing effects in the traditional sense. You are defining behavioral rules, and the buffer applies whichever ones match its state. Pure Data’s `[select]`, `[route]`, and `[spigot]` objects already embody the insight that message routing is a form of computation. MayaFlux’s grammar system extends that idea by letting the data itself determine which transformations apply. Both approaches share the same conceptual root, expressed through different architectural lenses.

</div>

---

## Live Coding Native Code

<div class="card">

```cpp
// JIT compile C++ at runtime
Lila::jit_compile_and_execute(R"(
    auto new_wave = vega.Polynomial(std::vector {
        get_uniform_random(-1.0, 1.0),
        get_uniform_random(-1.0, 1.0),
        get_uniform_random(-1.0, 1.0)
    });
    new_wave >> DAC::instance();
)");
```

Pure Data’s interpreted expression system prioritized immediacy and fluid patching, which is why it became so powerful in performance contexts. MayaFlux builds on a different technological foundation by using LLVM’s JIT capabilities, allowing native C++ to be compiled on the fly while preserving that sense of immediacy. Both paths honor the same impulse: make live computational manipulation part of the creative act.

</div>

---

## The Mental Shift

<div class="card wide">

**Stop thinking:**

"I need a metro object" → Think: "I need periodic evaluation at this precision"  
"I need to convert audio to control" → Think: "I need to process data at different rates"  
"I need fexpr~ for recursion" → Think: "Recursive mode is just one of three equally valid processing modes"  
"I need arrays for storage" → Think: "Buffers gather temporal data, transform it, release it"

**Start thinking:**

Time is a processing precision I choose  
"Audio" and "control" are the same data at different evaluation frequencies  
Every transformation step can trigger events  
Recursive processing is natural  
Buffers aren't storage, they are temporal accumulation spaces

Pure Data’s architectural decisions were shaped by the realities and insights of its moment. They provided clarity, predictability, and exceptional pedagogical strength. MayaFlux explores what becomes possible when you design for a contemporary computational landscape, where evaluation frequencies, data access, and temporal structure can be fluid and programmable.

Your PD knowledge translates directly. Oscillators, filters, envelopes, and patching logic remain meaningful; the vocabulary remains familiar. The substrate simply shifts from working within a modular-synthesis metaphor to shaping computation itself as the medium.

</div>

---

## Where we are headed

<div class="card wide">

Try the simplest pattern:

```cpp
void compose() {
    auto wave = vega.Sine(440.0, 0.3)[0] | Audio;
    wave->on_tick([](NodeContext ctx) {
        if (ctx.value > 0.9) std::cout << "Peak!\n";
    });
}
```

Run it. You will see "Peak!" printed at the exact moments the sine crosses 0.9.

This is the beginning of the paradigm shift: computational precision becomes something you can compose with directly.

Then explore:

- Replace your `[fexpr~]` patches with `PolynomialMode::RECURSIVE`
- Chain `PolynomialProcessor` → `LogicProcessor` → `FeedbackProcessor`
- Make audio peaks spawn visual particles

You are not recreating PD patches in MayaFlux. You are recognizing which patterns emerged from PD’s architecture and which possibilities open when the computational substrate itself becomes the material you shape.

</div>

---

Everything you know about sound design applies. The creative concepts carry forward.

The foundation shifts from analog simulation to digital-first data transformation, opening expressive patterns that PD, by design, did not aim to expose. This is not about better or worse. It is about working in a different conceptual space.
