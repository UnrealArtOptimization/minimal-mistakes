---
title: "Unreal's rendering passes"
excerpt: ""
permalink: "/book/profiling/passes/"
---

{% capture icon_settings %}<i class="fa fa-sliders fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_resolution %}<i class="fa fa-television fa-fw" style="color: #ab131c" aria-hidden="true"></i>{% endcapture %}
{% capture icon_number %}<i class="fa fa-tags fa-fw" style="color: #485cbe" aria-hidden="true"></i>{% endcapture %}
{% capture icon_triangles %}<i class="fa fa-cube fa-fw" style="color: #72b4e6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_area %}<i class="fa fa-dot-circle-o fa-fw" style="color: #42ad82" aria-hidden="true"></i>{% endcapture %}
{% capture icon_overdraw %}<i class="fa fa-database fa-fw" style="color: #ddbd3b" aria-hidden="true"></i>{% endcapture %}
{% capture icon_complexity %}<i class="fa fa-gears fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}

{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* What are the _rendering passes_ in a real-time engine
* Over 20 kinds of passes in Unreal -- like lighting, the base pass or the mysterious HZB
* What affects their cost (as seen in [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}) or `stat gpu`)
* What are the opportunities for optimization

# Video

If you prefer a video version of this lesson, you can [watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA). Still, please verify the information by reading this chapter, as it contains some important errata.

{% capture tutorialvideo %}C3lumWdwHmA?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

# What is a rendering pass

Let's begin with explaining what do we call a _pass_ in the rendering pipeline. A pass is a set of draw calls (be sure to read [what they are]({{ site.baseurl }}{% link book/profiling/index.md %})) executed on the GPU. They are grouped together by the function they have in the pipeline, like rendering transparent meshes or doing post processing. This organization is done for convenience and to ensure proper order of execution - as some passes may need the output of a particular previous pass.

{% include figure image_path="/assets/images/passes_reflections.jpg" alt="" caption="__Figure:__ The result of reflection passes" %}

Passes render geometry -- a.k.a meshes. This can mean 3D models in your scene or even just a single full-screen quad, as is the case with post processing. The output of most passes looks quite alien when you [extract it]({{ site.baseurl }}{% link book/profiling/external.md %}). Depth calculation or shadow projection don't look like your familiar textured scenes. Actually, only the base pass and translucency use the full materials we set up in the editor. Most other passes perform their tasks without them, using their own specialized shaders instead.

## Cost of a pass

To intuitively understand what can affect the cost of a pass, it's useful to look at its inputs and its output.

A huge complexity of fragment (pixel) shaders used by a pass, combined with a big number of pixels to process, makes the cost of the pass pixel-bound. The total polygon count of meshes it works on decides if the pass is geometry-bound. There's also a dependency on memory amount and bandwith.

The __base pass__ is an example of a pass affected by all three factors. It takes visible 3D models (geometry) and renders them with full materials (pixel count), including textures (memory). Then it writes the final information into a resolution-dependent G-Buffer (so it's memory bandwith again).

If some passes take in just the G-Buffer, but not any 3D meshes - like post process effects do - then obviously they will be only pixel-bound. An increase in game's rendering resolution will directly affect their cost. On the other hand, it means that the changes in the amount of 3D meshes mean nothing to post process passes. For example, Unreal's ambient occlusion is a post process operation. It uses the hierarhical Z-buffer and some content from the G-Buffer, for example normals. By understanding what this pass requires, we know where to look for optimization opportunities -- in AO's settings and resolution adjustments, not in the scene's content.

{% include figure image_path="/assets/images/passes_gbuffer_all.jpg" alt="" caption="__Figure:__ The final image and various components of the G-Buffer which were used to render it" %}

# Using information from this chapter

The major part of this chapter is a guide to every significant rendering pass in Unreal. You don't have to read the entire thing. Treat it more as a handbook, which you use to understand the output of the ["Stat GPU" command]({{ site.baseurl }}{% link book/measuring-performance.md %}) or with [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}). Both of them show you the cost of each rendering pass. This allows you to precisely locate issues in a scene, like too many shadowed lights or too many translucent materials.

This chapter also provides extensive information about dependencies of each pass. It allows you to address the most probable sources of trouble first. Much of this information was gathered by reading the engine's source code and from practical tests. Be aware of potential mistakes here. Sources are provided in the footnotes, so you can fact-check it yourself.

## Optimizing

This chapter contains also a lot of optimization advice, to help you fix a bottleneck once you locate it. How would you approach optimization? In general, it's a good practice to _hide or disable_ the suspected cause of a problem. It will allow you to instantly check if your predictions were right, by looking at the change in miliseconds per frame.

There are several ways of doing this. You can hide a whole category of objects or effects. Press `[~]` in the viewport when __Playing in Editor__ and type `show`. This should propose a list of auto-completions, which you can browse with up and down arrows. Select a command and press `[Enter]`. Frequently used `show` commands include:
* `show DirectionalLights`
* `show DirectLighting`
* `show InstancedGrass`, `InstancedFoliage`
* `show InstancedStaticMeshes`,
* `show Landscape`
* `show PostProcessing`
* `show ScreenSpaceReflections`
* `show StaticMeshes`
* `show Translucency`
* `show VolumetricLightmap` (since UE 4.18)

{% include figure image_path="/assets/images/cmd_show.jpg" alt="" caption="__Figure:__ Entering a command in Unreal Engine" %}

Another way is to prevent individual objects from being displayed in game. Select one or multiple objects, then go to __Properties → Rendering__ and check __Actor Hidden in Game__. You can do it also when playing the game in editor. Press `[F8]` to __Eject__ from a running game, select objects in __Outliner__ instead of the viewport and change the setting. Then return to the game by pressing `F8` in the viewport again.

# Guide to passes and categories

Below begins an extensive description of almost all rendering passes that you can find in a profiler. Every position in the list is laid out in the following format:

* _Responsible for_ ...
* _Cost affected by_ ...
* Role of the pass
* Optimization advice

Some passes are much more important or customizable than others. Many of them react to changes in the scene, while others -- most notably post processes -- stay dependent on the resolution only. That's why the amount of information dedicated to each category varies greatly.

Don't feel forced to read the entire chapter at once. Jump straight to the pass you're interested in :)

* [Base pass]({{ site.baseurl }}{% link book/profiling/passes-base.md %})
* Geometry passes
    * [PrePass DDM_*]({{ site.baseurl }}{% link book/profiling/passes-geometry.md %}#prepass)
    * [HZB (Setup Mips)]({{ site.baseurl }}{% link book/profiling/passes-geometry.md %}#hzb-setup-mips)
    * [ParticleSimulation]({{ site.baseurl }}{% link book/profiling/passes-geometry.md %}#particlesimulation-particleinjection)
    * [ParticleInjection]({{ site.baseurl }}{% link book/profiling/passes-geometry.md %}#particlesimulation-particleinjection)
    * [RenderVelocities]({{ site.baseurl }}{% link book/profiling/passes-geometry.md %}#rendervelocities)
* Lighting passes
    * [LightCompositionTasks_PreLighting]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#lightcompositiontasks_prelighting)
    * [CompositionAfterLighting]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#compositionafterlighting)
    * [ComputeLightGrid]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#computelightgrid)
    * [Lights → NonShadowedLights]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#lights--nonshadowedlights)
    * [Lights → ShadowedLights]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#lights--shadowedlights)
    * [ShadowDepths]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#shadowdepths)
    * [ShadowProjection]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#shadowprojection)
    * [Translucency]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#translucency)
    * [Translucent Lighting]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#translucentlighting)
    * [Fog, ExponentialHeightFog]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#fog-exponentialheightfog)
    * [ReflectionEnvironment]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#reflectionenvironment)
    * [ScreenSpaceReflections]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#screenspacereflections)
* Post processing pass
    * [BokehDOFRecombine]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [TemporalAA]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [VelocityFlatten]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [MotionBlur]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [PostProcessEyeAdaptation]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [Tonemapper]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})
    * [PostProcessUpscale]({{ site.baseurl }}{% link book/profiling/passes-postprocess.md %})