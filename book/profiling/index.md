---
title: "Introduction to profiling"
excerpt: ""
permalink: "/book/profiling/"
---

{% include toc icon="columns" title=page.title %}

Since at least two generation of consoles, real-time graphics performance is no longer just a matter of pure polygon count. Features that are fundamental to physically based rendering - from specular reflections, through diffuse lighting to ambient occlusion - are largely decoupled from the cost of geometry, being dependent on the amount of pixels instead. Other issues - like quad overshading or texture bandwidth - need more attention now, as graphics chips became great at processing polygons, causing the bottlenecks to appear elsewhere.

To understand profilers' output, we need the knowledge about engine's inner workings.
