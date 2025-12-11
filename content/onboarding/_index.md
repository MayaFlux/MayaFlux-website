---
title: "Rosetta Stone: Unlearning to Relearn:"

cascade:
  build:
    list: never
    publishResources: false
---

<div class="card wide">
<p>
You already know how to think creatively in computational space.
</p>

<p>
Not because you learned it somewhere else, but because the tools you've used trained you to think in specific patterns: patterns that made sense for those tools' architectures, their historical constraints, their commercial compromises.
</p>

<p>
This isn't about "migration guides" or "feature comparison tables." Those documents assume the goal is to recreate what you already do, just in a different environment. That's not the goal.
</p>

<p>
<strong>The goal is to recognize which patterns you learned are computational truths, and which are artifacts of tools that were built to solve yesterday's problems.</strong>
</p>
</div>

<div class="card">
<h2>These Aren't Translations</h2>

<p>
When you learn a human language, you don't just memorize word for word translations. You learn to <em>think</em> in that language: its idioms, its grammar, its ways of structuring ideas that don't map 1:1 to your native tongue.
</p>

<p>
The documents that follow work the same way. They don't tell you "in Pure Data you use [metro], in MayaFlux you use Kriya::metro." That's missing the point.
</p>

<p>
They tell you: "Pure Data taught you that time is discrete ticks at control rate. MayaFlux asks: what if time is just another processing precision you choose, like sample rate or frame rate or whatever computational resolution your idea requires?"
</p>

<p><strong>The shift isn't about syntax. It's about possibility space.</strong></p>
</div>

<div class="card">
<h2>What You're Actually Unlearning</h2>

<p>Every tool you've used made architectural decisions that became invisible to you:</p>

<p>
<strong>Pure Data:</strong> Separated audio rate from control rate because that's how analog hardware worked. Digital computation has no such boundary.
</p>

<p>
<strong>SuperCollider:</strong> Separated the language from the server because network audio was the goal. Your code doesn't need that boundary.
</p>

<p>
<strong>p5.js:</strong> Made you clear and redraw every frame because that's how display hardware worked in 1985. Modern GPUs process data pipelines, not draw calls.
</p>

<p>
<strong>openFrameworks:</strong> Built on OpenGL because that was the graphics API available. OpenGL is deprecated. Your creative ideas aren't.
</p>

<p>
<strong>Max/MSP:</strong> Made everything visual because WYSIWYG was revolutionary in 1988. Text is actually more precise for complex logic.
</p>

<p>
None of these were <em>wrong</em> choices. They were brilliant solutions to the constraints of their time.
</p>

<p><strong>But you don't have those constraints anymore.</strong></p>

<p>
MayaFlux doesn't exist because those tools are "bad." It exists because computational substrate has evolved, and new paradigms become possible when you're not maintaining backward compatibility with architectures from decades ago.
</p>
</div>

<div class="card">
<h2>This Is About Mental Models, Not Features</h2>

<p>
You don't need to "learn MayaFlux" before reading these. You need to be willing to question assumptions you didn't know you had.
</p>

<p>
When Pure Data made you think "audio and control are different domains," that became invisible. It's just "how things work." These documents make those invisible assumptions visible, then ask: "is that actually true, or just true for Pure Data?"
</p>

<p>
When SuperCollider made you write SynthDefs that got sent to a server, that separation became natural. "Of course synthesis happens on the server, control happens in the language." These documents ask: "what if everything just... runs where you write it?"
</p>

<p>
When p5.js made you write a draw() loop, that became "how graphics work." These documents ask: "what if you never draw anything, and instead define how data should flow to the GPU?"
</p>

<p><strong>The "features" are just consequences of different mental models.</strong></p>

<p>
Once you see <em>why</em> your current tools work the way they do, and recognize that those reasons were historical accidents, not computational laws, the MayaFlux API becomes obvious. You're not memorizing a new syntax. You're thinking in a paradigm where those old boundaries don't exist.
</p>
</div>

<div class="card">
<h2>How to Read These</h2>

<p>
Pick the tool that shaped your thinking the most. Not the one you're "most expert" in. The one whose patterns feel most <em>natural</em> to you.
</p>

<p>
That's the tool whose assumptions are most deeply embedded in how you think about creative computing.
</p>

<p>
Read that document. Not to memorize mappings, but to watch your own assumptions surface and dissolve.
</p>

<p>
You'll recognize patterns: "oh, I always thought X was necessary because..." And then you'll see: "...but that was just Y's architecture, not a computational requirement."
</p>

<p><strong>That moment: that's what these documents are for.</strong></p>

<p>
Once you've had it, the MayaFlux API isn't something you "learn." It's something you recognize as the obvious way to express ideas you already had but couldn't fully articulate in tools that weren't built for them.
</p>
</div>

<div class="card">
<h2>You Already Know How to Do This</h2>

<p>
Every creative practitioner does this constantly.
</p>

<p>
A painter trained in oil learns watercolor: not by finding "oil equivalents" in water based media, but by recognizing which techniques were paint specific and which were about light, form, composition.
</p>

<p>
A jazz musician learning classical doesn't just "translate" their improvisational vocabulary. They recognize which patterns were jazz specific (call and response, swing feel) and which were music universal (harmonic progression, melodic contour).
</p>

<p>
<strong>You already know how to distinguish tool specific patterns from domain truths.</strong>
</p>

<p>
These documents just make that process explicit for creative computation.
</p>
</div>

# Choose Your Starting Tool

<div class="card-grid">

<div class="card vertical">
<h3><a href="pure_data/">From Pure Data</a></h3>
<p>
<strong>What you learned:</strong> Message passing, control rate vs audio rate, event driven patching
</p>
<p>
<strong>What changes:</strong> Data flows continuously; transformations happen at every sample; "rates" are just processing precisions you choose
</p>
<p><em>"Metro isn't a metronome. It's time evaluated at whatever precision your idea requires."</em></p>
</div>

<div class="card vertical">
<h3><a href="from_supercollider/">From SuperCollider</a></h3>
<p>
<strong>What you learned:</strong> SynthDefs on the server, Patterns in the language, OSC boundaries between thought and sound
</p>
<p>
<strong>What changes:</strong> No server. Everything you write runs right where you write it. Complete access to every computational step.
</p>
<p><em>"Nodes aren't UGens. They're transformations you can reach inside and modify."</em></p>
</div>

<div class="card vertical">
<h3><a href="from_p5js/">From p5.js</a></h3>
<p>
<strong>What you learned:</strong> setup() once, draw() every frame, imperative rendering loops
</p>
<p>
<strong>What changes:</strong> No draw calls. Define how geometry gets generated; GPU processes it. Data pipelines, not rendering commands.
</p>
<p><em>"You don't draw particles. You define how particle data flows to the GPU."</em></p>
</div>

<div class="card vertical">
<h3><a href="from_openframeworks/">From openFrameworks</a></h3>
<p>
<strong>What you learned:</strong> Callback lifecycle (setup/update/draw), OpenGL immediate mode, audio as separate addon
</p>
<p>
<strong>What changes:</strong> Unified data flow architecture. Audio and visuals use identical infrastructure. Vulkan, not deprecated OpenGL.
</p>
<p><em>"Audio nodes can control visuals. Video can modulate synthesis. Because it's all just numbers."</em></p>
</div>

<div class="card vertical">
<h3><a href="from_max_msp/">From Max/MSP</a></h3>
<p>
<strong>What you learned:</strong> Visual patching, message dispatch, object libraries
</p>
<p>
<strong>What changes:</strong> Explicit pipelines you compose with code. Git friendly text instead of opaque binaries. Cross domain coordination that Max makes painful.
</p>
<p><em>"Same compositional thinking, but your patches are version controlled code."</em></p>
</div>

<div class="card vertical">
<h3><a href="from_processing/">From Processing</a></h3>
<p>
<strong>What you learned:</strong> Creative coding as educational scaffolding, simplified APIs, "code as sketch"
</p>
<p>
<strong>What changes:</strong> Full computational power without training wheels. Same creative immediacy, but nothing's hidden "for your own good."
</p>
<p><em>"Processing taught you to think computationally. MayaFlux removes the guardrails."</em></p>
</div>

</div>

<div class="card wide">
<h2>What Happens After</h2>

<p>
You'll read one of these. Something will click differently than you expected.
</p>

<p>
Not "oh, I know how to translate my workflow now." More like "wait, I've been accepting limitations that weren't actually necessary."
</p>

<p>
Then you'll have a moment where you think about something you've always wanted to build but couldn't quite articulate in your current tools. And you'll realize: "oh. MayaFlux could actually do that."
</p>

<p>
<strong>That's when you start building.</strong>
</p>

<p>
Not because you "learned MayaFlux." Because you recognized that your creative ideas were never tool specific: you just needed substrate that didn't constrain them to yesterday's architectures.
</p>
</div>

<div class="card wide">
<h2>These Documents Are Incomplete</h2>

<p>
Deliberately.
</p>

<p>
They don't cover every feature. They don't explain every API call. They don't give you a comprehensive migration path.
</p>

<p>
Because that's not the point. The point is the mental shift. Once you have that, the API documentation makes sense. The examples become obvious. The patterns emerge naturally.
</p>

<p>
<strong>Start with the paradigm shift. The implementation details come after.</strong>
</p>

<p>
Welcome. Pick your starting tool. Be willing to question everything you thought was "just how things work."
</p>

<p>The substrate will meet you there.</p>
</div>
