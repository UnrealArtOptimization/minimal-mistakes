---
title: "Post processing pass"
excerpt: ""
permalink: "/book/profiling/passes-postprocess/"
---

{% capture icon_settings %}<i class="fa fa-sliders fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_resolution %}<i class="fa fa-television fa-fw" style="color: #ab131c" aria-hidden="true"></i>{% endcapture %}
{% capture icon_number %}<i class="fa fa-tags fa-fw" style="color: #485cbe" aria-hidden="true"></i>{% endcapture %}
{% capture icon_triangles %}<i class="fa fa-cube fa-fw" style="color: #72b4e6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_area %}<i class="fa fa-dot-circle-o fa-fw" style="color: #42ad82" aria-hidden="true"></i>{% endcapture %}
{% capture icon_overdraw %}<i class="fa fa-database fa-fw" style="color: #ddbd3b" aria-hidden="true"></i>{% endcapture %}
{% capture icon_complexity %}<i class="fa fa-gears fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}

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
