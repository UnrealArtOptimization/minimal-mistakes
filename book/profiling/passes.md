---
title: "Unreal's rendering passes"
excerpt: ""
permalink: "/book/profiling/passes/"
---

{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* What are the _rendering passes_ in a real-time engine
* Over 20 kinds of passes in Unreal -- like lighting, the base pass or the mysterious HZB
* What affects their cost (as seen in [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}) or `stat gpu`)
* What are the opportunities for optimization

## Video

If you prefer a video version of this lesson, you can [watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA). Still, please be sure to verify the information by reading this chapter, as it contains some important errata.

{% capture tutorialvideo %}C3lumWdwHmA?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

## What is a rendering pass

Let's begin with explaining what do we call a _pass_ in the rendering pipeline. A pass is a set of draw calls (be sure to read [what they are]({{ site.baseurl }}{% link book/profiling/index.md %})) executed on the GPU. They are grouped together by the function they have in the pipeline, like rendering transparent meshes or doing post processing. This organization is done for convenience and to ensure proper order of execution - as some passes may need the output of a particular previous pass.

Passes render geometry -- a.k.a meshes. This can mean 3D models in your scene or even just a single full-screen quad, as is the case with post processing. The output of most passes looks quite alien when you [extract it]({{ site.baseurl }}{% link book/profiling/external.md %}). Depth calculation or shadow projection don't look like your familiar textured scenes. Actually, only the base pass and translucency use the full materials we set up in the editor. Most other passes perform their tasks without them, using their own specialized shaders instead.

### Cost of a pass

To intuitively understand what can affect the cost of a pass, it's useful to look at its inputs and its output.

A huge complexity of fragment (pixel) shaders used by a pass, combined with a big number of pixels to process, makes the cost of the pass pixel-bound. The total polygon count of meshes it works on decides if the pass is geometry-bound. There's also a dependency on memory amount and bandwith.

The base pass is an example of a pass affected by all three factors. It takes visible 3D models (geometry) and renders them with full materials (pixel count), including textures (memory). Then it writes the final information into a resolution-dependent GBuffer (so it's memory bandwith again).

If some passes take in just the GBuffer, but not any 3D meshes - like post process effects do - then obviously they will be only pixel-bound. An increase in game's rendering resolution will directly affect their cost. On the other hand, it means that the changes in the amount of 3D meshes mean nothing to post process passes. For example, Unreal's ambient occlusion is a post process operation. It uses the hierarhical Z-buffer and some content from the GBuffer, for example normals. By understanding what this pass requires, we know where to look for optimization opportunities -- in AO's settings and resolution adjustments, not in the scene's content.

This chapter provides information about dependencies of each pass. It was derived from reading the engine's source code and from practical tests. Be aware of potential mistakes here -- there are sources provided in the footnotes, so you can fact-check it yourself.

There's also a lot of optimization advice here, to help you fix a bottleneck once you locate it.

## Lighting

Let us begin with lighting passes. They often can be the heaviest part of the frame if your project relies on dynamic, shadowed light sources.

### LightCompositionTasks_PreLighting

Responsible for:
* Screen-space ambient occlusion
* Decals (non-DBuffer type)

Cost affected by:
* Rendering resolution
* Ambient occlusion radius and fade out distance
* Number of decals (excluding DBuffer decals)

You may think that ambient occlusion is a post process operation. Yes, it is -- but it's not listed in the __PostProcessing__ category. Instead, you can find it in __LightCompositionTasks__. The full names of its sub-passes in the profiler -- something similar to `AmbientOcclusionPS (2880x1620)` -- reveal additional information, for example the resolution ambient occlusion was rendered in. There's also the half-resolution (`1440x810`), which is how ambient occlusion is usually done for performance reasons.

The cost of this pass is mostly affected by rendering resolution. You can also control ambient occlusion radius and fade out distance. The radius can be overriden by using a __Postprocess Volume__ and changing its settings. Ambient occlusion's intensity doesn't have any influence on performance, but the radius does. And in the volume's __Advanced__ drop-down category you can also set __Fade Out Distance__, changing its maximum range from the camera. Keeping it short matters for performance of __LightCompositionTasks__.

The __LightCompositionTasks_PreLighting__ pass has also to work on decals. The greater the number of decals (of standard type, so not DBuffer decals), the longer it takes to compute it.

### CompositionAfterLighting

Note: In `stat gpu` it's called __CompositionPostLighting__.
{: .notice--info}

Responsible for:
* Subsurface scattering (SSS) of __Subsurface Profile__ type.

Cost affected by:
* Rendering resolution
* Screen area covered by materials with SSS


### ComputeLightGrid

Responsible for:
* Optimizing lighting in forward shading 

Cost affected by:
* Number of lights
* Number of reflection capture actors

According to the comment in Unreal's source code[^lightgrid], this pass "culls local lights to a grid in frustum space". In other words: it assigns lights to cells in a grid (shaped like a pyramid along camera view). This operation has a cost of its own but it pays off later, making it faster to determine which lights affect which meshes[^clustered].

### Lights → NonShadowedLights

Responsible for:
* Lights in deferred rendering which don't cast shadows

Cost affected by:
* Rendering resolution
* Number of movable and stationary lights
* Radius of lights

Description TODO.

### Lights → ShadowedLights

Responsible for:
* Lights which cast dynamic shadows

Cost affected by:
* Rendering resolution
* Number of movable and stationary lights
* Radius of lights
* Triangle count of shadow-casting meshes

Description TODO.

### ShadowDepths

Responsible for:
* Generating depth maps for shadow-casting lights

Cost affected by:
* Number and range of shadow-casting lights
* Number and triangle count of movable shadow-casting meshes
* Shadow quality settings

It's like rendering scene's depth from the light's point of view.

To control the resolution of shadows, which directly affects the cost, use `sg.ShadowQuality x`, where `x` is a number between 0 and 4.

### ShadowProjection

Responsible for:
* Final rendering of shadows[^shadowproj]

Cost affected by:
* Rendering resolution
* Number and range of shadow-casting lights (movable and stationary)
* Translucency lighting volume resolution

In GPU Visualizer it's shown per light, in __Light__ category. In `stat gpu` it's a separate total number.

## Reflections

### ReflectionEnvironment

Responsible for:
* Reading and blending reflection capture actor's results into a full-screen reflection buffer

Cost affected by:
* Number and radius of reflection capture actors
* Rendering resolution

Description TODO.

### ScreenSpaceReflections

Responsible for:
* Real-time dynamic reflections
* Done in post process using a screen-space ray tracing technique

Cost affected by:
* Rendering resolution
* Quality settings

The general quality setting is `r.SSR.Quality n`, where `n` is a number between 0 and 4.

Their use can be limited to surfaces having roughness below certain threshold. This helps with performance, because big roughness causes the rays to spread over wider angle, increasing the cost. To set the threshold, use `r.SSR.MaxRoughness x`, with `x` being a float number between 0.0 and 1.0.

## Base pass

Responsible for:
* Rendering final attributes of __Opaque__ or __Masked__ materials to the GBuffer
* Reading static lighting and saving it to the GBuffer
* Applying DBuffer decals
* Applying fog
* Calculating final velocity (from packed 3D velocity)
* In forward renderer: dynamic lighting

Cost affected by:
* Rendering resolution
* Number of objects
* Shader complexity
* Number of decals
* Triangle count

Description TODO.

## Translucency and its lighting

Note: In  `stat gpu` there are only two categories: __Translucency__ and __Translucent Lighting__. In GPU Visualizer (and in text logs) the work on translucency is split into more fine-grained statistics.
{: .notice--info}

### Translucency

Responsible for:
* Rendering translucent materials (like the base pass)
* Lighting of translucent materials of __Surface ForwardShading__ type.

Cost affected by:
* Rendering resolution
* Total pixel area of translucent polygons
* Overdraw
* If __Surface ForwardShading__ enabled in material: Number and radius of lights

Description TODO.

### Translucent Lighting

Responsible for:
* Creating a global volumetric texture used to simplify lighting of translucent meshes

Cost affected by:
* Translucency lighting volume resolution
* Number of lights with __Affect Translucent Lighting__ enabled

Lighting of translucent objects is not computed directly. Instead, a pyramid-shaped grid is calculated from the view of the camera. It's done to speed up rendering of lit translucency. This kind of caching makes use of the assumption that translucent materials are usually used for particles, volumetric effects -- and as such don't require precise lighting.

You can see the effect when changing a material's __Blend Mode__ to __Translucent__. The object will now have more simplified lighting. Much more simplified, actually, compared to opaque materials. The volumetric cache can be avoided by using a more costly shading mode, __Surface ForwardShading__.

The resolution of the volume can be controlled with `r.TranslucencyLightingVolumeDim xx`. Reasonable values seem to fit between 16 and 128, with 64 being a default (as of UE 4.17). It can be also disabled entirely with `r.TranslucentLightingVolume 0`

In GPU Visualizer, the statistic is split into __ClearTranslucentVolumeLighting__ and __FilterTranslucentVolume__. The latter performs filtering (smoothing out) of volume[^filtertranslucent], probably to prevent potential aliasing issues. The behavior can be disabled for performance with `r.TranslucencyVolumeBlur 0` (default is 1).

You can also exclude individual lights from being rendered into the volume. It's done by disabling __Affect Translucent Lighting__ setting in light's properties. The actual cost of a single light can be found in GPU Visualizer's __Light__ pass, in every light's drop-down list, under names like __InjectNonShadowedTranslucentLighting__.

### Fog, ExponentialHeightFog

Responsible for:
* Rendering the fog of Exponential Height Fog type[^fog]

Cost affected by:
* Rendering resolution

Description TODO.

## Geometry

### PrePass DDM_...

Responsible for:
* Early rendering of depth (Z) from non-translucent meshes

Cost affected by:
* Triangle count of meshes with __Opaque__ materials
* Depending on __Early Z__ setting: Overdraw, triangle count and complexity of __Masked__ materials

Its results are required by DBuffer decals. They may also be used by occlusion culling.

__Engine → Rendering → Optimizations → Early Z-pass__

### HZB (Setup Mips)

Responsible for:
* Generating the Hierarchical Z-Buffer

Cost affected by:
* Rendering resolution

The HZB is used by an occlusion culling method[^hzbocclusion] and by screen-space techniques for ambient occlusion and reflections[^hzbuse].

Warning: Time may be extreme in editor, but don't worry. It's usually a false alarm, due to an bug in Unreal. Just check the value in an actual build. The bug is hopefully fixed since UE 4.18[^hzbbug].
{: .notice--warning}

### ParticleSimulation, ParticleInjection

Responsible for:
* Particle simulation on the GPU (only of __GPU Sprites__ particle type)

Cost affected by:
* Number of particles spawned by __GPU Sprites__ emitters
* __Collision (Scene Depth)__ module

Description TODO.

### RenderVelocities

Responsible for:
* Saving velocity of each vertex (used later by motion blur and temporal anti-aliasing)

Cost affected by:
* Number of moving objects
* Triangle count of moving objects

Description TODO.

## PostProcessing

Responsible for:
* Depth of Field (__BokehDOFRecombine__)
* Temporal anti-aliasing (__TemporalAA__)
* Reading velocity values (__VelocityFlatten__)
* Motion blur (__MotionBlur__)
* Auto exposure (__PostProcessEyeAdaptation__)
* Tone mapping (__Tonemapper__)
* Upscaling from rendering resolution to display's resolution (__PostProcessUpscale__)

Cost affected by:
* Rendering resolution
* Number and quality of post processing features
* Number and complexity of __blendables__ (post process materials)

Description TODO.

## Footnotes

## Sources

[^lightgrid]: [Unreal Engine source code, DeferredShadingRenderer.h, line 74](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h#L74)
[^clustered]: [Slides from "Practical Clustered Shading", p. 34, Ola Olsson, Emil Persson, efficientshading.com](http://efficientshading.com/2013/08/01/practical-clustered-deferred-and-forward-shading/)
[^filtertranslucent]: [Unreal Engine source code, TranslucentLightingShaders.usf, line 108](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Shaders/Private/TranslucentLightingShaders.usf#L108)
[^shadowproj]: [Unreal Engine source code, ShadowRendering.cpp, line 1440](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp#L1440)
[^fog]: [Unreal Engine source code, FogRendering.cpp, line 431](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/FogRendering.cpp#L431)
[^hzbocclusion]: [Accepted answer to "How does object occlusion and culling work in UE4?", Tim Hobson, answers.unrealengine.com](https://answers.unrealengine.com/questions/312646/how-does-object-occlusion-and-culling-work-in-ue4.html)
[^hzbuse]: ["Screen Space Reflections in Killing Floor 2", Sakib Saikia, sakibsaikia.github.io](https://sakibsaikia.github.io/graphics/2016/12/25/Screen-Space-Reflection-in-Killing-Floor-2.html)
[^hzbbug]: [Bug UE-3448HZB, "Setup Mips taking considerable time in GPU Visualizer", issues.unrealengine.com](https://issues.unrealengine.com/issue/UE-33448)