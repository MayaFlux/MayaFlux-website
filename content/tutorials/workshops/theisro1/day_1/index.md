# ISRO Workshop: Day 1

_Numbers don't know they're audio. Numbers don't know they're pixels.
A resonant body vibrates. A point appears. The same data made both happen.
Today we build five systems that make this tangible._

---

Each section in this day follows one arc:

1. Run the code. Listen and watch.
2. Read the commentary below the code.
3. Open the expansions when you want to know how it works.
4. Change numbers. Break things. Rebuild.

Every example produces real sound and real graphics simultaneously.
The point is not the sound itself or the image itself.
The point is that they share the same substrate.

---

## Tutorial: First Bell

```cpp
void compose() { day1_a_first_bell(); }
```

Run this. You hear a metallic bell struck at slow, regular intervals. A spiral of points grows from
the center of the window. When the bell rings, the spiral accelerates. When it decays, the spiral
barely moves.

The bell is a `ModalNetwork`: 12 resonant modes with inharmonic frequency ratios.
The timing source is a `Sine` oscillator at 0.3 Hz. A `Logic` node watches that sine and fires
when it crosses a threshold. Each firing excites the bell at a random pitch and intensity.

The visual field reads the bell's audio buffer every frame, computes RMS energy, and uses that
energy to control spiral growth speed and point size.

Nothing here is "routed from audio to visuals." The bell's audio buffer is just a vector of doubles.
The visual callback reads that vector. Same numbers, different interpretation.

### Expansion 1: ModalNetwork

<details>
<summary>Click to expand: Physical modeling via frequency decomposition</summary>

`ModalNetwork` synthesizes sound by summing decaying sinusoidal oscillators (modes).
Each mode has a frequency ratio, amplitude, and decay rate. When you call `excite()`,
all modes receive energy simultaneously and decay independently.

```cpp
auto bell = vega.ModalNetwork(12, 220.0, ModalNetwork::Spectrum::INHARMONIC);
```

12 modes. 220 Hz fundamental. `INHARMONIC` means the frequency ratios are non-integer
multiples of the fundamental (1.0, 2.76, 5.40, ...). This is what makes it sound like
metal rather than a vibrating string.

The `[{ 0, 1 }]` syntax assigns the network to stereo output channels.
`| Audio` registers it with the audio-rate processing subsystem.

`excite(strength)` injects energy into all modes. `set_fundamental(freq)` shifts all
mode frequencies proportionally. `excite_at_position(pos, strength)` weights each
mode differently based on the excitation point, producing different timbres from the
same body.

</details>

### Expansion 2: Logic as Event Detector

<details>
<summary>Click to expand: Continuous to discrete conversion</summary>

The sine oscillator produces a continuous stream of values between -1.0 and 1.0.
The `Logic` node watches this stream and produces binary output: true or false.

```cpp
auto swing = vega.Logic(LogicOperator::THRESHOLD, 0.85);
trigger_osc >> swing;
```

`THRESHOLD` mode: output is true when input exceeds 0.85, false otherwise.
The `>>` operator connects the sine's output to the logic node's input.

`on_change_to(false, ...)` fires a callback the moment the logic transitions from
true to false. This happens once per sine cycle, at the peak. The callback excites the bell.

The result: one event per sine cycle. Perfectly periodic. We will break this regularity
in the next examples.

</details>

### Expansion 3: Visual Energy Mapping

<details>
<summary>Click to expand: Reading audio state from the graphics thread</summary>

```cpp
float energy = bell->get_audio_buffer().has_value()
    ? static_cast<float>(rms(bell->get_audio_buffer().value())) * 5.0F
    : 0.0F;
```

`get_audio_buffer()` returns the network's most recent output buffer (512 samples).
`rms()` computes the root mean square: a single number summarizing the buffer's energy.

The audio subsystem runs at 48 kHz. The visual callback runs at ~60 Hz via `schedule_metro(0.016, ...)`.
There is no synchronization lock between them. The visual thread reads whatever buffer the audio
thread last produced. This is safe because `get_audio_buffer()` returns an optional copy, not a
reference to live memory.

The energy value controls spiral growth rate, rotation speed, and point size. High energy (bell
ringing) produces rapid expansion. Low energy (bell decaying) produces near-stillness.

</details>

### Try It

```cpp
// Change the trigger rate: faster oscillator, more frequent strikes
auto trigger_osc = vega.Sine(1.5, 1.0);

// Change the threshold: higher = tighter trigger window
auto swing = vega.Logic(LogicOperator::THRESHOLD, 0.95);

// Change the spectrum: HARMONIC for a more tonal, string-like quality
auto bell = vega.ModalNetwork(12, 220.0, ModalNetwork::Spectrum::HARMONIC);

// More modes = richer spectrum, slower computation
auto bell = vega.ModalNetwork(32, 110.0, ModalNetwork::Spectrum::INHARMONIC);
```

---

## Tutorial: Wandering String

```cpp
void compose() { day1_b_wandering_string(); }
```

Run this. You hear a plucked string. The timing is irregular: sometimes events cluster together,
sometimes long silences. Each pluck has a different timbre because the pluck position shifts.
Points burst at the pluck location and accumulate until the field resets.

Click anywhere in the window to pluck the string manually. Left side = near the bridge (bright).
Right side = center (warm).

### Expansion 1: WaveguideNetwork

<details>
<summary>Click to expand: Time-domain physical modeling</summary>

Where `ModalNetwork` decomposes resonance into frequency-domain modes, `WaveguideNetwork`
simulates wave propagation directly through delay lines.

```cpp
auto string = vega.WaveguideNetwork(
    WaveguideNetwork::WaveguideType::STRING, 196.0);
```

A wave circulates through a delay line whose length determines pitch.
A loop filter at the termination simulates frequency-dependent damping: high frequencies
decay faster than low ones, just as on a real string.

`pluck(position, strength)` fills the delay line with a shaped noise burst. Position
determines the spectral content: plucking at 0.5 (center) suppresses even harmonics.
Plucking near 0.0 or 1.0 (bridge) produces a brighter, thinner sound.

`ModalNetwork` and `WaveguideNetwork` are complementary. Modal for percussion, bells,
and resonant objects where you want explicit control over spectral peaks.
Waveguide for strings, tubes, and structures where wave propagation and position matter.

</details>

### Expansion 2: Brownian Timing

<details>
<summary>Click to expand: Stochastic processes as timing sources</summary>

```cpp
auto walker = vega.Random(Kinesis::Stochastic::Algorithm::BROWNIAN);
```

A Brownian random walk generates values that drift continuously. Each sample adds a
small random step to the previous value. The path wanders, but never jumps.

The `Logic` node watches this walk and fires when it crosses a threshold (0.3):

```cpp
auto gate = vega.Logic([state](double input) {
    bool crossed = (state->last_walker < 0.3) && (input >= 0.3);
    state->last_walker = input;
    return crossed;
});
```

Because Brownian motion has no period, the trigger timing is fundamentally irregular.
The walk might cross the threshold twice in quick succession, then wander below it
for a long time. This produces organic, breath-like timing that a periodic oscillator
cannot achieve.

Available algorithms in `Kinesis::Stochastic`:
- `UNIFORM`: flat probability, memoryless
- `NORMAL`: Gaussian distribution, memoryless
- `EXPONENTIAL`: clustered events with long tails
- `BROWNIAN`: random walk, drift-based
- `PERLIN`: coherent noise, spatially continuous
- `GENDY`: Xenakis dynamic stochastic synthesis

Each produces a different temporal character when used as a trigger source.

</details>

### Try It

```cpp
// Tighter Brownian step size: slower drift, longer gaps between events
walker->configure("step_size", 0.005);

// Use Perlin noise instead: smoother, more flowing timing
auto walker = vega.Random(Kinesis::Stochastic::Algorithm::PERLIN);

// Model a tube instead of a string (bidirectional waveguide, odd harmonics)
auto tube = vega.WaveguideNetwork(
    WaveguideNetwork::WaveguideType::TUBE, 196.0);
```

---

## Tutorial: Breathing Vowel

```cpp
void compose() { day1_c_breathing_vowel(); }
```

Run this. You hear a continuous, evolving vowel sound. Not struck, not plucked: a stream
of noise filtered through resonant peaks that shift over time. The rhythm is not periodic.
Events cluster in threes and fours with irregular gaps.

Press 1 through 5 to switch vowel presets (A, E, I, O, U). The formant frequencies jump
immediately. Listen to how the same noise source produces radically different vowel identities
just by changing three filter frequencies.

### Expansion 1: ResonatorNetwork

<details>
<summary>Click to expand: Formant synthesis via parallel bandpass filters</summary>

```cpp
auto vowel = vega.ResonatorNetwork(
    5, ResonatorNetwork::FormantPreset::VOWEL_A);
```

5 second-order IIR bandpass filters in parallel. Each tuned to a formant frequency and
bandwidth (Q factor). When noise passes through them, the resonant peaks shape the
spectrum into something that sounds like a human vowel.

Unlike ModalNetwork (which generates its own sound from excitation energy),
ResonatorNetwork requires an external signal:

```cpp
auto noise = vega.Random();
vowel->set_exciter(noise);
```

Feed white noise and the network becomes a formant synthesizer.
Feed a pitched glottal pulse and it voices a vowel.
Feed any signal and it performs spectral morphing toward the target formant profile.

Each resonator can be individually controlled:

```cpp
vowel->set_frequency(0, 700.0);   // first formant
vowel->set_q(0, 15.0);            // bandwidth of first formant
vowel->set_resonator_gain(2, 0.5); // reduce third formant's contribution
```

</details>

### Expansion 2: Sequential Logic

<details>
<summary>Click to expand: Pattern-dependent triggering</summary>

The breathing rhythm comes from sequential logic: a mode where the Logic node
maintains a history of recent boolean states and evaluates a function over that history.

```cpp
breath->set_sequential_function(
    [](std::span<bool> history) {
        int count = 0;
        for (bool b : history) {
            if (b) ++count;
        }
        return count == 3;
    },
    4);
```

The node remembers its last 4 evaluations. It fires true only when exactly 3 of those 4
were true. This creates a pattern: the trigger cannot fire on consecutive cycles (it needs
at least one false in the window), but it also cannot stay silent for long (it needs 3
trues to accumulate).

The result: irregular clusters with natural pauses. Not random (there is structure).
Not periodic (no fixed interval). The history window creates a self-regulating rhythm.

Compare this to the sine-based trigger in the first example. The sine produces exactly
one event per cycle, always. Sequential logic produces events that depend on their own past.

Other Logic modes:
- `DirectFunction`: stateless, evaluates each sample independently
- `MultiInputFunction`: evaluates multiple inputs simultaneously
- `TemporalFunction`: evaluates based on both value and elapsed time
- `SequentialFunction`: evaluates based on history (used here)

</details>

### Try It

```cpp
// Require 4 of 4 true: very rare events, long silences
breath->set_sequential_function(
    [](std::span<bool> history) {
        return std::ranges::all_of(history, [](bool b) { return b; });
    }, 4);

// Require alternating pattern: true-false-true-false
breath->set_sequential_function(
    [](std::span<bool> history) {
        if (history.size() < 4) return false;
        return history[0] && !history[1] && history[2] && !history[3];
    }, 4);

// Longer history window: slower rhythm, more selective
breath->set_sequential_function(
    [](std::span<bool> history) {
        int count = 0;
        for (bool b : history) { if (b) ++count; }
        return count == 5;
    }, 8);
```

---

## Tutorial: Two Bodies

```cpp
void compose() { day1_d_two_bodies(); }
```

Run this. You hear a string and a bell alternating. The timing is erratic: events arrive
in clusters separated by long pauses. This is exponential distribution at work.
The left half of the visual field shows string energy as particle bursts. The right half
shows bell energy as a spiral. When one body has energy, it tints the other body's color.

Click left side of the window to pluck the string. Click right side to strike the bell.

### Expansion 1: Manual Timing Control

<details>
<summary>Click to expand: Driving nodes outside the audio graph</summary>

In the previous examples, the trigger source (sine, Brownian walker) was connected via `>>`
to a Logic node, and the audio subsystem ticked both automatically at sample rate.

Here, neither the random node nor the logic node is registered with the audio graph.
Instead, a `schedule_metro` at 1-second intervals ticks them manually:

```cpp
schedule_metro(1, [expo_random, trigger]() {
    expo_random->process_sample();
    trigger->process_sample();
});
```

This is explicit temporal control. You decide when these nodes evaluate: once per second,
once per frame, once per mouse click, or on any other schedule you define. The nodes
themselves do not care. They process one sample when you call `process_sample()` and do
nothing otherwise.

The connection between them still works via `set_input_node`:

```cpp
trigger->set_input_node(expo_random);
```

When `trigger->process_sample()` runs, it pulls the latest value from `expo_random`
(which was just ticked on the line above) and evaluates its logic function against it.

There are simpler ways to achieve irregular timing. You could call `excite()` directly
inside the metro callback with a probability check, or use `EventChain` for sequenced
timing. This approach is a demonstration of the fact that nodes are not bound to any
particular subsystem or rate. They are functions you call when you choose to.

</details>

### Expansion 2: Exponential Distribution Character

<details>
<summary>Click to expand: Clustered events with long tails</summary>

```cpp
auto expo_random = vega.Random(Kinesis::Stochastic::Algorithm::EXPONENTIAL);
```

Exponential distribution produces values that cluster near zero with a long tail toward
high values. When the logic node checks whether the output spiked above 0.7, this means:
most ticks produce low values (no event), but occasionally a high value arrives and fires
the trigger. Because the distribution is memoryless, high values can arrive back-to-back
or be separated by long gaps.

This is the temporal character of natural processes: raindrops, nerve firings, geiger
counters. Events are not evenly spaced. They clump and scatter.

Compare the four timing approaches so far:
- **Sine threshold** (First Bell): perfectly periodic, one event per cycle
- **Brownian crossing** (Wandering String): irregular but smooth drift, events
  when the walk crosses a boundary
- **Sequential logic** (Breathing Vowel): pattern-dependent, self-regulating history
- **Manual metro + exponential** (Two Bodies): controlled evaluation rate with
  stochastic event density

Each stochastic algorithm produces a different temporal texture. The synthesis model
(bell, string) provides the sound. The stochastic model provides the rhythm.
They are independent choices. And the evaluation rate is a third, orthogonal choice.

</details>

### Expansion 2: Cross-body Visual Influence

<details>
<summary>Click to expand: Energy sharing between visual representations</summary>

Each body's energy is measured independently:

```cpp
float str_energy = ...rms(string->get_audio_buffer()...)...;
float bell_energy = ...rms(bell->get_audio_buffer()...)...;
```

But the color of one body's visual representation incorporates the other's energy:

```cpp
// String burst color: red channel fixed, green/blue shift with bell energy
.color = glm::vec3(1.0F, 0.4F + bell_energy * 0.5F, 0.2F + bell_energy * 0.3F)

// Bell spiral color: blue channel fixed, red shifts with string energy
.color = glm::vec3(0.3F + str_energy * 0.4F, 0.4F, 1.0F)
```

Neither body "knows" about the other at the audio level. They are independent
synthesis engines. The visual layer reads both and creates a relationship that exists
only in the representation. This is a compositional decision made at the data level:
which numbers influence which other numbers, and how.

</details>

### Try It

```cpp
// Change the manual tick rate: faster = more chances for events
schedule_metro(0.1, [expo_random, trigger]() {
    expo_random->process_sample();
    trigger->process_sample();
});

// Simpler alternative: skip the node machinery entirely, use probability
schedule_metro(0.5, [string, bell, state]() {
    if (get_uniform_random(0.0, 1.0) > 0.6) {
        if (state->string_turn) {
            string->pluck(get_uniform_random(0.1, 0.9), 0.7);
        } else {
            bell->excite(0.7);
        }
        state->string_turn = !state->string_turn;
    }
});

// Both bodies on the same channel (mono, spatial collapse)
auto string = vega.WaveguideNetwork(...)[0] | Audio;
auto bell = vega.ModalNetwork(...)[0] | Audio;

// Map string energy to bell's decay (cross-body parameter influence)
bell->map_parameter("decay", string, MappingMode::BROADCAST);
```

---

## Tutorial: Playground

```cpp
void compose() { day1_e_playground(); }
```

Run this. A coupled 12-mode bell with slow pitch drift. Three visual modes. Three rhythm sources.
Mouse excitation.

**Controls:**
- **1 / 2 / 3**: Switch visual mode (spiral / burst / field)
- **Space**: Cycle rhythm source (slow sine / fast sine / exponential random)
- **M**: Collapse to mono (both channels)
- **S**: Left channel only
- **Mouse click**: Strike the bell at click position (x = position, y = intensity)

This is the full interactive system. Explore combinations. Switch the rhythm to exponential
while in field mode. Switch to fast sine while in burst mode. Strike the bell manually while
the automatic trigger runs. Listen to how the pitch drift modulates the timbre continuously
while the rhythm source controls the event density independently.

### Expansion 1: Mode Coupling

<details>
<summary>Click to expand: Energy transfer between resonant modes</summary>

```cpp
bell->set_coupling_enabled(true);
bell->set_mode_coupling(0, 1, 0.15);
```

Without coupling, each mode decays independently. With coupling enabled, energy from one
mode leaks into its neighbors. The bell's sound evolves after excitation: the initial
spectrum shifts as energy redistributes across modes.

Coupling strength controls how quickly energy transfers. Higher values produce faster
spectral evolution and a more "alive" quality. Lower values keep the modes more independent,
closer to a traditional additive synthesis model.

This has no analog equivalent. In physical instruments, coupling exists but is fixed by
the material. Here you control it as a continuous parameter, or map it to an external signal.

</details>

### Expansion 2: Parameter Mapping

<details>
<summary>Click to expand: Continuous modulation across domains</summary>

```cpp
auto pitch_drift = vega.Sine(0.05, 80.0)[0] | Audio;
pitch_drift->enable_mock_process(true);
bell->map_parameter("frequency", pitch_drift, MappingMode::BROADCAST);
```

`map_parameter` creates a persistent connection: every processing cycle, the bell reads
the drift node's last output and applies it to its fundamental frequency.

`BROADCAST` mode applies one value to all modes simultaneously.
`ONE_TO_ONE` mode (used with network sources) applies per-mode values.

The drift node runs at audio rate but its effect on the bell is perceptual at a much slower
scale (0.05 Hz = 20 second cycle). The bell's spectrum slowly wanders through pitch space.

The same node's output also controls visual hue in the metro callback. One oscillator,
two interpretations: pitch in audio, color in graphics. Not routed. Read.

</details>

---

## Day 1 Summary

Five systems. Three synthesis models. Three timing strategies. Three visual modes.

**Synthesis:**
- `ModalNetwork`: frequency-domain, decaying sinusoidal modes, struck bodies
- `WaveguideNetwork`: time-domain, wave propagation in delay lines, plucked strings
- `ResonatorNetwork`: parallel bandpass filters, formant shaping of external signals

**Timing:**
- Periodic: sine oscillator through threshold logic (predictable, regular)
- Drift: Brownian random walk through crossing detection (irregular, organic)
- Pattern: sequential logic over a history window (self-regulating clusters)
- Manual: `schedule_metro` ticking nodes at a chosen rate, with exponential randomness deciding events

**Logic modes:**
- `THRESHOLD`: simple binary test against a value
- `CUSTOM` lambda: manual crossing detection with state
- `SequentialFunction`: pattern matching over a history window

**Visual strategies:**
- Spiral accumulation: growth and periodic reset
- Burst emission: energy-proportional particle count
- Field diffusion: scattered distribution driven by amplitude

All five examples share one architectural fact:
audio state is just data, visual state is just data, and one reads the other
because they are both vectors of numbers in the same address space.

Tomorrow we go deeper.
