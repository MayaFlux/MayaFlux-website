---
title: "Temporal Plurality"
layout: "tutorial"
---

{{< framing-card >}}

<h2>Time as Material</h2>
<p>
Note: This tutorial is a work in progress. More cards, examples and demos will be updated soon.

The first three tutorials in this series taught you what data is, how it transforms, and how it becomes geometry. Every example ran continuously: nodes processed samples, buffers cycled, the engine drove everything at its own rate. You declared structure and the system ran it.

<br>
<br>
This tutorial takes that question back. Not by overriding the engine, but by choosing precisely when your code runs, for how long, under what conditions, and in response to what. The mechanisms range from a single callback attached to a node that was already computing, to a coroutine you write from scratch that suspends and resumes at sample-accurate positions in time. Between those two ends is everything you need to make computation temporal rather than just continuous.

<br>
<br>
The eight cards that follow move from the simplest attachment (a callback on an existing node's tick) through increasingly direct control, ending at the raw coroutine infrastructure that everything else is built from. By the end, none of the earlier cards will be opaque: you will know what runs underneath `schedule_metro`, `EventChain`, `line`, `Trigger`, and `Fabric`, because you will have written equivalents yourself.
</p>

{{< /framing-card >}}

<hr>

<img src="/t4_c1.gif" alt="Timed Points" width="960">

<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="1 of 8" title="Card 1: Declare Once vs. Control" page="/tutorials/sculpting-data/temporal_plurality/01-difference-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>

<hr>

<img src="/t4_c2.gif" alt="Timed Lines" width="960">
<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="2 of 8" title="Card 2: Control as Side Effect" page="/tutorials/sculpting-data/temporal_plurality/02-side-effect-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>


<hr>

<img src="/t4_c3.gif" alt="Timed Particles" width="960">

<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="3 of 8" title="Card 3: Tiered Control" page="/tutorials/sculpting-data/temporal_plurality/03-tiered-control-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>

<hr>

<div id="audio-interface">
    <audio id="player" controls src="/t4_c4.wav" loop></audio>
    <script>
        const player = document.getElementById('player');
        player.volume = 0.2;
        console.log("Audio interface loaded");
    </script>
</div>

<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="4 of 8" title="Card 4: ONLYWHENs" page="/tutorials/sculpting-data/temporal_plurality/04-onlywhen-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>

<hr>

<img src="/t4_c5_1.gif" alt="External Face" width="960">
<img src="/t4_c5_2.gif" alt="External Sun" width="960">

<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="5 of 8" title="Card 5: External Time" page="/tutorials/sculpting-data/temporal_plurality/05-external-time-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>

<hr>


<div id="video-interface">
    <video id="player" controls src="/t4_c6.mp4" loop></video>
    <script>
        const player = document.getElementById('player');
        console.log("Video interface loaded");
    </script>
</div>

<section class="tutorial-section">
  <div class="tutorial-row">
    
{{< tutorial-card step="6 of 8" title="Card 6: Creative Variations" page="/tutorials/sculpting-data/temporal_plurality/06-creative-variations-detail" open="true" >}}
{{< /tutorial-card >}}


    
  </div>
</section>

<hr>

<section class="tutorial-section">
  <header class="tutorial-group">
  </header>
  <div class="tutorial-row">
{{< tutorial-card step="7 of 8" title="Card 7: Declarative Compositions" page="/tutorials/sculpting-data/temporal_plurality/07-declarative-compositions" open="true" >}}
{{< /tutorial-card >}}
  </div>
  <div class="tutorial-row">
{{< tutorial-card step="8 of 8" title="Card 8: Raw Routines" page="/tutorials/sculpting-data/temporal_plurality/08-raw-routines" open="true" >}}
{{< /tutorial-card >}}
  </div>
</section>

{{< framing-card >}}

<h2>Quick Reference</h2>
<p>
  <em
    >Pick the highest-level mechanism whose shape fits. Drop a layer only when
    its shape is in your way.<br />
    The time-based entries below are routines with a loop around them. The node
    hooks are callbacks inside a node's own processing.</em
  >
</p>

<h3>Time on the sample clock</h3>
<ul>
  <li>
    When you need code to run at a fixed interval: <code>schedule_metro</code>.
    When the interval has to be computed at each step, or the body needs more
    than one wait: a <code>SoundRoutine</code> with <code>SampleDelay</code>.
  </li>
  <li>
    When you need code to run once after a delay: <code>Kriya::Timer</code>. It
    holds one pending callback, and scheduling again replaces it.
  </li>
  <li>
    When you need a bracketed start and end over a duration:
    <code>Kriya::TimedAction</code>. The start function runs at once and the end
    function runs after the duration.
  </li>
  <li>
    When you need a node to exist for a bounded duration:
    <code>node &gt;&gt; Time(N) | Audio</code>.
  </li>
  <li>
    When you need a finite ordered sequence of timed events:
    <code>schedule_sequence</code> or <code>EventChain</code>.
  </li>
  <li>
    When you need an indefinitely repeating generative sequence:
    <code>schedule_pattern</code>.
  </li>
  <li>
    When you need a continuously drifting scalar readable by any code:
    <code>schedule_task</code> with <code>create_line</code>, read via
    <code>get_line_value</code>.
  </li>
</ul>

<h3>Signals inside nodes</h3>
<ul>
  <li>
    When you need code to run on every sample from a specific node:
    <code>node-&gt;on_tick</code>, or <code>on_tick_if</code> to add a condition.
    The callback runs inside the node's per-sample processing, so anything slow
    in it is paid on every sample. Use sparingly.
  </li>
  <li>
    When you need code to run only at a specific moment in a node's cycle:
    <code>impulse-&gt;on_impulse</code>, <code>phasor-&gt;on_phase_wrap</code>,
    <code>phasor-&gt;on_threshold</code>, <code>counter-&gt;on_count</code>.
  </li>
  <li>
    When you need code to run while a signal condition is true, continuously:
    Logic <code>while_true</code>.
  </li>
  <li>
    When you need code to run exactly once at a state transition: Logic
    <code>on_change_to</code>.
  </li>
</ul>

<h3>External input</h3>
<ul>
  <li>
    When you need to react to keyboard or mouse:
    <code>on_key_pressed</code>, <code>on_mouse_move</code>,
    <code>on_mouse_pressed</code>. These run on the event thread, not the audio
    thread, so share values accordingly (see below).
  </li>
  <li>
    When you need to react to MIDI, OSC, or HID: <code>vega.read_midi</code> /
    <code>vega.read_osc</code> / <code>vega.read_hid</code>, then hook on the
    node.
  </li>
</ul>

<h3>Buffers and space</h3>
<ul>
  <li>
    When you need audio processing to accumulate across multiple buffer cycles
    before acting: <code>BufferPipeline</code> with <code>PHASED</code>.
  </li>
  <li>
    When you need audio processing to apply per cycle with minimal latency:
    <code>BufferPipeline</code> with <code>STREAMING</code>.
  </li>
  <li>
    When you need a spatial entity that fires on an interval, a key, or a
    choreographed path: <code>Fabric</code> with <code>Wiring</code>.
  </li>
</ul>

<h3>Raw routines</h3>
<ul>
  <li>
    When you need complete control of the coroutine and its state, several waits
    in one body, or delays computed at the moment they are reached: a
    <code>SoundRoutine</code>.
  </li>
  <li>
    When you need work at frame rate, or a routine that writes data the audio
    thread reads: a <code>GraphicsRoutine</code> with <code>FrameDelay</code>.
  </li>
  <li>
    When you need one script that spans both clocks: a
    <code>CrossRoutine</code> with <code>MultiRateDelay</code>. Wait on frames
    only to run on the graphics thread, on samples only to run on the audio
    thread, on both to hold until the slower clock arrives. Use the
    <code>align</code> helper from Card 8c whenever you change axes.
  </li>
  <li>
    When you need computation whose pace is a condition and not a clock: a
    <code>FreeRoutine</code> with <code>ConditionAwaiter</code>. It costs one
    spinning thread, so keep the predicate cheap.
  </li>
  <li>
    When you need to register one: <code>schedule_task</code> takes a Sound,
    Graphics or Free routine. A Cross routine goes to the scheduler's
    <code>add_task</code> for now.
  </li>
</ul>

<h3>Sharing values between mechanisms</h3>
<p>
  Decide by thread, not by domain. A key handler and an audio routine are on
  different threads even though both are control.
</p>
<ul>
  <li>
    When only one routine touches the value: a local variable in the coroutine.
  </li>
  <li>
    When outside code needs to read or nudge one routine's state: promise state
    through the task handle. Write through existing keys only, and create every
    key inside the coroutine before its first wait.
  </li>
  <li>
    When two mechanisms on different threads (audio, graphics, event, free) need
    a value: <code>std::atomic&lt;T&gt;</code> for one number, a
    <code>shared_ptr</code>-owned struct of atomics for several, published with
    a generation counter. For sample data, a container that owns its own
    synchronisation, such as <code>DynamicSoundStream</code>.
  </li>
  <li>When the audio thread is one of the two: no mutex, ever.</li>
</ul>

<hr />

<h2>What This Tutorial Does Not Cover</h2>
<p>
  Three areas sit next to everything here and each needs its own document. Each
  is another answer to the question this tutorial has asked from the start: what
  does this code wait for?
</p>

<p>
  <strong>Lila and live temporal control.</strong> Lila is a JIT environment
  where you can rewrite a coroutine, replace a pattern function, or swap an
  Emitter's influence function while the piece is running. It is not a different
  scheduling mechanism. It is the same mechanisms with the
  <code>compose()</code> boundary removed. The constraint this tutorial assumed,
  that temporal structure is declared at startup and then runs, no longer holds.
  A running coroutine is not edited in place. You cancel it and schedule its
  replacement under the same name, and the new frame starts with fresh state,
  promise map included. Three rules from Card 8 stop being advice there. A
  routine takes its inputs as parameters, because the snippet that created it
  returns before the routine's next resume. A cross routine added mid-session
  aligns its clocks first. A free routine whose first predicate is already true
  runs on the thread that evaluated the snippet.
</p>

<p>
  <strong>Yantra and completion time.</strong> <code>ComputeMatrix</code> and the
  granular workflow run computation on background threads and deliver results
  through callbacks. The callback is a temporal event of its own: it fires when
  the computation finishes. That is a third axis beside the sample clock and the
  frame clock, and unlike them it has no rate. Completion time is
  nondeterministic, and nothing bounds it unless you add a bound, so a design
  that installs a result decides in advance what happens when the result is late.
  Installing it into a live pipeline uses what you have already met. Publish it
  through a generation counter, as Card 8d does, or cancel a routine and schedule
  a new one that carries the result. <code>restart_task</code> alone will not do
  it, because it does not rewind the coroutine. A <code>FreeRoutine</code> is the
  in-tutorial relative: a computation with no clock whose result appears when it
  appears.
</p>

<p>
  <strong>Nexus and spatial triggers.</strong> A <code>Sensor</code>'s perception
  function fires on what lies within its query radius at <code>commit()</code>
  time. The trigger is spatial, not temporal. It combines with everything here: a
  Sensor that fires only inside a Logic gate's open window, a pattern that emits
  new Emitter positions, a coroutine that reads Sensor output and drives a
  Physics operator. The spatial index itself belongs to the Nexus series. The
  scheduling underneath is what you have used throughout: Fabric's
  <code>.use(...)</code> runs the same routines you wrote in Card 8, and
  registers and cancels them with the entity.
</p>

<p>
  Every mechanism in this tutorial waits on something: a sample count, a frame
  count, both, a condition. The three areas above add three more ways for a
  routine to be told to go: someone rewrites it, a computation finishes, an
  entity arrives. The routines themselves do not change.
</p>

{{< /framing-card >}}
