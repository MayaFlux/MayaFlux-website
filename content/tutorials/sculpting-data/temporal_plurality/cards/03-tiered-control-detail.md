---
build:
  list: never
---

#### Tutorial: Sequence

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });

    auto particles = vega.ParticleNetwork(
        300,
        glm::vec3(-1.5f, -1.5f, -0.5f),
        glm::vec3(1.5f, 1.5f, 0.5f),
        Kinesis::SpatialDistribution::RANDOM_VOLUME
    ) | Graphics;

    auto* physics = particles->create_operator<PhysicsOperator>();
    physics->set_drag(0.01f);
    physics->set_bounds_mode(PhysicsOperator::BoundsMode::BOUNCE);

    auto buffer = vega.NetworkGeometryBuffer(particles) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    MayaFlux::schedule_sequence({
        { 0.0, [physics]() {
            physics->set_turbulence_strength(0.8f);
        }},
        { 2.0, [physics]() {
            physics->set_turbulence_strength(0.0f);
            physics->enable_spatial_interactions(true);
            physics->set_interaction_radius(0.4f);
            physics->set_spring_stiffness(0.6f);
        }},
        { 2.5, [physics]() {
            physics->set_repulsion_strength(2.0f);
            physics->set_interaction_radius(0.2f);
        }},
        { 2.0, [physics]() {
            physics->enable_spatial_interactions(false);
            physics->set_drag(0.08f);
        }},
        { 2.0, [physics]() {
            physics->set_drag(0.01f);
            physics->set_turbulence_strength(0.4f);
        }},
    });
}
```

Run this. Particles start in chaotic turbulence, then settle as spatial interactions switch on and spring forces pull them into loose clusters. Repulsion tightens, clusters compress. Interactions cut and drag bleeds off momentum. Then turbulence returns at half strength and the arc ends.

The sequence is a vector of `(delay, callback)` pairs. Each delay is the interval after the previous event. Cumulative time: 0 + 2.0 + 2.5 + 2.0 + 2.0 = 8.5 seconds total.

#### Tutorial: EventChain

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });

    auto particles = vega.ParticleNetwork(
        300,
        glm::vec3(-1.5f, -1.5f, -0.5f),
        glm::vec3(1.5f, 1.5f, 0.5f),
        Kinesis::SpatialDistribution::RANDOM_VOLUME
    ) | Graphics;

    auto* physics = particles->create_operator<PhysicsOperator>();
    physics->set_drag(0.01f);
    physics->set_bounds_mode(PhysicsOperator::BoundsMode::BOUNCE);

    auto buffer = vega.NetworkGeometryBuffer(particles) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    Kriya::EventChain chain(*MayaFlux::get_scheduler());

    chain.then([physics]() {
             physics->set_turbulence_strength(0.8f);
         })
         .then([physics]() {
             physics->set_turbulence_strength(0.0f);
             physics->enable_spatial_interactions(true);
             physics->set_interaction_radius(0.4f);
             physics->set_spring_stiffness(0.6f);
         }, 2.0)
         .then([physics]() {
             physics->set_repulsion_strength(2.0f);
             physics->set_interaction_radius(0.2f);
         }, 2.5)
         .then([physics]() {
             physics->enable_spatial_interactions(false);
             physics->set_drag(0.08f);
         }, 2.0)
         .then([physics]() {
             physics->set_drag(0.01f);
             physics->set_turbulence_strength(0.4f);
         }, 2.0)
         .start();
}
```

Run this. The same arc as the sequence above, same timing, same result.

The structural difference is readability and composability. `EventChain` is a builder: each `.then()` returns the same chain, so you can read the choreography top to bottom. For short sequences the difference is minor; for longer choreographies with repetition or a completion callback, the chain's additional methods become significant.

Example: EventChain with repetition and completion

``` cpp
void compose() {
    auto window = MayaFlux::create_window({ "Time", 1200, 800 });

    auto particles = vega.ParticleNetwork(
        300,
        glm::vec3(-1.5f, -1.5f, -0.5f),
        glm::vec3(1.5f, 1.5f, 0.5f),
        Kinesis::SpatialDistribution::RANDOM_VOLUME
    ) | Graphics;

    auto* physics = particles->create_operator<PhysicsOperator>();
    physics->set_drag(0.02f);
    physics->set_bounds_mode(PhysicsOperator::BoundsMode::WRAP);

    auto buffer = vega.NetworkGeometryBuffer(particles) | Graphics;
    buffer->setup_rendering({ .target_window = window });
    window->show();

    Kriya::EventChain chain(*MayaFlux::get_scheduler());

    chain.then([physics]() {
             physics->set_attraction_point(glm::vec3(
                 get_uniform_random(-0.8f, 0.8f),
                 get_uniform_random(-0.8f, 0.8f),
                 0.0f
             ));
         }, 1.2)
         .repeat(3)
         .then([physics]() {
             physics->clear_attraction_point();
             physics->set_turbulence_strength(0.6f);
         }, 1.5)
         .on_complete([physics]() {
             physics->set_turbulence_strength(0.0f);
             physics->set_drag(0.02f);
         })
         .times(3)
         .start();
}
```

The attraction point jumps to four random positions 1.2 seconds apart, pulling the cloud toward each in sequence. Then attraction clears and turbulence scatters everything. That full arc runs 3 times. After the third pass `on_complete` cuts the turbulence and resets drag.

`.repeat(3)` appends 3 more copies of the first event: 1 + 3 = 4 attraction moves total. `.times(3)` runs the whole list 3 times from the start.

------------------------------------------------------------------------


{{< tutorial-detail title="Deep dive" >}}
Metro and node hooks repeat indefinitely - they have no concept of a beginning or an end. Sequence and `EventChain` are for computation with a defined shape over time: a finite set of moments, each happening once, in order. The difference between them is lifecycle management, not timing precision.


{{< tutorial-detail title="Expansion 1: Delays are relative, not absolute" >}}


Both `schedule_sequence` and `EventChain` use relative delays: each delay is the interval after the previous event, not a position on a global timeline. The first event's delay is measured from `start()` or from when the sequence is scheduled.

This has a practical consequence when inserting events mid-sequence. Adding a new event between two existing ones only requires specifying its delay from the preceding event - no other delay values need updating. If you need absolute timing, convert manually: t=0, t=2.5, t=4.0 becomes delays 0, 2.5, 1.5.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 2: `on_complete` fires on cancel too" >}}


`on_complete` fires regardless of how the chain stops - after the final event completes normally, and also when `cancel()` is called. An internal flag prevents it firing twice. This means `on_complete` is suitable for cleanup that must happen whether the chain runs to conclusion or is interrupted: restoring physics state, releasing a resource, enabling a UI element.

If you only want a callback on normal completion, put it as the final `.then()` with zero delay instead.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 3: `wait()` and `every()`" >}}


`.wait(seconds)` inserts a pause with no action - equivalent to `.then([](){}, seconds)` but expresses intent clearly:

``` cpp
chain.then([physics]() { physics->set_turbulence_strength(1.2f); })
     .wait(3.0)
     .then([physics]() { physics->set_turbulence_strength(0.0f); })
     .start();
```

`.every(interval, action)` is syntactic sugar for `.then(action, interval)` - clearer when describing a repeated periodic action rather than a one-time transition.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 4: When to use which" >}}


Metro has no end. Use it when the work is ongoing and structurally uniform - polling, continuous accumulation, periodic display updates. It knows nothing about what came before or after.

`schedule_sequence` has an end but no handle. Use it when you have a fixed choreography that will not need to be cancelled, repeated, or observed from outside. Declare it, fire it, forget it.

`EventChain` has an end and a handle. Use it when the sequence is part of a larger system: you need to know when it finishes, restart it from a key event, cancel it in response to something else, or compose it with another chain via `on_complete`. The fluent API is a secondary benefit; the lifecycle management is the reason to reach for it.

Node hooks (`on_impulse`, `on_increment`, `while_true`) have no inherent temporal shape. They fire as a consequence of computation that is already happening. Use them when the rate or condition is itself a signal, not a design decision made at construction time.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 5: `schedule_sequence` vs `EventChain` at a glance" >}}


Both are built on the same coroutine: one `SoundRoutine` that iterates its event list and `co_await SampleDelay{N}` between entries. The abstraction difference is entirely in the API surface, not in timing precision or scheduling behavior.

`schedule_sequence` is the minimal form: flat vector, fire-and-forget, no handle returned, runs once. `EventChain` gives you `.cancel()`, `.is_active()`, `.times()`, `.repeat()`, `.on_complete()`, `.wait()`, `.every()`.


{{< /tutorial-detail >}}


{{< tutorial-detail title="Expansion 6: Audio-reactive particles via `map_parameter`" >}}


`ParticleNetwork` exposes named parameters driveable by any node output via `map_parameter`. The physics are not limited to what you set at construction:

``` cpp
auto lfo = vega.Sine(0.15f) | Audio[0];
auto chaos = vega.Random();
lfo->set_frequency_modulator(chaos);
particles->map_parameter("turbulence", lfo, MappingMode::BROADCAST);

auto env = vega.Sine(0.05f);
particles->map_parameter("interaction_radius", env, MappingMode::BROADCAST);
```

Available broadcast parameters: `gravity_x`, `gravity_y`, `gravity_z`, `drag`, `turbulence`, `interaction_radius`, `spring_stiffness`. `BROADCAST` applies the same value to all particles. `ONE_TO_ONE` maps per-particle outputs from another network directly to per-particle parameters.

This composes with `EventChain`: the chain handles large structural transitions while mapped nodes handle continuous variation between those transitions.



{{< /tutorial-detail >}}
{{< /tutorial-detail >}}



