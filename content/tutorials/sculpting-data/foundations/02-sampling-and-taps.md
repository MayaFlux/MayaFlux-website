---
build:
  list: never
---

### The Next Step

In the last section a recording played from start to end. Here the same recording becomes something you read from: any region of it, as many times at once as you like, at any speed and in either direction.

A tape machine has one head and a motor. The head sits at one position, the motor moves the tape at one speed, and turning around means stopping the spool. A recording in memory has no head and no motor. It is a list of numbers, and every voice that reads it can have a position that is any function of time. That is what separates the first half of this section, which a patient person with enough machines could do, from the second half, which they could not.

Each block below is a complete `compose()`. A file dialog opens: choose a sound file at least three seconds long. To skip the dialog, give the first argument a path, for example `create_sampler()`.

### A Region of a Sound
```cpp
void compose() {
    const auto rate = Config::get_sample_rate();

    auto sampler = store(create_sampler());

    sampler->play_continuous(0, sampler->slice_from_range(0, rate - 1));
    sampler->play_continuous(1, sampler->slice_from_range(rate, rate * 5 / 2 - 1, 1));
}
```
Run this code. Two loops of the same recording play together: the first second, and the second and a half after it. One is a second long and the other a second and a half, so they drift in and out of step and only line up again every three seconds.

### Taps on a Sound
```cpp
void compose() {
    auto taps = create_tap_set()
        .tap().speed(1.0).on_channel(0)
        .tap().enters_after(2.0).speed(1.5).level(0.8).on_channel(1)
        .tap().enters_after(4.0).speed(2.0).level(0.6).on_channels({ 0, 1 })
        .start();

    store(taps);
}
```
Run this code. The recording plays on the left. Two seconds later a second reading joins on the right, a fifth higher. Two seconds after that a third reading joins on both sides, an octave higher. They are the same samples, read at different speeds, so each is a different pitch and finishes at a different time.

Change `1.5` to `1.001` and run again: two readings that start together and slowly drift apart.

You have, in each case:

- One recording, loaded once and held in memory
- Several independent readings of it, each with its own position
- All of them mixed into your speakers

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: What `create_sampler` Does" >}}

`create_sampler` loads a recording for reading from, and gets a **sampler** ready to play it. Without it:
```cpp
auto stream = get_io_manager()->load_audio_bounded("path/to/your/file.wav", 48000 * 5, true);

auto sampler = std::make_shared<Kriya::SamplingPipeline>(
    stream, *get_buffer_manager(), *get_scheduler(), 0, Config::get_buffer_size());
sampler->build();
```
1.  `load_audio_bounded` reads the file into a `DynamicSoundStream`, resampled to your project's sample rate and laid out so any frame can be reached directly
2.  The `SamplingPipeline` is made with the stream, the engine's managers, an output channel and the block size
3.  `build()` supplies the sampler's buffer to that output channel and starts the pipeline that drives it every audio cycle

The numbers are the ones `create_sampler` uses by default: `48000 * 5` frames, and `true` to cut the file off there. So by default only the first five seconds at 48 kHz are loaded. To read more of the file, ask for more frames: `create_sampler("file.wav", 48000 * 20)`.

A sampler plays one channel of the file. By default that is the channel with the same number as its output channel, which is channel 0 for a stereo file's left side. `create_sampler(path, frames, true, 1)` plays channel 1 on output 1, and `create_samplers(path)` makes one sampler per channel.

`store` keeps the sampler alive. A sampler stops sounding when the last handle to it goes away, and the handle in `compose()` goes away when `compose()` returns.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Streams, and Why Many Voices Can Share One" >}}

In the last card the sound lived in a `SoundFileContainer`: one reader walking through it from start to end. A sampler's sound lives in a `DynamicSoundStream`. It holds the same samples, but any number of voices can read it at once, because each voice keeps its own position and its own slot.

That is why the card can play two loops of one recording together, and why the tap set can read the same samples three ways. Nothing is copied. The recording exists once.

A sampler and a tap set can even share one stream:
```cpp
auto sampler = store(create_sampler());
auto taps = create_tap_set_from_stream(sampler->get_stream()).tap().speed(2.0).start();
store(taps);
```
`create_sampler_from_stream` and `create_tap_set_from_stream` take a stream you already have. The forms that load a file open a dialog, or take a path as their first argument, because they have to load first.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Voices and Slices" >}}

Each voice of a sampler is a **slice**: a region of the stream together with how to play it. `slice_from_range(start, end)` makes one, and the end frame is included.

A slice carries its own settings:

- `looping` and `loop_count`: whether it repeats, and how many times
- `scale`: its level
- `speed`: see the next expansion
- `time_map`: a function from time to position, in the expansion after that
- `source_channel`: which channel of the stream it plays

`play_continuous(0, slice)` is the short form of three steps:
```cpp
auto slice = sampler->slice_from_range(0, rate - 1);
slice.looping = true;
sampler->load(0, slice);
sampler->play(0);
```
1.  `load` puts the slice in voice slot 0 and gives it its own cursor
2.  `play` rewinds that cursor to the start of the region and switches the voice on
3.  Every audio cycle, each voice that is on reads its next block, scales it, and adds it to the sampler's buffer. A voice that is off adds silence

A voice that does not loop switches itself off when it reaches the end of its region. Edits to a playing voice are picked up at the next block: `sampler->slice(0).looping = false;`.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: What `speed` Means on a Voice" >}}

A plain voice reads one block of the recording, then moves its position forward by one block times `speed`. At `speed = 1.0` that is exactly one block, so it plays the recording as it is.

At any other speed it skips or overlaps blocks. Above 1.0 it jumps over part of the recording. Below 1.0 each block starts before the previous one has finished, so stretches are heard twice. What you hear is not a change of pitch but a rhythmic skipping. Speed must also be above zero, so a plain voice cannot play backwards.

If you want the pitch to change, or to read backwards, give the voice a time map instead. That is what the tap set does.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Time Maps" >}}

A **time map** is a function from the seconds since a voice started to a position in the recording, in frames. A voice with a time map no longer steps through blocks. Each audio cycle it works out where the map says it should be at the start and at the end of the block, and reads every frame in between, interpolating between neighbouring frames. Because it reads frames at a different rate than they were recorded, the pitch changes, and a map that runs backwards plays backwards.

A voice at half speed, looping a two second region:
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    const auto rate = Config::get_sample_rate();

    auto sampler = store(create_sampler());
    sampler->play_continuous(0, sampler->slice_from_range(0, rate * 2 - 1)
        .with_time_map(Kinesis::TimeMaps::quadratic(0.0, rate * 0.5, 0.0)));
}
```
The map is `quadratic(start, velocity, acceleration)`: it starts at frame 0 and moves at half the recording's own rate, which is `rate * 0.5` frames per second. The factories in `Kinesis::TimeMaps`:

- `linear(from, to, duration)`: a straight path, then stays
- `quadratic(start, velocity, acceleration)`: constant acceleration, so a speed that rises or falls
- `triangle(lo, hi, velocity, start)`: back and forth between two positions
- `piecewise_linear(points, duration)`: through a list of positions, such as a hand drawn curve
- `exponential(from, to, duration)`: a constant ratio each step, for values that shrink or grow geometrically

A map replaces `speed`. While a voice has a map, `loop_count` is ignored.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: What a Tap Is" >}}

A tap is a voice with a time map that moves at a speed you can change while it plays. Without the builder, one tap looks like this:
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    const auto rate = static_cast<double>(Config::get_sample_rate());

    auto sampler = store(create_sampler());
    auto velocity = std::make_shared<double>(1.5 * rate);

    sampler->play_continuous(0, sampler->slice_from_range(0, static_cast<uint64_t>(rate * 4.0) - 1)
        .with_time_map(Kinesis::TimeMaps::integrated(0.0, velocity)));
}
```
`integrated` adds up a velocity as time passes, and reads that velocity from a value you keep. Write `*velocity = rate * 0.5;` while it plays and the voice slows from where it is, with no jump. Zero holds it still, and a negative number plays it backwards.

`create_tap_set` does this for every tap you declare, and keeps the velocities for you:

- `tap()` begins a new tap, and the calls that follow describe it
- `speed(ratio)`, with a negative ratio meaning backwards, and `backward()` for reading from the end of the region
- `enters_after(seconds)`, `level(gain)`, `on_channel(n)` and `on_channels({ 0, 1 })`
- `region(start_frame, end_frame)`, which applies to every tap, and `loop(false)` to play once

Taps loop by default. After `start()` the set can still be changed:
```cpp
taps.level(1, 0.3);
taps.speed(0, 0.5);
taps.stop();
```
`speed(tap, ratio)` carries on from where the tap is, so it never jumps. `stop()` silences every tap, including ones that have not entered yet.

One sampler is made for each output channel a set uses, and each tap is a voice on it. A mono file plays on whichever output a tap is given.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: Why Speed Is Pitch, and Why 1.001 Drifts" >}}

A tap reads the recording by interpolating between frames, so reading it faster or slower changes the pitch. A speed of 1.5 is a frequency ratio of 3 to 2, a fifth. A speed of 2.0 is an octave.

Two taps at 1.0 and 1.001 read the same samples a thousandth apart in speed. They start together and sound as one. After a second one is a thousandth of a second ahead, and the two together beat and then separate, a slow phase shift between two copies of one sound.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: Adding Voices Makes the Others Quieter" >}}

Every voice and every tap that sounds on an output channel adds into that channel's mix. By default the engine divides the mix by the number of sources feeding it, so a new voice makes the existing ones quieter in return.

To turn that off for the two stereo channels:
```cpp
set_mix_normalization_for_channels({ 0, 1 }, false);
```
With it off the voices add at full level. Lower each voice's `level` to keep the total under control.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: Stutter, a Repeat That Changes Length" >}}

`repeat_every(seconds)` restarts a tap from the start of its region every so many seconds, so a long recording becomes a short loop. Given a time map instead of a number, the length is read at the start of each repeat. A length that shrinks gives a stutter.

Each repeat restarts the tap's own clock, and the cut between repeats is exact to the frame, not the block.
```cpp
.repeat_every(Kinesis::TimeMaps::exponential(0.5, 0.03, 8.0))
```
This repeat starts at half a second and shrinks by the same ratio every step until it is 30 milliseconds, reached after eight seconds. Because each repeat is shorter than the one before, the sound speeds up into a buzz. `linear(0.4, 0.05, 6.0)` shrinks by the same amount every step instead, a more even slowing of the bounces.

A tap with `lag` ignores `repeat_every` and `speed`. A lagged tap trails the write head of a recording that is still being made, as in the last example of this section.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 10: Beyond Tape, Positions Are Functions" >}}

On tape, position follows a motor. One head sits at one place, the speed changes pitch and length together, turning around means stopping and reversing the spool, a canon needs one machine per voice, and an echo needs a second head at a fixed distance. A sound that is still happening cannot be played back until it has finished and been rewound.

In memory there is no head and no motor. A voice with a time map has a position that is whatever function of time you give it, returning a frame number. That is the whole mechanism, and it is why the examples above are possible:

- A sine gives a smooth reversal, with no stop of the spool. The sound slows to a halt at the turn and glides away again
- A list of positions, such as the random walk, gives any path you can compute or measure. The list can come from anywhere
- A different function on each voice gives each voice its own time

You write the function directly, as a `Kinesis::TimeMap`:
```cpp
Kinesis::TimeMap { .fn = [](const double& t) -> double { return 48000.0 * t; } }
```
`t` is the seconds since the voice started and the result is a position in frames. This one is just normal speed at 48 kHz. Every factory in `Kinesis::TimeMaps` is a function like this one.

A tap's map can be replaced while it plays: `taps.slice(0).with_time_map(map)`. Setting a new map restarts that voice's clock at zero.

**The live case.** `create_ring(8.0)` makes a stream that keeps only the most recent eight seconds of whatever is written into it, overwriting the oldest, and `record_into` writes a live signal into it every audio cycle. A tap with `lag` reads that ring a certain distance behind the write head, and because the lag is a time map too, the distance itself can change. A lag that grows is a tap falling behind, so it plays slower and lower. A lag that swings is a pitch that wobbles. A lagged tap stays at least two blocks behind the head, so it never reads what is being written.

`ring->snapshot(frames)` copies the most recent frames into a new stream, and any tap set can read it. That is the fork: a stretch of the past, taken out of the flow and given to a voice that keeps going without it.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### A Bouncing Ball

A recording that bounces, speeds up, and turns to a buzz, with a second reading running backwards behind it:
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    const auto rate = Config::get_sample_rate();

    auto taps = create_tap_set()
        .region(rate / 2, rate * 3 / 2)
        .tap().speed(1.0).on_channel(0)
            .repeat_every(Kinesis::TimeMaps::exponential(0.5, 0.03, 8.0))
        .tap().enters_after(3.0).speed(0.5).backward().level(0.7).on_channel(1)
            .repeat_every(Kinesis::TimeMaps::exponential(0.4, 0.05, 8.0))
        .start();

    store(taps);
}
```
Run it. Both taps read the same one second of the file, from half a second in. The left bounces forward and the right bounces backward at half speed. Each repeat is shorter than the one before, so they both accelerate toward a buzz.

Then change one thing at a time:

- **The three numbers in `exponential`:** start length, end length, and seconds to get there. Try `0.8, 0.01, 20.0` for a slow collapse.
- **`.region(...)`:** move the one second window to another part of the file.
- **`exponential` to `linear`:** the bounces slow evenly rather than by ratio.
- **Delete `.backward()`:** both taps now run forward, at different speeds.

### Beyond the Tape

Everything above a tape music studio could do with enough machines and enough patience. What follows it could not. A motor cannot follow an equation, a spool cannot wander, and a recorder cannot play back a sound that has only just happened and is still going on.

### A Playhead That Obeys an Equation
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    const auto rate = static_cast<double>(Config::get_sample_rate());
    const double centre = rate * 2.0;
    const double reach = rate * 1.5;

    auto taps = create_tap_set(Config::get_sample_rate() * 10)
        .tap().on_channel(0)
        .tap().on_channel(1)
        .start();

    taps.slice(0).with_time_map(Kinesis::TimeMap {
        .fn = [centre, reach](const double& t) -> double { return centre + reach * std::sin(t * 0.50); } });
    taps.slice(1).with_time_map(Kinesis::TimeMap {
        .fn = [centre, reach](const double& t) -> double { return centre + reach * std::sin(t * 0.75); } });

    store(taps);
}
```
Run it. Each channel's playhead swings along a sine curve across the three seconds of the file from half a second to three and a half seconds in. At each end of the swing it slows to a stop, reverses, and speeds up again, so the sound glides to a standstill, turns around, and glides back up through its pitch. The right side swings at three halves the speed of the left, so the two never settle into agreement: they line up again only every twenty-five seconds.

Then change one thing at a time:

- **`0.50` and `0.75`:** how fast each side swings. Try `0.50` and `0.5003` for a slow phase shift.
- **`reach`:** how far it swings. A wider swing is a faster one, and a higher pitch at the centre.
- **The equation itself:** `std::sin(t * 0.5) * std::sin(t * 0.13)` is two swings multiplied, an irregular one that never quite repeats.

### A Drunken Playhead

The same idea, with a path that nobody wrote:
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    const auto rate = static_cast<double>(Config::get_sample_rate());

    std::vector<double> path { rate * 2.0 };
    for (int step = 0; step < 400; ++step) {
        path.push_back(std::clamp(path.back() + get_uniform_random(-0.2, 0.2) * rate, rate * 0.5, rate * 3.5));
    }

    auto taps = create_tap_set(Config::get_sample_rate() * 10).tap().start();
    taps.slice(0).with_time_map(Kinesis::TimeMaps::piecewise_linear(path, 120.0));

    store(taps);
}
```
Run it. The path is a random walk: 400 steps, each moving up to a fifth of a second forward or back from the last, kept between half a second and three and a half seconds. The playhead follows it over two minutes, wandering forward and backward through the recording at speeds that are never the same twice. Run it again and it is a different walk.

### Forking the Past

This one needs a microphone, headphones, and the input switched on as in the first section. A tape machine can play a sound back only after it has finished and been rewound. This reads a sound that is still going on:
```cpp
void compose() {
    auto ring = create_ring(8.0);
    auto recorder = record_into(ring, Kriya::BufferOperation::capture_input_from(get_buffer_manager(), 0));

    auto echoes = create_tap_set_from_stream(ring)
        .tap().lag(0.4)
        .tap().lag(Kinesis::TimeMaps::linear(0.3, 6.0, 40.0)).level(0.6)
        .tap().lag(Kinesis::TimeMaps::triangle(1.0, 5.0, 0.2, 1.0)).level(0.4)
        .start();

    auto forks = std::make_shared<std::vector<Kriya::TapSet>>();

    auto window = create_window({ "Fork the past", 800, 600 });
    window->show();

    on_key_pressed(window, IO::Keys::C, [ring, forks]() {
        auto held = ring->snapshot(static_cast<uint64_t>(3.0 * ring->get_sample_rate()));
        if (held) {
            forks->push_back(create_tap_set_from_stream(held).tap().start());
        }
    });

    store(recorder);
    store(echoes);
    store(forks);
}
```
Run it and speak or play. You hear yourself live, and three readings of your own past trail behind you. One is a short echo, 0.4 seconds behind. One falls further behind as it goes, from 0.3 seconds to 6, so it sounds lower and lower. One swings between 1 and 5 seconds behind, so its pitch wobbles.

Press C. The last three seconds are copied into a new voice that loops on its own, even after you have gone quiet. Press it again for another. The past of a live sound has become material you can keep, and you can fork as many as you like.

### What You Achieved

You have:

- Loaded one recording into a stream that many voices can read at once
- Played regions of it as loops, in step or out of step
- Read it at different speeds and in both directions, with the pitch following the speed
- Made a repeat that shrinks, with a second reading running backward behind it
- Given a playhead an equation and a random walk to follow, instead of a motor
- Read a live sound a moment after it happened, and forked its past into a voice of its own

Everything here treated the recording as something to read. In the next section it is cut into grains, each grain is measured, and the measurements decide the order they play in.

{{< /tutorial-detail >}}
