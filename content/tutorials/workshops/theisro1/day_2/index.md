---
title: "ISRO Workshop: Day 2"
---

_Yesterday: resonant bodies, stochastic timing, points as visual output.
Today: rhythm machines, structural graphics, and three ways to organize geometry on the GPU.
The same drum engine drives five increasingly complex visual systems.
By the end, audio events create, deform, and destroy spatial structures._

---

## Tutorial: Rhythm Machine

```cpp
void compose() { day2_a_rhythm_machine(); }
```

Run this. Press **Space** to start. You hear a kick, snare, and hi-hat.
Three columns of red, green, and blue points rise and fall with each voice's energy.
Press **C** to make the hat pattern probabilistic instead of metronomic.

No samples. No presets. Each voice is built from two things: a waveform and an envelope.

### Expansion 1: Phasor Envelopes

<details>
<summary>Click to expand: Synthesis from ramps</summary>

A `Phasor` generates a linearly increasing ramp from 0 to 1, then wraps. When you call
`reset()`, the ramp restarts from 0. This is your "trigger."

The envelope is a `Polynomial` that applies an exponential decay to the phasor's output:

```cpp
auto kick_env = vega.Polynomial([](double x) {
    return std::exp(-x * 15.0);
});
kick_env->set_input_node(kick_phasor);
```

When the phasor resets, x goes to 0, `exp(0) = 1.0` (full amplitude).
As the phasor ramps up, `exp(-x * 15)` decays toward zero. Rate constant 15 gives a
fast decay. The snare uses 25 (faster). The hat uses 50 (nearly instantaneous).

The sine oscillator uses this envelope as its amplitude modulator:

```cpp
auto kick_tone = vega.Sine(55.0)[0] | Audio;
kick_tone->set_amplitude_modulator(kick_env);
```

55 Hz sine, channel 0 (left), amplitude controlled by the decay envelope.
`reset()` the phasor, the sine swells and decays. That is a kick drum.

The snare uses filtered noise instead of a sine: `Random() → FIR → * envelope`.
The hat is an 8 kHz sine with a very fast decay. All three follow the same pattern:
waveform × envelope, triggered by phasor reset.

</details>

### Expansion 2: schedule_pattern

<details>
<summary>Click to expand: Step-based sequencing</summary>

```cpp
schedule_pattern(
    [](uint64_t step) { return (step % 4 == 2); },
    [snare_phasor, state](std::any hit) {
        if (std::any_cast<bool>(hit)) snare_phasor->reset();
    },
    0.25, "snare_pattern");
```

`schedule_pattern` takes two callbacks and a step interval. The first callback receives the
current step number and returns a boolean: should this step produce an event? The second
callback receives that boolean and acts on it.

Step interval 0.25 means one step every 250 ms. `step % 4 == 2` means: fire on every
4th step, offset by 2. At 120 BPM (0.5s per beat), this produces a snare on beats 2 and 4.

The hat pattern uses probability when chaos mode is active:

```cpp
[state](uint64_t step) {
    if (state->chaos_mode)
        return get_uniform_random(0.0, 1.0) > 0.6;
    return true;
}
```

60% chance per step. Compare this to Day 1's stochastic timing sources (Brownian,
exponential). There, timing itself was continuous and irregular. Here, the grid is
fixed (steps are evenly spaced) and the decision of whether to fire is stochastic.
Both produce irregular patterns, but from different structural premises.

</details>

### Try It

```cpp
// Faster tempo: halve the kick interval
schedule_metro(0.25, [kick_phasor, state]() { ... }, "kick_layer");

// Change kick pitch on every hit
kick_phasor->reset();
kick_tone->set_frequency(get_uniform_random(40.0, 80.0));

// Different snare pattern: every 3rd step (polyrhythm against kick)
[](uint64_t step) { return (step % 3 == 0); }
```

---

## Tutorial: Rhythm Topology

```cpp
void compose() { day2_b_rhythm_topology(); }
```

Run this. Press **Space**. Same drum engine, completely different visual. Each drum hit
deposits a colored point at a random spatial location. A minimum spanning tree connects
them: lines emerge from proximity relationships. Kick points cluster near the center (red).
Snare points scatter in the upper half (green). Hat points ring the perimeter (blue).

Click anywhere to add a point manually (yellow). Press **R** to clear.

The graph is not drawn by the drum engine. It is inferred by the topology algorithm from
the positions that the drum engine happens to deposit. The structure is emergent.

### Expansion 1: TopologyGeneratorNode

<details>
<summary>Click to expand: Points define locations, connections emerge from geometry</summary>

```cpp
auto topology = vega.TopologyGeneratorNode(
    Kinesis::ProximityMode::MINIMUM_SPANNING_TREE, false, 200);
```

`TopologyGeneratorNode` takes a set of points and computes connections between them
based on a proximity algorithm. `MINIMUM_SPANNING_TREE` finds the tree that connects
all points with minimum total edge length. Other modes:

- `K_NEAREST`: connect each point to its k nearest neighbors
- `NEAREST_NEIGHBOR`: connect each point to its single nearest neighbor
- `RADIUS_THRESHOLD`: connect all points within a radius
- `SEQUENTIAL`: connect points in insertion order (chain)
- `GABRIEL`: connect points whose diametral sphere contains no other points

Each produces a different visual structure from the same point set.
The points are `LineVertex` (position, color, thickness) and the output is `LINE_LIST`
topology: pairs of vertices forming line segments.

`regenerate_topology()` recomputes all connections. This is called periodically (not
every frame), and is the expensive operation. Between regenerations, individual points
can be updated via `update_point()` without rebuilding the graph.

</details>

### Expansion 2: Audio Modulating Graph Properties

<details>
<summary>Click to expand: Energy reshapes the structure</summary>

Every 0.5 seconds, audio energy modulates the graph's visual properties:

```cpp
size_t target_k = 2 + static_cast<size_t>(kick_e * 4.0F);
topology->set_k_neighbors(target_k);
topology->set_line_color(glm::vec3(brightness, brightness * 0.8F, 1.0F), false);
topology->set_line_thickness(0.6F + kick_e * 2.0F, false);
topology->regenerate_topology();
```

Kick energy increases k (more connections, denser graph). Combined energy increases
brightness and thickness. The `false` parameter means "don't force uniform": each point
retains its own color, but the base tint shifts.

This is the first example where audio does not just trigger events or map to position,
but reshapes the structural connectivity of the visual representation. The graph topology
itself is a function of audio state.

</details>

### Try It

```cpp
// Switch to K_NEAREST for a different graph character
auto topology = vega.TopologyGeneratorNode(
    Kinesis::ProximityMode::K_NEAREST, false, 200);

// Deposit at mouse position instead of random (per-voice)
on_mouse_pressed(window, IO::MouseButtons::Left, [window, kick_phasor, topology](double x, double y) {
    kick_phasor->reset();
    topology->add_point({
        .position = glm::vec3(normalize_coords(x, y, window)),
        .color = glm::vec3(1.0F, 0.2F, 0.2F), .thickness = 2.5F });
});
```

---

## Tutorial: Living Topology

```cpp
void compose() { day2_c_living_topology(); }
```

Run this. Press **Space**. A Lissajous figure breathes: vertices radiate outward on kick
hits, shear on snare hits, jitter on clap hits. The topology lines stretch and compress
without being rebuilt. Every 16 hat hits, the proximity mode cycles (MST → KNN → nearest
neighbor → sequential chain).

Press **Q/W/E** to switch the vertex distribution (Lissajous / Fibonacci sphere / torus).

This is the critical difference from the previous example: the graph is not rebuilt every
frame. `update_point()` moves vertices. The topology lines follow because they reference
vertex indices, not positions. Snare hits are the only events that call
`regenerate_topology()`, creating a moment where the graph snaps to the new deformed
positions and reconnects.

### Expansion 1: Deformation Without Rebuild

<details>
<summary>Click to expand: update_point() vs regenerate_topology()</summary>

Each frame, all 34 vertices are displaced from their home positions:

```cpp
for (size_t i = 0; i < N; ++i) {
    // ... compute displaced position from home + expansion + shear + jitter ...
    topo->update_point(i, { .position = pos, .color = ..., .thickness = ... });
}
```

`update_point(index, vertex)` modifies vertex data in place. The GPU receives the new
positions on the next upload cycle. The topology connections (which pairs of vertices
are linked) remain unchanged. Lines stretch and compress as their endpoint vertices
move. This is O(N) per frame, always.

`regenerate_topology()` recomputes all connections from the current vertex positions.
This is O(N² log N) for MST. It only runs on snare hits. When it fires, the graph
structure snaps to match the current spatial arrangement. Vertices that drifted close
together get connected. Vertices that drifted apart lose their connections.

This creates a two-timescale system: continuous deformation (every frame) with
discrete structural updates (on snare hits). The visual result is a structure that
breathes and stretches fluidly, then periodically crystallizes into a new configuration.

</details>

### Expansion 2: Five Audio Layers, Five Effects

<details>
<summary>Click to expand: How each voice maps to a different deformation</summary>

- **Kick**: radial expansion. Vertices push outward from center. Decays over time (×0.97 per frame).
- **Snare**: shear rotation + topology regeneration. Upper and lower halves rotate in opposite directions. Each hit adds 0.2 radians of shear and rebuilds the graph.
- **Hat**: proximity mode cycle. Every 16 hits, the connection algorithm changes (MST → KNN → nearest → sequential). Same points, different structure.
- **Clap**: jitter injection. Each vertex receives a random displacement vector that decays over time (×0.93 per frame).
- **Bass**: line thickness. Continuous modulation, not event-driven. `bass->get_last_output()` directly scales the thickness parameter.

Five audio layers, five independent effects, one visual system. No routing, no "send."
Each callback reads the data it needs from the nodes it captures.

</details>

### Try It

```cpp
// Increase vertex count for denser structure
constexpr size_t N = 89;

// Make clap jitter much larger (dramatic displacement)
j = glm::vec3(
    static_cast<float>(get_uniform_random(-0.3, 0.3)),
    static_cast<float>(get_uniform_random(-0.3, 0.3)), 0.0F);

// Add a slow continuous rotation to all vertices
float global_angle = frame_count * 0.001F;
// Apply as additional transform after expansion + shear
```

---

## Tutorial: Rhythm Particles

```cpp
void compose() { day2_d_rhythm_particles(); }
```

Run this. Press **Space**. 120 particles in a Fibonacci sphere. They drift, repel,
and attract each other. Each drum voice imparts a different physical effect:
kick tightens the field (attraction accumulates), snare toggles between crystalline
and gaseous states, hat moves the attractor in an orbit, clap flips boundary mode
(wrap ↔ bounce).

Press **Q/W/E** for physics material presets: crystalline lattice, fluid medium, gas.
Left click to place an attractor. Right click to release it.
**1/2/3/4** for routing modes (left / right / stereo / ping-pong).

### Expansion 1: Three Geometry Organization Methods

<details>
<summary>Click to expand: GeometryBuffer vs NetworkGeometryBuffer vs CompositeGeometryBuffer</summary>

Day 1 used `GeometryBuffer`: one `GeometryWriterNode` (like `PointCollectionNode`),
one buffer, one draw call. You generate vertices, buffer uploads them, renderer draws them.

This example uses `NetworkGeometryBuffer`: a `NodeNetwork` (ParticleNetwork with 120 internal
PointNodes) drives the buffer. The buffer aggregates ALL internal node geometry into a single
GPU upload. The network's operator (PhysicsOperator) handles state evolution. You interact
with the network through physics parameters, not individual vertex manipulation.

```cpp
auto particles = vega.ParticleNetwork(120, bounds_min, bounds_max, distribution) | Graphics;
auto physics = particles->create_operator<PhysicsOperator>();
auto geom_buf = vega.NetworkGeometryBuffer(particles) | Graphics;
```

The third method, `CompositeGeometryBuffer` (next example), takes multiple independent
GeometryWriterNodes and aggregates them into a single GPU buffer, each rendered with
its own pipeline and topology.

Three levels:
- `GeometryBuffer`: one node → one buffer → one draw call
- `NetworkGeometryBuffer`: one network (many internal nodes) → one buffer → one draw call
- `CompositeGeometryBuffer`: many independent nodes → one buffer → multiple draw calls

</details>

### Expansion 2: Parameter Mapping for Continuous Modulation

<details>
<summary>Click to expand: Bass drone controls drag</summary>

```cpp
auto drag_modulator = vega.Polynomial([](double x) {
    return 0.04 + std::abs(x) * 0.08;
})[0] | Audio;
drag_modulator->set_input_node(bass_lfo);
particles->map_parameter("drag", drag_modulator, MappingMode::BROADCAST);
```

The bass drone's LFO (0.15 Hz) drives a polynomial that maps its output to a drag
range of 0.04 to 0.12. `map_parameter("drag", ...)` creates a persistent connection:
every physics update reads the modulator's last output and applies it as the drag
coefficient.

When the bass swells, drag increases, particles slow down, the field tightens.
When the bass dips, drag decreases, particles coast freely, the field loosens.

This is continuous, not event-driven. The mapping exists independently of the rhythm
engine. It creates a slow, breathing quality underneath the percussive events.

</details>

### Try It

```cpp
// Map kick energy to repulsion (particles explode on kick)
auto repulsion_mod = vega.Polynomial([](double x) {
    return 0.1 + std::abs(x) * 2.0;
})[0] | Audio;
repulsion_mod->set_input_node(kick_env);
particles->map_parameter("repulsion_strength", repulsion_mod, MappingMode::BROADCAST);

// Start with TORUS distribution instead of FIBONACCI_SPHERE
auto particles = vega.ParticleNetwork(120,
    glm::vec3(-0.5F), glm::vec3(0.5F),
    Kinesis::SpatialDistribution::TORUS) | Graphics;
```

---

## Tutorial: Composite Scene

```cpp
void compose() { day2_e_composite_scene(); }
```

Run this. Press **Space**. Three visual layers occupy the same window:
a breathing Catmull-Rom path (lines), a growing topology graph (lines with different
shaders), and energy dots (points). All three are driven by the same drum engine.
One GPU buffer upload per frame. Three independent draw calls.

Click to add topology points. Press **R** to clear topology and dots.

### Expansion 1: CompositeGeometryBuffer

<details>
<summary>Click to expand: Multiple topologies, one buffer</summary>

```cpp
auto composite = std::make_shared<CompositeGeometryBuffer>();
composite->setup_processors(ProcessingToken::GRAPHICS_BACKEND);
composite->add_geometry("path", path, PrimitiveTopology::LINE_LIST, window);
composite->add_geometry("topo", topo, PrimitiveTopology::LINE_LIST, window);
composite->add_geometry("dots", dots, PrimitiveTopology::POINT_LIST, window);
```

`CompositeGeometryBuffer` aggregates vertex data from all three nodes into a single
contiguous GPU buffer. The `CompositeGeometryProcessor` computes byte offsets and
vertex counts for each collection during the upload phase.

Each collection gets its own `RenderProcessor` with its own Vulkan pipeline (different
shaders, different topology). The RenderProcessor's `set_vertex_range()` tells it which
subset of the shared buffer to draw:

```
Buffer layout: [path vertices | topo vertices | dot vertices]
Render "path": draw from offset 0, count N_path, LINE_LIST pipeline
Render "topo": draw from offset N_path, count N_topo, LINE_LIST pipeline
Render "dots": draw from offset N_path + N_topo, count N_dots, POINT_LIST pipeline
```

One CPU → GPU upload. Three draw calls with independent pipeline state.
This is efficient: a single staging buffer transfer instead of three.

Without CompositeGeometryBuffer you would need three separate GeometryBuffers, each
with its own GPU allocation and upload. That works fine, but the composite approach
is cleaner when the geometry types are logically related and you want to manage them
as a unit.

</details>

### Expansion 2: Three Organization Methods Compared

<details>
<summary>Click to expand: When to use which</summary>

**GeometryBuffer** (Day 1, Day 2a):
- One GeometryWriterNode per buffer.
- You generate vertices directly (PointCollectionNode, PathGeneratorNode, TopologyGeneratorNode).
- Simplest. Use when you have one type of geometry with one topology.

**NetworkGeometryBuffer** (Day 2d):
- One NodeNetwork (containing many internal nodes) per buffer.
- The network's operator manages state (PhysicsOperator, TopologyOperator, PathOperator).
- Use for particle systems, point clouds, and any system where nodes have relationships.

**CompositeGeometryBuffer** (Day 2e):
- Multiple independent GeometryWriterNodes in one buffer.
- Each node has its own topology and pipeline.
- Use when you want mixed geometry types (points + lines + triangles) in one scene,
  with one buffer upload but independent rendering.

All three produce the same result at the GPU level: vertex data in a buffer, draw calls
with pipeline state. The difference is how you organize the CPU side: one node, one
network, or many nodes.

</details>

---

## Day 2 Summary

Same drum engine, five visual architectures.

**Audio:**
- Phasor reset as trigger mechanism (not Logic node, not stochastic)
- `schedule_pattern` for step-based sequencing with optional probability
- `schedule_metro` for periodic events (kick, modulation)
- Node multiplication (`filter * envelope`) for shaped noise
- `map_parameter` for continuous modulation (bass → drag)

**Graphics escalation:**
- Points as energy meters (bar chart: simplest)
- TopologyGeneratorNode: points deposited by audio, connections inferred by algorithm
- update_point() deformation: continuous displacement, discrete topology rebuild
- ParticleNetwork + PhysicsOperator: audio drives physical forces
- CompositeGeometryBuffer: mixed geometry types in one buffer

**Three geometry paths:**
- `GeometryBuffer`: one node, one buffer, one pipeline
- `NetworkGeometryBuffer`: one network, one buffer, one pipeline
- `CompositeGeometryBuffer`: many nodes, one buffer, many pipelines

**Conceptual shift from Day 1:**
Day 1 treated visuals as reactive output: "bell rings, spiral grows."
Day 2 treats visuals as structural material: audio events create, connect,
deform, and destroy spatial relationships. The visual system has its own
structural logic (topology algorithms, physics simulation, interpolation)
that audio modulates from outside. Neither domain is primary.
