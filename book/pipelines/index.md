---
title: "GPU and Rendering Pipelines"
excerpt: ""
permalink: "/book/pipelines/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* How to become best buddies with the GPU, working arm-in-arm towards a common goal
* How to stop pushing the content through a bottleneck and try alternative approaches
* Why you won't (likely) become best buddies with the graphics card driver

# Video

If you prefer a video version of this lesson, you can [{{ icon_link }} watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA).

{% capture tutorialvideo %}UZH4vZ0NDAw?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

# Anatomy of a frame

Every frame rendered by a real-time engine is a result of work done both by the _CPU_ and the _GPU (aka graphics card)_. In simplification, the CPU prepares the data and issues commands for the GPU to process.

The engine code can't control the GPU directly. Instead, it has to send the commands using an _API: application programming interface_. OpenGL, DirectX, Vulkan or Metal are examples of most popular APIs, though it may also be a game console's proprietary interface, like Playstation 4's GNM. In the case of PC and mobile, these API function calls must be further translated by a graphics _driver_. 

An important kind of an API function you should remember is the _draw call_. It's mentioned throughout this book countless times. A _draw call_ is a request to draw given meshes (precisely: batches of triangles) on the screen (or into a buffer), using specified _shaders_. I explain the idea in detail later in this chapter.

As for the data, the memory on the GPU is (in most cases) a physically separate piece of hardware - video RAM, aka _VRAM_. So there is also the need to move data between the main system RAM and VRAM.

What it all means is that the graphics unit has to wait. It can only start drawing after:
* the game engine's rendering code has finished (on the CPU)
* the resulting draw calls were translated into direct GPU code (by the driver, which runs on the CPU)
* the necessary data was pushed from RAM to VRAM (if not already present there)

When this is ready, the graphics card can start doing its job.

# The GPU pipeline

When the GPU receives a draw call, triangles are transformed by projection matrices to follow a correct perspective for the current point of view. This is a work of a _vertex shader_. Then the GPU begins to _rasterize_ them by executing a _fragment shader_. This distinction between different kinds of shaders is an important backbone of the entire GPU _pipeline_.

In a simplified way, a DirectX 11 / OpenGL 4 pipeline can be depicted as these steps:

__Vertex Shader → Tessellation → Geometry Shader → Rasterization → Fragment Shader__
{: .notice--info}

There's also a piece of hardware responsible for clipping triangles outside the screen. Only vertex shading, fragment shading and tessellation can be utilized from within Unreal's standard tools, so I'll leave looking for other steps' definition to the curious.

The order of these steps is pre-determined on a hardware level. Back in the days of PlayStation 1 and T&L-class PC graphics accelerators, the entire middle part of the pipeline was _fixed_. The only thing a programmer could control were the inputs, like meshes, texture and general surface parameters. Since at least GeForce 3 there was big push for _programmable_ pipelines. The hardware is now very flexible in terms of shading, with the majority of work being controlled in detail by custom shader programs.

## Rasterization

Rasterization is currently the prevalent method of real-time shading. For the sake of making the explanation easier, let me describe the steps of triangle rasterization and shading of their fragments (which means screen pixels) as a single "rasterization" concept.

In its idea, rasterization is closer to the rendering algorithms of early CG movies, rather than to ray tracing-based solutions that became dominant in Hollywood and architecture visualization since 2010s. While ray tracing and path tracing are simulating the path of light emitted from the camera and colliding with surfaces, rasterization only deals with drawing the immediately visible surface. If multiple triangles occupy a single pixel, all of them will be drawn, one after another. (However, game engines use a depth buffer to cut the unnecessary work in the case of fully opaque meshes).

The area belonging to a given triangle in screen space is filled pixel by pixel. The resulting _color_ (though please think of it as a value: a vector) is a direct output of the _fragment shader_ program. It doesn't any knowledge about other triangles in the scene. All it can work with are the data from the vertex shader - and textures, provided as its input.

Is it possible that a shader doesn't know about triangles? How can it perform convincing lighting then? In fact, every phenomenon in real-time rendering other than the direct lighting is only present because of additional _data_. That includes reflections of the environment, secondary light bounces (aka global illumination) and a thing that seems so foundational - shadows. This data is often generated or processed by multiple rendering _passes_: extra steps of rasterization, performed with different shaders. This is a job for the engine's own pipeline.

# Engine's rendering pipeline

The engine's pipeline includes both CPU and GPU-based operations. It includes the whole setup of multiple passes mentioned before. The [chapter about passes]({{ site.baseurl }}{% link book/profiling/passes.md %}) lists over 20 of them done by Unreal Engine every frame. The resulting information is provided as an input for subsequent passes' shaders.

There's much more to Unreal's pipeline, though. For example, occlusion takes care of discarding meshes which are not visible from camera's perspective. It's done an early stage of the pipeline, before they're even sent to be processed by shaders. Position and properties of light sources are provided to appropriate passes - depending on the choice between [deferred and forward]({{ site.baseurl }}{% link book/pipelines/forward-vs-deferred.md %}) shading.

Only after the last pass is finished the image can be displayed on the screen. If the vertical sync (VSync) is enabled, the image may also be delayed or discarded to achieve a required frame rate (for example 60 frames per second).

# Parallelism and pipeline

On the GPU we have concepts of parallelism and the pipeline. If something is parallel, it means that multiple cores, hundreds or thousands of cores, work on the same task simultaneously. So obviously, cores are best utilized when they are working on the same task. For example, the same big triangle, that is shaded with a single material. This is important because all cores assigned for a specific task on the GPU have to be exactly in the same place, for example: exactly the same moment in the shader.

Now, pipeline is like an assembly line. So the frame is pided into steps and for example, the vertex shader passes the data further to the pixel shader where the pixels are computed. and only then it's passed onto the screen. So this means that the pixel shader can't continue before it's fed with data from the vertex shader. Now, the simplified pipeline in OpenGL and DirectX 11 looks like that: We start with providing vertices to the vertex shader, it does it's transformations, then comes the tessallation phase, the geometry shader - which is the only one we have direct access to as an artist in Unreal Engine - and last comes the pixel shader which is responsible for materials, lighting and postprocess.

# Draw calls

{% include custom/wip-warning.md %}

An important thing, that can really affect your framerate are the draw calls. So draw calls are the commands sent by the CPU to control the GPU. So, for example, commands like: "change my mesh" or: "change my material" because if you want to draw a triangle set with a different material, different shader, you have to dispatch a command from the CPU first, which then goes through the driver, only then is translated and only then is submitted to the GPU. So having a lot of materials, a lot of different, separate objects, is a lot of work for the CPU. You can check your amount of draw calls by pressing `~` (tilde) and entering: STAT SceneRendering.

Are draw calls batched in UE4? No. There was a tweet about "fast path" https://twitter.com/joatski/status/679302537393119232