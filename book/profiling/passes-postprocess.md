---
title: "Post processing pass"
excerpt: ""
permalink: "/book/profiling/passes-postprocess/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

# PostProcessing

**Responsible for:**

* Depth of Field (__BokehDOFRecombine__)
* Temporal anti-aliasing (__TemporalAA__)
* Reading velocity values (__VelocityFlatten__)
* Motion blur (__MotionBlur__)
* Auto exposure (__PostProcessEyeAdaptation__)
* Tone mapping (__Tonemapper__)
* Upscaling from rendering resolution to display's resolution (__PostProcessUpscale__)

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Number and quality of post processing features
* {{ icon_overdraw }} Number and complexity of __blendables__ (post process materials)

Description TODO. The final pass in the rendering pipeline.
