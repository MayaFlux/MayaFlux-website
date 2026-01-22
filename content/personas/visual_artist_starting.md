# Starting as a Visual Artist

_[Back to Personas](../)_

<div class="card wide">
<p>
You've been working with pixels, frames, rendering loops. Drawing to canvases, updating vertex buffers, managing draw calls.
MayaFlux asks you to think differently: not as rendering pipelines, but as data transformation substrate.
</p>

<p>
This isn't about making MayaFlux behave like Processing or openFrameworks. It's about discovering what becomes possible
when you treat visuals as numbers flowing through computational decisions rather than pixels painted to screens.
</p>
</div>

---

<div class="card">
<h2>Your First Image</h2>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Display Image", 1920, 1080 });

    // Load image from file
    IO::ImageReader img_reader;
    img_reader.open("your_image.png");
    auto tex_buffer = img_reader.create_texture_buffer();

    register_graphics_buffer(tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);

    tex_buffer->setup_rendering({
        .target_window = window
    });

    window->show();
}
```

<p>
<code>ImageReader::create_texture_buffer()</code> loads pixel data and creates a textured quad.
<code>setup_rendering()</code> configures default shaders and rendering.
<code>window->show()</code> makes it visible.
</p>

<p>
You didn't write a render loop. You declared what exists, and the substrate displays it at 60 FPS automatically.
The image isn't "drawn" — it exists as data that gets presented each frame.
</p>
</div>

---

<div class="card">
<h2>Moving the Image: Transform the Quad</h2>

<p>
Traditional: Manually update vertex positions in a draw loop.
MayaFlux: Declare position transformations as temporal events.
</p>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Moving Image", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({
        .target_window = window
    });

    window->show();

    // Circular motion
    MayaFlux::schedule_metro(0.016, [tex_buffer]() {  // ~60 FPS
        static float angle = 0.0f;
        angle += 0.05f;

        float x = std::cos(angle) * 0.3f;
        float y = std::sin(angle) * 0.3f;

        tex_buffer->set_position(x, y);
    });
}
```

<p>
<code>schedule_metro(0.016, ...)</code> creates a temporal event every 16ms (60 FPS).
<code>set_position(x, y)</code> marks the geometry as dirty — the processor regenerates vertices next frame.
Circular motion emerges from trigonometric functions, not manual animation keyframes.
</p>

<p>
Try different motion patterns:
</p>

```cpp
// Lissajous curve
MayaFlux::schedule_metro(0.016, [tex_buffer]() {
    static float t = 0.0f;
    t += 0.05f;

    float x = std::sin(3.0f * t) * 0.5f;
    float y = std::cos(5.0f * t) * 0.5f;

    tex_buffer->set_position(x, y);
});

// Random walk
MayaFlux::schedule_metro(0.016, [tex_buffer]() {
    static float x = 0.0f, y = 0.0f;

    x += (get_uniform_random() - 0.5f) * 0.02f;
    y += (get_uniform_random() - 0.5f) * 0.02f;

    // Keep in bounds
    x = std::clamp(x, -0.8f, 0.8f);
    y = std::clamp(y, -0.8f, 0.8f);

    tex_buffer->set_position(x, y);
});

// Scale pulsing
MayaFlux::schedule_metro(0.016, [tex_buffer]() {
    static float t = 0.0f;
    t += 0.05f;

    float scale = 0.7f + 0.3f * std::sin(t);
    tex_buffer->set_scale(scale, scale);
});

// Rotation
MayaFlux::schedule_metro(0.016, [tex_buffer]() {
    static float rotation = 0.0f;
    rotation += 0.02f;

    tex_buffer->set_rotation(rotation);
});
```

<p>
You're not "moving an image frame by frame." You're declaring mathematical transformations on vertex data that the substrate applies continuously.
</p>
</div>

---

<div class="card">
<h2>Distorting the Image: Manipulate UV Coordinates</h2>

<p>
Instead of moving the quad's position, you can warp how the texture is sampled by modifying UV coordinates.
</p>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "UV Distortion", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    // Get base quad vertices
    auto base_vertices = std::vector<TextureBuffer::QuadVertex>{
        { { -0.5f, -0.5f, 0.0f }, { 0.0f, 1.0f } },  // Bottom-left
        { {  0.5f, -0.5f, 0.0f }, { 1.0f, 1.0f } },  // Bottom-right
        { { -0.5f,  0.5f, 0.0f }, { 0.0f, 0.0f } },  // Top-left
        { {  0.5f,  0.5f, 0.0f }, { 1.0f, 0.0f } }   // Top-right
    };

    tex_buffer->setup_rendering({
        .target_window = window
    });

    window->show();

    // Wave distortion through UV manipulation
    MayaFlux::schedule_metro(0.016, [tex_buffer, base_vertices]() mutable {
        static float time = 0.0f;
        time += 0.05f;

        std::vector<TextureBuffer::QuadVertex> warped = base_vertices;

        for (auto& v : warped) {
            // Wave the UVs (which controls texture sampling)
            float wave = std::sin(v.texcoord.y * 10.0f + time) * 0.1f;
            v.texcoord.x += wave;
        }

        tex_buffer->set_custom_vertices(warped);
    });
}
```

<p>
<code>set_custom_vertices()</code> replaces the quad's vertex data entirely.
By modifying only the <code>texcoord</code> field (UV coordinates), you change where each vertex samples from the texture.
Result: The image appears to wave horizontally.
</p>

<p>
More complex UV distortions:
</p>

```cpp
// Spiral distortion
MayaFlux::schedule_metro(0.016, [tex_buffer, base_vertices]() mutable {
    static float time = 0.0f;
    time += 0.03f;

    std::vector<TextureBuffer::QuadVertex> warped = base_vertices;

    for (auto& v : warped) {
        // Center-relative UVs
        float cu = v.texcoord.x - 0.5f;
        float cv = v.texcoord.y - 0.5f;
        float dist = std::sqrt(cu * cu + cv * cv);

        // Spiral rotation based on distance
        float angle = dist * 10.0f + time;
        float cos_a = std::cos(angle);
        float sin_a = std::sin(angle);

        float new_u = cu * cos_a - cv * sin_a;
        float new_v = cu * sin_a + cv * cos_a;

        v.texcoord.x = 0.5f + new_u;
        v.texcoord.y = 0.5f + new_v;
    }

    tex_buffer->set_custom_vertices(warped);
});

// Zoom pulse
MayaFlux::schedule_metro(0.016, [tex_buffer, base_vertices]() mutable {
    static float time = 0.0f;
    time += 0.05f;

    std::vector<TextureBuffer::QuadVertex> warped = base_vertices;

    float zoom = 1.0f + 0.3f * std::sin(time);

    for (auto& v : warped) {
        // Scale UVs around center
        float cu = v.texcoord.x - 0.5f;
        float cv = v.texcoord.y - 0.5f;

        v.texcoord.x = 0.5f + cu * zoom;
        v.texcoord.y = 0.5f + cv * zoom;
    }

    tex_buffer->set_custom_vertices(warped);
});

// Perspective skew
MayaFlux::schedule_metro(0.016, [tex_buffer, base_vertices]() mutable {
    static float time = 0.0f;
    time += 0.05f;

    std::vector<TextureBuffer::QuadVertex> warped = base_vertices;

    float skew = std::sin(time) * 0.3f;

    for (size_t i = 0; i < warped.size(); i++) {
        // Skew based on vertical position
        float y_factor = warped[i].position.y;  // -0.5 to 0.5
        warped[i].texcoord.x += y_factor * skew;
    }

    tex_buffer->set_custom_vertices(warped);
});
```

<p>
You're not "distorting pixels." You're transforming the sampling coordinates that determine which texture pixels appear at which screen positions.
The image data never changes — only how it's accessed.
</p>
</div>

---

<div class="card">
<h2>Beyond Quads: Point Clouds and Geometry</h2>

<p>
Images and quads are just one way to work with visual data. For procedural forms, particle systems, and algorithmic geometry, use point collections.
</p>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Point Spiral", 1920, 1080 });

    auto points = vega.PointCollectionNode() | Graphics;

    // Generate spiral pattern
    for (int i = 0; i < 500; i++) {
        float t = i * 0.1f;
        float radius = t * 0.05f;
        float x = radius * std::cos(t);
        float y = radius * std::sin(t);

        // Color gradient
        float hue = static_cast<float>(i) / 500.0f;
        glm::vec3 color(
            std::sin(hue * 6.28f) * 0.5f + 0.5f,
            std::cos(hue * 6.28f) * 0.5f + 0.5f,
            std::sin(hue * 6.28f * 1.5f) * 0.5f + 0.5f
        );

        points->add_point({
            glm::vec3(x, y, 0.0f),
            color,
            8.0f
        });
    }

    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    window->show();
}
```

<p>
<code>PointCollectionNode</code> aggregates geometric data into a single structure.
Each point has position, color, and size. Add 500 points, get one buffer, one GPU upload, one draw call.
</p>

<p>
Growing structures over time:
</p>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Growing Spiral", 1920, 1080 });

    auto points = vega.PointCollectionNode(1000) | Graphics;
    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    window->show();

    static float angle = 0.0f;
    static float radius = 0.0f;

    MayaFlux::schedule_metro(0.016, [points]() {
        angle += 0.02f;
        radius += 0.001f;

        if (radius > 1.0f) {
            points->clear_points();
            radius = 0.0f;
        }

        // Add 10 new points each frame
        for (int i = 0; i < 10; i++) {
            float local_angle = angle + (i / 10.0f) * 6.28f;
            float x = std::cos(local_angle) * radius;
            float y = std::sin(local_angle) * radius;

            float brightness = 1.0f - radius;

            points->add_point({
                glm::vec3(x, y, 0.0f),
                glm::vec3(brightness, brightness * 0.8f, brightness * 0.5f),
                10.0f + (i * 2.0f)
            });
        }
    });
}
```

<p>
Points accumulate over time. When capacity is reached, clear and restart. Spiral patterns emerge from simple mathematical increments.
You're managing data that exists across temporal frames, not "drawing and erasing."
</p>

<p>
Animated patterns:
</p>

```cpp
// Lissajous curves
void compose() {
    auto window = MayaFlux::create_window({ "Lissajous", 1920, 1080 });

    auto points = vega.PointCollectionNode() | Graphics;

    for (int i = 0; i < 500; i++) {
        float t = i * 0.02f;
        float x = std::sin(3.0f * t) * 0.8f;
        float y = std::cos(5.0f * t) * 0.8f;
        float intensity = (std::sin(t * 2.0f) + 1.0f) * 0.5f;

        points->add_point({
            glm::vec3(x, y, 0.0f),
            glm::vec3(intensity, 1.0f - intensity, 0.5f),
            6.0f
        });
    }

    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    window->show();
}

// Wave interference
void compose() {
    auto window = MayaFlux::create_window({ "Wave Interference", 1920, 1080 });

    auto points = vega.PointCollectionNode() | Graphics;

    for (int x = -20; x <= 20; x++) {
        for (int y = -20; y <= 20; y++) {
            float nx = x / 20.0f;
            float ny = y / 20.0f;
            float dist = std::sqrt(nx * nx + ny * ny);
            float wave = std::sin(dist * 10.0f) * 0.5f + 0.5f;

            points->add_point({
                glm::vec3(nx * 0.9f, ny * 0.9f, 0.0f),
                glm::vec3(wave, wave, 1.0f),
                4.0f
            });
        }
    }

    auto buffer = vega.GeometryBuffer(points) | Graphics;
    buffer->setup_rendering({ .target_window = window });

    window->show();
}
```

<p>
These patterns aren't "drawn" — they're **data generated from mathematical functions**.
Position, color, size all emerge from equations. Change the equation, change the form.
</p>
</div>

---

<div class="card wide">
<h2>What You've Learned</h2>

<p>You now understand the foundations:</p>

<ul>
<li><strong>Images as data</strong> — load once, transform continuously</li>
<li><strong>Quad transformations</strong> — position, scale, rotation modify geometry</li>
<li><strong>UV manipulation</strong> — distort texture sampling without changing pixels</li>
<li><strong>Point collections</strong> — aggregate geometric data into efficient structures</li>
<li><strong>Temporal scheduling</strong> — <code>schedule_metro</code> for continuous evolution</li>
<li><strong>Mathematical generation</strong> — forms emerge from equations, not drawing commands</li>
</ul>

<p>
You didn't learn "how to use MayaFlux's rendering API." You learned to think about visuals as:
</p>

<ul>
<li>Data that exists in memory and transforms over time</li>
<li>Mathematical relationships that generate forms</li>
<li>Declarative structures rather than imperative draw calls</li>
</ul>

<p>
Traditional creative coding: "Draw this shape at this position every frame."
MayaFlux: "This data exists and evolves according to these rules."
</p>
</div>

<div class="card wide">
<h2>What Comes Next</h2>

<p>The examples above manipulate data on the CPU — positions calculated in C++, uploaded to GPU each frame.
But there's another paradigm: <strong>let the GPU do the computation.</strong></p>

<p>Instead of:</p>

<ol>
<li>Calculate 50,000 positions in C++</li>
<li>Upload to GPU (600KB per frame)</li>
<li>Render</li>
</ol>

<p>You can:</p>

<ol>
<li>Send 16 bytes of parameters to GPU</li>
<li>GPU calculates 50,000 positions in parallel</li>
<li>Render directly</li>
</ol>

<p><strong>Shaders</strong> are where this happens. You'll learn:</p>

<ul>
<li><strong>Fragment shaders</strong> — pixel-level transformations (distortions, effects, filters)</li>
<li><strong>Vertex shaders</strong> — procedural geometry generation on GPU</li>
<li><strong>Node bindings</strong> — audio data driving shader parameters in real-time</li>
<li><strong>Cross-domain flow</strong> — sound controlling visuals, visuals affecting sound</li>
</ul>

<p>
But the foundation remains the same: nodes produce data, buffers manage it, processors transform it, tokens schedule it.
The substrate doesn't change. Your access to it deepens.
</p>
</div>

<div class="card">
<h2>Shaders: Computation on the GPU</h2>

<p>
Everything so far calculated positions and colors on the CPU, then uploaded results to the GPU for display.
Now: send parameters to the GPU, let it compute everything in parallel.
</p>

<p>
Traditional approach: Calculate 50,000 positions in C++, upload 600KB per frame.
Shader approach: Send 16 bytes of parameters, GPU calculates 50,000 positions in parallel.
</p>

</div>

---

<div class="card">
<h2>Fragment Shaders: Pixel-Level Transformation</h2>

<p>
Fragment shaders run once per pixel. You write a function that transforms UV coordinates or colors.
The GPU executes it millions of times in parallel.
</p>

<p>
Start with the image distortion from earlier, but move the math into the shader:
</p>

<b>distort_wave.frag</b>

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float time;
    float frequency;
    float amplitude;
} params;

void main() {
    vec2 uv = fragTexCoord;

    // Horizontal wave distortion
    float wave = sin(uv.y * params.frequency + params.time) * params.amplitude;
    uv.x += wave;

    outColor = texture(texSampler, uv);
}
```

<b>C++ Code:</b>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Wave Distortion", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "distort_wave.frag" });

    window->show();

    // Create parameter nodes - processed at graphics rate
    auto time_node = vega.Phasor(50)[0] | Graphics; // Time accumulator

    // For constant values, use Polynomial in DIRECT mode with constant function
    auto freq_poly = vega.Polynomial([](double x) { return 10.0; }) | Graphics;

    auto amp_poly = vega.Polynomial([](double x) { return 0.1; }) | Graphics;

    // Push constant structure
    struct WaveParams {
        float time;
        float frequency;
        float amplitude;
    };

    // Create shader config
    auto shader_config = Buffers::ShaderConfig { "distort_wave.frag" };
    shader_config.push_constant_size = sizeof(WaveParams);

    // Create NodeBindingsProcessor
    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<WaveParams>();

    // Bind nodes to push constant offsets
    node_bindings->bind_node("time", time_node,
        offsetof(WaveParams, time), sizeof(float));
    node_bindings->bind_node("frequency", freq_poly,
        offsetof(WaveParams, frequency), sizeof(float));
    node_bindings->bind_node("amplitude", amp_poly,
        offsetof(WaveParams, amplitude), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
<code>layout(push_constant)</code> declares parameters that update every frame.
<code>NodeBindingsProcessor</code> automatically writes node outputs to these parameters.
The GPU runs <code>main()</code> for every pixel — 2 million times at 1080p — in parallel.
</p>

<p>
No UV manipulation loop. No vertex updates. Just: "Here's the math, GPU execute it everywhere."
</p>

---

<p>
More complex distortions:
</p>

**distort_spiral.frag**

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float time;
    float spiral_intensity;
    float zoom;
} params;

void main() {
    vec2 uv = fragTexCoord;
    vec2 center = vec2(0.5, 0.5);

    // Center-relative coordinates
    vec2 delta = uv - center;
    float dist = length(delta);
    float angle = atan(delta.y, delta.x);

    // Spiral distortion
    float spiral = angle + dist * 10.0 + params.time;

    // Apply rotation
    vec2 distorted;
    distorted.x = dist * cos(spiral);
    distorted.y = dist * sin(spiral);

    // Scale and recenter
    distorted = distorted * params.zoom * params.spiral_intensity + center;

    outColor = texture(texSampler, distorted);
}
```

<b>C++ Code:</b>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Spiral", 1920, 1080 });
    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "distort_spiral.frag" });

    window->show();

    auto time_node = vega.Phasor(30)[0] | Graphics;

    // Modulated intensity: 0.5 + 0.3 * sin(time * 0.5)
    auto intensity_osc = vega.Sine(0.015, 0.3) | Graphics; // 0.5Hz * 0.03 = 0.015, amp 0.3
    auto intensity_base = vega.Polynomial([](double x) { return 0.5; }) | Graphics;
    auto intensity_node = intensity_base + intensity_osc;

    // Modulated zoom: 1.0 + 0.2 * cos(time * 0.3)
    // Note: We don't have cosine, so use Sine with phase offset or Polynomial
    auto zoom_base = vega.Polynomial([](double x) { return 1.0; })[0] | Graphics;
    auto zoom_osc = vega.Sine(0.009, 0.2)[0] | Graphics; // Approximate cos behavior
    auto zoom_node = zoom_base + zoom_osc;

    struct SpiralParams {
        float time;
        float spiral_intensity;
        float zoom;
    };

    auto shader_config = Buffers::ShaderConfig { "distort_spiral.frag" };
    shader_config.push_constant_size = sizeof(SpiralParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<SpiralParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(SpiralParams, time), sizeof(float));
    node_bindings->bind_node("intensity", intensity_node,
        offsetof(SpiralParams, spiral_intensity), sizeof(float));
    node_bindings->bind_node("zoom", zoom_node,
        offsetof(SpiralParams, zoom), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

**distort_kaleidoscope.frag**

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float time;
    float segments;
    float rotation;
} params;

void main() {
    vec2 uv = fragTexCoord;
    vec2 center = vec2(0.5, 0.5);

    vec2 delta = uv - center;
    float angle = atan(delta.y, delta.x) + params.rotation;
    float dist = length(delta);

    // Kaleidoscope mirror segments
    float segment_angle = 6.28318 / params.segments;
    angle = mod(angle, segment_angle);

    // Mirror every other segment
    if (mod(floor(angle / segment_angle), 2.0) > 0.5) {
        angle = segment_angle - angle;
    }

    // Reconstruct UV
    vec2 kaleidoscope_uv;
    kaleidoscope_uv.x = 0.5 + dist * cos(angle);
    kaleidoscope_uv.y = 0.5 + dist * sin(angle);

    outColor = texture(texSampler, kaleidoscope_uv);
}
```

<b>C++ Code:</b>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Kaleidoscope", 1920, 1080 });
    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "distort_kaleidoscope.frag" });

    window->show();

    auto time_node = vega.Phasor(0.2)[0] | Graphics;

    // Segments: 6.0 + 4.0 * sin(time * 0.3)
    auto seg_base = vega.Polynomial([](double x) { return 6.0; })[0] | Graphics;
    auto seg_osc = vega.Sine(0.006, 4.0)[0] | Graphics; // 0.3Hz * 0.02 = 0.006
    auto segments_node = seg_base + seg_osc;

    // Rotation: time * 0.5
    auto rotation_scale = vega.Polynomial([](double x) { return 0.5; })[0] | Graphics;
    auto rotation_node = time_node * rotation_scale;

    struct KaleidoscopeParams {
        float time;
        float segments;
        float rotation;
    };

    auto shader_config = Buffers::ShaderConfig { "distort_kaleidoscope.frag" };
    shader_config.push_constant_size = sizeof(KaleidoscopeParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<KaleidoscopeParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(KaleidoscopeParams, time), sizeof(float));
    node_bindings->bind_node("segments", segments_node,
        offsetof(KaleidoscopeParams, segments), sizeof(float));
    node_bindings->bind_node("rotation", rotation_node,
        offsetof(KaleidoscopeParams, rotation), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

**distort_feedback.frag**

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float time;
    float feedback_amount;
    float zoom;
} params;

void main() {
    vec2 uv = fragTexCoord;

    // Sample texture to distort UVs based on image content itself
    vec2 sample_offset = texture(texSampler, uv).rg - 0.5;

    // Feedback loop: texture influences its own sampling
    uv += sample_offset * params.feedback_amount;

    // Zoom to prevent boundary artifacts
    vec2 center = vec2(0.5, 0.5);
    uv = (uv - center) * params.zoom + center;

    outColor = texture(texSampler, uv);
}
```

<b>C++ Code:</b>

````cpp
void compose()
{
    auto window = MayaFlux::create_window({ "Feedback Distortion", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "distort_feedback.frag" });

    window->show();

    auto time_node = vega.Phasor(0.02) | Graphics;

    // Feedback amount oscillates
    auto feedback_base = vega.Polynomial([](double x) { return 0.3; }) | Graphics;
    auto feedback_mod = vega.Sine(0.008, 0.2) | Graphics; // Slow modulation
    auto feedback_node = (feedback_base + feedback_mod) | Graphics;

    // Zoom varies to prevent runaway feedback
    auto zoom_base = vega.Polynomial([](double x) { return 1.0; }) | Graphics;
    auto zoom_mod = vega.Sine(0.006, 0.05) | Graphics;
    auto zoom_node = (zoom_base + zoom_mod) | Graphics;

    struct FeedbackParams {
        float time;
        float feedback_amount;
        float zoom;
    };

    auto shader_config = Buffers::ShaderConfig { "distort_feedback.frag" };
    shader_config.push_constant_size = sizeof(FeedbackParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<FeedbackParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(FeedbackParams, time), sizeof(float));
    node_bindings->bind_node("feedback", feedback_node,
        offsetof(FeedbackParams, feedback_amount), sizeof(float));
    node_bindings->bind_node("zoom", zoom_node,
        offsetof(FeedbackParams, zoom), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
    ```

</div>

---

<div class="card">
<h2>Vertex Shaders: Procedural Geometry</h2>

<p>
Vertex shaders run once per vertex. But vertices don't need to come from CPU data—you can generate positions from <code>gl_VertexIndex</code>.
</p>

<p>
Remember the spiral from earlier? We generated 500 positions in C++. Now: generate them on GPU.
</p>

**spiral.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float time;
    float radius_scale;
    float turn_speed;
} params;

void main() {
    // gl_VertexIndex is our iterator: 0, 1, 2, 3...
    float t = float(gl_VertexIndex) * 0.1;

    // Spiral equation
    float radius = t * 0.05 * params.radius_scale;
    float angle = t + params.time * params.turn_speed;

    float x = radius * cos(angle);
    float y = radius * sin(angle);

    gl_Position = vec4(x, y, 0.0, 1.0);

    // Color gradient along spiral
    float hue = float(gl_VertexIndex) / 500.0;
    fragColor = vec3(
        sin(hue * 6.28) * 0.5 + 0.5,
        cos(hue * 6.28) * 0.5 + 0.5,
        sin(hue * 9.42) * 0.5 + 0.5
    );

    gl_PointSize = 8.0;
}
````

**simple_passthrough.frag**

```cpp
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

**C++ Code:**

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "GPU Spiral", 1920, 1080 });

    auto points = vega.PointCollectionNode(500) | Graphics;
    for (int i = 0; i < 500; i++) {
        points->add_point({
            glm::vec3(0.0f), glm::vec3(1.0f), 1.0f // Dummy data
        });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "spiral.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    auto time_node = vega.Phasor(100)[0] | Graphics;
    auto radius_node = vega.Polynomial([](double x) { return 1.5; })[0] | Graphics;
    auto speed_node = vega.Polynomial([](double x) { return 0.3; })[0] | Graphics;

    struct SpiralParams {
        float time;
        float radius_scale;
        float turn_speed;
    };

    auto shader_config = Buffers::ShaderConfig { "spiral.vert" };
    shader_config.push_constant_size = sizeof(SpiralParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<SpiralParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(SpiralParams, time), sizeof(float));
    node_bindings->bind_node("radius", radius_node,
        offsetof(SpiralParams, radius_scale), sizeof(float));
    node_bindings->bind_node("speed", speed_node,
        offsetof(SpiralParams, turn_speed), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
The vertex data is meaningless—just a count. The shader generates positions from <code>gl_VertexIndex</code>.
500 vertices uploaded once. Every frame: send 12 bytes of parameters, GPU computes 500 positions.
</p>

---

<p>
More complex procedural geometry:
</p>

**lissajous.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float time;
    float freq_x;
    float freq_y;
    float phase_offset;
} params;

void main() {
    float t = float(gl_VertexIndex) * 0.02 + params.time;

    // Lissajous curve equations
    float x = sin(params.freq_x * t) * 0.8;
    float y = cos(params.freq_y * t + params.phase_offset) * 0.8;

    gl_Position = vec4(x, y, 0.0, 1.0);

    float intensity = (sin(t * 2.0) + 1.0) * 0.5;
    fragColor = vec3(intensity, 1.0 - intensity, 0.5);

    gl_PointSize = 6.0;
}
```

<b> C++ Code:</b>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Lissajous", 1920, 1080 });

    auto points = vega.PointCollectionNode(500) | Graphics;
    for (int i = 0; i < 500; i++) {
        points->add_point({ glm::vec3(0.0f), glm::vec3(1.0f), 1.0f });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "lissajous.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    auto time_node = vega.Phasor(20)[0] | Graphics;
    auto freq_x = vega.Polynomial([](double x) { return 3.0; })[0] | Graphics;
    auto freq_y = vega.Polynomial([](double x) { return 5.0; })[0] | Graphics;
    auto phase = vega.Polynomial([](double x) { return 0.0; })[0] | Graphics;

    struct LissajousParams {
        float time;
        float freq_x;
        float freq_y;
        float phase_offset;
    };

    auto shader_config = Buffers::ShaderConfig { "lissajous.vert" };
    shader_config.push_constant_size = sizeof(LissajousParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<LissajousParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(LissajousParams, time), sizeof(float));
    node_bindings->bind_node("freq_x", freq_x,
        offsetof(LissajousParams, freq_x), sizeof(float));
    node_bindings->bind_node("freq_y", freq_y,
        offsetof(LissajousParams, freq_y), sizeof(float));
    node_bindings->bind_node("phase", phase,
        offsetof(LissajousParams, phase_offset), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

**attractor.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float time;
    float sigma;
    float rho;
    float beta;
} params;

void main() {
    // Integrate Lorenz equations in vertex shader
    vec3 pos = vec3(0.1, 0.0, 0.0);
    float dt = 0.01;

    // Iterate based on vertex ID
    for (int i = 0; i < gl_VertexIndex && i < 10000; i++) {
        float dx = params.sigma * (pos.y - pos.x);
        float dy = pos.x * (params.rho - pos.z) - pos.y;
        float dz = pos.x * pos.y - params.beta * pos.z;

        pos.x += dx * dt;
        pos.y += dy * dt;
        pos.z += dz * dt;
    }

    // Project 3D to 2D
    gl_Position = vec4(pos.xy * 0.02, 0.0, 1.0);

    // Color by trajectory position
    float alpha = float(gl_VertexIndex) / 10000.0;
    fragColor = vec3(alpha * 0.8 + 0.2);

    gl_PointSize = 2.0;
}
```

<b>C++ Code:</b>

````cpp
void compose()
{
    auto window = MayaFlux::create_window({ "Lorenz Attractor", 1920, 1080 });

    auto points = vega.PointCollectionNode(10000) | Graphics;
    for (int i = 0; i < 10000; i++) {
        points->add_point({ glm::vec3(0.0f), glm::vec3(1.0f), 1.0f });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "attractor.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    auto time_node = vega.Phasor(0.01) | Graphics;

    // Lorenz parameters - can be modulated for morphing behavior
    auto sigma_base = vega.Polynomial([](double x) { return 10.0; }) | Graphics;
    auto sigma_mod = vega.Sine(0.003, 0.5) | Graphics; // Subtle modulation
    auto sigma_node = (sigma_base + sigma_mod) | Graphics;

    auto rho_node = vega.Polynomial([](double x) { return 28.0; }) | Graphics;
    auto beta_node = vega.Polynomial([](double x) { return 8.0 / 3.0; }) | Graphics;

    struct AttractorParams {
        float time;
        float sigma;
        float rho;
        float beta;
    };

    auto shader_config = Buffers::ShaderConfig { "attractor.vert" };
    shader_config.push_constant_size = sizeof(AttractorParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<AttractorParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(AttractorParams, time), sizeof(float));
    node_bindings->bind_node("sigma", sigma_node,
        offsetof(AttractorParams, sigma), sizeof(float));
    node_bindings->bind_node("rho", rho_node,
        offsetof(AttractorParams, rho), sizeof(float));
    node_bindings->bind_node("beta", beta_node,
        offsetof(AttractorParams, beta), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
    ```

**mandala.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float time;
    float spike_frequency;
    float spike_amplitude;
    float rotation_speed;
} params;

void main() {
    int total_angles = 360;
    int total_layers = 20;

    int layer = gl_VertexIndex / total_angles;
    int angle_index = gl_VertexIndex % total_angles;

    if (layer >= total_layers) {
        gl_Position = vec4(0.0, 0.0, -1.0, 1.0);
        return;
    }

    float angle = float(angle_index) * 3.14159 / 180.0;
    float layer_phase = float(layer) * 0.1;

    // Base radius
    float base_radius = 0.3 + float(layer) * 0.02;

    // Spike function
    float spike_amp = params.spike_amplitude * sin(float(layer) * 0.3 + params.time);
    float spike = spike_amp * pow(abs(sin(angle * params.spike_frequency)), 3.0);

    float r = base_radius + spike;

    // Rotate
    float rotated_angle = angle + params.time * params.rotation_speed + layer_phase;

    float x = r * cos(rotated_angle);
    float y = r * sin(rotated_angle);

    gl_Position = vec4(x, y, 0.0, 1.0);

    // Color
    float brightness = 0.3 + r * 0.5;
    fragColor = vec3(brightness, brightness * 0.9, brightness * 0.1);

    if (abs(sin(angle * params.spike_frequency)) > 0.9) {
        fragColor = vec3(1.0, 0.95, 0.8);
    }

    gl_PointSize = 4.0;
}
````

<b>C++ Code:</b>

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Mandala", 1920, 1080 });

    const uint32_t vertex_count = 360 * 20;
    auto points = vega.PointCollectionNode(vertex_count) | Graphics;
    for (uint32_t i = 0; i < vertex_count; i++) {
        points->add_point({ glm::vec3(0.0f), glm::vec3(1.0f), 1.0f });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "mandala.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    auto time_node = vega.Phasor(0.02) | Graphics;

    // Varying spike count: 7.0 + 2.0 * sin(time * 0.3)
    auto spike_base = vega.Polynomial([](double x) { return 7.0; }) | Graphics;
    auto spike_mod = vega.Sine(0.006, 2.0) | Graphics; // 0.3Hz * 0.02 = 0.006
    auto spike_freq_node = (spike_base + spike_mod) | Graphics;

    auto spike_amp_node = vega.Polynomial([](double x) { return 0.4; }) | Graphics;
    auto rotation_speed_node = vega.Polynomial([](double x) { return 0.3; }) | Graphics;

    struct MandalaParams {
        float time;
        float spike_frequency;
        float spike_amplitude;
        float rotation_speed;
    };

    auto shader_config = Buffers::ShaderConfig { "mandala.vert" };
    shader_config.push_constant_size = sizeof(MandalaParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<MandalaParams>();

    node_bindings->bind_node("time", time_node,
        offsetof(MandalaParams, time), sizeof(float));
    node_bindings->bind_node("spike_freq", spike_freq_node,
        offsetof(MandalaParams, spike_frequency), sizeof(float));
    node_bindings->bind_node("spike_amp", spike_amp_node,
        offsetof(MandalaParams, spike_amplitude), sizeof(float));
    node_bindings->bind_node("rotation", rotation_speed_node,
        offsetof(MandalaParams, rotation_speed), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

</div>

---

<div class="card wide">
<h2>What Changed</h2>

<p>
Before: 50,000 positions calculated on CPU → 600KB uploaded per frame → GPU renders
</p>

<p>
Now: 12-16 bytes of parameters uploaded per frame → GPU calculates 50,000 positions → GPU renders
</p>

<p>
<strong>~37,500x less bandwidth.</strong>
</p>

<p>
But more importantly: the conceptual shift.
</p>

<p>
You're not "generating geometry and sending it to the GPU."
You're "defining an algorithm and executing it massively parallel on the GPU."
</p>

<p>
The spiral doesn't exist in RAM. The Lorenz attractor doesn't exist as a buffer of vertices.
They exist as <strong>shader programs</strong>—mathematical functions evaluated in parallel.
</p>

<p>
Traditional creative coding: Data → Transform → Upload → Display
MayaFlux with shaders: Parameters → GPU computes everything → Display
</p>

<p>
This is the endpoint of "data as computational substrate." The data isn't data anymore—it's <strong>pure algorithm</strong>.
</p>
</div>

<div class="card wide">
<h2>The Simplicity</h2>

<p>
Notice what you did:
</p>

<ol>
<li>Wrote GLSL functions (just math)</li>
<li>Declared push constant structure (just a C++ struct)</li>
<li>Bound nodes to offsets (same pattern as before)</li>
<li>Called <code>setup_rendering()</code> with custom shaders</li>
</ol>

<p>
The framework didn't change. You didn't learn a "shader system."
You wrote math in GLSL instead of C++, and MayaFlux routes the data.
</p>

<p>
The pattern is identical to everything before:
</p>

<ul>
<li>Nodes produce numbers</li>
<li>Processors consume them</li>
<li>Tokens determine where things run</li>
</ul>

<p>
Fragment shaders, vertex shaders, compute shaders (later)—all the same pattern.
Just different places to execute the math.
</p>
</div>

<div class="card wide">
<h2>Next: Audio Driving Shaders</h2>

<p>
Right now the parameters are static or simple counters.
Next section: Replace those placeholder nodes with actual sonic processes.
</p>

<p>
Recursive feedback networks driving spiral intensity.
Chaotic attractors controlling kaleidoscope segments.
Stochastic density fields modulating distortion parameters.
</p>

<p>
Same shaders. Same infrastructure. Just replace the nodes.
</p>
</div>

<div class="card">
<h2>Audio Processes Driving Shader Parameters</h2>

<p>
Now we connect actual sonic processes to shader parameters. Not "audio visualization"—shared computational substrate generating both audible and visible structure.
</p>

</div>

---

<div class="card">
<h2>Recursive Temporal Networks → Image Distortion</h2>

<p>
Multi-tap delay networks with cross-feedback create spectral density that shifts over time.
Those same density patterns drive UV distortion parameters.
</p>

**distort_network.frag**

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float density;
    float coherence;
    float spectral_tilt;
} params;

void main() {
    vec2 uv = fragTexCoord;

    // Density controls distortion magnitude
    float wave1 = sin(uv.x * 20.0 * params.coherence) * params.density * 0.1;
    float wave2 = cos(uv.y * 15.0 * params.coherence) * params.density * 0.1;

    // Spectral tilt controls direction bias
    vec2 distortion = vec2(wave1, wave2) * (1.0 + params.spectral_tilt);

    // Chromatic aberration based on coherence
    vec3 color;
    float aberration = (1.0 - params.coherence) * 0.02;
    color.r = texture(texSampler, uv + distortion + vec2(aberration, 0.0)).r;
    color.g = texture(texSampler, uv + distortion).g;
    color.b = texture(texSampler, uv + distortion - vec2(aberration, 0.0)).b;

    outColor = vec4(color, 1.0);
}
```

**C++ Code:**

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Delay Network Distortion", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "distort_network.frag" });

    window->show();

    // Audio-rate processing
    auto noise = vega.Random()[0] | Audio;
    auto sparse_trigger = vega.Logic(LogicOperator::THRESHOLD, 0.97)[0] | Audio;
    sparse_trigger->set_input_node(noise);

    // Multi-tap delay network at audio rate
    auto delay_network = vega.Polynomial(
                             [noise](std::span<double> history) {
                                 double input = noise->get_last_output() * 0.05;

                                 double d1 = history[41];
                                 double d2 = history[89];
                                 double d3 = history[179];
                                 double d4 = history[359];
                                 double d5 = history[521];

                                 return input + d1 * 0.35 + d2 * 0.28 - d3 * 0.21 + d4 * 0.15 - d5 * 0.12;
                             },
                             PolynomialMode::RECURSIVE,
                             600)[0]
        | Audio;

    delay_network->set_initial_conditions({ 0.8, -0.3, 0.5, -0.2, 0.4 });

    // Feature extraction at audio rate
    auto density_extractor = vega.Polynomial(
                                 [](std::span<double> history) {
                                     double sum = 0.0;
                                     for (int i = 0; i < 64 && i < history.size(); i++) {
                                         sum += history[i] * history[i];
                                     }
                                     return std::sqrt(sum / 64.0);
                                 },
                                 PolynomialMode::FEEDFORWARD,
                                 128)[0]
        | Audio;

    delay_network >> density_extractor;

    auto coherence_extractor = vega.Polynomial(
                                   [](std::span<double> history) {
                                       int crossings = 0;
                                       for (int i = 1; i < 32 && i < history.size(); i++) {
                                           if ((history[i - 1] > 0.0) != (history[i] > 0.0)) {
                                               crossings++;
                                           }
                                       }
                                       return 1.0 - (crossings / 32.0);
                                   },
                                   PolynomialMode::FEEDFORWARD,
                                   64)[0]
        | Audio;

    delay_network >> coherence_extractor;

    auto tilt_extractor = vega.Polynomial(
                              [](std::span<double> history) {
                                  double recent = 0.0, distant = 0.0;
                                  for (int i = 0; i < 16; i++)
                                      recent += std::abs(history[i]);
                                  for (int i = 48; i < 64; i++)
                                      distant += std::abs(history[i]);
                                  return (recent - distant) / (recent + distant + 0.001);
                              },
                              PolynomialMode::FEEDFORWARD,
                              128)[0]
        | Audio;

    delay_network >> tilt_extractor;

    // Graphics-rate extraction nodes (only for shader binding)
    auto density_gfx = vega.Polynomial([density_extractor](double x) {
        return density_extractor->get_last_output() * 5.0;
    }) | Graphics;

    auto coherence_gfx = vega.Polynomial([coherence_extractor](double x) {
        return coherence_extractor->get_last_output();
    }) | Graphics;

    auto tilt_gfx = vega.Polynomial([tilt_extractor](double x) {
        return tilt_extractor->get_last_output();
    }) | Graphics;

    // Bind to shader using NodeBindingsProcessor
    struct NetworkParams {
        float density;
        float coherence;
        float spectral_tilt;
    };

    auto shader_config = Buffers::ShaderConfig { "distort_network.frag" };
    shader_config.push_constant_size = sizeof(NetworkParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<NetworkParams>();

    node_bindings->bind_node("density", density_gfx,
        offsetof(NetworkParams, density), sizeof(float));
    node_bindings->bind_node("coherence", coherence_gfx,
        offsetof(NetworkParams, coherence), sizeof(float));
    node_bindings->bind_node("tilt", tilt_gfx,
        offsetof(NetworkParams, spectral_tilt), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
The audio creates a resonant network—impulses decay through multiple delay paths, creating spectral interference.
Three feature extractors measure different aspects: overall energy, temporal coherence, spectral balance.
Those measurements directly modulate distortion magnitude, chromatic aberration, and directional bias.
</p>

<p>
Same numbers. Ears hear decay patterns. Eyes see distortion patterns. No conversion—just routing.
</p>

</div>

---

<div class="card">
<h2>Stochastic Processes → Kaleidoscope Morphology</h2>

<p>
Probability distributions shaped by correlated noise sources create density fields.
Those fields control kaleidoscope segmentation and rotation.
</p>

**kaleidoscope_stochastic.frag**

```cpp
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(binding = 0) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float segments;
    float rotation;
    float mirror_probability;
    float color_shift;
} params;

void main() {
    vec2 uv = fragTexCoord;
    vec2 center = vec2(0.5, 0.5);

    vec2 delta = uv - center;
    float angle = atan(delta.y, delta.x) + params.rotation;
    float dist = length(delta);

    // Segments determined by stochastic density
    float segment_angle = 6.28318 / params.segments;
    float segment_id = floor(angle / segment_angle);
    angle = mod(angle, segment_angle);

    // Mirror based on probability threshold
    if (mod(segment_id, 2.0) < params.mirror_probability) {
        angle = segment_angle - angle;
    }

    vec2 kaleidoscope_uv;
    kaleidoscope_uv.x = 0.5 + dist * cos(angle);
    kaleidoscope_uv.y = 0.5 + dist * sin(angle);

    vec4 sampled = texture(texSampler, kaleidoscope_uv);

    // Color shift based on correlated noise
    sampled.rgb = mix(sampled.rgb, sampled.gbr, params.color_shift);

    outColor = sampled;
}
```

**C++ Code:**

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Stochastic Kaleidoscope", 1920, 1080 });

    auto tex_buffer = vega.read_image("your_image.png") | Graphics;

    tex_buffer->setup_rendering({ .target_window = window,
        .fragment_shader = "kaleidoscope_stochastic.frag" });

    window->show();

    // Audio-rate stochastic processing
    auto noise1 = vega.Random();
    auto noise2 = vega.Random();
    auto noise3 = vega.Random();

    // Density shaping
    auto density1 = vega.Polynomial(
                        [](double x) {
                            return std::pow(std::abs(x), 3.0) * (x > 0 ? 1.0 : -1.0);
                        })[0]
        | Audio;

    noise1 >> density1;

    // Correlation processing
    auto segment_correlator = vega.Polynomial(
                                  [](double x) {
                                      return x * x * 0.5 + std::tanh(x + x) * 0.5;
                                  })[0]
        | Audio;

    segment_correlator->set_input_node(noise2);

    density1 >> segment_correlator;

    // Segment mapping
    auto segment_mapper = vega.Polynomial(
                              [](double x) {
                                  return 3.0 + std::abs(x) * 9.0;
                              })[0]
        | Audio;

    segment_correlator >> segment_mapper;

    // Mirror probability
    auto mirror_prob = vega.Polynomial(
                           [](double x) {
                               return (x + 1.0) * 0.5;
                           })[0]
        | Audio;

    density1 >> mirror_prob;

    // Rotation accumulator
    auto rotation_accumulator = vega.Polynomial(
                                    [noise3](std::span<double> history) {
                                        return history[0] + history[0] * 0.02;
                                    },
                                    PolynomialMode::FEEDFORWARD,
                                    64)[0]
        | Audio;

    rotation_accumulator->set_input_node(noise3);

    rotation_accumulator->set_initial_conditions({ 0.0 });

    // Color shift processing
    auto color_shift = vega.Polynomial(
                           [noise3](double x) {
                               double n3 = noise3->get_last_output();
                               return (x + x + n3) / 3.0;
                           })[0]
        | Audio;

    color_shift->set_input_node(noise2);

    density1 >> color_shift;

    auto color_mapper = vega.Polynomial(
                            [](double x) {
                                return std::abs(x) * 0.5;
                            })[0]
        | Audio;

    color_shift >> color_mapper;

    // Bind to shader
    struct KaleidoscopeParams {
        float segments;
        float rotation;
        float mirror_probability;
        float color_shift;
    };

    auto shader_config = Buffers::ShaderConfig { "kaleidoscope_stochastic.frag" };
    shader_config.push_constant_size = sizeof(KaleidoscopeParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<KaleidoscopeParams>();

    node_bindings->bind_node("segments", segment_mapper,
        offsetof(KaleidoscopeParams, segments), sizeof(float));
    node_bindings->bind_node("rotation", rotation_accumulator,
        offsetof(KaleidoscopeParams, rotation), sizeof(float));
    node_bindings->bind_node("mirror", mirror_prob,
        offsetof(KaleidoscopeParams, mirror_probability), sizeof(float));
    node_bindings->bind_node("color", color_mapper,
        offsetof(KaleidoscopeParams, color_shift), sizeof(float));

    add_processor(node_bindings, tex_buffer, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
Three noise generators feed through probability distribution shapers, multiplicative correlators, and integration.
The result: segment count varies stochastically, rotation drifts randomly, mirror probability fluctuates, color shifts correlate.
</p>

<p>
You hear density fluctuations, correlation patterns, drift. You see kaleidoscope morphology change.
Same stochastic process. Different perceptual channels.
</p>

</div>

---

<div class="card">
<h2>Chaotic Dynamics → Procedural Geometry</h2>

<p>
Ikeda map generates chaotic trajectories. Use that trajectory to control both sonic timbre and geometric deformation.
</p>

**chaos_geometry.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float chaos_x;
    float chaos_y;
    float orbit_scale;
    float distortion;
} params;

void main() {
    float t = float(gl_VertexIndex) * 0.1;

    // Base spiral
    float radius = t * 0.05;
    float angle = t;

    // Modulate by chaotic attractor state
    radius *= (1.0 + params.chaos_x * params.distortion);
    angle += params.chaos_y * params.orbit_scale;

    float x = radius * cos(angle);
    float y = radius * sin(angle);

    gl_Position = vec4(x, y, 0.0, 1.0);

    // Color by attractor position
    float hue = (params.chaos_x + params.chaos_y) * 0.5;
    fragColor = vec3(
        abs(params.chaos_x),
        abs(params.chaos_y),
        abs(hue)
    );

    gl_PointSize = 6.0 + abs(params.chaos_x) * 4.0;
}
```

**C++ Code:**

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Chaotic Geometry", 1920, 1080 });

    auto points = vega.PointCollectionNode(500) | Graphics;
    for (int i = 0; i < 500; i++) {
        points->add_point({ glm::vec3(0.0f), glm::vec3(1.0f), 1.0f });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "chaos_geometry.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    // Ikeda map state variables
    static double x = 0.1, y = 0.1;
    static double u = 0.9;

    // Audio-rate modulation
    // auto u_modulator = vega.Sine(0.05, 0.15);
    auto u_modulator = vega.Random();
    auto u_envelope = vega.Polynomial(
                          [](double x) {
                              return 0.85 + (x + 1.0) * 0.075;
                          })[0]
        | Audio;
    u_envelope->set_input_node(u_modulator);

    // Chaotic trajectory generation
    auto chaos_generator = vega.Polynomial(
                               [](double input) {
                                   double t = 0.4 - 6.0 / (1.0 + x * x + y * y);

                                   double x_new = 1.0 + input * (x * std::cos(t) - y * std::sin(t));
                                   double y_new = input * (x * std::sin(t) + y * std::cos(t));

                                   x = x_new;
                                   y = y_new;

                                   return (x + y) * 0.1;
                               })[0]
        | Audio;

    chaos_generator->set_input_node(u_envelope);

    // Velocity tracking
    auto velocity_tracker = vega.Polynomial(
                                [](std::span<double> history) {
                                    if (history.size() < 4)
                                        return 0.0;

                                    double dx = history[0] - history[1];
                                    double dy = history[2] - history[3];
                                    return std::sqrt(dx * dx + dy * dy);
                                },
                                PolynomialMode::FEEDFORWARD,
                                16)[0]
        | Audio;

    chaos_generator >> velocity_tracker;

    // Graphics-rate extraction
    auto chaos_x_gfx = vega.Polynomial([](double input) {
        return x * 0.5;
    }) | Graphics;

    auto chaos_y_gfx = vega.Polynomial([](double input) {
        return y * 0.5;
    }) | Graphics;

    auto orbit_gfx = vega.Polynomial([velocity_tracker](double input) {
        return velocity_tracker->get_last_output() * 2.0;
    }) | Graphics;

    auto distortion_gfx = vega.Polynomial([velocity_tracker](double input) {
        return 1.0 + velocity_tracker->get_last_output() * 0.5;
    }) | Graphics;

    // Bind to shader
    struct ChaosParams {
        float chaos_x;
        float chaos_y;
        float orbit_scale;
        float distortion;
    };

    auto shader_config = Buffers::ShaderConfig { "chaos_geometry.vert" };
    shader_config.push_constant_size = sizeof(ChaosParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<ChaosParams>();

    node_bindings->bind_node("chaos_x", chaos_x_gfx,
        offsetof(ChaosParams, chaos_x), sizeof(float));
    node_bindings->bind_node("chaos_y", chaos_y_gfx,
        offsetof(ChaosParams, chaos_y), sizeof(float));
    node_bindings->bind_node("orbit", orbit_gfx,
        offsetof(ChaosParams, orbit_scale), sizeof(float));
    node_bindings->bind_node("distortion", distortion_gfx,
        offsetof(ChaosParams, distortion), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
Ikeda map state (x, y) becomes both audio signal and geometric deformation parameters.
Trajectory velocity controls orbit scaling and distortion amount.
The u parameter slowly modulates, causing the attractor to morph between periodic and chaotic regimes.
</p>

<p>
You hear the system transition from stable oscillation to chaos and back.
You see the spiral geometry deform and twist in sync with those transitions.
</p>

</div>

---

<div class="card">
<h2>Granular Synthesis → Particle Density Fields</h2>

<p>
Asynchronous grain clouds with varying density and pitch create textural sound masses.
Same density parameters control particle spawn rates and spatial distribution.
</p>

**grain_particles.vert**

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float grain_density;
    float pitch_dispersion;
    float spatial_spread;
    float time;
} params;

// Simple hash for pseudo-random per-vertex
float hash(float n) {
    return fract(sin(n) * 43758.5453);
}

void main() {
    float id = float(gl_VertexIndex);

    // Random position based on vertex ID and time
    float angle = hash(id * 0.1 + params.time) * 6.28318;
    float dist = hash(id * 0.2 + params.time * 0.5) * params.spatial_spread;

    float x = cos(angle) * dist;
    float y = sin(angle) * dist;

    gl_Position = vec4(x, y, 0.0, 1.0);

    // Color by pitch dispersion
    float hue = hash(id * 0.3) * params.pitch_dispersion;
    fragColor = vec3(
        0.5 + hue * 0.5,
        0.7 - hue * 0.3,
        0.9 - hue * 0.4
    );

    // Size by density
    gl_PointSize = 2.0 + params.grain_density * 6.0;
}
```

**C++ Code:**

```cpp
void compose() {
    auto window = MayaFlux::create_window({ "Granular Field", 1920, 1080 });

    auto points = vega.PointCollectionNode(2000) | Graphics;
    for (int i = 0; i < 2000; i++) {
        points->add_point({ glm::vec3(0.0f), glm::vec3(1.0f), 1.0f });
    }

    auto geom = vega.GeometryBuffer(points) | Graphics;

    geom->setup_rendering({ .target_window = window,
        .vertex_shader = "grain_particles.vert",
        .fragment_shader = "simple_passthrough.frag" });

    window->show();

    // Audio-rate granular processing
    auto noise = vega.Random();

    // Density envelope
    auto density_lfo = vega.Sine(0.1, 1.0);
    auto density_shaper = vega.Polynomial(
                              [](double x) {
                                  return std::pow((x + 1.0) * 0.5, 2.0);
                              })[0]
        | Audio;
    density_shaper->set_input_node(density_lfo);

    // Grain trigger logic
    auto grain_trigger = vega.Logic(
                             [density_shaper](double x) {
                                 double threshold = 1.0 - density_shaper->get_last_output();
                                 return x > threshold;
                             })[0]
        | Audio;
    grain_trigger->set_input_node(noise);

    // Pitch dispersion
    auto pitch_lfo = vega.Sine(0.07, 0.5);
    auto pitch_mapper = vega.Polynomial(
                            [](double x) {
                                return std::abs(x);
                            })[0]
        | Audio;
    pitch_mapper->set_input_node(pitch_lfo);

    // Spatial spread
    auto spread_lfo = vega.Sine(0.13, 1.0);
    auto spread_mapper = vega.Polynomial(
                             [](double x) {
                                 return 0.3 + (x + 1.0) * 0.35;
                             })[0]
        | Audio;
    spread_mapper->set_input_node(spread_lfo);

    // Granular synthesis engine
    auto grain_synthesizer = vega.Polynomial(
                                 [grain_trigger, pitch_mapper, noise, density_shaper](std::span<double> history) {
                                     static int grain_phase = 0;
                                     static double grain_pitch = 1.0;

                                     if (grain_trigger->get_last_output() > 0.5) {
                                         grain_phase = 0;
                                         grain_pitch = 1.0 + (noise->get_last_output() * pitch_mapper->get_last_output());
                                     }

                                     if (grain_phase < 512) {
                                         double envelope = std::sin((grain_phase / 512.0) * 3.14159);
                                         double grain = std::sin(grain_phase * 0.01 * grain_pitch) * envelope;
                                         grain_phase++;

                                         return grain * density_shaper->get_last_output();
                                     }

                                     return 0.0;
                                 },
                                 PolynomialMode::FEEDFORWARD,
                                 1024)[0]
        | Audio;

    // Graphics-rate extraction with time accumulator
    auto time_accum = vega.Phasor(60.0) | Graphics; // 60 Hz for smooth animation

    auto density_gfx = vega.Polynomial([density_shaper](double x) {
        return density_shaper->get_last_output();
    }) | Graphics;

    auto pitch_gfx = vega.Polynomial([pitch_mapper](double x) {
        return pitch_mapper->get_last_output();
    }) | Graphics;

    auto spread_gfx = vega.Polynomial([spread_mapper](double x) {
        return spread_mapper->get_last_output();
    }) | Graphics;

    // Bind to shader
    struct GrainParams {
        float grain_density;
        float pitch_dispersion;
        float spatial_spread;
        float time;
    };

    auto shader_config = Buffers::ShaderConfig { "grain_particles.vert" };
    shader_config.push_constant_size = sizeof(GrainParams);

    auto node_bindings = std::make_shared<Buffers::NodeBindingsProcessor>(shader_config);
    node_bindings->set_push_constant_size<GrainParams>();

    node_bindings->bind_node("density", density_gfx,
        offsetof(GrainParams, grain_density), sizeof(float));
    node_bindings->bind_node("pitch", pitch_gfx,
        offsetof(GrainParams, pitch_dispersion), sizeof(float));
    node_bindings->bind_node("spread", spread_gfx,
        offsetof(GrainParams, spatial_spread), sizeof(float));
    node_bindings->bind_node("time", time_accum,
        offsetof(GrainParams, time), sizeof(float));

    add_processor(node_bindings, geom, Buffers::ProcessingToken::GRAPHICS_BACKEND);
}
```

<p>
Grain density controls both trigger probability (how often grains spawn sonically) and point size (visual density).
Pitch dispersion affects grain transposition range and particle color variation.
Spatial spread modulates stereo width (implied) and geometric scatter.
</p>

<p>
Dense grain clouds → dense particle fields.
Sparse grains → sparse particles.
Wide pitch variation → wide color variation.
The parameters mean the same thing in both domains.
</p>

</div>

---

<div class="card wide">
<h2>What This Reveals</h2>

<p>
These aren't "visualizations of audio." They're <strong>unified processes with multiple outputs</strong>.
</p>

<p>
The delay network doesn't "make sound that affects visuals."
It creates temporal interference patterns. Some energy goes to speakers. Some modulates UV coordinates.
</p>

<p>
The stochastic system doesn't "generate audio that controls graphics."
It generates probability distributions. Some create sonic density. Some create visual segmentation.
</p>

<p>
The chaotic attractor isn't "sonified and also visualized."
It's a dynamical system. One output channel → DAC. Two output channels → shader parameters.
</p>

<p>
This is why MayaFlux doesn't have "audio → visual mapping."
The processes are already unified. You just route outputs.
</p>

</div>

<div class="card wide">
<h2>The Simplicity Remains</h2>

<p>
Compare what you wrote to traditional creative coding:
</p>

**Traditional approach:**

```javascript
// Analyze audio
let fft = new FFT();
let spectrum = fft.analyze(audioBuffer);

// Map to parameters
let segments = map(spectrum[0], 0, 255, 3, 12);
let rotation = map(spectrum[5], 0, 255, 0, TWO_PI);

// Apply to shader
shader.setUniform("segments", segments);
shader.setUniform("rotation", rotation);
```

**MayaFlux:**

```cpp
// Audio process
auto segment_correlator = vega.Polynomial([noise2](double x) {
    return x * noise2->get_last_output() * 0.5;
}) | Audio;

// Get value, send to shader
params.segments = segment_correlator->get_last_output();
render_proc->set_push_constant_data_raw(&params, sizeof(params));
```

<p>
No FFT class. No analysis step. No mapping function. No "send to parameter."
Just: create process, get output, write to struct.
</p>

<p>
The audio processes are sophisticated—recursive networks, stochastic correlators, chaotic dynamics, granular synthesis.
But the connection to visuals is trivial: <code>get_last_output()</code>.
</p>

<p>
The complexity lives in the <strong>algorithmic processes</strong>, not the framework plumbing.
MayaFlux doesn't get in your way.
</p>

</div>

<div class="card wide">
<h2>Where This Leads</h2>

<p>
You've seen nodes produce data, processors transform it, shaders consume it. Audio processes driving visual parameters, stochastic systems controlling both domains simultaneously, chaotic dynamics generating unified structure.
</p>

<p>
But these examples are just <strong>entry points</strong>—bridges from familiar territory into MayaFlux's paradigm. They leverage what you already know about visual processing to demonstrate the framework's approach. The actual creative territory extends far beyond what's shown here.
</p>

</div>

---

<div class="card">
<h2>What You Haven't Seen Yet</h2>

<p>
These tutorials demonstrated:
</p>

<ul>
<li>Image distortion through UV manipulation</li>
<li>Procedural geometry from mathematical functions</li>
<li>Audio processes modulating shader parameters</li>
<li>Cross-domain data flow through shared infrastructure</li>
</ul>

<p>
But MayaFlux's scope includes approaches you haven't encountered:
</p>

<ul>
<li><strong>Compute shaders for parallel physics</strong> — particle systems with 100,000+ agents, modal resonators simulating physical materials, fluid dynamics generating both sonic and visual textures</li>
<li><strong>Feedback loops across domains</strong> — visual analysis driving audio processes driving visual parameters, creating self-modifying systems</li>
<li><strong>Network topologies</strong> — ModalNetwork for physical modeling synthesis, ParticleNetwork for emergent behaviors, custom NodeNetwork structures for domain-specific logic</li>
<li><strong>Temporal coordination beyond metro</strong> — coroutines synchronizing processes at different rates, awaitable buffers for deterministic timing, event-driven architectures</li>
<li><strong>Live coding through Lila</strong> — JIT compilation of C++ with ~1 frame latency, enabling algorithmic improvisation and runtime exploration</li>
<li><strong>Custom processor chains</strong> — building your own BufferProcessor implementations, GPU compute pipelines, specialized transformation strategies</li>
</ul>

<p>
The examples here used familiar building blocks (sine waves, images, simple distortions) to teach the pattern. Real work involves inventing novel processes that have no analogs in traditional tools.
</p>

</div>

---

<div class="card">
<h2>The Pedagogical Approach</h2>

<p>
This tutorial series started with "what you know" and gradually revealed MayaFlux's actual paradigm:
</p>

<p>
<strong>First:</strong> "Here's an image. Move it." (Familiar: display, transform, animate)
</p>

<p>
<strong>Then:</strong> "Here's geometry. Generate it mathematically." (Shift: data from equations, not assets)
</p>

<p>
<strong>Then:</strong> "Here's a shader. Let the GPU compute." (Shift: algorithms, not data transfer)
</p>

<p>
<strong>Finally:</strong> "Here's an audio process. Same numbers, different outputs." (Paradigm: unified computation)
</p>

<p>
Each step moved further from "creative coding as drawing" toward "creative coding as computational substrate design." The visuals became less important than the <strong>processes generating them</strong>.
</p>

<p>
But even the final examples—delay networks, stochastic systems, chaotic attractors—are still <strong>pedagogical bridges</strong>. They're sophisticated enough to demonstrate the paradigm, simple enough to understand quickly. Real creative work involves processes that don't fit tutorial narratives.
</p>

</div>

---

<div class="card">
<h2>Beyond These Examples</h2>

<p>
Consider what becomes possible when you stop thinking in terms of "audio visualization" or "generative graphics":
</p>

<p>
<strong>Recursive modal synthesis</strong> where each resonator's output feeds back through a network of coupled oscillators, and the same coupling coefficients control both spectral evolution and geometric deformation of 3D meshes. Not "sound controlling visuals"—shared dynamical system with multiple perceptual outputs.
</p>

<p>
<strong>Stochastic texture synthesis</strong> where Markov chains operating on spectral features generate both sonic grain clouds and pixel displacement fields. The probability distributions don't "visualize audio"—they define the computational substrate that manifests in both domains.
</p>

<p>
<strong>Cellular automata on compute shaders</strong> processing millions of cells per frame, where rule sets produce both sample-rate audio buffers (through read-back) and realtime visual evolution. The automaton isn't "making sound and graphics"—it's a unified discrete dynamical system with observers in different channels.
</p>

<p>
<strong>Grammar-based structural generation</strong> where L-systems or other formal grammars define both temporal event sequences (MIDI-like note structures) and spatial branching patterns (tree-like geometries). The grammar doesn't "control" anything—it <strong>is</strong> the structure, interpreted through different lenses.
</p>

<p>
These approaches have no equivalents in Processing, openFrameworks, Max/MSP, or Pure Data. They require thinking about creative computation differently—not as "making audio" or "making visuals," but as <strong>designing computational conditions where interesting structure emerges</strong>.
</p>

</div>

---

<div class="card wide">
<h2>What Makes MayaFlux Different</h2>

<p>
Traditional tools enforce boundaries:
</p>

<ul>
<li>Processing/p5.js: Visual canvas, occasional audio as afterthought</li>
<li>SuperCollider/Pure Data: Audio synthesis, occasional visual feedback</li>
<li>Max/MSP: Patching between domains, but domains remain separate</li>
<li>TouchDesigner: Unified interface, but analog-inspired paradigm (cables, operators, channels)</li>
</ul>

<p>
MayaFlux removes these boundaries by refusing to impose them in the first place. There's no "audio subsystem" and "graphics subsystem" that need bridging. There's computational substrate that produces numbers, and those numbers can be routed anywhere.
</p>

<p>
This isn't about "integration" or "interoperability." It's about <strong>never separating in the first place</strong>.
</p>

<p>
When you write:
</p>

```cpp
auto process = vega.Polynomial([](std::span<double> history) {
    // Complex temporal logic
    return computed_value;
}, PolynomialMode::RECURSIVE, 512)[0] | Audio;
```

<p>
That process doesn't "make audio." It produces numbers at sample rate. What you do with those numbers—send to DAC, modulate shader parameters, control network topology, trigger events—is up to you. The process doesn't know. It doesn't care. It just computes.
</p>

<p>
This is what "digital-first" means: data is data, numbers are numbers, computation is computation. The disciplinary boundaries (audio, visual, control) are <strong>routing decisions</strong>, not architectural constraints.
</p>

</div>

---

<div class="card">
<h2>The Path Forward</h2>

<p>
If you came here from visual arts, you now understand:
</p>

<ul>
<li>How to create and transform visual data in MayaFlux</li>
<li>How shaders enable GPU-parallel computation</li>
<li>How audio-rate processes generate parameters</li>
<li>How the same infrastructure handles all data types</li>
</ul>

<p>
Your next steps:
</p>

<ol>
<li><strong>Explore compute shaders</strong> — move beyond vertex/fragment into general GPU computation</li>
<li><strong>Design custom processes</strong> — write Polynomial functions that implement novel algorithms</li>
<li><strong>Build feedback networks</strong> — connect outputs back to inputs for emergent behavior</li>
<li><strong>Study the other tutorials</strong> — musicians and programmers approach from different angles, learn their perspectives</li>
<li><strong>Read the technical documentation</strong> — understand NodeNetwork, BufferProcessor, domain tokens, coroutine coordination</li>
</ol>

<p>
Most importantly: <strong>stop thinking about "making visuals."</strong> Start thinking about <strong>designing computational processes</strong> that happen to have visual manifestations.
</p>

<p>
The examples here showed familiar patterns to ease the transition. Real work involves patterns that don't exist yet—approaches that only become possible when you stop accepting the limitations of analog-inspired frameworks.
</p>

</div>

---

<div class="card wide">
<h2>Remember</h2>

<p>
These tutorials are <strong>bridges, not destinations</strong>. They connect your existing knowledge to MayaFlux's paradigm. They show enough to make the framework comprehensible, not enough to exhaust its possibilities.
</p>

<p>
The spiral examples, kaleidoscope distortions, delay networks, granular fields—these are <strong>pedagogical constructs</strong>. They're simple enough to understand in a tutorial, complex enough to demonstrate the paradigm. They're not templates to copy; they're patterns to transcend.
</p>

<p>
MayaFlux doesn't exist to replicate what's already possible in Processing or TouchDesigner. It exists to enable approaches that <strong>aren't possible</strong> when audio and visuals are treated as separate domains requiring translation.
</p>

<p>
When you find yourself thinking "how do I map audio to visuals in MayaFlux?"—stop. That's the wrong question. The right question is: "what computational process generates the structure I want, and how do I route its outputs?"
</p>

<p>
The numbers don't belong to a domain. <strong>You</strong> decide where they go.
</p>

</div>

---

<div class="card">
<h2>Next: Choose Your Path</h2>

<p>
Explore the other persona tutorials to see different entry points into the same infrastructure:
</p>

<ul>
<li><strong><a href="../musician_starting/">Starting as a Musician</a></strong> — approaching MayaFlux from audio synthesis and digital signal processing</li>
<li><strong><a href="../programmer/">Starting as a Programmer</a></strong> — understanding the architectural patterns, memory models, and C++20 features [Coming Soon]</li>
</ul>

<p>
Or dive into the technical documentation: <strong><a href="https://mayaflux.github.io/MayaFlux/">Complete API</a></strong> 
</p>

<p>
The framework is yours. The constraints you experienced in other tools don't exist here. What you build next is up to you.
</p>

</div>
