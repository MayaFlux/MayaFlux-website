---
build:
  list: never
---

## The Floor of the System

Every scheduling primitive in the previous cards is a coroutine underneath. This card shows what those look like written directly.

{{< tutorial-subcard title="A Phrase in One Function (SoundRoutine)" >}}

A `SoundRoutine` is a C++ coroutine function with return type `Vruta::SoundRoutine`. `co_await GetAudioPromise{}` gives you a reference to the promise that lives in the coroutine frame. `co_await SampleDelay{N}` suspends for exactly N samples and the body carries on from the next line.

The body below is a whole phrase. A steady pulse, a three-note figure with uneven gaps, a fill whose gaps shrink by a fixed ratio, then a rest. Each note is a wait for the gate, then a wait for the remainder. Eight bars on screen jump when their note sounds.

``` cpp

void set_bar(const std::shared_ptr&ltMeshNetwork>& net, size_t i, float h)
{
    const float x = -0.84f + 0.24f * static_cast&ltfloat>(i);
    auto& slot = net-&gtslots()[i];
    slot.local_transform = glm::translate(glm::mat4(1.f), glm::vec3(x, -1.f + h, 0.f)) 
                                         * glm::scale(glm::mat4(1.f), 
                                         glm::vec3(1.f, h, 1.f));
    slot.dirty = true;
}

Vruta::SoundRoutine phrase(
    std::shared_ptr&ltMeshNetwork> bars,
    std::function&ltvoid()> play_kick,
    std::function&ltvoid()> play_hat)
{
    auto& promise = co_await GetAudioPromise {};
    promise.set_state("tempo", 1.0f);
    promise.set_state("bar", uint32_t { 0 });

    auto wait = [&](double beats) {
        const float tempo = *promise.get_state&ltfloat>("tempo");
        return SampleDelay { seconds_to_samples(0.5 * beats / tempo) };
    };
    auto on = [&](size_t i, const std::function&ltvoid()>& play) {
        play();
        set_bar(bars, i, 0.5f);
    };
    auto off = [&](size_t i) { set_bar(bars, i, 0.05f); };

    const double figure[] = { 0.75, 0.75, 0.5 };

    while (true) {
        if (promise.should_terminate)
            break;

        for (size_t i = 0; i < 4; ++i) {
            on(i, play_kick);
            co_await wait(0.15);
            off(i);
            co_await wait(0.85);
        }

        for (size_t i = 0; i < 3; ++i) {
            on(4 + i, play_hat);
            co_await wait(0.1);
            off(4 + i);
            co_await wait(figure[i] - 0.1);
        }

        double gap = 0.5;
        for (size_t i = 0; i < 8; ++i) {
            if (i % 2 == 0)
                on(i, play_kick);
            else
                on(i, play_hat);
            co_await wait(gap * 0.4);
            off(i);
            co_await wait(gap * 0.6);
            gap *= 0.78;
        }

        co_await wait(2.0);

        *promise.get_state&ltuint32_t>("bar") += 1;
    }
}

void compose()
{
    auto window = create_window({ "Phrase", 1200, 800 });

    auto bars = vega.MeshNetwork() | Graphics;

    const std::vector&ltuint32_t> quad = { 0, 1, 2, 2, 3, 0 };
    for (size_t i = 0; i < 8; ++i) {
        const glm::vec3 c = glm::mix(
            glm::vec3(.2f, .5f, .9f), glm::vec3(.9f, .4f, .2f),
            static_cast&ltfloat>(i) / 7.f);
        const float w = 0.08f;
        std::vector&ltMeshVertex> v = {
            { { -w, -1.f, 0.f }, c, 1.f, {}, { 0, 0, 1 }, { 1, 0, 0 } },
            { { w, -1.f, 0.f }, c, 1.f, {}, { 0, 0, 1 }, { 1, 0, 0 } },
            { { w, 1.f, 0.f }, c, 1.f, {}, { 0, 0, 1 }, { 1, 0, 0 } },
            { { -w, 1.f, 0.f }, c, 1.f, {}, { 0, 0, 1 }, { 1, 0, 0 } },
        };
        auto node = std::make_shared&ltMeshWriterNode>(4);
        node-&gtset_mesh(v, quad);
        bars-&gtadd_slot("bar" + std::to_string(i), node);
    }
    for (size_t i = 0; i < 8; ++i) {
        set_bar(bars, i, 0.05f);
    }

    auto buf = vega.MeshNetworkBuffer(bars) | Graphics;
    buf-&gtsetup_rendering({ .target_window = window });
    window-&gtshow();

    auto kick = create_sampler("path/to/kick.wav");
    kick-&gtload(0, kick-&gtslice_from_stream());
    auto hat = create_sampler("path/to/hat.wav");
    hat-&gtload(0, hat-&gtslice_from_stream());

    schedule_task("phrase",
        phrase(bars, [kick] { kick-&gtplay(0); }, [hat] { hat-&gtplay(0); }));

    on_key_pressed(window, IO::Keys::Space, []() {
        if (auto task = get_scheduler()-&gtget_task("phrase")) {
            if (auto* t = task-&gtget_state&ltfloat>("tempo")) {
                *t = std::min(3.0f, *t + 0.1f);
            }
        }
    });

    on_key_pressed(window, IO::Keys::S, []() {
        if (auto task = get_scheduler()-&gtget_task("phrase")) {
            if (auto* t = task-&gtget_state&ltfloat>("tempo")) {
                *t = std::max(0.3f, *t - 0.1f);
            }
        }
    });
}
      
```

Run this. Four even kicks, one bar lighting per kick. Three hats on bars five to seven with a long-long-short spacing. Eight alternating hits that tighten until they blur. Two beats of silence, and the phrase starts over. Press Space and the whole phrase speeds up, mid-section. Press S and it slows.

Change `0.78` to `0.9` and the fill tightens slowly instead of collapsing. Change the figure array to `{ 0.5, 0.5, 0.5 }` and the uneven spacing goes even. Add a fifth section by pasting another loop above the rest: nothing else needs to change.

**The body is the score.** It reads top to bottom in the order the sound happens. There is no step counter and no branching on "which section am I in". The position in the function is that information.


{{< tutorial-detail title="Deep dive" >}}
## Expansion 1: Many Waits, One Body

Click to expand: What a callback cannot do

The phrase above contains about two dozen `co_await` expressions across nine different durations. Every one is a line of source, and the source reads in the order things happen.

A `schedule_metro` callback fires at one fixed interval. To play this phrase from one you would hold a step counter, decide on every tick which section it belongs to, track how much of the current gap has elapsed, and branch. That is a state machine written by hand. A `sequence` is a flat list of delays built before it starts, so it cannot compute the next gap from a value that only exists when the previous note has finished. The fill does exactly that: `gap *= 0.78`.

A routine keeps that state for free:

- Loop counters, the running `gap`, and the current section are locals in the coroutine frame.
- The program counter is the state machine. Being on line 40 means "in the fill".
- `for`, `if` and nested loops wrap waits like any other statement. The delay is evaluated when the wait is reached, so it can read anything, including the tempo that a key press changed a moment ago.

A wait is not a sleep. Nothing blocks. The audio thread carries on with everything else, and the scheduler resumes the body on the sample where the delay ends.

## Expansion 2: What schedule_task Does

Click to expand: From coroutine call to first resume

Calling `phrase(...)` does not schedule anything by itself. The coroutine starts running immediately on the calling thread, up to its first suspension. That first suspension is `co_await GetAudioPromise{}`, which marks the routine as waiting for initialisation and hands you the promise reference.

`schedule_task(name, std::move(routine))` takes the routine by rvalue reference and registers it with the scheduler under that name. The coroutine moves into the scheduler, so you do not keep a handle. Its third parameter, `initialize`, defaults to `false`. New tasks wait in a pending queue that the scheduler drains at the start of its next processing call. On that pass the routine sees its initialisation marker, stamps `next_sample` with the current clock position and resumes the body for the first time. From there the routine is resumed whenever the sample clock reaches `next_sample`.

Every `co_await SampleDelay{N}` adds N to `next_sample` and suspends. The scheduler compares the clock against that number once per sample. The routine resumes on the sample where the clock reaches the target.

The same call has overloads for `GraphicsRoutine` and `FreeRoutine`, so Cards 8b and 8d use it too. `CrossRoutine` has no overload yet and goes through the scheduler directly, as Card 8c shows.

## Expansion 3: Inputs Are Parameters, Not Captures

Click to expand: Why the coroutine is a function and not a lambda

`compose()` returns as soon as it has registered everything, and its locals are gone. The coroutine frame is heap-allocated and outlives it. That difference decides where the routine's inputs may live.

Function *parameters* are copied into the frame. A pointer, a `shared_ptr` or a reference stays valid because the frame owns the copy. Lambda *captures* are different: they live in the closure object, the frame only keeps a pointer to that object, and the closure sat on the stack of `compose()`. A capturing lambda coroutine that is still suspended after `compose()` returns reads freed memory on its next resume.

So each routine in this card is a function whose inputs arrive as parameters. The small lambdas inside `phrase` (`wait`, `on`, `off`) are fine because they are locals of the frame. They are not coroutines and they do not outlive it.

The samplers arrive as `std::function<void()>`. The function parameter is copied into the frame together with the captured `shared_ptr` inside it, so the sampler lives as long as the routine does. A parameter declared `auto` would also work, but it turns the routine into a template, and inside a template a call such as `promise.get_state<float>("tempo")` has to be written `promise.template get_state<float>("tempo")`. Card 8d has one `auto` parameter in a routine that never calls `get_state`, which is why it compiles as written.

## Expansion 4: Promise State Is a Map in the Frame

Click to expand: set_state, get_state, and what is safe from outside

`set_state<T>(key, value)` stores a value in an `unordered_map<std::string, std::any>` inside the coroutine frame. `get_state<T>(key)` returns `T*`, or `nullptr` if the key is missing or the type does not match.

The phrase keeps `"tempo"` there so the outside world can reach it, and `"bar"` as a counter the outside world can read. The frame stays put for the coroutine's lifetime, so a pointer from `get_state` stays valid across suspensions. The key handlers reach the routine with `get_scheduler()->get_task("phrase")`, which returns it by name. Before the scheduler has drained its pending queue that lookup returns `nullptr`, which is why the handlers test it.

**What is safe and what is not:**

- Writing through a pointer to an *existing* key from another thread is a plain float write. Both key handlers do this. The worst outcome is that the coroutine sees the old or the new value on its next `wait`.
- Calling `set_state` with a *new* key from another thread is a race. It can rehash the map while the coroutine holds pointers into it. Create every key you intend to share inside the coroutine before the first delay, as the phrase does.
- `get_task` walks the scheduler's task list without a lock, and the audio thread changes that list when it drains additions and removals. Looking a task up from a key handler is fine in a program that registers its tasks once at start and reads them later. For anything that adds and removes tasks while it runs, share a `shared_ptr`-owned struct with atomic fields instead. Card 8d does this.

State the routine alone touches does not belong in the map at all. `gap` is an ordinary local.

## Expansion 5: Stopping, Cancelling, Restarting

Click to expand: Three ways to end or rewind a routine

- `task->set_should_terminate(true)`: the loop checks the flag at the top and breaks. Cooperative. Nothing is interrupted mid-phrase.
- `MayaFlux::cancel_task("phrase")`: removes the task from the scheduler and sets the terminate flag. The frame is released when the last reference drops.
- `restart_task("phrase")`: sets a `"restart"` state flag to true and resumes the routine. It does not rewind the coroutine. The body has to read that flag and reset its own state, which is what `Kriya::line` does when built restartable.

The flag is only checked at the top of the loop, so a terminate request lets the current phrase finish. Move the check inside the sections to make it leave sooner.

## Expansion 6: Everything Before This Card Is a Routine Like This

Click to expand: The abstraction stack is transparent

- **metro**: one loop with a fixed delay and a callback.
- **Timer**: one delay, one callback, return.
- **sequence / EventChain**: a loop over a vector of `(delay, callback)` pairs.
- **Trigger, Gate, Toggle**: a loop calling `process_sample` on a Logic node every sample and letting the node's own callbacks decide.
- **line**: a loop writing a running value into promise state under `"current_value"`, read from outside through the handle.

Each of them fixes one shape of waiting. A routine is the layer where the shape is yours.

## Expansion 7: Mesh Transforms From the Audio Thread

Click to expand: An honest note about set_bar

`set_bar` runs on the audio thread and writes a 4x4 matrix that the graphics thread reads later. A read that lands in the middle of a write can draw one frame with a mixed transform. For a bar chart that is invisible, and it does not block the audio callback, which is the constraint that matters.

If a value must arrive whole, publish it through an atomic and apply it from a `GraphicsRoutine`. Card 8d shows the pattern.

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="Drawing a Wavetable (GraphicsRoutine)" >}}

A `GraphicsRoutine` is structurally identical to a `SoundRoutine`. The differences are the clock and the thread. `co_await FrameDelay{N}` suspends for N frames of the frame clock, and the routine resumes on the graphics thread.

This one crosses into sound. The routine computes one cycle of a waveform as 480 numbers. Every frame it writes them into a stream that a sampler loops, and draws them as a curve. The picture is the sound. Its body is a script on the frame clock: glide to a shape, hold it, flutter, rest, move to the next.

``` cpp

constexpr size_t kTable = 480;
constexpr double kTau = 6.283185307179586;

double shape_sine(double x)   { return std::sin(kTau * x); }
double shape_saw(double x)    { return 2.0 * x - 1.0; }
double shape_square(double x) { return x < 0.5 ? 0.8 : -0.8; }
double shape_hollow(double x) {
    return 0.6 * std::sin(kTau * 3.0 * x) + 0.4 * std::sin(kTau * 5.0 * x);
}

GraphicsRoutine scribe(
    std::shared_ptr<Kakshya::DynamicSoundStream> stream,
    std::shared_ptr<::PathGeneratorNode> path)
{
    auto& promise = co_await GetGraphicsPromise{};

    using Shape = double (*)(double);
    const Shape shapes[] = { shape_sine, shape_saw, shape_square, shape_hollow };

    std::vector<double> table(kTable, 0.0);
    std::vector<double> out(kTable, 0.0);

    auto publish = [&](double gain) {
        for (size_t i = 0; i < kTable; ++i) {
            out[i] = table[i] * gain * 0.4;
        }
        stream->write_frames(std::span<const double>(out));

        path->clear_path();
        for (size_t i = 0; i < kTable; i += 5) {
            const float x = -0.9f + 1.8f * static_cast<float>(i) / static_cast<float>(kTable);
            const float y = static_cast<float>(table[i] * gain) * 0.6f;
            path->add_control_point({ glm::vec3(x, y, 0.f), glm::vec3(.4f, .8f, 1.f), 3.f });
        }
    };

    while (true) {
        for (const Shape shape : shapes) {
            if (promise.should_terminate) co_return;

            for (int frame = 0; frame < 90; ++frame) {
                for (size_t i = 0; i < kTable; ++i) {
                    const double target = shape(static_cast<double>(i) / kTable);
                    table[i] += (target - table[i]) * 0.06;
                }
                publish(1.0);
                co_await FrameDelay{ 1 };
            }

            co_await FrameDelay{ 150 };

            for (int k = 0; k < 6; ++k) {
                publish(k % 2 == 0 ? 0.25 : 1.0);
                co_await FrameDelay{ 4 };
            }

            publish(1.0);
            co_await FrameDelay{ 60 };
        }
    }
}

void compose() {
    auto window = create_window({ "Wavetable", 1200, 800 });

    auto path = vega.PathGeneratorNode(
        Kinesis::InterpolationMode::CATMULL_ROM, 4, 128) | Graphics;
    auto buffer = vega.GeometryBuffer(path) | Graphics;
    buffer->setup_rendering({
        .target_window = window,
        .topology = Portal::Graphics::PrimitiveTopology::LINE_STRIP });
    window->show();

    auto stream = std::make_shared<Kakshya::DynamicSoundStream>(48000, 1);
    std::vector<double> seed(kTable);
    for (size_t i = 0; i < kTable; ++i) {
        seed[i] = 0.4 * shape_sine(static_cast<double>(i) / kTable);
    }
    stream->write_frames(std::span<const double>(seed));

    auto layer = create_sampler_from_stream(stream, 0);
    layer->play_continuous(0, layer->slice_from_stream());
    store(layer);

    schedule_task("scribe", scribe(stream, path));
}
      
```

Run this. A low hum starts. On screen a curve rises out of a flat line and settles into one sine cycle. Then it slides toward a saw and the hum brightens as the corner sharpens. It holds, flutters in level four times, rests, and moves on to a square, then a hollow two-harmonic shape, then back to the sine.

Change `0.06` to `0.01` and the glide takes seconds: the timbre slowly opens instead of stepping. Change `FrameDelay{ 1 }` in the glide to `FrameDelay{ 8 }` and you hear the table update as a zipper, because you have set the update rate. Change `kTable` to `240` and the pitch doubles.

**The frame clock is writing audio.** Nothing in this routine touches the audio thread. It writes numbers into a container, and the audio thread reads that container on its own schedule.


{{< tutorial-detail title="Deep dive" >}}
## Expansion 1: The Picture Is the Sound

Click to expand: One table, two readers

`table` holds one cycle of the waveform. `publish` reads it twice. Once to fill `out` for the audio side, with a gain and a safety scale of 0.4. Once to place 96 control points for the curve, taking every fifth of the 480 values.

The sampler loops the stream, so the stream's length is the wave's period. 480 samples at 48 kHz is 100 Hz. The curve you see is exactly the shape of that one period, which is why the timbre follows the picture.

The seed write in `compose()` comes before the sampler is created. The sampler slices the stream as it stands when you ask, and an empty stream slices to nothing.

## Expansion 2: A Graphics Thread Writing What the Audio Thread Reads

Click to expand: How the container makes this safe

`DynamicSoundStream` guards its data with a sequence lock. A write takes the write side. A read that overlaps a write notices and retries. So the audio thread never takes a mutex, and a half-written table is not what it plays.

"Graphics" in `GraphicsRoutine` names a clock and a thread. It does not name a GPU boundary. Writing samples from it is ordinary programming, and the one constraint in the whole design is that nothing blocks the audio callback.

The write is 480 doubles. It lands once per frame, so a table that changes smoothly is heard as a smooth change. A table that jumps is heard as a step, which is the zipper from the `FrameDelay{ 8 }` experiment.

## Expansion 3: Waits Compose Like Code

Click to expand: Four different delays in one loop

The body waits in four different ways per shape:

- `FrameDelay{ 1 }` ninety times: the glide, one step per frame.
- `FrameDelay{ 150 }` once: the hold, two and a half seconds.
- `FrameDelay{ 4 }` six times: the flutter, fifteen updates a second.
- `FrameDelay{ 60 }` once: the rest before the next shape.

Nothing schedules these against each other. Each wait is a line, and the next line runs when it ends. A gate over a sample loop, a hold, a fast gating figure and a rest would each be a separate object in a callback-based design. Here they are consecutive statements, and the loop around them makes the whole thing repeat.

## Expansion 4: Locals Are State

Click to expand: What does not need the promise map

`table`, `out` and the shape list are locals of the coroutine. They live in the frame, and only this routine touches them, so there is no reason to route them through `set_state`. The promise map is for values that outside code needs to reach. Everything else can be a variable.

The only object shared with another thread is the stream, and the container owns its own synchronisation.

## Expansion 5: FrameDelay Counts Frames, Not Seconds

Click to expand: Converting between domains

`FrameDelay{N}` is a count of frame-clock ticks. If you want seconds, convert with the frame rate: `sched.seconds_to_units(0.25, Vruta::ProcessingToken::FRAME_ACCURATE)`. `seconds_to_samples` converts against the audio rate, so it gives the wrong number here.

Time in the scheduler is domain integers with seconds as the bridge between them. Sample counts and frame counts never mix directly. Card 8c is the case where one suspension needs both.

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="A Score Across Two Clocks (CrossRoutine)" >}}

A `SoundRoutine` answers to the sample clock and a `GraphicsRoutine` answers to the frame clock. A `CrossRoutine` answers to both. `co_await MultiRateDelay{ samples, frames }` arms the clocks you give a nonzero count, and the routine resumes only after every armed clock has reached its target. A zero on one axis disarms that clock for the suspension.

That gives one body three kinds of wait. Frames only: the code after it runs on the graphics thread. Samples only: it runs on the audio thread. Both: it runs when the slower clock arrives. The score below uses all three. A spiral is drawn one point per frame. A half-second barrier holds until both clocks have passed it. Then a roll of kicks accelerates on the sample clock. `CrossRoutine` has no `schedule_task` overload yet, so it is wrapped in a `shared_ptr` and handed to the scheduler directly.

``` cpp

void align(cross_promise& promise, TaskScheduler& sched) {
    promise.next_sample.store(
        sched.current_units(ProcessingToken::SAMPLE_ACCURATE),
        std::memory_order_release);
    promise.next_frame.store(
        sched.current_units(ProcessingToken::FRAME_ACCURATE),
        std::memory_order_release);
}

CrossRoutine score(
    TaskScheduler& sched,
    std::shared_ptr<::PathGeneratorNode> path,
    std::function<void()> play_kick)
{
    auto& promise = co_await GetCrossPromise{};

    const double gaps[] = { 0.30, 0.22, 0.16, 0.11, 0.08, 0.06 };

    while (true) {
        if (promise.should_terminate) break;

        align(promise, sched);
        co_await MultiRateDelay{ 0, 1 };

        path->clear_path();
        for (int i = 0; i < 90; ++i) {
            const float t = static_cast<float>(i) / 89.f;
            const float angle = 0.21f * static_cast<float>(i);
            const float radius = 0.06f + 0.009f * static_cast<float>(i);
            path->add_control_point({
                glm::vec3(std::cos(angle) * radius, std::sin(angle) * radius, 0.f),
                glm::mix(glm::vec3(.2f, .6f, 1.f), glm::vec3(1.f, .4f, .3f), t),
                2.f + 3.f * t });
            co_await MultiRateDelay{ 0, 1 };
        }

        align(promise, sched);
        co_await MultiRateDelay{ seconds_to_samples(0.5), 30 };

        align(promise, sched);
        co_await MultiRateDelay{ 1, 0 };

        for (const double gap : gaps) {
            play_kick();
            co_await MultiRateDelay{ seconds_to_samples(gap), 0 };
        }
    }
}

void compose() {
    auto window = create_window({ "Score", 1200, 800 });

    auto path = vega.PathGeneratorNode(
        Kinesis::InterpolationMode::CATMULL_ROM, 6, 60) | Graphics;
    auto buffer = vega.GeometryBuffer(path) | Graphics;
    buffer->setup_rendering({
        .target_window = window,
        .topology = Portal::Graphics::PrimitiveTopology::LINE_STRIP });
    window->show();

    auto kick = create_sampler("path/to/kick.wav");
    kick->load(0, kick->slice_from_stream());

    auto routine = std::make_shared<CrossRoutine>(
        score(*get_scheduler(), path, [kick] { kick->play(0); }));
    get_scheduler()->add_task(routine, "score");
}
      
```

Run this. A spiral unwinds from the centre, one point per frame, blue shifting to orange. Only the newest sixty points survive, so the curve is a comet that chases its own tail. When the last point lands there is half a second of nothing. Then six kicks arrive with shrinking gaps, 300 ms down to 60 ms, and the spiral starts over.

Change the `30` in the barrier to `60` and the rest doubles: the frame clock is now the slower of the two. Change `seconds_to_samples(0.5)` to `seconds_to_samples(2.0)` and the sample clock governs instead. Delete the two `align` calls before the roll and the kicks bunch together into one burst.

**This routine hops threads by choosing its waits.** The spiral points are added on the graphics thread and the kicks are fired on the audio thread, from one function, with no queue between them.


{{< tutorial-detail title="Deep dive" >}}
## Expansion 1: Arming Decides the Thread

Click to expand: Who resumes you

A cross routine lives in one task list. Both the sample-clock pump on the audio thread and the frame-clock pump on the graphics thread scan it, and each offers the routine a resume with its own context. What each pump can do with that offer depends on which clocks the current wait armed:

| Suspension | Armed clocks | Body resumes on |
|----|----|----|
| `MultiRateDelay{ 0, 1 }` | frame only | the graphics thread |
| `MultiRateDelay{ 24000, 0 }` | sample only | the audio thread |
| `MultiRateDelay{ 24000, 30 }` | sample and frame | whichever clock arrives last |
| `MultiRateDelay{ 0, 0 }` | none | does not suspend |

A pump that has nothing armed on its clock cannot satisfy the wait, so it cannot be the thread that resumes. That is why the spiral, which waits on frames only, runs entirely on the graphics thread, and the roll, which waits on samples only, runs entirely on the audio thread.

Two spots have no guaranteed thread. The code before the first wait runs on whichever pump initialised the routine, so the score does nothing thread-sensitive there. And the code right after the barrier runs on whichever clock was slower, so the score spends the barrier's landing on a one-sample wait to get back onto the audio thread before firing a kick.

## Expansion 2: All Armed Clocks, Then One Resume

Click to expand: What the gate does

On each offer the routine:

1.  Marks the offering clock satisfied if the position has reached the target and that clock was armed.
2.  Returns without resuming if any armed clock is still unsatisfied.
3.  Otherwise performs a compare-exchange from `MULTIPLE` to `NONE` on its delay context. Only the thread whose exchange succeeds resumes the handle. If both pumps arrive at the same moment, one wins and the other backs off.
4.  Clears both satisfaction flags before resuming, so the next suspension starts clean.

## Expansion 3: Unarmed Clocks Fall Behind

Click to expand: Why the score calls align

Each wait adds its counts to two stored targets, `next_sample` and `next_frame`. Nothing pulls a target up to the present time. A clock that a wait leaves unarmed keeps its old target while real time moves on.

In the score, the spiral arms only frames, so for a second and a half `next_sample` stands still. When the roll then waits `{ 14400, 0 }`, its target is the stale value plus 14400, which is already in the past. Every wait is satisfied on the spot and the kicks fire back to back. The same thing happens the other way round when the roll hands back to the spiral.

The fix is the `align` helper. It stores the current position of each clock into the two targets. Call it before any phase that leaves the axis it last used. The promise fields are public atomics, so it is safe to write them while the routine is running: no pump reads them until the next suspension.

The same mistake shows up at registration. A cross routine is initialised by whichever pump sees it first, and that pump seeds both targets from its own clock. The sample clock counts far faster than the frame clock, so a routine that an audio pump initialises late in a session can start with a frame target thousands of frames ahead and wait effectively forever. The first `align` in the score overwrites that seed.

## Expansion 4: What the Barrier Is For

Click to expand: A wait neither clock can shorten

`MultiRateDelay{ seconds_to_samples(0.5), 30 }` is half a second on both clocks. While both run at their normal rates the two targets arrive together and the wait behaves like any half-second rest.

The difference appears when one clock is late. If the graphics thread stalls for a moment, the routine holds until the frame clock has actually reached its target, and the roll starts after the picture has caught up. If audio dropouts hold back the sample clock, the roll waits for those too. Neither domain runs ahead of the other across a barrier. A `SoundRoutine` cannot make that promise about frames, and a `GraphicsRoutine` cannot make it about samples.

## Expansion 5: Fabric Handles This for You

Click to expand: The .use(CrossFactory) path

When a cross routine needs to be owned and cancelled with a Nexus entity, pass a factory that returns a `Vruta::CrossRoutine` to `Fabric`'s wiring `.use(...)`. Fabric registers it with the scheduler and cancels it when the entity is released, so you do not call `add_task` or `cancel_task` yourself. This is the same mechanism Card 7 used for graphics routines, extended to the third type.

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="Life Off the Clocks (FreeRoutine)" >}}

A `FreeRoutine` has no clock. It suspends on `co_await ConditionAwaiter{ predicate }` and a dedicated scheduler thread evaluates the predicate over and over. The moment it returns true, the routine resumes on that thread.

That makes it the place for computation whose pace comes from something other than time. Here it steps Conway's Game of Life on a 48 by 48 grid. Its body waits three times, on three different conditions: not paused, asked for a step, and, if the world has died, told to reseed. A graphics routine draws each generation and asks for the next.

``` cpp

constexpr int kSide = 48;
using Grid = std::array<uint8_t, kSide * kSide>;

struct Life {
    std::array<Grid, 2>   grid {};
    std::atomic<int>      front { 0 };
    std::atomic<bool>     want { false };
    std::atomic<bool>     paused { false };
    std::atomic<bool>     reseed { false };
    std::atomic<uint64_t> generation { 0 };
};

uint32_t randomize(Grid& g, std::mt19937& rng) {
    uint32_t alive = 0;
    for (auto& c : g) {
        c = (rng() % 4) == 0;
        alive += c;
    }
    return alive;
}

uint32_t step(const Grid& in, Grid& out) {
    uint32_t alive = 0;
    for (int y = 0; y < kSide; ++y) {
        for (int x = 0; x < kSide; ++x) {
            int n = 0;
            for (int dy = -1; dy <= 1; ++dy) {
                for (int dx = -1; dx <= 1; ++dx) {
                    if (dx != 0 || dy != 0) {
                        n += in[((y + dy + kSide) % kSide) * kSide + (x + dx + kSide) % kSide];
                    }
                }
            }
            const uint8_t next = in[y * kSide + x] ? (n == 2 || n == 3) : (n == 3);
            out[y * kSide + x] = next;
            alive += next;
        }
    }
    return alive;
}

FreeRoutine evolve(std::shared_ptr<Life> life) {
    std::mt19937 rng(7);

    while (true) {
        co_await ConditionAwaiter{ [life] {
            return !life->paused.load(std::memory_order_acquire);
        } };

        co_await ConditionAwaiter{ [life] {
            return life->want.load(std::memory_order_acquire);
        } };

        const int f = life->front.load(std::memory_order_relaxed);
        uint32_t alive = step(life->grid[f], life->grid[1 - f]);

        if (alive == 0) {
            co_await ConditionAwaiter{ [life] {
                return life->reseed.load(std::memory_order_acquire);
            } };
        }
        if (life->reseed.exchange(false, std::memory_order_acq_rel)) {
            alive = randomize(life->grid[1 - f], rng);
        }

        life->front.store(1 - f, std::memory_order_release);
        life->generation.fetch_add(1, std::memory_order_release);
        life->want.store(false, std::memory_order_release);
    }
}

GraphicsRoutine show(std::shared_ptr<Life> life, auto points) {
    auto& promise = co_await GetGraphicsPromise{};
    uint64_t seen = ~uint64_t { 0 };

    while (true) {
        if (promise.should_terminate) break;

        const uint64_t g = life->generation.load(std::memory_order_acquire);
        if (g != seen) {
            seen = g;
            const Grid& cells = life->grid[life->front.load(std::memory_order_acquire)];

            points->clear_points();
            for (int y = 0; y < kSide; ++y) {
                for (int x = 0; x < kSide; ++x) {
                    if (!cells[y * kSide + x]) continue;
                    points->add_point({
                        glm::vec3(-0.95f + 1.9f * static_cast<float>(x) / (kSide - 1),
                                  -0.95f + 1.9f * static_cast<float>(y) / (kSide - 1), 0.f),
                        glm::vec3(.4f, 1.f, .6f),
                        6.f });
                }
            }

            life->want.store(true, std::memory_order_release);
        }

        co_await FrameDelay{ 3 };
    }
}

void compose() {
    auto window = create_window({ "Life", 1000, 1000 });

    auto points = vega.PointCollectionNode() | Graphics;
    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    auto life = std::make_shared<Life>();
    std::mt19937 seed(1);
    randomize(life->grid[0], seed);

    schedule_task("life", evolve(life));
    schedule_task("life_view", show(life, points));

    on_key_pressed(window, IO::Keys::Space, [life]() {
        life->paused.store(!life->paused.load());
    });

    on_key_pressed(window, IO::Keys::R, [life]() {
        life->reseed.store(true);
    });
}
      
```

Run this. A field of green cells churns and settles into blinkers, blocks and gliders, about twenty generations a second. Space freezes it and releases it. R scatters a new random world over the current one.

Change `FrameDelay{ 3 }` to `FrameDelay{ 1 }` and it runs at frame rate. Change `rng() % 4` to `rng() % 2` and the world starts crowded and dies back. Change `kSide` to `128` and the routine does sixteen thousand cells per step on a thread that neither clock owns.

**The step ran on a third thread.** The graphics routine asked and read. The audio thread never knew.


{{< tutorial-detail title="Deep dive" >}}
## Expansion 1: Three Waits, Three Predicates

Click to expand: A body that waits on different things in turn

Each iteration of `evolve` passes through up to three waits:

1.  `!paused`: block here while the world is frozen.
2.  `want`: block until the viewer asks for the next generation.
3.  `reseed`: only if the step produced an empty grid, block until someone says what to do about it.

The predicates share nothing except that each returns a bool. A `SoundRoutine` waits on time and a `GraphicsRoutine` waits on frames. A `FreeRoutine` waits on whatever you can write as a function, and it can wait on something different every line.

When the grid dies, the routine has already computed the empty generation into the back buffer but has not published it. The screen keeps showing the last living generation until R is pressed, then the reseeded grid replaces the empty one. The R key also works on a living world, since the reseed flag is consumed after the extinction check.

## Expansion 2: The Handshake Keeps Two Threads Off One Grid

Click to expand: Double buffering with one request per response

The grid exists twice. `front` names the generation the viewer may read. The free routine only ever writes the other one, and flips `front` as its last act.

The viewer reads only in a frame where `generation` has changed, and it sets `want` only after it has finished reading. So a request always follows a completed read, and a response always follows a completed write. The reader is never looking at a buffer the writer is filling, and there is no lock. The release store on `generation` and the acquire load in the viewer are what carry the finished grid across.

The initial seed in `compose()` is written before either routine is scheduled, so it needs no synchronisation.

## Expansion 3: The Condition Thread Spins

Click to expand: What the cost is and what pays for it

The scheduler starts one dedicated thread the first time a `FreeRoutine` is added. That thread loops without sleeping: it drains pending operations, then for every routine in the list it checks whether the routine is armed and, if so, evaluates the predicate. With nothing to do it yields, but it does not block.

In practice that is one core running near full occupancy for as long as any free routine exists. It is a design choice: resume latency is the time to the next pass, not a timer period or a wake-up.

Two things follow from that.

- **The predicate is the throttle.** A cheap `atomic<bool>` load is nearly free. A predicate that allocates or takes a lock is paid for millions of times a second.
- **The routine body is the hot path.** Whatever you do after the `co_await` runs on that thread while every other free routine waits. Long work delays the others.

## Expansion 4: The First Predicate Can Be True at Start

Click to expand: A start-up detail specific to free routines

Like every routine, calling the coroutine function runs its body immediately up to the first suspension. For sound, graphics and cross routines the first suspension is the promise request, which always suspends. A `FreeRoutine` has no promise request. Its first suspension is whatever your first `co_await ConditionAwaiter` is.

If that predicate is already true, `await_ready` returns true, the coroutine does not suspend, and the body carries on *on the thread that called the function*, which is your `compose()`. In `evolve` the first predicate, `!paused`, is true at start, so the body runs through it on the composing thread and arms on the second wait, `want`. Nothing between them has a side effect, so it is harmless. Anything with side effects before the first false predicate would run on the wrong thread.

## Expansion 5: What Belongs Here

Click to expand: Work with no clock of its own

A free routine suits any iterative process whose pace comes from something other than time. Cellular automata that step when a request arrives. A physics integrator that runs as fast as it can and publishes snapshots. A model that infers when input is ready. A search that continues until a target is met.

What does not belong here is anything that has a natural clock. If it should happen every 500 ms, it is a `SoundRoutine`. If it should happen every frame, it is a `GraphicsRoutine`. The free routine is for what those two cannot express.

## Expansion 6: Cancelling and Ending

Click to expand: Same verbs, one difference

`cancel_task("life")` and `set_should_terminate(true)` work as with the other routines. The pump thread checks `should_terminate` before every resume, so a cancelled routine never resumes again.

Because the body only runs after a predicate is true, a routine parked on `want` never reaches a `should_terminate` check inside your loop. That is fine here, since the scheduler stops resuming it. If you need to run cleanup on exit, add the flag to the predicate: `return want.load() || promise.should_terminate;`, then check it after the `co_await`.

{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

