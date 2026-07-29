---
title: "The Rendering Model"
---

## The loop was never load-bearing

A violin is made of wood, and wood has grain, density, a resonant response that changes with humidity. For centuries, luthiers have worked so that none of that becomes the player's problem. The glue they use, the veneer they need, the exact curve of a bridge, all of it absorbed on the maker's side so the physical truth of the material never limits what the player can do with it.

But the constraints that do reach the player are not simply obstacles either. A bow drawn too hard, a string pushed past its comfortable range, a valve half-depressed, these are how extended technique gets discovered in the first place. Sul ponticello, multiphonics, prepared piano: none of these were designed in. They are the residue of someone working against a physical limit hard enough that the fighting itself became a vocabulary. The constraint can be leaned on, partially defeated, negotiated with in degrees, and every degree of that negotiation is available as expression. That's what makes a physical limit generative rather than merely restrictive: it has a texture, a give, a range between "easy" and "impossible" that a player can occupy anywhere they like.

A software boundary rarely offers that. `update()` runs once per tick or it doesn't run. There is no version of "pushing against" a callback contract, no partial defeat, no extended technique for a function signature. Either the framework's choice governs your program or it doesn't, and the only way to get to the far side of that boundary is to remove it, not to lean on it until it gives a little.

`draw()`, called once per frame, is exactly this kind of wall.

It's worth being precise about what the actual hardware constraint is, because it's real, and it's easy to conflate with the wall built on top of it. A display needs frames at a steady rate, roughly twenty-four or more per second, or motion stops reading as motion. That number belongs to the hardware. It is not negotiable, and it sits on the luthier's side of the line, same as the density of spruce.

But the requirement stops there. Frames need to exist at that rate. Nothing about the requirement says a single callback has to own the job of producing them, and nothing says the person writing the program has to think in units of one-redraw-per-tick, personally, every time, to satisfy a number that belongs to the display, not to them. That's exactly the cascade the luthier principle warns against: a hardware constraint, real and physical, handed forward into user space as though the user were the one who had to meet it. `update()`/`draw()` is that cascade, given a function signature. Unless hitting a fixed redraw rate is itself the creative material someone is working with, which is a legitimate thing to want, nobody should have to carry that number around as a mental tax just to put something on screen.

Here is what actually happens instead.

There is one root object collecting every GPU-bound buffer in the system, and it carries two processors, a default one and a final one. The default processor is the one actually asking Vulkan to draw. It walks every child buffer that has something to upload or compute, records each one's work into its own command buffer, dispatches it, and does this for as many buffers as exist, whether that's three or three hundred, before anything reaches a screen. The drawing has already happened by the time the final processor runs.

The final processor doesn't draw anything at all. Its only job is to take everything the default processor already recorded and dispatched, fold it into one primary command buffer per window, close out dynamic rendering, and put the result on screen: one asynchronous submission, one present, per window, per cycle. A window's screen updates because a batch of already-drawn work reached its natural end and got presented, not because a callback reached the bottom of a function body it runs every sixteen milliseconds regardless of whether there was anything to flush.

<svg width="100%" viewBox="0 0 680 460" role="img" xmlns="http://www.w3.org/2000/svg">
<title>RootGraphicsBuffer processing pipeline</title>
<desc>Three buffers reach the root three different ways. The default processor records and dispatches each one's draw or compute work to Vulkan. The final processor only takes what has already been drawn and presents it to the window's swapchain.</desc>
<defs>
<marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker>
</defs>
<style>
  text { font-family: sans-serif; fill: #2C2C2A; }
  .th { font-size: 14px; font-weight: 500; }
  .ts { font-size: 12px; fill: #5F5E5A; }
  .box-gray { fill: #F1EFE8; stroke: #5F5E5A; }
  .box-blue { fill: #E6F1FB; stroke: #185FA5; }
  .box-coral { fill: #FAECE7; stroke: #993C1D; }
  .box-purple { fill: #EEEDFE; stroke: #534AB7; }
  .box-teal { fill: none; stroke: #0F6E56; }
  .arr { stroke: #5F5E5A; stroke-width: 1; }
  @media (prefers-color-scheme: dark) {
    text { fill: #D3D1C7; }
    .ts { fill: #B4B2A9; }
    .box-gray { fill: #444441; stroke: #B4B2A9; }
    .box-blue { fill: #0C447C; stroke: #85B7EB; }
    .box-coral { fill: #4A1B0C; stroke: #F0997B; }
    .box-purple { fill: #26215C; stroke: #AFA9EC; }
    .box-teal { stroke: #5DCAA5; }
    .arr { stroke: #B4B2A9; }
  }
</style>

<g class="box-gray" style="color:#5F5E5A">
<rect x="40" y="30" width="150" height="48" rx="8" stroke-width="0.5"/>
<text class="th" x="115" y="48" text-anchor="middle" dominant-baseline="central">Buffer A</text>
<text class="ts" x="115" y="66" text-anchor="middle" dominant-baseline="central">registered</text>
</g>
<g class="box-gray" style="color:#5F5E5A">
<rect x="265" y="30" width="150" height="48" rx="8" stroke-width="0.5"/>
<text class="th" x="340" y="48" text-anchor="middle" dominant-baseline="central">Buffer B</text>
<text class="ts" x="340" y="66" text-anchor="middle" dominant-baseline="central">process_default()</text>
</g>
<g class="box-gray" style="color:#5F5E5A">
<rect x="490" y="30" width="150" height="48" rx="8" stroke-width="0.5"/>
<text class="th" x="565" y="48" text-anchor="middle" dominant-baseline="central">Buffer C</text>
<text class="ts" x="565" y="66" text-anchor="middle" dominant-baseline="central">coroutine chain</text>
</g>

<line x1="115" y1="78" x2="115" y2="112" class="arr" marker-end="url(#arrow)"/>
<line x1="340" y1="78" x2="340" y2="112" class="arr" marker-end="url(#arrow)"/>
<line x1="565" y1="78" x2="565" y2="112" class="arr" marker-end="url(#arrow)"/>

<g class="box-teal" style="color:#0F6E56">
<rect x="40" y="112" width="600" height="230" rx="16" stroke-width="0.5"/>
<text class="th" x="60" y="136">RootGraphicsBuffer</text>
</g>

<g class="box-blue" style="color:#185FA5">
<rect x="70" y="156" width="540" height="90" rx="12" stroke-width="0.5"/>
<text class="th" x="90" y="178">Default processor</text>
<text class="ts" x="90" y="196">Records and dispatches each buffer's draw or</text>
<text class="ts" x="90" y="212">compute work to Vulkan. This is where drawing happens.</text>
</g>

<line x1="340" y1="246" x2="340" y2="270" class="arr" marker-end="url(#arrow)"/>
<text class="ts" x="360" y="262">already drawn</text>

<g class="box-coral" style="color:#993C1D">
<rect x="70" y="270" width="540" height="56" rx="12" stroke-width="0.5"/>
<text class="th" x="90" y="291">Final processor</text>
<text class="ts" x="90" y="309">Draws nothing. Folds recordings into one primary buffer per window.</text>
</g>

<line x1="340" y1="342" x2="340" y2="376" class="arr" marker-end="url(#arrow)"/>

<g class="box-purple" style="color:#534AB7">
<rect x="220" y="376" width="240" height="48" rx="8" stroke-width="0.5"/>
<text class="th" x="340" y="394" text-anchor="middle" dominant-baseline="central">Present to screen</text>
<text class="ts" x="340" y="412" text-anchor="middle" dominant-baseline="central">async, once per window per cycle</text>
</g>
</svg>

None of the buffers feeding into this know or care how they got there. A buffer reaches the root three ways, and all three are just different ways of saying "this needs to run." Register it once, normally, and it's swept up automatically every cycle the root processes, no further code needed. Call its `process_default()` directly, and it runs immediately, right now, outside whatever cycle the root would otherwise use. Or drive it from inside a coroutine, a chain's `process()` call sitting between two `co_await`s, so the work happens exactly when that routine says to and not on any fixed schedule at all. All three paths end up in the same accumulate-then-flush pipeline. The pipeline doesn't know or ask which one a given buffer arrived through.

This is what removing the cascade actually buys, concretely, not as a philosophy. A buffer that only needs to update when a sensor value crosses a threshold can sit idle for minutes and cost nothing, instead of participating in a redraw it has nothing to contribute to every sixteen milliseconds. A slow, expensive computation, a large mesh rebuild, an analysis pass over accumulated history, can run inside a coroutine at whatever pace it actually needs, five times a second, once every ten seconds, once, ever, without fighting a callback that assumes every piece of the scene has an opinion every frame. Two unrelated pieces of work, one driven by audio arriving on its own schedule, one driven by input events arriving on theirs, can each register their own timing and never negotiate with each other, because neither was ever forced to describe itself in terms of the other's tick.

None of this was available while `draw()` stood in for both "the hardware needs frames" and "you must think in frames." Once those two facts are separated, the frame rate stays exactly as real as it always was, and the person building something stops being the one who has to carry it.

The rest of this document is what becomes buildable once that tick is no longer assumed: not a faster way to draw the same things, but structures that were never expressible as something you draw in the first place.

## A window watching itself

Open two windows. The first plays a video. The second shows nothing yet.

```cpp
auto win1 = MayaFlux::create_window({ .title = "source", .width = 1280, .height = 720 });
win1->show();

auto ct = get_io_manager()->load_video("res/source.mkv");
auto video_buf = get_io_manager()->hook_video_container_to_buffer(ct);
video_buf->setup_rendering({ .target_window = win1 });

auto win2 = MayaFlux::create_window({ .title = "history", .width = win1->width(), .height = win1->height() });
win2->show();
```

Nothing unusual so far. Two windows, one playing a file. The second window is where it stops looking like anything you've built before.

```cpp
auto win1_container = std::make_shared<Kakshya::WindowContainer>(win1);
win1_container->create_default_processor();

auto history = std::make_shared<std::vector<std::shared_ptr<Core::VKImage>>>();

schedule_metro(0.5, [win1_container, history]() {
    win1_container->process_default();
    if (win1_container->has_data())
        if (auto img = win1_container->to_image())
            history->emplace_back(std::move(img));
}, "capture");
```

`win1_container` is not a screenshot mechanism. It is `win1`'s rendered surface, addressed as data, twice a second, forever, growing a history of what that window has shown. `to_image()` hands back a `VKImage`, GPU-resident, no pixels touched by the CPU. That image goes straight into a `std::vector`.

Now feed it to the second window.

```cpp
auto display_buf = vega.TextureBuffer(win1->width(), win1->height(), win1_container->get_image_format());

schedule_metro(1.0 / 60.0, [history, display_buf, win2]() {
    if (history->empty()) return;
    static bool registered = false;
    if (!registered) {
        register_graphics_buffer(display_buf, Buffers::ProcessingToken::GRAPHICS_BACKEND);
        display_buf->setup_rendering({ .target_window = win2 });
        registered = true;
    }
    const size_t idx = history->size() > 40 ? history->size() - 40 : 0;
    display_buf->set_gpu_texture((*history)[idx]);
}, "display");
```

`set_gpu_texture` takes the `VKImage` directly. Not a file path. Not a decoded buffer. The exact image that came out of `win1`'s own render, twenty seconds ago, is now what `win2` renders. Two windows, one playing forward, one playing the first one's own past.

Nobody declared a primary window. `win1` was never special, it just happened to be created first. `win2` isn't a debug view or an overlay, it's a peer that happens to consume the first one's output as its input. Either window could point at the other's history, or at its own. A window's rendered surface, once you stop assuming there has to be exactly one canvas that everything draws to, turns out to be exactly as available as any other piece of data in the system, because it always was data, the "canvas" was never a separate kind of thing.

There is no `draw()` here either, and nothing had to be removed to get this result. Both `schedule_metro` calls are coroutines resumed on their own schedule, doing exactly the work asked of them and nothing else. Neither window polls the other, neither polls anything. Each one renders because a buffer it's wired to changed, the same mechanism already at work in the pipeline above, just with two windows and a history vector standing in for "three buffers, three arrival paths."

## Closing the loop

The twenty-frame lag above is arbitrary, a fixed offset into history. Make it a decision instead.

A `SamplingPipeline` is playing an audio file, at a speed that changes based on what `win2` is currently showing:

```cpp
schedule_metro(1.0 / 60.0, [sampler, container /* win1_container */]() {
    double brightness = 0.0;
    uint32_t n = 0;
    for (uint32_t x = 0; x < container_width; x += 40, ++n)
        brightness += container->get_value_at({ container_height / 2, x, 0 });
    sampler->slice(0).speed = 0.5 + (brightness / n) * 1.5;
}, "brightness_drive");
```

Brighter frame, faster playback. And the sampler's own playback state, not a fixed offset, is what selects which frame from history gets shown next:

```cpp
const auto& sl = sampler->slice(0);
const double velocity = sl.speed * 0.4 + sl.cursor_remainder * 0.8;
img_cursor += velocity;
display_buf->set_gpu_texture(history[static_cast<size_t>(img_cursor) % history.size()]);
```

Now trace the circle all the way around. `win1` renders a frame. The capture metro reads that frame into history. The display metro picks a frame from history, using the sampler's speed to decide which one, and shows it in `win2`. The brightness metro reads `win2`'s current frame and sets the sampler's speed. The sampler's speed is what picked the frame that produced that brightness.

Nothing here is audio-reactive-visuals, and nothing here is visuals-driving-audio either, both descriptions assume a direction, a source domain and a destination domain, and there isn't one. There's a value (brightness), feeding a value (playback speed), feeding a value (cursor position), feeding a value (brightness), around a cycle with no privileged starting point and no domain boundary anywhere in it. Whether you call the middle step "audio" or "control data" was never a fact about the sampler. It's a fact about which root you registered it under, which happened elsewhere, for unrelated reasons, and does not change what the loop does.

Try placing this inside `while (!glfwWindowShouldClose(window))`. The same three problems show up immediately, and none of them are edge cases, they're the ordinary shape of the exact thing just built:

- **Two windows, one loop.** A `while` loop belongs to one window, or it belongs to neither and now has to poll both by hand: check each close flag, swap between two GL contexts, decide whose turn it is to `glfwSwapBuffers()` this iteration. Arbitrating two windows was never the loop's job, and now it has no choice.
- **Audio has its own thread.** `sampler->slice(0).speed` cannot be written safely from inside a loop that also owns rendering, because the audio callback runs on its own OS thread, on its own schedule, unrelated to `glfwPollEvents()`. Getting a value from render space into audio-callback space safely means a mutex, a lock-free ring buffer, or an atomic, built and defended by hand, for every value that needs to cross.
- **The history buffer needs a home.** Twenty seconds of accumulating frames needs somewhere to live that survives across iterations: no leaks as old frames get evicted, no stale pointers, no reading an index the capture code hasn't written yet. None of this is clever-difficult, it's tedious-difficult, bookkeeping reinvented because nothing beneath the loop already understood "one side fills this, one side drains it, and reading while writing is well-defined."

A `while` loop can be made to do all of it, given enough manual threading and enough state tracked in structs built to route around it. What it can't do is make any of that disappear, because it was designed to redraw one scene, once per tick, and everything asked of it beyond that becomes a workaround, not a feature.

A window that reads another window's history, closes a loop through an audio pipeline, and settles into a self-sustaining rhythm where each domain's state is entirely a function of the others, was never expressible as something you draw. It had to be expressible as something that flows.

## Anti primitive worship

A thinking model, not a specific tool, treats a small catalog of shapes as load-bearing: rectangle, circle, sphere, box. Reach for one, get exactly one, and if what you need isn't in the catalog, the next rung on the ladder is usually "load a mesh from disk." Nothing in between. Here's what that model actually costs, mechanically, independent of which framework happens to implement it.

**Every call re-does the setup.**

```cpp
// pseudocode, the shape any immediate-mode draw call takes
void draw_something(args...) {
    vbo = allocate_vertex_buffer(args);      // every call
    ibo = allocate_index_buffer(args);       // every call, if indexed
    pipeline = resolve_or_build_pipeline();  // every call
    shader = bind_default_shader();          // every call
    upload(vbo, ibo);                        // every call
    issue_draw_call();
}
```

Every draw call is expressed as a complete description of one thing to draw. Whether the implementation caches state or batches uploads underneath is an optimization. The programming model still asks you to think in independent drawing operations rather than persistent geometry, and no amount of caching under the hood changes what the programmer is holding in their head while they write the call.

MayaFlux doesn't optimize this pattern, it removes the pattern. A `GeometryWriterNode` is constructed once and referenced from then on:

```cpp
auto topo = vega.TopologyGeneratorNode(Kinesis::ProximityMode::K_NEAREST);
topo->add_point({.position = {0, 0, 0}});
topo->add_point({.position = {1, 0, 0}});
// later, same node, no re-construction:
topo->add_point({.position = {0.5F, 1, 0}}); // topology regenerates from the accumulated set
```

There was never a call to re-issue. `add_point` mutated a node that already exists, already has a GPU buffer bound to it, and already knows how to mark itself dirty. The upload that follows isn't a cache doing someone a favor, it's the only upload path this node has ever had.

**Nothing persists after the fact. It has to be re-issued.**

```cpp
// pseudocode
float x = 0.0F;

while (running) {
    x += 0.01F;              // state lives outside the call
    draw_rect(x, 0.0F, w, h); // the call has no memory of last frame
}
```

The rectangle was never a thing that persists and gets nudged. It's a verb, re-invoked, and every property that should feel like it's changing over time, position, size, color, has to live somewhere else, in a variable the programmer manages by hand, then gets re-fed into the call from scratch on the next tick. There is no rectangle to hold a reference to. There's only ever the moment of drawing one.

**The catalog has a floor and a ceiling, nothing between them.**

```cpp
draw_circle(...);   // floor: a name for a shape
draw_sphere(...);   // floor: a name for a shape

load_mesh("model.fbx"); // ceiling: someone else's authored geometry
```

What sits between "a name for a shape" and "a file someone else made in a different program" is where almost everything interesting actually lives: a shape defined by an equation, a shape that emerges from connecting a sparse set of points by some rule, a shape built from accumulating segments over time, a shape that responds to a signal every frame. None of that has a name in the catalog, and none of it is a file on disk. The model simply doesn't have a slot for it.

### What replaces the catalog

`GeometryWriterNode` is the base every generator in this system inherits from, and every one of its children holds a *set*, never a single shape:

```cpp
auto points = vega.PointCollectionNode();        // a set of points
auto lines  = vega.LineSegmentsNode();           // a set of independent segments
auto path   = vega.PathGeneratorNode(mode, ...); // a set of control points, interpolated
auto topo   = vega.TopologyGeneratorNode(mode);  // a set of points, connections inferred
auto mesh   = vega.MeshWriterNode();             // vertex bytes + index bytes, any topology
```

Add to any of these after construction, and the node holds the change:
The hierarchy is flat by design: pick the node that matches what you want to accumulate: points, independent segments, connected paths, inferred topology, or raw bytes; and the buffer handles the rest.

```cpp
auto segs = vega.LineSegmentsNode();
segs->add_line({.position = {-0.5F, 0.0F, 0.0F}}, {.position = {0.5F, 0.0F, 0.0F}});
segs->add_normal(path->get_all_vertices(), 0.05F); // derived from a different node's output
```

Nothing here got re-issued. The node accumulated a segment, then accumulated another derived from an unrelated path's own vertex data. It's still the same node, still holding the same growing set, on the next frame and the one after that.

**One writer, one buffer:**

```cpp
auto buffer = std::make_shared<GeometryBuffer>(topo);
buffer->setup_rendering({ .target_window = window, .topology = PrimitiveTopology::TRIANGLE_LIST });
```

**Several writers, independent topologies, one GPU upload:**

```cpp
auto composite = std::make_shared<CompositeGeometryBuffer>();
composite->add_geometry("path", path, PrimitiveTopology::LINE_STRIP);
composite->add_geometry("normals", lines, PrimitiveTopology::LINE_LIST);
composite->add_geometry("anchors", points, PrimitiveTopology::POINT_LIST);
composite->setup_processors(ProcessingToken::GRAPHICS_BACKEND);
```

Three nodes, three topologies, one upload, three draw calls issued from data that already lives together on the GPU. Nobody asked whether that's "batching," it's just what happens when more than one writer feeds the same buffer.

### The middle Kinesis actually fills

Between "name a shape" and "load someone else's file" is math, and there's a lot of it:

```cpp
// any two-parameter function becomes a surface: torus, Möbius band,
// spherical harmonic deformation, terrain shaped by an audio node's output
auto surface = Kinesis::generate_parametric_surface(
    [](float u, float v) { return my_surface_fn(u, v); }, u_segs, v_segs);

// a circle extruded along any 3D path, radius itself a function of position
auto tube = Kinesis::generate_tube(path_points,
    [](float t) { return 0.1F + 0.05F * std::sin(t * 6.28F); }, 12);

// a scalar field, any combination of distance functions, resolved into
// a triangle mesh via marching cubes: gyroids, blended metaballs, noise fields
auto isosurface = Kinesis::generate_sdf_mesh(field, bounds_min, bounds_max, 64, 64, 64);
```

None of these are catalog shapes. None of them are imported files. They're functions, evaluated at whatever resolution is asked of them, producing exactly the geometry the function describes and nothing the catalog happened to anticipate.

### None of this makes a circle harder to get

Sometimes a circle is just a circle. Kinesis doesn't push that need aside in favor of the math-first cases above, the same file that generates SDF isosurfaces also generates the shapes the catalog model would call primitives:

```cpp
auto circle_pts = Kinesis::generate_circle(center, radius, 64);
auto rect_pts   = Kinesis::generate_rectangle(center, width, height);
auto ngon_pts   = Kinesis::generate_regular_polygon(center, radius, sides);
auto box_mesh   = Kinesis::generate_box(center, half_extents);
auto wire_box   = Kinesis::cuboid_wireframe(center, half_extents, color);
```

The difference from the catalog model isn't that these are missing, it's what they return and where they go. `generate_circle` hands back a `std::vector<glm::vec3>`, not a rendered thing, and that vector still has to reach a `GeometryWriterNode` before anything appears on screen: a `PathGeneratorNode` for the outline, a `MeshWriterNode` if it needs filling. There's no special-cased `draw_circle` that shortcuts straight to the screen and skips the substrate everything else goes through. A circle enters the pipeline the same way a marching-cubes isosurface does. The only difference is how its vertices were computed.

### And when even the CPU round trip is too much

```cpp
auto buf = std::make_shared<ComputeMeshBuffer>(
    bounds_min, bounds_max, 40, 40, 40, /*iso_level=*/0.0F, "sdf_field.comp")
    | Graphics;
buf->setup_rendering({ .target_window = window });

schedule_metro(1.0 / 60.0, [buf]() {
    buf->get_field_processor()->set_time(get_elapsed_time());
    buf->set_dirty();
}, "sdf_anim");
```

The field is evaluated on the GPU. The isosurface is extracted on the GPU. The vertices are written directly into the same `VKBuffer` a `RenderProcessor` draws from. Nothing here ever exists as a `std::vector` on the CPU side, not even once, past the initial lookup table upload. CPU-generated meshes, imported meshes, Kinesis generators, and GPU-written fields all converge on the same representation: geometry already resident in GPU memory. The rendering pipeline doesn't distinguish how those vertices came into existence because, by the time they reach it, that history is no longer relevant.

## One geometry, many places, and a body that keeps changing shape

Two problems that look unrelated turn out to be the same problem asked twice: how do you draw the same thing many times without paying for it many times, and how do you keep drawing a thing whose actual shape won't hold still. Instancing answers the first. Mesh-as-two-spans answers the second. Neither one is a bolted-on feature with its own special type. Both are what falls out of treating geometry as data that can be multiplied or rewritten, rather than a fixed shape that has to be re-authored to change.

### Instancing: no dedicated type, because none was needed

There is no `Instance` class carrying its own transform, its own lifecycle, its own special-cased draw path. If you want instancing, you have geometry that describes instance positions, and a vertex shader that reads them. That's the whole idea, and everything below is just what it looks like worked out in full.

A template is built once, from anything that already produces a `GeometryWriterNode`:

```cpp
auto net = std::make_shared<InstanceNetwork>();
auto tmpl = std::make_shared<PathGeneratorNode>(mode, samples, max_control_points);

for (uint32_t i = 0; i < 8; ++i) {
    uint32_t idx = net->add_slot("spine_" + std::to_string(i), tmpl);
    net->get_slot(idx).transform = glm::rotate(glm::mat4{1},
        glm::radians(45.0F * i), glm::vec3(0, 1, 0));
}
```

This is close to what `membrane` actually does with its `Spine`: a single Bézier tube, its control points taken from the three farthest vertices of an entirely different piece of geometry (the `Lattice`) on every frame, instanced eight times in a ring. What gets uploaded to the GPU once is the template's vertex data. What gets uploaded every frame anything moves is a packed array of per-slot `mat4` transforms into one SSBO, and a single instanced draw call reading `gl_InstanceIndex` to pick the right matrix per copy. Eight draw calls were never issued. The shape, recomputed from another object's own live data, was never re-authored eight times either.

What varies is a field, not a loop of manually-tracked positions:

```cpp
auto op = net->create_operator<InstanceFieldOperator>();
op->bind_position(0, Kinesis::VectorField {
    [](glm::vec3 p) { return p + glm::vec3(0.01F, 0.0F, 0.0F); }
});
```

`VectorField` is a `Tendency<glm::vec3, glm::vec3>`, the same construct that drives a camera in the next section. Transforms don't get looped over and nudged by hand. A function gets applied to a position, and it happens to run once per slot because slots exist, not because anyone wrote a per-slot update.

The same operator can hand the whole job to the GPU instead of evaluating a CPU lambda per slot:

```cpp
auto exec = std::make_shared<Yantra::ShaderExecutionContext<>>(
    Yantra::GpuComputeConfig { "wave_field.comp", { 256, 1, 1 }, sizeof(WaveFieldPC) });
exec->in_out(0, initial_transforms).push(WaveFieldPC { N, GRID_W, 0.0F, amplitude, frequency, speed });

auto field_op = net->create_operator<InstanceFieldOperator>();
field_op->set_gpu_executor(std::move(exec), true);
```

`wave_field.comp` reads every slot's current transform from one SSBO and writes the displaced result back into the same buffer, in place, for however many slots exist, a grid of a hundred and forty-four in one dispatch is no different in kind from four. The operator that accepted a CPU `VectorField` a moment ago now accepts a compute shader instead, same slot semantics, same dirty propagation, same one instanced draw call at the end. Nothing about the API changed shape to allow this. The CPU and GPU paths were always the same seam.

### The same pull, applied to vertices instead of copies

Instancing multiplies one template. A related but distinct question: what if there isn't one template, there's a whole loaded network, dozens of named submeshes from an FBX import, and every one of them needs its own vertices pulled by its own field, at the same time, on the GPU.

```cpp
auto net = vega.read_mesh_network("res/fbx/Wolf.fbx") | Graphics;
auto buf = vega.MeshNetworkBuffer(net) | Graphics;
buf->setup_rendering({ .target_window = window });

auto exec = std::make_shared<Yantra::ShaderExecutionContext<>>(
    Yantra::GpuComputeConfig { "mesh_pull_field.comp", { 256, 1, 1 }, sizeof(MeshPullFieldPC) });
exec->in_out(0, all_slots_vertices_concatenated)
    .input(1, per_slot_vertex_offsets, Yantra::GpuBufferBinding::ElementType::UINT32)
    .push(initial_pc);

auto field_op = net->get_operator_chain()->emplace<MeshFieldOperator>();
field_op->set_gpu_executor(std::move(exec), false);
```

Every slot's vertex data gets packed sequentially into one buffer, dispatched once, and unpacked back into each slot's own `set_mesh_vertices()` by index range. A loaded wolf, however many parts it was authored in, gets pulled by a field that operates on raw vertex bytes with no idea any of it came from an FBX file, because by the time it reaches the shader it never was one. `MeshNetworkBuffer` already concatenates every slot into one combined vertex and index buffer for a single draw call, rebasing indices across slot boundaries, so the field operator is deforming exactly the same combined buffer the renderer draws from, not a separate copy that has to be reconciled back.

### Mesh: two spans, and neither one waits for the other

A mesh, loaded, generated, or deformed, is a span of vertex bytes and a span of indices, described by a layout. Nothing about that representation privileges provenance:

```cpp
auto buf = std::make_shared<MeshBuffer>(mesh_data);
buf->setup_rendering({ .target_window = window });
```

Whether `mesh_data` came from an FBX import, `Kinesis::generate_parametric_surface`, or a marching-cubes pass, `MeshBuffer` doesn't ask. What it does ask is which of the two spans changed:

```cpp
buf->set_vertex_data(new_bytes);   // marks vertices dirty only
buf->set_index_data(new_indices);  // marks indices dirty only
```

These are two independent atomics, checked and cleared separately by the upload path. A surface can be rewritten every frame while its connectivity stays untouched for seconds at a time, then restructured once, on some threshold, without the continuous vertex rewrite ever pausing to wait for it:

```cpp
schedule_metro(1.0 / 60.0, [mesh, envelope]() {
    auto verts = mesh->get_mesh_vertices();
    for (size_t i = 0; i < verts.size(); ++i)
        verts[i].position = base_verts[i].position
            + fault_dirs[i] * displacement_from(envelope, i);
    mesh->set_mesh_vertices(verts);
}, "deform");

schedule_metro(0.1, [mesh, envelope]() {
    if (envelope->get_last_output() > threshold)
        mesh->set_mesh_indices(retopologize(mesh->get_mesh_indices()));
}, "tear");
```

This is close to `compose_drone_disintegration`: fault directions computed once, displacement driven every frame by live spectral energy, and a separate, slower check on the same energy deciding when connectivity itself gives way. When the index rewrite fires, the mesh is not deformed. It is re-triangulated. Two objects that are geometrically identical but connected differently are not the same mesh, and no amount of vertex displacement produces that difference, only a rewritten index span does. This is only possible because vertex data and index data are separate streams that can be rewritten independently, at whatever two rates the situation actually calls for, sixty times a second for one, once past a threshold for the other.

Nothing about this is free of consequence, and it's worth being precise about where the boundary actually sits. `set_mesh_vertices` is not safe to call from just anywhere; it competes with the graphics processor uploading that same buffer, so it belongs in a coroutine or metro running on the correct execution context, not directly inside a high-frequency event callback like `on_mouse_move`, which can fire over a thousand times a second and has no business touching mesh data itself. The pattern that does work reliably: write a cheap atomic from the event callback, let a regularly-scheduled routine read that atomic and perform the actual mutation. The independence of vertex and index dirty flags doesn't waive the ordinary rule that GPU-bound state has one thread that's allowed to touch it.

## The camera and the light are the same idea

A camera positions a viewer. A light positions an effect on a surface. Two different jobs, and in almost every framework, two different object types, each with its own class, its own scene-graph slot, its own special-cased path through the renderer. Here they turn out to be the same thing twice: something that hands numbers to a shader, and nothing more, until `Nexus` optionally wraps it in a way that lets those same numbers reach several places at once.

### A camera is 128 bytes and a function

```cpp
struct ViewTransform {
    glm::mat4 view { 1.0F };
    glm::mat4 projection { 1.0F };
};
static_assert(sizeof(ViewTransform) == 128, "Vulkan minimum push constant size");
```

That's the entire type. No position field with an implied "forward" vector, no scene attachment, no lifecycle. It's sized to the Vulkan minimum push constant guarantee, but the transform itself doesn't travel as a push constant: `RenderProcessor` allocates it as a small UBO, at descriptor set 0, and rewrites that buffer's contents before every draw. Nothing about it assumes a camera exists behind it, and nothing about it is fixed once uploaded.

```cpp
render_processor->set_view_transform_source([envelope]() {
    return Kinesis::look_at_perspective(
        glm::vec3(0.0F, 0.0F, 4.0F + envelope->get_last_output() * 3.0F),
        glm::vec3(0.0F), glm::radians(55.0F), 16.0F / 9.0F, 0.01F, 1000.0F);
});
```

`set_view_transform_source` takes a callable and invokes it once per draw, and whatever it returns is what gets written into the UBO that cycle. That's the entirety of "having a camera": a function, called when the renderer needs the matrices, with total freedom over where the numbers come from. Whether the function reads a fixed eye position, an orbit computed from mouse drag, or, as in `matrix_and_mesh`'s `compose_resonant_orbit`, five formant resonator outputs setting azimuth, elevation, radius, field of view, and roll from live audio every frame, the render processor doesn't know the difference and was never written to care. Anything that can produce two `glm::mat4`s, a physics step, a recorded path, a network message, a value read off another buffer entirely, can be the thing filling that UBO on any given frame.

### A light is 48 bytes and a function

```cpp
struct InfluenceUBO {
    glm::vec3 position { 0.0F };
    float intensity { 1.0F };
    glm::vec3 color { 1.0F, 1.0F, 1.0F };
    float radius { 1.0F };
    float size { 1.0F };
};
static_assert(sizeof(InfluenceUBO) == 48, "std140 alignment");
```

Same pattern. No `Light` class, no falloff model baked into a type, no scene attachment. Just position, intensity, color, radius, size, the plain data a shader would want regardless of what's producing it.

```cpp
auto glow = std::make_shared<Nexus::Emitter>(
    [](const Nexus::InfluenceContext&) { /* no CPU-side effect needed */ });
glow->set_position(strike_position);
glow->set_intensity(strike_strength);
glow->set_influence_target(lattice_processor);   // set=1, binding=0, wired automatically
glow->set_influence_target(spine_processor);     // same UBO, another surface entirely
```

`set_influence_target` allocates the UBO once, registers the binding on whatever `RenderProcessor` it's given, and from that point on, every commit packs position/intensity/color/radius/size into it without further instruction. Call it again on a second, unrelated processor and both surfaces read the same influence, because it's the same number, read twice, not two lights kept in sync by hand.

Calling it a "light" at all is already an assumption worth dropping. `InfluenceUBO` has no idea it's lighting anything. The same struct, the same `InfluenceContext` that fills it, is what `membrane`'s cursor uses to strike a modal resonator: `ctx.position` becomes a strike location and a strength on the audio side, in the very same `invoke()` call that uploads that position into a shader as a lit point on a surface. Nothing in the type distinguishes "this instance is a light" from "this instance is a plectrum." A different influence function bound to the same `Emitter` could just as easily read that position and spawn geometry, retune a filter bank, decide when a mesh retriangulates, or drive three of those at once. The struct is a value. Whatever reads it decides what it means.

### Nexus doesn't replace this. It multiplies where the same number can go.

An `Emitter` is the bare version: a position, an influence function, optionally a render target. `Nexus::Locus`, the thing that plays the role of a camera in `membrane`, is not a different kind of object built on top of some `Camera` base. It's an `Agent`, the same influence machinery `Emitter` has, just also carrying a perception side and registered in a `Fabric` for spatial queries. Nothing about being a camera required a new type. It required the same 48-byte influence pattern, plus a view transform sourced from the same position.

This is exactly what `membrane` does: the Locus's position feeds three places from one commit. `get_view_transform_source` reads it to build the render processor's `ViewTransform`, so the camera looks from where the spatial agent is. `set_influence_target` reads the same position, intensity, and color into the influence UBO bound to each lit shader, so proximity becomes visible as light on the surface. And the influence callback itself reads a proximity value computed from that same position against the gyroid's bounding volume, driving warp strength and excitation. Three destinations, one number, because a camera and a light were never separate categories of object to begin with, just two different shaders reading two different small structs, and nothing stopped the same position from filling both.

## What was never a type

Four different questions ran through this document. What draws to the screen. What a shape is before it's a mesh. How one piece of geometry becomes many, and how a body keeps changing shape without falling apart. What a camera is, and what a light is. Four questions, and one answer kept showing up underneath all of them: none of these were ever special categories of thing. Each one turned out to be a value, sitting somewhere, read at whatever rate the situation actually called for.

`draw()` was never a fact about hardware, it was a fact about who was made to carry the hardware's timing in their head. A shape was never obligated to come from a fixed catalog or a file on disk, because a shape is just what a function produces when it's evaluated, and a function can be evaluated at any resolution asked of it. An instance was never a type either, just a template and a place to put a transform, multiplied as many times as there happen to be slots. A mesh was never one atomic thing, just two spans that are conventionally drawn together but were never required to change at the same speed. A camera was 128 bytes in a buffer. A light was 48. Neither carried a scene graph, a lifecycle, or a reason to exist as a class of its own, because neither one is a noun. They're both just numbers a shader was going to read anyway.

This is not four coincidences. It's one habit, applied four times: wherever a framework hands you a named object, ask what value that object is actually standing in for, and what rate it changes at. The name is almost always doing less work than it appears to. A `Camera` class earns its keep only if something about *being a camera* constrains what the numbers can be or where they can come from, and nothing does. The same is true of an `Instance`, a `Light`, a scheduled `draw()` tick, and, going back to where this document started, a `while` loop that assumes it alone owns the screen.

What actually does the constraining, in every one of these cases, is something real and worth keeping straight from the naming habit: a display needs frames at a steady rate, a descriptor set has a binding budget, a push constant block has a 128-byte floor, a vertex buffer needs a consistent layout to be drawn at all. Those are the luthier's wood grain, the resonant frequency that doesn't move no matter who's asking. Everything else, the class hierarchy, the special-cased draw path, the separate API for "generated" versus "loaded" geometry, was scaffolding built around those few real constraints, mistaken over time for being made of the same material.

Nexus is the clearest place to watch the difference land, because it doesn't undo any of this. `Locus` isn't a `Camera` with extra features, it's an `Agent`, the same influence machinery `Emitter` has, and it becomes a camera only because one of the things reading its position happens to be a view transform source. The same position reaches a shader as light, a resonator as a strike, a warp field as proximity, in the same commit, because it was never three separate systems that needed reconciling. It was one number, and three different readers.

That's the whole shape of what changes coming from a framework built the other way. Not less code for the same result, a different question asked at every point where a type used to be assumed: not "what class do I need," but "what value is this, and who else might want to read it."
