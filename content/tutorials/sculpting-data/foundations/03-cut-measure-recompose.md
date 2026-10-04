---
build:
  list: never
---

### The Next Step

A recording is a sequence in time, and the order it was made in is only one of many. Cut it into hundreds of small grains, measure each grain in whatever way you care about, and the recording becomes a population you can order, thin out, thicken and rebuild.

By hand this is a great deal of arithmetic: windows and hops, a measurement per window, a sort, and the stitching of everything back into audio without clicks. Here you choose three things. How big a grain is, what to measure, and which way to order.

Each block below is a complete `compose()`. A file dialog opens when you run it. Choose a recording a minute or more long if you can, since the effects are about how a whole recording is rearranged.

### Quiet to Loud
```cpp
void compose() {
    auto sound = vega.read_audio();
    const auto grain = Config::get_sample_rate() / 10;

    auto crescendo = Granular::process_to_container(
        sound, AnalysisType::FEATURE,
        Granular::GranularConfig { .grain_size = grain, .hop_size = grain, .feature_key = "rms" },
        "rms");

    crescendo | Audio;
}
```
Run this code. The recording is cut into tenths of a second, each tenth is measured for loudness, and all of them are laid end to end from the quietest to the loudest. A whole recording becomes one long crescendo, and every moment of it is a real moment from the original.

Add `.ascending = false` after `.feature_key = "rms"` and the arc runs from the loudest to the quietest.

### Dull to Bright
```cpp
void compose() {
    auto sound = vega.read_audio();

    auto config = Granular::GranularConfig {
        .grain_size = 4096,
        .hop_size = 1024,
        .feature_key = "brightness",
        .ascending = true,
        .taper = [](std::span<double> grain) { Kinesis::Discrete::apply_hann(grain); }
    };

    auto analyzer = make_persistent_shared<StandardEnergyAnalyzer>(
        EnergyAnalyzerConfig { .window_size = 512, .hop_size = 256, .method = EnergyMethod::SPECTRAL });

    auto measure = Granular::AttributeExecutor([analyzer](std::span<const double> grain, const ExecutionContext&) -> double {
        if (grain.empty()) {
            return 0.0;
        }
        std::vector<Kakshya::DataVariant> input { Kakshya::DataVariant(std::vector<double>(grain.begin(), grain.end())) };
        return extract_scalar_energy(analyzer->analyze_energy(input), "mean_energy");
    });

    auto dawn = Granular::process_to_container(
        sound, measure, config, Granular::GranularOutput::CONTAINER_ADDITIVE);

    dawn | Audio;
}
```
Run this code. This time you wrote the measurement yourself: the spectral energy of each grain. The grains are ordered from the lowest to the highest, and each overlaps the next by three quarters under a soft window, so the joins vanish. What you hear is the recording's timbre brightening from first to last, a colour change with no melody to follow.

### Turbulence
```cpp
void compose() {
    auto sound = vega.read_audio();

    auto cloud = Granular::process_to_container(
        sound, AnalysisType::STATISTICAL,
        Granular::GranularConfig { .grain_size = 1536, .hop_size = 384, .feature_key = "variance", .ascending = false },
        "variance",
        Granular::GranularOutput::CONTAINER_ADDITIVE);

    cloud | Audio;
}
```
Run this code. Each grain is measured by how much its samples vary, and the most restless ones come first. There is no window, and every point is covered by four grains, so the grains pile on top of each other. The result is a dense, saturated cloud that is loudest where the recording was most agitated and settles as the grains calm.

You have, in each case:

- A recording cut into grains, with a number measured for each
- A new order, chosen by that number
- A new recording, built from the old one, that plays through your speakers

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: The Four Steps" >}}

A granular call does four things in a row:

1.  **Segment:** cut the recording into grains of `grain_size` frames, starting a new one every `hop_size` frames
2.  **Attribute:** measure each grain and write the number onto it under the name `feature_key`
3.  **Sort:** order the grains by that number, lowest first or highest first
4.  **Reconstruct:** write the grains out in that order as a new recording

`process_to_container` runs all four. `Granular::process` runs the first three and gives you the ordered list, so you can look at it:
```cpp
void compose() {
    auto sound = vega.read_audio();
    auto config = Granular::GranularConfig { .grain_size = 4800, .hop_size = 4800, .feature_key = "rms" };

    auto grains = Granular::process(sound, AnalysisType::FEATURE, config, "rms");

    for (size_t i = 0; i < 4 && i < grains.data.regions.size(); ++i) {
        auto onset = grains.data.regions[i].start_coordinates[0];
        auto rms = grains.data.regions[i].get_attribute<double>("rms");
        std::cout << "grain at frame " << onset << ", rms " << rms.value_or(-1.0) << "\n";
    }
}
```
This prints the four quietest grains: where each came from in the original, and how loud it was. The ordered list is the whole composition. Reconstruction only reads it out.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: What a Grain Is" >}}

A grain is a region of the recording. It starts at a frame, called its onset, and covers `grain_size` frames across every channel. It carries its measurements as named attributes: the onset, and whatever you measured under `feature_key`.

Segmentation makes whole grains only. The last stretch that is too short for a full grain is dropped. With the default hop of half a grain, every sample lands in two grains. With `hop_size` equal to `grain_size` every sample lands in exactly one.

Grains are measured on one channel, `channel` in the config, which is 0 by default. The grains themselves are moved with all their channels together, so a stereo recording stays stereo.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Choosing What to Measure" >}}

There are two ways to measure.

**A named measurement.** Pass an `AnalysisType` and a name. `AnalysisType::FEATURE` with `"rms"` is the loudness of each grain. `AnalysisType::STATISTICAL` with `"variance"` is how much its samples vary. The number is written on each grain under `feature_key`, and the sort reads that same name.

**Your own measurement.** Pass an `AttributeExecutor`: a function that receives one grain's samples and returns one number. The second block above measures spectral energy. The analyzer is made once, before the grains are measured, with its settings named, and the function reuses it for every grain. Change the method and the order changes:
```cpp
auto analyzer = make_persistent_shared<StandardEnergyAnalyzer>(
    EnergyAnalyzerConfig { .window_size = 512, .hop_size = 256, .method = EnergyMethod::ZERO_CROSSING });
```
`window_size` and `hop_size` are how the analyzer walks along the grain. `extract_scalar_energy(analysis, "mean_energy")` then takes the average over those windows as the single number written on the grain. Other names it understands are `"max_energy"`, `"min_energy"`, `"variance"` and `"dynamic_range"`.
Zero crossings count how often the signal changes sign, a rough measure of how noisy a grain is. `EnergyMethod::RMS` is loudness. Any function you can write over a span of samples will do.

The measure is the whole character of the result. Loudness arranges a recording by dynamics. Spectral energy arranges it by colour. Variance arranges it by agitation. The same recording gives a different piece for each.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Ordering" >}}

`ascending = true` puts the lowest measurement first. `false` puts the highest first. That one switch reverses the whole arc.

The sort runs on the CPU by default. Set `gpu_sort_threshold` to a grain count, and any call with at least that many grains sorts on the GPU instead:
```cpp
Granular::GranularConfig { .grain_size = 512, .hop_size = 256, .feature_key = "rms", .gpu_sort_threshold = 1024 }
```
A minute of audio cut into grains of 256 frames is more than ten thousand of them, which is where the GPU path pays for itself. The default of 0 means never.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Two Ways to Write the Grains Out" >}}

**Concatenative** output (`CONTAINER`, `STREAM`) places the grains end to end, with no overlap. `hop_size` then only decides which parts of the recording are cut, and a hop smaller than the grain repeats material. This is the first block above: tenth-second grains, each used once, in a new order.

**Overlap-add** output (`CONTAINER_ADDITIVE`, `STREAM_ADDITIVE`) places grain number i at i times `hop_size` and adds the grains together where they overlap. The second and third blocks use it. The grain taper decides how the overlaps sound:

- A Hann window, `Kinesis::Discrete::apply_hann`, fades each grain in and out smoothly, so overlaps are seamless
- A trapezoid, `Kinesis::Discrete::apply_trapezoid(grain, grain.size() / 8)`, keeps the body of each grain and fades only the edges, which keeps more of the grain's own character
- No taper at all sums the grains raw. Where they overlap the level builds up, which is what makes the third block a saturated cloud rather than a smooth one

With a hop of a quarter of the grain, as in the second block, each moment of the output is four grains laid over each other.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: A Container, or a Stream" >}}

`process_to_container` returns a `SoundFileContainer`, the same kind of thing `vega.read_audio()` gives you. `| Audio` plays it from start to end, exactly as in the first card.

`process_to_stream` returns a `DynamicSoundStream`, the same kind of thing the samplers of the last card read from. That makes everything from that card available to the result:
```cpp
auto stream = Granular::process_to_stream(
    sound, AnalysisType::FEATURE,
    Granular::GranularConfig { .grain_size = grain, .hop_size = grain, .feature_key = "rms" },
    "rms");

auto sampler = store(create_sampler_from_stream(stream, 0));
sampler->play_continuous(0, sampler->slice_from_stream());
```
`create_sampler_from_stream` takes the stream you already have, so no file path is needed. Loop it, read it at any speed or backwards, or give a tap set several readings at once. The next section does that.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: The Whole Configuration" >}}

Every field of `GranularConfig`, with its default:

- `grain_size = 1024`, frames per grain
- `hop_size = 512`, frames between the starts of grains
- `feature_key = "feature"`, the name the measurement is written under
- `channel = 0`, which channel is measured
- `ascending = true`, the order
- `gpu_sort_threshold = 0`, grain count at which the sort moves to the GPU, 0 for never
- `attribution_context`, left at its default
- `taper`, empty by default, which means no taper

Fields you leave out keep their defaults. Fields you do name have to be written in the order above.

Granular is a workflow, so it is switched on by a line at the top of your `src/user_project.hpp`, above the `#include` of `MayaFlux.hpp`: `#define MAYAFLUX_WORKFLOW_GRANULAR`. Without it `Granular::` does not exist, and a define placed after the include does nothing. The project file already has it.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: Changing the Rules" >}}

The three first steps are rules in a grammar, not fixed code, and each can be replaced on its own while the others carry on. A rule that cuts at every rising zero crossing, instead of at fixed hops, gives grains that begin where the waveform crosses zero. A second measurement rule can add a roughness number to every grain alongside the first, to use later or to order by.

This is how the pieces in the `examples` folder, such as the zero crossing segmentation and the dual feature sort, are made. It is also where the workflow stops being a function call and becomes a place to compose: the segmentation, the measure and the order are the three decisions of a piece, and each can be anything.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: Long Recordings Without Waiting" >}}

Segmenting and measuring a long recording takes time, and a call that returns a recording makes you wait for it. The asynchronous forms run in the background and hand you the result when it is ready, so something else can play meanwhile:
```cpp
void compose() {
    auto sound = vega.read_audio();
    auto window = create_window({ "Granular", 800, 600 });
    window->show();

    auto config = Granular::GranularConfig { .grain_size = 4800, .hop_size = 4800, .feature_key = "rms" };
    auto matrix = Granular::make_granular_matrix(ComputationContext::SPECTRAL, Granular::GranularOutput::STREAM);
    auto sampler = std::make_shared<std::shared_ptr<Kriya::SamplingPipeline>>();

    Granular::process_to_stream_async(
        matrix, sound, AnalysisType::FEATURE,
        [sampler](std::shared_ptr<Kakshya::DynamicSoundStream> stream) {
            if (stream) {
                *sampler = create_sampler_from_stream(stream, 0);
            }
        },
        config, "rms");

    sound | Audio;

    on_key_pressed(window, IO::Keys::Q, [sampler]() {
        if (*sampler) {
            (*sampler)->play(0, (*sampler)->slice_from_stream());
        }
    });
}
```
The original plays at once. When the rearranged version is ready the callback builds a sampler for it, and pressing Q starts it. The callback runs on the worker thread, so it only builds the sampler. Playing is left to the key press.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### Contrary Motion

One ordering, read in both directions at once, so the sound brightens on one side as it dulls on the other:
```cpp
void compose() {
    auto sound = vega.read_audio();
    const auto grain = Config::get_sample_rate() / 10;

    auto stream = Granular::process_to_stream(
        sound, AnalysisType::FEATURE,
        Granular::GranularConfig { .grain_size = grain, .hop_size = grain, .feature_key = "rms" },
        "rms");

    auto taps = create_tap_set_from_stream(stream)
        .tap().speed(1.0).on_channel(0)
        .tap().speed(-1.0).on_channel(1)
        .start();

    store(taps);
}
```
Run it. The left channel plays the crescendo, quiet to loud. The right channel plays the same crescendo backwards, loud to quiet. They cross in the middle, and then loop. Nothing here is a second recording: it is one ordering of grains and two readings of it.

Then change one thing at a time:

- **`.speed(-1.0)` to `.speed(-0.5)`:** the descending side takes twice as long, so the two cross at a different point every time round.
- **`grain`:** `Config::get_sample_rate() / 40` makes the grains a fortieth of a second, and the arc turns to a texture.
- **`"rms"` to `"variance"` with `AnalysisType::STATISTICAL`:** order by agitation instead of loudness.

### The Same Gesture at Three Scales

One crescendo, built three times with grains of different sizes, all sounding together:
```cpp
void compose() {
    auto sound = vega.read_audio();
    const auto rate = Config::get_sample_rate();

    for (const double seconds : { 0.05, 0.2, 0.8 }) {
        const auto grain = static_cast<uint32_t>(rate * seconds);

        auto stream = Granular::process_to_stream(
            sound, AnalysisType::FEATURE,
            Granular::GranularConfig { .grain_size = grain, .hop_size = grain, .feature_key = "rms" },
            "rms");

        store(create_tap_set_from_stream(stream).tap().level(0.5).on_channels({ 0, 1 }).start());
    }
}
```
Run it. Each of the three layers rises from quiet to loud over the whole recording, but built from different pieces. With grains of a fiftieth of a second the rise is a fine, smooth texture. With grains of eight tenths of a second it is a staircase of recognisable fragments. All three rise together and so they never quite agree.

Change the three numbers in `{ 0.05, 0.2, 0.8 }` to move the scales, or add a fourth.

### What You Achieved

You have:

- Cut a recording into grains and measured each one
- Ordered the grains by loudness, by spectral energy and by variance
- Chosen whether grains sit end to end or overlap, and how the overlap is windowed
- Turned the result into a container to play, or a stream to read from many ways
- Built one gesture at several scales, and one ordering that runs both ways

Everything here rearranged a recording that already existed. In the next section the numbers inside the recording become pixels.

{{< /tutorial-detail >}}
