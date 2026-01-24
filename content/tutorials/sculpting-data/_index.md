---
title: "Sculpting Data: Overview"
slug: "sculpting-data"
layout: "tutorial"
weight: 10
description: "How to navigate the Sculpting Data series: the philosophy, structure, and tutorial order."
hideChildren: true

cascade:
  build:
    list: never
    publishResources: false
---

Sculpting Data is a guided sequence that teaches MayaFlux through one principle:

> **Data is material. You shape it.**

It is meant to be read progressively, but each part stands alone.

This page explains **how to use the series** and **what to expect in each tutorial**.

## Before You Begin

<div class="card wide">
<p>You don’t need detailed C++ knowledge; only curiosity and willingness to experiment.
It is recommended to read the cards below before starting the tutorials, as it explains the structure and flow of the series.
</p>

#### Quick link to Part I

👉 <strong><a href="/tutorials/sculpting-data/foundations/">Part I — Foundations of Form</a></strong></p>

</div>

{{< rawhtml >}}

<!-- =======================  TERMINOLOGY NOTE  ======================= -->

<div class="card collapsible">

  <div class="collapsible-header">
    <h3><b>!IMPORTANT:</b> Terminology Note Across Versions</h3>
    <p class="hint">Click to expand (read once)</p>
  </div>

    <p><strong>What changed?</strong></p>

    <ul>
        <li>(OLD)<code>ContainerBuffer</code> → <code>SoundContainerBuffer</code></li>
        <li>(OLD)<code>ContainerToBufferAdapter</code> → <code>SoundStreamReader</code></li>
        <li>(OLD)<code>StreamWriteProcessor</code> → <code>SoundStreamWriter</code></li>
        <li>(OLD)<code>FileBridgeBuffer</code> → <code>SoundFileBridge</code></li>
    </ul>

  <div class="collapsible-body">

    <p>
    MayaFlux is currently in active architectural consolidation.
    As part of this, several internal classes were renamed to make
    <strong>domain boundaries explicit</strong> before graphics and
    multi-domain I/O support arrive.
    </p>

    <p>
    You may notice names in this tutorial that differ from those
    in MayaFlux <strong>0.1.x</strong>.
    This is intentional.
    </p>

    <hr>

    <p>
    These renames do <strong>not</strong> introduce new behavior.
    They clarify that these components are <em>audio-domain infrastructure</em>,
    and prevent accidental misuse as graphics and video containers are added.
    </p>

    <p>
    MayaFlux is currently in active architectural consolidation.
    As part of this, several internal classes were renamed to make
    <strong>domain boundaries explicit</strong> before graphics and
    multi-domain I/O support arrive.
    </p>


    <hr>

    <p><strong>How to read this tutorial</strong></p>

    <ul>
      <li>
        If you are on <strong>MayaFlux 0.1.x</strong>,
        mentally substitute the older names.
      </li>
      <li>
        If you are on <strong>0.2+</strong>,
        the names here match the codebase directly.
      </li>
      <li>
        In all cases, the <strong>architecture, data flow, and concepts are identical</strong>.
      </li>
    </ul>

    <p>
    These tutorials describe <em>roles and relationships</em>,
    not version-specific symbols.
    </p>

  </div>

</div>

{{< /rawhtml >}}

---

## How to Use These Tutorials

<div class="card-grid">

<div class="card vertical">
<h3>Flow First</h3>
<p>Move linearly through each tutorial. Sections are designed to be short and runnable.</p>
</div>

<div class="card vertical">
<h3>Return When Ready</h3>
<p>Advanced readers can thumb through the explanations.  
You can skip them and return at any time without losing the thread.</p>
</div>

<div class="card vertical">
<h3>Optional Depth</h3>
<p>Open collapsed API/Design panels only when you want to dive deeper.</p>
</div>

</div>

---

## Structure of Every Section

<div class="card-grid breakout">

<div class="card vertical">
<h3>1. Section Heading</h3>
<p>Each section begins with <code>Tutorial: &lt;Concept&gt;</code>.  
This signals that a runnable example follows.</p>
</div>

<div class="card vertical">
<h3>2. Runnable Example</h3>
<p>The spine of the section. Short, minimal, and executable immediately.</p>
</div>

<div class="card vertical">
<h3>3. Optional Panels</h3>
<p>API notes, design notes, and advanced variations — all collapsed by default.</p>
</div>

<div class="card vertical">
<h3>4. Try It → Recap</h3>
<p>A quick checkpoint: summary, modified snippet, optional variant.</p>
</div>
</div>

<div class="card vertical wide">
<h3>5. Flow Controls</h3>
<p>Every part ends with navigation to the next part, the overview, and the full tutorials index.</p>
</div>

---

## Recommended Reading Flow

<div class="card-grid">

<div class="card vertical">
<ol>
<li>Read the section heading</li>
<li>Run the example</li>
<li>Skip to next section named "Tutorial", <b>OR</b></li>
<li>Open deep-dive only if needed</li>
<li>Use Try It → Recap</li>
<li>Move on</li>
</ol>
</div>

<div class="card vertical">
<p>You can complete the series linearly in 1–2 hours,<br>
or spend days exploring optional subsections.</p>
</div>

</div>

---

## Sculpting Data Tutorials

<div class="card-grid">

<div class="card wide">
<h3><a href="/tutorials/sculpting-data/foundations/">Part I — Foundations of Form</a></h3>
<p>Containers → Buffers → Form.</p>
</div>

<div class="card wide">
<h3><a href="/tutorials/sculpting-data/processing_expression">Part II — Processing Expression </a></h3>
<p>Processors, operations, shaping behaviour.</p>
</div>

<div class="card wide">
<h3><a href="/tutorials/sculpting-data/visual_materiality_i">Part III — Visual Materiality I </a></h3>
<p>Introduction to declarative graphics</p>
</div>

<div class="card wide">
<h3>Part IV — Contextual Intelligence <em>(planned)</em></h3>
<p>Conditional adaptation and dynamic pipelines.</p>
</div>

</div>

---
