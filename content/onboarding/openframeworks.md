---
title: "From openFrameworks: Names Versus Numbers"

cascade:
  build:
    list: never
    publishResources: false
---

<div class="card wide">
<p>
openFrameworks gave you a lifecycle and a canvas. <code>setup()</code> once, <code>update()</code> and <code>draw()</code> every frame, one window, one OpenGL context, primitives you call by name.
</p>

<p>
MayaFlux gives you a substrate. There is no canvas, no draw call, no camera object, no mesh type distinct from any other block of numbers, and no clock sitting outside the code deciding when things happen. What there is: numbers flowing through a shared computational fabric, and coroutines that carry their own sense of time, resuming exactly when a sample position or a frame position, or both, arrives.
</p>

<p>
<strong>This isn't a smaller version of the same idea. It's a different category, and almost everything that felt like a fixed constraint in openFrameworks turns out to be a decision MayaFlux never had to make.</strong>
</p>
</div>

<div class="card">
<h2>What This Page Is</h2>

<p>
The <a href="../">Rosetta Stone landing page</a> already made the general case for unlearning tool-specific assumptions. This page is the openFrameworks-specific entry point, and it is deliberately thin. The actual weight lives in three documents below, each covering a cluster of ideas that only make sense once you've seen what they unlock, not just how they're spelled.
</p>

<p>
Every section in every one of those three documents is built around a concrete question: what could you not build before, and what specific mechanism makes it buildable now. Not "this architecture is flexible." A named consequence, stated as a fact about what happens.
</p>
</div>

<div class="card wide">
<h2>The One Reframe That Explains Everything Else</h2>

<p>
openFrameworks names things: a camera, a mesh, a clock. Each name comes with a fixed shape and a fixed set of things you're allowed to do with it. MayaFlux keeps asking the same question about every one of those names: is this a real boundary, or just a name someone gave a number at some point.
</p>

<p>
A view transform is two <code>glm::mat4</code> fields in a uniform buffer slot. If you put a matrix there, the vertex shader reads it. If you don't, nothing breaks, the geometry renders without a transform. There is no camera object behind it, just a function you supply that gets called every frame and happens to produce a matrix. A mesh is two byte spans, vertex data and index data, described by a layout. Whether those bytes came from an FBX import, a parametric surface, or last frame's audio amplitude crossing a threshold is not visible to anything downstream, because nothing downstream asked.
</p>

<p>
Time follows the same pattern, and it's the one most worth stating precisely rather than loosely. There is a class called <code>Clock</code> somewhere in the source, but that's just English, a convenient word for "the thing that scans a task list and decides who runs next." The coroutine itself is what carries time: its own promise holds a sample position and a frame position, and it suspends until one of them, or both, arrives. A coroutine that waits on both is not being synchronized by some external timer. It is time, in the same sense that a signal is a signal rather than a description of one.
</p>

<p>
None of this is a metaphor. <code>set_view_transform_source</code> takes a <code>std::function&lt;ViewTransform()&gt;</code> and calls it every frame. Five formant resonator outputs can compute azimuth, elevation, radius, field of view, and roll directly from live audio node output, with no camera being "controlled by audio." A vertex buffer can be rewritten every frame while its index buffer only rebuilds when amplitude crosses a threshold, two independent operations on one signal, with no mesh-specific code path treating either as special.
</p>
</div>

<div class="card-grid">

<div class="card vertical">
<h3><a href="rendering-model/">Document 1: The Rendering Model</a></h3>
<p>
<strong>What you learned:</strong> <code>setup/update/draw</code>, one window, <code>ofDrawX</code> primitives, meshes as a separate importer bolted onto the drawing API.
</p>
<p>
<strong>What changes:</strong> No lifecycle, no canvas, no primitive catalog. Geometry is math you generate, not a shape you call by name. Meshes, instancing, and batching all share one substrate instead of three separate code paths.
</p>
<p><em>"A mesh is two byte spans and a layout. Whether they came from an FBX file or a threshold crossing in an audio signal isn't visible to anything downstream, because nothing downstream asked."</em></p>
</div>

<div class="card vertical">
<h3><a href="live-gpu-dialogue/">Document 2: Constant, Live Dialogue With the GPU</a></h3>
<p>
<strong>What you learned:</strong> <code>ofShader::load()</code> as a compile-and-forget black box, readback as a screenshot feature, compute shaders as an occasional escape hatch.
</p>
<p>
<strong>What changes:</strong> Every binding, push constant, and specialization constant stays a live, rebindable object. A rendered frame is data you can read, crop, and feed back in. GPGPU is a peer discipline, not a visualizer trick.
</p>
<p><em>"The rendered surface becomes source material for its own history."</em></p>
</div>

<div class="card vertical">
<h3><a href="time-and-the-live-loop/">Document 3: Time, Coordination, and the Live Loop</a></h3>
<p>
<strong>What you learned:</strong> <code>ofGetElapsedTimef()</code> polled once per frame, audio as an addon library, edit-compile-relaunch as the unavoidable cost of changing anything.
</p>
<p>
<strong>What changes:</strong> Coroutines suspend on sample count, frame count, or both. Light is an influence function, not a scene object. Code changes in a running process, at the same latency as a single buffer cycle, without losing state.
</p>
<p><em>"Moving toward the gyroid changes all three geometries simultaneously, not through any explicit connection between them, but because they all read the same atomic value each frame."</em></p>
</div>

</div>

<div class="card wide">
<h2>Read Them in Any Order</h2>

<p>
These three documents cover disjoint concerns, not escalating difficulty. There is no prerequisite chain. If the rendering model is the loudest shock coming from openFrameworks, start there. If the live-coding story is what actually brought you here, start with the third. Each stands on its own.
</p>

<p>
Where useful, sections link out to <code>membrane</code>, the 0.4 showcase example, and to worked audio-driven viewport and mesh examples, as concrete proof rather than description. If a claim in these documents doesn't point to running code, that's a gap worth telling us about.
</p>
</div>

<div class="card wide">
<h2>This Page Will Change</h2>

<p>
The three documents it links to are still being written, and the code underneath all of it keeps moving. What's here now is the baseline: accurate as of today, verified against the actual source rather than asserted from memory, and built to be revised rather than treated as final.
</p>
</div>
