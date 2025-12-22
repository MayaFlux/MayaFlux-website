---
title: "From MaxMSP to MayaFlux: Transcending the sanitized digital realm"
---

<div class="card wide">
<h2> Max/MSP, PD, and Where This Document Actually Begins </h2>

Max/MSP occupies a unique position in creative computation. Out of the box, its scope is vast:

- real-time audio (MSP)
- low-level DSP authoring (gen~)
- graphics and video (Jitter)
- UI construction
- MIDI, OSC, device integration
- and object ecosystems accumulated over decades.

That breadth matters. It shaped how many people learned to think computationally
by assembling systems visually, combining sound, image, time, and interaction inside a single environment.

But breadth also hides assumptions. Pure Data is not a “cut-down Max.”
It is intentionally minimal, making very few assumptions, exposing its mechanics directly,
and staying close to computational essentials: message passing, signal processing, explicit timing, and visible state.

PD can reproduce a surprising amount of Max functionality (even gen~) like behavior via C externals,
which is far more straightforward than most expect. Yet it refuses to
become an all-encompassing environment. That restraint is architectural, not accidental.

Because of this, PD is the clearest place to surface assumptions you may not realize you learned.
If you come from Max/MSP, read the <strong><a href="../pure_data/">Pure Data onboarding</a></strong>
onboarding before continuing—not because you “need PD knowledge” to understand MayaFlux,
but because PD strips the ecosystem down to its bones, making visible what Max’s scale, polish, and convenience layers can obscure.

</div>

---

<div class="card">

## What This Section Is About (and What It Isn’t)

This document is **not** a general tour of Max/MSP.

It does not attempt to cover UI widgets, Live workflows, object libraries,
or patch-as-artifact practices.

Instead, it focuses on two places where Max most clearly exposes
its internal tensions:

- **gen~** — where DSP becomes computational
- **Jitter** — where data becomes multi-dimensional

These are the points where Max stops simulating hardware
and starts brushing against digital-first thinking.

Do not read this as a translation guide.
Do not look for object-by-object equivalents.

Read it to notice moments where you think:

_“I always assumed this had to work this way.”_

Those assumptions — about time, data, state, and structure —
are the subject of this Rosetta Stone.

</div>

---

<div class="card wide">

## gen~: A Paradigm Trapped Inside Another Paradigm

<p>
gen~ was a radical addition to Max/MSP.
</p>

<p>
It gave you sample-by-sample processing. Explicit state management. The ability to <em>author</em> DSP algorithms instead of just assembling pre-made objects. Mathematical thinking instead of cable routing.
</p>

<p>
But here's what's rarely said out loud:
</p>

<p>
<strong>gen~ is constantly fighting the architectural assumptions of the environment it lives inside.</strong>
</p>
</div>

---

<div class="card-group columns breakout">

<div class="card vertical">
<h2>What gen~ Was Trying to Be</h2>

gen~ emerged from a simple realization: **the patcher paradigm breaks down when you need to think sample-by-sample.**

Visual patching is brilliant for:

- Routing messages between objects
- Assembling signal chains from pre-built units
- Making control flow visible and manipulable

But it becomes **architecturally incoherent** when you need:

- Access to previous samples for recursive algorithms
- Conditional logic evaluated 48,000 times per second
- State that persists across multiple samples
- Mathematical transformations that don't map to "objects connected by cables"

gen~ tried to solve this by creating a **sandboxed computational space** where you could think in transformations instead of object graphs.

And it worked—sort of.

</div>

<div class="card vertical">
<h2> What gen~ Is Actually Fighting</h2>

The problem is that gen~ didn't get to redesign Max's architecture. It had to **coexist** with it.

This created fundamental tensions that shape everything you do in gen~:

<details>
<summary><strong>Visual Patching vs Computational Density</strong></summary>

gen~ patches are still **visual graphs**. But sample-accurate DSP requires thinking in dense mathematical expressions, not spatially-arranged boxes.

Want to implement a simple FIR filter with 64 taps?  
In code: `sum(coefficients[i] * history[i])` for i in 0..63  
In gen~: 64 `[*]` objects, 64 `[history]` objects, 63 `[+]` objects, all carefully arranged spatially

**The visual paradigm becomes cognitive overhead** instead of clarifying aid. You're not looking at the algorithm—you're looking at plumbing.

</details>

<details>
<summary><strong>Message Scheduling vs Sample-Rate Computation</strong></summary>

Max's scheduler runs at ~1ms resolution. Audio runs at ~20μs resolution (48kHz).

gen~ bridges this by being **completely isolated from Max's scheduler**. It runs in its own thread, protected from message-timing vagaries.

This is architecturally sensible. It's also **a massive constraint**.

**You cannot**:

- Have sample-accurate control from Max messages (everything gets smoothed/interpolated)
- Trigger events from gen~ back to Max at sample rate (you'd flood the scheduler)
- Coordinate gen~ processes with sample-accurate timing (each gen~ is its own island)

The sandboxing that makes gen~ work **also makes it fundamentally separate** from the rest of your patch.

</details>

<details>
<summary><strong>Object Metaphors vs Data Flow</strong></summary>

Max teaches you to think in **objects with inputs and outputs**.

gen~ needs you to think in **mathematical transformations of data streams**.

But gen~ is still built from objects: `[+]`, `[*]`, `[>]`, `[history]`

So you end up with this hybrid: **mathematical thinking expressed through object metaphors**.

```
[> 0.5]  // Is this "a comparator object" or "the boolean expression x > 0.5"?
```

The answer is both, and that cognitive friction is always present.

</details>

<details>
<summary><strong>Explicit State Management in a Stateless Paradigm</strong></summary>

Max's core paradigm is **stateless message passing**. Objects don't inherently remember anything—state is an exception you explicitly manage with `[int]`, `[float]`, etc.

gen~ inherits this, so **history is expensive and explicit**:

```
[history 1]  // Allocate one sample of memory
[history 2]  // Allocate two samples of memory
[history 100]  // Now you're manually managing a circular buffer
```

You learn to **treat memory as scarce** because the paradigm makes it architecturally heavyweight.

But in actual DSP, history is **fundamental**. Filters are defined by feedback. Recursive algorithms are ubiquitous. Every oscillator maintains phase state.

gen~ makes you work around its paradigm instead of working with your algorithm.

</details>

<details>
<summary><strong> Analog Metaphors Constraining Digital Thinking</strong></summary>

Max/MSP was designed to **simulate modular synthesis**. Even gen~ carries forward assumptions from this:

- Signals are continuous streams (not discrete transformations)
- Processing happens in "objects" (not functions)
- State is exceptional (not fundamental)
- Sample-by-sample means "like analog hardware, but digital"

But **digital computation is not analog simulation**.

</details>

gen~ gives you the ability to think digitally, then constrains you to express it through analog-derived metaphors. You can build a Karplus-Strong string, but you have to think about "delay lines" and "feedback" instead of recursive difference equations.

</div>

<div class="card vertical">
<h2>The Architectural Bind</h2>

gen~ occupies an uncomfortable middle ground:

**Too computational for Max's paradigm**  
(sample-rate logic doesn't fit message-passing architecture)

**Too constrained by Max's paradigm**  
(forced to express transformations through visual object graphs)

This creates friction at every level:

- You think in math but patch in objects
- You need dense computation but work in visual space
- You want recursive algorithms but manage explicit history
- You have sample-accurate potential but scheduler-isolated reality

**gen~ is brilliant at what it achieved within these constraints.**

But the constraints remain.

And once you recognize them as **architectural choices, not computational laws**, the question becomes:

**What becomes possible when you design for digital-first thinking from the ground up?**

</div>

</div>

---

# Seeing the Difference: Karplus-Strong String Synthesis

Let's start with a classic: Karplus-Strong plucked string. This is something gen~ handles well, and comparing the approaches reveals different ways of thinking about the same algorithm.

---

## In gen~: Objects Express the Algorithm

Here's a Karplus-Strong string in gen~:

```
[noise~]
    |
[*~ 0.5]  // initial pluck
    |
[+~ [history 100]]
        |
    [*~ 0.5]  // averaging
        |
    [+~ [history 1]]
        |
    [*~ 0.5]  // averaging (complete lowpass)
        |
    [*~ 0.996]  // feedback attenuation
        |
    [history 100]
        |
    [out 1~]
```

The visual layout shows the signal path. Each `[history]` object allocates memory. The feedback routing is explicit. You build the algorithm by connecting objects spatially.

This makes the **data flow** visible. You can see exactly where feedback happens, where filtering occurs, where attenuation applies.

---

## In MayaFlux: Mathematics Is Code

```cpp
auto string = vega.Polynomial(
    [](const std::deque<double>& history) {
        return 0.996 * (history[0] + history[1]) / 2.0;
    },
    PolynomialMode::RECURSIVE,
    100  // delay length determines pitch (~480Hz at 48kHz)
)[0] | Audio;

// Excite with noise burst
string->set_initial_conditions(
    std::vector<double>(100, get_uniform_random(-1.0, 1.0))
);

auto random = vega.Random();
string->set_input_node(random);
```

The **algorithm itself** is literal code: `0.996 * (history[0] + history[1]) / 2.0`

This is the same mathematics you'd write on paper. No object placement. No manual wiring. History is just a parameter you declare (100 samples).

**Different thinking, same result.** gen~ shows you the routing. MayaFlux shows you the equation.

---

## Expanding the Pattern: Temporal Coordination

Once you have the string, new patterns open up. Here's sample-accurate rhythmic coordination between multiple recursive processes:

```cpp
auto rhythm_pattern = [](Vruta::TaskScheduler& sched) -> Vruta::SoundRoutine {
    auto& promise = co_await Kriya::GetPromise{};

    auto string1 = vega.Polynomial(/*...Karplus-Strong 1...*/)[0] | Audio;
    auto string2 = vega.Polynomial(/*...Karplus-Strong 2...*/)[1] | Audio;

    while (!promise.should_terminate) {
        // Pluck first string
        string1->set_initial_conditions(noise_burst());

        // Wait exactly 1024 samples
        co_await SampleDelay{1024};

        // Pluck second string in exact rhythmic relationship
        string2->set_initial_conditions(noise_burst());

        co_await SampleDelay{2048};
    }
};
```

Two recursive algorithms coordinated with **exact sample-count relationships**. The coroutine suspends, waits precisely 1024 samples, resumes, triggers the next event.

This is temporal composition as code. The timing relationships are explicit. The coordination is sample-accurate.

---

## Buffer Pipelines: Data Flow as Syntax

Complex buffer operations become declarative:

```cpp
auto input = vega.AudioBuffer()[0] | Audio;
auto output = vega.AudioBuffer()[1] | Audio;

auto pipeline = MayaFlux::create_buffer_pipeline();
pipeline->with_strategy(ExecutionStrategy::PHASED);

pipeline
    // Accumulate 20 buffer cycles
    >> BufferOperation::capture_from(input)
        .for_cycles(20)

    // Transform accumulated batch
    >> BufferOperation::transform([](const auto& data, uint32_t cycle) {
        auto samples = std::get<std::vector<double>>(data);
        // Process 20 * 512 samples as single batch
        for (auto& s : samples) {
            s = std::tanh(s * 2.0);
        }
        return samples;
    })

    >> BufferOperation::route_to_buffer(output);

pipeline->execute_buffer_rate();
```

The `>>` operator composes operations. `PHASED` strategy means: gather all captures first, then transform, then route. Data flow becomes syntactic structure.

This isn't better or worse than careful `[buffer~]` management—it's a different way of expressing the same computational intent.

---

## Computation Grammars: Data Evaluates Itself

Here's a pattern that emerges from digital-first thinking: let data characteristics determine transformations:

```cpp
auto grammar = std::make_shared<ComputationGrammar>();

grammar->create_rule("dynamic_compression")
    .matches_type<std::vector<double>>()
    .when([](const std::any& data, const ExecutionContext& ctx) {
        const auto& samples = std::any_cast<std::vector<double>>(data);
        return peak_level(samples) > 0.8;
    })
    .executes([](const std::any& data, const ExecutionContext& ctx) {
        auto samples = std::any_cast<std::vector<double>>(data);
        for (auto& s : samples) {
            s = std::tanh(s * 2.0);
        }
        return samples;
    })
    .build();

auto pipeline = std::make_shared<ComputePipeline<InputType, OutputType>>(grammar);

// Apply to buffer processor
MayaFlux::attac_quick_process([grammar, pipeline](auto& buf) {
    buf->get_data() = pipeline->process(input_data);
}, your_audio_buffer);
```

Every processing cycle, the grammar evaluates: "Does this data match the rule conditions?" If peak level > 0.8, apply compression. The transformation is determined by **what the data is**, not just preset routing.

This is data-driven processing—rules that activate based on signal characteristics.

---

## The Pattern

**gen~ taught many of us to think in:**

- Visual signal routing
- Explicit state management
- Sample-by-sample processing

**MayaFlux explores:**

- Algorithms as literal mathematics
- History as natural parameter
- Temporal relationships as coroutine syntax
- Data flow as declarative composition
- Rules that evaluate data characteristics

Not better. **Different substrate, different patterns.**

---

# Where gen~'s Architecture Shows: Real-World Patterns

The Karplus-Strong example shows mathematical clarity. But gen~'s constraints become more apparent when you build systems that need:

- **Complex state machines** (onset detection, envelope followers, pattern recognition)
- **Multi-rate processing** (running different algorithms at different update rates)
- **Dynamic algorithm switching** (changing processing based on signal analysis)
- **Cross-process coordination** (multiple gen~ instances working together)

Let's examine real patterns where the visual/object paradigm creates friction.

---

## Pattern 1: Envelope Follower with State-Dependent Behavior

### What You Actually Need

An envelope follower that changes its attack/release characteristics based on signal behavior. If the signal is percussive (fast transients), use fast attack. If sustained (smooth), use slower tracking.

### In gen~: State Management Sprawl

```
[in 1]
    |
[delta~]  // differentiate to detect rate of change
    |
[abs~]
    |
[>~ 0.1]  // threshold for "fast transient"
    |
[history 1]  // store previous state
    |
[change~]  // detect state transitions
    |
[selector~]  // route to different envelope followers
    |  |
  [env1] [env2]  // two separate envelope follower patches
    |  |
    +--+
       |
    [out 1]
```

But this doesn't work well because:

1. `[selector~]` switches abruptly (clicks)
2. You need to maintain attack/release state for BOTH followers even though only one is active
3. The state transition logic requires multiple `[history]` objects to track hysteresis
4. Crossfading between states requires duplicating the entire signal path

**The real gen~ patch balloons to 30+ objects** just to handle state-dependent envelope following.

### In MayaFlux: State Is Accessible

```cpp
auto envelope_func = [](const std::deque<double>& history) {
    // Access both input history and output history
    double input = history[0];
    double prev_output = history[1];
    double rate_of_change = std::abs(input - history[2]);

    // State-dependent attack/release
    double attack = (rate_of_change > 0.1) ? 0.01 : 0.1;
    double release = (rate_of_change > 0.1) ? 0.1 : 0.5;

    // Single envelope follower with adaptive coefficients
    double coeff = (input > prev_output) ? attack : release;
    return prev_output + coeff * (std::abs(input) - prev_output);
};

auto envelope_follower = vega.Polynomial(
    envelope_func,
    PolynomialMode::RECURSIVE,
    3  // track current input, previous output, previous input
)[0] | Audio;

```

<details>

<summary><strong>or at buffer level directly:</strong></summary>

```cpp
auto buffer = vega.NodeBuffer(vega.Sine(200))[0] | Audio;

auto env_processor = MayaFlux::create_processor<PolynomialProcessor>(
    buffer,
    PolynomialProcessor::ProcessMode::SAMPLE_BY_SAMPLE,
    64,
    envelope_func,
    PolynomialMode::RECURSIVE,
    3
);
```

</details>

**One node.** The state logic is in the algorithm itself. No object sprawl. No manual state tracking. The coefficients adapt based on signal characteristics **within the same processing function**.

---

## Pattern 2: Spectral Gate with History-Based Triggering

### What You Actually Need

A gate that opens when a specific frequency band has been loud for N consecutive samples, but closes immediately if it drops below threshold. This prevents false triggers from transient noise.

### In gen~: Architectural Impossibility

You can't do multi-bin FFT analysis in gen~ at all—it requires `[pfft~]` which lives outside gen~ in MSP. So you'd need:

1. Route signal out of gen~ to `[pfft~]`
2. Analyze bins in Max (not sample-accurate)
3. Send gate control back to gen~ via inlet (smoothed, not sample-accurate)
4. Manage threshold crossing with `[history]` chains in gen~

**The sample-accurate gate you wanted is now impossible** because spectral analysis can't happen inside gen~'s sample-rate world.

### In MayaFlux: Unified Processing Domain

```cpp
// Logic node with history-based triggering
auto spectral_gate = vega.Logic(
    [](const std::deque<bool>& history) {
        // Custom logic: need 10 consecutive true samples to open
        int consecutive_true = 0;
        for (int i = 0; i < std::min(10, (int)history.size()); i++) {
            if (history[i]) consecutive_true++;
            else break;
        }
        return consecutive_true >= 10;
    },
    LogicMode::SEQUENTIAL,
    10
)[0] | Audio;

// Feed it spectral analysis data
auto buffer = vega.AudioBuffer()[0] | Audio;

// Process buffer with spectral analysis, then threshold
auto pipeline = MayaFlux::create_buffer_pipeline();
    pipeline
    >> BufferOperation::capture_from(buffer).for_cycles(1)
    >> BufferOperation::transform([](const auto& data, uint32_t cycle) {
        auto samples = std::get<std::vector<double>>(data);
        // Perform FFT, extract specific bin
        double bin_magnitude = analyze_bin(samples, 440.0, 48000);
        return std::vector<double>{bin_magnitude};
    })
    >> BufferOperation::route_to_buffer(analysis_buffer);

// Gate logic node reads the analysis
spectral_gate->set_input_node(analysis_buffer_node);

// Use gate to control another process
spectral_gate->on_change_to(true, [synth]() {
    synth->set_amplitude(1.0);
});

spectral_gate->on_change_to(false, [synth]() {
    synth->set_amplitude(0.0);
});
```

<details>
<summary><strong>Or using buffer level processors directly:</strong></summary>

```cpp
auto buffer = vega.AudioBuffer()[0] | Audio;
auto analysis_buffer = vega.AudioBuffer();

auto pipeline = MayaFlux::create_buffer_pipeline();
pipeline
    >> BufferOperation::capture_from(buffer).for_cycles(1)
    >> BufferOperation::transform([](const auto& data, uint32_t cycle) {
        auto samples = std::get<std::vector<double>>(data);
        double bin_magnitude = analyze_bin(samples, 440.0, 48000);
        return std::vector<double>{bin_magnitude};
    })
    >> BufferOperation::route_to_buffer(analysis_buffer);

// Logic processor with history-based triggering
auto gate = MayaFlux::create_processor<LogicProcessor>(
    analysis_buffer,
    [](const std::deque<bool>& history) {
        int consecutive_true = 0;
        for (int i = 0; i < std::min(10, (int)history.size()); i++) {
            if (history[i]) consecutive_true++;
            else break;
        }
        return consecutive_true >= 10;
    },
    LogicMode::SEQUENTIAL,
    10
);

gate->set_modulation_type(LogicProcessor::ModulationType::ZERO_ON_FALSE);

```

</details>

**Spectral analysis, history-based logic, and sample-accurate triggering** all in one coherent system. No domain boundaries. No architectural impossibilities.

---

## Pattern 3: Dual-Rate Modulation (Slow LFO, Fast Audio-Rate Warping)

### What You Actually Need

An LFO running at 0.5Hz that controls the frequency of an audio-rate oscillator, but with audio-rate phase distortion applied to the LFO itself based on the oscillator's output.

This is: slow modulation → fast oscillator → fast feedback to modulation shaping.

### In gen~: Rate Mismatch Hell

gen~ runs everything at audio rate. There's no "slow rate" for LFOs. So:

1. Your 0.5Hz LFO still generates 48,000 samples per second (99.99% redundant computation)
2. The feedback path requires `[history]` to store the oscillator output
3. Phase distortion requires manually managing phase accumulation with more `[history]` objects
4. You can't dynamically change the LFO rate without recreating the entire `[phasor~]` calculation

**It works, but wastes massive amounts of computation** on redundant LFO samples.

### In MayaFlux: Multi-Rate Is Natural

```cpp
// Slow LFO (evaluated 10 times per second, not 48000)
auto lfo = vega.Sine(0.5)[0] | Time(10);  // 10Hz update rate

// Audio-rate oscillator
auto osc = vega.Sine(440.0)[0] | Audio;

// Audio-rate phase distortion modulator
auto phase_distortion = vega.Polynomial(
    [](const std::deque<double>& history) {
        return std::fmod(history[0] + history[1] * 0.1, 1.0);
    },
    PolynomialMode::RECURSIVE,
    2
)[0] | Audio;

// Connect: LFO modulates oscillator frequency
osc->set_frequency_node(lfo);

// Oscillator output shapes phase distortion
phase_distortion->set_input_node(osc);

// Phase distortion modulates LFO (cross-rate feedback!)
lfo->on_tick([phase_distortion](NodeContext ctx) {
    // Read audio-rate distortion, use to warp LFO phase
    // This happens 10x per second, not 48000x
});
```

**Different update rates, cross-rate modulation, efficient computation.** The LFO isn't generating 48,000 useless samples. The phase distortion runs at audio rate. They coordinate without architectural friction.

---

## Pattern 4: Dynamic Algorithm Switching

### What You Actually Need

Switch between three different distortion algorithms based on input signal RMS:

- Below 0.3: Clean passthrough
- 0.3-0.7: Soft saturation
- Above 0.7: Hard clipping

### In gen~: Prebuild All Algorithms

```
[in 1]
    |
    +-- [slide~ 4410]  // 100ms RMS averaging
    |       |
    |   [selector~ 3]  // route to one of three algorithms
    |    /   |   \
    | [pass] [tanh] [clip]
    |    \   |   /
    |    [selector~ 3]  // route back from one of three
    |        |
    +--------+
         |
      [out 1]
```

**Problems:**

1. All three algorithms compute every sample (waste)
2. `[selector~]` creates clicks during transitions
3. Adding a fourth algorithm requires rewiring the entire patch
4. You can't change the thresholds without editing the visual patch

### In MayaFlux: Grammar-Driven Selection

```cpp
auto grammar = std::make_shared<ComputationGrammar>();

grammar->create_rule("passthrough")
    .with_priority(100)
    .matches_type<std::vector<double>>()
    .executes([](const std::any& input, const ExecutionContext& ctx) {
        auto samples = std::any_cast<std::vector<double>>(input);
        double rms = calculate_rms(samples);
        if (rms < 0.3) {
            return input;  // passthrough
        }
        return std::any{};  // doesn't match, try next rule
    })
    .build();

grammar->create_rule("soft_saturation")
    .with_priority(90)
    .matches_type<std::vector<double>>()
    .executes([](const std::any& input, const ExecutionContext& ctx) {
        auto samples = std::any_cast<std::vector<double>>(input);
        double rms = calculate_rms(samples);
        if (rms >= 0.3 && rms < 0.7) {
            for (auto& s : samples) s = std::tanh(s * 2.0);
            return samples;
        }
        return std::any{};
    })
    .build();

grammar->create_rule("hard_clipping")
    .with_priority(80)
    .matches_type<std::vector<double>>()
    .executes([](const std::any& input, const ExecutionContext& ctx) {
        auto samples = std::any_cast<std::vector<double>>(input);
        double rms = calculate_rms(samples);
        if (rms >= 0.7) {
            for (auto& s : samples) s = std::clamp(s, -1.0, 1.0);
            return samples;
        }
        return std::any{};
    })
    .build();

// Apply to buffer processing
auto pipeline = std::make_shared<ComputePipeline<InputType, OutputType>>(grammar);
// Grammar evaluates each cycle, picks ONE algorithm based on data characteristics
```

**Only the matched algorithm executes.** No clicks. Add/remove algorithms by adding/removing rules—no patch rewiring. Thresholds are in code, not visual layout.

---

## The Pattern Emerges

gen~'s architectural constraints become visible when you need:

1. **Complex state machines** → Object sprawl
2. **Cross-domain analysis** → Architectural impossibility
3. **Multi-rate efficiency** → Everything runs at audio rate anyway
4. **Dynamic algorithm selection** → Prebuild everything, selector switching

**MayaFlux patterns:**

1. **State is data** → Access history directly in algorithms
2. **Unified domains** → Spectral analysis, logic, audio in one space
3. **Multi-rate native** → Declare update rates, not workarounds
4. **Data-driven dispatch** → Grammars evaluate characteristics, not prebuilt routing

This isn't "MayaFlux is more powerful." It's: **different architecture enables different patterns.**

---

# A Note on Visuals and Cross-Modal Work

<div class="card subtle">

This document focuses deliberately on audio-rate and buffer-level computation,
because this is where gen~’s architectural constraints become most visible.

Visual processing, spatial data, and cross-modal coordination are not omissions.
They require their own substrate-level treatment.

Those sections will follow after:

- the _Sculpting Data_ visuals tutorial,
- the graphics programmer persona,
- and the p5.js onboarding are in place.

At that point, the same questions raised here about time, state, and structure
will be revisited through geometry, matrices, and GPU pipelines,
without forcing premature analogies.

For now, consider this the audio-side of a larger argument.

</div>
