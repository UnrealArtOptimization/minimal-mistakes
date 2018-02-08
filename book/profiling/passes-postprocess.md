---
title: "Post processing pass"
excerpt: ""
permalink: "/book/profiling/passes-postprocess/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

[← Back to all passes]({{ site.baseurl }}{% link book/profiling/passes.md %}){: .btn .btn--prev}

# PostProcessing

<div class="notice" markdown="1">
__Responsible for:__

* Depth of Field (__BokehDOFRecombine__)
* Temporal anti-aliasing (__TemporalAA__)
* Reading velocity values (__VelocityFlatten__)
* Motion blur (__MotionBlur__)
* Auto exposure (__PostProcessEyeAdaptation__)
* Tone mapping (__Tonemapper__)
* Upscaling from rendering resolution to display's resolution (__PostProcessUpscale__)

__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Number and quality of post processing features
* {{ icon_overdraw }} Number and complexity of __blendables__ (post process materials)
</div>

Description TODO. The final pass in the rendering pipeline.
