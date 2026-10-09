---
build:
  list: never
---

### The Next Step

A simulation sounds like something that belongs to heavy software and long renders. It does not. A simulation is a grid of numbers, a few rules that rewrite the grid sixty times a second, and a way to draw it. That is data, like a recording or a picture, and it can be loaded, listened to, steered and rewritten in the same way.

Smoke is a grid with a density in every cell. Heat is a second grid. The rules say that heat rises, that density is carried along, and that the air cannot be squeezed. Everything else is a choice. How thick the air is, how fast the heat fades, where something is added and how quickly, and what the whole thing looks like are numbers you set, and any of them can be driven by a sound, a pointer, or another number.

The two blocks below show it. The first loads a fire that someone else made in another program, sets it burning again, and lets a drone decide when it flares. The second builds a fire from nothing, and puts your hands on it.

### Fire From a File

```cpp
#include "MayaFlux/SimulationIncludes.hpp"

void compose() {
    auto window = create_window({ "Fire From a File", 1600, 900 });
    window->show();

    auto volume = vega.read_volume();
    if (!volume) {
        return;
    }

    const auto bounds = volume->get_bounds();
    const glm::vec3 centre = (bounds.min + bounds.max) * 0.5F;
    const float radius = glm::length(bounds.max - bounds.min) * 0.5F;

    Buffers::ScalarRef density { .name = "density" };
    Buffers::ScalarRef temperature { .name = "temperature" };

    vega.evolve(
        volume,
        { .time_step = 1.0F / 60.0F,
            .viscosity = 0.00015F * radius,
            .jacobi_iterations = 28,
            .carried = { { density, 0.997F } },
            .buoyancy = Buffers::VolumeGridBuffer::FlowConfig::Buoyancy {
                .temperature = temperature,
                .density = density,
                .temperature_gain = 1.0F,
                .density_gain = 0.9F * radius } },
        { .target_window = window });

    volume->seed("velocity", Kinesis::VectorField { [](glm::vec3) { return glm::vec3(0.0F); } });
    volume->seed("pressure", Kinesis::SpatialField { [](glm::vec3) { return 0.0F; } });

    auto smoke = std::make_shared<Buffers::RaymarchBuffer>(
                     volume->get_lattice(),
                     [volume] { return volume->read_handle("density"); },
                     volume->get_field_bytes("density"))
        | Graphics;

    smoke->setup_rendering({ .target_window = window });
    smoke->get_render_processor()->set_view_transform(
        Kinesis::look_at_perspective(
            centre + glm::vec3(radius, radius * 0.6F, radius) * 1.5F, centre,
            0.9F, 16.0F / 9.0F, radius * 0.02F, radius * 10.0F));

    smoke->march_processor()->configure({
        .absorption = 10.0F,
        .emission = 2.0F,
        .cool = { 0.60F, 0.10F, 0.02F },
        .hot = { 1.00F, 0.85F, 0.40F },
        .threshold = 0.01F });

    const float height = bounds.max.y - bounds.min.y;
    const glm::vec3 base(centre.x, bounds.min.y + 0.15F * height, centre.z);

    auto heat = volume->influx("temperature", { .center = base, .rate = 0.0F });
    auto soot = volume->influx("density", { .center = base, .rate = 0.0F });

    constexpr std::array<double, 7> roots { 55.0, 110.0, 164.8, 220.0, 329.6, 440.0, 659.3 };

    auto drone = std::make_shared<std::vector<std::shared_ptr<Nodes::Generator::Sine>>>();
    auto tuning = std::make_shared<std::array<double, 7>>();
    for (size_t i = 0; i < roots.size(); ++i) {
        (*tuning)[i] = roots[i] + get_uniform_random(-3.0, 3.0);
        auto breath = vega.Sine(0.04F + 0.02F * static_cast<float>(i), 0.12);
        auto partial = vega.Sine(static_cast<float>((*tuning)[i]), 0.25) | Audio[i % 2];
        partial->set_amplitude_modulator(breath);
        drone->push_back(partial);
    }

    auto kick = std::make_shared<float>(0.0F);
    auto armed = std::make_shared<bool>(false);

    Kriya::EventChain chain(*get_scheduler());
    chain.then([armed]() { *armed = true; }, 12.0).start();

    schedule_metro(2.5, [drone, tuning, roots, kick, armed]() {
        double moved = 0.0;
        for (size_t i = 0; i < roots.size(); ++i) {
            const double next = roots[i] + get_uniform_random(-4.0, 4.0);
            moved += std::abs(next - (*tuning)[i]);
            (*tuning)[i] = next;
            (*drone)[i]->set_frequency(static_cast<float>(next));
        }
        if (*armed) {
            *kick = static_cast<float>(std::clamp(moved / 14.0, 0.25, 1.0));
        }
    });

    schedule_metro(
        1.0 / 60.0,
        [kick, heat, soot]() {
            *kick *= 0.97F;
            heat->set_rate(5.0F * *kick);
            soot->set_rate(2.4F * *kick);
        },
        "fire_kick", Vruta::ProcessingToken::FRAME_ACCURATE);
}
```
Run this code. A file dialog opens. Choose a `.vdb` file with a density field and a temperature field. The fire sample from the OpenVDB website works, and so does a fire you have exported from another program.

The fire appears as it was saved, and starts to move again under rules of your choosing. A drone of seven slightly detuned tones plays underneath it. For the first twelve seconds nothing connects the two, and the fire burns on its own. Then every time the drone changes its tuning, which is every two and a half seconds, a flare of heat and smoke is added at the bottom of the fire. A small shift gives a small flare and a big shift gives a big one.

You do nothing here. The sound and the fire do it between them.

Change one thing at a time:

- **`0.997F` to `0.9995F`:** the smoke hangs around much longer
- **`0.00015F` to `0.0015F`:** the air thickens, and the fire turns slow and heavy
- **`12.0` to `0.0`:** the drone steers the fire from the first second
- **`5.0F * *kick` to `15.0F * *kick`:** every change in the drone is an explosion
- **`roots`:** your own seven frequencies. Closer tones beat more slowly and make fewer, longer flares

{{< tutorial-detail title="Also: Check What Is In the File" >}}

A file from another program names its fields however it likes. This block assumes `density` and `temperature`. To see what yours holds, add this after the file is loaded:
```cpp
for (const auto& name : volume->get_field_names()) {
    MF_PRINT("field {} stride {}\n", name, volume->get_field_stride(name));
}
```
A stride of 4 is one number per cell. A stride of 16 is a direction. Use the name of a field with a stride of 4 for `density`.

{{< /tutorial-detail >}}

### Hands on the Fire

The box comes first. It is a fire that burns without end, with a camera, a pointer and a few keys, kept in one function so that the part that matters is short. Paste both parts into one file.

```cpp
struct Hand {
    glm::vec3 at { 0.0F };
    bool down = false;
    bool smoke = false;
    float size = 1.0F;
    float burst = 0.0F;
};

struct Ember {
    std::shared_ptr<Core::Window> window;
    std::shared_ptr<Buffers::VolumeGridBuffer> volume;
    std::shared_ptr<Buffers::InfluxProcessor> heat;
    std::shared_ptr<Buffers::InfluxProcessor> soot;
    std::shared_ptr<Hand> hand = std::make_shared<Hand>();
    glm::vec3 source { 0.0F, -0.8F, 0.0F };
};

Ember make_ember() {
    constexpr float fov = 0.85F;
    constexpr float aspect = 16.0F / 9.0F;
    constexpr float distance = 2.9F;

    Ember e;
    e.window = create_window({ "Hands on the Fire", 1600, 900 });
    e.window->show();

    const Kinesis::Lattice3D lattice {
        .resolution = glm::uvec3(64),
        .bounds = { .min = glm::vec3(-1.0F), .max = glm::vec3(1.0F) }
    };

    auto volume = std::make_shared<Buffers::VolumeGridBuffer>(lattice);

    auto density = volume->declare_scalar("density");
    auto temperature = volume->declare_scalar("temperature");

    e.volume = vega.evolve(
        volume,
        { .time_step = 1.0F / 60.0F,
            .viscosity = 0.00012F,
            .jacobi_iterations = 32,
            .walls = true,
            .carried = { { temperature, 0.992F }, { density, 0.9985F } },
            .buoyancy = Buffers::VolumeGridBuffer::FlowConfig::Buoyancy {
                .temperature = temperature,
                .density = density,
                .temperature_gain = 3.4F,
                .density_gain = 1.1F } },
        { .target_window = e.window });

    volume->seed("velocity", Kinesis::VectorField { [](glm::vec3) { return glm::vec3(0.0F); } });
    volume->seed("pressure", Kinesis::SpatialField { [](glm::vec3) { return 0.0F; } });
    volume->seed(temperature, Kinesis::SpatialField { [](glm::vec3) { return 0.0F; } });
    volume->seed(density, Kinesis::SpatialField { [](glm::vec3) { return 0.0F; } });

    e.heat = volume->influx("temperature", { .center = e.source, .radius = 0.16F, .rate = 2.8F });
    e.soot = volume->influx("density", { .center = e.source, .radius = 0.2F, .rate = 1.2F });

    const auto view = [] {
        return Kinesis::look_at_perspective(
            glm::vec3(0.0F, 0.0F, distance), glm::vec3(0.0F),
            fov, aspect, 0.1F, 40.0F);
    };

    Portal::Graphics::BlendAttachmentConfig additive;
    additive.blend_enable = true;
    additive.src_color_factor = Portal::Graphics::BlendFactor::SRC_ALPHA;
    additive.dst_color_factor = Portal::Graphics::BlendFactor::ONE;
    additive.src_alpha_factor = Portal::Graphics::BlendFactor::ZERO;
    additive.dst_alpha_factor = Portal::Graphics::BlendFactor::ONE;

    auto fire = std::make_shared<Buffers::RaymarchBuffer>(
                    lattice,
                    [volume, temperature] { return volume->read_handle(temperature); },
                    volume->get_field_bytes(temperature))
        | Graphics;

    fire->setup_rendering({ .target_window = e.window });
    fire->get_render_processor()->set_view_transform_source(view, false, Portal::Graphics::CullMode::FRONT);
    fire->get_render_processor()->set_blend_attachment(additive);
    fire->march_processor()->configure({
        .step_scale = 0.55F,
        .absorption = 7.0F,
        .emission = 3.6F,
        .cool = { 0.60F, 0.08F, 0.01F },
        .hot = { 1.00F, 0.90F, 0.58F },
        .threshold = 0.05F });

    auto smoke = std::make_shared<Buffers::RaymarchBuffer>(
                     lattice,
                     [volume, density] { return volume->read_handle(density); },
                     volume->get_field_bytes(density))
        | Graphics;

    smoke->setup_rendering({ .target_window = e.window });
    smoke->get_render_processor()->set_view_transform_source(view, false, Portal::Graphics::CullMode::FRONT);
    smoke->march_processor()->configure({
        .absorption = 10.0F,
        .cool = { 0.28F, 0.28F, 0.32F },
        .hot = { 0.85F, 0.83F, 0.80F },
        .threshold = 0.006F });

    auto window = e.window;
    auto hand = e.hand;

    auto reach = [window](double x, double y) {
        const auto ndc = normalize_coords(x, y, window);
        const float half = std::tan(fov * 0.5F) * distance;
        return glm::vec3(
            std::clamp(ndc.x * half * aspect, -1.0F, 1.0F),
            std::clamp(-ndc.y * half, -1.0F, 1.0F),
            0.0F);
    };

    on_mouse_move(window, [hand, reach](double x, double y) { hand->at = reach(x, y); });
    on_mouse_pressed(window, IO::MouseButtons::Left, [hand, reach](double x, double y) {
        hand->at = reach(x, y);
        hand->down = true;
    });
    on_mouse_released(window, IO::MouseButtons::Left, [hand](double, double) { hand->down = false; });
    on_mouse_pressed(window, IO::MouseButtons::Right, [hand, reach](double x, double y) {
        hand->at = reach(x, y);
        hand->smoke = true;
    });
    on_mouse_released(window, IO::MouseButtons::Right, [hand](double, double) { hand->smoke = false; });
    on_scroll(window, [hand](double, double dy) {
        hand->size = std::clamp(hand->size * (1.0F + 0.1F * static_cast<float>(dy)), 0.4F, 3.0F);
    });
    on_key_pressed(window, IO::Keys::Space, [hand]() { hand->burst = 1.0F; });

    return e;
}
```
Now the part that matters. It measures the fire, relates the measure to your hand, and turns the result into the fire's rates and into sound:
```cpp
#include "MayaFlux/SimulationIncludes.hpp"

void compose() {
    using namespace Nodes::Network;

    auto e = make_ember();
    auto hand = e.hand;

    auto cells = std::make_shared<std::vector<float>>(e.volume->get_lattice().cell_count());
    auto fill = std::make_shared<double>(0.0);

    schedule_metro(
        0.5,
        [e, cells, fill]() {
            e.volume->read_field("density", cells->data(), cells->size() * sizeof(float));

            size_t held = 0;
            for (const float v : *cells) {
                if (std::isfinite(v) && v > 0.05F) {
                    ++held;
                }
            }
            *fill = static_cast<double>(held) / static_cast<double>(cells->size());
        },
        "ember_survey", Vruta::ProcessingToken::FRAME_ACCURATE);

    auto relation = std::make_shared<RelationNetwork>();
    relation->set_output_mode(OutputMode::GRAPHICS_BIND);

    relation->add_slots({
        { .name = "x" },
        { .name = "y" },
        { .name = "down" },
        { .name = "speed" },
        { .name = "fill" },
        { .name = "heat" },
        { .name = "soot" },
        { .name = "size" },
        { .name = "smoke" },
        { .name = "burst" },
    });

    auto read = relation->add<StepOperator>([](StepOperator& self, std::span<RelationSlot> slots, uint32_t) {
        const double x = self.parameter("x");
        const double y = self.parameter("y");
        slots[3].level = std::hypot(x - slots[0].level, y - slots[1].level) * 60.0;
        slots[0].level = x;
        slots[1].level = y;
        slots[2].level = self.parameter("down");
        slots[4].level = self.parameter("fill");
        slots[7].level = self.parameter("size");
        slots[8].level = self.parameter("smoke");
        slots[9].level = self.parameter("burst");
    });
    read->map("x", Source([hand] { return static_cast<double>(hand->at.x); }));
    read->map("y", Source([hand] { return static_cast<double>(hand->at.y); }));
    read->map("down", Source([hand] { return hand->down ? 1.0 : 0.0; }));
    read->map("fill", Source([fill] { return *fill; }));
    read->map("size", Source([hand] { return static_cast<double>(hand->size); }));
    read->map("smoke", Source([hand] { return hand->smoke ? 1.0 : 0.0; }));
    read->map("burst", Source([hand] { return static_cast<double>(hand->burst); }));

    relation->add<DeriveOperator>(5, std::vector<size_t> { 2, 3, 9 },
        [](std::span<const double> in) {
            return 2.8 + in[0] * (1.6 + 1.4 * std::min(in[1], 2.0)) + in[2] * 5.0;
        });

    relation->add<DeriveOperator>(6, std::vector<size_t> { 5, 4, 8 },
        [](std::span<const double> in) {
            return (in[0] * 0.43 + in[2] * 2.4) * (1.0 - std::clamp(in[1] * 2.5, 0.0, 1.0));
        });

    relation->add<StepOperator>([e](StepOperator&, std::span<RelationSlot> slots, uint32_t) {
        const bool held = slots[2].level > 0.5 || slots[8].level > 0.5;
        const glm::vec3 at(static_cast<float>(slots[0].level), static_cast<float>(slots[1].level), 0.0F);
        const auto size = static_cast<float>(slots[7].level);

        e.heat->set_center(held ? at : e.source);
        e.soot->set_center(held ? at : e.source);
        e.heat->set_radius(0.16F * size);
        e.soot->set_radius(0.2F * size);
        e.heat->set_rate(static_cast<float>(slots[5].level));
        e.soot->set_rate(static_cast<float>(slots[6].level));
    });

    schedule_metro(
        1.0 / 60.0,
        [hand]() { hand->burst *= 0.93F; },
        "ember_burst", Vruta::ProcessingToken::FRAME_ACCURATE);

    auto net = relation | Graphics;

    auto low = vega.Sine(110.0F, 0.0);
    auto high = vega.Sine(660.0F, 0.0);

    auto sound = std::make_shared<RelationNetwork>();
    sound->set_output_mode(OutputMode::AUDIO_COMPUTE);
    sound->set_output_scale(2.0);
    sound->add_slots({ { .name = "low", .node = low }, { .name = "high", .node = high } });

    auto voice = sound->add<StepOperator>([low, high](StepOperator& self, std::span<RelationSlot>, uint32_t) {
        const double lift = std::clamp((self.parameter("heat") - 2.8) / 4.0, 0.0, 1.0);
        const double stir = std::clamp(self.parameter("speed") / 2.0, 0.0, 1.0) * self.parameter("down");
        low->set_amplitude(0.04 + 0.14 * lift);
        high->set_amplitude(0.12 * stir);
        high->set_frequency(static_cast<float>(330.0 + 165.0 * (self.parameter("height") + 1.0)));
    });
    voice->map("heat", Source(net, 5));
    voice->map("speed", Source(net, 3));
    voice->map("down", Source(net, 2));
    voice->map("height", Source(net, 1));

    sound->add<AdvanceOperator>(*sound);
    sound->add<CombineOperator>(2);

    sound | Audio[0];
    auto speaker = vega.NetworkAudioBuffer(0, 512, sound) | Audio[{ 0, 1 }];
}
```
Run this code. A fire burns at the bottom of a box, and goes on burning. It never settles, because a small flame is always being added.

Move the pointer, and the fire follows it when you hold a button:

- **Hold the left button** to move the flame to the pointer and feed it. The faster you drag, the hotter it burns.
- **Hold the right button** to put smoke in and no heat, which is a cold dark plume.
- **Scroll** to make the flame wider or narrower.
- **Press Space** for a burst of heat that fades over about a second.

The fire is also its own limit. Every half second the box is measured for how much of it is filled with smoke, and the fuller it is, the less smoke the flame makes. A box that has filled up thins itself out, and nothing needs resetting.

You also hear it. A low tone swells as the flame gets hotter, and a high tone sounds while you drag and follows how fast you go. Its pitch follows how high the pointer is.

Everything between your hand and the fire passes through one small network of named values: where the pointer is, how fast it moves, how full the box is, and how hot and sooty the flame should be. You can read them, change them, and send them anywhere else.

Change one thing at a time:

- **`2.8` in the heat formula to `0.0`:** the fire only burns while you hold a button
- **`0.992F` to `0.98F`:** the heat fades faster, and the flame is short
- **`1.1F` in `density_gain` to `0.2F`:** smoke stops dragging the plume and it rises lazily
- **`2.5` in the soot formula to `0.5`:** the box can fill almost to the top before it thins
- **`glm::uvec3(64)` to `glm::uvec3(96)`:** a finer fire at several times the cost

You have, in both blocks:

- A simulation that you loaded or built, running on its own
- Ways to feed it that you can change while it runs: a sound, a pointer, a key
- A picture of it that you chose, and not one that came with it

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: A Simulation Is Data" >}}

A fluid solver feels like a separate world with its own tools. In MayaFlux it is not one. The volume is a buffer of numbers, the rules are processors that run on it every frame, and the picture is another buffer that reads it. All three are objects you hold, and every setting on them is a number you can change at any moment.

That is why a drone can flare a fire, or a mouse can feed one, with no special support. A drone changing its tuning is a function that sets a rate. A pointer is a function that sets a position. The fire does not care where the number came from.

It also means you are not limited to a simulation as it ships. You can start from a file someone else made, as in the first block, or from a function you wrote, as in the second Try It. You can measure it while it runs, as in the second block, and let what you measure change how it behaves. You can save what it does and load it again later, and the file is the same kind of thing that arrived.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: What a Volume Is" >}}

A volume is a box divided into a grid of cells. `Kinesis::Lattice3D` says how many cells there are and what part of the world the box covers:
```cpp
const Kinesis::Lattice3D lattice {
    .resolution = glm::uvec3(64),
    .bounds = { .min = glm::vec3(-1.0F), .max = glm::vec3(1.0F) }
};
auto volume = std::make_shared<Buffers::VolumeGridBuffer>(lattice);
```
Each thing the volume remembers is a field, a name with one value for every cell:
```cpp
auto density = volume->declare_scalar("density");
auto flow = volume->declare_vector("velocity");
```
A scalar field is one number per cell, how much smoke there is. A vector field has a direction in every cell, here which way the air is moving. `declare_scalar` and `declare_vector` give a field that keeps the previous step as well, which the rules need to read from one and write to the other. `declare_scratch` gives one that does not.

Memory is the cost to watch. A 64 by 64 by 64 box has 262,144 cells. A scalar field is one megabyte for each copy it keeps, and a field that keeps two is two. A direction field takes four times a scalar field. A flow adds several of these. Raise the resolution a little and the cost rises with the cube of it, so 96 cells a side is more than three times 64. Lower `64` on a small graphics card, and raise it on a large one.

To give a field its starting values, hand it a function from a position to a number:
```cpp
volume->seed(density, Kinesis::SpatialField { [](glm::vec3 p) { return std::exp(-glm::dot(p, p) / 0.05F); } });
volume->seed("velocity", Kinesis::VectorField { [](glm::vec3 p) { return glm::vec3(-p.z, 0.0F, p.x); } });
```
The first puts a soft ball of smoke at the middle. The second sets the whole box turning about the vertical. The function runs once for every cell.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: Loading a Volume" >}}

`vega.read_volume()` opens a file dialog, and `vega.read_volume("path/to/file.vdb")` takes a path. Either gives back a volume with one field for each grid in the file, already registered. A `.vdb` file can come from any program that writes OpenVDB.

A file you did not make has its own scale. Nothing says it is two units across, so the first block works out its size from what the volume reports:
```cpp
const auto bounds = volume->get_bounds();
const glm::vec3 centre = (bounds.min + bounds.max) * 0.5F;
const float radius = glm::length(bounds.max - bounds.min) * 0.5F;
```
The camera sits at a multiple of that radius, and the thickness of the air and the pull of the smoke are multiplied by it. A flow tuned for a box two units across means nothing at another size, and this is what carries it over.

A fire file has the fields a fire needs, density and temperature. It does not have a velocity or a pressure, which the rules need and which the file never stored. `vega.evolve` adds any field it needs that the volume lacks, under the names `velocity`, `pressure` and `divergence`, and the block then fills the new ones with zero. It does not touch a field the file already has.

One line you will see in the log is expected: `setup_rendering: no SurfaceConfig was supplied at construction`. `vega.evolve` always tries to draw a volume as a solid surface, and this volume was not built for one. The flow is already running when the line appears, and the next expansion shows what draws it instead.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: The Flow" >}}

`vega.evolve(volume, config, render)` takes a volume that has not been drawn yet, and does the whole sequence of setting it going: it declares what is missing, registers it, builds the stages that move the fluid, and attaches rendering. The config says how the fluid behaves:
```cpp
{ .time_step = 1.0F / 60.0F,
    .viscosity = 0.00012F,
    .jacobi_iterations = 32,
    .walls = true,
    .carried = { { temperature, 0.992F }, { density, 0.9985F } },
    .buoyancy = Buffers::VolumeGridBuffer::FlowConfig::Buoyancy {
        .temperature = temperature,
        .density = density,
        .temperature_gain = 3.4F,
        .density_gain = 1.1F } }
```
- **`time_step`** is how much time one step covers. Keep it equal to the frame time.
- **`viscosity`** is how thick the air is. Water is small and honey is large. It smooths the motion, and a large value kills fine swirls.
- **`jacobi_iterations`** is how many passes make the air incompressible each step. Fewer passes are cheaper and let the air squash, which looks like tearing.
- **`walls`** stops the flow leaking through the sides of the box.
- **`carried`** lists the fields the air carries along, each with a number just under one. A field is multiplied by that number every step, so smoke fades slowly at 0.9985 and heat fades faster at 0.992. A number of exactly one never fades.
- **`buoyancy`** makes heat rise and smoke weigh. The two gains say how strongly each pushes. Change them and the same fire becomes a lazy plume or a jet.

The stages run in a fixed order each step: the flow carries itself along, buoyancy pushes it, viscosity smooths it, the walls hold it, then the air is made incompressible, and last the carried fields are moved by the corrected flow. You do not build these one by one. They stay in the volume's processing chain if you ever want to reach one.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Feeding a Fire" >}}

A simulation with nothing added to it runs down. Heat cools and smoke thins until the box is quiet, which is what the examples without a source did. `volume->influx` adds a stage that puts something into one field every step:
```cpp
auto heat = volume->influx("temperature", { .center = source, .radius = 0.16F, .rate = 2.8F });
```
- **`center`** is where the soft round blob sits.
- **`radius`** is how wide it is. Leave it out and it is 8 percent of the box's smallest side.
- **`rate`** is how much it adds each second at its middle. A rate of zero keeps the stage but adds nothing.
- **`time_step`** and **`falloff`** are also there, for a different step or a harder edge.

Two sources feed the fire, heat into `temperature` and smoke into `density`. Their rates are balanced against the fading of the fields they feed. A source that adds faster than the field fades makes the box fill without end, and one that adds slower leaves it empty.

The stage object is what you keep. Its setters work while the fire runs, and the second block moves the centre to the pointer, widens the radius with the scroll wheel and sets the rate from the relation network every frame. The first block leaves the rate at zero and raises it for a moment each time the drone changes.

Stages added after `vega.evolve` run behind the flow stages, so what they add is moved by the flow one step later. It is not visible in the picture.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Drawing a Volume" >}}

`Buffers::RaymarchBuffer` draws a field by looking through it. It is a box around the lattice, and every pixel of the box sends a ray in and adds up what the ray passes through, so thin smoke is faint and thick smoke is solid. Nothing is thresholded into a surface. A field that fades, fades in the picture.

```cpp
auto smoke = std::make_shared<Buffers::RaymarchBuffer>(
                 lattice,
                 [volume, density] { return volume->read_handle(density); },
                 volume->get_field_bytes(density))
    | Graphics;
```
The function is called each frame because the field swaps between two copies as it runs, and the picture must read the current one. Several buffers can read one volume. The second block draws temperature as fire and density as smoke from the same box. They read the same cycle's result, one frame behind the flow.

`march_processor()->configure({...})` sets how it looks. Name what you want to change and leave the rest:

- **`absorption`** is how quickly the volume turns opaque
- **`emission`** is how brightly it glows
- **`cool` and `hot`** are the colours at the faint end and the dense end
- **`threshold`** drops everything fainter than it, which clears the smeared tail a fire leaves
- **`density_scale`** is a quick way to make the whole thing thicker or thinner
- **`step_scale`** is the length of a step as a fraction of one cell
- **`max_steps`** is how many steps a ray may take. Left out, it is enough to cross the box from corner to corner.

Fire is drawn with additive blending so overlapping layers get brighter, and smoke without it, so it dims what is behind it. The buffers are drawn from the front, so the camera can sit inside the box.

There is another way to draw a volume. Give it a `SurfaceConfig` when you build it and it draws the shape of one value, a solid surface like the skin of a balloon. This looks right while the field has a firm edge. While a fire fades and smears, the edge is eaten away and the surface breaks into facets and ends. That is why the fire here is looked through and not thresholded.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: A Sound That Changes the Fire" >}}

The first block never touches the fire from the sound. It touches something the sound also reads.

A tone is a `Sine` made with a frequency and an amplitude, `vega.Sine(220.0F, 0.25)`. Seven of them sit on a stack of intervals from 55 to 659 hertz, each nudged a few hertz off its note, so they beat against each other slowly. Each has a second, very slow `Sine` as its amplitude modulator, a different rate for each, so the chord thickens and thins by itself.

Every two and a half seconds a scheduled function moves every tone to a new nearby pitch, and adds up how far they all moved. That number is the news. A large total becomes a large flare, a small one a small flare, and the flare is held in `kick`. A second function runs every frame, lets `kick` fall by 3 percent, and sets both sources' rates from it. The fire sees only the rates.

The first twelve seconds are left alone with an event chain:
```cpp
Kriya::EventChain chain(*get_scheduler());
chain.then([armed]() { *armed = true; }, 12.0).start();
```
`then(action, delay)` waits, then runs the action. Here the action switches on a flag, and the drone's function only sets `kick` once the flag is up. It gives the loaded fire time to be seen as it was made before anything starts to interfere with it.

Any change is a good trigger. The measure here is how far the tuning moved. It could as easily be how loud the sound is, a note arriving, or a pointer moving. The earlier cards measured sounds in many ways, and any of those numbers could stand where `moved` is.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: The Relation Network" >}}

The second block keeps every number that connects your hands to the fire in one place, a `RelationNetwork`. It is a list of named slots, each holding one number, and a chain of operators that run in order every frame.

The slots are `x`, `y`, `down`, `speed`, `fill`, `heat`, `soot`, `size`, `smoke` and `burst`. The operators are three kinds:

- **A step** is a function over all the slots. The first one reads the world into the slots. The last reads the slots out into the fire's sources.
- **A derive** makes one slot from others. The `heat` slot is the pointer being down, plus how fast it moves, plus any burst, on top of a steady base. The `soot` slot is the heat, plus any right button, reduced as `fill` rises.
- **A source** is what a step reads its inputs from. `read->map("x", Source(...))` ties a name to a function that returns a number once a frame, and the step reads it with `self.parameter("x")`. A source can also be a node, a constant, or a slot of another network.

The order is the point. The values are read, then the derives work out what they mean, then the last step applies them. To change how the fire answers your hand you change one derive, and the sources stay the same.

This network runs once a frame, on the same clock that draws the picture. That is why it may measure the volume and write straight into the fire's sources. The measuring is a download from the graphics card, which is slow and must never be done from the audio thread. So it is done on a half second timer, and the network reads the result.

The sound is a second network of the audio kind. It has two tones as slots, and a step that sets their level and pitch from the first network's slots, with `Source(net, 5)` reading slot 5 of it. It runs on the audio thread, once for each block of sound. It reads numbers the frame side wrote and does nothing else with the fire. `sound->set_output_scale(2.0)` sets how loud the network is. `vega.NetworkAudioBuffer` carries its block to the speakers, and registering the network with `| Audio[0]` as well lets the audio engine run it.

Nothing about the network is specific to fire. The same slots could hold a measure of a recording, a number from a tablet, or one from another volume, and the derives would relate them just the same.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: Pointer and Keys to Space" >}}

A mouse reports a position in the window. The fire lives in a box in the world. The block joins them with the simplest possible assumption. The camera looks straight at the middle of the box from a fixed distance, so every window position corresponds to a point on the plane through the middle of the box:
```cpp
const auto ndc = normalize_coords(x, y, window);
const float half = std::tan(fov * 0.5F) * distance;
return glm::vec3(ndc.x * half * aspect, -ndc.y * half, 0.0F);
```
`normalize_coords` gives -1 to 1 across the window, and the top is -1, so the vertical is flipped. The result is clamped to the box. The numbers `fov`, `aspect` and `distance` are the same ones that place the camera, so the pointer and the picture stay in step. If you move the camera, move them together.

A pointer does not tell you where it is in depth, so the fire is always touched in the middle plane. That is enough to paint on, and a way to choose depth, such as a key or the scroll wheel, is easy to add to the hand.

The callbacks run when the mouse moves, and write plain values into a small `Hand`. The relation network reads them once a frame. No callback touches the fire. A burst is the same pattern: Space sets `burst` to one, and a frame function lets it fall by 7 percent each frame.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 10: Keeping a Take" >}}

A volume can be saved and loaded like any other data, in the same format that came in:
```cpp
auto io = get_io_manager();
io->save_volume(volume, "take.vdb");
io->wait_for_pending_saves();
```
The download from the graphics card happens at once and the writing happens in the background, so `wait_for_pending_saves()` is needed before reading the file back. `load_volume` reads it as the first block did, and any program that reads OpenVDB can open it.

To record a stretch of the fire as a sequence of files, one for each frame it is sampled, there is `capture_volume`. The Try It shows it on a key.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### A Start You Author

Nothing is loaded. A function says where the smoke is, a second function says how the air is moving, and the rules stir one into the other:
```cpp
#include "MayaFlux/SimulationIncludes.hpp"

void compose() {
    auto window = create_window({ "A Start You Author", 1600, 900 });
    window->show();

    const Kinesis::Lattice3D lattice {
        .resolution = glm::uvec3(64),
        .bounds = { .min = glm::vec3(-1.0F), .max = glm::vec3(1.0F) }
    };

    auto volume = std::make_shared<Buffers::VolumeGridBuffer>(lattice);
    auto density = volume->declare_scalar("density");

    vega.evolve(
        volume,
        { .time_step = 1.0F / 60.0F,
            .viscosity = 0.00005F,
            .jacobi_iterations = 32,
            .walls = true,
            .carried = { { density, 0.9998F } } },
        { .target_window = window });

    volume->seed("pressure", Kinesis::SpatialField { [](glm::vec3) { return 0.0F; } });

    volume->seed(density, Kinesis::SpatialField { [](glm::vec3 p) {
        const float shell = std::exp(-std::pow(glm::length(p) - 0.6F, 2.0F) / 0.02F);
        const float bands = 0.5F + 0.5F * std::sin(p.y * 14.0F);
        return shell * bands;
    } });

    volume->seed("velocity", Kinesis::VectorField { [](glm::vec3 p) {
        return glm::vec3(-p.z, 0.4F * std::sin(p.x * 5.0F), p.x) * 1.4F;
    } });

    auto smoke = std::make_shared<Buffers::RaymarchBuffer>(
                     lattice,
                     [volume, density] { return volume->read_handle(density); },
                     volume->get_field_bytes(density))
        | Graphics;

    smoke->setup_rendering({ .target_window = window });
    smoke->get_render_processor()->set_view_transform(
        Kinesis::look_at_perspective(
            glm::vec3(1.6F, 1.0F, 2.6F), glm::vec3(0.0F),
            0.9F, 16.0F / 9.0F, 0.1F, 20.0F));

    smoke->march_processor()->configure({
        .absorption = 12.0F,
        .emission = 1.6F,
        .cool = { 0.10F, 0.20F, 0.35F },
        .hot = { 0.70F, 0.85F, 1.00F },
        .threshold = 0.02F });
}
```
Run it. A hollow shell of striped smoke hangs in the box, and the air begins to turn through it. The stripes are drawn out into long threads, then into finer ones, and the threads thin until the grid can no longer hold them. There is no source and no buoyancy, so this is a sculpture that is stirred, not a fire that burns, and nothing you do can bring back what it has smeared.

Change one thing at a time:

- **`14.0F` to `30.0F`:** many thin stripes, which blur out sooner
- **`0.00005F` to `0.002F`:** the air is thick, and the shell hardly moves
- **`1.4F` to `4.0F`:** a violent stir
- **`0.6F` to `0.3F`:** a small shell that rolls up on itself
- **The vector field:** swap `-p.z` and `p.x` for your own function of `p`, and the air moves as you describe

{{< tutorial-detail title="Also: Keep a Take" >}}

Add this after the volume is running, and a key starts and stops a recording, one file for every second frame:
```cpp
auto take = make_persistent_shared<uint32_t>(0);

on_key_pressed(window, IO::Keys::Enter, [take, volume] {
    auto io = get_io_manager();
    if (*take != 0) {
        io->stop_volume_capture(*take);
        *take = 0;
        return;
    }
    *take = io->capture_volume(volume, "take/smoke.{:04}.vdb", { "density" }, {}, 240, 2);
});
```
The `{:04}` in the name becomes the frame number. The last two numbers are the most frames to keep and the gap between them. A recording of that kind opens in other programs as an animated volume.

{{< /tutorial-detail >}}

### What You Achieved

You have:

- Loaded a volume made elsewhere, put it back under rules, and drawn it by looking through it
- Let a drone decide when a fire flares, and held the fire back until you had seen it as it was
- Built a fire from nothing and fed it from a pointer, a key, and a measure of the fire itself
- Kept the numbers between your hands and the fire in one small network you can read and rewrite
- Started a simulation from a function of your own, and recorded one

A simulation is not a thing apart. Its fields, its rules and its picture are objects you hold, and every setting on them is a number that anything can drive.

{{< /tutorial-detail >}}
