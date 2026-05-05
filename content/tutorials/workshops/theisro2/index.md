---
title: "Workshop 2 - Computational Audio-Visual Thinking"
---

## Quick info
- **Online**
- **Organized with the Indian Sonic Research Organization**
- **May 8-10 2026 - Core session (4pm-7pm IST) (12:30-3:30 CET). Optional 1 hour prior and after. See schedule below.**
- **Sign up on the discord server to reserve your spot and get updates: [Discord](https://discord.gg/GrKtdHB68)**
- [Download](/download): Required before the session. (5-30 minutes based on OS, hardware, internet connection)

---

## Overview

Three days of building, unravelling, and breaking things in MayaFlux, a framework where audio, visuals, and any other data stream share the same numerical substrate. Not mapped to each other. Not synchronized. The same numbers, interpreted differently depending on where you point them.

Day 1 we build one system together, live, in front of everyone. It starts as a particle field in 3D space and ends as something where the camera position influences what you hear, the light influences the synthesis, and the whole thing runs itself. Each step survives into the next. Nothing gets replaced. You watch one piece become itself over the course of the day.

Day 2 we start with a complete, running system and take it apart. The system does something that no other tool in this space can express: audio picking spatial regions from a live image of itself, the result mixed with the sound rendered as its own texture, two windows with causality running in both directions simultaneously. We pull threads until we understand what it is.

Day 3 is yours. You pick a direction, work toward something you can show, and I am there to unblock you, not steer you. It ends with everyone showing whatever state they reached.

## Cost (None)

MayaFlux, the knowledge around it, and the pedagogy it emerges from are intended to expand what people can do with computation across sound, visuals, and beyond. This is about opening up imagination, expression, and computational scope, not restricting access to them.

As long as it is sustainable for me, this will remain free.

---

## C++ and programming experience (not required)

This needs to be said without softening it.

**C++ knowledge is not required. It will not help you more than curiosity will.** It might actually slow you down if it makes you want to understand every line before you run anything.

This workshop will not teach you C++. There will be no detours into language syntax, no explanations of templates or memory management, no standalone programming exercises. If you come wanting that, this is the wrong three days.

What the workshop runs on is this: you see a system doing something, you change a number, you observe what shifts. You break it, you find out what breaks. You swap a component, you hear or see the difference. Understanding accumulates through that cycle, not through explanation. The code is how we talk to the machine. It is not the point.

Someone who has never written a line of code in their life but is genuinely curious about what happens when a light source feeds back into a synthesis parameter will get more out of this than someone with ten years of C++ experience who needs to understand the type system before touching anything.

The only things that matter here:

- Willingness to run something before understanding it
- Comfort with not knowing what something does until you break it
- Curiosity about what computers can actually do when you stop asking them to imitate older tools

If that is you, the language on the screen is a detail.

---

## Structure

**Day 1: One system, assembled live**

We start from a working base and add to it. A particle field in 3D space with a moving camera. Synthesis that reacts to audio state. A light source. The light feeding back into the sound. Complexity growing in both domains while the architecture stays the same. Manual interaction replaced by declared relationships. The camera becoming an agent that influences what computes.

Each addition is motivated by where the system already is, not by a syllabus. By the end of the day you have watched one piece grow from a point in space into something with its own behaviour.

**Day 2: One system, taken apart**

We start with a complete system already running. Two windows. Audio distorting a video source in one. Audio picking spatial regions from that distorted image in the other, mixed with the audio waveform rendered as texture directly. Causality runs in both directions.

We pull threads. We make modifications as understanding develops. The mechanics can shift based on what the group is engaging with.

**Day 3: Your direction**

No guided content. You choose what to work on. A few prepared starting points are available if you want a constraint to push against. The session ends with everyone showing whatever state they reached. Surprises over successes.

---

## Practical

Please [downlood](/download) Weave for your OS. It is the central installer, dependency manager, and project creator for MayaFlux.

Hardware is not an issue. MayaFlux ran inside a container on an immutable Steam Deck at the performance you saw. If that worked, your machine works.

OS is a different matter. Supported:

- Windows 10 or 11
- macOS 15.6 or higher (Intel or Apple Silicon - Intel is technically supported but not personally tested for the graphics pipeline)
- Linux: - Fedora 43 or higher,
         - Ubuntu 25.10 derivatives or higher,
         - any rolling release distro with LLVM 21 available/

Headphones required. A camera is useful for Day 2 but not required, a video file works as a substitute.

MayaFlux installed before the session (installation guide sent in advance, 5-30 minutes depending on machine and connection).

Installation and setup questions: mayafluxcollective@proton.me

More at mayaflux.org

## Location:

[Discord](https://discord.gg/GrKtdHB68)
The meeting room will be announced in the server channel calendar on the day of the event.

## Schedule

* **Core session (all 3 days):**
  4:00–7:00 PM IST / 12:30–3:30 PM CET

* **Pre-session (optional):**
  3:00–4:00 PM IST / 11:30 AM–12:30 PM CET
  Installation help, fixing prior issues (not for first-time setup)

* **Post-session (optional):**
  7:00–8:00 PM IST / 3:30–4:30 PM CET
  Q&A, deeper dives, extra examples, discussion

## In the meantime:

Please feel free to explore the various documentation on this website
- [0.3 Release showcase](/releases/news/the_shape_of_material/)
- [Getting started as a musician (basic)](/personas/musician_starting)
- [Getting started as a visual artist (basic)](/personas/visual_artist_starting)
- [Conceptual crossover from PureData](/onboarding/pure_data)
- [Conceptual crossover from Max](/onboarding/max_msp)
- [Full scope tutorials (Three complete, 4th in progress)](/tutorials/sculpting-data/)
