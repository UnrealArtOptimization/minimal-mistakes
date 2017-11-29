---
title: "Lighting passes"
excerpt: ""
permalink: "/book/profiling/passes-lighting/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

Lighting can often be the heaviest part of the frame. This is especially likely if your project relies on dynamic, shadowed light sources.

## LightCompositionTasks_PreLighting

{% include figure image_path="/assets/images/passes_ao.jpg" alt="" caption="__Figure:__ Screen-space ambient occlusion" %}

**Responsible for:**

* Screen-space ambient occlusion
* Decals (non-DBuffer type)

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Ambient occlusion radius and fade out distance
* {{ icon_number }} Number of decals (excluding DBuffer decals)

Ambient occlusion is a post process operation (with optional static, precomputed part). It takes the information about the scene from the G-Buffer and the hierarchical Z-Buffer. Thanks to that, it can perform all calculations in screen space, avoiding the need for querying any 3D geometry. However, it's not listed in the __PostProcessing__ category. Instead, you can find it in __LightCompositionTasks__. The full names of its sub-passes in the profiler -- something similar to `AmbientOcclusionPS (2880x1620) Upsample=1` -- reveal additional information. The half-resolution (`1440x810 Upsample=0`) is used for doing all the math, for performance reasons. Then the result is simply upscaled to the full resolution.

The __LightCompositionTasks_PreLighting__ pass has also to work on decals. The greater the number of decals (of standard type, not DBuffer decals), the longer it takes to compute it.

**Optimization**

The cost of this pass is mostly affected by the rendering resolution. You can also control ambient occlusion radius and fade out distance. The radius can be overriden by using a __Postprocess Volume__ and changing its settings. Ambient occlusion's intensity doesn't have any influence on performance, but the radius does. And in the volume's __Advanced__ drop-down category you can also set __Fade Out Distance__, changing its maximum range from the camera. Keeping it short matters for performance of __LightCompositionTasks__.

## CompositionAfterLighting

Note: In `stat gpu` it's called __CompositionPostLighting__.
{: .notice--info}

**Responsible for:**

* Subsurface scattering (SSS) of __Subsurface Profile__ type.

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Screen area covered by materials with SSS

There are two types of subsurface scattering in Unreal's materials. The older one is a very simple trick of softening the diffuse part of lighting. The newer one, called __Subsurface Profile__, is much more sophisticated. The time shown in __CompositionAfterLighting__ refers to the latter. This kind of SSS accounts for the thickness of objects using a separate buffer. It comes with a cost of approximating the thickness of geometry in real-time, then of using the buffer in shaders.

**Optimization**

To reduce the cost, you have to limit the amount of objects using the new SSS and keep their total screen-space area in check. You can also use the fake method or disable SSS in lower levels of detail (LOD).

## ComputeLightGrid

**Responsible for:**

* Optimizing lighting in forward shading 

**Cost affected by:**

* {{ icon_number }} Number of lights
* {{ icon_number }} Number of reflection capture actors

According to the comment in Unreal's source code[^lightgrid], this pass "culls local lights to a grid in frustum space". In other words: it assigns lights to cells in a grid (shaped like a pyramid along camera view). This operation has a cost of its own but it pays off later, making it faster to determine which lights affect which meshes[^clustered].

## Lights → NonShadowedLights

**Responsible for:**

* Lights in deferred rendering that don't cast shadows

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_number }} Number of movable and stationary lights
* {{ icon_area }} Radius of lights

This pass renders lighting for the _deferred shading_ -- the default method by which Unreal handles lights and materials. Please read how it works -- and how is it different from the other mode, forward rendering -- in the [relevant chapter]({{ site.baseurl }}{% link book/pipelines/forward-vs-deferred.md %}).

**Optimization**

To control the cost of non-shadowed lights, use each light's __Attenuation Distance__ and keep their total number in check. The time it takes to calculate all lighting can be simply derived from the number of pixels affected by each light. Occluded areas (as well as off-screen) do not count, as they're not visible to the camera. When lights' radii overlap, some pixels will be processed more than once. It's called __overdraw__ and can be visualized in the viewport with __Optimization Viewmodes → Light Complexity__.

Static lights don't count to the overdraw, because they're stored as precomputed (aka _baked_) lightmaps. They are not considered being "lights" anymore. If you're using baked lighting in your project (and most probably you do), set the mobility of all lights to __Static__, except for some most important ones.

# Shadows

## Lights → ShadowedLights

**Responsible for:**

* Lights that cast dynamic shadows

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_number }} Number of movable and stationary lights
* {{ icon_area }} Radius of lights
* {{ icon_triangles }} Triangle count of shadow-casting meshes

Description TODO.

## ShadowDepths

{% include figure image_path="/assets/images/passes_shadow.jpg" alt="" caption="__Figure:__ Shadow-casting spot light" %}

**Responsible for:**

* Generating depth maps for shadow-casting lights

**Cost affected by:**

* {{ icon_area }} Number and range of shadow-casting lights
* {{ icon_triangles }} Number and triangle count of movable shadow-casting meshes
* {{ icon_settings }} Shadow quality settings

The __ShadowDepths__ pass generates depth information for shadow-casting lights. It's like rendering the scene's depth from each light's point of view. The result is a texture, aka _depth map_.[^shadowtheory]

Then the engine calculates the distance of each pixel to the light source from the camera point of view -- but still in light's coordinate space. By comparing this value with the depth map, during the __ShadowProjection__ pass, it can test whether a pixel is lit by the given light or is in shadow.

{% include figure image_path="/assets/images/passes_shadow_depths.jpg" alt="" caption="__Figure:__ _Left:_ the scene from light's point of view. _Right:_ Approximate frustums of spot lights" %}

The cost of it is mostly affected by the number and the range of shadow-casting lights. What also matters is the number and triangle count of movable shadow-casting objects. Depending on the combination, you can have static meshes and static lights -- then the cost is zero, because static lights [are just textures](#lights--nonshadowedlights): they're precomputed. But you can also have, for example, stationary or movable lights and static or movable objects in their range. Both of these combinations require the engine to generate shadows from such meshes, separately for each light.

**Optimization**

Shadows are usually one of the most heavy parts of rendering, if care is not taken. The easiest way to control their performance hit is to disable shadows for every light source that doesn't need them. Very often you'll find that you can get away with the lack of shadows, without the player noticing it, especially for sources that stay far away from the area of player's movement. It can be also a good workaround to enable shadow-casting just for a single lamp in a bigger group of lights.

For shadowed lights, the biggest factor is the amount of polygons that will have to be rendered into a depth map. Light source's __Attenuation Radius__ sets the hard limit for the drawing range. Shadowed spot lights are typically less heavy than point lights, because their volume is a cone, not a full sphere. Their additional setting is __Outer Cone Angle__ -- the smaller, the better for rendering speed, because probably less object will fall into this lamp's volume.

Directional light features a multi-resolution method of rendering shadows, called _cascaded shadow maps_. It's needed because of the huge area a directional light (usually the sun) has to cover. Epic's wiki provides [{{ icon_link }} tips for using CSMs](https://wiki.unrealengine.com/LightingTroubleshootingGuide#Directional_Light_ONLY:_Cascaded_Shadow_Maps_Settings:). The most significant gains come from __Dynamic Shadow Distance__ parameter. The quality in close distance to the camera can be adujsted with __Cascade Distribution Exponent__, at the expense of farther planes.

To control the resolution of shadows, use `sg.ShadowQuality X` in the INI files, where `X` is a number between 0 and 4. High resolution shadows can reduce visible jagging (aliasing), at the cost of rendering, storing and reading back bigger depth maps.

## ShadowProjection

**Responsible for:**

* Final rendering of shadows[^shadowproj]

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Number and range of shadow-casting lights (movable and stationary)
* {{ icon_settings }} Translucency lighting volume resolution

Note: In GPU Visualizer it's shown per light, in __Light__ category. In `stat gpu` it's a separate total number.
{: .notice--info}

Shadow projection is the final rendering of shadows. This process reads the [depth map](#shadowdepths) and compares it with the scene, to detect which areas lie in shadow.

**Optimization**

Unlike __ShadowDepths__, __ShadowProjection__'s performance is also dependent on the final rendering resolution of the game. All the advice from other shadow passes applies here as well.

# Translucency and its lighting

## Translucency

**Responsible for:**

* Rendering translucent materials
* Lighting of materials that use __Surface ForwardShading__.

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_area }} Total pixel area of translucent polygons
* {{ icon_overdraw }} Overdraw
* If __Surface ForwardShading__ enabled in material: Number and radius of lights

Note: In  `stat gpu` there are only two categories: __Translucency__ and __Translucent Lighting__. In GPU Visualizer (and in text logs) the work on translucency is split into more fine-grained statistics.
{: .notice--info}

Description TODO. This is basically [the base pass]({{ site.baseurl }}{% link book/profiling/passes-base.md %}) for translucent materials. Most of the advice regaring the base pass applies here as well.

## Translucent Lighting

**Responsible for:**

* Creating a global volumetric texture used to simplify lighting of translucent meshes

**Cost affected by:**

* {{ icon_settings }} Translucency lighting volume resolution
* {{ icon_number }} Number of lights with __Affect Translucent Lighting__ enabled

Lighting of translucent objects is not computed directly. Instead, a pyramid-shaped grid is calculated from the view of the camera. It's done to speed up rendering of lit translucency. This kind of caching makes use of the assumption that translucent materials are usually used for particles, volumetric effects -- and as such don't require precise lighting.

You can see the effect when changing a material's __Blend Mode__ to __Translucent__. The object will now have more simplified lighting. Much more simplified, actually, compared to opaque materials. The volumetric cache can be avoided by using a more costly shading mode, __Surface ForwardShading__.

The resolution of the volume can be controlled with `r.TranslucencyLightingVolumeDim xx`. Reasonable values seem to fit between 16 and 128, with 64 being a default (as of UE 4.17). It can be also disabled entirely with `r.TranslucentLightingVolume 0`

In GPU Visualizer, the statistic is split into __ClearTranslucentVolumeLighting__ and __FilterTranslucentVolume__. The latter performs filtering (smoothing out) of volume[^filtertranslucent], probably to prevent potential aliasing issues. The behavior can be disabled for performance with `r.TranslucencyVolumeBlur 0` (default is 1).

You can also exclude individual lights from being rendered into the volume. It's done by disabling __Affect Translucent Lighting__ setting in light's properties. The actual cost of a single light can be found in GPU Visualizer's __Light__ pass, in every light's drop-down list, under names like __InjectNonShadowedTranslucentLighting__.

## Fog, ExponentialHeightFog

{% include figure image_path="/assets/images/passes_fog_comparison.jpg" alt="" caption="__Figure:__ _Bottom to top:_ __1.__ Fog with volumetric lighting enabled. __2.__ Standard exponential fog. __3.__ No fog." %}

**Responsible for:**

* Rendering the fog of Exponential Height Fog type[^fog]

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution

Description TODO.

# Reflections

{% include figure image_path="/assets/images/passes_reflections.jpg" alt="" caption="__Figure:__ Results of __ReflectionEnvironment__ and __ScreenSpaceReflections__ passes, combined" %}

## ReflectionEnvironment

**Responsible for:**

* Reading and blending reflection capture actors' results into a full-screen reflection buffer

**Cost affected by:**

* {{ icon_area }} Number and radius of reflection capture actors
* {{ icon_resolution }} Rendering resolution

Description TODO. Reads reflection maps from __Sphere and Box Reflection Capture__ actors and blends them into a full-screen reflection buffer.

All reflection capture probes are 128x128 pixel images. You can change this dimension in __Project Settings__. They are stored as a single big array of textures. That's the reason you can have only 341 such probes loaded in the world at once[^reflectiondocs].
{: .notice--info}


## ScreenSpaceReflections

**Responsible for:**

* Real-time dynamic reflections
* Done in post process using a screen-space ray tracing technique

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_settings }} Quality settings

Description TODO.

**Optimization**

The general quality setting is `r.SSR.Quality n`, where `n` is a number between 0 and 4.

Their use can be limited to surfaces having roughness below certain threshold. This helps with performance, because big roughness causes the rays to spread over wider angle, increasing the cost. To set the threshold, use `r.SSR.MaxRoughness x`, with `x` being a float number between 0.0 and 1.0.

# Footnotes

[^clustered]: [Slides from "Practical Clustered Shading", p. 34, Ola Olsson, Emil Persson, efficientshading.com](http://efficientshading.com/2013/08/01/practical-clustered-deferred-and-forward-shading/)
[^lightgrid]: [Unreal Engine source code, DeferredShadingRenderer.h, line 74](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h#L74)
[^filtertranslucent]: [Unreal Engine source code, TranslucentLightingShaders.usf, line 108](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Shaders/Private/TranslucentLightingShaders.usf#L108)
[^shadowproj]: [Unreal Engine source code, ShadowRendering.cpp, line 1440](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp#L1440)
[^fog]: [Unreal Engine source code, FogRendering.cpp, line 431](https://github.com/EpicGames/UnrealEngine/blob/1d2c1e48bf49836a4fee1465be87ab3f27d5ae3a/Engine/Source/Runtime/Renderer/Private/FogRendering.cpp#L431)
[^reflectiondocs]: ["Limitations", in "Reflection Environment", Unreal Engine documentation, docs.unrealengine.com](https://docs.unrealengine.com/latest/INT/Engine/Rendering/LightingAndShadows/ReflectionEnvironment/index.html#limitations)
[^shadowtheory]: ["Tutorial 16 : Shadow mapping", OpenGL-Tutorial, http://www.opengl-tutorial.org](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-16-shadow-mapping/)