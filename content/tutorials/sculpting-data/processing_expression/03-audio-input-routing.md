---
build:
  list: never
---

#### Routing Ecosystem Overview

MayaFlux provides sophisticated routing capabilities for capturing, distributing, and cloning audio across multiple channels:

- **Audio Input**: Capture live microphone input with real-time processing
- **Buffer Supply**: Route one buffer to multiple output channels efficiently
- **Buffer Cloning**: Create independent copies for parallel processing

These systems enable complex signal routing, multi-channel processing, and efficient resource utilization. Click to explore each in detail.

{{< tutorial-subcard title="Capturing Audio Input" >}}

#### The Simplest Path

So far: buffers read from files or generate from nodes. Now: capture from your microphone.
```cpp
void settings() {
    auto& stream = MayaFlux::Config::get_global_stream_info();
    stream.input.enabled = true;   // Enable microphone input
    stream.input.channels = 1;      // Mono input
}

void compose() {
    // Create a buffer that listens to microphone channel 0
    auto mic_buffer = MayaFlux::create_input_listener_buffer(0, true);

    // Add processing to the live input
    auto distortion = vega.Polynomial([](double x) { return std::tanh(x * 3.0); });
    MayaFlux::create_processor<PolynomialProcessor>(mic_buffer, distortion);
}
```
Run this. Speak into your microphone. You hear yourself with distortion applied in real-time.

{{< tutorial-detail title="Expansion 1: What `create_input_listener_buffer()` Does" >}}

MayaFlux has a dedicated **input subsystem** parallel to the output system.

**Architecture:**
```
Hardware (Microphone)
    ↓
Audio Driver (RtAudio)
    ↓
BufferManager::process_input()
    ↓
InputAudioBuffer (per input channel)
    ↓
InputAccessProcessor (dispatches to listeners)
    ↓
Your listener buffers
```
When you call `create_input_listener_buffer(channel, add_to_output)`:

1.  Creates a new `AudioBuffer`
2.  Registers it with `InputAudioBuffer[channel]` as a **listener**
3.  If `add_to_output=true`: Also registers it with output channel (so it plays back)

**Each audio cycle:**

- Driver captures microphone data
- `InputAudioBuffer` receives it
- `InputAccessProcessor` **copies** data to all registered listeners
- Your buffer gets fresh input every cycle

**Key insight:** `InputAudioBuffer` is a **hub**. Multiple buffers can listen to the same input channel simultaneously.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Manual Input Registration" >}}

`create_input_listener_buffer()` is convenience. You can do it manually:
```cpp
// Create your own buffer
auto buffer = vega.AudioBuffer() | Audio[0];

// Register it as input listener
MayaFlux::read_from_audio_input(buffer, 0);  // Listen to input channel 0

// Later, stop listening:
MayaFlux::detach_from_audio_input(buffer, 0);
```
**When to use manual registration:**

- You already have a buffer (don't want to create a new one)
- You want to dynamically start/stop listening (e.g., record button)
- You need finer control over buffer lifecycle

**Example: Record button**
```cpp
auto recorder = vega.AudioBuffer() | Audio[0];

// Start recording
MayaFlux::read_from_audio_input(recorder, 0);

// Stop recording (after some time)
MayaFlux::detach_from_audio_input(recorder, 0);
```
The buffer continues to exist and process, but stops receiving new input.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Input Without Playback" >}}

Often you want to **capture** input without **playing** it back:
```cpp
// Create listener but don't add to output
auto mic_capture = MayaFlux::create_input_listener_buffer(0, false);  // false = silent

// Capture to a stream for analysis
auto stream = std::make_shared<DynamicSoundStream>(48000, 1);
auto writer = std::make_shared<SoundStreamWriter>(stream);
mic_capture->get_processing_chain()->add_processor(writer);
```
**Result:** Microphone data is captured to `stream`, but you don't hear it.

**Use cases:**

- Recording without monitoring
- Voice analysis (pitch detection, speech recognition)
- Trigger detection (clap to start/stop)
- Level metering / VU display

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Real-time vocal effects chain
auto mic = MayaFlux::create_input_listener_buffer(0, true);

auto pitch_shift = vega.Polynomial([](double x) { return x * 1.5; });  // Naive pitch shift
auto reverb = vega.FeedbackBuffer(0, 512, 0.3f, 2400);  // Simple reverb
auto gate = vega.Logic(LogicOperator::THRESHOLD, 0.05);  // Noise gate

MayaFlux::create_processor<PolynomialProcessor>(mic, pitch_shift);
MayaFlux::create_processor<LogicProcessor>(mic, gate);

// Record to disk simultaneously
auto stream = std::make_shared<DynamicSoundStream>(48000, 1);
auto writer = std::make_shared<SoundStreamWriter>(stream);
mic->get_processing_chain()->add_processor(writer, mic);
// After session: save 'stream' to file

// Voice-triggered synthesis
auto mic_silent = MayaFlux::create_input_listener_buffer(0, false);
auto trigger = vega.Logic(LogicOperator::EDGE);
trigger->set_edge_detection(EdgeType::RISING, 0.3);
auto trigger_proc = MayaFlux::create_processor<LogicProcessor>(mic_silent, trigger);

// When trigger fires, start a synthesizer...
```

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="Buffer Supply (Routing to Multiple Channels)" >}}

#### The Pattern

One buffer, multiple output channels.
```cpp
void compose() {
    auto sine = vega.Sine(440.0);
    auto buffer = vega.NodeBuffer(0, 512, sine) | Audio[0];  // Registered to channel 0

    // Supply this buffer to channels 1 and 2 as well
    MayaFlux::supply_buffer_to_channels(buffer, {1, 2}, 0.5);  // 50% mix level
}
```
Run this. You hear the same 440 Hz sine on **all three channels** (left, center, right in surround setup).

The buffer processes **once**, but outputs to **three channels**.

{{< tutorial-detail title="Expansion 1: What \"Supply\" Means" >}}

**Registration** (`vega.AudioBuffer() | Audio[0]`):

- Adds buffer as a **child** of `RootAudioBuffer[0]`
- Buffer processes during channel 0's cycle
- Output **accumulates** into channel 0

**Supply** (`supply_buffer_to_channels`):

- Adds buffer's **output** to other channels
- Buffer still processes in its original channel
- Output is **copied** to supplied channels

**Analogy:**

- Registration = "This buffer lives in channel 0"
- Supply = "After processing in channel 0, send copies to channels 1 and 2"

**Architecture:**
```
Buffer processes in channel 0
    ↓
Output goes to RootAudioBuffer[0]
    ↓
MixProcessor copies output to RootAudioBuffer[1]
    ↓
MixProcessor copies output to RootAudioBuffer[2]
```
**Key:** The buffer only processes **once**. Supply is a **routing** operation, not a duplication of processing.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Mix Levels" >}}

The `mix` parameter controls how much of the buffer's output is sent:
```cpp
MayaFlux::supply_buffer_to_channel(buffer, 1, 1.0);  // 100% (unity gain)
MayaFlux::supply_buffer_to_channel(buffer, 2, 0.5);  // 50% (half amplitude)
MayaFlux::supply_buffer_to_channel(buffer, 3, 0.1);  // 10% (quiet)
```
**Use case: Stereo width control**
```cpp
auto mono_source = vega.Sine(440.0);
auto buffer = vega.NodeBuffer(0, 512, mono_source) | Audio[0];

// Send to left (full) and right (half) for asymmetric stereo
MayaFlux::supply_buffer_to_channel(buffer, 0, 1.0);  // Left
MayaFlux::supply_buffer_to_channel(buffer, 1, 0.5);  // Right (quieter)
```
**Use case: Send effects**
```cpp
auto dry = vega.NodeBuffer(0, 512, sine) | Audio[0];  // Dry signal, channel 0

// Send 30% to reverb channel
MayaFlux::supply_buffer_to_channel(dry, 2, 0.3);  // Channel 2 = reverb bus
```
Mix is **additive**. If channel already has content, supply **adds** to it.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Removing Supply" >}}

You can remove supply relationships:
```cpp
auto buffer = vega.NodeBuffer(0, 512, sine) | Audio[0];
MayaFlux::supply_buffer_to_channel(buffer, 1);

// Later: stop sending to channel 1
MayaFlux::remove_supplied_buffer_from_channel(buffer, 1);

// Or remove from multiple channels at once
MayaFlux::remove_supplied_buffer_from_channels(buffer, {1, 2, 3});
```
**Use case: Mute individual sends**

- Buffer still processes
- Output still goes to its registered channel
- Supplied channels no longer receive it

**Use case: Dynamic routing matrices**
```cpp
if (user_pressed_button_A) {
    MayaFlux::supply_buffer_to_channel(buffer, 1);
} else {
    MayaFlux::remove_supplied_buffer_from_channel(buffer, 1);
}
```
---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Quad panning (4-channel surround)
auto source = vega.Sine(220.0);
auto buffer = vega.NodeBuffer(0, 512, source) | Audio[0];

// Distribute to 4 corners with different levels (panning)
MayaFlux::supply_buffer_to_channel(buffer, 0, 0.7);  // Front-left
MayaFlux::supply_buffer_to_channel(buffer, 1, 0.3);  // Front-right
MayaFlux::supply_buffer_to_channel(buffer, 2, 0.2);  // Rear-left
MayaFlux::supply_buffer_to_channel(buffer, 3, 0.1);  // Rear-right

// Send effects architecture
auto guitar = vega.NodeBuffer(0, 512, source) | Audio[0];  // Channel 0 = dry
auto reverb_bus = vega.FeedbackBuffer(1, 512, 0.7f, 4800) | Audio[1];  // Channel 1 = reverb
auto delay_bus = vega.FeedbackBuffer(2, 512, 0.6f, 9600) | Audio[2];   // Channel 2 = delay

MayaFlux::supply_buffer_to_channel(guitar, 1, 0.4);  // 40% to reverb
MayaFlux::supply_buffer_to_channel(guitar, 2, 0.2);  // 20% to delay

// Multi-band processing (split frequency ranges across channels)
// Process each band independently, then sum
```

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="Buffer Cloning" >}}

#### The Pattern

One buffer specification, multiple independent instances.
```cpp
void compose() {
    auto sine = vega.Sine(440.0);
    auto buffer = vega.NodeBuffer(0, 512, sine);  // Don't register yet

    // Clone to channels 0, 1, 2
    MayaFlux::clone_buffer_to_channels(buffer, {0, 1, 2});
}
```
Run this. You hear **three independent sine waves** on three channels.

Each clone processes **independently**. The clones don't share data.

{{< tutorial-detail title="Expansion 1: Clone vs. Supply" >}}

**Supply:**

- One buffer processes **once**
- Output is **copied** to multiple channels
- Processing cost: **1× processing**
- Memory: **One buffer**
- Use when: Same signal needs to go to multiple places

**Clone:**

- Multiple buffers process **independently**
- Each has its own data, state, processing chain
- Processing cost: **N× processing** (N = number of clones)
- Memory: **N buffers**
- Use when: Similar buffers need independent processing

**Example: Supply use case**
```cpp
// One reverb output to stereo speakers
auto reverb = vega.FeedbackBuffer(0, 512, 0.8f, 4800) | Audio[0];
MayaFlux::supply_buffer_to_channel(reverb, 1);  // Copy to right channel
// Cost: 1× reverb processing
```
**Example: Clone use case**
```cpp
// Independent noise generators per channel
auto noise_template = vega.NodeBuffer(0, 512, vega.Random(-1.0, 1.0));
MayaFlux::clone_buffer_to_channels(noise_template, {0, 1, 2, 3});
// Cost: 4× noise processing (each with different random seed/state)
// Result: Decorrelated noise on each channel
```

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Cloning Preserves Structure" >}}

When you clone a buffer, each clone receives:

- **Same buffer type** (NodeBuffer, FeedbackBuffer, etc.)
- **Same default processor** configuration
- **Same processing chain** (all added processors)
- **Independent data** (not shared; each clone has its own samples)
- **Independent state** (feedback buffers have separate history)

**Example: Clone a processed buffer**
```cpp
auto sine = vega.Sine(440.0);
auto buffer = vega.NodeBuffer(0, 512, sine);

// Add processing before cloning
auto distortion = vega.Polynomial([](double x) { return std::tanh(x * 2.0); });
MayaFlux::create_processor<PolynomialProcessor>(buffer, distortion);

// Now clone
MayaFlux::clone_buffer_to_channels(buffer, {0, 1, 2});

// Result: Each channel gets sine → distortion (independently processed)
```
Each clone has its own instance of the distortion processor. They don't share state.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Post-Clone Modification" >}}

After cloning, you can modify individual clones:
```cpp
auto buffer = vega.NodeBuffer(0, 512, vega.Sine(440.0));
// Store cloned buffers for later reference
auto cloned_buffers = MayaFlux::clone_buffer_to_channels(buffer, { 0, 1, 2 });


// Add different processing to each
std::vector<double> coeffs_a_2 = { 0.2, 0.3, 0.2 };
std::vector<double> coeffs_b_2 = { 1.0, -0.7 };
auto filter1 = vega.IIR(coeffs_a_1, coeffs_b_1);
auto filter2 = vega.IIR(coeffs_a_2, coeffs_b_2);

MayaFlux::create_processor<FilterProcessor>(cloned_buffers[0], filter1);
MayaFlux::create_processor<FilterProcessor>(cloned_buffers[1], filter2);

// Now channel 0 has one filter, channel 1 has a different filter
```
**Use case:** Stereo decorrelation (same source, slightly different processing per channel)

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Stereo chorus (cloned with phase offset)
auto lfo = vega.Sine(0.5);  // Slow LFO
auto buffer = vega.NodeBuffer(0, 512, lfo);
MayaFlux::clone_buffer_to_channels(buffer, {0, 1});
// Modify one clone to have phase offset (requires accessing clone directly)

// Multi-channel granular synthesis
auto grain_template = vega.NodeBuffer(0, 512, vega.Random(-0.1, 0.1));
MayaFlux::clone_buffer_to_channels(grain_template, {0, 1, 2, 3, 4, 5, 6, 7});
// Each channel generates independent grains

// Independent feedback loops per channel
auto feedback_template = vega.FeedbackBuffer(0, 512, 0.8f, 1000);
MayaFlux::clone_buffer_to_channels(feedback_template, {0, 1, 2, 3});
// Excite each with different input → 4 independent resonances
```

---

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-detail title="Closing: The Routing Ecosystem" >}}

You now understand:

**Input Capture:**

- `InputAudioBuffer`: Hardware input hub
- `InputAccessProcessor`: Dispatches to listeners
- `create_input_listener_buffer()`: Quick setup
- `read_from_audio_input()` / `detach_from_audio_input()`: Manual control

**Buffer Supply:**

- `supply_buffer_to_channel()`: Route one buffer to multiple outputs
- Mix levels: Control send amounts
- Efficiency: Process once, output many times
- `remove_supplied_buffer_from_channel()`: Dynamic routing changes

**Buffer Cloning:**

- `clone_buffer_to_channels()`: Create independent copies
- Preserves structure: Type, processors, chains
- Independent state: Each clone processes separately
- Post-clone modification: Differentiate behavior after creation

**Mental Model:**
```
Input (Microphone)
    ↓
InputAudioBuffer → Listener buffers (capture)
    ↓
Processing chains (transform)
    ↓
Supply (route to multiple channels)
    OR
Clone (create independent instances)
    ↓
RootAudioBuffer (mix per channel)
    ↓
Output (Speakers)
```
**Next:** BufferPipeline (declarative multi-stage workflows with temporal control)

{{< /tutorial-detail >}}

