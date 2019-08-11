---
title: "Unreal's Rendering Passes"
excerpt: ""
permalink: "/book/profiling/passes/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

{% include figure image_path="/assets/images/passes_ao.jpg" alt="" caption="__Figure:__ Ambient occlusion -- the result of the PreLighting pass" %}

In this chapter you'll learn about:

* What is a _rendering pass_
* Over 20 kinds of passes in Unreal -- lighting, the base pass or the mysterious HZB
* What affects their cost (as seen in the [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}))
* How to optimize each rendering pass

# Video

If you prefer a video version of this lesson, you can [{{ icon_link }} watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA). Still, please verify the information by reading this chapter, as it contains some important errata.

{% capture tutorialvideo %}C3lumWdwHmA?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

# What is a rendering pass

Let's begin with explaining what do we call a _pass_ in the rendering pipeline. A pass is a set of [_draw calls_]({{ site.baseurl }}{% link book/profiling/index.md %}) executed on the GPU. They are grouped together by the function they have in the pipeline, like rendering transparent meshes or doing post processing. This organization is done for convenience and to ensure proper order of execution -- as some passes may need the output of a particular previous pass.

{% include figure image_path="/assets/images/passes_gbuffer_all.jpg" alt="" caption="__Figure:__ _Top left:_ Final look of the scene, after lighting and post processes. _Top right:_ World-space normals in G-Buffer. _Bottom left:_ Base color (aka albedo) in G-Buffer. _Bottom right:_ Roughness in G-Buffer." %}

Passes render meshes. This can mean 3D models in your scene or even just a single full-screen quad, as is the case with post processing. The output of most passes looks a bit strange when you [extract it]({{ site.baseurl }}{% link book/profiling/external.md %}). That's because the color represents data, like a pixel normal direction. Depth calculation or shadow projection don't look like your familiar textures too. Actually, only the base pass and translucency use the full materials we set up in the editor. Most other passes perform their tasks without them, using their own specialized shaders instead.

If you want to dive deep into the inner workings of Unreal's pipeline, I recommend reading ["How Unreal Renders a Frame"](https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame/) by Kostas Anagnostou.

# Cost of a pass

To intuitively understand what can affect the cost of a pass, it's useful to look at its inputs and its output.

A huge complexity of fragment (pixel) shaders used by a pass, combined with a big number of pixels to process, makes the cost of the pass pixel-bound. The total polygon count of meshes it works on decides if the pass is geometry-bound. There's also a dependency on memory amount and bandwith.

The __base pass__ is an example of a pass affected by all three factors. It takes visible 3D models (geometry) and renders them with full materials (pixel count), including textures (memory). Then it writes the final information into a resolution-dependent G-Buffer (so it's memory bandwith again).

If some passes take in just the G-Buffer, but not any 3D meshes - like post process effects do - then obviously they will be only pixel-bound. An increase in game's rendering resolution will directly affect their cost. On the other hand, it means that the changes in the amount of 3D meshes mean nothing to post process passes. For example, Unreal's ambient occlusion is a post process operation. It uses the hierarhical Z-buffer and some content from the G-Buffer, for example normals. By understanding what this pass requires, we know where to look for optimization opportunities -- in AO's settings and resolution adjustments, not in the scene's content.

# Guide to passes and categories

The major part of this chapter is a guide to every significant rendering pass in Unreal. You don't have to read the entire thing. Treat it more as a handbook, which you use to understand the output of the ["Stat GPU" command]({{ site.baseurl }}{% link book/process/measuring-performance.md %}) or with [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}). Both of them show you the cost of each rendering pass. This allows you to precisely locate issues in a scene, like too many shadowed lights or too many translucent materials.

Much of this information was gathered by reading the engine's source code and from practical tests. Be aware of potential mistakes here. Sources are provided in the footnotes, so you can fact-check it yourself.

Every pass' description is laid out in the following format:

* _Responsible for_ ...
* _Cost affected by_ ...
* Role of the pass
* Optimization advice

Some passes are much more important or customizable than others. Many of them react to changes in the scene, while others -- most notably post processes -- stay dependent on the resolution only. That's why the amount of information dedicated to each category varies greatly.

# Passes: Table of contents

* [__1. Base pass__](#basepass)
* [__2. Geometry__](#geometry)
    * [PrePass DDM_*](#prepass)
    * [HZB (Setup Mips)](#hzb-setup-mips)
    * [ParticleSimulation](#particlesimulation-particleinjection)
    * [ParticleInjection](#particlesimulation-particleinjection)
    * [RenderVelocities](#rendervelocities)
* [__3. Lighting__](#lighting)
    * [LightCompositionTasks_PreLighting](#lightcompositiontasks_prelighting)
    * [CompositionAfterLighting](#compositionafterlighting)
    * [ComputeLightGrid](#computelightgrid)
    * [Lights → NonShadowedLights](#lights--nonshadowedlights)
    * [Lights → ShadowedLights](#lights--shadowedlights)
    * [ShadowDepths](#shadowdepths)
    * [ShadowProjection](#shadowprojection)
    * [Translucency](#translucency)
    * [Translucent Lighting](#translucentlighting)
    * [Fog, ExponentialHeightFog](#fog-exponentialheightfog)
    * [ReflectionEnvironment](#reflectionenvironment)
    * [ScreenSpaceReflections](#screenspacereflections)
* [__4. Post processing__](#postprocessing)

# Base pass

<div class="notice" markdown="1">
__Responsible for:__

* Rendering final attributes of __Opaque__ or __Masked__ materials to the G-Buffer
* Reading static lighting and saving it to the G-Buffer
* Applying DBuffer decals
* Applying fog
* Calculating final velocity (from packed 3D velocity)
* In forward renderer: dynamic lighting
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_complexity }} Shader complexity
* {{ icon_number }} Number of objects
* {{ icon_number }} Number of decals
* {{ icon_triangles }} Triangle count
</div>

The base pass renders _material properties_, i.e. the output of node networks made in the __Material Editor__. In the default, deferred rendering mode, it saves the properties -- base color, roughness, world-space normal etc. -- into the [{{ icon_wiki }} G-Buffer](https://en.wikipedia.org/wiki/Deferred_shading) (a set of full-screen textures). Then it leaves the lighting calculation for another pass. This material data is also used later by various other passes, most notably by screen-space techniques: dynamic reflections and ambient occlusion.

But that's just a part of this pass' responsibilities. It reads static lighting -- from lightmaps, indirect lighting caches -- and applies it, by blending it immediately with material's color.

In the [forward rendering mode]({{ site.baseurl }}{% link book/pipelines/forward-vs-deferred.md %}), the base pass also calculates dynamic lighting and immediately mixes it into the color. Therefore, in foward, it takes over all responsibilities of the [lighting passes](#lighting).

The base pass also applies DBuffer decals. The decals in view are projected onto objects, their shaders are run and the results blended with the contents of the G-Buffer. Fog is also mixed in during this pass.

__Base pass optimization__

Shader complexity is obviously the most important factor here. You can check it roughly using the __Shader Complexity__ view mode. It's also useful to look at the number displayed in the __Stats__ window in __Material Editor__ -- just be sure to check this number after you press __Apply__. The "good" value of shader instructions depends on the rendering resolution of your game and the area covered by the material on screen. To test how much the performance is affected by a shader, replace it temporarily with a basic material and compare miliseconds spent in the base pass.

Watch out for memory bandwith. Textures used by materials, as well as lightmaps, need to be read every frame. Even if you re-use textures a lot (which is a good thing), it only helps to save on the GPU memory and on reading from the CPU RAM. The data still needs to be transferred to the local cache and then to shader cores. A dedicated chapter explains the [memory-related costs]({{ site.baseurl }}{% link book/pipelines/memory.md %}).

Increasing rendering resolution directly affects the weight of the G-Buffer and other full-screen textures. The memory size of the G-Buffer can be read with the command `stat RHI`. You can temporarily scale the rendering resolution with `r.ScreenPercentage XX`, e.g. `20` or `150`. This will allow you to see how much additional VRAM is needed for a given resolution.

Decals are an oft-overlooked source of trouble. This pass can take a serious hit when there are too many decals in view. Keep their material complexity low as well, though this is rarely an issue with decals.

The costs (and optimization advice) which you'll normally see in __Lights__ passes apply here instead when using forward rendering. In addition to the general rules, keep light overlap (aka overdraw) in check. Forward renderers are genetically allergic to multiple lights hitting the same object.

{{ btn_top }}

# Geometry

## PrePass

{% include figure image_path="/assets/images/passes_prepass.jpg" alt="" caption="__Figure:__ Depth buffer (aka z-buffer) - the result of pre-pass" %}

<div class="notice" markdown="1">
__Responsible for:__

* Early rendering of depth (Z) from non-translucent meshes
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_triangles }} Triangle count of meshes with __Opaque__ materials
* {{ icon_overdraw }} Depending on __Early Z__ setting: Triangle count and complexity of __Masked__ materials
</div>

Description TODO. Its results are required by DBuffer decals. The pre-pass may also be used by occlusion culling.

__PrePass optimization__

This pass is influenced by the choice in __Engine → Rendering → Optimizations → Early Z-pass__.

## HZB (Setup Mips)

<div class="notice" markdown="1">
__Responsible for:__

* Generating the Hierarchical Z-Buffer
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
</div>

The HZB is used by an occlusion culling method[^hzbocclusion] and by screen-space techniques for ambient occlusion and reflections[^hzbuse].

Warning: Time may be extreme in editor, but don't worry. It's usually a false alarm, due to an bug in Unreal. Just check the value in an actual build. The bug is hopefully fixed since UE 4.18[^hzbbug].
{: .notice--warning}

## ParticleSimulation, ParticleInjection

<div class="notice" markdown="1">
__Responsible for:__

* Particle simulation on the GPU (only of __GPU Sprites__ particle type)
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_number }} Number of particles spawned by __GPU Sprites__ emitters
* {{ icon_settings }} __Collision (Scene Depth)__ module
</div>

{% include figure image_path="/assets/images/passes_particles.jpg" alt="" caption="__Figure:__ GPU-simulated particles with screen-space collision enabled" %}

If you enabled __GPU Sprites__ in a particle emitter, then their physics simulation is done here. The cost depends on the number of particles spawned by such emitters. Enabling the __Collision__ module increases the complexity of the simulation. The collision of GPU sprites with the rest of the scene is tested against screen-space data (for example the Z-depth). This makes it faster than the traditional, CPU-based collision of particles against actual 3D meshes. Still, it's not entirely free -- and the cost is moved to the GPU.

__Particle passes optimization__

The bigger the number of particles that have to be simulated (especially collide), the bigger the cost. Remember that you can use particle level of detail (__LOD__) to limit the number of particles being spawned. You may get away with much smaller numbers if the emitter is viewed from far distance.

## RenderVelocities

<div class="notice" markdown="1">
__Responsible for:__

* Saving velocity of each vertex (used later by motion blur and temporal anti-aliasing)
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_number }} Number of moving objects
* {{ icon_triangles }} Triangle count of moving objects
</div>

Measures the velocity of every moving vertex and saves it into the motion blur velocity buffer.

{{ btn_top }}

# Lighting

Lighting can often be the heaviest part of the frame. This is especially likely if your project relies on dynamic, shadowed light sources.

## Direct lighting

### ComputeLightGrid

<div class="notice" markdown="1">
__Responsible for:__

* Optimizing lighting in forward shading 
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_number }} Number of lights
* {{ icon_number }} Number of reflection capture actors
</div>

According to the comment in Unreal's source code[^lightgrid], this pass "culls local lights to a grid in frustum space". In other words: it assigns lights to cells in a grid (shaped like a pyramid along camera view). This operation has a cost of its own but it pays off later, making it faster to determine which lights affect which meshes[^clustered].

### Lights → NonShadowedLights

<div class="notice" markdown="1">
__Responsible for:__

* Lights in deferred rendering that don't cast shadows
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_number }} Number of movable and stationary lights
* {{ icon_area }} Radius of lights
</div>

This pass renders lighting for the _deferred shading_ -- the default method by which Unreal handles lights and materials. Please read how it works -- and how is it different from the other mode, forward rendering -- in the [relevant chapter]({{ site.baseurl }}{% link book/pipelines/forward-vs-deferred.md %}).

__NonShadowedLights optimization__

To control the cost of non-shadowed lights, use each light's __Attenuation Distance__ and keep their total number in check. The time it takes to calculate all lighting can be simply derived from the number of pixels affected by each light. Occluded areas (as well as off-screen) do not count, as they're not visible to the camera. When lights' radii overlap, some pixels will be processed more than once. It's called __overdraw__ and can be visualized in the viewport with __Optimization Viewmodes → Light Complexity__.

Static lights don't count to the overdraw, because they're stored as precomputed (aka _baked_) lightmaps. They are not considered being "lights" anymore. If you're using baked lighting in your project (and most probably you do), set the mobility of all lights to __Static__, except for some most important ones.

## Shadows

### Lights → ShadowedLights

<div class="notice" markdown="1">
__Responsible for:__

* Lights that cast dynamic shadows
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_number }} Number of movable and stationary lights
* {{ icon_area }} Radius of lights
* {{ icon_triangles }} Triangle count of shadow-casting meshes
</div>

Description TODO.

### ShadowDepths

<div class="notice" markdown="1">
__Responsible for:__

* Generating depth maps for shadow-casting lights
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_area }} Number and range of shadow-casting lights
* {{ icon_triangles }} Number and triangle count of movable shadow-casting meshes
* {{ icon_settings }} Shadow quality settings
</div>

{% include figure image_path="/assets/images/passes_shadow.jpg" alt="" caption="__Figure:__ Shadow-casting spot light" %}

__ShadowDepths__ pass generates depth information for shadow-casting lights. It's like rendering the scene's depth from each light's point of view. The result is a texture, aka _depth map_.[^shadowtheory]

Then the engine calculates the distance of each pixel to the light source from the camera point of view -- but still in light's coordinate space. By comparing this value with the depth map, during the __ShadowProjection__ pass, it can test whether a pixel is lit by the given light or is in shadow.

{% include figure image_path="/assets/images/passes_shadow_depths.jpg" alt="" caption="__Figure:__ _Left:_ the scene from light's point of view. _Right:_ Approximate frustums of spot lights" %}

The cost of it is mostly affected by the number and the range of shadow-casting lights. What also matters is the number and triangle count of movable shadow-casting objects. Depending on the combination, you can have static meshes and static lights -- then the cost is zero, because static lights [are just textures](#lights--nonshadowedlights): they're precomputed. But you can also have, for example, stationary or movable lights and static or movable objects in their range. Both of these combinations require the engine to generate shadows from such meshes, separately for each light.

__ShadowDepths optimization__

Shadows are usually one of the most heavy parts of rendering, if care is not taken. The easiest way to control their performance hit is to disable shadows for every light source that doesn't need them. Very often you'll find that you can get away with the lack of shadows, without the player noticing it, especially for sources that stay far away from the area of player's movement. It can be also a good workaround to enable shadow-casting just for a single lamp in a bigger group of lights.

For shadowed lights, the biggest factor is the amount of polygons that will have to be rendered into a depth map. Light source's __Attenuation Radius__ sets the hard limit for the drawing range. Shadowed spot lights are typically less heavy than point lights, because their volume is a cone, not a full sphere. Their additional setting is __Outer Cone Angle__ -- the smaller, the better for rendering speed, because probably less object will fall into this lamp's volume.

Directional light features a multi-resolution method of rendering shadows, called _cascaded shadow maps_. It's needed because of the huge area a directional light (usually the sun) has to cover. Epic's wiki provides [{{ icon_link }} tips for using CSMs](https://wiki.unrealengine.com/LightingTroubleshootingGuide#Directional_Light_ONLY:_Cascaded_Shadow_Maps_Settings:). The most significant gains come from __Dynamic Shadow Distance__ parameter. The quality in close distance to the camera can be adujsted with __Cascade Distribution Exponent__, at the expense of farther planes.

To control the resolution of shadows, use `sg.ShadowQuality X` in the INI files, where `X` is a number between 0 and 4. High resolution shadows can reduce visible jagging (aliasing), at the cost of rendering, storing and reading back bigger depth maps.

### ShadowProjection

<div class="notice" markdown="1">
__Responsible for:__

* Final rendering of shadows[^shadowproj]
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Number and range of shadow-casting lights (movable and stationary)
* {{ icon_settings }} Translucency lighting volume resolution
</div>

Note: In GPU Visualizer it's shown per light, in __Light__ category. In `stat gpu` it's a separate total number.
{: .notice--info}

Shadow projection is the final rendering of shadows. This process reads the [depth map](#shadowdepths) and compares it with the scene, to detect which areas lie in shadow.

__ShadowProjection optimization__

Unlike __ShadowDepths__, __ShadowProjection__'s performance is also dependent on the final rendering resolution of the game. All the advice from other shadow passes applies here as well.

## Indirect lighting

### LightCompositionTasks_PreLighting

{% include figure image_path="/assets/images/passes_ao.jpg" alt="" caption="__Figure:__ Screen-space ambient occlusion" %}

<div class="notice" markdown="1">
__Responsible for:__

* Screen-space ambient occlusion
* Decals (non-DBuffer type)
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Ambient occlusion radius and fade out distance
* {{ icon_number }} Number of decals (excluding DBuffer decals)
</div>

Ambient occlusion is a post process operation (with optional static, precomputed part). It takes the information about the scene from the G-Buffer and the hierarchical Z-Buffer. Thanks to that, it can perform all calculations in screen space, avoiding the need for querying any 3D geometry. However, it's not listed in the __PostProcessing__ category. Instead, you can find it in __LightCompositionTasks__. The full names of its sub-passes in the profiler -- something similar to `AmbientOcclusionPS (2880x1620) Upsample=1` -- reveal additional information. The half-resolution (`1440x810 Upsample=0`) is used for doing all the math, for performance reasons. Then the result is simply upscaled to the full resolution.

The __LightCompositionTasks_PreLighting__ pass has also to work on decals. The greater the number of decals (of standard type, not DBuffer decals), the longer it takes to compute it.

__PreLighting optimization__

The cost of this pass is mostly affected by the rendering resolution. You can also control ambient occlusion radius and fade out distance. The radius can be overriden by using a __Postprocess Volume__ and changing its settings. Ambient occlusion's intensity doesn't have any influence on performance, but the radius does. And in the volume's __Advanced__ drop-down category you can also set __Fade Out Distance__, changing its maximum range from the camera. Keeping it short matters for performance of __LightCompositionTasks__.

### CompositionAfterLighting

<div class="notice" markdown="1">
__Responsible for:__

* Subsurface scattering (SSS) of __Subsurface Profile__ type.
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Screen area covered by materials with SSS
</div>

Note: In `stat gpu` it's called __CompositionPostLighting__.
{: .notice--info}

There are two types of subsurface scattering in Unreal's materials. The older one is a very simple trick of softening the diffuse part of lighting. The newer one, called __Subsurface Profile__, is much more sophisticated. The time shown in __CompositionAfterLighting__ refers to the latter. This kind of SSS accounts for the thickness of objects using a separate buffer. It comes with a cost of approximating the thickness of geometry in real-time, then of using the buffer in shaders.

__CompositionAfterLighting optimization__

To reduce the cost, you have to limit the amount of objects using the new SSS and keep their total screen-space area in check. You can also use the fake method or disable SSS in lower levels of detail (LOD).

## Translucency and its lighting

### Translucency

<div class="notice" markdown="1">
__Responsible for:__

* Rendering translucent materials
* Lighting of materials that use __Surface ForwardShading__.
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Total pixel area of translucent polygons
* {{ icon_overdraw }} Overdraw
* If __Surface ForwardShading__ enabled in material: Number and radius of lights
</div>

Note: In  `stat gpu` there are only two categories: __Translucency__ and __Translucent Lighting__. In GPU Visualizer (and in text logs) the work on translucency is split into more fine-grained statistics.
{: .notice--info}

Description TODO. This is basically [the base pass](#basepass) for translucent materials. Most of the advice regaring the base pass applies here as well.

### Translucent Lighting

<div class="notice" markdown="1">
__Responsible for:__

* Creating a global volumetric texture used to simplify lighting of translucent meshes
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_settings }} Translucency lighting volume resolution
* {{ icon_number }} Number of lights with __Affect Translucent Lighting__ enabled
</div>

Lighting of translucent objects is not computed directly. Instead, a pyramid-shaped grid is calculated from the view of the camera. It's done to speed up rendering of lit translucency. This kind of caching makes use of the assumption that translucent materials are usually used for particles, volumetric effects -- and as such don't require precise lighting.

You can see the effect when changing a material's __Blend Mode__ to __Translucent__. The object will now have more simplified lighting. Much more simplified, actually, compared to opaque materials. The volumetric cache can be avoided by using a more costly shading mode, __Surface ForwardShading__.

__Translucent lighting optimization__

The resolution of the translucent lighting volume can be controlled with `r.TranslucencyLightingVolumeDim xx`. Reasonable values seem to fit between 16 and 128, with 64 being a default (as of UE 4.17). It can be also disabled entirely with `r.TranslucentLightingVolume 0`

In GPU Visualizer, the statistic is split into __ClearTranslucentVolumeLighting__ and __FilterTranslucentVolume__. The latter performs filtering (smoothing out) of volume[^filtertranslucent], probably to prevent potential aliasing issues. The behavior can be disabled for performance with `r.TranslucencyVolumeBlur 0` (default is 1).

You can also exclude individual lights from being rendered into the volume. It's done by disabling __Affect Translucent Lighting__ setting in light's properties. The actual cost of a single light can be found in GPU Visualizer's __Light__ pass, in every light's drop-down list, under names like __InjectNonShadowedTranslucentLighting__.

### Fog, ExponentialHeightFog

{% include figure image_path="/assets/images/passes_fog_comparison.jpg" alt="" caption="__Figure:__ _Bottom to top:_ __1.__ Fog with volumetric lighting enabled. __2.__ Standard exponential fog. __3.__ No fog." %}

<div class="notice" markdown="1">
__Responsible for:__

* Rendering the fog of Exponential Height Fog type[^fog]
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
</div>

Description TODO.

## Reflections

{% include figure image_path="/assets/images/passes_reflections.jpg" alt="" caption="__Figure:__ Results of __ReflectionEnvironment__ and __ScreenSpaceReflections__ passes, combined" %}

### ReflectionEnvironment

<div class="notice" markdown="1">
__Responsible for:__

* Reading and blending reflection capture actors' results into a full-screen reflection buffer
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_area }} Number and radius of reflection capture actors
* {{ icon_resolution }} Rendering resolution
</div>

Reads reflection maps from __Sphere and Box Reflection Capture__ actors and blends them into a full-screen reflection buffer.

All reflection capture probes are 128x128 pixel images. You can change this dimension in __Project Settings__. They are stored as a single big array of textures. That's the reason you can have only 341 such probes loaded in the world at once[^reflectiondocs].
{: .notice--info}

### ScreenSpaceReflections

<div class="notice" markdown="1">
__Responsible for:__

* Real-time dynamic reflections
* Done in post process using a screen-space ray tracing technique
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Quality settings
</div>

Description TODO.

__ScreenSpaceReflections optimization__

The general quality setting is `r.SSR.Quality n`, where `n` is a number between 0 and 4.

Their use can be limited to surfaces having roughness below certain threshold. This helps with performance, because big roughness causes the rays to spread over wider angle, increasing the cost. To set the threshold, use `r.SSR.MaxRoughness x`, with `x` being a float number between 0.0 and 1.0.

{{ btn_top }}

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
</div>

<div class="notice" markdown="1">
__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Number and quality of post processing features
* {{ icon_overdraw }} Number and complexity of __blendables__ (post process materials)
</div>

Description TODO. The final pass in the rendering pipeline, dealing with full-screen effects.

{{ btn_top }}

# Footnotes

[^hzbocclusion]: [Accepted answer to "How does object occlusion and culling work in UE4?", Tim Hobson, answers.unrealengine.com](https://answers.unrealengine.com/questions/312646/how-does-object-occlusion-and-culling-work-in-ue4.html)
[^hzbuse]: ["Screen Space Reflections in Killing Floor 2", Sakib Saikia, sakibsaikia.github.io](https://sakibsaikia.github.io/graphics/2016/12/25/Screen-Space-Reflection-in-Killing-Floor-2.html)
[^hzbbug]: [Bug UE-3448HZB, "Setup Mips taking considerable time in GPU Visualizer", issues.unrealengine.com](https://issues.unrealengine.com/issue/UE-33448)
[^clustered]: [Slides from "Practical Clustered Shading", p. 34, Ola Olsson, Emil Persson, efficientshading.com](http://efficientshading.com/2013/08/01/practical-clustered-deferred-and-forward-shading/)
[^lightgrid]: [Unreal Engine source code, DeferredShadingRenderer.h, line 74](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h#L74)
[^filtertranslucent]: [Unreal Engine source code, TranslucentLightingShaders.usf, line 108](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Shaders/Private/TranslucentLightingShaders.usf#L108)
[^shadowproj]: [Unreal Engine source code, ShadowRendering.cpp, line 1440](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp#L1440)
[^fog]: [Unreal Engine source code, FogRendering.cpp, line 431](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/FogRendering.cpp#L431)
[^reflectiondocs]: ["Limitations", in "Reflection Environment", Unreal Engine documentation, docs.unrealengine.com](https://docs.unrealengine.com/latest/INT/Engine/Rendering/LightingAndShadows/ReflectionEnvironment/index.html#limitations)
[^shadowtheory]: ["Tutorial 16 : Shadow mapping", OpenGL-Tutorial, http://www.opengl-tutorial.org](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-16-shadow-mapping/)
