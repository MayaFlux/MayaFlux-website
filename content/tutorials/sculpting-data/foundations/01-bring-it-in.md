---
build:
  list: never
---

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

The input can also be turned on from a file, without recompiling. See Expansion 8.

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

    video | Graphics;
    auto picture = get_associated_buffer(video);
    picture->setup_rendering({ .target_window = window });

    window->show();
}
```
Run this code. The picture plays in the window and the sound plays through your speakers. `video | Graphics` hooks the picture, and the sound too when it was asked for.

### Video, Picture Only
```cpp
void compose() {
    auto window = create_window({ .title = "Video", .width = 1280, .height = 720 });

    auto [video, audio] = choose_video({});

    video | Graphics;
    auto picture = get_associated_buffer(video);
    picture->setup_rendering({ .target_window = window });

    window->show();
}
```
The only difference from the block above is the empty `{}`. Without `EXTRACT_AUDIO` the sound is never decoded, so `audio` is empty and there is nothing to hook.

{{< tutorial-detail title="Also: Picture from a Camera" >}}

A camera is a live video source. It has no file to browse to, so two small windows open instead: one to choose the camera, and one to choose a size and frame rate.
```cpp
void compose() {
    auto window = create_window({ .title = "Camera", .width = 1280, .height = 720 });

    auto camera = vega.read_camera() | Graphics;
    auto picture = get_associated_buffer(camera);
    picture->setup_rendering({ .target_window = window });

    window->show();
}
```
Your live picture appears in the window.

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
- `video | Graphics` is `get_io_manager()->hook_video_container_to_buffer(video)`, and the same for the sound that came with it. A camera takes `hook_camera_to_buffer(camera)`
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

{{< tutorial-detail title="Expansion 8: Settings, in Code and in a File" >}}

Everything in this card ran on the engine's defaults: 48 kHz, 512 frames per block, stereo output and 60 frames per second. `settings()` is where you change them.

`settings()` is a function in your `src/user_project.hpp`. It runs before the engine starts, and what you set there is fixed for the life of the program. By the time `compose()` runs the engine is already going, and changing a setting then logs a warning and has no effect. That is why the microphone needs `settings()`: the sound card's input is opened when the engine starts, so it has to be asked for beforehand.

Settings come in five sections:

- **stream:** sample rate, block size, and the input and output channels and devices
- **graphics:** frame rate, the window surface, which GPU to use, the default font
- **input:** MIDI, OSC and other devices
- **network:** UDP and TCP
- **journal:** how much the engine tells you, and where

In code:
```cpp
void settings() {
    auto& stream = Config::get_global_stream_info();
    stream.buffer_size = 256;
    stream.input.enabled = true;
    stream.input.channels = 1;

    Config::get_global_graphics_config().target_frame_rate = 60;
}
```
The same settings in a file, `mayaflux.json`, in your project's source folder:
```json
{
  "stream": {
    "buffer_size": 256,
    "input": { "enabled": true, "channels": 1 }
  },
  "graphics": {
    "target_frame_rate": 60
  }
}
```
The file is read every time the program starts, so changing it needs no rebuild. The launcher picks up `mayaflux.json` by itself when it exists, so `settings()` can stay empty. The keys are the names of the config fields, and any field you leave out keeps its default.

Do not confuse `"stream"` and the `"input"` inside it, which is your sound card's input, with the top level `"input"` section, which is for MIDI, OSC and other devices.

**Choosing the file.** To use a file from somewhere else, start the program with `--config path/to/file.json`.

**Which one wins.** The file is read before `settings()`, so when both set the same value, `settings()` wins. Start the program with `--config-override` to turn that around: the file is then read after `settings()` and replaces all of it, so anything the file does not mention goes back to its default.

**Reading settings back.** Once the engine is running you can read what was set:
```cpp
void compose() {
    uint32_t rate = Config::get_sample_rate();
    uint32_t block = Config::get_buffer_size();
    uint32_t channels = Config::get_num_out_channels();
}
```
Logging is the one exception to the rule that settings are fixed at start: the journal settings take effect at once.

`docs/Settings.md` lists every field.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 9: Image, Pixels in a TextureBuffer" >}}

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

{{< tutorial-detail title="Expansion 10: What `setup_rendering` Does" >}}

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

{{< tutorial-detail title="Expansion 11: Model, One Buffer per Mesh" >}}

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

{{< tutorial-detail title="Expansion 12: Model as a Network" >}}

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

{{< tutorial-detail title="Expansion 13: Video, a Pair of Containers" >}}

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

{{< tutorial-detail title="Expansion 14: Video, the Buffer and the Hook" >}}

`video | Graphics` hooks the picture and, when the file has sound you asked for, the sound. By hand, for each:
```cpp
auto picture = get_io_manager()->hook_video_container_to_buffer(video);
picture->setup_rendering({ .target_window = window });

auto buffers = get_io_manager()->hook_audio_container_to_buffers(audio);
```
The last line is `audio | Audio`. When the file has no sound, or you did not ask for it, `audio` is empty and there is nothing to hook. Afterwards `get_associated_buffer(video)` finds the picture's buffer, and `get_associated_buffers(video)` finds the sound's, one per channel.

`hook_video_container_to_buffer` made a `VideoContainerBuffer`, which is a `TextureBuffer` that copies the current frame into its texture each cycle. That is why `setup_rendering` is the same line as for an image. When the container reaches its end the buffer removes itself, so the video plays once.

One pipe registers both, but they are still two objects. Nothing in this card ties their clocks together.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 15: Why the Video Is Wired by Hand" >}}

A recording only belongs to audio, and a picture only belongs to graphics, so one `|` is enough for each. A video file belongs to both. A shortcut would have to guess where the picture and the sound should go, and a wrong guess is invisible until it sounds or looks wrong.

So here you write both lines. Wiring by hand is not the hard way. It is the way that shows what the machinery is doing, and it is the way you keep when you want to send the sound somewhere else.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 16: Camera, the Same Hook for a Live Source" >}}

`vega.read_camera` is the camera version of `vega.read_audio` and the video steps:
```cpp
auto camera = vega.read_camera();
auto buffer = get_io_manager()->hook_camera_to_buffer(camera);
```
The first line asks which camera and which mode, and opens it. It gives back the camera itself, the way `vega.read_audio` gives back the sound, and nothing is shown yet. The second hooks it to a buffer, and `camera | Graphics` does the same, after which `get_associated_buffer(camera)` finds the buffer. That is the same kind of hook as `hook_video_container_to_buffer`, so what you get back is the same kind of buffer and the same `setup_rendering` line applies. The buffer is already registered, so there is no `| Graphics`.

The camera is opened through FFmpeg. A camera has no file and no end: frames arrive as the device makes them, and a separate thread decodes one when the graphics cycle asks for it, so the device never holds up drawing. The mode you choose is a request, and the device falls back to the nearest one it accepts, so it may give something else. Frames arrive as four channels: red, green, blue and alpha.

To skip the windows, pass a `CameraConfig` with the device name: `vega.read_camera({ .device_name = "/dev/video0" })`. `/dev/video0` is the first camera on Linux. On macOS use `"0"`, and on Windows use `"video=Integrated Camera"` or the name your camera reports. `choose_camera(false)` skips only the mode window. If you close a window or the device cannot be opened, the call returns nothing.

A camera gives picture only. Its sound counterpart is the microphone.

{{< /tutorial-detail >}}

{{< tutorial-detail title="Expansion 17: Every Choice the Dialog Was Making for You" >}}

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
