---
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
        Most creative coding tools simulate analog hardware: virtual knobs, cables, circuit boards.
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
                <p>Audio, visual, and control signals share the same numerical substrate. A node output can route to an RtAudio callback and a Vulkan push constant simultaneously, with no translation layer between them.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Cross-Modal Node Bindings</h3>
                <p>Node outputs bind directly to GPU shader parameters. NodeBindingsProcessor writes scalar values to push constants; DescriptorBindingsProcessor handles vectors, matrices, and structured arrays via UBOs and SSBOs. Audio envelopes, spectral data, and control signals reach the GPU through the same node API, with no bridging code.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Physical Modelling Networks</h3>
                <p>ModalNetwork, WaveguideNetwork, and ResonatorNetwork implement physical modelling synthesis as first-class node graph citizens. Excitation, coupling, boundary conditions, and spatial routing are all live-configurable parameters.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Live Signal Matrix</h3>
                <p>IOManager provides a unified interface across the full signal matrix: audio files, video files, live camera devices, and image assets. All sources wire into the same buffer and processor architecture.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Live Coding with Lila</h3>
                <p>An embedded Clang interpreter evaluates arbitrary C++20 at runtime via LLVM ORC JIT. Latency is one buffer cycle. Algorithms change without stopping audio or tearing down the graphics context.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Coroutine Temporal Control</h3>
                <p>C++20 coroutines are the scheduling primitive. SampleDelay, FrameDelay, and MultiRateDelay awaiters let temporal intent cross domain boundaries. Time is compositional material, not a callback interval.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>Lock-Free Architecture</h3>
                <p>No mutexes in the real-time path. atomic_ref, CAS-based dispatch, and lock-free registration lists coordinate all four execution contexts: RtAudio callbacks, Vulkan render threads, async input backends, and user coroutines.</p>
            </div>

            <hr>
            <div class="feature-card">
                <h3>MIDI and HID Input</h3>
                <p>RtMidi and HIDAPI backends feed an async InputManager that dispatches to InputNode instances in the node graph. Controllers, sensors, and custom HID devices become signal sources with the same routing API as any other node.</p>
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
                <p>MayaFlux removes disciplinary boundaries: sound, visuals, control signals are all data.</p>
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
        MayaFlux wasn’t built to improve on existing tools, it was built because they could not support the work already happening.
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

        <p><strong>Version 0.2.0 : Alpha. Stable core.</strong></p>
        <hr>
        <p><strong>Stable:</strong> Audio processing, lock-free node graphs, coroutine scheduler, channel routing with crossfade transitions.</p>
        <hr>
        <p><strong>Stable:</strong> Vulkan 1.3 dynamic rendering, multi-window, geometry nodes, texture pipeline, compute shaders.</p>
        <hr>
        <p><strong>Stable:</strong> Live camera input (Linux, macOS, Windows), video file playback, image loading via IOManager.</p>
        <hr>
        <p><strong>Stable:</strong> MIDI and HID input backends, async dispatch, InputNode infrastructure.</p>

        <hr>
        <p><strong>Stable:</strong> Lila, LLVM ORC JIT for live C++20 evaluation at sub-buffer latency.</p>
        <hr>
        <p><strong>In progress:</strong> Kinesis mathematical substrate: spectral analysis, geometric computation, statistical operations.</p>
        <hr>
        <p><strong>Next (0.3):</strong> 3D expansion. The mathematical substrate and ViewTransform infrastructure are in place; 0.3 adds dedicated tooling, examples, and the surface area that makes 3D a first-class workflow rather than hooks only.</p>
        <hr>
        <p><strong>Planned (0.4):</strong> Kinesis as first-class analysis namespace: Eigen-level linear algebra, FluCoMa-level audio analysis primitives.</p>
        <hr>
        <p><strong>Planned (0.5):</strong> Native UI framework, building on existing Region and WindowContainer infrastructure.</p>

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

        <ul>
        <li><a href="/download/">Download Weave</a></li>
        <li><a href="https://mayaflux.github.io/MayaFlux/">Documentation</a></li>
        <li><a href="https://github.com/MayaFlux/MayaFlux">Source Code</a></li>
        <li><a href="/releases/"> Release Notes</a></li>
        <li><a href="/posters/cpponline-2026/"> C++ Online conference poster</a></li>
        </ul>

    </div>

</div>

<!-- ====================================== -->
<!--  CARD 7 — Quick teaser                 -->
<!-- ====================================== -->

<div class="card collapsible wide">
    <div class="collapsible-header">
        <h2>Quick teaser</h2>
        <p class="hint">Click to expand</p>
    </div>

    <div class="collapsible-body">

    <!-- Video will replace this code block when ready -->

<pre><code class="cpp">
void camera_warp()
{
    auto window = create_window({ .title = "Camera Warp", .width = 1920, .height = 1080 });

    // Open camera device — platform string: /dev/video0, "0", or "video=Integrated Camera"
    auto manager = get_io_manager();
    auto camera = manager->open_camera({ .device_name = "/dev/video0",
        .target_width = 1920,
        .target_height = 1080,
        .target_fps = 30.0 });

    // Wire camera into graphics buffer system
    auto tex = manager->hook_camera_to_buffer(camera);
    tex->setup_rendering({
        .target_window = window,
        .fragment_shader = "polar_warp.frag",
    });
    tex->get_render_processor()->enable_alpha_blending();
    window->show();

    // Three audio nodes — each drives one shader parameter
    auto osc = vega.Sine(5.F, 1.0);

    auto radial_node = vega.Polynomial([](double x) {
        return std::abs(x) * 0.5;
    }) | Graphics;
    radial_node->set_input_node(osc);

    auto angular_node = vega.Polynomial([](double x) {
        return std::abs(x) * 2.5;
    }) | Graphics;
    angular_node->set_input_node(osc);

    auto chroma_node = vega.Polynomial([](double x) {
        return std::abs(x) * 0.15;
    }) | Graphics;
    chroma_node->set_input_node(osc);

    // Bind node outputs directly to shader push constants — no bridging code
    struct Params {
        float radial_scale;
        float angular_velocity;
        float chroma_split;
    };

    auto bindings = create_processor<Buffers::NodeBindingsProcessor>(tex,
        Buffers::ShaderConfig { "polar_warp.frag" });

    bindings->set_push_constant_size<Params>();
    bindings->bind_node("radial", radial_node, offsetof(Params, radial_scale), sizeof(float));
    bindings->bind_node("angular", angular_node, offsetof(Params, angular_velocity), sizeof(float));
    bindings->bind_node("chroma", chroma_node, offsetof(Params, chroma_split), sizeof(float));
}
</code></pre>

</br>
polar_warp.frag:

<pre><code class="cpp">
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform Params {
    float radial_scale;
    float angular_velocity;
    float chroma_split;
};

void main() {
vec2 center = fragTexCoord - 0.5;

    float dist = length(center);
    float angle = atan(center.y, center.x);

    float r = dist + radial_scale * 0.1 * sin(dist * 20.0 + angular_velocity);
    float theta = angle + angular_velocity * 0.3 + sin(dist * 8.0) * radial_scale * 0.5;

    vec2 uv_r = vec2(0.5 + r * cos(theta - chroma_split), 0.5 + r * sin(theta - chroma_split));
    vec2 uv_g = vec2(0.5 + r * cos(theta), 0.5 + r * sin(theta));
    vec2 uv_b = vec2(0.5 + r * cos(theta + chroma_split), 0.5 + r * sin(theta + chroma_split));

    float red = texture(texSampler, uv_r).r;
    float green = texture(texSampler, uv_g).g;
    float blue = texture(texSampler, uv_b).b;

    outColor = vec4(red, green, blue, 1.0);

}

</code></pre>

    </div>

</div>

{{< /rawhtml >}}
