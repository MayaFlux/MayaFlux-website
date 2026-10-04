---
title: "Part I : Foundations of Form"
layout: "tutorial"
---

{{< framing-card >}}

### How to Use This

Each section begins with a snippet -> run it first.\
Deep-dive panels are optional, and closed until you open them.\
End with **Try It → Recap** to consolidate.

### What You'll Learn

- How data from outside behaves as material
- Containers → Buffers
- Routing data by domain, quickly and by hand
- How a file becomes sound, picture and form

{{< /framing-card >}}

{{< tutorial-card step="1 of 9" title="Bring It In" open="true" >}}

### The Simplest First Step

Data comes from outside: a recording, a picture, a 3D model, a film. MayaFlux does not care where it came from. Bring it in, give it somewhere to go, and it plays or appears as it is.

Each block below is a complete `compose()` for your `src/user_project.hpp`. Pick one and run it. A file dialog opens: choose a file of the matching kind.

### Sound
```cpp
void compose() {
    vega.read_audio() | Audio;
}
```
Run this code. The file plays through your speakers.

To skip the dialog, give it a path: `vega.read_audio("path/to/file.wav")`.

{{< tutorial-detail title="Also: Sound from a Microphone" >}}

Live sound is data from outside too. First tell the engine to open your sound card's input, in `settings()`. Then listen to it in `compose()`:
```cpp
void settings() {
    Config::get_global_stream_info().input.enabled = true;
    Config::get_global_stream_info().input.channels = 1;
}

void compose() {
    create_input_listener_buffer(0, true);
}
```
What the microphone hears comes out of your speakers on the same channel. Use headphones, or the speakers will feed back into the microphone.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Also: Turning the Input On Without Recompiling" >}}

`settings()` is compiled into your program, so changing it means a rebuild. The same settings can live in a JSON file that is read every time the program starts. Create `mayaflux.json` in your project's source folder:
```json
{
  "stream": {
    "input": { "enabled": true, "channels": 1 }
  }
}
```
Run the program as before. The launcher picks the file up by itself, so `settings()` can stay empty and the microphone block above needs only its `compose()`.

Only the fields you write are changed. Everything else keeps its default. The keys are the names of the config structs, so any setting described in `docs/Settings.md` can go in the file. Do not confuse `"stream"` with its `"input"` inside, which is your sound card's input, with the top level `"input"` section, which is for MIDI, OSC and other devices.

To use a file from somewhere else, start the program with `--config path/to/file.json`.

If the file and `settings()` set the same value, `settings()` wins, because the file is read first. Start the program with `--config-override` to turn that around: the file is then read after `settings()`, and it replaces all of it. Anything the file does not mention goes back to its default.

{{< /tutorial-detail >}}

### Image
```cpp
void compose() {
    auto window = create_window({ .title = "Image", .width = 1280, .height = 720 });

    auto image = vega.read_image() | Graphics;
    image->setup_rendering({ .target_window = window });

    window->show();
}
```
Run this code. The image appears in a window.

Change `width` and `height` and run again: the same picture, a different window.

### Model
```cpp
void compose() {
    auto window = create_window({ .title = "Model", .width = 1280, .height = 720 });

    auto meshes = vega.read_mesh() | Graphics;
    for (auto& mesh : meshes) {
        mesh->setup_rendering({ .target_window = window });
    }

    window->show();
}
```
Run this code. The model is drawn into the window.

A model file can hold several meshes, so the loader gives you a group and you set up each one.

{{< tutorial-detail title="Also: Model as a Network of Parts" >}}

The same file, read so that its parts stay together as one network:
```cpp
void compose() {
    auto window = create_window({ .title = "Model parts", .width = 1280, .height = 720 });

    auto network = vega.read_mesh_network() | Graphics;
    auto form = vega.mint(StructureConfig::Model { .network = network, .render = { .target_window = window } });

    window->show();
}
```
It looks like the block above. The difference is underneath: each part of the model is a slot in one network, and each slot keeps its own transform, so parts can later be moved independently.

{{< /tutorial-detail >}}

### Video, with Sound
```cpp
void compose() {
    auto window = create_window({ .title = "Video", .width = 1280, .height = 720 });

    auto [video, audio] = choose_video({ .video_options = IO::VideoReadOptions::EXTRACT_AUDIO });

    auto picture = get_io_manager()->hook_video_container_to_buffer(video);
    picture->setup_rendering({ .target_window = window });
    audio | Audio;

    window->show();
}
```
Run this code. The picture plays in the window and the sound plays through your speakers.

### Video, Picture Only
```cpp
void compose() {
    auto window = create_window({ .title = "Video", .width = 1280, .height = 720 });

    auto [video, audio] = choose_video({});

    auto picture = get_io_manager()->hook_video_container_to_buffer(video);
    picture->setup_rendering({ .target_window = window });

    window->show();
}
```
The only difference from the block above is the empty `{}`. Without `EXTRACT_AUDIO` the sound is never decoded, so `audio` is empty and there is nothing to route.

{{< tutorial-detail title="Also: Picture from a Camera" >}}

A camera is a live video source. It has no file to browse to, so you name the device:
```cpp
void compose() {
    auto window = create_window({ .title = "Camera", .width = 1280, .height = 720 });

    auto camera = vega.read_camera({ .device_name = "/dev/video0" });
    camera->setup_rendering({ .target_window = window });

    window->show();
}
```
Your live picture appears in the window. `/dev/video0` is the first camera on Linux. On macOS use `"0"`, and on Windows use `"video=Integrated Camera"` or the name your camera reports.

{{< /tutorial-detail >}}

Each example assumes you pick a file. If you cancel the dialog, nothing loads and the lines that follow have nothing to work on.

You have, in each case:

- Data from outside, held in memory
- A domain it was routed to: audio for the sound, graphics for the picture
- A result you can hear or see after one run

{{< tutorial-detail title="Explanations" >}}

{{< tutorial-detail title="Expansion 1: What Is `vega`?" >}}

`vega` is a **fluent interface**: a convenience layer that takes MayaFlux's power and hides the verbosity without hiding the machinery.

Making complex logic less verbose is a good way to encourage more people to explore. But complexity that is presented well is not an obstacle. It is what gives you something to choose between, and choices are where agency and creativity come from. So `vega` shortens the path to the first result and leaves every door on that path open.

If you didn't have `vega`, loading a sound file would look like this:
```cpp
auto reader = std::make_shared<IO::SoundFileReader>();

if (!reader->can_read("path/to/file.wav")) {
    return;
}

reader->set_target_sample_rate(Config::get_sample_rate());
reader->set_audio_options(IO::AudioReadOptions::DEINTERLEAVE);

auto options = IO::FileReadOptions::EXTRACT_METADATA | IO::FileReadOptions::EXTRACT_REGIONS;
if (!reader->open("path/to/file.wav", options)) {
    return;
}

auto container = std::dynamic_pointer_cast<Kakshya::SoundFileContainer>(reader->create_container());

if (!reader->load_into_container(container)) {
    return;
}

container->set_memory_layout(Kakshya::MemoryLayout::ROW_MAJOR);

auto processor = std::dynamic_pointer_cast<Kakshya::ContiguousAccessProcessor>(container->get_default_processor());
if (!processor) {
    processor = std::make_shared<Kakshya::ContiguousAccessProcessor>();
    container->set_default_processor(processor);
}
processor->set_output_size({ Config::get_buffer_size(), container->get_num_channels() });
processor->set_auto_advance(true);
```
Depending on your exposure to programming, this can feel complex or liberating. Being explicit means you decide what is created, when, and with which settings.

`vega` says: "You just want to load a file? Say so."
```cpp
auto container = vega.read_audio("path/to/file.wav");
```
Same machinery underneath: the same reader, the same resampling to your project's sample rate, the same processor setup.

**What `vega` does:**

- Picks the reader from the file
- Applies sensible defaults
- Checks for errors
- Builds and configures the container
- Returns the real object

**What `vega` doesn't do:**

- Hide the idea. It removes the *verbosity*, not the *machinery*.
- Make the object less capable. What you get back is the full container.
- Remove your ability to do it explicitly. You can always write the long version when you need control.

The short syntax is convenience. The long syntax is control. MayaFlux gives you both.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Containers and Buffers" >}}

A **container** is a large collection of data held as a whole: every sample of a recording, every frame of a film. It knows its own shape and size, and it has a processor that knows how to hand out pieces of it.

A **buffer** holds one step of that, and the information needed to use it:

- For sound, a buffer holds one block of samples.
- For a picture, a buffer holds the data for one frame, plus the rendering information that goes with it: which shaders draw it, which window it goes to, and the fields those need, whether they apply to one frame or to all of them.

Containers are where material rests. Buffers are where it gets used.

This is why every example above has two parts: a loader that makes a container (or, for an image, a buffer that already holds its pixels), and a registration that gives a buffer somewhere to go.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: What Does `| Audio` or `| Graphics` Do?" >}}

The `|` operator is **quick registration**. It hands the object to a domain and gives you the same object back, so the line keeps working as a value:
```cpp
auto image = vega.read_image() | Graphics;
```
`Audio` and `Graphics` are domains. A domain tells the engine when and where to process the object: audio at the sound card's pace, graphics once per frame.

Registration is the part that is the same every time, so `|` does it for you. Written by hand:

- `container | Audio` is `get_io_manager()->hook_audio_container_to_buffers(container)`
- `buffer | Graphics` is `register_graphics_buffer(buffer)`
- `network | Graphics` is `register_node_network(network, Nodes::ProcessingToken::VISUAL_RATE)`, after switching the network to graphics output if it is not already

Whenever the choice is load bearing, you do it by hand. That is why the image and the model need a second line to choose a window, and why the video is wired by hand: one file can feed more than one domain, and only you know where each part should go.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Loading Is Separate from Routing" >}}

`vega.read_audio()` on its own does this:

- Opens the file and decodes it
- Resamples to your project's sample rate
- Builds a container with a processor attached
- Returns the container

It does **not** start playback, create buffers, or connect to your audio hardware. Try it:
```cpp
void compose() {
    auto container = vega.read_audio();
}
```
The file loads and nothing plays. The container sits in memory, and you decide what happens next: route it to your speakers with `| Audio`, or analyse it, or send its numbers somewhere else.

Loading is separate from routing. You can load a file and send it to hardware immediately, or spend the next twenty lines building something before it ever plays.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Sound, the Container and Its Processor" >}}

`vega.read_audio()` opened the dialog, then decoded the whole file into memory:

1.  Created a reader and checked that the file is readable
2.  Resampled to the engine's sample rate
3.  **Deinterleaved** the samples, so each channel is its own array
4.  Created a `SoundFileContainer` and loaded every sample into it
5.  Set the memory layout to row major
6.  Configured a `ContiguousAccessProcessor`: the container's default processor, which knows how to hand out the samples block by block

That processor does the work of access. It:

- Sets its output size to one buffer's worth of samples, for every channel
- Tracks where in the file you are
- Auto-advances: each time it is asked, it moves forward
- Gives channels separately, so each can be processed on its own

The processor is why a container is more than a data holder. It has built in logic for how it should be consumed.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Sound, Per-Channel Buffers" >}}

`| Audio` hooked the container to buffers. By hand:
```cpp
auto container = vega.read_audio();
auto buffers = get_io_manager()->hook_audio_container_to_buffers(container);
```
That call creates **one buffer per channel**, and the buffer manager owns them. Inside it is this loop:
```cpp
auto buffer_manager = get_buffer_manager();

for (uint32_t channel = 0; channel < container->get_num_channels(); ++channel) {
    auto buffer = buffer_manager->create_audio_buffer<Buffers::SoundContainerBuffer>(
        Buffers::ProcessingToken::AUDIO_BACKEND, channel, container, channel);
    buffer->initialize();
}
```
Step by step, for each channel:

1.  Create a `SoundContainerBuffer`, a buffer that reads from a container
2.  Tag it with `AUDIO_BACKEND`: it feeds the sound card at the audio pace
3.  Put it on the output channel with that number
4.  Tell it which channel of the container to read
5.  Initialize it

A stereo file's left channel feeds output 0 and its right channel feeds output 1. Channels stay separate so that each can later have its own processing.

Every audio cycle, each buffer asks the container's processor for the next block of samples and passes it on. Get the buffers back later with `get_io_manager()->get_audio_buffers(container)`.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 7: Sound, the Microphone" >}}

`input.enabled` and `input.channels` ask the engine to open the sound card's input before anything runs. The engine then keeps one input buffer per hardware channel and fills it every audio cycle with the samples that arrived.

`create_input_listener_buffer(0, true)` does this by hand:
```cpp
auto buffer = std::make_shared<Buffers::AudioBuffer>(0);
register_audio_buffer(buffer, 0);
read_from_audio_input(buffer, 0);
```
1.  Make an audio buffer for channel 0
2.  Register it on output channel 0, which is the `true`: it is why you hear it
3.  Register it as a listener of input channel 0

Each cycle the input buffer copies its samples into every listener. With `false` the listener exists and receives sound, but nothing sends it to the speakers.

If you leave input disabled there is no input channel to listen to, and the call logs an error that the channel is out of range.

This is the one sound example with no container. Live sound has no whole to hold: it exists one block at a time, and the buffer is all there is.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 8: Image, Pixels in a TextureBuffer" >}}

`vega.read_image()` decoded the file into pixels with four channels: red, green, blue and alpha. Common formats are eight bits per channel. An `.exr` file stays floating point. The pixels went into a `TextureBuffer`, which also holds a flat rectangle for the picture to be drawn on.

An image never becomes a container. It goes straight to a buffer, because a picture is one frame. By hand:
```cpp
IO::ImageReader reader;
reader.open("path/to/image.png");
auto image = reader.create_texture_buffer();
```
`| Graphics` is then `register_graphics_buffer(image)`. It initializes the buffer on the GPU and adds it to the buffers the engine processes each frame.

At that point the buffer exists and is processed, but nothing draws it, because nothing says where.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: What `setup_rendering` Does" >}}

`setup_rendering` says where. It is the line that turns a buffer holding a picture into a picture on screen. With the defaults written out:
```cpp
image->setup_rendering({
    .target_window = window,
    .fragment_shader = "texture.frag.spv",
});
```
The `TextureBuffer` already carries its defaults: the vertex shader `texture.vert.spv`, the fragment shader `texture.frag.spv`, and a texture binding named `texSampler`. What `setup_rendering` does with them:

1.  Keeps the shaders, using yours where you give them
2.  Records the window the buffer draws to
3.  Declares the texture binding on the render pipeline
4.  Binds the buffer's picture to that binding
5.  Adds a render processor to the buffer's processing chain, aimed at the window

A `TextureBuffer` draws as a triangle strip and ignores any other topology. There is no default window: you choose one, and that is the second line.

The same call, with the same meaning, appears on every buffer that draws. `MeshBuffer` and `MeshNetworkBuffer` have their own, and the video buffer uses the `TextureBuffer` one.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 10: Model, One Buffer per Mesh" >}}

`vega.read_mesh()` read every mesh in the file. By hand:
```cpp
const auto folder = std::filesystem::path("path/to/model.glb").parent_path();
auto resolver = [folder](const std::string& name) {
    return IO::ImageReader::load_texture((folder / name).generic_string());
};

IO::ModelReader reader;
reader.open("path/to/model.glb");
auto meshes = reader.create_mesh_buffers(resolver);
reader.close();

for (auto& mesh : meshes) {
    register_graphics_buffer(mesh);
}
```
1.  The reader imports the file, triangulating faces and generating normals when the file has none
2.  Each mesh becomes a `MeshBuffer` holding its vertices, with position, normal, tangent, texture coordinates and color, and its triangle indices
3.  The resolver finds each mesh's diffuse texture next to the model, and the texture is bound to that buffer when it is found
4.  Registering each buffer is what `| Graphics` does on the group

`setup_rendering` then adds the render processor, with depth testing on so nearer surfaces hide farther ones. It picks the textured shader when the mesh has a texture and the plain one when it does not.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 11: Model as a Network" >}}

`vega.read_mesh_network()` read the same file, but kept its parts together. Each mesh became a slot in a `MeshNetwork`: a name, a node holding that mesh's geometry, a transform of its own, and the diffuse texture when one was found.

`| Graphics` registered the network as a visual network, one the engine advances every frame. A network has no way to draw itself, because drawing needs a buffer, so something still has to make one. By hand:
```cpp
register_node_network(network, Nodes::ProcessingToken::VISUAL_RATE);

auto form = std::make_shared<Buffers::MeshNetworkBuffer>(network);
register_graphics_buffer(form);
form->setup_rendering({ .target_window = window });
```
`vega.mint` does that whole sequence: it registers the network if it is not registered, wraps it in a `MeshNetworkBuffer`, and sets up rendering aimed at the window you give it. Then it returns the real buffer. That is why the window goes inside the `StructureConfig::Model` and not in a separate line.

The network buffer draws every slot with its own transform, and uses each slot's texture when it has one.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 12: Video, a Pair of Containers" >}}

`choose_video` returns a pair: the video container first, the audio second. Destructuring it with `auto [video, audio]` gives each half a name. With a path instead of the dialog:
```cpp
auto [video, audio] = get_io_manager()->load_video(
    "path/to/film.mp4", { .video_options = IO::VideoReadOptions::EXTRACT_AUDIO });
```
What that load does:

1.  Checks the file is readable
2.  Applies the options: video options, audio options, and the size to decode at
3.  When `EXTRACT_AUDIO` is set, asks for audio at the engine's sample rate
4.  Opens the file and registers the reader with the IO manager, which gives it an id the decode machinery uses to find it
5.  Creates a `VideoFileContainer` and loads it
6.  Configures its processor: a `FrameAccessProcessor` that auto-advances at your project's frame rate
7.  When audio was asked for, takes the audio track, configures it as for any sound file, and returns it as the second half

The video container holds the film's frames and information about them. The processor advances through the frames, a little each graphics cycle, scaled by the file's own frame rate against the engine's, so the picture moves at the speed the file was made for.

The audio is also kept by the IO manager, keyed by its video: `get_io_manager()->get_extracted_audio(video)` finds it later.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 13: Video, the Buffer and the Hook" >}}

The picture and the sound are routed separately. By hand, for each:
```cpp
auto picture = get_io_manager()->hook_video_container_to_buffer(video);
picture->setup_rendering({ .target_window = window });

auto buffers = get_io_manager()->hook_audio_container_to_buffers(audio);
```
The last line is `audio | Audio`. When the file has no sound, or you did not ask for it, `audio` is empty and `| Audio` does nothing.

`hook_video_container_to_buffer` made a `VideoContainerBuffer`, which is a `TextureBuffer` that copies the current frame into its texture each cycle. That is why `setup_rendering` is the same line as for an image. When the container reaches its end the buffer removes itself, so the video plays once.

Picture and sound are two objects registered separately. Nothing in this card ties their clocks together.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 14: Why the Video Is Wired by Hand" >}}

A recording only belongs to audio, and a picture only belongs to graphics, so one `|` is enough for each. A video file belongs to both. A shortcut would have to guess where the picture and the sound should go, and a wrong guess is invisible until it sounds or looks wrong.

So here you write both lines. Wiring by hand is not the hard way. It is the way that shows what the machinery is doing, and it is the way you keep when you want to send the sound somewhere else.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 15: Camera, the Same Hook for a Live Source" >}}

`vega.read_camera` is the camera version of the video steps. By hand:
```cpp
auto camera = get_io_manager()->open_camera({ .device_name = "/dev/video0" });
auto buffer = get_io_manager()->hook_camera_to_buffer(camera);
```
It opens the device, then hooks it to a buffer. That is the same kind of hook as `hook_video_container_to_buffer`, so what you get back is the same kind of buffer and the same `setup_rendering` line applies.

The camera is opened through FFmpeg. A camera has no file and no end: frames arrive as the device makes them, and a separate thread decodes one when the graphics cycle asks for it, so the device never holds up drawing. The size and frame rate you ask for are requests, 1920 by 1080 at 30 frames per second by default. The device may give something else. Frames arrive as four channels: red, green, blue and alpha.

There is no dialog because a camera is a device and not a file, so you pass a `CameraConfig` with its name. If the device cannot be opened, the call logs an error and returns nothing.

A camera gives picture only. Its sound counterpart is the microphone.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 16: Every Choice the Dialog Was Making for You" >}}

`vega.read_audio()`, `read_image()` and `read_mesh()` with no argument open the dialog and load what you choose. With a path they skip it.

`choose_video` is the dialog form for video. It takes the options for loading: an empty `{}` is the defaults, and `EXTRACT_AUDIO` asks for the sound as well. Its path form is `get_io_manager()->load_video(path, options)`, shown above.

If you cancel, nothing is loaded and the call returns an empty result. The examples in this card assume you chose a file.

{{< /tutorial-detail >}}

{{< tutorial-detail title="The Fluent vs. Explicit Comparison" >}}

### Fluent (What happens behind the scenes)
```cpp
vega.read_audio() | Audio;
```
This single line loads the file into a container, creates a buffer for every channel, and registers them with the sound card. Nothing plays until the `| Audio`, which is when the connection happens.

### Explicit (What's actually happening)
```cpp
auto container = vega.read_audio();
auto buffers = get_io_manager()->hook_audio_container_to_buffers(container);
```
**Understanding the difference:**

- The fluent version loads *and* hooks in one line
- The explicit version separates the two, so you can inspect or change the container before it is hooked
- Both do the same thing: one is convenience, one is control

---

{{< /tutorial-detail >}}

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

Start from the sound:
```cpp
void compose() {
    vega.read_audio("path/to/your/file.wav") | Audio;
}
```
Replace `"path/to/your/file.wav"` with an actual path and run it. Then change one thing at a time:

- **Image:** change `width` and `height`. The window changes size.
- **Video:** delete `EXTRACT_AUDIO` and replace it with `{}`. The picture still plays, and the sound is gone.
- **Sound:** delete `| Audio`. The file still loads, and nothing plays.

### What You Achieved

You have:

- Brought in a sound, an image, a model and a video as they are
- Seen that loading and routing are separate steps, and why the video is wired by hand
- A container for large data and a buffer for one step of it
- A domain for each result: audio at the sound card's pace, graphics once per frame
- `setup_rendering`, the line that gives a buffer a window

Everything here ran the data as it is. In the next section the sound stops being a whole file. You will pick a region of it, play it faster or backwards, and let several readings of the same recording sound at once.

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

---

{{< tutorial-card step="2 of 5" title="Connect to Buffers" open="true" >}}

#### The Next Step

You have a Container loaded. Now you need to send it somewhere.
```cpp
auto sound_container = vega.read_audio("path/to/file.wav");
auto buffers = get_io_manager::hook_audio_container_to_buffers(sound_container);
```
Run this code. Your file plays.

The Container + the hook call. Together they form the path from disk to speakers. This section shows you what that connection does.

{{< tutorial-detail title="Expansion 1: What Are Buffers?" >}}

A **Buffer** is a temporal accumulator: a space where data gathers until it's ready to be released, then it resets and gathers again.

Buffers don't store your entire file. They store chunks. At your project's sample rate (48 kHz), a typical buffer might hold 512 or 4096 samples: a handful of milliseconds of audio.

Here's why this matters:

Your audio interface (speakers, headphones) has a fixed callback rate. It says: "Give me 512 samples of audio, and do it every 10 milliseconds. Repeat forever until playback stops."

Buffers are the industry standard method to meet this demand.

1.  **Gathers** - accumulates samples from your Container (via its processor)
2.  **Holds** - keeps those samples temporarily
3.  **Releases** - sends them to hardware
4.  **Resets** - becomes empty and ready for the next chunk

This cycle repeats thousands of times per minute. Buffers make that possible.

Without buffers, you'd have to manually manage these chunks yourself. With buffers, MayaFlux handles the cycle. Your Container's processor feeds data into them. The buffers exhale it to your ears.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Why Per-Channel Buffers?" >}}

A stereo file has 2 channels. A multichannel file might have 4 or 8 channels. MayaFlux doesn't merge them into one buffer.

Instead, it creates **one buffer per channel**.

Why? Because channels are independent processing domains. A stereo file's left channel and right channel:

- Can be processed differently
- Can be routed to different outputs
- Can have different process chains
- Can be analyzed separately
- Can coordinate with each other without conflict

When you hook a stereo Container to buffers, MayaFlux creates:

- Buffer for channel 0 (left)
- Buffer for channel 1 (right)

Each buffer:

- Pulls samples from the Container's channel 0 or channel 1 (via the Container's processor)
- Gets filled with 512/4096/etc. samples
- Sends those samples to the audio interface's corresponding output

This per-channel design is why you can later insert processing on a per-channel basis. Insert a filter on channel 0? The first channel gets filtered. Leave channel 1 alone? The second channel plays unprocessed. This flexibility is only possible because channels are architecturally separate at the buffer level.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: The Buffer Manager and Buffer Lifecycle" >}}

MayaFlux has a **buffer manager**; a central system that creates, tracks, and coordinates all buffers in your program.

When you call `get_io_manager::hook_audio_container_to_buffers()`, here's what happens:
```cpp
auto buffer_manager = MayaFlux::get_buffer_manager();
uint32_t num_channels = container->get_num_channels();

for (uint32_t channel = 0; channel < num_channels; ++channel) {
    auto container_buffer = buffer_manager->create_audio_buffer<SoundContainerBuffer>(
        ProcessingToken::AUDIO_BACKEND,
        channel,
        container,
        channel);
    container_buffer->initialize();
}
```
Step by step:

1.  **Get the buffer manager** - a global system that owns all buffers
2.  **Ask the Container: how many channels?** - determines the loop count
3.  **For each channel:**
    - Create an audio buffer of type `SoundContainerBuffer` (a buffer that reads from a Container)
    - Tag it with `AUDIO_BACKEND` (more on this in Expansion 5)
    - Tell it which channel matrix the buffer should belong to
    - Tell it which channel in the Container to read from
    - Initialize it (prepare it for the callback cycle)

Now the buffer manager knows:

- These buffers exist
- These buffers are tied to this Container
- These buffers should feed the audio hardware
- These buffers are ready to cycle

When the audio callback fires (every 10ms at 48 kHz), the buffer manager wakes up all its `AUDIO_BACKEND` buffers and says: "Time for the next chunk. Fill yourselves."

Each buffer asks its Container's processor: "Give me 512 samples from your channel."

The processor pulls from the Container, advances its position, and hands back a chunk.

The buffer receives it and passes it to the audio interface.

Repeat forever.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: SoundContainerBuffer:The Bridge" >}}

You created a `SoundContainerBuffer`, not just a generic `Buffer`. Why the distinction?

A **Buffer** is abstract, it's a temporal accumulator. But abstract things don't know where their data comes from.

A **SoundContainerBuffer** is specific: it's a buffer that knows:

- "My data comes from a Container"
- "My Container has a processor that chunks data"
- "I ask that processor for samples from a specific channel"

When the callback fires, the SoundContainerBuffer doesn't generate samples. It asks: "Container, give me the next 512 samples from your channel 0."

The Container's processor (remember `ContiguousAccessProcessor` from Section 1?) handles this. It:

- Knows where in the file you are (it tracks position)
- Knows how much data to chunk (512 samples)
- Pulls that many samples from its memory
- Auto-advances (moves the position forward)
- Returns the chunk

The SoundContainerBuffer receives it. Done.

This is the architecture: **Buffers don't generate or transform. They request and relay.** The Container's processor does the work. The buffer coordinates timing with hardware.

Later, when you add processing nodes or attach processing chains, you'll insert them between the Container's output and the buffer's input. The buffer still doesn't transform: it still just relays. But what it relays will have been processed first.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Processing Token::AUDIO_BACKEND" >}}

In the buffer creation code:
```cpp
auto container_buffer = buffer_manager->create_audio_buffer<SoundContainerBuffer>(
    ProcessingToken::AUDIO_BACKEND,
    channel,
    container,
    channel);
```
Notice `ProcessingToken::AUDIO_BACKEND`. This is a **token**, a semantic marker that tells MayaFlux:

- **This buffer is audio-domain** (not graphics, not compute)
- **This buffer is connected to the hardware backend** (it's the final destination before speakers)
- **This buffer runs at audio callback rate** (every ~10ms at 48 kHz, every 512 samples)
- **This buffer synchronizes with the real-time audio clock**

Tokens are how MayaFlux coordinates different processing domains without confusion. Later, you might have:

- `AUDIO_BACKEND` buffers - connected to speakers (hardware real-time)
- `AUDIO_PARALLEL` buffers - internal processing (process chains, analysis, etc.)
- `GRAPHICS_BACKEND` buffers - visual domain (frame-rate, not sample-rate)

Each token tells the system what timing, synchronization, and backend this buffer belongs to.

For now: `AUDIO_BACKEND` means "this buffer is feeding your ears directly. It must keep real-time pace with the audio interface."

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Accessing the Buffers" >}}

When you call `vega.read_audio() | Audio`, MayaFlux creates the buffers internally. But now, with the ability to get those buffers back, you have access to them:
```cpp
auto sound_container = vega.read_audio("path/to/file.wav") | Audio;
auto buffers = get_io_manager::get_audio_buffers(sound_container);

// Now you have the buffers as a vector:
// buffers[0] → channel 0
// buffers[1] → channel 1 (if stereo)
// etc.
```
Why is this useful? Because buffers own **processing chains**. And processing chains are where you'll insert processes, analysis, transformations - everything that turns passive playback into active processing.

Each buffer has a method:
```cpp
auto chain = buffers[0]->get_processing_chain();
```
This gives you access to the chain that currently handles that buffer's data. Right now, the chain just reads from the Container and writes to the hardware. But you can modify that chain.

- Add processors.
- Analyze data.
- Route to different destinations.

This is the foundation for Section 3. You load a file, get the buffers, access their chains, and inject processing into those chains.

{{< /tutorial-detail >}}

{{< tutorial-detail title="The Fluent vs. Explicit Comparison" >}}

### Fluent (What happens behind the scenes)
```cpp
vega.read_audio("path/to/file.wav") | Audio;
```
This single line does all of the above: creates a Container, creates per-channel buffers, hooks them to the audio hardware, and starts playback. No file plays until the `| Audio` operator, which is when the connection happens.

### Explicit (What's actually happening)
```cpp
auto sound_container = vega.read_audio("path/to/file.wav") | Audio;
auto buffers = get_io_manager::get_audio_buffers(sound_container);
// File is loaded, buffers exist, but no connection to hardware yet
// Buffers have chains, but nothing is using them

// To actually play, you'd need to ensure they're registered
// (vega.read_audio() | Audio does this automatically)
```
**Understanding the difference:**

- The fluent version (`| Audio`) triggers buffer creation *and* hardware connection
- The explicit version gives you the buffers so you can inspect and modify them *before* hooking to hardware
- Both do the same thing: one is convenience, one is control

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

```cpp
void compose() {
    vega.read_audio("path/to/your/file.wav") | Audio;
}
```
Replace `"path/to/your/file.wav"` with an actual path.

### What You Achieved

You have:

- A Container loaded with all audio data (deinterleaved, resampled, ready)
- Per-channel buffers created, each tied to a Container channel
- Buffers registered with the buffer manager and audio interface
- The callback cycle running, continuously pulling chunks and feeding them to speakers
- Your file plays start-to-finish automatically

No code running during playback; just the callback cycle doing its work, thousands of times per minute.

In the next section, we'll modify these buffers' processing chains. We'll insert a filter processor and hear how it changes the sound. This is where MayaFlux's power truly shines, transforming passive playback into active, real-time audio processing.

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

---

{{< tutorial-card step="3 of 5" title="Buffers Own Chains" open="true" >}}

#### The Simplest Path

You have buffers. You can modify what flows through them.
```cpp
auto sound_container = vega.read_audio("path/to/file.wav") | Audio;
auto buffers = get_io_manager::get_audio_buffers(sound_container);

auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
auto filter_processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
```
Run this code. Your file plays with a low-pass filter applied.

The filter smooths the audio, reduces high frequencies. Listen to the difference.

That's it. Three lines of code: load, get buffers, insert filter. The rest of this section shows you what just happened.

{{< tutorial-detail title="Expansion 1: What Is `vega.IIR()`?" >}}

`vega.IIR()` creates a filter node, a computation unit that processes audio samples one at a time.

An **IIR filter** (Infinite Impulse Response) is a mathematical operation that transforms samples based on feedback coefficients. The two parameters are:

- **Feedforward coefficients** `{0.1, 0.2, 0.1}` - how the current and past input samples contribute
- **Feedback coefficients** `{1.0, -0.6}` - how past output samples contribute

You don't need to understand the math. Just know: this creates a filter that smooths audio.

`vega` is the fluent interface: it subsumes verbosity. Without it:
```cpp
// Without vega - explicit
auto filter = std::make_shared<Nodes::Filters::IIR>(
    std::vector<double>{0.1, 0.2, 0.1},
    std::vector<double>{1.0, -0.6}
);
```
With vega:
```cpp
auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
```
Same filter. Same capabilities. Vega just hides the verbosity.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: What Is `MayaFlux::create_processor()`?" >}}

A **node** (like `vega.IIR()`) is a computational unit: it processes one sample at a time.

A **processor** is a buffer-aware wrapper around that node. It knows:

- How to extract data from a buffer
- How to feed samples to the node
- How to put the transformed samples back in the buffer
- How to handle the buffer's processing cycle

`create_processor()` wraps your filter node in a processor and attaches it to a buffer's processing chain.
```cpp
auto filter_processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
```
What this does:

1.  Takes your filter node
2.  Creates a `FilterProcessor` that knows how to apply that node to buffer data
3.  Adds the processor to `buffers[0]`'s processing chain (implicit: this happens automatically)
4.  Returns the processor so you can reference it later if needed

The buffer now has this processor in its chain. Each cycle, the buffer runs the processor, which applies the filter to all samples in that cycle.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: What Is a Processing Chain?" >}}

Each buffer owns a **processing chain**, an ordered sequence of processors that transform data.

Your buffer's default processor was:

- **SoundStreamReader** - reads from the Container, fills the buffer

When `create_processor()` adds your FilterProcessor, the chain becomes:

1.  Default processor: SoundStreamReader (reads from Container)
2.  **FilterProcessor** (applies your filter) ← You just added this
3.  Other processors you might add later (e.g., Writer to send to hardware)

Each cycle:

- Adapter fills the buffer with 512 samples from the Container
- FilterProcessor runs and modifies those 512 samples by applying the filter
- Other processors run in sequence

Data flows: **Container → \[filtered\] → Speakers**

The chain is ordered. Processors run in sequence. Output of one becomes input to next.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Adding Processor to Another Channel (Optional)" >}}

Your stereo file has two channels. Right now, only channel 0 is filtered.

You can add the same processor to channel 1:
```cpp
auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
auto fp0 = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
auto fp1 = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[1], filter);
```
Or more simply, add the existing processor to another buffer:
```cpp
auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
auto filter_processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
MayaFlux::add_processor(filter_processor, buffers[1], MayaFlux::Buffers::ProcessingToken::AUDIO_BACKEND);
```
`add_processor()` adds an existing processor to a buffer's chain.

`create_processor()` creates a processor and adds it implicitly.

Both do the same underlying thing: they add the processor to the buffer's chain. `create_processor()` just combines creation and addition in one call.

Now both channels are filtered by the same IIR node. Different channel buffers can share the same processor or have independent ones; your choice.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: What Happens Inside" >}}

When you call:
```cpp
auto filter_processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
```
MayaFlux does this:
```cpp
// 1. Create a new FilterProcessor wrapping your filter node
auto processor = std::make_shared<MayaFlux::Buffers::FilterProcessor>(filter);

// 2. Get the buffer's processing chain
auto chain = buffers[0]->get_processing_chain();

// 3. Add the processor to the chain
chain->add_processor(processor, buffers[0]);

// 4. Return the processor
return processor;
```
When `add_processor()` is called separately:
```cpp
MayaFlux::add_processor(filter_processor, buffers[1], MayaFlux::Buffers::ProcessingToken::AUDIO_BACKEND);
```
MayaFlux does this:
```cpp
// Get the buffer manager
auto buffer_manager = MayaFlux::get_buffer_manager();

// Get channel 1's buffer for AUDIO_BACKEND token
auto buffer = buffer_manager->get_buffer(ProcessingToken::AUDIO_BACKEND, 1);

// Get its processing chain
auto chain = buffer->get_processing_chain();

// Add the processor
chain->add_processor(processor, buffer);
```
The machinery is consistent: **processors are added to chains, chains are owned by buffers, buffers execute chains each cycle.**

You don't need to write this explicitly; the convenience functions handle it. But this is what's happening.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: Processors Are Reusable Building Blocks" >}}

A processor is a building block. Once created, it can be:

- Added to multiple buffers (same processor, multiple channels)
- Composed with other processors (insert multiple processors)
- Swapped out (remove and replace)
- Queried (ask for its state, parameters, etc.)

Example: two channels with the same filter:
```cpp
auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
auto processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
MayaFlux::add_processor(processor, buffers[1]);
```
Example: stacking processors (requires understanding of chains, shown later):
```cpp
auto filter1 = vega.IIR(...);
auto fp1 = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter1);

auto filter2 = vega.IIR(...); // Different filter
auto fp2 = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter2);
```
Now `buffers[0]` has two FilterProcessors in its chain. Data flows through both sequentially.

Processors are the creative atoms of MayaFlux. Everything builds from them.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Try It → Recap" >}}

```cpp
void compose() {
    auto sound_container = vega.read_audio("path/to/your/file.wav") | Audio;
    auto buffers = get_io_manager::get_audio_buffers(sound_container);

    auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
    auto filter_processor = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
}
```
Replace `"path/to/your/file.wav"` with an actual path.

Run the program. Listen. The audio is filtered.

Now try modifying the coefficients:
```cpp
auto filter = vega.IIR(std::vector{0.05, 0.3, 0.05}, std::vector{1.0, -0.8});
```
Listen again. Different sound. You're sculpting the filter response.

### What You Achieved

You've just inserted a processor into a buffer's chain and heard the result. That's the foundation for everything that follows.

In the next section, we'll interrupt this passive playback. We'll insert a processing node between the Container and the buffers. And you'll see why this architecture, i.e buffers as relays, not generators, enables powerful real-time transformation.

For a comprehensive tutorial on buffer processors and related concepts, visit the [Buffer Processors Tutorial](./ProcessingExpression.md).

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

---

{{< tutorial-card step="4 of 5" title="Timing, Streams, and Bridges" open="true" >}}

#### The Current Continous Flow

What you've done so far is simple and powerful:
```cpp
auto sound_container = vega.read_audio("path/to/file.wav") | Audio;
auto buffers = get_io_manager::get_audio_buffers(sound_container);
auto filter = vega.IIR(std::vector{0.1, 0.2, 0.1}, std::vector{1.0, -0.6});
auto fp = MayaFlux::create_processor<MayaFlux::Buffers::FilterProcessor>(buffers[0], filter);
```
This flow is designed for **full-file playback**:

- Load the entire file into a Container
- Route it through buffers
- Add general purpose processes
- Play to speakers (RtAudio backend via SubsystemManagers)

Clean. Direct. No timing control.

That's intentional.

There are other features - region looping, seeking, playback control, but they don't fit this tutorial. These sections are purely for: **file → buffers → output, uninterrupted.**

{{< tutorial-detail title="Where We're Going" >}}

Here's what the next section enables:
```cpp
auto pipeline = MayaFlux::create_buffer_pipeline();
pipeline->with_strategy(ExecutionStrategy::PHASED); // Execute each phase fully before next op

pipeline
    >> BufferOperation::capture_file_from("path/to/file.wav", 0)  // From channel 0
    .for_cycles(20)  // Process 20 buffer cycles
    >> BufferOperation::transform([](auto& data, uint32_t cycle) {
        // Data now has 20 buffer cycles of audio from the file
        // i.e 20 x 512 samples if buffer size is 512
        auto zero_crossings = MayaFlux::zero_crossings(data);

        std::cout << "Zero crossings at indices:\n";
        for (const auto& sample : zero_crossings) {
            std::cout << sample << "\t";
        }
        std::cout << "\n";

        return data;
    });

pipeline->execute_buffer_rate();  // Schedule and run
```
This processes exactly 20 buffer cycles from the file (with any process you want), accumulates the result in a stream, and executes the pipeline.

The file isn't playing to speakers. It's being captured, processed, and stored in a stream. **Timing is under your control.** You decide how many buffer cycles to process. This section builds the foundation for buffer pipelines. Understanding the architecture below explains why the code snippet works.

In this section, we will introduce the machinery for everything beyond simplicity. We're not building code that has audio yet. We're establishing the architecture that enables timing control, streaming, capture, and composition.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 1: The Architecture of Containers" >}}

A Container (like SoundFileContainer) holds all data upfront:

- Load entire file into memory
- Data is fixed size
- Processor knows where in the file you are
- Designed for sequential access: read start, advance, read next chunk, repeat until end

This works perfectly for "play the whole file". It also works for as yet unexplored controls over the same timeline, such as looping, seeking positions, jumping to regions, etc.

But it doesn't work for:

- **Recording**: You don't know the final size upfront
- **Structuring**: You need to manipulate boundaries
- **Streaming**: Data arrives in chunks; size grows dynamically
- **Capture**: You want to save specific buffer cycles, not the whole file

For these use cases, you need a different data structure.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: Enter DynamicSoundStream" >}}

A **DynamicSoundStream** is a child class of `SignalSourceContainer` much like `SoundFileContainer` that we have been using. It has the same interface as `SoundFileContainer` (channels, frames, metadata, regions). But it has different semantics:

- **Dynamic size**: Starts small, grows as data arrives
- **Transient modes**: Can operate as a circular buffer (fixed size, overwrites old data)
- **Sequential writing**: Designed to accept data sequentially from processors
- **No inherent structure**: Unlike SoundFileContainer (which knows "this is a file with a start and end"), DynamicSoundStream is just a growing reservoir of data.

Think of it as:

- **SoundFileContainer**: "I am this exact file, with this exact data"
- **DynamicSoundStream**: "I am a space where audio data accumulates. I don't know how much will arrive."

DynamicSoundStream has powerful capabilities:

- **Auto-resize mode**: Grows as data arrives (good for recording)
- **Circular mode**: Fixed capacity, wraps around (good for delay lines or rolling analysis)
- **Position tracking**: Knows where reads/writes are in the stream
- **Capacity pre-allocation**: You can reserve space if you know approximate size

You don't create DynamicSoundStream directly (yet). It's managed implicitly by other systems. But understanding what it is explains everything that follows.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: SoundStreamWriter" >}}

You've seen `BufferProcessors` like `FilterProcessor` that transform data in place.

But `SoundStreamWriter` is more general. It can write buffer data to **any** `DynamicSoundStream`, not just locally to attached buffers (or from hardware: hitherto unexplored `InputListenerProcessor`).

When a processor runs each buffer cycle:

1.  Buffer gets filled with 512 samples (from Container or elsewhere)
2.  Processors run (your `FilterProcessor`, for example)
3.  `SoundStreamWriter` writes the (now-processed) samples to a `DynamicSoundStream`

The `DynamicSoundStream` accumulates these writes:

- Cycle 1: 512 samples written
- Cycle 2: Next 512 samples written (total: 1024)
- Cycle 3: Next 512 samples written (total: 1536)
- ...

After N cycles, the `DynamicSoundStream` contains N × 512 samples of processed audio.

This is how you capture buffer data. Not by sampling the buffer once, by continuously writing it to a stream through a processor.

`SoundStreamWriter` is the bridge between buffers (which live in real-time) and streams (which accumulate).

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: SoundFileBridge: Controlled Flow" >}}

**SoundFileBridge** is a specialized buffer that orchestrates reading from a file and writing to a stream, with timing control through buffer cycles.

Internally, SoundFileBridge creates a processing chain:
```text
SoundFileContainer (source file)
    ↓
SoundStreamReader (reads from file, advances position)
    ↓
[Your processors here: filters, etc.]
    ↓
SoundStreamWriter (writes to internal DynamicSoundStream)
    ↓
DynamicSoundStream (accumulates output)
```
The key difference from your simple load/play flow:

- Instead of routing to hardware, data goes to a DynamicSoundStream
- You control **how many buffer cycles** run (e.g., "process 10 cycles of this file")
- After N cycles, the stream holds N × buffer_size samples of the processed result

SoundFileBridge represents: **"Read from this file, process through this chain, accumulate result in this stream, for exactly this many cycles."**

This gives you timing control. You don't play the whole file. You process exactly N cycles, then stop.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 5: Why This Architecture?" >}}

The architecture separates concerns:

- **Reading**: Done by SoundStreamReader (reads from SoundFileContainer in controlled chunks)
- **Processing**: Done by your custom processors
- **Writing**: Done by SoundStreamWriter (writes results to DynamicSoundStream)
- **Accumulation**: Done by DynamicSoundStream (holds the result)

Each layer is independent:

- You can swap the reader (use a different Container)
- You can insert any number of processors
- You can swap the writer (write to hardware, to disk, to memory, to GPU)
- The stream is just a data holder; it doesn't care what filled it

This is why SoundFileBridge is powerful: it composes these layers without forcing you to wire them manually.

And it's why understanding this section matters: **the next tutorial (BufferOperation) builds on top of this composition**, adding temporal coordination and pipeline semantics.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 6: From File to Cycle" >}}

A **cycle** is one complete buffer processing round:

- 512 samples from the source
- Processed through all processors
- Written to the destination stream

At 48 kHz, one cycle is 512 ÷ 48000 ≈ 10.67 milliseconds of audio.

When you say "process this file for 20 cycles," you mean:

- Run 20 iterations of: read 512 → process → write 512
- Result: 10,240 samples (≈ 213 ms of audio at 48 kHz)

Timing control is expressed in **cycles**, not time. This is intentional:

- Cycles are deterministic (you know exactly how much data will be processed)
- Cycles are aligned with buffer boundaries (no partial processing)
- Cycles decouple from hardware timing (no real-time constraints)

SoundFileBridge lets you say: "Process this file for exactly N cycles," then accumulate the result in a stream.

This is the foundation for everything BufferOperation does: it extends this cycle-based thinking to composition and coordination.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="The Three Key Concepts" >}}

At this point, understand:

1.  **DynamicSoundStream**: A container that grows dynamically, can operate in circular mode, designed to accumulate data from processors
2.  **SoundStreamWriter**: The processor that writes buffer data sequentially to a DynamicSoundStream
3.  **SoundFileBridge**: A buffer that creates a chain (reader → your processors → writer), and lets you control how many buffer cycles run

These three concepts enable timing control. You're no longer at the mercy of real-time callbacks. You can process exactly N cycles, accumulate results, and move on.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="Why This Section Has No Audio Code" >}}

This is intentional. The concepts here are essential, and expose the architecture behind everything that follows. It is also a hint at the fact that modal output is not the only use case for MayaFlux.

- SoundFileBridge is too low-level. You'd create it manually, call setup_chain_and_processor(), manage cycles yourself
- DynamicSoundStream is too generic. Without a driver, you'd just accumulate data with no purpose
- SoundStreamWriter is just a piece, alone, it doesn't tell you how many cycles to run

The **next tutorial introduces BufferOperation**, which wraps these concepts into high-level, composable patterns:

- `BufferOperation::capture_file()` - wrap SoundFileBridge, accumulate N cycles, return the stream
- `BufferOperation::file_to_stream()` - connect file reading to stream writing, with cycle control
- `BufferOperation::route_to_container()` - send processor output to a stream

Once you understand SoundFileBridge, DynamicSoundStream, and cycle-based timing, BufferOperation will feel natural. It's just syntactic sugar on top of this architecture.

For now: **internalize the architecture. The next section shows how to use it.**

### What You Should Internalize

- Containers hold data (SoundFileContainer holds files; DynamicSoundStream holds growing data)
- Processors transform data (your FilterProcessor, SoundStreamWriter, etc.)
- Buffers orchestrate cycles (read N cycles, run processors, write results)
- Streams accumulate (DynamicSoundStream holds results after cycles complete)
- Timing is expressed in cycles (deterministic, aligned with buffer boundaries, decoupled from real-time)

This is the mental model for everything that follows. Pipelines, capture, routing; they all build on this foundation.

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

---

{{< tutorial-card step="5 of 5" title="Buffer Pipelines (Teaser)" open="true" >}}

```cpp
void compose() {
    // Create an empty audio buffer (will hold captured data)
    auto capture_buffer = vega.AudioBuffer() | Audio[1];
    // Create a pipeline
    auto pipeline = MayaFlux::create_buffer_pipeline();
    // Set strategy to streaming (process as data arrives)
    pipeline->with_strategy(ExecutionStrategy::STREAMING);

    // Declare the flow:
    pipeline
        >> BufferOperation::capture_file_from("path/to/audio/.wav", 0)
            .for_cycles(1) // Essential for streaming
        >> BufferOperation::route_to_buffer(capture_buffer) // Route captured data to our buffer
        >> BufferOperation::modify_buffer(capture_buffer, [](std::shared_ptr<AudioBuffer> buffer) {
            for (auto& sample : buffer->get_data()) {
                sample *= MayaFlux::get_uniform_random(-0.5, 0.5); // random "texture" between 0 and 0.5
            }
        });

    // Execute: runs continuously at buffer rate
    pipeline->execute_buffer_rate();
}
```
Run this. You'll hear the file play back with noisy texture. But the file never played to speakers directly: it was captured, processed, accumulated, then routed.

### The Next Level

Everything you've learned so far processes data in isolation: load a file, add a processor, output to hardware.

But what if you want to:

- **Capture** a specific number of buffer cycles from a file
- **Process** those cycles through custom logic
- **Route** the result to a buffer for playback
- **Do all of this in one declarative statement**

That's what buffer pipelines do.

{{< tutorial-detail title="Expansion 1: What Is a Pipeline?" >}}

A **pipeline** is a declarative sequence of buffer operations that compose to form a complete computational event.

Unlike the previous sections where you manually:

1.  Load a file
2.  Get buffers
3.  Create processors
4.  Add to chains

...a pipeline lets you describe the entire flow in one statement:
```cpp
pipeline
    >> Operation1
    >> Operation2
    >> Operation3;
```
The `>>` operator chains operations. The pipeline executes them in order, handling all the machinery (cycles, buffer management, timing) invisibly.

This is why you've been learning the foundation first: **pipelines are just syntactic sugar over SoundFileBridge, DynamicSoundStream, SoundStreamWriter, and buffer cycles.**

Understanding the previous sections makes this section obvious. You're not learning new concepts; you're composing concepts you already understand.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 2: BufferOperation Types" >}}

BufferOperation is a toolkit. Common operations:

- **capture_file()** - Read N cycles from a file, accumulate in internal stream
- **modify_buffer()** - Apply custom logic to a specific AudioBuffer
- **route_to_buffer()** - Send accumulated result to an AudioBuffer for playback
- **route_to_container()** - Send result to a DynamicSoundStream (for recording, analysis, etc.)
- **transform()** - Map/reduce on accumulated data (structural transformation)
- **dispatch()** - Execute arbitrary code with access to the data

Each operation is a building block. Pipeline chains them together.

The full set of operations is the subject of its own tutorial. This section just shows the pattern.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 3: The `on_capture_processing` Pattern" >}}

Notice in the example:
```cpp
>> BufferOperation::modify([](auto& data, uint32_t cycle) {
    // Called every cycle as data accumulates
    for (auto& sample : data) {
        sample *= 0.5;
    }
})
```
The `modify` operation runs **each cycle**, meaning:

- Cycle 1: 512 samples captured, modified by your lambda
- Cycle 2: Next 512 samples captured, modified
- Cycle 3: And so on

This is `on_capture_processing`: your custom logic runs as data arrives, not automated by external managers.

Automatic mode simply expects buffer manager to handle the processing of attached processors. On Demand mode expects users to provide callback timing logic.

For now: understand that pipelines let you hook custom logic into the capture/process/route flow.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 4: Why This Matters" >}}

Before pipelines, your workflow was:

1.  Load file (Container)
2.  Get buffers
3.  Add processors to buffers
4.  Play to hardware
5.  Everything was real-time

With pipelines, your workflow is:

1.  Declare capture (file, cycle count)
2.  Declare processing (what to do each cycle)
3.  Declare output (where result goes)
4.  Execute (all at once, deterministic, no real-time constraints)

The key difference: **determinism**. You know exactly what will happen because you've declared the entire flow.

This is the foundation for everything beyond this tutorial:

- Recording sessions
- Batch processing
- Data analysis pipelines
- Complex temporal arrangements
- Multi-file composition

All of it starts with this pattern: **declare → execute → observe**.

---

{{< /tutorial-detail >}}

{{< tutorial-detail title="What Happens Next" >}}

The full **Buffer Pipelines** tutorial is its own comprehensive guide. It covers:

- All BufferOperation types
- Composition patterns (chaining operations)
- Timing and cycle coordination
- Error handling and introspection
- Advanced patterns (branching, conditional operations, etc.)

This section is just the proof-of-concept: "Here's what becomes possible when everything you've learned composes."

---
### Try It (Optional)

The code above will run if you have:

- A `.wav` file at `"path/to/file.wav"`
- All the machinery from Sections 1-3 understood

If you want to experiment, use a real file path and run it.

But the main point is: **understand what's happening**, not just make it work.

- You're capturing from a file
- Each cycle, your lambda processes 512 samples
- Results accumulate in capture_buffer
- Then capture_buffer plays to hardware

This is real composition. Not playback. Not presets. Declarative data transformation.

---

{{< /tutorial-detail >}}

{{< /tutorial-card >}}

---

{{< framing-card >}}

### The Philosophy

You've now seen the complete stack:

1.  **Containers** hold data (load files)
2.  **Buffers** coordinate cycles (chunk processing)
3.  **Processors** transform data (effects, analysis)
4.  **Chains** order processors (sequence operations)
5.  **Pipelines** compose chains (declare complete flows)

Each layer builds on the previous. None is magic. All are composable.

This is how MayaFlux thinks about computation: as layered, declarative, composable building blocks.

Pipelines are where that thinking becomes powerful. They're not a special feature, they're just the final layer of composition.

---
### Next: The Full Pipeline Tutorial

When you're ready, the standalone **"Buffer Pipelines"** tutorial dives deep into:

- Every BufferOperation type with examples
- How to compose complex workflows
- Error handling and debugging
- Performance considerations
- Real-world use cases

For now: you've seen the teaser. Everything you've learned so far is the foundation for that depth.

You understand how information flows. Pipelines just let you declare that flow elegantly.

{{< /framing-card >}}

