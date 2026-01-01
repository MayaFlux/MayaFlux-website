---
title: "MayaFlux"
---

{{< rawhtml >}}

<!-- ====================================== -->
<!--  CARD 1 — Digital Paradigm             -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Beyond Analog Metaphors</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">
        <p>
        Most creative coding tools simulate analog hardware—virtual knobs, cables, circuit boards.
        This imposes constraints that don't exist in computation.
        </p>

        <p><strong>MayaFlux embraces true digital paradigms.</strong></p>
    </div>

</div>

<!-- ====================================== -->
<!--  CARD 2 — What Makes MayaFlux Different -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>What Makes MayaFlux Different</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <div class="feature-grid">
            <div class="feature-card">
                <h3>Unified Data Streams</h3>
                <p>Audio, visual, and control signals are all numerical streams that can interact freely.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Digital-First Thinking</h3>
                <p>Recursive algorithms, grammar-defined operations, and non-analog logic are first-class citizens.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Live Coding with Lila</h3>
                <p>Modify C++ code at runtime via LLVM21 JIT—change algorithms without stopping audio.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Coroutine Temporal Control</h3>
                <p>C++20 coroutines treat time as compositional material, enabling sample-accurate scheduling.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Lock-Free Architecture</h3>
                <p>Atomic operations ensure real-time safety without locks or contention.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>GPU Compute Integration</h3>
                <p>Vulkan compute shaders integrate directly into audio streams.</p>
            </div>
        </div>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 3 — Philosophy                   -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Philosophy</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <div class="philosophy-points">

            <div class="philosophy-point">
                <h3>Data is Data</h3>
                <p>MayaFlux removes disciplinary boundaries—sound, visuals, control signals are all data.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Code as Creative Material</h3>
                <p>Data transformation is the creative act; programming is compositional structure.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Time as Structure</h3>
                <p>Temporal relationships are part of artistic expression, not implementation detail.</p>
            </div>

            <hr>
            <div class="philosophy-point">
                <h3>Hooks Everywhere</h3>
                <p>No protective abstractions; full access to computational machinery when you need it.</p>
            </div>

        </div>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 4 — Built From Necessity         -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Built From Necessity</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p>
        MayaFlux wasn’t built to improve on existing tools—it was built because they could not support the work already happening.
        It is the culmination of years of experience in performance, production, and education.
        <br>
        <br>
        Built by the author with:
        </p>
        <ul>
            <li><strong>15+ years of interdisciplinary performance</strong> across Chennai, Delhi, and the Netherlands</li>
            <li><strong>Production audio engineering</strong> for Unreal Engine 5 and Metro: Awakening VR</li>
            <li><strong>Experimental creative computing education</strong></li>
            <li>Experience pushing the limits of instruments, Eurorack, and DSP systems</li>
        </ul>

        <br>
        <br>
        <p>
        When tools protect you from complexity, they also protect you from possibility.
        </p>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 5 — Current Status               -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Current Status</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p><strong>Alpha:</strong> Core audio subsystem stable. Vulkan rendering in active development. Lila JIT working.</p>
        <hr>
        <p><strong>Production-ready:</strong> Audio processing, node graphs, memory management.</p>
        <hr>
        <p><strong>Validated:</strong> GPU compute routing, cross-modal connectors.</p>
        <hr>
        <p><strong>Future:</strong> Full 3D graphics stack, networking, OpenCV integration</p>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 6 — Call to Action               -->
<!-- ====================================== -->

<div class="card collapsible">
    <div class="collapsible-header">
        <h2>Ready to Explore?</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

        <p>
        MayaFlux is for creators who've outgrown callback-driven thinking and want unified streams across audio, visual, and algorithmic composition.
        </p>

        <p>
        <a href="/download/">Download Weave</a> ·
        <a href="https://mayaflux.github.io/MayaFlux/">Documentation</a> ·
        <a href="https://github.com/MayaFlux/MayaFlux">Source Code</a>
        </p>

    </div>

</div>

<div class="card collapsible wide">
    <div class="collapsible-header">
        <h2>Quick teaser</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">
    <pre><code class="language-cpp">

#pragma once
#define MAYASIMPLE
#include "MayaFlux/MayaFlux.hpp"

void settings() {
// Low-latency audio setup
auto& stream = MayaFlux::Config::get_global_stream_info();
stream.sample_rate = 48000;
}

void compose() {

    // 1. Create the bell
    auto bell = vega.ModalNetwork(
                    12,
                    220.0,
                    ModalNetwork::Spectrum::INHARMONIC)[0]
        | Audio;

    // 2. Create audio-driven logic
    auto source_sine = vega.Sine(0.2, 1.0f); // 0.2 Hz slow oscillator

    static double last_input = 0.0;
    auto logic = vega.Logic([](double input) {
        // Arhythmic: true when sine crosses zero AND going positive
        bool crossed_zero = (last_input < 0.0) && (input >= 0.0);
        last_input = input;
        return crossed_zero;
    });

    source_sine >> logic;

    // 3. When logic fires, excite the bell
    logic->on_change_to(true, [bell](auto& ctx) {
        if (ctx.value >= 0) {
            bell->excite(get_uniform_random(0.5f, 0.9f));
            bell->set_fundamental(get_uniform_random(220.0f, 1000.0f));
        }
    });

    // 4. Graphics (same as before)
    auto window = MayaFlux::create_window({ "Audio-Driven Bell", 1280, 720 });
    auto points = vega.PointCollectionNode(500) | Graphics;
    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window });
    window->show();

    // 5. Visualize: points grow when bell strikes (when logic fires)
    MayaFlux::schedule_metro(0.016, [points]() {
        static float angle = 0.0f;
        static float radius = 0.0f;

        if (last_input != 0) {
            angle += 0.5f; // Quick burst on strike
            radius += 0.002f;
        } else {
            angle += 0.01f; // Slow growth otherwise
            radius += 0.0001f;
        }

        if (radius > 1.0f) {
            radius = 0.0f;
            points->clear_points();
        }

        float x = std::cos(angle) * radius;
        float y = std::sin(angle) * radius * (16.0f / 9.0f);
        float brightness = 1.0f - (radius * 0.7f);

        points->add_point(Nodes::GpuSync::PointVertex {
            .position = glm::vec3(x, y, 0.0f),
            .color = glm::vec3(brightness, brightness * 0.8f, 1.0f),
            .size = 8.0f + radius * 4.0f });
    });
    </code></pre>
    </div>

</div>

{{< /rawhtml >}}
