---
build:
  list: never
---

{{< tutorial-card step="1 of 7" title="Polynomial Waveshaping" open="true" >}}

#### The Simplest Path

Run this code. Your file plays with harmonic distortion.
```cpp
void compose() {
    auto sound = vega.read_audio("path/to/file.wav") | Audio;
    auto buffers = get_io_manager::get_audio_buffers();

    // Polynomial: x² generates harmonics
    auto poly = vega.Polynomial([](double x) { return x * x; });
    auto processor = MayaFlux::create_processor<PolynomialProcessor>(buffers[0], poly);
}
```
Replace `"path/to/file.wav"` with an actual path.

The audio sounds richer and warmer, with subtle saturation. That's harmonic content added by the squaring function.

{{< tutorial-detail title="Expansion 1: Why Polynomials Shape Sound" >}}

When you write `x * x`, you're not "squaring numbers." You're defining a **transfer curve**:

- Input -1.0 → Output 1.0
- Input 0.5 → Output 0.25 (quieter)
- Input 1.0 → Output 1.0 (same)

This asymmetry adds harmonics. The waveform's shape **bends**: its geometry changes.

Analog distortion (tubes, tape) works this way: input voltage doesn't map linearly to output. The circuit's response curve adds character.

Polynomials let you design that curve digitally. `x * x` is gentle. `x * x * x` adds different harmonics (odd instead of even). `std::tanh(x)` mimics tube saturation.

You're sculpting frequency response through function shape.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: What `vega.Polynomial()` Creates" >}}

`vega.Polynomial([](double x) { return x * x; })` creates a **Polynomial node**, a mathematical expression that processes one sample at a time.

By itself, the node doesn't touch your audio. You wrap it in a **PolynomialProcessor**:
```cpp
auto processor = MayaFlux::create_processor<PolynomialProcessor>(buffers[0], poly);
```
**Why this separation?**

- **Node**: The math itself (reusable, chainable, inspectable)
- **Processor**: The attachment mechanism that knows *how* to apply the node to a buffer

Same node, different processors → different results. You'll see this pattern everywhere in MayaFlux.

The node is the *idea*. The processor is the *application*.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: PolynomialMode::DIRECT" >}}

Polynomials have three modes:

- **DIRECT**: `f(x)` where x is the current sample
- **RECURSIVE**: `f(y[n-1], y[n-2], ...)` where output depends on previous outputs
- **FEEDFORWARD**: `f(x[n], x[n-1], ...)` where output depends on input history

In DIRECT mode (what you're using now), each sample is transformed independently. This is **memoryless** waveshaping.

Later sections explore time-aware modes. RECURSIVE creates filters and feedback. FEEDFORWARD creates delay-based effects.

For now: DIRECT mode = instant transformation. No memory. No delay.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: What `create_processor()` Does" >}}

When you call:
```cpp
auto processor = MayaFlux::create_processor<PolynomialProcessor>(buffers[0], poly);
```
MayaFlux does this:

1.  Creates a `PolynomialProcessor` wrapping your polynomial node
2.  Gets `buffers[0]`'s processing chain
3.  Adds the processor to that chain
4.  Returns the processor handle

The buffer now runs your polynomial on every cycle:

- 512 samples arrive from the Container
- Your polynomial processes each sample: `y = x * x`
- Transformed samples continue to speakers

The processor is now part of the buffer's flow. It runs automatically every cycle until removed.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Cubic distortion (more aggressive, odd harmonics)
auto poly = vega.Polynomial([](double x) { return x * x * x; });

// Chebyshev waveshaping (precise harmonic control)
auto poly = vega.Polynomial([](double x) { return 2*x*x - 1; });

// Soft clipping (analog-style limiting)
auto poly = vega.Polynomial([](double x) {
    return x / (1.0 + std::abs(x));
});

// Extreme fold-back distortion
auto poly = vega.Polynomial([](double x) {
    return std::sin(x * 5.0);
});
```
Listen to each. Same structure, different curves. Each curve generates different harmonic content.

Think of it not as "processing audio" but as **sculpting the transfer function**.

---

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

{{< tutorial-card step="2 of 7" title="Recursive Polynomials (Filters and Feedback)" open="true" >}}

#### The Next Step

You have memoryless waveshaping. Now add memory.
```cpp
void compose() {
    auto sound = vega.read_audio("path/to/file.wav") | Audio;
    auto buffers = get_io_manager::get_audio_buffers(sound);

    // Recursive: output depends on previous outputs
    auto recursive = vega.Polynomial(
        [](std::span<double> history) {
            // history[0] = previous output, history[1] = two samples ago
            return 0.5 * history[0] + 0.3 * history[1];
        },
        PolynomialMode::RECURSIVE,
        2  // remember 2 previous outputs
    );

    auto processor = MayaFlux::create_processor<PolynomialProcessor>(buffers[0], recursive);
}
```
Run this. You hear echo and resonance as the signal feeds back into itself.

{{< tutorial-detail title="Expansion 1: Why This Is a Filter" >}}

Classic IIR filter equation:
```text
y[n] = b0*x[n] + a1*y[n-1] + a2*y[n-2]
```
Your recursive polynomial **is** that filter, just written as a lambda:
```cpp
[](std::span<double> history) {
    return 0.5 * history[0] + 0.3 * history[1];
}
```
Difference: You can write **nonlinear** feedback:
```cpp
[](std::span<double> history) {
    return history[0] * std::sin(history[1]);  // nonlinear!
}
```
Traditional DSP libraries can't do this. Fixed coefficients only.

Polynomials let you design arbitrary recursive functions, not just linear filters.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: The History Buffer" >}}

When you write:
```cpp
PolynomialMode::RECURSIVE, 2
```
The polynomial maintains a buffer of **previous outputs**:
```text
history[0] = y[n-1]  (last output)
history[1] = y[n-2]  (two samples ago)
```
Each cycle:

1.  Your lambda reads from `history`
2.  Computes new output
3.  Polynomial pushes output into `history` (shifts everything down)
4.  Loop repeats

The buffer size determines how far back you can look. Larger buffers = longer memory.

For a 100-sample buffer at 48 kHz:
```text
100 samples ÷ 48000 Hz ≈ 2 ms of history
```
This is how you build delays, reverbs, resonant filters, and anything else that needs temporal memory.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Stability Warning" >}}

**Critical rule**: Keep feedback coefficients summing to \< 1.0 for guaranteed stability.

**Safe:**
```cpp
return 0.6*history[0] + 0.3*history[1];  // sum = 0.9 < 1.0
```
**Dangerous:**
```cpp
return 1.2*history[0];  // WILL EXPLODE (unbounded growth)
```
Why? Each cycle multiplies previous output by 1.2. Exponential growth. Your speakers won't thank you.

MayaFlux won't stop you. This is a creative tool, not a safety guard. Instability can be interesting (briefly). Controlled feedback explosion creates chaotic textures.

But for stable filters: keep gain \< 1.0.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Initial Conditions" >}}

Recursive polynomials need starting values. Default: `[0.0, 0.0, ...]`

You can seed them:
```cpp
recursive->set_initial_conditions({0.5, -0.3, 0.1});
```
**Why?**

1.  **Impulse responses**: Inject energy without external input. The filter "pings" on its own.
2.  **Self-oscillation**: Non-zero initial conditions + feedback gain ≥ 1.0 = continuous tone.
3.  **Warm start**: Resume from previous state instead of cold-starting at zero.

Example (resonant ping):
```cpp
auto resonator = vega.Polynomial(
    [](std::span<double> history) {
        return 0.99 * history[0] - 0.5 * history[1];
    },
    PolynomialMode::RECURSIVE,
    2
);
resonator->set_initial_conditions({1.0, 0.0});  // kick-start the resonance
```
---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Karplus-Strong string synthesis (plucked string)
auto string = vega.Polynomial(
    [](std::span<double> history) {
        return 0.996 * (history[0] + history[1]) / 2.0;  // lowpass + feedback
    },
    PolynomialMode::RECURSIVE,
    100  // ~480 Hz at 48kHz
);
string->set_initial_conditions(
    std::vector<double>(100, vega.Random(-1.0, 1.0))
);  // noise burst

// Nonlinear resonator (saturating feedback)
auto nonlinear = vega.Polynomial(
    [](std::span<double> history) {
        double fb = 0.8 * history[0];
        return std::tanh(fb * 3.0);  // soft saturation in loop
    },
    PolynomialMode::RECURSIVE,
    1
);

// Comb filter (delay-based coloration)
auto comb = vega.Polynomial(
    [](std::span<double> history) {
        return history[0] + 0.5 * history[50];  // 50-sample delay
    },
    PolynomialMode::RECURSIVE,
    50
);
```

---

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

{{< tutorial-card step="3 of 7" title="Logic as Decision Maker" open="true" >}}

#### The Simplest Path

Run this code. You'll hear rhythmic pulses.
```cpp
void compose() {
    auto buffer = vega.AudioBuffer() | Audio[0];

    // Logic node: threshold detection
    auto logic = vega.Logic(LogicOperator::THRESHOLD, 0.0);

    auto processor = MayaFlux::create_processor<LogicProcessor>(
        buffer,
        logic
    );

    processor->set_modulation_type(LogicProcessor::ModulationType::REPLACE);

    // Feed a sine wave into the logic node
    auto sine = vega.Sine(2.0);
    logic->set_input_node(sine);
}
```
What you hear: a 2 Hz pulse train, beeping every half second.

The sine wave crosses zero twice per cycle. Logic detects the crossings. Output becomes binary: 1.0 (high) or 0.0 (low).

{{< tutorial-detail title="Expansion 1: What Logic Does" >}}

`LogicProcessor` makes **binary decisions** about audio.

Every sample asks: *"Is this value TRUE or FALSE?"* (based on threshold)

Output: 0.0 or 1.0.

**Uses:**

- **Gate**: Silence audio below threshold (noise reduction)
- **Trigger**: Fire events when signal crosses boundary (drums, envelopes)
- **Rhythm**: Convert continuous modulation into discrete beats

Example: Feed a slow LFO (0.5 Hz sine) into logic → square wave clock.

Digital doesn't care what the input "means", only whether it passes the test.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Logic node needs an input" >}}

`Logic` nodes need an input signal to evaluate. This is also true for other nodes like `Polynomial`.

So far, you did not have to manually set inputs because you used `SoundContainerBuffer` which automatically feeds audio into processors.

So, instead of creating an `AudioBuffer`, you can load a file:
```cpp
auto sound = vega.read_audio("path/to/file.wav") | Audio;
auto buffers = get_io_manager::get_audio_buffers(sound);
auto logic = vega.Logic(LogicOperator::THRESHOLD, 0.0);

auto processor = MayaFlux::create_processor<LogicProcessor>(
    buffer[0],
    logic
);

processor->set_modulation_type(LogicProcessor::ModulationType::REPLACE);
```
The audio from the file is automatically fed into the logic node. Considering how all previous examples relied on file contents, and the natutre of rhythmic pulses not exploiting the intricacies or richness of audio files, we are using a sine wave as inputs of the logic node in the main example.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: LogicOperator Types" >}}

`LogicOperator` defines the test:

- **THRESHOLD**: `x > threshold` → 1.0, else 0.0
- **HYSTERESIS**: Two thresholds (open/close) to avoid flutter
- **EDGE**: Trigger on transitions (0→1 or 1→0)
- **AND/OR/XOR/NOT**: Boolean algebra on current vs. previous sample
- **CUSTOM**: Your function

Right now you're using THRESHOLD, the simplest test.

Example (hysteresis gate for noisy signals):
```cpp
auto gate = vega.Logic(LogicOperator::HYSTERESIS);
gate->set_hysteresis_thresholds(0.1, 0.3);  // open at 0.3, close at 0.1
```
Signal must exceed 0.3 to open, then drops below 0.1 to close. Prevents rapid on/off flickering.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: ModulationType - Readymade Transformations" >}}

**ModulationType** provides readymade ways to apply binary logic to audio:

**Basic Operations:**

- **REPLACE**: Audio becomes 0.0 or 1.0 (bit reduction)
- **MULTIPLY**: Audio × logic (standard gate - preserves timbre)
- **ADD**: Audio + logic (adds impulse on logic high)

**Creative Operations:**

- **INVERT_ON_TRUE**: Phase flip when logic high (ring mod effect)
- **HOLD_ON_FALSE**: Freeze audio when logic low (granular stutter)
- **ZERO_ON_FALSE**: Hard silence when logic low (noise gate)
- **CROSSFADE**: Smooth fade based on logic (dynamic blending)
- **THRESHOLD_REMAP**: Binary amplitude switch (tremolo from logic)
- **SAMPLE_AND_HOLD**: Freeze on logic changes (glitch/stutter)
- **CUSTOM**: Your function

Example (granular freeze effect):
```cpp
processor->set_modulation_type(
    LogicProcessor::ModulationType::HOLD_ON_FALSE
);
// Audio freezes whenever logic goes low - creates stuttering repeats
```
Example (amplitude tremolo):
```cpp
processor->set_modulation_type(
    LogicProcessor::ModulationType::THRESHOLD_REMAP
);
processor->set_threshold_remap_values(1.0, 0.2);
// Creates rhythmic volume changes based on logic pattern
```
Logic becomes a **compositional control** for transforming audio in musical ways.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Hard gate (silence below threshold)
auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.2);
auto proc = MayaFlux::create_processor<LogicProcessor>(buffer, gate);
proc->set_modulation_type(LogicProcessor::ModulationType::ZERO_ON_FALSE);

// Granular stutter (freeze on quiet passages)
auto freeze = vega.Logic(LogicOperator::THRESHOLD, 0.3);
auto proc = MayaFlux::create_processor<LogicProcessor>(buffer, freeze);
proc->set_modulation_type(LogicProcessor::ModulationType::HOLD_ON_FALSE);

// Bit crusher (reduce to 1-bit audio)
auto crusher = vega.Logic(LogicOperator::THRESHOLD, 0.0);
auto proc = MayaFlux::create_processor<LogicProcessor>(buffer, crusher);
proc->set_modulation_type(LogicProcessor::ModulationType::REPLACE);

// Rhythmic tremolo from LFO
auto lfo = vega.Sine(4.0);  // 4 Hz
auto trem_logic = vega.Logic(LogicOperator::THRESHOLD, 0.0);
trem_logic->set_input_node(lfo);
auto proc = MayaFlux::create_processor<LogicProcessor>(buffer, trem_logic);
proc->set_modulation_type(
    LogicProcessor::ModulationType::THRESHOLD_REMAP
);
proc->set_threshold_remap_values(1.0, 0.3);  // Pumping rhythm
```

---

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

{{< tutorial-card step="4 of 7" title="Combining Polynomial + Logic" open="true" >}}

#### The Pattern

Load a file. Detect transients with logic. Apply polynomial only when transient detected.
```cpp
void compose() {
    auto sound = vega.read_audio("drums.wav") | Audio;
    auto buffers = get_io_manager::get_audio_buffers(sound);

    auto bitcrush = vega.Logic(LogicOperator::THRESHOLD, 0.0);
    auto crush_proc =
        MayaFlux::create_processor<LogicProcessor>(
            buffers[0], bitcrush
        );
    crush_proc->set_modulation_type(
        LogicProcessor::ModulationType::REPLACE
    );

    // Step 2: Freeze audio in chunks - granular stutter
    auto clock = vega.Sine(4.0); // 4 Hz freeze rate
    auto freeze_logic = vega.Logic(LogicOperator::THRESHOLD, 0.0);
    freeze_logic->set_input_node(clock);
    auto freeze_proc =
        MayaFlux::create_processor<LogicProcessor>(
            buffers[0], freeze_logic
        );
    freeze_proc->set_modulation_type(
        LogicProcessor::ModulationType::HOLD_ON_FALSE
    );

    // Step 3: Extreme waveshaping distortion
    auto destroyer = std::make_shared<Polynomial>([](double x) {
        return std::copysign(1.0, x) *
            std::pow(std::abs(x), 0.3); // Extreme compression
    });
    auto poly_proc =
        MayaFlux::create_processor<PolynomialProcessor>(
            buffers[0], destroyer
        );

    chain->add_processor(crush_proc, buffers[0]);
    chain->add_processor(freeze_proc, buffers[0]);
    chain->add_processor(poly_proc, buffers[0]);

    buffers[0]->set_processing_chain(chain);
}
```

{{< tutorial-detail title="Expansion 1: Processing Chains as Transformation Pipelines" >}}

You just built a **transformation pipeline**:
```text
bitcrush → freeze → destroy
```
Each processor transforms the output of the previous one. This is **compositional signal processing**: you build complex effects by chaining simple operations.

The power comes from **order dependency**:
```text
gate → distort    // Clean transients, heavy saturation
distort → gate    // Distorted everything, then choppy
```
Swap the order = completely different sound.

Extend it:
```text
detect transients → sample-and-hold → bitcrush → wavefold → compress
```
Traditional plugins give you "distortion with 3 knobs." You compose the distortion algorithm itself.

**Every processor is a building block.** Chain them to create effects that don't exist as plugins:

- Bitcrush → Freeze → Invert = Glitch stutterer
- Remap → Fold → Gate = Rhythmic harmonizer
- Threshold → Hold → Distort = Transient emphasizer

Logic + Polynomial + Chains = **programmable audio transformation system**.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Chain Order Matters" >}}

Swap the order of logic and polynomial → different result:
```text
Logic → Polynomial  // Detect, then distort
Polynomial → Logic  // Distort, then detect
```
Processors are **non-commutative**. Audio math doesn't follow algebra rules.

Order determines signal flow. You're building a graph, not an equation.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Adaptive dynamics (compress quiet, expand loud)
auto logic = vega.Logic(LogicOperator::THRESHOLD, 0.3);
auto poly_compress =
    vega.Polynomial([](double x) { return x * 2.0; });
auto poly_expand =
    vega.Polynomial([](double x) { return x * 0.5; });

// Route based on logic state (requires custom modulation)
```

---

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

{{< tutorial-card step="5 of 7" title="Processing Chains and Buffer Architecture" open="true" >}}

#### Tutorial: Explicit Chain Building

##### The Simplest Path

You've been adding processors one at a time. Now control their order explicitly.
```cpp
void compose() {
    auto sound = vega.read_audio("path/to/file.wav") | Audio;
    auto buffer = get_io_manager::get_audio_buffers(sound)[0];

    // Create an empty chain
    auto chain = MayaFlux::create_processing_chain();

    // Build the chain: Distortion → Gate → Compression
    auto distortion =
        vega.Polynomial([](double x) { return std::tanh(x * 2.0); });
    auto gate =
        vega.Logic(LogicOperator::THRESHOLD, 0.1);
    auto compression =
        vega.Polynomial([](double x) {
            return x / (1.0 + std::abs(x));
        });

    chain->add_processor(
        std::make_shared<PolynomialProcessor>(distortion),
        buffer
    );
    chain->add_processor(
        std::make_shared<LogicProcessor>(gate),
        buffer
    );
    chain->add_processor(
        std::make_shared<PolynomialProcessor>(compression),
        buffer
    );

    // Attach the chain to the buffer
    buffer->set_processing_chain(chain);
}
```
Run this. You hear: clean audio → saturated → gated (silence below threshold) → compressed (controlled peaks).

**Swap the order:**
```cpp
chain->add_processor(gate_processor);       // Gate first
chain->add_processor(distortion_processor); // Then distort
chain->add_processor(compression_processor);
```
Different sound. Order matters.

{{< tutorial-detail title="Expansion 1: What `create_processor()` Was Doing" >}}

Previously, when you wrote:
```cpp
auto processor =
    MayaFlux::create_processor<PolynomialProcessor>(buffer, poly);
```
MayaFlux did this behind the scenes:

1.  Created the processor
2.  Got the buffer's existing processing chain
3.  **Automatically added the processor to that chain**
4.  Returned the processor

You didn't see this because it was implicit. The processor was silently appended to whatever chain existed.

**Now you're building chains explicitly:**
```cpp
auto chain = MayaFlux::create_processing_chain();  // Empty chain
chain->add_processor(proc1);  // Manual control
chain->add_processor(proc2);
buffer->set_processing_chain(chain);  // Replace buffer's chain
```
**When to use explicit chains:**

- You need precise order control
- You're building reusable processor "presets"
- You want to swap entire chains dynamically (e.g., switch between clean/distorted modes)
- You're debugging processor interactions

**When implicit is fine:**

- Simple cases (1-2 processors)
- Order doesn't matter (parallel-like effects)
- Rapid prototyping

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Chain Execution Order" >}}

Chains execute like a for-loop over processors:
```cpp
for (auto& processor : chain->get_processors()) {
    processor->process(buffer);
}
```
Data flows sequentially:
```text
Container → Buffer (512 samples)
    ↓
Processor₁: Distortion (modifies samples in-place)
    ↓
Processor₂: Gate (zeroes out quiet samples)
    ↓
Processor₃: Compression (reduces peaks)
    ↓
Speakers
```
Each processor sees the **output** of the previous processor.

**This is not parallel processing.** No branches. No simultaneous paths. Pure sequential transformation.

(Parallel routing requires `BufferPipeline`, which is covered in a later tutorial.)

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Default Processors vs. Chain Processors" >}}

Every buffer has two processing stages:

**Stage 1: Default Processor** (runs first, always)

- Defined by buffer type
- Handles data **acquisition** or **generation**
- Examples:
  - `SoundContainerBuffer`: reads from file/stream
  - `NodeBuffer`: evaluates a node
  - `FeedbackBuffer`: mixes current + previous buffer
  - `AudioBuffer`: none (generic accumulator)

**Stage 2: Processing Chain** (runs second)

- Your custom processors
- Handles data **transformation**
- Examples: filters, waveshaping, logic, etc.

**Execution flow:**
```text
1. Buffer's default processor runs (fills buffer with data)
2. Processing chain runs (transforms that data)
3. Result goes to speakers
```
When you add processors via `create_processor()`, they go into **Stage 2** (the chain).

The **default processor** is fixed per buffer type. You can replace it, but usually you don't need to. The chain is where creativity happens.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Stack multiple distortions (cascading saturation)
auto chain = MayaFlux::create_processing_chain();
auto light =
    vega.Polynomial([](double x) { return std::tanh(x * 1.5); });
auto heavy =
    vega.Polynomial([](double x) { return std::tanh(x * 5.0); });
auto fold =
    vega.Polynomial([](double x) { return std::sin(x * 3.0); });

chain->add_processor(
    std::make_shared<PolynomialProcessor>(light),
    buffer
);
chain->add_processor(
    std::make_shared<PolynomialProcessor>(heavy),
    buffer
);
chain->add_processor(
    std::make_shared<PolynomialProcessor>(fold),
    buffer
);

// Insert gating between stages
auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.2);
chain->add_processor(
    std::make_shared<PolynomialProcessor>(light),
    buffer
);
chain->add_processor(
    std::make_shared<LogicProcessor>(gate),
    buffer
);  // Gate the distortion
chain->add_processor(
    std::make_shared<PolynomialProcessor>(heavy),
    buffer
); // Distort the gated signal

buffer->set_processing_chain(chain);
```

{{< /tutorial-detail >}}

{{< /tutorial-card >}}
