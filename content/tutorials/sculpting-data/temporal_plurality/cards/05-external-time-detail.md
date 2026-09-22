---
build:
  list: never
---

All previous cards drew time from internal sources: oscillators, counters, coroutines, signal conditions. This card is about input that arrives from outside the processing graph entirely: a key press, a mouse position, an OSC message. None of these has a sample-accurate clock. They arrive when they arrive.

The mechanism is the same regardless of source. External events convert into either window event coroutines or `InputNode` values. From that point the hook vocabulary from the previous cards applies normally.

#### Tutorial: Keyboard events reshaping mesh slot transforms

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "External", 1200, 800 });

    auto make_face = [](glm::vec3 color, glm::vec3 normal, glm::vec3 tangent,
                         std::array<glm::vec3, 4> corners)
        -> std::pair<std::vector<MeshVertex>, std::vector<uint32_t>> {
        std::vector<MeshVertex> verts;
        verts.reserve(4);
        const std::array<glm::vec2, 4> uvs = { glm::vec2{0,0},{1,0},{1,1},{0,1} };
        for (int i = 0; i < 4; ++i)
            verts.push_back({ corners[i], color, 1.f, uvs[i], normal, tangent });
        return { verts, { 0, 1, 2, 2, 3, 0 } };
    };

    constexpr float H = 0.5f;

    struct FaceSpec {
        std::string name;
        glm::vec3 color, normal, tangent;
        std::array<glm::vec3, 4> corners;
    };
    const std::vector<FaceSpec> specs = {
        { "front",  {0.9f,0.3f,0.2f}, { 0, 0, 1}, { 1, 0, 0}, {{{-H,-H, H},{ H,-H, H},{ H, H, H},{-H, H, H}}} },
        { "back",   {0.2f,0.5f,0.9f}, { 0, 0,-1}, {-1, 0, 0}, {{{ H,-H,-H},{-H,-H,-H},{-H, H,-H},{ H, H,-H}}} },
        { "top",    {0.2f,0.9f,0.3f}, { 0, 1, 0}, { 1, 0, 0}, {{{-H, H,-H},{ H, H,-H},{ H, H, H},{-H, H, H}}} },
        { "bottom", {0.9f,0.8f,0.1f}, { 0,-1, 0}, { 1, 0, 0}, {{{-H,-H, H},{ H,-H, H},{ H,-H,-H},{-H,-H,-H}}} },
        { "right",  {0.7f,0.2f,0.9f}, { 1, 0, 0}, { 0, 0,-1}, {{{ H,-H, H},{ H,-H,-H},{ H, H,-H},{ H, H, H}}} },
        { "left",   {0.9f,0.5f,0.1f}, {-1, 0, 0}, { 0, 0, 1}, {{{-H,-H,-H},{-H,-H, H},{-H, H, H},{-H, H,-H}}} },
    };

    auto net = vega.MeshNetwork() | Graphics;

    struct SlotState {
        glm::vec3 normal;
        std::atomic<float> offset   { 0.f };
        std::atomic<float> velocity { 0.f };
        std::atomic<bool>  held     { false };
    };
    auto states = std::make_shared<std::vector<SlotState>>(specs.size());

    for (size_t i = 0; i < specs.size(); ++i) {
        auto [verts, indices] = make_face(
            specs[i].color, specs[i].normal, specs[i].tangent, specs[i].corners);
        auto node = std::make_shared<Nodes::GpuSync::MeshWriterNode>(4);
        node->set_mesh(verts, indices);
        net->add_slot(specs[i].name, node);
        (*states)[i].normal = specs[i].normal;
    }

    auto buf = vega.MeshNetworkBuffer(net) | Graphics;
    buf->setup_rendering({ .target_window = window });
    buf->get_render_processor()->set_view_transform(
        Kinesis::look_at_perspective(
            {2.5f, 2.0f, 3.5f}, {0,0,0},
            glm::radians(50.f), 1200.f/800.f, 0.01f, 1000.f));
    window->show();

    MayaFlux::schedule_metro(1.0 / 60.0, [net, states]() {
        auto& slots = net->slots();
        for (size_t i = 0; i < slots.size(); ++i) {
            auto& s = (*states)[i];
            float v = s.velocity.load();
            float o = s.offset.load();
            if (s.held.load()) v += 0.012f;
            o += v;
            v *= 0.88f;
            o *= 0.94f;
            s.offset.store(o);
            s.velocity.store(v);
            slots[i].local_transform = glm::translate(glm::mat4(1.f), s.normal * o);
            slots[i].dirty = true;
        }
    });

    MayaFlux::on_key_pressed(window,  IO::Keys::I, [states]() { (*states)[0].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::I, [states]() { (*states)[0].held.store(false, std::memory_order_relaxed); });
    MayaFlux::on_key_pressed(window,  IO::Keys::O, [states]() { (*states)[1].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::O, [states]() { (*states)[1].held.store(false, std::memory_order_relaxed); });
    MayaFlux::on_key_pressed(window,  IO::Keys::K, [states]() { (*states)[2].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::K, [states]() { (*states)[2].held.store(false, std::memory_order_relaxed); });
    MayaFlux::on_key_pressed(window,  IO::Keys::J, [states]() { (*states)[3].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::J, [states]() { (*states)[3].held.store(false, std::memory_order_relaxed); });
    MayaFlux::on_key_pressed(window,  IO::Keys::H, [states]() { (*states)[4].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::H, [states]() { (*states)[4].held.store(false, std::memory_order_relaxed); });
    MayaFlux::on_key_pressed(window,  IO::Keys::L, [states]() { (*states)[5].held.store(true,  std::memory_order_relaxed); });
    MayaFlux::on_key_released(window, IO::Keys::L, [states]() { (*states)[5].held.store(false, std::memory_order_relaxed); });

    bind_viewport_preset(window,
        buf->get_render_processor(), ViewportPresetMode::Fly, {}, "cube_keys");
}
```

Run this. Six colored faces sit assembled as a cube. Press I and the front face slides outward along its normal. Hold it and it continues to push. Release and it decays back. H/J/K/L/I/O each control one face independently. Multiple keys held simultaneously push multiple faces outward. The Fly preset binds its own camera keys separately; both sets of bindings coexist and do different things.

#### Tutorial: Mouse position deforming mesh vertices

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "External", 1200, 800 });

    constexpr int   N    = 24;
    constexpr float SIZE = 2.0f;
    constexpr float STEP = SIZE / static_cast<float>(N - 1);

    std::vector<uint32_t> indices;
    for (int row = 0; row < N - 1; ++row)
        for (int col = 0; col < N - 1; ++col) {
            uint32_t tl = row * N + col;
            indices.insert(indices.end(), { tl, tl+N, tl+1, tl+1, tl+N, tl+N+1 });
        }

    std::vector<MeshVertex> verts(N * N);
    for (int row = 0; row < N; ++row)
        for (int col = 0; col < N; ++col) {
            auto& v   = verts[row * N + col];
            v.position = { col * STEP - SIZE * 0.5f, 0.f, row * STEP - SIZE * 0.5f };
            v.color    = { 0.3f, 0.6f, 0.9f };
            v.normal   = { 0.f, 1.f, 0.f };
            v.tangent  = { 1.f, 0.f, 0.f };
            v.uv       = { float(col)/(N-1), float(row)/(N-1) };
            v.weight   = 0.f;
        }

    auto mesh = vega.MeshWriterNode(N * N) | Graphics;
    mesh->set_mesh(verts, indices);

    auto buf = vega.GeometryBuffer(mesh) | Graphics;
    buf->setup_rendering({ .target_window = window });
    buf->get_render_processor()->set_view_transform(
        Kinesis::look_at_perspective(
            {0.f, 3.f, 3.f}, {0.f, 0.f, 0.f},
            glm::radians(50.f), 1200.f/800.f, 0.1f, 100.f));
    window->show();

    auto mouse_x = std::make_shared<std::atomic<float>>(0.f);
    auto mouse_y = std::make_shared<std::atomic<float>>(0.f);

    MayaFlux::on_mouse_move(window, [mouse_x, mouse_y, window](double x, double y) {
        const auto& ws = window->get_state();
        mouse_x->store(float(x / ws.current_width)  * 2.f - 1.f, std::memory_order_relaxed);
        mouse_y->store(float(y / ws.current_height) * 2.f - 1.f, std::memory_order_relaxed);
    });

    MayaFlux::schedule_metro(1.0 / 60.0, [mesh, verts, mouse_x, mouse_y]() mutable {
        const float mx = mouse_x->load(std::memory_order_relaxed);
        const float my = mouse_y->load(std::memory_order_relaxed);
        for (auto& v : verts) {
            const float dx = v.position.x - mx;
            const float dz = v.position.z - my;
            const float y  = std::exp(-(dx*dx + dz*dz) * 2.5f) * 0.6f;
            v.position.y   = y;
            v.weight       = y / 0.6f;
            v.color        = glm::mix(
                glm::vec3(0.1f, 0.3f, 0.8f),
                glm::vec3(0.9f, 0.7f, 0.2f),
                v.weight);
        }
        mesh->set_mesh_vertices(verts);
    });
}
```

Run this. A flat grid deforms under a Gaussian hill centered on the mouse cursor. Move the mouse and the hill follows, coloring from blue to gold at the peak. Change `2.5f` in the exponent to `8.0f` for a sharper, narrower peak. Change `0.6f` for a taller hill.

#### Tutorial: OSC driving mesh slot transforms

OSC example with optional internal sender for testing without hardware

``` cpp

// Not needed if you have an actual OSC hardware sending messages to port 9000
void osc_sender() {
    auto sink = make_persistent_shared<Portal::Network::NetworkSink>(
        Portal::Network::StreamConfig {
            .name      = "osc_loopback",
            .endpoint  = { .address = "127.0.0.1", .port = 9000 },
            .profile   = Portal::Network::StreamProfile::REALTIME_SMALL,
            .transport = Portal::Network::NetworkTransportHint::UDP,
        });
    float t = 0.f;
    schedule_metro(0.05, [&t, sink]() {
        if (!sink->is_open()) return;
        float rotate_val = (std::sin(t * 0.4f)  + 1.f) * 0.5f;
        float scale_val  = (std::sin(t * 0.17f) + 1.f) * 0.5f;
        auto r = Portal::Network::serialize_osc("/rotate", {{ rotate_val }});
        auto s = Portal::Network::serialize_osc("/scale",  {{ scale_val  }});
        sink->send({ r.data(), r.size() });
        sink->send({ s.data(), s.size() });
        t += 0.05f;
    }, "osc_loopback_sender");
}

void settings() {
    auto& cfg       = MayaFlux::Config::get_global_stream_info();
    cfg.osc.enabled = true;
    cfg.osc.port    = 9000;
}

void compose() {
    osc_sender(); // ONLY NEEDED if you do not want to use an actual OSC hardware

    auto window = MayaFlux::create_window({ "External", 1200, 800 });

    auto make_tetra = [](glm::vec3 color)
        -> std::pair<std::vector<MeshVertex>, std::vector<uint32_t>>
    {
        std::vector<MeshVertex> v = {
            {{ 0.f,  0.6f, 0.f }, color, 1.f, {0.5f,1.f}, { 0, 1, 0}, {1,0,0}},
            {{-0.5f,-0.3f,-0.5f}, color, 1.f, {0.f, 0.f}, { 0,-1, 0}, {1,0,0}},
            {{ 0.5f,-0.3f,-0.5f}, color, 1.f, {1.f, 0.f}, { 0,-1, 0}, {1,0,0}},
            {{ 0.f, -0.3f, 0.5f}, color, 1.f, {0.5f,0.f}, { 0,-1, 0}, {1,0,0}},
        };
        return { v, { 0,1,2, 0,2,3, 0,3,1, 1,3,2 } };
    };

    auto net = vega.MeshNetwork() | Graphics;

    for (auto [color, tx] : std::array{
             std::pair{ glm::vec3{0.8f,0.3f,0.2f}, -0.8f },
             std::pair{ glm::vec3{0.2f,0.5f,0.9f},  0.8f } })
    {
        auto [verts, indices] = make_tetra(color);
        auto node = std::make_shared<Nodes::GpuSync::MeshWriterNode>(4);
        node->set_mesh(verts, indices);
        net->add_slot("", node);
        net->get_slot(net->slot_count() - 1).local_transform =
            glm::translate(glm::mat4(1.f), {tx, 0.f, 0.f});
    }

    auto buf = vega.MeshNetworkBuffer(net) | Graphics;
    buf->setup_rendering({ .target_window = window });
    buf->get_render_processor()->set_view_transform(
        Kinesis::look_at_perspective(
            {0.f, 2.f, 4.f}, {0.f, 0.f, 0.f},
            glm::radians(45.f), 1200.f/800.f, 0.1f, 100.f));
    window->show();

    auto rotate_ctrl = vega.read_osc(
        OSCConfig::normalized(0.0, 1.0),
        Core::InputBinding::osc("/rotate"));

    auto scale_ctrl = vega.read_osc(
        OSCConfig::normalized(0.0, 1.0),
        Core::InputBinding::osc("/scale"));

    float angle = 0.f;
    MayaFlux::schedule_metro(1.0 / 60.0, [net, rotate_ctrl, scale_ctrl, &angle]() {
        const float speed = static_cast<float>(rotate_ctrl->get_last_output());
        const float scale = 0.4f + static_cast<float>(scale_ctrl->get_last_output()) * 1.2f;
        angle += speed * 0.08f;

        auto& slot_a = net->get_slot(0);
        slot_a.local_transform =
            glm::translate(glm::mat4(1.f), {-0.8f, 0.f, 0.f}) *
            glm::rotate(glm::mat4(1.f), angle, {0.f, 1.f, 0.f});
        slot_a.dirty = true;

        auto& slot_b = net->get_slot(1);
        slot_b.local_transform =
            glm::translate(glm::mat4(1.f), {0.8f, 0.f, 0.f}) *
            glm::scale(glm::mat4(1.f), glm::vec3(scale));
        slot_b.dirty = true;
    });
}
```

Run this. The left tetrahedron spins at a rate driven by `/rotate`, the right one breathes in scale driven by `/scale`. The loopback sender in `osc_sender()` generates both values internally so no external client is needed. Replace it with any OSC sender targeting port 9000 and the same addresses to control it from a phone, another program, or hardware.

`read_osc` registers the node internally before returning. No `| Audio` pipe is needed. `get_last_output()` on the graphics metro thread is a lock-free read of an atomic written by the OSC receive thread.

------------------------------------------------------------------------


{{< tutorial-detail title="Deep dive" >}}
Key events, mouse movement, and OSC messages all arrive on threads that have nothing to do with the audio scheduler or the graphics frame loop. The pattern for all three is the same: the external event writes to an atomic or an `InputNode` internal value, and the scheduler or metro reads it on its own cadence. The two sides never share a lock.

The only thing that changes between input sources is the binding call. Once the value is in an atomic or a node, everything downstream is identical.


{{< tutorial-detail title="Expansion 1: Event domain vs scheduler domain" >}}


`on_key_pressed`, `on_mouse_move`, and `on_mouse_pressed` create `Vruta::Event` coroutines managed by `EventManager`, not by `TaskScheduler`. They run on the windowing thread when GLFW delivers events. The `TaskScheduler` runs on the audio thread. These are different execution contexts with no shared lock.

Atomics are the correct bridge: write in the event coroutine, read in the scheduler task. Never call mesh mutation methods directly from event callbacks. `set_mesh_vertices` is not thread-safe against the graphics processor uploading the same buffer. Write to atomics in the event callback and apply the mutation in the metro.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: The held-state accumulator pattern" >}}


The keyboard tutorial does not route key events to velocity directly. It routes them to a boolean flag, and a separate metro reads that flag each frame to accumulate velocity. This split is intentional.

Key events are sparse and irregular: they arrive when the OS delivers them. The metro is regular at 60Hz. If you wrote velocity directly in the key callback, the push rate would be event-rate (unpredictable) rather than frame-rate (uniform). Two keys held for the same duration would push different amounts depending on how many events the OS generated.

The flag-and-accumulate pattern converts asynchronous signal to time-uniform force. The OS delivers events, the events write a state, the regular integrator reads that state on a fixed cadence.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: `on_mouse_pressed` vs `on_mouse_move` vs `on_key_pressed`" >}}


`on_key_pressed` fires once per physical key-down event. It does not repeat. Holding a key does not re-fire it; that is what `on_key_released` brackets. Use it for one-shot state changes: begin a sequence, toggle a flag, fire a `TimedAction`.

`on_mouse_pressed` fires once per button-down, with a position argument. Use it for picking or anchoring: record the cursor position at click time, start measuring drag distance from there.

`on_mouse_move` fires on every cursor movement event the OS generates. At high mouse sensitivity and low system load this can exceed 1000 events per second. The callback must be cheap. Writing two atomics is cheap. Calling `set_mesh_vertices` from inside it is not.

All three create `Vruta::Event` coroutines. None of them block or sleep: they run their callback and yield immediately.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: OSC input is asynchronous then synchronous" >}}


The OSC message arrives over UDP on a network thread, gets pushed into a lock-free queue in `InputManager`, dispatched to the matching `OSCNode`, and written to a `std::atomic<double>` inside the node. From that point it behaves like any other node: `get_last_output()` reads the atomic. The asynchrony is fully contained inside the input infrastructure. The processing graph never sees a thread boundary.

`read_osc` calls `register_input_node` internally before returning, so no `| Audio` pipe is needed. This is different from nodes created via the macro-generated `vega.Sine()` path, which only constructs and returns without registering.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: MIDI and HID follow the same pattern" >}}


``` cpp
// MIDI CC 74 -> filter cutoff
auto filter_ctrl = vega.read_midi(
    MIDIConfig::cc(74, 0.0, 1.0),
    Core::InputBinding::midi_cc(74));

// HID gamepad left stick X axis -> rotation speed
auto stick_x = vega.read_hid(
    HIDConfig::axis(0, -1.0, 1.0),
    Core::InputBinding::hid(0, HIDAxis::LEFT_X));
```

Both return nodes. `get_last_output()` reads their current value the same way as an OSC node. The binding call registers with `InputManager` to route incoming hardware events to the node's internal atomic. The processing graph sees no difference between a MIDI CC value, a gamepad axis, and a Sine oscillator. They are all numbers.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 6: Gaussian deformation as a general pattern" >}}


The mouse example uses a Gaussian: `exp(-dist^2 * falloff) * height`. The falloff and height parameters are decoupled. Falloff controls how far the influence spreads; height controls its magnitude. Changing falloff from `2.5` to `12.0` produces a sharp, nearly binary on/off at close range.

A sum of multiple Gaussians with different centers, falloffs, and signs produces terrain: positive centers are hills, negative are craters, overlapping terms produce ridges. The cursor position is just one source; OSC coordinates, counter phase, or any computed value works equally.

``` cpp
// Two cursors: one hill, one crater
const float y_hill   =  std::exp(-(dx1*dx1 + dz1*dz1) * 3.0f) * 0.5f;
const float y_crater = -std::exp(-(dx2*dx2 + dz2*dz2) * 5.0f) * 0.3f;
v.position.y = y_hill + y_crater;
```



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}



