---
build:
  list: never
---

Every card so far has been imperative. You decide when things run, you hold the state, and you write the callbacks. Here you describe a flow or spatial relationship and the system runs it.

Two systems. `BufferPipeline` for data flows that span more than one buffer cycle.
`Fabric` for spatial entities whose position is the only thing audio and graphics share.

{{< tutorial-subcard title="Pipelines: declared data flows" >}}

#### Tutorial: Live waveshaper, one cycle in one cycle out

``` cpp
void settings() {
    auto& stream = MayaFlux::Config::get_global_stream_info();
    stream.input.enabled  = true;
    stream.input.channels = 1;
}

void compose() {
    auto mic = MayaFlux::create_input_listener_buffer(0, true);

    auto pipeline = MayaFlux::create_buffer_pipeline();
    pipeline->with_strategy(Kriya::ExecutionStrategy::STREAMING);

    pipeline
        >> BufferOperation::capture_from(mic).for_cycles(1)
        >> BufferOperation::modify_buffer(mic,
            [](const std::shared_ptr<Buffers::AudioBuffer>& buf) {
                for (auto& s : buf->get_data()) {
                    s = std::tanh(s * 6.0) * 0.7;
                }
            }).as_streaming();

    pipeline->execute_buffer_rate();
}
```

Run this with a microphone connected. You hear your input through heavy `tanh` saturation. The pipeline captures one buffer cycle from the mic, modifies it in place, and the modified data leaves through the mic buffer's normal routing. No explicit loop, no cycle counter, no callback registered on any node.

Change `6.0` to `1.5` and the saturation softens toward gentle clipping. Change it to `20.0` for hard rectangular clipping. The lambda is the only thing that changes. The pipeline structure stays identical.

#### Tutorial: Mouse-painted accumulating looper

``` cpp
void settings() {
    auto& stream = MayaFlux::Config::get_global_stream_info();
    stream.input.enabled  = true;
    stream.input.channels = 1;
}

void compose() {
    auto window = MayaFlux::create_window({ "Looper", 800, 600 });
    window->show();

    auto mic = MayaFlux::create_input_listener_buffer(0, true);

    const uint32_t buf_size = MayaFlux::get_buffer_manager()
        ->get_buffer_size(Buffers::ProcessingToken::AUDIO_BACKEND);
    const uint64_t loop_frames = static_cast<uint64_t>(buf_size) * 64;

    auto loop = std::make_shared<Kakshya::DynamicSoundStream>(48000, 1);
    loop->set_auto_resize(false);
    loop->ensure_capacity(loop_frames);

    auto layer = MayaFlux::create_sampler_from_stream(loop, 0);
    layer->play_continuous(0, layer->slice_from_stream());
    store(layer);

    auto& brush_pos   = make_persistent(0.5f);
    auto& brush_width = make_persistent(0.5f);
    auto& wipe        = make_persistent(false);

    MayaFlux::on_mouse_move(window, [&brush_pos, &brush_width, window]
    (double x, double y) {
        const auto ndc = normalize_coords(x, y, window);
        brush_pos = ndc.x;
        brush_width = 0.05f + 0.6f * ndc.y;
    });

    MayaFlux::on_key_pressed(window, IO::Keys::Space, [&wipe]() { wipe = true; });

    auto pipeline = MayaFlux::create_buffer_pipeline();
    pipeline->with_strategy(Kriya::ExecutionStrategy::PHASED)
             .capture_timing(Vruta::DelayContext::BUFFER_BASED);

    pipeline
        >> BufferOperation::capture_from(mic).for_cycles(64)
        >> BufferOperation::dispatch_to(
            [loop, loop_frames, &brush_pos, &brush_width, &wipe]
            (Kakshya::DataVariant& data, uint32_t) {
                const auto& fresh = std::get<std::vector<double>>(data);

                std::vector<double> existing(loop_frames, 0.0);
                loop->get_channel_frames(existing, 0, 0);

                if (wipe) {
                    wipe = false;
                    std::ranges::fill(existing, 0.0);
                }

                const size_t n = std::min<size_t>(fresh.size(), loop_frames);
                std::vector<double> mixed(loop_frames, 0.0);
                for (size_t i = 0; i < loop_frames; ++i) {
                    const double pos   = static_cast<double>(i) / loop_frames;
                    const double d     = std::abs(pos - brush_pos);
                    const double brush = std::exp(-(d * d) / (brush_width * brush_width));

                    const double keep = 0.98 - 0.5 * brush;
                    const double prev = existing[i] * keep;
                    const double live = (i < n) ? fresh[i] * brush : 0.0;
                    mixed[i] = std::tanh(prev + live);
                }

                loop->write_frames(
                    std::span<const double>(mixed.data(), mixed.size()), 0, 0);
            });

    pipeline->execute_buffer_rate();
}
```

Run this with a microphone and make some sound. An empty window opens. The loop is a strip of time laid left to right across the window, played back continuously underneath you. Wherever the pointer sits, that region of the loop receives the fresh input painted in while the rest decays. Drag across the window and you smear new sound into the part of the loop the cursor passes over, leaving the rest to fade. Move down for a wider brush, up for a narrow one. Space wipes the loop.

Stop moving and the loop stabilises into whatever you last painted, repeating. Keep painting the same spot and that region thickens toward saturation while the rest thins out. The texture is the record of where your cursor has been.


{{< tutorial-detail title="Deep dive" >}}
A `BufferPipeline` is a chain of operations joined by `>>` that runs on the scheduler. You build the chain once and call `execute_buffer_rate`. From that point the coroutine infrastructure handles every cycle. The two examples differ in one decision: the execution strategy.

`STREAMING` sends each captured cycle straight through the operations that follow it. Latency is one buffer cycle. It is the right choice for live effects where the processed output should appear the moment the input arrives. The waveshaper is exactly this: capture one cycle, modify it in place, done.

`PHASED` accumulates across all the `for_cycles(N)` iterations before any processing operation runs. The looper captures 64 buffer cycles into one block before the dispatch sees it. This is the strategy for anything that needs a span of time at once rather than one cycle: accumulation, reversal, block analysis, anything where the operation needs context beyond a single buffer.


{{< tutorial-detail title="Expansion 1: Why the looper cannot be a node" >}}


A node processes one sample at a time. It has no access to a block of past samples unless it maintains its own ring buffer internally. The looper needs the whole 64-cycle block in hand to paint a brush across it: the brush at the cursor position touches samples thousands of indices apart, and the falloff is computed across the entire block at once.

`PHASED` gives you that block as a single `std::vector<double>`. The dispatch reads the whole loop, mixes the fresh input into the brushed region, applies the decay everywhere, and writes the whole thing back. There is no per-sample formulation of this that produces the same result, because the decay and the brush both depend on a sample's position within the block, not on its value.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: The loop is a stream, the playback reads it, they never coordinate" >}}


`loop` is a `DynamicSoundStream` sized once via `ensure_capacity`. A second sampler, `layer`, plays it on a continuous loop. The pipeline writes new contents into the stream every 64 cycles. The playback reads whatever is currently there on its next pass.

Nothing connects the two. The pipeline does not tell the sampler anything changed. The sampler does not ask the pipeline for data. The stream is the only shared state, and the write and the read meet there. This is the same decoupling the whole card is about: declare two flows against a shared target and let them run at their own rates.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: `dispatch_to` versus `modify_buffer` versus `transform`" >}}


`modify_buffer` attaches a processor to a buffer and mutates it in place during that buffer's normal processing. It is the streaming path: the waveshaper uses it because the modification belongs to the live signal as it flows through.

`dispatch_to` hands the accumulated data to a handler that does whatever it likes with it and produces no routed output of its own. The looper uses it because the endpoint is a write into a stream, not a value flowing to a next operation. The handler is a sink.

`transform` sits between them: it takes the data, returns a new `DataVariant`, and that result flows to the next operation in the chain, typically a `route_to_buffer` or `route_to_container`. Use it when the processed block needs to continue down the pipeline rather than terminate in a side effect.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: `make_persistent` and the painting controls" >}}


`brush_pos`, `brush_width`, and `wipe` are declared with `make_persistent` because the pipeline dispatch and the mouse callbacks both outlive `compose()`. The function returns immediately after building the chain and registering the events, destroying all stack variables. `make_persistent` allocates them with process lifetime and hands back references both sides can hold.

The mouse callback writes the cursor's normalized X into `brush_pos` and maps the Y to a brush width. The dispatch reads both each pass. The window does nothing visual here; it is purely a surface for the pointer to move across, a control space rather than a display. Card 5 covered this pattern: external events write a value, the processing side reads it on its own cadence.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: What `for_cycles` means in each strategy" >}}


On a capture operation, `for_cycles(N)` means the capture executes N times per pipeline cycle. Under `STREAMING` with `for_cycles(1)` that is one capture flowing straight through. Under `PHASED` with `for_cycles(64)` the capture runs 64 times and the results concatenate into one block before the dispatch runs once on the whole thing.

Raising the looper's count from 64 lengthens the loop, lowering it shortens it. At 64 cycles and a typical buffer size the loop is on the order of a second. The brush, the decay, and the playback all scale with it automatically because they are expressed as fractions of the block, not as absolute sample counts.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}

{{< tutorial-subcard title="Fabric: one gesture for sound and light" >}}

#### Tutorial: Fabric, one gesture for sound and light

``` cpp
Vruta::GraphicsRoutine weave_mode(
    Vruta::TaskScheduler&,
    std::shared_ptr<Nodes::GpuSync::PathGeneratorNode> path,
    std::shared_ptr<Nodes::Network::ModalNetwork> modal,
    size_t mode_index,
    glm::vec3 color) {
    auto& p = co_await Kriya::GetGraphicsPromise{};
    glm::vec3 pos(0.0f);

    while (!p.should_terminate) {
        const auto& modes = modal->get_modes();
        if (mode_index < modes.size()) {
            const float a = static_cast<float>(modes[mode_index].amplitude);
            const float angle = static_cast<float>(mode_index) / modes.size()
                * glm::two_pi<float>();
            const glm::vec3 target(
                std::cos(angle) * (0.3f + a * 2.5f),
                std::sin(angle) * (0.3f + a * 2.5f),
                0.0f);
            pos = glm::mix(pos, target, 0.05f);
            path->add_control_point({ pos, color, 1.0f + a * 4.0f });
        }
        co_await Kriya::FrameDelay { .frames_to_wait = 2 };
    }
}

void compose() {
    auto window = MayaFlux::create_window({ "Gesture", 1200, 800 });
    window->show();

    auto modal = vega.ModalNetwork(8, 110.0,
        Nodes::Network::ModalNetwork::Spectrum::INHARMONIC, 8.0) | Audio[{ 0, 1 }];
    modal->set_coupling_enabled(true);

    auto composite = vega.CompositeGeometryBuffer() | Graphics;

    std::vector<std::shared_ptr<Nodes::GpuSync::PathGeneratorNode>> paths;
    for (size_t i = 0; i < 8; ++i) {
        const glm::vec3 color = glm::mix(
            glm::vec3(0.2f, 0.4f, 0.9f),
            glm::vec3(0.9f, 0.3f, 0.2f),
            static_cast<float>(i) / 7.0f);

        auto path = vega.PathGeneratorNode(
            Kinesis::InterpolationMode::CATMULL_ROM, 4, 256) | Graphics;
        path->add_control_point({ glm::vec3(0.0f), color, 1.0f });
        paths.push_back(path);

        composite->add_geometry("m" + std::to_string(i), path,
            Portal::Graphics::PrimitiveTopology::LINE_STRIP,
            { .target_window   = window,
              .vertex_shader   = "line_lit.vert",
              .fragment_shader = "line_lit.frag",
              .geometry_shader = "line_lit.geom" });
    }

    auto& sched = *MayaFlux::get_scheduler();
    auto& evmgr = *MayaFlux::get_event_manager();
    auto fabric = std::make_shared<Nexus::Fabric>(sched, evmgr);

    for (size_t i = 0; i < 8; ++i) {
        auto driver = std::make_shared<Nexus::Emitter>(
            [](const Nexus::InfluenceContext&) {});
        fabric->wire(driver)
            .use([paths, modal, i](Vruta::TaskScheduler& s) -> Vruta::GraphicsRoutine {
                const glm::vec3 c = glm::mix(
                    glm::vec3(0.2f, 0.4f, 0.9f),
                    glm::vec3(0.9f, 0.3f, 0.2f),
                    static_cast<float>(i) / 7.0f);
                return weave_mode(s, paths[i], modal, i, c);
            })
            .finalise();
    }

    auto cursor = std::make_shared<Nexus::Emitter>(
        [modal](const Nexus::InfluenceContext& ctx) {
            const float t = (ctx.position.x + 1.0f) * 0.5f;
            const float strength = (ctx.position.y + 1.0f) * 0.5f;
            modal->excite_at_position(t, strength * 0.6f);
        });

    cursor->set_position(glm::vec3(0.0f));
    cursor->set_color(glm::vec3(1.0f, 0.9f, 0.7f));
    cursor->set_radius(1.5f);
    for (const auto& proc : composite->get_render_processors()) {
        cursor->set_influence_target(proc);
    }

    fabric->wire(cursor).every(1.0 / 60.0).finalise();

    MayaFlux::on_mouse_move(window, [cursor, window](double x, double y) {
        const auto& ws = window->get_state();
        cursor->set_position(glm::vec3(
            static_cast<float>(x / ws.current_width) * 2.0f - 1.0f,
            1.0f - static_cast<float>(y / ws.current_height) * 2.0f,
            0.0f));
    });

    auto commit_loop = [](Vruta::TaskScheduler&,
                          std::shared_ptr<Nexus::Fabric> fab) -> Vruta::GraphicsRoutine {
        auto& p = co_await Kriya::GetGraphicsPromise {};
        while (!p.should_terminate) {
            fab->commit();
            co_await Kriya::FrameDelay { .frames_to_wait = 1 };
        }
    };

    MayaFlux::schedule_task("gesture_commit",
        commit_loop(sched, fabric), false);
}
```

Run this and move the mouse across the window. You strike a resonant body. Where the cursor sits decides which modes ring and how hard: horizontal position sweeps which part of the body is struck, vertical position sets how hard. The eight curves are woven from the network's own mode amplitudes, so the louder a mode rings the further its curve reaches out. And the geometry lights up around the cursor, because the same position that strikes the sound also drives the shader.

One `Emitter` does all of this. Its position is the only input. From that single position the sound is struck, the curves are shaped by what rings, and the light falls where the strike lands. Move the cursor and all three move together because they are the same number.


{{< tutorial-detail title="Deep dive" >}}

{{< tutorial-detail title="Expansion 6: What an Emitter actually is" >}}


An `Emitter` is a point in space with an influence function. It is not a thing you draw. When the `Fabric` commits, it reads the Emitter's position and calls the influence function with that position in an `InfluenceContext`. The cursor Emitter's function takes `ctx.position` and calls `excite_at_position` on the modal network. That is the entire audio side: position in, excitation out.

The position is not a convenience. It is the shared variable. The audio reads it as a strike location. The shader reads it as a light position. The curves read the consequence of the strike. Nothing else connects these three domains. In an analog setup they would be three separate signals you route by hand. Here they are one number, read three ways.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 7: `set_influence_target` and the shader UBO" >}}


`cursor->set_influence_target(proc)` creates a uniform buffer matching the `Influence` block the `line_lit` shaders declare at `set = 1, binding = 0`, registers it on the render processor, and binds it. On every commit, the Emitter's position, color, intensity, and radius are packed into that buffer automatically. No descriptor wiring appears in user code.

The fragment shader reads `position` from that block and lights each fragment by its distance from it. So the cursor's world position, written once per commit, becomes the lit point on screen with no glue between the Emitter and the shader beyond this one call. The loop over `get_render_processors()` binds the same Emitter to all eight curve processors, so every curve is lit by the same gesture.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 8: `weave_mode` draws the sound" >}}


Each of the eight curves is driven by a `GraphicsRoutine` that reads one mode's amplitude every other frame and pushes a control point. The point's distance from center and its thickness both scale with that amplitude. When you strike the body and a mode rings, its curve reaches outward and thickens, then retracts as the mode decays.

The routine is wired through a silent `Emitter` via `.use(...)`, which is how Fabric attaches an arbitrary graphics coroutine to an entity's lifecycle. The Emitter here carries no influence of its own; it exists so the Fabric owns and cancels the coroutine. The curves are not told what the sound is doing. They read the network's mode state directly. The image is a reading of the same physical model the cursor is exciting.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 9: Why commit runs on a GraphicsRoutine" >}}


`fabric->commit()` reads positions, publishes the spatial snapshot, and fires every wired entity, including the influence-target UBO upload, which touches GPU resources. GPU work belongs on the graphics thread. So commit is driven by a `GraphicsRoutine` that resumes once per frame via `FrameDelay{1}`, scheduled with `schedule_task`, not by an audio-rate metro.

This is the same thread boundary Card 5 described from the other side. There, external events on the windowing thread wrote values the audio thread read. Here, a graphics-thread coroutine drives the commit so the UBO upload and any render-sink dispatch happen where the GPU work is safe. The cursor Emitter's audio side, the modal excitation, runs through that same commit but only touches network amplitudes, which is cheap and safe to set from there.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 10: Declaring against a shared target, in both halves of the card" >}}


The two systems look different but share a shape. In the looper, two flows, the pipeline and the playback sampler, meet at one stream and never coordinate. In the Fabric example, three flows, the audio excitation, the curve weaving, and the shader lighting, meet at one position and never coordinate.

Neither half has a controller orchestrating the parts. You declare each flow against the shared thing, a stream or a position, and the result is their superposition. This is the digital counterpart to independent voices in an ensemble, except the voices here run on different substrates, the buffer scheduler and the frame clock, and interact only through the value they read and write in common.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}

{{< /tutorial-subcard >}}
