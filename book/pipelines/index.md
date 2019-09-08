---
title: "GPU and Rendering Pipelines"
excerpt: ""
permalink: "/book/pipelines/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* How to become best buddies with the graphics card
* Why you won't (likely) become best buddies with the graphics driver
* How to overcome the inevitable bottlenecks
* What are the draw calls and how to minimize their number

# Video

If you prefer a video version of this lesson, you can [{{ icon_link }} watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=C3lumWdwHmA).

{% capture tutorialvideo %}UZH4vZ0NDAw?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

# The anatomy of a frame

Every frame rendered by a real-time engine is a result of work done both by the CPU and the GPU (aka the graphics card). In simplification, the CPU prepares the data and issues commands (_draw calls_) for the GPU to process.

The engine code can't control the GPU directly. Instead, it has to send the commands using an _API: application programming interface_. OpenGL, DirectX, Vulkan or Metal are examples of most popular APIs, though it may also be a game console's proprietary interface, like Playstation 4's GNM. In the case of PC and mobile, these API function calls must be further translated by a graphics _driver_. 

An important kind of an API function you should remember is the _draw call_. It's mentioned throughout this book countless times. A _draw call_ is a request to draw given meshes (precisely: batches of triangles) on the screen (or into a buffer), using specified _shaders_. I explain the idea in detail later in this chapter.

As for the data, the memory on the GPU is (in most cases) a physically separate piece of hardware - video RAM, aka _VRAM_. So there is also the need to move data between the main system RAM and VRAM.

What it all means is that the graphics unit has to wait. It can only start drawing after:
* the game engine's rendering code has finished (on the CPU)
* the resulting draw calls were translated into direct GPU code (by the driver, which runs on the CPU)
* the necessary data was pushed from RAM to VRAM (if not already present there)

When this is ready, the graphics card can start doing its job.

# Draw calls

The goal of the graphics code is to draw what was requested. In an ideal situation, polygons would be transformed, shaded and displayed on the screen. Things are nowhere that simple, though, especially on the PC and mobile. While it's true that the GPU handles the majority of the graphics-related computations, it still needs to be controlled by the CPU. The commands what to draw -- aka the _draw calls_ -- are submitted by the engine code. Even the most powerful graphics card can be bottlenecked by an overworked central unit. To make things worse, draw calls (before Vulkan and DX12) had to be submitted through the graphics card driver -- itself a piece of software, adding another heavy layer of indirection.

For people experienced with object-oriented code, it may be surprising that the classic graphics APIs hold just a single global state of what to render. To render another mesh or to switch the material means to issue a draw call, changing the globally binded resource. It includes situations like:

* A different mesh is to be rendered
* A different shader (material) is to be applied
* Parameters for the shader change
* A switch to another [rendering pass]({{ site.baseurl }}{% link book/profiling/passes.md %}) happens
* A full-screen (post-process) shader is applied
* A render target's content is to be read by the CPU (for example particle simulations)

All of these would typically require a draw call. A realistic amount of draw calls that a mid-range DX11-era PC could handle in 16 ms was between hundreds and thousands. The games usually feature dozens of thousands of meshes in the visible scene. In a physically-based pipeline, each of those meshes probably uses at least 3 textures. Dispatching this many commands would be lethal to performance.

To mitigate this issue, programmers have came up with a lot of clever solutions over the years. The most important is _batching_, or at least _sorting_, the requests. The point is to combine the calls if the mesh and all its parameters are identical. A building may consist of thousands of pieces, but probably a lot of those are just instances of the same mesh. If these meshes were merged into a single object, it would require only 1 draw call.

It's especially effective for the static parts of a scene, for objects that are guaranteed not to change or move. Such a process of scanning the scene for the candidates is called static batching. Unity is among the engines that employ this method. When pushed too far, though, it can negatively affect systems like culling (hiding objects out of sight), as now the engine is dealing with huge monolithic objects, instead of smaller pieces.

Unreal Engine devs have preferred a lighter variant -- just sorting the list of objects to be drawn by their mesh and shader parameters. This allows to render a multitude of objects before switching the parameters with another draw call. While not as effective in reducing the number of commands as batching, such approach retains more flexibility.

The effectiveness of both techniques depends on the number of unique meshes and materials in the scene. A building may consist of thousands of pieces. However, if the most of them would be just instances of the same reusable fragments -- the engine would be able to greatly reduce the number of necessary draw calls. In 3D modelling workflows such approach is called a _modular_ environment art.

You can check the amount of draw calls in the running game by pressing `~` key (tilde) and typing: `stat SceneRendering`.

<div class="notice--info" markdown="1">
Did I say instances? A technique called _GPU instancing_ is the ultimate form of batching. It allows to load a single mesh, then simply stamp copies with almost zero CPU-related overhead.

There are a lot of limitations though. Almost nothing can change between instances, except for the position, rotation and scale (what is exposed per instance depends on the engine). All copies have to use the same material and properties. It's still perfectly valid for a belt of asteroid, for example, as I show in [{{ icon_link }} my tutorial about instancing](https://www.youtube.com/watch?v=oMIbV2rQO4k).

It has been available as a manual solution in UE4, but since version 4.23, Unreal also tries to find the exact copies automatically. Then it converts them into instances if possible.
</div>

# Parallelism and pipeline

GPUs are massively parallel. It means they use hundreds or thousands of cores, working on the same task simultaneously. Compare that to the x86 CPU realm, where 4 to 16 cores is a standard. All cores assigned to the same wavefront (task) on the GPU will follow the same lines of code, synchronized. They will be just handling a different vertex/pixel. That's why branching is harder than on the CPU, making some algorithm inefficient or impossible. For dealing with graphics, however, such tiny yet abundant cores are a perfect match -- think of all the neighboring pixels using the same shader and textures.

The GPU is not a huge single workshop, though. It's rather a network of several dedicated factories -- pipeline _stages_. Those facilities cooperate, but one has to finish its production before the second one can process the payload further. The vertex shaders need to transform the polygons first, so that the fragment shaders can shade these triangles, computing the pixels' final color. This means that the pixel shader has to wait until it's fed with data.

In a simplified way, a DirectX 11 / OpenGL 4 pipeline can be depicted like that:

__Vertex Shader → Tessellation → Geometry Shader → Rasterization → Fragment Shader__
{: .notice--info}

When the GPU receives a draw call, triangles are transformed by projection matrices to follow a correct perspective for the current point of view. This is a work of a _vertex shader_. Then the GPU begins to _rasterize_ them by executing a _fragment shader_. This distinction between different kinds of shaders is an important backbone of the entire GPU _pipeline_.

There's also a piece of hardware responsible for clipping triangles outside the screen. Only vertex shading, fragment shading and tessellation can be utilized from within Unreal's standard tools, so I'll leave looking for other steps' definition to the curious.

The order of these steps is pre-determined on a hardware level. Back in the days of PlayStation 1 and T&L-class PC graphics accelerators, the entire middle part of the pipeline was _fixed_. The only thing a programmer could control were the inputs, like meshes, texture and general surface parameters. Since at least GeForce 3 there was big push for _programmable_ pipelines. The hardware is now very flexible in terms of shading, with the majority of work being controlled in detail by custom shader programs.

## Rasterization

Rasterization is currently the prevalent method of real-time shading. For the sake of making the explanation easier, let me describe the steps of triangle rasterization and shading of their fragments (which means screen pixels) as a single "rasterization" concept.

In its idea, rasterization is closer to the rendering algorithms of early CG movies, rather than to the path tracing-based solutions that became dominant in Hollywood and architecture visualization since 2010s. While ray tracing and path tracing are simulating the path of light emitted from the camera and colliding with surfaces, rasterization only deals with drawing the immediately visible surface. If multiple triangles occupy a single pixel, all of them will be drawn, one after another. (However, game engines use a depth buffer to cut the unnecessary work in the case of fully opaque meshes).

The area belonging to a given triangle in screen space is filled pixel by pixel. The resulting _color_ (though please think of it as a value: a vector) is a direct output of the _fragment shader_ program. It doesn't any knowledge about other triangles in the scene. All it can work with are the data from the vertex shader - and textures, provided as its input.

Is it possible that a shader doesn't know about triangles? How can it perform convincing lighting then? In fact, every phenomenon in real-time rendering other than the direct lighting is only present because of additional _data_. That includes reflections of the environment, secondary light bounces (aka global illumination) and a thing that seems so foundational - shadows. This data is often generated or processed by multiple rendering _passes_: extra steps of rasterization, performed with different shaders. This is a job for the engine's own pipeline.

# Engine's rendering pipeline

The engine's pipeline includes both CPU and GPU-based operations. It includes the whole setup of multiple passes mentioned before. The [chapter about passes]({{ site.baseurl }}{% link book/profiling/passes.md %}) lists over 20 of them done by Unreal Engine every frame. The resulting information is provided as an input for subsequent passes' shaders.

There's much more to Unreal's pipeline, though. For example, occlusion takes care of discarding meshes which are not visible from camera's perspective. It's done an early stage of the pipeline, before they're even sent to be processed by shaders. Position and properties of light sources are provided to appropriate passes - depending on the choice between [deferred and forward]({{ site.baseurl }}{% link book/pipelines/forward-vs-deferred.md %}) shading.

Only after the last pass is finished the image can be displayed on the screen. If the vertical sync (VSync) is enabled, the image may also be delayed or discarded to achieve a required frame rate (for example 60 frames per second).
