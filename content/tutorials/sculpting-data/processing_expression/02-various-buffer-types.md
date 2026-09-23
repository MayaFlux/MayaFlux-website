---
build:
  list: never
---

#### Buffer Ecosystem Overview

MayaFlux provides specialized buffer types for different generation and processing patterns:

- **NodeBuffer**: Generate audio from mathematical nodes
- **FeedbackBuffer**: Recursive temporal processing with memory
- **SoundStreamWriter**: Capture processed audio to containers

Each buffer type has default processors and specific use cases. Click to explore each in detail.

{{< tutorial-subcard title="Generating from Nodes (NodeBuffer)" >}}

#### The Next Pattern

So far: buffers read from files, nodes affect buffer processing.\
Now: buffers **generate** from nodes.
```cpp
void compose() {
    // Create a sine node
    auto sine = vega.Sine(440.0);

    // Create a NodeBuffer that captures the sine's output
    auto node_buffer = vega.NodeBuffer(0, 512, sine) | Audio[0];

    // Add processing to the generated audio
    auto distortion = vega.Polynomial([](double x) { return x * x * x; });
    MayaFlux::create_processor<PolynomialProcessor>(node_buffer, distortion);
}
```
Run this. You hear a 440 Hz sine wave with cubic distortion.

No file loaded. The buffer **generates** audio by evaluating the node 512 times per cycle.

{{< tutorial-detail title="Expansion 1: What NodeBuffer Does" >}}

`NodeBuffer` connects the **node system** (sample-by-sample evaluation) to the **buffer system** (block-based processing).

**Default processor: `NodeSourceProcessor`**

Each cycle:

1.  Node is evaluated 512 times: `node->process_sample()`
2.  Results fill the buffer
3.  Processing chain runs (your custom processors)
4.  Buffer outputs to speakers

**Why this matters:**

Nodes are mathematical expressions (infinite generators). Buffers are temporal accumulators (finite chunks).

`NodeBuffer` bridges the two: **continuous expression → discrete blocks**.

Without `NodeBuffer`, you'd manually call `node->process_sample()` 512 times and copy results into a buffer. `NodeBuffer` automates this.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: The `clear_before_process` Parameter" >}}

`NodeBuffer` has a flag: `clear_before_process`
```cpp
auto node_buffer = vega.NodeBuffer(0, 512, sine, true);  // Clear first (default)
```
**true (default)**: Buffer is zeroed, then filled with node output

- Result: pure node output

**false**: Node output is **added** to existing buffer content

- Result: node output + previous buffer state

Why use `false`?

- **Layering**: Multiple nodes contributing to the same buffer
- **Feedback**: Previous cycle's output influences current cycle
- **Additive synthesis**: Mix multiple generators

Example (layering):
```cpp
auto sine = vega.Sine(440.0);
auto buffer = vega.NodeBuffer(0, 512, sine, true) | Audio[0];  // First node clears

auto noise = vega.Random();
auto noise_buffer = vega.NodeBuffer(0, 512, noise, false) | Audio[0];  // Adds to sine
```
Result: sine + noise.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: NodeSourceProcessor Mix Parameter" >}}

`NodeSourceProcessor` has a `mix` parameter (default: 0.5):
```cpp
auto processor = std::make_shared<NodeSourceProcessor>(node, 0.7f);
```
**Mix = 0.0**: Preserve existing buffer content (node output ignored)\
**Mix = 0.5**: Equal blend of existing + node output\
**Mix = 1.0**: Replace with node output (existing content overwritten)

This is a **cross-fade** between what's in the buffer and what the node generates.

**Use case**: Smoothly transition between sources, or create feedback loops where node output gradually replaces decaying buffer content.

Most of the time, you'll use the default (1.0 via `clear_before_process=true`). But for creative effects, `mix` is powerful.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Additive synthesis (multiple generators in one buffer)
auto fund = vega.Sine(220.0);
auto harm2 = vega.Sine(440.0);
auto harm3 = vega.Sine(660.0);

auto buffer = vega.NodeBuffer(0, 512, fund, true) | Audio[0];   // First clears
vega.NodeBuffer(0, 512, harm2, false) | Audio[0];  // Adds harmonic 2
vega.NodeBuffer(0, 512, harm3, false) | Audio[0];  // Adds harmonic 3

// Waveshaping a generated tone
auto sine = vega.Sine(110.0);
auto buffer2 = vega.NodeBuffer(0, 512, sine) | Audio[1];
auto waveshape = vega.Polynomial([](double x) { return std::tanh(x * 10.0); });
MayaFlux::create_processor<PolynomialProcessor>(buffer2, waveshape);
```

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="FeedbackBuffer (Recursive Audio)" >}}

#### The Pattern

Buffers that **remember** their previous state.
```cpp
void compose() {
    // FeedbackBuffer: 70% feedback, 512 samples delay
    auto feedback_buf = vega.FeedbackBuffer(0, 512, 0.7f, 512) | Audio[0];

    // Feed an impulse into the buffer to kick-start resonance
    auto impulse = vega.Impulse(2.0);  // 2 Hz pulse train
    vega.NodeBuffer(0, 512, impulse, false) | Audio[0];  // Adds to feedback buffer

    // WARN: Remember to turn OFF after a few seconds as feedback can build up!
}
```
Run this. You hear: repeating echoes, each 70% of the previous amplitude.

The buffer **feeds back into itself**: output becomes input next cycle.

{{< tutorial-detail title="Expansion 1: What FeedbackBuffer Does" >}}

**Default processor: `FeedbackProcessor`**

Each cycle:

1.  Current buffer content: `buffer[n]`
2.  Previous buffer content: `previous_buffer[n-1]`
3.  Output: `buffer[n] + (feedback_amount * previous_buffer[n-1])`
4.  Store output as next cycle's "previous"

This is a **simple delay line** with feedback.

**Parameters:**

- `feedback_amount`: 0.0–1.0 (how much previous state contributes)
- `feed_samples`: Delay length in samples

Example: `FeedbackBuffer(0, 512, 0.7, 512)` creates:

- 512-sample delay (~10.6 ms at 48 kHz)
- 70% feedback (echoes decay to 0.7 → 0.49 → 0.343 → ...)

**Stability:** Keep `feedback_amount < 1.0` or output will grow unbounded.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: FeedbackBuffer Limitations" >}}

`FeedbackBuffer` is deliberately simple. It implements **one specific recursive algorithm**: linear feedback delay.

**Limitations:**

1.  **Fixed feedback coefficient**: Can't modulate feedback amount per sample (it's buffer-wide)
2.  **No filtering in loop**: Can't insert lowpass/highpass in the feedback path
3.  **No cross-channel feedback**: Single-channel only
4.  **No time-varying delay**: Delay length is fixed at creation

**Why these limitations?**

`FeedbackBuffer` is a **building block**, not a complete reverb/delay effect.

For complex feedback systems:

- Use `PolynomialProcessor` in `RECURSIVE` mode (per-sample nonlinear feedback)
- Use `BufferPipeline` to route buffers back to themselves with processing
- Build custom feedback networks with multiple buffers

`FeedbackBuffer` is for **simple echoes and resonances**: quick and efficient.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: When to Use FeedbackBuffer" >}}

**Use `FeedbackBuffer` when:**

- You need a simple delay line with fixed feedback
- Building Karplus-Strong string synthesis
- Creating rhythmic echoes
- Implementing comb filters

**Use `PolynomialProcessor(RECURSIVE)` when:**

- You need nonlinear feedback (saturation, distortion in loop)
- Feedback amount varies per sample
- Building filters with arbitrary feedback functions

**Use `BufferPipeline` when:**

- You need complex routing (buffer A → process → buffer B → back to A)
- Multi-buffer feedback networks
- Cross-channel feedback

**Example: Filtered feedback (requires multiple approaches):**
```cpp
// FeedbackBuffer can't do this alone:
// current + lowpass(feedback * previous)

// Solution: Use PolynomialProcessor RECURSIVE mode with filtering
auto filtered_feedback = vega.Polynomial(
    [](std::span<double> amp; history) {
        double fb = 0.7 * history[0];
        return fb * 0.5 + history[1] * 0.5;  // Simple lowpass
    },
    PolynomialMode::RECURSIVE,
    2
);
```
---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Karplus-Strong string (plucked string synthesis)
auto feedback_buf = vega.FeedbackBuffer(0, 512, 0.996f, 100) | Audio[0];  // ~480 Hz
// Excite with noise burst
auto noise = vega.Random();
vega.NodeBuffer(0, 512, noise, false) | Audio[0];

// Ping-pong delay (two-buffer technique, covered later)
// auto left = vega.FeedbackBuffer(0, 512, 0.6f, 2400) | Audio[0];
// auto right = vega.FeedbackBuffer(1, 512, 0.6f, 2400) | Audio[1];
// Route left → right, right → left (needs BufferPipeline)

// Resonant comb filter
auto feedback_buf2 = vega.FeedbackBuffer(0, 512, 0.95f, 50) | Audio[1];
auto input = vega.Sine(220.0);
vega.NodeBuffer(0, 512, input, false) | Audio[1];
```

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="SoundStreamWriter (Capturing Audio)" >}}

#### The Pattern

Processors that **write** buffer data somewhere (instead of transforming it).
```cpp
void compose() {
    auto sound = vega.read_audio("path/to/file.wav") | Audio;
    auto buffer = get_io_manager::get_audio_buffers(sound)[0];

    // Create a DynamicSoundStream (accumulator for captured audio)
    auto capture_stream = std::make_shared<DynamicSoundStream>(48000, 2);

    // Create a processor that writes buffer data to the stream
    auto writer = std::make_shared<SoundStreamWriter>(capture_stream);

    // Add to buffer's processing chain
    auto chain = buffer->get_processing_chain();
    chain->add_processor(writer);

    // File plays AND is captured to stream simultaneously
}
```
Run this. The file plays **and** is written to `capture_stream` every cycle.

After playback, `capture_stream` contains a copy of the entire file (processed through any other processors in the chain before the writer).

{{< tutorial-detail title="Expansion 1: What SoundStreamWriter Does" >}}

`SoundStreamWriter` is the **inverse** of `SoundContainerBuffer`:

- **SoundContainerBuffer**: reads from container → fills buffer (source)
- **SoundStreamWriter**: reads from buffer → writes to container (sink)

**Each cycle:**

1.  Extract 512 samples from the buffer
2.  Write them to the `DynamicSoundStream` at the current write position
3.  Increment write position by 512

The stream grows dynamically as data arrives. No pre-allocation needed (though you can for performance).

**Use cases:**

- Record processed audio to memory
- Capture intermediate processing stages for analysis
- Build delay lines / loopers
- Create feedback paths (buffer → stream → buffer)

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Channel-Aware Writing" >}}

`SoundStreamWriter` respects buffer channel IDs:
```cpp
auto left_buffer = buffers[0];   // channel 0
auto right_buffer = buffers[1];  // channel 1

auto stream = std::make_shared<DynamicSoundStream>(48000, 2);  // stereo

auto writer_L = std::make_shared<SoundStreamWriter>(stream);
auto writer_R = std::make_shared<SoundStreamWriter>(stream);

// Add to respective buffers
left_buffer->get_processing_chain()->add_processor(writer_L);
right_buffer->get_processing_chain()->add_processor(writer_R);
```
Result: Stereo file captured to stereo stream, with channels preserved.

**Critical:** Buffer's `channel_id` determines which stream channel receives data. Mismatch = warning + skip.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Position Management" >}}

`SoundStreamWriter` tracks where it's writing:
```cpp
writer->set_write_position(0);        // Write from start
writer->set_write_position(48000);    // Write from 1-second mark
writer->reset_position();             // Reset to 0

// Time-based positioning
writer->set_write_position_time(2.5); // Write from 2.5 seconds

uint64_t pos = writer->get_write_position();           // Get current frame position
double time = writer->get_write_position_time();       // Get current time position
```
**Why control position?**

- **Overdubbing**: Write new audio over existing content
- **Looping**: Reset position to create cyclic recording
- **Multi-pass recording**: Capture different takes at different positions

Default behavior: append at end. Position auto-increments.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Circular Mode" >}}

`DynamicSoundStream` can operate in **circular mode**:
```cpp
auto stream = std::make_shared<DynamicSoundStream>(48000, 2);
stream->enable_circular_buffer(48000);  // 1 second capacity

auto writer = std::make_shared<SoundStreamWriter>(stream);
```
**Behavior:**

When write position reaches capacity, it wraps to 0. Old data is overwritten.

**Use cases:**

- **Delay lines**: Fixed-length delays for effects
- **Loopers**: Record N seconds, then loop
- **Rolling analysis**: Keep only the most recent N seconds

Without circular mode, the stream grows unbounded: useful for full recording, but problematic for long-running systems.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It" >}}

```cpp
// Record 5 seconds of audio
auto sound = vega.read_audio("path/to/file.wav") | Audio;
auto buffer = get_io_manager::get_audio_buffers(sound)[0];

auto stream = std::make_shared<DynamicSoundStream>(48000, 1);
stream->ensure_capacity(48000 * 5);  // Pre-allocate 5 seconds

auto writer = std::make_shared<SoundStreamWriter>(stream);
buffer->get_processing_chain()->add_processor(writer, buffer);

// After playback, stream contains the audio
// You can analyze it, write to disk, or feed it back

// Circular delay (1 second)
auto stream2 = std::make_shared<DynamicSoundStream>(48000, 1);
stream2->enable_circular_buffer(48000);  // Loop after 1 second

auto writer2 = std::make_shared<SoundStreamWriter>(stream);
buffer->get_processing_chain()->add_processor(writer, buffer);

// Stream now acts as a 1-second tape loop
```

---

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-detail title="Closing: The Buffer Ecosystem" >}}

You now understand:

**Buffer Types:**

- `AudioBuffer`: Generic accumulator
- `SoundContainerBuffer`: Reads from files/streams (default: `SoundStreamReader`)
- `NodeBuffer`: Generates from nodes (default: `NodeSourceProcessor`)
- `FeedbackBuffer`: Recursive delay (default: `FeedbackProcessor`)

**Processor Types:**

- `PolynomialProcessor`: Waveshaping, filters, recursive math
- `LogicProcessor`: Decisions, gates, triggers
- `SoundStreamWriter`: Capture to containers

**Processing Flow:**
```
Default Processor (acquire/generate data)
    ↓
Processing Chain (transform data)
    ↓
Output (speakers/containers/other buffers)
```
**Next:** Buffer routing, cloning, and supply mechanics: how to send processed buffers to multiple channels and domains.

{{< /tutorial-detail >}}

