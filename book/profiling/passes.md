---
title: "Unreal's rendering passes"
excerpt: ""
permalink: "/book/profiling/passes/"
---

{% include toc icon="columns" title=page.title %}

This chapter covers the majority of rendering passes in Unreal Engine. What they are, what affects their cost and what are the opportunities for optimization in the quality settings and in your content.

## Video

If you prefer a video version of this lesson, you can [watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA).

{% capture tutorialvideo %}C3lumWdwHmA?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

## What is a rendering pass

Let's begin with explaining what do we call a "pass" in the rendering pipeline. A pass is a set of draw calls (be sure to read [what they are]({{ site.baseurl }}{% link book/profiling/index.md %})) executed on the GPU. They are grouped together by the function they have in the pipeline, like rendering transparent meshes or doing post processing. This organization is done for convenience and to ensure proper order of execution - as some passes may need the output of a particular previous pass.

Passes render geometry -- a.k.a meshes. This can mean 3D models in your scene or even just a single full-screen quad, as is the case with post processing. The output of most passes looks quite alien when you [extract it]({{ site.baseurl }}{% link book/profiling/external.md %})): depth calculation or shadow projection don't look like your familiar textured scenes. Actually, only the base pass and translucency use the full materials we set up in the editor. Most other passes perform their tasks without them, using their own specialized shaders instead.

### Cost of a pass

To intuitively understand what can affect the cost of a pass, it's useful to look at its inputs and its output.

A huge complexity of fragment (pixel) shaders used by a pass, combined with a big number of pixels to process, makes the cost of the pass pixel-bound. The total polygon count of meshes it works on decides if the pass is geometry-bound. There's also a dependency on memory amount and bandwith.

The base pass is an example of a pass affected by all three factors. It takes visible 3D models (geometry) and renders them with full materials (pixel count), including textures (memory). Then it writes the final information into a resolution-dependent GBuffer (so it's memory bandwith again).

If some passes take in just the GBuffer, but not any 3D meshes - like post process effects do - then obviously they will be only pixel-bound. An increase in game's rendering resolution will directly affect their cost. On the other hand, it means that the changes in the amount of 3D meshes mean nothing to post process passes. For example, Unreal's ambient occlusion is a post process operation. It uses the hierarhical Z-buffer and some content from the GBuffer, for example normals. By understanding what this pass requires, we know where to look for optimization opportunities -- in AO's settings and resolution adjustments, not in the scene's content.

This chapter provides information about dependencies of each pass. It was derived from reading the engine's source code and from practical tests. Be aware of potential mistakes here -- there are sources provided in the footnotes, so you can fact-check it yourself.

There's also a lot of optimization advice here, to help you fix a bottleneck once you locate it.

## Lighting

Let us begin with lighting passes. Either they or the base pass are probably the most heavy part of your scene, unless you have a distinct art style or you have some optimization issues, like an excessive amount of particles.

### LightCompositionTasks_PreLighting

Responsible for:
* _Screen-space ambient occlusion_
* _Decals (non-DBuffer type)._

Cost affected by:
* Rendering resolution
* Ambient occlusion radius and fade out distance
* Number of decals (excluding DBuffer decals)

You may think that ambient occlusion is a post process operation. Yes, it is -- but it's not listed in the __PostProcessing__ category. Instead, you can find it in __LightCompositionTasks__. The full names of its sub-passes in the profiler -- something similar to `AmbientOcclusionPS (2880x1620)` -- reveal additional information, for example the resolution ambient occlusion was rendered in. There's also the half-resolution (`1440x810`), which is how ambient occlusion is usually done for performance reasons.

The cost of this pass is mostly affected by rendering resolution. You can also control ambient occlusion radius and fade out distance. The radius can be overriden by using a __Postprocess Volume__ and changing its settings. Ambient occlusion's intensity doesn't have any influence on performance, but the radius does. And in the volume's __Advanced__ drop-down category you can also set __Fade Out Distance__, changing its maximum range from the camera. Keeping it short matters for performance of __LightCompositionTasks__.

The __LightCompositionTasks_PreLighting__ pass has also to work on decals. The greater the number of decals (of standard type, so not DBuffer decals), the longer it takes to compute it.

## Base pass

## Translucency

## Geometry

## Post process
