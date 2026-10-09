---
build:
  list: never
---

### The Next Step

A picture is a grid of numbers, and a shader is the small program that decides, for every pixel, which of those numbers to read. Move the place it reads from and the picture moves, bends or tears. Nothing in the picture file changes.

A sound is also a list of numbers. Hand that list to the shader and the sound decides where each pixel looks. The picture is a still, and the sound is what moves it.

Each block below is a complete `compose()`, with the shader that goes with it. Save each shader in the `data/shaders` folder of your project, under the name written above it. The code asks for it by that file name alone. The program looks for `data/shaders` in the folder you run it from, and one folder up, so run it from the project folder or its build folder. If the window stays blank or never appears, the shader was not found: look in the terminal for an error about failing to read the shader, and check the folder and the name. A file dialog opens for the picture, and another for the sound.

### Rows of Sound
`data/shaders/audio_rows.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2D texSampler;

layout(set = 1, binding = 0) readonly buffer Audio {
    float samples[];
};

void main()
{
    int n = samples.length();
    if (n == 0) {
        outColor = texture(texSampler, fragTexCoord);
        return;
    }

    int row = clamp(int(fragTexCoord.y * float(n)), 0, n - 1);

    vec2 uv = fragTexCoord;
    uv.x += samples[row] * 0.25;

    outColor = texture(texSampler, uv);
}
```
```cpp
void compose() {
    auto window = create_window({ "Rows of Sound", 1280, 720 });
    window->show();

    auto picture = vega.read_image() | Graphics;
    picture->setup_rendering({ .target_window = window, .fragment_shader = "audio_rows.frag" });

    auto sampler = store(create_sampler());
    sampler->play_continuous(0, sampler->slice_from_stream().with_looping(true));

    Buffers::ShaderConfig config { "audio_rows.frag" };
    config.bindings["audio_data"] = Buffers::ShaderBinding(1, 0, Portal::Graphics::DescriptorRole::STORAGE);

    auto reader = create_processor<Buffers::DescriptorBindingsProcessor>(picture, config);
    reader->bind_audio_buffer("audio", sampler->get_buffer(), "audio_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
}
```
Run this code. The sound plays and the picture is pulled sideways by it. The top of the picture reads the start of the sound's current block, the bottom reads the end, so the whole waveform of the moment is drawn as a ripple running down the picture. Loud passages tear it apart, and in silence it settles back to exactly the picture you chose.

{{< tutorial-detail title="Also: Your Own Voice" >}}

The sound can be live. Replace the sampler lines with a listener on the microphone, and bind that instead:
```cpp
auto microphone = create_input_listener_buffer(0);
reader->bind_audio_buffer("audio", microphone, "audio_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
```
Speak and the picture bends as you speak. Input has to be switched on before the program starts, which Expansion 8 of the first section shows how to do without recompiling.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Also: A Film or a Camera as the Picture" >}}

The picture can move as well. Replace the first two picture lines with a film or a camera. `| Graphics` hooks each one to a buffer:
```cpp
auto film = get_io_manager()->load_video("path/to/film.mkv") | Graphics;
auto picture = get_associated_buffer(film);
picture->setup_rendering({ .target_window = window, .fragment_shader = "audio_rows.frag" });
```
```cpp
auto camera = vega.read_camera() | Graphics;
auto picture = get_associated_buffer(camera);
picture->setup_rendering({ .target_window = window, .fragment_shader = "audio_rows.frag" });
```
The film plays once. The camera asks which camera and which mode when it starts, and runs for as long as the program does.

{{< /tutorial-detail >}}

### Loudness as Weather
`data/shaders/ripple.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2D texSampler;

layout(push_constant) uniform Push {
    float level;
};

void main()
{
    vec2 centre = fragTexCoord - 0.5;
    float dist = length(centre);
    float angle = atan(centre.y, centre.x);

    float swell = level * 2.0;
    float r = dist + swell * 0.08 * sin(dist * 40.0);
    float theta = angle + swell * sin(dist * 8.0);
    float split = swell * 0.05;

    vec2 uv_r = 0.5 + r * vec2(cos(theta - split), sin(theta - split));
    vec2 uv_g = 0.5 + r * vec2(cos(theta), sin(theta));
    vec2 uv_b = 0.5 + r * vec2(cos(theta + split), sin(theta + split));

    outColor = vec4(
        texture(texSampler, uv_r).r,
        texture(texSampler, uv_g).g,
        texture(texSampler, uv_b).b,
        1.0);
}
```
```cpp
struct Push {
    float level;
};

void compose() {
    auto window = create_window({ "Loudness as Weather", 1280, 720 });
    window->show();

    auto picture = vega.read_image() | Graphics;
    picture->setup_rendering({ .target_window = window, .fragment_shader = "ripple.frag" });

    auto sampler = store(create_sampler());
    sampler->play_continuous(0, sampler->slice_from_stream().with_looping(true));

    auto block = sampler->get_buffer();
    auto renderer = picture->get_render_processor();
    renderer->set_push_constant_size<Push>();
    renderer->feed("level", [block]() {
        const auto& samples = block->get_data();
        double sum = 0.0;
        for (const double value : samples) {
            sum += std::abs(value);
        }
        return sum / std::max<size_t>(1, samples.size());
    }, offsetof(Push, level), sizeof(float));
}
```
Run this code. This time the shader gets one number per frame instead of the whole sound: the average level of the current block. The picture is twisted round its centre and its colours are pulled apart, as far as the sound is loud. Quiet passages leave it nearly still, and loud ones swirl it.

You have, in both cases:

- A picture that a shader draws
- A sound that reaches the shader, whole or as one number
- A picture that only moves while the sound does

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: What the Shader Does, and Where It Lives" >}}

A fragment shader runs once for every pixel of the window. It receives `fragTexCoord`, the position of that pixel across the picture from 0 to 1, and it returns a colour. The line that matters is:
```glsl
outColor = texture(texSampler, uv);
```
`texture` reads the picture at `uv`. If `uv` is `fragTexCoord`, you get the picture as it is. Add something to `uv` before the read and each pixel borrows its colour from a different place.

The file name in `.fragment_shader` is looked up when the program starts. A name ending in `.frag` is compiled from source then, with no extra build step, and a name ending in `.spv` is loaded as already compiled. The search looks beside where you run the program in `shaders` and `data/shaders`, and the same two names one folder up. A project made from the Weave template keeps its shaders in `data/shaders`. A name that is found nowhere is logged as an error and the shader is not built, which is how a blank window usually starts.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Drawing the Picture" >}}

`vega.read_image()` returns a picture that is loaded but not yet on screen. `| Graphics` registers it, and `setup_rendering` says where and how to draw it. Both calls matter, in this order:
```cpp
auto picture = vega.read_image() | Graphics;
picture->setup_rendering({ .target_window = window, .fragment_shader = "audio_rows.frag" });
```
`target_window` is the window to draw into. `fragment_shader` replaces the default shader, which draws the picture as it is. Leave it out and you have the first section again.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: The Sound as a Storage Buffer" >}}

A storage buffer is a list of numbers that the shader can read at any position. `bind_audio_buffer` makes one from a block of sound:
```cpp
Buffers::ShaderConfig config { "audio_rows.frag" };
config.bindings["audio_data"] = Buffers::ShaderBinding(1, 0, Portal::Graphics::DescriptorRole::STORAGE);

auto reader = create_processor<Buffers::DescriptorBindingsProcessor>(picture, config);
reader->bind_audio_buffer("audio", sampler->get_buffer(), "audio_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
```
Three names are in play, and they are easy to mix up:

- `"audio_data"` is the key in `config.bindings`. It declares which set and binding the shader reads, here set 1 and binding 0, and the shader says the same: `layout(set = 1, binding = 0)`
- `"audio"` is the name you give this binding, in case you want to remove it later
- Set 0 belongs to the picture itself. Your own data goes in set 1

Every time the picture is drawn, the latest block of the sound is converted from doubles to floats and copied to the GPU. The shader asks the buffer how long it is with `samples.length()`, and `readonly` tells it that it will not write. The block is the buffer size of the engine, 512 samples unless you changed it.

`DescriptorRole::STORAGE` suits a list that can be any length. `DescriptorRole::UNIFORM` is for a small fixed block of numbers.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Where the Block of Sound Comes From" >}}

`bind_audio_buffer` takes any audio buffer. The sampler's buffer, `sampler->get_buffer()`, is the block it is playing right now. A microphone listener, `create_input_listener_buffer(0)`, is the block that just arrived. The shader does not know the difference.

A network can be bound too, with `bind_network`. A resonator, a bell or a string that you have sent to the speakers with `| Audio` already produces a block of sound every cycle, and that block goes to the shader:
```cpp
reader->bind_network("bell", bell, "audio_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
```
A network that produces neither sound nor geometry cannot be bound, and `bind_network` says so in the log.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: One Number per Frame" >}}

A push constant is a handful of numbers sent with every draw, small and fast. The shader declares them, and your program declares the same layout:
```glsl
layout(push_constant) uniform Push {
    float level;
};
```
```cpp
struct Push {
    float level;
};

renderer->set_push_constant_size<Push>();
renderer->feed("level", source, offsetof(Push, level), sizeof(float));
```
`feed` takes a function that is called once every frame. Whatever it returns is written at `offsetof(Push, level)`. Return a `double` and it is stored as a float. Only four or eight bytes are written, so each field gets its own `feed`:
```cpp
renderer->feed("level", [block]() { return mean_level(block); }, offsetof(Push, level), sizeof(float));
renderer->feed("tilt", [state]() { return state->tilt; }, offsetof(Push, tilt), sizeof(float));
```
Anything that can be a number works as the source: the level of a block, a value a key press changed, the output of a node. Use push constants for a few numbers, and a storage buffer when the shader needs the whole shape.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Three Ways to Read the Sound" >}}

The C++ is the same in every case, and the shader decides what the sound means. Three readings:

**By position.** The picture's height is the block's time. This is the first block:
```glsl
int row = clamp(int(fragTexCoord.y * float(n)), 0, n - 1);
uv.x += samples[row] * 0.25;
```

**A few points as strengths.** Take eight evenly spaced samples and let each set the strength of one sine wave across the picture, a slow wave to a fast one. The sound's shape becomes a bend with eight layers:
```glsl
for (int i = 0; i < 8; ++i) {
    float freq = float(i + 1) * 3.14159;
    shift += samples[i * (n / 8)] * sin(uv.y * freq + float(i));
}
```

**Cells with a mind of their own.** Cut the picture into a grid. Give each cell its own offset from a hash of its position, and let the sound decide how far the offsets reach and how fine the grid is:
```glsl
float grid_n = mix(4.0, 14.0, clamp(energy, 0.0, 1.0));
vec2 cell = floor(uv * grid_n);
vec2 cell_rand = hash22(cell) * 2.0 - 1.0;
uv += cell_rand * (0.02 + abs(cell_energy) * 0.12);
```
`hash22` is a small function that turns a cell's position into two numbers that look random. A fuller version of this shader also tears some cells off to random places when a strike arrives.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: Rewriting the Picture on the GPU" >}}

Everything above leaves the picture alone and changes where it is read. A compute shader can instead write a new picture. This one blends a stack of pictures into one, with a weight for each:
```cpp
#include "MayaFlux/Yantra/Executors/TextureExecutionContext.hpp"

using namespace MayaFlux::Yantra;
using namespace MayaFlux::Portal::Graphics;

void compose() {
    auto window = create_window({ "Morph", 1280, 720 });
    window->show();

    auto a = IO::ImageReader::load(std::string("path/to/a.png"), 4);
    auto b = IO::ImageReader::load(std::string("path/to/b.png"), 4);
    auto c = IO::ImageReader::load(std::string("path/to/c.png"), 4);

    const uint32_t width = a->width;
    const uint32_t height = a->height;
    const size_t layer_bytes = a->byte_size();

    std::vector<uint8_t> stack(layer_bytes * 3);
    std::memcpy(stack.data(), a->data(), layer_bytes);
    std::memcpy(stack.data() + layer_bytes, b->data(), layer_bytes);
    std::memcpy(stack.data() + layer_bytes * 2, c->data(), layer_bytes);

    auto& loom = Portal::Graphics::get_texture_manager();
    auto array = loom.create_2d_array(width, height, 3, ImageFormat::RGBA8, stack.data());

    auto layers = std::make_shared<Kakshya::TextureContainer>(width, height, ImageFormat::RGBA8, 3);
    layers->from_image_array(array);

    struct MorphPC {
        uint32_t layer_count;
        uint32_t normalise;
        uint32_t layer_offset;
        uint32_t pad;
    };

    const std::vector<GpuBufferBinding> weights_binding = {
        { .set = 0, .binding = 2, .direction = GpuBufferBinding::Direction::INPUT, .element_type = GpuBufferBinding::ElementType::FLOAT32 },
    };

    auto morph = std::make_shared<TextureExecutionContext>(
        GpuComputeConfig { "texture_array_morph.comp", { 16, 16, 1 }, sizeof(MorphPC) },
        ImageFormat::RGBA8, TextureExecutionContext::OutputMode::CONTAINER, 1, weights_binding);

    morph->set_push_constants(MorphPC { .layer_count = 3, .normalise = 1, .layer_offset = 0, .pad = 0 });
    morph->set_binding_data(2, std::vector<float> { 1.0F, 0.2F, 0.6F });

    DataIO input;
    input.container = std::static_pointer_cast<Kakshya::SignalSourceContainer>(layers);
    morph->execute(input, {});

    auto result = vega.TextureBuffer(width, height, ImageFormat::RGBA8) | Graphics;
    result->setup_rendering({ .target_window = window });
    result->set_gpu_texture(morph->get_output_image(0));
}
```
The three pictures must have the same size. The shader reads one weight per layer from a list of floats and divides by their total, so the weights are proportions and not fixed amounts. Here the first picture is strongest, the third is at about half of it, and the second is faint. Change the numbers and run again to see a different blend.

The weights are just a list of numbers. They could as easily be the energy of a few bands of a sound as three constants.

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

### Throw the Paint
`data/shaders/splash.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2D texSampler;

layout(push_constant) uniform Splash {
    float level;
    float x;
    float y;
};

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

void main()
{
    const float grid = 32.0;
    vec2 origin = vec2(x, y);
    vec2 cell = floor(fragTexCoord * grid);
    vec2 centre = (cell + 0.5) / grid;

    float away = length(centre - origin);
    float reach = level * 6.0;
    float torn = step(hash21(cell + 91.7), clamp(reach - away, 0.0, 1.0));

    vec2 outward = normalize(centre - origin + vec2(0.0001, 0.00013));
    vec2 flung = outward * hash21(cell + 5.0) * 0.5 * reach;
    vec2 scatter = (hash22(cell + 271.8) - 0.5) * 0.2 * reach;

    vec2 uv = fragTexCoord - torn * (flung + scatter);
    outColor = texture(texSampler, uv);
}
```
```cpp
struct Splash {
    float level;
    float x;
    float y;
};

void compose() {
    auto window = create_window({ "Throw the Paint", 1280, 720 });
    window->show();

    auto picture = vega.read_image() | Graphics;
    picture->setup_rendering({ .target_window = window, .fragment_shader = "splash.frag" });

    auto bell = vega.ModalNetwork(8, 110.0, ModalNetwork::Spectrum::INHARMONIC, 4.0) | Audio[{ 0, 1 }];
    auto origin = std::make_shared<glm::vec2>(0.5F, 0.5F);

    auto renderer = picture->get_render_processor();
    renderer->set_push_constant_size<Splash>();
    renderer->feed("level", [bell]() {
        const auto block = bell->get_audio_buffer();
        if (!block) {
            return 0.0;
        }
        double sum = 0.0;
        for (const double value : *block) {
            sum += std::abs(value);
        }
        return sum / std::max<size_t>(1, block->size());
    }, offsetof(Splash, level), sizeof(float));
    renderer->feed("x", [origin]() { return static_cast<double>(origin->x); }, offsetof(Splash, x), sizeof(float));
    renderer->feed("y", [origin]() { return static_cast<double>(origin->y); }, offsetof(Splash, y), sizeof(float));

    on_mouse_pressed(window, IO::MouseButtons::Left, [bell, origin, window](double x, double y) {
        const auto ndc = normalize_coords(x, y, window);
        *origin = glm::vec2((ndc.x + 1.0F) * 0.5F, (ndc.y + 1.0F) * 0.5F);
        bell->set_fundamental(get_uniform_random(60.0, 400.0));
        bell->excite(get_uniform_random(0.5, 1.0));
    });
}
```
Run it. The picture sits still and silent. Click, and a bell of a new pitch is struck, and a splash is thrown from where you clicked. The picture is cut into thirty-two by thirty-two cells, and the loudness of the bell decides how far the splash reaches. Inside that reach, cells are torn from where they were and flung outward from the click, each by its own distance and in its own slightly different direction. As the bell rings down, the splash reaches less and less, and the cells fall back into the picture.

Nothing is painted and nothing accumulates. The cells are the picture's own, thrown by the sound. If the splash starts from the wrong side of the window, change `ndc.y + 1.0F` to `1.0F - ndc.y`.

Then change one thing at a time:

- **`32.0` to `10.0` in the shader:** a few large slabs of picture, thrown like boards
- **`32.0` to `120.0`:** fine spatter, like flicked droplets
- **`4.0` to `12.0`:** the bell rings much longer, and the splash lingers
- **`* 6.0` to `* 2.0`:** a smaller reach for the same sound

### The Red Room
`data/shaders/red_room.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2D texSampler;

layout(push_constant) uniform Push {
    float level;
};

void main()
{
    float swell = clamp(level * 4.0, 0.0, 1.0);

    vec2 uv = fragTexCoord;
    float zigzag = abs(fract(uv.y * 10.0) - 0.5) * 2.0;
    uv.x += (zigzag - 0.5) * swell * 0.2;

    vec3 near = texture(texSampler, uv).rgb;
    vec3 ghost = texture(texSampler, uv + vec2(swell * 0.06, 0.0)).rgb;
    vec3 colour = mix(near, ghost, 0.5 * swell);

    float grey = dot(colour, vec3(0.299, 0.587, 0.114));
    vec3 dull = vec3(grey) * 0.35;
    vec3 flooded = colour * vec3(1.4, 0.35, 0.3);

    outColor = vec4(mix(dull, flooded, swell), 1.0);
}
```
Use the C++ of Loudness as Weather, with `ripple.frag` changed to `red_room.frag` in `setup_rendering`. Choose a low, slow recording: a drone, a room tone, a voice far away.

Run it. In the quiet the picture is drained to a dull grey and darkened, and it waits. When the sound arrives the picture floods red and green and blue drain out of it, a second image slides out of the first, and every band of the picture shears sideways in a zigzag, like a floor of chevrons. When the sound drops away it goes back to grey, and it does not feel the same as before.

Change `4.0` to `10.0` and a soft sound is enough to flood it. Change `10.0` in `uv.y * 10.0` to `3.0` for broad, slow chevrons, or `40.0` for a tight, nervous flicker.

### Left Ear, Right Ear
`data/shaders/two_axes.frag`
```glsl
#version 460

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(set = 0, binding = 1) uniform sampler2D texSampler;

layout(set = 1, binding = 0) readonly buffer Left {
    float ch0[];
};

layout(set = 1, binding = 1) readonly buffer Right {
    float ch1[];
};

void main()
{
    int nl = ch0.length();
    int nr = ch1.length();
    if (nl == 0 || nr == 0) {
        outColor = texture(texSampler, fragTexCoord);
        return;
    }

    vec2 uv = fragTexCoord;
    uv.x += ch0[clamp(int(fragTexCoord.y * float(nl)), 0, nl - 1)] * 0.25;
    uv.y += ch1[clamp(int(fragTexCoord.x * float(nr)), 0, nr - 1)] * 0.25;

    outColor = texture(texSampler, uv);
}
```
```cpp
void compose() {
    auto window = create_window({ "Left Ear, Right Ear", 1280, 720 });
    window->show();

    auto picture = vega.read_image() | Graphics;
    picture->setup_rendering({ .target_window = window, .fragment_shader = "two_axes.frag" });

    auto samplers = create_samplers(48000 * 10);
    store(samplers);
    if (samplers.size() < 2) {
        return;
    }

    for (const auto& sampler : samplers) {
        sampler->play_continuous(0, sampler->slice_from_stream().with_looping(true));
    }

    Buffers::ShaderConfig config { "two_axes.frag" };
    config.bindings["left_data"] = Buffers::ShaderBinding(1, 0, Portal::Graphics::DescriptorRole::STORAGE);
    config.bindings["right_data"] = Buffers::ShaderBinding(1, 1, Portal::Graphics::DescriptorRole::STORAGE);

    auto reader = create_processor<Buffers::DescriptorBindingsProcessor>(picture, config);
    reader->bind_audio_buffer("left", samplers[0]->get_buffer(), "left_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
    reader->bind_audio_buffer("right", samplers[1]->get_buffer(), "right_data", 1, Portal::Graphics::DescriptorRole::STORAGE);
}
```
Run it with a stereo recording. Each channel of the file gets its own sampler, and each sampler's block goes to its own storage buffer. The left channel pushes the rows sideways and the right channel pushes the columns up and down. A sound that sits left bends the picture one way, and a sound that sits right bends it the other. A sound that moves between the ears moves the distortion from one axis to the other.

### What You Achieved

You have:

- Drawn a picture with a shader of your own
- Sent a sound to the shader as a whole list, and as a single number
- Read the same sound three ways, by position, as a few strengths and cell by cell
- Thrown a picture's cells outward from a click with a bell, and drained and flooded one with a drone
- Bent a picture differently on two axes with the two ears of a recording
- Seen a compute shader write a new picture from a stack of pictures and a list of weights

Nothing here changed the picture or the sound. The shader was the meeting point, and the sound decided how the picture was read. In the next section many pictures are stacked at once, stills, films and moments of the past, and a shader decides what the pile becomes.

{{< /tutorial-detail >}}
