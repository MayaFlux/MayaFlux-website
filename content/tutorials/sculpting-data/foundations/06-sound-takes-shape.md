---
build:
  list: never
---

### The Next Step

A shape is two lists. One is a list of points in space, and the other is a list of triangles that join them. That is all a vase, a mountain or a creature is to the computer, and it is data like any other.

A sound has many sides at once. It is loud or soft, high or low, bassy, sharp, steady, or struck. A shape has many sides too: joints that turn, a surface that can be pressed, a thread that can be drawn through space. This section gives each side of the sound to a different side of a shape, and lets the shape have weight and memory, so that the sound pushes it instead of dictating it.

`vega.mint` takes a shape and puts it on screen in one call. Each block below is a complete `compose()`.

The first block needs nothing from you. A clock of nodes plucks a string and moves the figure, so the sound and the dance keep the same time. The next two listen to the microphone, because a shape that follows a voice or a clap in the room is the quickest way to see what each measurement means. Input has to be switched on before the program starts, which Expansion 8 of the first section shows how to do. The last one plays a whole recording instead, and listens to it as it plays.

Save each shader in the `data/shaders` folder of your project, under the name written above it. The code asks for it by that file name alone. The program looks for `data/shaders` in the folder you run it from, and one folder up, so run it from the project folder or its build folder. If the window stays blank or never appears, the shader was not found: look in the terminal for an error about failing to read the shader, and check the folder and the name.

### A Figure That Dances
`data/shaders/figure.frag`
```glsl
#version 460

layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_normal;
layout(location = 3) flat in uint in_slot;
layout(location = 4) in vec3 in_world_pos;

layout(location = 0) out vec4 out_color;

void main()
{
    vec3 n = normalize(in_normal);
    float key = max(dot(n, normalize(vec3(0.5, 0.8, 0.6))), 0.0);
    float rim = pow(1.0 - max(n.z, 0.0), 3.0);
    float k = float(in_slot) / 10.0;
    vec3 base = 0.55 + 0.45 * cos(6.2831 * (vec3(0.0, 0.33, 0.67) + k));
    out_color = vec4(base * (0.18 + 0.82 * key) + 0.25 * rim, 1.0);
}
```
```cpp
void compose() {
    auto window = create_window({ "A Figure That Dances", 1280, 800 });
    window->show();

    auto motion = [](float hz, std::function<double(double)> shape) {
        auto chain = vega.Phasor(hz) >> vega.Polynomial(shape);
        chain->exclude_output_from_graph();
        return chain;
    };

    auto sway = motion(0.4F, [](double x) { return std::sin(6.2832 * x); });
    auto reach = motion(0.12F, [](double x) { return 0.15 - 0.15 * std::cos(6.2832 * x); });
    auto thump = motion(2.0F, [](double x) { return 0.6 * std::exp(-6.0 * x); });
    auto whip = motion(2.0F, [](double x) { return std::exp(-4.0 * x) * std::sin(18.0 * x); });

    auto harp = vega.WaveguideNetwork(WaveguideNetwork::WaveguideType::STRING, 110.0) | Audio[{ 0, 1 }];
    harp->set_loss_factor(0.985);
    harp->set_exciter_type(WaveguideNetwork::ExciterType::NOISE_BURST);
    harp->set_exciter_duration(0.004);
    harp->set_output_scale(3.0);

    auto octave = std::make_shared<double>(1.0);
    auto raised = vega.Logic(0.2) | Audio[0];
    raised->set_input_node(reach);
    raised->exclude_output_from_graph();
    raised->on_change_to(true, [octave](const auto&) { *octave = 2.0; });
    raised->on_change_to(false, [octave](const auto&) { *octave = 1.0; });

    auto lead = std::make_shared<bool>(false);
    auto count = std::make_shared<int>(0);
    const std::array<double, 5> scale { 110.0, 123.5, 146.8, 165.0, 196.0 };

    auto step = vega.Impulse(2.0F) | Audio[0];
    step->exclude_output_from_graph();
    step->on_impulse([=](const auto&) {
        *lead = !*lead;
        harp->set_fundamental(scale[static_cast<size_t>((*count)++) % scale.size()] * *octave);
        harp->pluck(0.28, 0.9);
    });

    auto limb = [](float length, float radius) {
        return [=] {
            const std::vector<glm::vec3> path { { 0.0F, 0.0F, 0.0F }, { 0.0F, -length, 0.0F } };
            return Kinesis::generate_tube(std::span(path), [=](float) { return radius; }, 10, true);
        };
    };

    auto body = std::make_shared<Nodes::Network::MeshNetwork>();

    auto figure = vega.mint(StructureConfig::Model {
        .network = body,
        .components = {
            { .expression = limb(-0.6F, 0.14F) },
            { .expression = limb(-0.22F, 0.1F), .parent_component = 0 },
            { .expression = limb(0.55F, 0.06F), .parent_component = 0 },
            { .expression = limb(0.55F, 0.06F), .parent_component = 0 },
            { .expression = limb(0.8F, 0.08F), .parent_component = 0 },
            { .expression = limb(0.8F, 0.08F), .parent_component = 0 },
        },
        .render = { .target_window = window, .fragment_shader = "figure.frag" },
    });

    bind_orbit_preset(window, { .focal_point = { 0.0F, 0.9F, 0.0F }, .initial_distance = 4.5F });

    auto joint = [body](uint32_t part, const glm::vec3& at, const glm::vec3& angles) {
        auto& slot = body->get_slot(part);
        slot.local_transform = glm::translate(glm::mat4(1.0F), at) * glm::mat4_cast(glm::quat(angles));
        slot.dirty = true;
    };

    schedule_metro(1.0 / 60.0, [=]() {
        const auto lean = static_cast<float>(sway->get_last_output());
        const auto arms = 0.2F + 8.0F * static_cast<float>(reach->get_last_output());
        const auto bend = static_cast<float>(thump->get_last_output());
        const auto fling = 2.5F * static_cast<float>(whip->get_last_output());
        const float left = *lead ? 1.0F : 0.2F;

        joint(0, { 0.0F, 1.0F - 0.25F * bend, 0.0F }, { 0.0F, 0.0F, 0.2F * lean });
        joint(1, { 0.0F, 0.6F, 0.0F }, { 0.0F, 0.0F, -0.15F * lean });
        joint(2, { 0.2F, 0.5F, 0.0F }, { -left * fling, 0.0F, arms });
        joint(3, { -0.2F, 0.5F, 0.0F }, { -(1.2F - left) * fling, 0.0F, -arms });
        joint(4, { 0.1F, 0.0F, 0.0F }, { -0.8F * bend, 0.0F, 0.0F });
        joint(5, { -0.1F, 0.0F, 0.0F }, { -0.8F * bend, 0.0F, 0.0F });
    }, "dancer", Vruta::ProcessingToken::FRAME_ACCURATE);
}
```
Run this code. A figure of six parts stands in the window, and you hear a plucked string walking up and down a five note scale, two notes a second. Nothing is measured and nothing is loaded. A few nodes keep time, and everything else follows them:

- **The pulse** is a silent clock that ticks twice a second. Each tick is an event, and one function answers it three ways: it picks the next note, plucks the string, and hands the lead to the other arm.
- **The thump** is an envelope that jumps up on every tick and falls away. It is how far the knees dip and the body drops.
- **The whip** is a damped swing that restarts on every tick. It throws the leading arm forward and lets it settle.
- **The reach** is a slow swell over eight seconds that holds the arms higher and lower. A `Logic` node watches it, and when it passes a threshold on the way up, the next notes are an octave higher, and when it falls back they come down again. The arms are high when the notes are high.
- **The sway** is a slow wave that leans the body.

All the motions run at the same two beats a second and start together, so the knees, the arm and the note land at the same moment.

Drag with the middle mouse button to walk round it.

Change one thing at a time:

- **`2.0F` to `3.0F` in `thump`, `whip` and `step`:** a faster dance and a faster melody
- **`0.12F` to `0.5F`:** the arms rise and fall in two seconds, and the register jumps often
- **`18.0` to `6.0` in `whip`:** a slower, lazier swing. `40.0` is a shiver
- **`0.985` to `0.9995`:** the strings ring on and the notes blur together
- **`3.0` in `set_output_scale`:** how loud the strings are. `1.0` is the network's own level
- **The numbers in `scale`:** your own five notes. Put in `110.0, 220.0, 110.0, 220.0, 330.0` for a bounce
- **`0.2` to `0.1` in `vega.Logic`:** the melody spends most of its time an octave up

{{< tutorial-detail title="Also: Dance to Your Own Sound" >}}

A node is a stream of numbers, and so is a recording or a microphone. To make the arms follow a sound you bring in, replace the `reach` chain with a measurement of a block of it:
```cpp
auto block = create_input_listener_buffer(0);
```
```cpp
const auto& samples = block->get_data();
const auto n = static_cast<uint32_t>(samples.size());
const auto arms = 0.2F + 8.0F * static_cast<float>(Kinesis::Discrete::rms(samples, 1, n, n)[0]);
```
The first line goes before the `schedule_metro` and the rest replace the `arms` line inside it. Input has to be switched on before the program starts, which Expansion 8 of the first section shows how to do. The next blocks listen to the microphone like this, and Expansion 8 below lists what else a block can tell you besides its level.

{{< /tutorial-detail >}}

### The Potter's Wheel Remembers
`data/shaders/glaze.frag`
```glsl
#version 460

layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_world_pos;

layout(location = 0) out vec4 out_color;

void main()
{
    vec3 n = normalize(cross(dFdx(in_world_pos), dFdy(in_world_pos)));
    float light = abs(dot(n, normalize(vec3(0.5, 0.8, 0.6))));
    vec3 glaze = mix(vec3(0.93, 0.78, 0.55), vec3(0.42, 0.20, 0.14), in_uv.y);
    out_color = vec4(glaze * (0.25 + 0.75 * light), 1.0);
}
```
```cpp
#include "MayaFlux/IO/Model/ModelExport.hpp"

struct Wheel {
    float turns = 0.0F;
    float height = 0.0F;
};

void compose() {
    auto window = create_window({ "The Potter's Wheel Remembers", 1280, 800 });
    window->show();

    auto block = create_input_listener_buffer(0);

    constexpr int columns = 96;
    constexpr int rows = 64;

    auto clay = std::make_shared<std::vector<float>>(columns * rows, 0.0F);

    auto vessel = [clay] {
        return Kinesis::generate_parametric_surface(
            [clay](float u, float v) {
                const int i = static_cast<int>(u * columns + 0.5F) % columns;
                const int j = std::clamp(static_cast<int>(v * (rows - 1) + 0.5F), 0, rows - 1);
                const float belly = 0.55F * (1.0F + 0.15F * std::sin(glm::pi<float>() * v));
                const float foot = std::sqrt(std::clamp(v * 12.0F, 0.0F, 1.0F));
                const float r = belly * foot * (1.0F - (*clay)[j * columns + i]);
                const float a = u * glm::two_pi<float>();
                return glm::vec3(r * std::cos(a), v * 2.0F - 1.0F, r * std::sin(a));
            },
            columns, rows - 1);
    };

    auto node = std::make_shared<MeshWriterNode>((columns + 1) * rows);
    node->set_mesh(vessel());

    auto pot = vega.mint(StructureConfig::Object {
        .writer = node,
        .render = { .target_window = window, .fragment_shader = "glaze.frag" },
    });

    bind_orbit_preset(window);

    auto wheel = std::make_shared<Wheel>();

    schedule_metro(1.0 / 60.0, [node, block, clay, wheel, vessel]() {
        const auto& samples = block->get_data();
        const auto n = static_cast<uint32_t>(samples.size());
        if (n < 16) {
            return;
        }

        constexpr float dt = 1.0F / 60.0F;

        const auto loud = static_cast<float>(Kinesis::Discrete::rms(samples, 1, n, n)[0]);
        const auto sharp = static_cast<float>(Kinesis::Discrete::kurtosis(samples, 1, n, n)[0]);
        const auto pitch = static_cast<float>(
            Kinesis::Discrete::estimate_frequency(samples, Config::get_sample_rate(), 0.0, 0.0));
        const bool heard = loud > 0.01F;

        wheel->turns += dt * 0.35F;
        wheel->turns -= std::floor(wheel->turns);

        if (heard && pitch > 0.0F) {
            const float target = std::clamp(std::log2(pitch / 110.0F) / 4.0F, 0.0F, 1.0F);
            wheel->height += (target - wheel->height) * 0.2F;
        }

        for (auto& cell : *clay) {
            cell *= 0.998F;
        }

        if (heard) {
            const float bend = std::clamp((sharp + 1.5F) / 8.0F, 0.0F, 1.0F);
            const float sigma = 1.5F + 5.0F * (1.0F - bend);
            const int reach = static_cast<int>(sigma * 2.5F);
            const int ic = static_cast<int>(wheel->turns * columns);
            const int jc = static_cast<int>(wheel->height * (rows - 1));

            for (int dj = -reach; dj <= reach; ++dj) {
                const int j = jc + dj;
                if (j < 0 || j >= rows) {
                    continue;
                }
                for (int di = -reach; di <= reach; ++di) {
                    const int i = (ic + di + columns) % columns;
                    const float falloff = std::exp(-static_cast<float>(di * di + dj * dj) / (2.0F * sigma * sigma));
                    float& cell = (*clay)[j * columns + i];
                    cell = std::min(0.6F, cell + loud * 3.0F * dt * falloff);
                }
            }
        }

        node->set_mesh(vessel());
    }, "potter", Vruta::ProcessingToken::FRAME_ACCURATE);

    on_key_pressed(window, IO::Keys::F, [node]() {
        if (!IO::save_mesh_snapshot(node, "pot_{}.gltf")) {
            MF_PRINT("pot: export failed");
        }
    });
}
```
Run this code. A plain pot stands in the window, and a hand you cannot see circles it once every three seconds or so. Make a sound and the hand presses into the clay, and the clay stays pressed:

- **How loud the sound is** sets how deep the hand presses
- **How high the sound is** sets where on the pot it presses. A low hum cuts near the foot and a whistle near the lip, and a note that glides leaves a slanting groove
- **How sharp the sound is** sets the shape of the hand. A smooth tone is a flat palm that leaves a broad shallow groove, and a click is a fingertip that leaves a small pit

Sing a phrase and the notes are cut round the pot as a helix, high notes climbing and low notes sinking. Then the clay slowly relaxes and the marks soften over a few seconds. The pot remembers longer than the sound lasts, and a shorter memory than the whole performance is what lets you make more than one mark.

The pitch here is found by counting how often the sound crosses zero. It is clean for a hum, a whistle or a flute, and it wanders for a rich voice, which makes the groove wander too.

Press **F** and the pot on screen at that moment is saved as a `pot_` file next to the program, with the time in its name. Open it in any 3D program, or print it. The sound has made an object.

Change one thing at a time:

- **`0.998F` to `1.0F`:** the clay never relaxes, and everything you sing stays. Then `0.99F`: marks last about a second
- **`0.35F` to `1.5F`:** the hand circles faster, and each sound is cut shorter and sharper
- **`3.0F` to `8.0F` in the depth:** deep grooves from soft sounds
- **`5.0F` to `1.0F` in `sigma`:** the palm shrinks, and every sound is a pit
- **`110.0F` to `220.0F`:** the lowest notes sit at the foot only for a higher voice

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: What a Mesh Is" >}}

A mesh is a list of vertices and a list of indices. Each vertex is a point with a few numbers attached:

- `position`, where it is
- `color`, three numbers
- `weight`, one free number
- `uv`, where it sits on a picture
- `normal`, which way the surface faces
- `tangent`, the direction along the surface

The indices are taken three at a time, and each triple is one triangle. A cube is eight vertices and thirty-six indices, twelve triangles. The two lists together are a `MeshData`.

You rarely write a mesh out by hand. The generators in `Kinesis` build one from a rule, and the whole mesh is the rule's result.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: What vega.mint Does" >}}

`vega.mint` takes a description of a visible structure, builds everything it needs, and returns the real buffer, so you can still reach into it. It has five forms:

- `StructureConfig::Object`, one shape
- `StructureConfig::Model`, a set of parts, each with its own transform, optionally in a hierarchy
- `StructureConfig::Instances`, one shape repeated many times, each repeat with its own transform
- `StructureConfig::Assembly`, several different sources drawn together
- `StructureConfig::Isosurface`, a surface pulled out of a field by a compute shader.

Every form needs `.render.target_window`. There is no default window, and a missing one is logged as an error.

An `Object` takes exactly one source:
```cpp
vega.mint(StructureConfig::Object { .mesh = data, .render = { .target_window = window } });
vega.mint(StructureConfig::Object { .expression = [] { return Kinesis::generate_box({ 0.0F, 0.0F, 0.0F }, { 0.5F, 0.5F, 0.5F }, 1); }, .render = { .target_window = window } });
vega.mint(StructureConfig::Object { .writer = node, .render = { .target_window = window } });
```
A `mesh` is a shape you already have. An `expression` is a function mint calls once to get one. A `writer` is a live source you can change afterwards, and it is the one the pot uses. Giving none or more than one is an error.

Pictures for the shape go in `.textures`, and `.render` is the same render config as in the earlier sections, so `.fragment_shader` names your own shader.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: The Generators" >}}

Each generator returns a `MeshData` and takes the rule for the shape:

**Parametric surface.** A function from two numbers to a point. `u` and `v` each run from 0 to 1 across the surface, and the two segment counts say how many steps to take along each:
```cpp
Kinesis::generate_parametric_surface(
    [](float u, float v) { return glm::vec3(u, std::sin(u * 6.0F) * v, v); }, 64, 64);
```
There are `u_segs + 1` by `v_segs + 1` vertices. The pot is this: `u` is the angle round it and `v` the height, and the function reads the clay table at that spot.

**Explicit surface.** A table of heights, one for each point of a grid. The surface is `y = f(x, z)`, and the table is `f`:
```cpp
Kinesis::generate_explicit_surface(heights, columns, rows, { 4.0F, 4.0F }, 1.0F);
```
`heights` holds `columns * rows` values, one row after another. The next arguments are the size of the surface in X and Z and how much a height is worth in the world. The membrane uses this. When your data is already a table, this is far cheaper than a parametric surface: the slopes are read from the table instead of calling a function three times for every vertex, the triangles are reused from one frame to the next, and a large grid is filled on several threads. A table of numbers from a file or the brightness of a picture makes a landscape the same way.

**Revolution.** A profile turned around the vertical axis. The function takes `t` from 0 at the foot to 1 at the lip and returns the radius and the height at that point. The sweep is how far round it goes, `glm::two_pi<float>()` for the full turn:
```cpp
Kinesis::generate_revolution(
    [](float t) { return glm::vec2(0.5F + 0.3F * std::sin(t * 9.0F), t * 2.0F - 1.0F); }, 96, 128, glm::two_pi<float>());
```
A profile that does not reach a radius of 0 at its ends leaves the shape open there. The figure's head is this, with a profile that closes at both ends.

**Tube.** A thick thread along a list of points. It takes the points, a function that gives the radius at each position `t` along the thread, how many sides it has, and whether its ends are closed:
```cpp
Kinesis::generate_tube(path, [](float t) { return 0.05F + 0.03F * t; }, 12, true);
```
Every limb of the figure and the spine of time are this. Points that sit on top of each other are allowed, so a thread that has collapsed to a point in a silent room does not break.

**Box and grid.** `generate_box(center, half_extents, subdivisions)` and `generate_grid(center, extent_x, extent_z, columns, rows, normal)` give the plain shapes.

In all of them the uv runs round the shape in one direction and along it in the other. The glaze uses `uv.y` as height up the pot, and the spine uses it as age along the thread.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Shapes That Change" >}}

To change a shape while it runs, give mint a `MeshWriterNode`, keep hold of it, and give it a new mesh:
```cpp
auto node = std::make_shared<MeshWriterNode>(vertex_count);
node->set_mesh(first_shape);

auto buffer = vega.mint(StructureConfig::Object { .writer = node, .render = { .target_window = window } });

node->set_mesh(next_shape);
```
Every frame, the buffer takes whatever the node holds and sends it to the graphics card. You do not register the node anywhere. `set_mesh` replaces the vertices and the triangles, and `set_mesh_vertices` replaces only the vertices when the triangles stay the same.

The shapes in this section keep the same number of vertices from frame to frame. Make the first one the size you want to keep.

Change it on the frame clock, so that the shape and the picture agree:
```cpp
schedule_metro(1.0 / 60.0, [node]() { }, "name", Vruta::ProcessingToken::FRAME_ACCURATE);
```
The cost to watch is the size of the shape. It is rebuilt on the processor and uploaded in full every frame, so doubling the vertices doubles the work. Raise the counts of the shapes in this section until the frame rate starts to drop, and no further.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Seeing the Shape" >}}

A shape is flat until you give it a view. With no view at all the window shows only the part of the shape inside a unit cube, from the front. Two presets give you a camera in one line:
```cpp
bind_orbit_preset(window);
bind_fly_preset(window);
```
**Orbit** circles a point. Drag with the middle mouse button to turn around it, hold the pan modifier and drag to slide, and scroll to come in or out. It starts five units away, looking at the origin, which suits a shape a unit or two across.

**Fly** is a first-person camera. Drag with the right mouse button to look, and use the movement keys to travel. It also starts five units back from the origin.

Both take a config for a shape of another size. The figure stands about two units tall with its feet at the floor, so it looks at a point nearer its middle:
```cpp
bind_orbit_preset(window, { .focal_point = { 0.0F, 0.9F, 0.0F }, .initial_distance = 4.5F });
```
Models are usually far larger than a unit. A model about three hundred high needs this:
```cpp
bind_orbit_preset(window, {
    .focal_point = { 0.0F, 80.0F, 0.0F },
    .initial_distance = 400.0F,
    .near_plane = 1.0F,
    .far_plane = 100000.0F,
});
```
If a model does not show, it is usually off to one side or too far away. Set the focal point to the middle of the model and the distance to about twice its height. Orbit and fly bind to every shape in the window, so call one after the shapes exist.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Light and Colour" >}}

The default shader draws each vertex in its own colour and does no lighting, so a shape looks flat. What your fragment shader receives depends on what kind of shape it draws.

An `Object` passes three things:
```glsl
layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_world_pos;
```
It does not pass the normal. The shaders for the pot, the membrane and the spine get one anyway, from the screen:
```glsl
vec3 n = normalize(cross(dFdx(in_world_pos), dFdy(in_world_pos)));
```
`dFdx` and `dFdy` say how the position changes from one pixel to the next, and their cross product is a line straight out of the surface. It gives each triangle a flat, faceted shade, and it points one way or the other depending on how the triangle is turned, so the light uses `abs(dot(n, light))`.

A `Model` passes the normal, and also the number of the part being drawn:
```glsl
layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_normal;
layout(location = 3) flat in uint in_slot;
layout(location = 4) in vec3 in_world_pos;
```
The figure uses `in_slot` to give every part its own colour. The number is the part's place in drawing order, where a parent comes before its children, so it is not always the order you listed them in.

`Instances` pass the normal but no part number:
```glsl
layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_normal;
layout(location = 3) in vec3 in_world_pos;
```
Anything that is a function of position, normal or uv works the same way: stripes, rings, a glaze.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: Nodes That Make and Move" >}}

A node is one number per sample, made by a rule and given to whatever asks for it. The same node can go to the speakers, to another node, to an event, or to a shape. The figure uses all four.

**A motion is a clock shaped by a function.** A `Phasor` counts from 0 to 1 and starts again, as often as its frequency says. A `Polynomial` takes a number and returns a function of it. Chain them with `>>` and the clock's number goes into the function:
```cpp
auto thump = vega.Phasor(2.0F) >> vega.Polynomial([](double x) { return 0.6 * std::exp(-6.0 * x); });
```
Twice a second the clock runs from 0 to 1, and the function turns that into a number that jumps up and falls away. Any function of `x` from 0 to 1 is a motion. A sine is a wave, a rising line is a sweep, and `exp(-4x) * sin(18x)` is a swing that dies away.

`a >> b` gives back a chain node. It runs `a`, hands the result to `b`, and its output is the output of `b`. It is registered for you, so it runs without any more calls.

**Silent nodes.** A node that is registered adds its output to the speakers. `chain->exclude_output_from_graph()` keeps it running, with its events firing, and leaves it out of the mix. The motions of the figure and its pulse clock are silent.

**One number, two places.** `thump->get_last_output()` is the number as it is right now, and the frame callback reads it to dip the knees. The same node can go to the speakers instead. A `Sine` adds the output of its modulator to its own amplitude, so a `Sine` made with an amplitude of `0.0`, the second argument, takes the modulator as its whole envelope:
```cpp
auto kick = vega.Sine(80.0F, 0.0) | Audio[0];
kick->set_amplitude_modulator(thump);
```
Then the thump is a sound you hear and a distance you see, from one node. The figure keeps its sound to the string and gives the string its notes from an event instead.

**Events.** A node tells you when something happens, and you hand it a function to call:
```cpp
step->on_impulse([=](const auto&) {
    *lead = !*lead;
    harp->set_fundamental(scale[static_cast<size_t>((*count)++) % scale.size()] * *octave);
    harp->pluck(0.28, 0.9);
});
raised->on_change_to(true, [octave](const auto&) { *octave = 2.0; });
```
The first function is the whole melody: it is called on every pulse, so it picks the next note of the scale and plucks. The second changes a number that the first one reads, which is how one event steers another.
- `Impulse::on_impulse` fires on each pulse. Excluded from the graph, an `Impulse` is a clock that nobody hears
- `Logic` turns the output of a node into on and off at a threshold. `set_input_node(reach)` feeds it, and `on_change_to(true, ...)`, `on_change_to(false, ...)`, `while_true`, `while_false` and `on_change` fire as it switches
- `Phasor::on_phase_wrap` fires when the count starts again, and `Counter::on_wrap` does the same for a `Counter`, which counts once for every sample
- `on_tick` fires on every sample, which is far more often than a frame needs

The functions run on the thread that processes the node, and that is the audio thread for a node in `Audio`. Keep them short: change a number or call a setter. The figure's events do this, and the drawing reads the result on the frame clock.

A node can be changed while it runs. `set_frequency` in the event above takes effect on the next sample.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: Hearing a Block in Many Ways" >}}

The sound is a block of numbers. For a microphone it is the block that just arrived, and for a sampler it is the block now playing:
```cpp
auto block = create_input_listener_buffer(0);
const auto& samples = block->get_data();
```
This is 512 numbers unless you changed the buffer size, and it is a new set every cycle. A block is very short, about ten milliseconds, so the measurements below are of the moment and not of a phrase.

A whole recording works the same way. `vega.read_audio()` returns the container, where you can turn on looping. Hooking it to the speakers gives back the buffers it plays through, one for each channel:
```cpp
recording->set_looping(true);
auto block = get_io_manager()->hook_audio_container_to_buffers(recording)[0];
```
That buffer is the block that is playing right now, a new one every cycle, exactly like the microphone's. The spine uses it, and every other block in this section would take it in the same one line, because they only ever read `block->get_data()`.

`Kinesis::Discrete` measures a list of numbers. Each function takes the list, a number of windows, a hop and a window size, and gives one value per window. To measure the whole block as one window, ask for one window with the hop and the window both equal to its length:
```cpp
const auto n = static_cast<uint32_t>(samples.size());
const double loud = Kinesis::Discrete::rms(samples, 1, n, n)[0];
```
What each measurement says about the moment:

- `rms` and `peak`: how loud. `peak` is the largest single sample and reacts to a click. `rms` is the steady level
- `low_frequency_energy(samples, 1, n, n, 0.02)`: how much sound there is in the lowest two percent of the spectrum. For 48000 samples a second and a block of 512, that is below about 400 Hz. The size of the number depends on how loud the sound is, which is why the blocks that use it take its square root and divide by a guess
- `spectral_energy`: all of the spectrum together
- `zero_crossing_rate`: how often the sound crosses zero, as a fraction of the block. It is high for a noisy or bright sound and low for a dull one
- `estimate_frequency(samples, sample_rate, fallback, threshold)`: a pitch, from the same zero crossings. It is cheap and reads a clean tone well. A rich voice has many crossings that are not the pitch, so it wanders. It gives `fallback` when the block is too short or has too few crossings
- `kurtosis`: how peaked the values are. A steady sine gives about -1.5, noise gives about 0, and a block holding a click gives a number many times larger. It is how the pot, the membrane and the spine tell a smooth sound from a sharp one
- `skewness`, `entropy`, `variance`: more ways to say how the values are spread. `entropy` is higher the more evenly spread they are

Two are worth leaving alone for a block this short:

- `dynamic_range` is the loudest sample over the quietest, in decibels. A block of real sound nearly always holds a sample very close to zero, so the answer is always huge and tells you nothing
- `onset_positions(samples, window, hop, threshold)` finds where the spectrum jumps, but it scales its result by its own largest jump. Even a block of room noise has a largest jump, so it always finds an onset. Gate it by loudness. A simpler way to call a block a strike is to ask whether it is loud and more than twice the slow average of the level

These are the same measurements the granular section sorted by. There they were taken over a whole recording. Here they are taken over the moment.

The block can come from anywhere that gives an audio buffer. The microphone, the sampler of a recording, and the sampler of a granular stream all do, and the shape does not know which. Change the one line that makes `block`.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: A Model Is Many Parts" >}}

A model is a set of parts, and each part has a transform that places it. A `Model` takes a list of components, each a shape of its own:
```cpp
auto net = std::make_shared<Nodes::Network::MeshNetwork>();

vega.mint(StructureConfig::Model {
    .network = net,
    .components = {
        { .expression = trunk },
        { .expression = branch, .parent_component = 0 },
    },
    .render = { .target_window = window },
});
```
Passing an empty network of your own is how you keep hold of the parts afterwards. Component `i` becomes slot `i` of the network.

A component names an earlier one as its parent. A part's place in the world is its parent's place times its own, worked out for you every frame:
```cpp
auto& slot = net->get_slot(1);
slot.local_transform = glm::rotate(glm::mat4(1.0F), 0.3F, glm::vec3(0.0F, 0.0F, 1.0F));
slot.dirty = true;
```
Two things to know. `local_transform` is relative to the parent, so turning the trunk turns the branch with it, and turning an upper arm carries the forearm and the hand. And nothing shows until you set `dirty = true`, which is what tells the graphics card to take the new transforms.

A part turns about its own origin. So build each limb hanging from its joint, with the joint at the origin of its shape. The figure's `limb` helper does this: the two points of the tube run from `0, 0, 0` down to `0, -length, 0`, and the part is placed with a translation to the joint on its parent. A positive length hangs down, and a negative one rises, which is how the torso and the head are made.

The figure has one slot for each part, numbered in the order they were listed: torso, head, left arm, right arm, left leg, right leg. Its `joint` helper builds a transform from a place and three angles, and `glm::quat(angles)` turns the angles into a rotation.

Adding a forearm is one more component whose parent is the arm, and one more `joint` line that places it at the end of the arm. Turn the arm and the forearm turns with it.

A loaded file is the same thing. `vega.read_mesh_network("path/to/model.fbx") | Graphics` gives a network with one slot for each mesh in the file, and the same `slots()` and `dirty` apply. A part of a loaded model turns about the origin of the model, so a part that sits away from the middle swings round it. The pictures that came with the file are already on the parts.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 10: Weight and Memory" >}}

Setting a shape directly from a number makes it twitch with the number and stop with it. A body, or a piece of clay, does not do that. Four ways to let a sound push without dictating.

**Follow slowly.** The shape moves a fraction of the way to where the sound says, every frame. A fraction of 0.1 is slow and 0.5 is quick:
```cpp
wheel->height += (target - wheel->height) * 0.2F;
```
The hand of the pot and the thread of the spine use this. It is a one pole filter, and it is the simplest weight there is.

**Be kicked.** A strike starts a motion, and the motion plays out by itself. The whip of the figure is this. It needs no spring code, because a function of the time since the strike already is a spring:
```cpp
auto whip = vega.Phasor(2.0F) >> vega.Polynomial([](double x) { return std::exp(-4.0 * x) * std::sin(18.0 * x); });
```
The clock restarts the function on every cycle, so the arm is thrown, overshoots, swings back and settles, and is thrown again. A bigger number inside `exp` is a tighter spring, and a smaller one keeps it swinging.
**Change the speed, not the place.** The sound does not set where the body is, only how fast it moves. A clock can be speeded up while it runs:
```cpp
auto beat = vega.Phasor(2.0F);
auto thump = beat >> vega.Polynomial(decay);
raised->on_change_to(true, [beat](const auto&) { beat->set_frequency(3.0F); });
```
So a body keeps its rhythm and the sound only pushes it faster or slower. A shape that moves at a rate set by the sound looks nothing like one that sits at a place set by it.

**Keep a table.** The sound writes into a table and a rule slowly forgets it. This is memory:
```cpp
for (auto& cell : *clay) {
    cell *= 0.998F;
}
```
The shape is built from the table every frame. The pot uses this, and so does the membrane below, where the table is a wave that spreads by itself. Everything the sound has written is still there, softening, for as long as the forgetting takes.

Which of these a mapping needs is the question to ask of each side of a sound. A strike wants a kick, a pitch wants a place, a bass wants a speed, and a gesture that should outlast the sound wants a table.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 11: One Shape, Many Times" >}}

Instances draw one shape many times in a single call, each with its own transform:
```cpp
auto choir = vega.mint(StructureConfig::Instances {
    .prototype_expression = vase_shape,
    .transforms = places,
    .render = { .target_window = window, .fragment_shader = "choir.frag" },
});

auto net = choir->get_network();
net->get_slot(2).transform = glm::translate(glm::mat4(1.0F), glm::vec3(0.0F, 1.0F, 0.0F));
net->get_slot(2).dirty = true;
```
Every copy shares the one shape and holds only a transform, so twelve copies or twelve thousand copies cost one shape. The transform is a full matrix, so a copy can be moved, turned and stretched.

Give each copy its own side of the sound. Put the measurements in a list and let copy `i` read item `i` modulo the length of the list:
```cpp
const std::array<float, 4> felt { loud, bass, sharp, pitch };
net->get_slot(i).transform = places[i] * glm::rotate(glm::mat4(1.0F), 3.0F * felt[i % 4], glm::vec3(0.0F, 1.0F, 0.0F));
net->get_slot(i).dirty = true;
```
A ring of copies then shows the character of the sound as a ring, where each position turns to a different aspect of it. The same list works on the slots of a loaded model.

When the copies should not all be the same shape, use a Model. When you want many and want them to move by a rule, an `InstanceFieldOperator` binds a function to each copy with `bind_transform`.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 12: Keeping a Shape" >}}

Whatever a shape has become can be saved. The pot does it on a key:
```cpp
#include "MayaFlux/IO/Model/ModelExport.hpp"

if (!IO::save_mesh_snapshot(node, "pot_{}.gltf")) {
    MF_PRINT("pot: export failed");
}
```
The `{}` in the name is replaced by the time in milliseconds, so every press gives a new file and none is overwritten. It saves the shape as it is at that moment, and the shape is a regular `gltf` file that Blender and other programs open.

The same call takes a mint buffer, a loaded model's parts, or a writer node. For a model, each part is saved where it currently stands, so a figure that has been dancing is saved in the pose it was in.

Sound made it, and a file keeps it.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### A Membrane That Keeps Ringing

`data/shaders/skin.frag`
```glsl
#version 460

layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_world_pos;

layout(location = 0) out vec4 out_color;

void main()
{
    vec3 n = normalize(cross(dFdx(in_world_pos), dFdy(in_world_pos)));
    float light = abs(dot(n, normalize(vec3(-0.4, 0.9, 0.3))));
    float h = clamp(in_world_pos.y * 0.8 + 0.5, 0.0, 1.0);
    vec3 colour = mix(vec3(0.05, 0.10, 0.22), vec3(0.95, 0.85, 0.70), h);
    out_color = vec4(colour * (0.2 + 0.8 * light), 1.0);
}
```
```cpp
#include "MayaFlux/IO/Model/ModelExport.hpp"

void compose() {
    auto window = create_window({ "A Membrane That Keeps Ringing", 1280, 800 });
    window->show();

    auto block = create_input_listener_buffer(0);

    constexpr int size = 64;
    auto height = std::make_shared<std::vector<float>>(size * size, 0.0F);
    auto speed = std::make_shared<std::vector<float>>(size * size, 0.0F);

    auto node = std::make_shared<MeshWriterNode>(size * size);
    node->set_mesh(Kinesis::generate_explicit_surface(*height, size, size, { 4.0F, 4.0F }, 1.0F));

    auto skin = vega.mint(StructureConfig::Object {
        .writer = node,
        .render = { .target_window = window, .fragment_shader = "skin.frag" },
    });

    bind_orbit_preset(window);

    schedule_metro(1.0 / 60.0, [node, block, height, speed]() {
        const auto& samples = block->get_data();
        const auto n = static_cast<uint32_t>(samples.size());
        if (n < 16) {
            return;
        }

        const auto loud = static_cast<float>(Kinesis::Discrete::rms(samples, 1, n, n)[0]);
        const auto sharp = static_cast<float>(Kinesis::Discrete::kurtosis(samples, 1, n, n)[0]);
        const auto pitch = static_cast<float>(
            Kinesis::Discrete::estimate_frequency(samples, Config::get_sample_rate(), 0.0, 0.0));

        if (loud > 0.01F && pitch > 0.0F) {
            const float across = std::clamp(std::log2(pitch / 110.0F) / 4.0F, 0.0F, 1.0F);
            const float depth = std::clamp((sharp + 1.5F) / 8.0F, 0.0F, 1.0F);
            const int cx = 3 + static_cast<int>(across * (size - 7));
            const int cy = 3 + static_cast<int>(depth * (size - 7));

            for (int dy = -2; dy <= 2; ++dy) {
                for (int dx = -2; dx <= 2; ++dx) {
                    const float falloff = std::exp(-0.5F * static_cast<float>(dx * dx + dy * dy));
                    (*speed)[(cy + dy) * size + cx + dx] += loud * 1.0F * falloff;
                }
            }
        }

        auto& h = *height;
        auto& v = *speed;

        for (int y = 1; y < size - 1; ++y) {
            for (int x = 1; x < size - 1; ++x) {
                const int k = y * size + x;
                const float curve = h[k - 1] + h[k + 1] + h[k - size] + h[k + size] - 4.0F * h[k];
                v[k] = (v[k] + 0.35F * curve) * 0.995F;
            }
        }
        for (int k = 0; k < size * size; ++k) {
            h[k] += v[k];
        }

        node->set_mesh(Kinesis::generate_explicit_surface(h, size, size, { 4.0F, 4.0F }, 1.0F));
    }, "membrane", Vruta::ProcessingToken::FRAME_ACCURATE);

    on_key_pressed(window, IO::Keys::F, [node]() {
        if (!IO::save_mesh_snapshot(node, "membrane_{}.gltf")) {
            MF_PRINT("membrane: export failed");
        }
    });
}
```
Run it. A flat skin lies in the window. Make a sound and it is struck, and the sound only strikes it:

- **How high the sound is** picks where along the skin it lands, low at the left and high at the right
- **How sharp the sound is** picks how far back it lands, a smooth tone near you and a click at the far edge
- **How loud the sound is** sets how hard the blow is

Whatever lands makes rings that spread, reach the edge, come back, and cross the rings from the other blows. After the sound has stopped the skin is still moving, and a loud clap at one spot leaves a pattern that takes seconds to die. The surface is a map of what kind of sound it was, and also a record of it.

Each frame, every point is pulled toward the average of its four neighbours, which is what makes a ring travel. The sound only adds speed to a few points.

Press **F** while it rings and you have the ripple frozen as an object.

Change one thing at a time:

- **`0.995F` to `0.9995F`:** it rings for a long time, and the patterns pile up
- **`0.35F` to `0.1F`:** slow, heavy waves. Do not raise it above `0.5F`, or the skin blows up
- **`1.0F` to `4.0F` in the blow:** a tall surface from soft sounds
- **`size = 64` to `128`:** a finer skin, four times the work each frame
- **Swap `across` and `depth`:** pitch lands front to back, and sharpness left to right

### A Spine of Time

`data/shaders/ember.frag`
```glsl
#version 460

layout(location = 0) in vec3 in_color;
layout(location = 1) in vec2 in_uv;
layout(location = 2) in vec3 in_world_pos;

layout(location = 0) out vec4 out_color;

void main()
{
    vec3 n = normalize(cross(dFdx(in_world_pos), dFdy(in_world_pos)));
    float light = abs(dot(n, normalize(vec3(0.5, 0.8, 0.6))));
    float age = in_uv.y;
    vec3 old = vec3(0.10, 0.14, 0.35);
    vec3 now = vec3(1.0, 0.72, 0.35);
    vec3 colour = mix(old, now, age * age);
    out_color = vec4(colour * (0.25 + 0.75 * light), 1.0);
}
```
```cpp
void compose() {
    auto window = create_window({ "A Spine of Time", 1280, 800 });
    window->show();

    auto recording = vega.read_audio();
    if (!recording) {
        return;
    }

    recording->set_looping(true);
    auto block = get_io_manager()->hook_audio_container_to_buffers(recording)[0];

    constexpr size_t points = 120;
    auto path = std::make_shared<std::vector<glm::vec3>>(points, glm::vec3(0.0F));
    auto loudness = std::make_shared<std::vector<double>>(points, 0.0);

    auto thread = [path, loudness] {
        const auto taper = Kinesis::TimeMaps::piecewise_linear(*loudness, 1.0);
        return Kinesis::generate_tube(
            std::span(*path),
            [&taper](float t) { return 0.02F + 0.2F * static_cast<float>(taper(t)); },
            12, true);
    };

    auto node = std::make_shared<MeshWriterNode>(points * 13 + 2);
    node->set_mesh(thread());

    auto spine = vega.mint(StructureConfig::Object {
        .writer = node,
        .render = { .target_window = window, .fragment_shader = "ember.frag" },
    });

    bind_orbit_preset(window);

    schedule_metro(1.0 / 60.0, [node, block, path, loudness, thread]() {
        const auto& samples = block->get_data();
        const auto n = static_cast<uint32_t>(samples.size());
        if (n < 16) {
            return;
        }

        const auto loud = static_cast<float>(Kinesis::Discrete::rms(samples, 1, n, n)[0]);
        const auto bass = static_cast<float>(Kinesis::Discrete::low_frequency_energy(samples, 1, n, n, 0.02)[0]);
        const auto sharp = static_cast<float>(Kinesis::Discrete::kurtosis(samples, 1, n, n)[0]);
        const auto pitch = static_cast<float>(
            Kinesis::Discrete::estimate_frequency(samples, Config::get_sample_rate(), 0.0, 0.0));
        const bool heard = loud > 0.01F;

        const float across = heard && pitch > 0.0F
            ? std::clamp(std::log2(pitch / 110.0F) / 4.0F, 0.0F, 1.0F)
            : 0.5F;
        const float depth = heard ? std::clamp((sharp + 1.5F) / 8.0F, 0.0F, 1.0F) : 0.5F;
        const float weight = heard ? std::min(1.0F, std::sqrt(bass) / 16.0F) : 0.5F;

        const glm::vec3 target(3.0F * (across - 0.5F), 3.0F * (depth - 0.5F), 3.0F * (weight - 0.5F));
        const glm::vec3 last = path->back();

        std::rotate(path->begin(), path->begin() + 1, path->end());
        path->back() = last + (target - last) * 0.25F;

        std::rotate(loudness->begin(), loudness->begin() + 1, loudness->end());
        loudness->back() = std::min(1.0, static_cast<double>(loud) * 4.0);

        node->set_mesh(thread());
    }, "spine", Vruta::ProcessingToken::FRAME_ACCURATE);
}
```
Run it. A dialog opens, and you choose a recording. It plays on a loop through the speakers, and the block that is playing right now is the one the thread listens to. A short thread grows in the window as the sound goes, about two seconds long, with the newest end hot and the oldest end cold. Its place in space is the character of the sound: pitch is left and right, sharpness is up and down, and how much low sound there is is forward and back. It is thick where the sound was loud and a wire where it was quiet.

This is a picture of what the sound was like, not of its waveform. A passage that holds one note is a short line, a rising scale is a long sweep to the right, and speech is a tangle that goes back on itself at every syllable. The same passage twice draws nearly the same loop twice. In a silence the thread shrinks into the middle.

`recording->set_looping(true)` is what keeps it going when the file ends. The hook gives back the buffer the file is playing through, one for each channel, and `[0]` is the first. From there it is the same kind of buffer as the microphone's, and the measurements are the same.

Change one thing at a time:

- **`0.25F` to `0.6F`:** the thread follows the sound jerkily. `0.05F` is languid, and it only draws the long shape of a phrase
- **`points = 120` to `300`:** a longer past, at a higher cost every frame
- **`3.0F` to `6.0F` in `target`:** a thread that fills the room
- **`12` to `4` sides:** a ribbon of a few flat faces, hard-edged

### What You Achieved

You have:

- Plucked a string from a clock and danced to the same clock, chained a clock into a function, and let events pick the notes and the arm that leads
- Pressed a pot with a voice, so the pitch picked where, the loudness picked how deep, and the sharpness picked the shape of the hand
- Struck a skin that goes on ringing after the sound has stopped
- Drawn the character of a sound as a thread through space
- Kept any of these as a file

Nothing here drew a shape by hand. A shape is a list of points and a list of triangles, a generator made the list from a rule, and the sound was one of the inputs to the rule. Each block heard a different side of the sound and let the shape have its own weight and memory. In the next sections the form fills space as a volume, and a table of numbers becomes a sound and a shape.

{{< /tutorial-detail >}}
