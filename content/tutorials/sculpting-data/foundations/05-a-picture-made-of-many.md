---
build:
  list: never
---

### The Next Step

A picture is a grid of numbers. A stack of pictures is a much bigger block of numbers: a still, a film, a moment from a minute ago, all held at once as the layers of a single texture. The shader can read any layer at any pixel, so it can lay them out like stones, sort them, or let each corner of the picture live in a different moment.

A Chimera is what fills that stack. You say what each layer is made of, a photograph, a film, a moment from the past, and the shader says what the pile means.

Every block below needs one small helper, which lists the pictures in a folder. Paste it above `compose()` in each file, together with the include:
```cpp
#include "MayaFlux/Kakshya/Source/DynamicVideoStream.hpp"

inline std::vector<std::string> pictures(const std::string& folder, size_t limit)
{
    std::vector<std::string> found;
    for (const auto& entry : std::filesystem::directory_iterator(folder)) {
        const auto extension = entry.path().extension().string();
        if (extension == ".png" || extension == ".jpg") {
            found.push_back(entry.path().string());
        }
    }
    std::sort(found.begin(), found.end());
    found.resize(std::min(found.size(), limit));
    return found;
}
```
Save each shader in the `data/shaders` folder of your project, under the name written above it. The code asks for it by that file name alone. The program looks for `data/shaders` in the folder you run it from, and one folder up, so run it from the project folder or its build folder. If the window stays blank or never appears, the shader was not found: look in the terminal for an error about failing to read the shader, and check the folder and the name. The stills are any folder of photographs you like, written as `path/to/stills`.

### A Pile of Pictures

```cpp
void compose() {
    auto window = create_window({ "A Pile of Pictures", 1024, 1024 });
    window->show();

    const auto paths = pictures("path/to/stills", 8);
    const auto count = paths.size();

    auto builder = create_chimera(
        { .width = 512, .height = 512, .ring_frames = count },
        { .target_window = window },
        Portal::Graphics::FitMode::COVER);

    for (const auto& path : paths) {
        builder.layer().from(path);
    }

    auto clock = std::make_shared<double>(0.0);

    builder.every(1.0 / 60.0, [clock, count](Kriya::Chimera& set) {
        *clock += 1.0 / 60.0;
        for (size_t i = 0; i < count; ++i) {
            const double wave = 0.5 + 0.5 * std::sin(*clock * 0.35 + static_cast<double>(i) * 2.1);
            set.level(i, 0.02 + std::pow(wave, 6.0));
        }
    });

    store(builder.start());
}
```
Run this code. All eight pictures are in the window at once, as layers of one texture, and each breathes at its own pace. At any moment two or three of them are strong and the rest are a faint trace, so you see double and triple exposures, a face over a street over a field, and then one of them sinks away and another rises out of the mix. It never settles on one picture.

The default shader does the mixing. It takes a mean of all the layers, weighted by each layer's `level`. Nothing else is happening: eight pictures, and eight numbers that change sixty times a second. The next blocks replace the shader, so that the pile means something other than a mix.

### Stones in a Ring
`data/shaders/stones.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2DArray textureArray;

layout(set = 1, binding = 0) readonly buffer LayerData {
    vec4 layerData[];
};

const uint MAX_LAYERS = 30u;

layout(push_constant) uniform PC {
    uint layer_count;
    uint mode;
    float weights[MAX_LAYERS];
} pc;

void main()
{
    vec3 colour = vec3(0.93, 0.91, 0.86);
    uint count = min(pc.layer_count, MAX_LAYERS);

    for (uint i = 0u; i < count; ++i) {
        float presence = pc.weights[i];
        if (presence <= 0.0) {
            continue;
        }

        vec4 params = layerData[2u * i];
        float radius = max(params.z, 0.0001);
        float c = cos(params.w);
        float s = sin(params.w);

        vec2 p = (fragTexCoord - params.xy) / radius;
        p = vec2(c * p.x + s * p.y, -s * p.x + c * p.y);

        float edge = 1.0 - smoothstep(0.7, 1.0, length(p));
        if (edge <= 0.0) {
            continue;
        }

        vec4 stone = texture(textureArray, vec3(p * 0.5 + 0.5, float(i)));
        colour = mix(colour, stone.rgb, edge * stone.a * presence);
    }

    outColor = vec4(colour, 1.0);
}
```
```cpp
void compose() {
    auto window = create_window({ "Stones in a Ring", 1024, 1024 });
    window->show();

    const auto paths = pictures("path/to/stills", 24);
    const auto stones = static_cast<uint32_t>(paths.size());

    auto builder = create_chimera(
        { .width = 512, .height = 512, .ring_frames = stones },
        { .target_window = window, .fragment_shader = "stones.frag" },
        Portal::Graphics::FitMode::COVER);

    builder.layer_data();

    for (uint32_t i = 0; i < stones; ++i) {
        const float angle = static_cast<float>(i) * 2.4F;
        const float reach = 0.09F * std::sqrt(static_cast<float>(i) + 0.5F);

        builder.layer().from(paths[i]).level(0.0)
            .params({ 0.5F + reach * std::cos(angle), 0.5F + reach * std::sin(angle), 0.07F, angle });
    }

    auto laid = std::make_shared<uint32_t>(0);
    auto presence = std::make_shared<std::vector<double>>(stones, 0.0);

    builder.every(1.0 / 60.0, [laid, presence](Kriya::Chimera& set) {
        for (size_t i = 0; i < presence->size(); ++i) {
            double& value = (*presence)[i];
            value = i < *laid ? std::min(value + 0.2, 1.0) : std::max(value - 0.01, 0.0);
            set.level(i, value);
        }
    });

    store(builder.start());

    auto pulse = vega.Impulse(2.0F) | Audio[0];
    pulse->exclude_output_from_graph();

    pulse->on_impulse([laid, stones](const auto&) {
        *laid = (*laid + 1) % (stones + 6);
    });
}
```
Run this code. A silent pulse every half second, and with each pulse a photograph is laid on a warm ground, each as a soft disc, spiralling outward from the middle the way seeds sit in a flower. They overlap like leaves. When the last stone is down the pulse goes on for six more beats, and then the tide takes them: every stone fades out slowly, and the laying starts again from the centre.

Each photograph is cropped to a disc by the shader, so pictures without a transparent background work. Where a stone lies and how big it is comes from `params`, one four-number set per layer. How visible it is comes from `level`.

{{< tutorial-detail title="Also: A Node That Only Keeps Time" >}}

The pulse is an `Impulse`, a node, so it needs to be processed to fire. Sending it to `Audio` does that, and it would also put its click in the speakers. `exclude_output_from_graph()` keeps the processing and leaves the click out: the node runs every sample and calls its hooks, but its output is not added to the sound.
```cpp
auto pulse = vega.Impulse(2.0F) | Audio[0];
pulse->exclude_output_from_graph();
```
Leave the line out and the stones are laid to an audible click. Any node can be used this way, a `Logic` gate, a `Random`, a slow `Sine`, when you want what it does and not what it sounds like.

{{< /tutorial-detail >}}

Then change one thing at a time:

- **`2.4F` to `0.5F`:** the stones fall in tight curling arms instead of a sunflower
- **`0.07F` to `0.14F`:** large stones that bury each other
- **`0.01` to `0.002`:** a very slow tide, so the next laying begins before the last has gone
- **`Impulse(2.0F)` to `Impulse(0.4F)`:** one stone every two and a half seconds

You have, in both blocks:

- A folder of pictures, each one a layer of the same texture
- A weight for each layer, changing as it runs
- A shader that decides how the layers are read
- Numbers per layer, placement and visibility, that you can change while it runs

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: A Layer Stack Is One Texture" >}}

An array texture is a texture with many layers of the same size. The shader reads one with a third coordinate, the layer:
```glsl
texture(textureArray, vec3(uv, float(layer)))
```
`create_chimera` makes the array and a Chimera to fill it:
```cpp
auto builder = create_chimera(
    { .width = 512, .height = 512, .ring_frames = 8 },
    { .target_window = window },
    Portal::Graphics::FitMode::CONTAIN);
```
The size is the size of every layer. `ring_frames` is the name the picture streams use for their length, and here it is the number of layers. Declaring more layers than that is an error and nothing starts. Declaring fewer is allowed, but the shader is told the array has eight, and the ones you never filled read as empty black. That is why the helper counts the pictures first and uses the count.

Layers are numbered in the order you declare them with `layer()`. The shipped push constants carry weights for the first thirty, so thirty is the practical limit for shaders that use `level`.

The shader is set through the render config, like any other:
```cpp
{ .target_window = window, .fragment_shader = "stones.frag" }
```
A name ending in `.frag` is compiled from source when the program starts, and one ending in `.frag.spv` is loaded already compiled.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Where Layers Come From" >}}

After `layer()`, one call says what fills it:
```cpp
builder.layer().from("path/to/still.png");
builder.layer().from(image_data);
builder.layer().from(gpu_image);
builder.layer().from(ring);
builder.layer().from_pipeline();
```
A path is loaded once, now, as colour with transparency. An `ImageData` is a picture you already have in memory. A GPU image is copied into the layer every frame, so a layer can follow something that another shader is writing. A ring is a film or camera being recorded, and `from_pipeline()` means the ring that the builder's own pipeline records, which the film blocks below use.

When a picture is not the size of the layer, a fit decides how it lands. The third argument of `create_chimera` sets it for every layer, and `.fit(...)` sets it for one:
```cpp
builder.layer().from("path/to/wide.png").fit(Portal::Graphics::FitMode::COVER);
```
- `STRETCH` squeezes the whole picture onto the layer
- `CONTAIN` keeps its shape and centres it, the rest of the layer stays transparent black
- `COVER` keeps its shape and fills the layer, cropping the picture
- `CENTER` does not scale it, and clips what does not fit
- `TILE` and `TILE_MIRRORED` repeat it at its own size

If a picture is a different size and no fit is given anywhere, the layer is not filled, and nothing is logged. When a layer stays empty, check the sizes first.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: The Ring and What It Costs" >}}

A film or a camera is brought in by recording it into a ring, a fixed number of frames that always holds the most recent ones. Layers then read moments out of the ring:
```cpp
builder.record(Kriya::BufferOperation::capture_to_stream(
    get_io_manager(), "path/to/film.mkv",
    IO::VideoLoadConfig { .target_width = 640, .target_height = 360 }, 600));
```
```cpp
builder.record(Kriya::BufferOperation::capture_to_stream(
    get_io_manager(), vega.read_camera(), 600));
```
The camera asks which camera and which mode, and the ring is sized from the mode you choose. The last number is the ring length in frames. The ring is filled once for every frame the window draws, 60 a second, whatever the film's own rate, so 600 frames is ten seconds. Each frame is read back from the GPU, and each ring layer is copied up to the GPU again every frame.

**What it costs.** A frame is its width times its height times four bytes, and the ring keeps all of them in memory:

| Size | One frame | A 600 frame ring (10 s) |
|------|-----------|-------------------------|
| 640 by 360 | 0.9 MB | 550 MB |
| 640 by 480 | 1.2 MB | 740 MB |
| 1280 by 720 | 3.7 MB | 2.2 GB |
| 1920 by 1080 | 8.3 MB | 5 GB |

Every layer costs a frame of video memory too, and every ring layer is uploaded every frame, which is a frame's worth of bytes times the layer count times sixty each second. The examples here use 640 by 360 and ten seconds so that they start on modest hardware. If you have a strong graphics card and plenty of memory, make the size and the ring larger, and give `target_width` and `target_height` the same numbers as the Chimera's width and height. `every_n_frames(2)` on a layer refreshes it every other frame and halves its cost.

A film plays once. When it ends the ring stops growing, so every layer that reads the past keeps whatever moment it was on. The camera runs for as long as the program does.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: The Verbs of a Ring Layer" >}}

Each call after `layer().from_pipeline()` says how that layer moves through the ring:
```cpp
builder.layer().from_pipeline().lag(2.0);
builder.layer().from_pipeline().lag(Kinesis::TimeMaps::triangle(0.5, 6.0, 0.4, 0.0)).smooth();
builder.layer().from_pipeline().speed(0.0).enters_after(5.0);
builder.layer().from_pipeline().speed(-0.5);
```
- `lag(seconds)` reads that many seconds behind the newest frame, never closer than two frames
- `lag(map)` lets the lag change in time. The map takes seconds since the layer entered and returns a lag in seconds. The time maps of the second section work here
- `speed(ratio)` plays freely instead. The layer starts at the newest frame and moves from there. Below 1 it falls further behind, 1 or above keeps it at the newest frame, 0 holds it on that moment, and a negative number plays backward
- `smooth()` blends the two frames around a fractional position, so a slow or changing lag does not step
- `enters_after(seconds)` keeps the layer empty until that long after the start
- `every_n_frames(n)` refreshes the layer every n frames

A layer that reaches a moment the ring no longer holds keeps its last frame and says so once.

While it runs, `cut` jumps a layer to another moment:
```cpp
set.cut(2, 4.0);
```
That moves layer 2 to four seconds behind the newest frame. A lagged layer carries on following its lag from there, and a free layer carries on playing.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: What the Shader Receives" >}}

Every array shader declares the same three things:
```glsl
layout(set = 0, binding = 1) uniform sampler2DArray textureArray;

layout(set = 1, binding = 0) readonly buffer LayerData {
    vec4 layerData[];
};

layout(push_constant) uniform PC {
    uint layer_count;
    uint mode;
    float weights[30];
} pc;
```
- `textureArray` is the stack
- `pc.layer_count` is how many layers the array holds, and `pc.weights[i]` is the `level` of layer i, which starts at 1
- `pc.mode` is a number only the shipped shaders use, and your own shader can ignore it
- `layerData` holds two `vec4` for each layer: `layerData[2 * i]` is its params, four numbers you set, and `layerData[2 * i + 1]` is its timing, which the Chimera writes

The timing has `x` as the seconds since the layer entered, `y` as how many frames behind the newest frame it reads, and `w` as 1 once it has been filled. A shader can use `w` to leave out layers that have not arrived.

`layerData` exists only when you ask for it. `builder.layer_data()` creates it, and so does giving any layer `params`. Call it before `start()`, and leave the block out of a shader that does not use it.

Nothing here has a meaning of its own. The weights and params are numbers. The stones shader reads weights as how visible a stone is and params as where it lies. The emergence shader below reads one param as a dial.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Knobs, Controls and Staying Alive" >}}

The running Chimera has controls, and they are the way to change the stack while it runs:
```cpp
set.level(3, 0.5);
set.params(3, glm::vec4(0.5F, 0.5F, 0.1F, 0.0F));
set.speed(2, -1.0);
set.cut(2, 4.0);
set.set(1, other_picture);
set.stop();
```
`level` and `params` write the numbers the shader reads. `speed` and `cut` move a layer through its ring. `set` gives a layer a new still, ring or GPU image. `stop` holds every layer on the frame it is on.

A knob does not have to belong to a layer. The emergence and paint blocks use `set.params(0, ...)` as one dial for the whole picture and read it as `layerData[0].x`.

The place to call the controls from is `every`:
```cpp
builder.every(1.0 / 60.0, [](Kriya::Chimera& set) {
});
```
It runs your function on the frame clock, once per interval, and the shortest interval is one frame. Anything that should reach the picture, a decay, a sweep, a value from a sound, goes through it.

The Chimera runs as long as a copy of it exists. `store(builder.start())` keeps one alive for the life of the program.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: Driving a Chimera From Sound" >}}

Sound is one way of deciding when and how far, not the only one. A node is a stream of numbers that can make a sound, and it can also say when something happens. These hooks fire as the node runs:

- an `Impulse` calls `on_impulse` at each pulse
- a `Logic` gate calls `on_change_to(true, ...)` and `on_change_to(false, ...)` when it opens and closes, `while_true` for as long as it is open, and `on_tick_if(condition, ...)` when a condition holds
- a `Counter` calls `on_increment`, `on_wrap` and `on_count(target, ...)`

A node that you send to `Audio` runs with the sound, and its hooks run there too. If you want the node's timing and not its sound, call `exclude_output_from_graph()` on it: it is processed as usual and its output is left out of the mix. The Chimera runs with the picture. The simple, safe way to join them is a plain value that both can see:
```cpp
auto laid = std::make_shared<uint32_t>(0);

pulse->on_impulse([laid](const auto&) { ++*laid; });

builder.every(1.0 / 60.0, [laid](Kriya::Chimera& set) {
    set.level(0, *laid > 0 ? 1.0 : 0.0);
});
```
The hook writes the number, and `every` reads it on the next frame and applies it. The Stones block is this, once per stone. The time block does it with a layer number and a distance.

A continuous number works the same way. The Paint Time block measures the loudness of the sound it is playing, and the sound of a bell, a string or a resonator that you have sent to the speakers can be measured in just the same way, with `get_audio_buffer()`.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: Emergence by Order" >}}

Put many pictures of the same kind of thing on top of each other and the thing they have in common is what holds still. Mixing, as the first block does, lets every picture leave a trace, and the traces cloud each other. Choosing does better. At every pixel the shader collects the layers' values, sorts them, and picks one by its rank: the lowest is the darkest of all the pictures at that pixel, the middle is the typical one, the highest is the brightest.

A shader of your own does this. `emerge.frag` below picks any rank with a dial, and can scatter the rank from grain to grain.

The default shader has modes too, chosen when the array is made with `create_texture_array`, and then handed to a Chimera:
```cpp
auto array = create_texture_array(
    { .width = 512, .height = 512, .ring_frames = paths.size() },
    { .target_window = window },
    Portal::Graphics::FitMode::COVER, 2);

auto builder = create_chimera(array);
```
The last number is the mode: 0 the mean weighted by each layer's `level`, 1 the sum, 2 the highest value at each pixel, 3 the layers laid over each other in order. `create_chimera(array)` takes an array that is already drawing, so you do not give it a render config again.

The sort is done for every pixel, every frame, so a lot of layers cost a lot. Thirty is the most the shipped weights cover.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: A Layer From the GPU" >}}

A layer can follow an image that a shader is writing. The compute morph of the previous section made one:
```cpp
builder.layer().from(morph->get_output_image(0)).every_n_frames(2);
```
The picture the morph wrote is copied into that layer every second frame. A layer fed this way sits in the stack like any other, so the same shader can sort it, place it or fade it together with stills and films.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### Emergence
`data/shaders/emerge.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2DArray textureArray;

layout(set = 1, binding = 0) readonly buffer LayerData {
    vec4 layerData[];
};

const uint MAX_LAYERS = 30u;

layout(push_constant) uniform PC {
    uint layer_count;
    uint mode;
    float weights[MAX_LAYERS];
} pc;

float hash21(vec2 p)
{
    p = fract(p * vec2(123.34, 456.21));
    p += dot(p, p + 45.32);
    return fract(p.x * p.y);
}

vec2 hash22(vec2 p)
{
    return vec2(hash21(p), hash21(p + 17.0));
}

float grain_scatter(vec2 uv, float density, float seed)
{
    vec2 p = uv * density;
    vec2 cell = floor(p);
    float nearest = 8.0;
    vec2 owner = cell;

    for (int y = -1; y <= 1; ++y) {
        for (int x = -1; x <= 1; ++x) {
            vec2 c = cell + vec2(float(x), float(y));
            float d = distance(p, c + hash22(c));
            if (d < nearest) {
                nearest = d;
                owner = c;
            }
        }
    }

    return hash21(owner + vec2(seed, seed * 0.37)) - 0.5;
}

void sort_values(inout float v[MAX_LAYERS], uint n)
{
    for (uint i = 1u; i < n; ++i) {
        float key = v[i];
        int j = int(i) - 1;
        while (j >= 0 && v[j] > key) {
            v[j + 1] = v[j];
            --j;
        }
        v[j + 1] = key;
    }
}

void main()
{
    float r[MAX_LAYERS];
    float g[MAX_LAYERS];
    float b[MAX_LAYERS];
    uint n = min(pc.layer_count, MAX_LAYERS);

    for (uint i = 0u; i < n; ++i) {
        vec3 c = texture(textureArray, vec3(fragTexCoord, float(i))).rgb;
        r[i] = c.r;
        g[i] = c.g;
        b[i] = c.b;
    }

    sort_values(r, n);
    sort_values(g, n);
    sort_values(b, n);

    vec4 knobs = layerData[0];
    float scatter = grain_scatter(fragTexCoord, 90.0, floor(knobs.z * 8.0) * 7.31);

    float pick = clamp(knobs.x + scatter * knobs.y, 0.0, 1.0) * float(n - 1u);
    uint lower = uint(floor(pick));
    uint upper = min(lower + 1u, n - 1u);
    float f = fract(pick);

    outColor = vec4(
        mix(r[lower], r[upper], f),
        mix(g[lower], g[upper], f),
        mix(b[lower], b[upper], f),
        1.0);
}
```
```cpp
void compose() {
    auto window = create_window({ "Emergence", 1024, 1024 });
    window->show();

    const auto paths = pictures("path/to/stills", 30);

    auto builder = create_chimera(
        { .width = 512, .height = 512, .ring_frames = paths.size() },
        { .target_window = window, .fragment_shader = "emerge.frag" },
        Portal::Graphics::FitMode::COVER);

    builder.layer_data();

    for (const auto& path : paths) {
        builder.layer().from(path);
    }

    auto clock = std::make_shared<double>(0.0);

    builder.every(1.0 / 60.0, [clock](Kriya::Chimera& set) {
        *clock += 1.0 / 60.0;
        const double t = *clock;
        const double dial = 0.5 + 0.5 * (0.6 * std::sin(t * 1.3) + 0.4 * std::sin(t * 3.7 + 1.0));
        const double spread = 0.5 + 0.5 * std::sin(t * 0.9 + 2.0);
        set.params(0, glm::vec4(static_cast<float>(dial), static_cast<float>(spread), static_cast<float>(t), 0.0F));
    });

    store(builder.start());
}
```
Use twenty to thirty photographs of the same kind of thing: windows, clouds, faces, a harbour at dawn, the same street on different days.

Run it. Two knobs move fast and out of step with each other. The first is the dial, which swings between the darkest of all your pictures and the brightest in a couple of seconds, with a second, quicker wobble on top, so it never repeats the same sweep. The second is the spread. The picture is scattered into a few thousand small, irregular grains, and at every moment each grain is given its own push away from the dial, drawn fresh eight times a second. When the spread is low the whole picture moves together. When it is high, neighbouring grains pick pictures of different ranks, and the window becomes a flickering dust where one grain shows the darkest picture, the next the typical one and the next the brightest.

What the pictures have in common keeps flashing out of the noise and being lost again. Nothing in the stack is blended. At each pixel one picture wins, and which one changes many times a second.

Change one thing at a time:

- **`1.3` and `3.7` to `0.2` and `0.5`:** the same piece, slowed to a drift
- **`t * 0.9 + 2.0` to a constant `0.0`:** no spread, the picture moves as one and the middle is clean
- **`90.0` in the shader to `300.0`:** the grains shrink to fine dust, like static
- **`90.0` to `20.0`:** coarse flakes, like torn paper
- **`8.0` in the shader to `1.0`:** the grains change once a second and hold their picture between
- **Fewer pictures, six:** the ranks become coarse, and the pictures show through as separate wins

{{< tutorial-detail title="Also: A Film Instead of Stills" >}}

Replace the stills with moments of a film. Each layer reads the film a little further back, so each pixel sorts the last nine seconds of what was there:
```cpp
auto builder = create_chimera(
    { .width = 640, .height = 360, .ring_frames = 24 },
    { .target_window = window, .fragment_shader = "emerge.frag" },
    Portal::Graphics::FitMode::STRETCH);

builder.layer_data();
builder.record(Kriya::BufferOperation::capture_to_stream(
    get_io_manager(), "path/to/film.mkv",
    IO::VideoLoadConfig { .target_width = 640, .target_height = 360 }, 600));

for (int i = 0; i < 24; ++i) {
    builder.layer().from_pipeline().lag(0.4 * i).every_n_frames(2);
}
```
Point it at a film of a street, a crowd or a room. The median of nine seconds is what stays put, and people who only pass through are gone, and at the extremes of the dial they leave trails, dark or light. To hold the clean middle, set the dial to a constant `0.5` and the spread to `0.0`. This is 24 ring layers refreshed every other frame at 640 by 360. A ring of 600 frames takes about 550 MB, so make the picture and the ring larger if your machine can carry it.

{{< /tutorial-detail >}}

### Time Is Not One Place
`data/shaders/uncanny.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2DArray textureArray;

layout(set = 1, binding = 0) readonly buffer LayerData {
    vec4 layerData[];
};

const uint MAX_LAYERS = 30u;

layout(push_constant) uniform PC {
    uint layer_count;
    uint mode;
    float weights[MAX_LAYERS];
} pc;

float hash21(vec2 p)
{
    p = fract(p * vec2(123.34, 456.21));
    p += dot(p, p + 45.32);
    return fract(p.x * p.y);
}

float value_noise(vec2 p)
{
    vec2 cell = floor(p);
    vec2 f = fract(p);
    f = f * f * (3.0 - 2.0 * f);
    return mix(
        mix(hash21(cell), hash21(cell + vec2(1.0, 0.0)), f.x),
        mix(hash21(cell + vec2(0.0, 1.0)), hash21(cell + vec2(1.0, 1.0)), f.x),
        f.y);
}

void main()
{
    float phase = layerData[0].x;
    vec2 p = fragTexCoord * vec2(3.0, 2.0);

    float n = value_noise(p + vec2(phase * 0.20, -phase * 0.13)) * 0.65
        + value_noise(p * 2.3 - vec2(phase * 0.07)) * 0.35;

    uint count = min(pc.layer_count, MAX_LAYERS);
    float pos = clamp((n - 0.25) / 0.5, 0.0, 0.999) * float(count);
    uint layer = min(uint(floor(pos)), count - 1u);

    if (layerData[2u * layer + 1u].w < 0.5) {
        layer = 0u;
    }

    float across = fract(pos);
    float seam = smoothstep(0.0, 0.05, across) * (1.0 - smoothstep(0.95, 1.0, across));

    vec3 colour = texture(textureArray, vec3(fragTexCoord, float(layer))).rgb;
    outColor = vec4(colour * mix(0.3, 1.0, seam), 1.0);
}
```
```cpp
#include "MayaFlux/Kinesis/Tendency/TimeMap.hpp"

void compose() {
    auto window = create_window({ "Time Is Not One Place", 1280, 720 });
    window->show();

    auto builder = create_chimera(
        { .width = 640, .height = 360, .ring_frames = 6 },
        { .target_window = window, .fragment_shader = "uncanny.frag" },
        Portal::Graphics::FitMode::STRETCH);

    builder.layer_data();
    builder.record(Kriya::BufferOperation::capture_to_stream(
        get_io_manager(), "path/to/film.mkv",
        IO::VideoLoadConfig { .target_width = 640, .target_height = 360 }, 600));

    builder.layer().from_pipeline().lag(0.0);
    builder.layer().from_pipeline().lag(3.0);
    builder.layer().from_pipeline().speed(0.0).enters_after(5.0);
    builder.layer().from_pipeline().lag(Kinesis::TimeMaps::triangle(0.5, 6.0, 0.4, 0.0)).smooth();
    builder.layer().from_pipeline().speed(-0.5).enters_after(2.0);
    builder.layer().from_pipeline().lag(6.0).smooth();

    auto phase = std::make_shared<double>(0.0);
    auto next = std::make_shared<std::pair<int, double>>(-1, 0.0);

    builder.every(1.0 / 60.0, [phase, next](Kriya::Chimera& set) {
        *phase += 1.0 / 60.0;
        if (next->first >= 0) {
            set.cut(static_cast<size_t>(next->first), next->second);
            next->first = -1;
            *phase += 2.5;
        }
        set.params(0, glm::vec4(static_cast<float>(*phase), 0.0F, 0.0F, 0.0F));
    });

    store(builder.start());

    auto tick = vega.Impulse(0.37F) | Audio[0];
    tick->exclude_output_from_graph();

    tick->on_impulse([next](const auto&) {
        next->first = static_cast<int>(get_uniform_random(1.0, 6.0));
        next->second = get_uniform_random(0.5, 8.0);
    });
}
```
Run it with a film that has a person, a room or a street in it.

Six layers read the same film at six different times: now, three seconds ago, a moment held still, a lag that wanders between half a second and six, one that plays backward at half speed, and one six seconds back. A slow field of noise decides, pixel by pixel, which of the six you see, so the picture is made of patches of different times with a dark seam between them. The patches drift. Someone's arm is where it is now, and the chair beside it is where it was.

Every few seconds the clock ticks, and one of the layers, chosen at random, is thrown to some other moment of the last eight seconds. The patch that belonged to it changes, and the whole field lurches. The room has reset in one place.

{{< tutorial-detail title="Memory and Size" >}}

This holds a ten second ring of 640 by 360 frames, about 550 MB, and uploads the six ring layers every frame. The size is modest on purpose. If your machine has the graphics memory and the processing for it, make the Chimera's width and height, the film's `target_width` and `target_height` and the ring length larger. A 1280 by 720 ring of 600 frames is 2.2 GB. Expansion 3 has the numbers.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Also: A Camera" >}}

A film plays once, and then every layer keeps the moment it was on. A camera does not end. Replace the recording line with:
```cpp
builder.record(Kriya::BufferOperation::capture_to_stream(
    get_io_manager(), vega.read_camera(), 600));
```
Choose the 640 x 480 mode when asked, to keep the ring modest. The Chimera stretches the picture to its own size.
Walk in front of it. You are in six places at six times, and every few seconds one of them is somewhere else. To skip the pickers, give `vega.read_camera` a `CameraConfig` with the device name, which Expansion 16 of the first section explains.

{{< /tutorial-detail >}}

Change one thing at a time:

- **`Impulse(0.37F)` to `Impulse(1.3F)`:** a nervous room that cannot settle
- **`2.5` in the phase to `0.0`:** the cuts change the layers but not the map
- **`vec2(3.0, 2.0)` in the shader to `vec2(12.0, 8.0)`:** the patches become a fine mosaic of times
- **`speed(0.0)` to `speed(0.05)`:** the held layer creeps, and does not stay still

### Paint Time
`data/shaders/paint_time.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2DArray textureArray;

layout(set = 1, binding = 0) readonly buffer LayerData {
    vec4 layerData[];
};

const uint MAX_LAYERS = 30u;

layout(push_constant) uniform PC {
    uint layer_count;
    uint mode;
    float weights[MAX_LAYERS];
} pc;

void main()
{
    vec3 paint = texture(textureArray, vec3(fragTexCoord, 0.0)).rgb;
    float brightness = dot(paint, vec3(0.299, 0.587, 0.114));
    float depth = clamp(layerData[0].x, 0.0, 1.0);

    uint moments = min(pc.layer_count, MAX_LAYERS) - 1u;
    float pos = clamp(brightness * depth, 0.0, 1.0) * float(moments - 1u);
    uint lower = uint(floor(pos));
    uint upper = min(lower + 1u, moments - 1u);

    vec4 a = texture(textureArray, vec3(fragTexCoord, float(1u + lower)));
    vec4 b = texture(textureArray, vec3(fragTexCoord, float(1u + upper)));

    outColor = mix(a, b, fract(pos));
}
```
```cpp
void compose() {
    auto window = create_window({ "Paint Time", 1280, 720 });
    window->show();

    auto builder = create_chimera(
        { .width = 640, .height = 360, .ring_frames = 17 },
        { .target_window = window, .fragment_shader = "paint_time.frag" },
        Portal::Graphics::FitMode::STRETCH);

    builder.layer_data();
    builder.record(Kriya::BufferOperation::capture_to_stream(
        get_io_manager(), "path/to/film.mkv",
        IO::VideoLoadConfig { .target_width = 640, .target_height = 360 }, 600));

    builder.layer().from("path/to/painted.png").fit(Portal::Graphics::FitMode::COVER);

    for (int i = 0; i < 16; ++i) {
        builder.layer().from_pipeline().lag(0.25 * i).every_n_frames(2);
    }

    auto sampler = store(create_sampler());
    sampler->play_continuous(0, sampler->slice_from_stream().with_looping(true));

    auto block = sampler->get_buffer();
    auto depth = std::make_shared<double>(0.0);

    builder.every(1.0 / 60.0, [block, depth](Kriya::Chimera& set) {
        const auto& samples = block->get_data();
        double sum = 0.0;
        for (const double value : samples) {
            sum += std::abs(value);
        }
        const double level = sum / std::max<size_t>(1, samples.size());
        *depth += (level * 8.0 - *depth) * 0.1;
        set.params(0, glm::vec4(static_cast<float>(std::clamp(*depth, 0.0, 1.0)), 0.0F, 0.0F, 0.0F));
    });

    store(builder.start());
}
```
Before you run it, paint a picture. Any size, any program: a black background with white strokes, a grey wash, a gradient from one corner. This picture is never shown. It is a map of time.

Run it with a film. In the quiet, the painting does nothing and you see the film as it is now. As the sound gets louder, every pixel reads the past by as much as the painting is bright there: black stays in the present, and white reaches nearly four seconds back. A white stroke across a face lets the face live a few seconds late, so as someone crosses the picture the stroke drags behind them. A grey wash is a veil of the recent past. A gradient is a slow tear from now to then. When the sound drops, the picture closes back into the present.

The still and the film are two layers of one stack. The painting is read as numbers, never drawn, and sixteen moments of the film are what it picks between.

Change one thing at a time:

- **`8.0` to `2.0`:** only loud passages open the past
- **`0.25 * i` to `0.5 * i`:** the white reaches eight seconds back, but the ring must hold it: use a ring of 600 frames or more
- **`0.1` to `0.01`:** the depth follows the sound slowly and smears its changes
- **A painting with soft edges, not hard ones:** the past blends into the present like a glaze, a hard edge gives a cut

### What You Achieved

You have:

- Stacked a folder of photographs into one texture, and read them with a shader instead of a plain mix
- Laid stones that appear with each pulse of a node, placed by numbers you gave every layer
- Sorted a pile of pictures at each pixel, so what they share rose out of them
- Read one film at six different times at once, and thrown one of them somewhere else on a tick
- Painted a map of time, and let the loudness of a sound decide how far into it the film reaches
- Seen how many megabytes a ring costs, and where to make it larger

Nothing here changed the pictures. The layers were data, the shader was the meeting point, and what the pile meant was a few lines of GLSL. In the next section the data stops being a picture: a sound becomes the vertices of a shape.

{{< /tutorial-detail >}}
