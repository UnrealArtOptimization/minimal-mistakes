---
title: "Measuring Performance"
excerpt: ""
permalink: "/book/process/measuring-performance/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* The importance of using milliseconds in game development
* Converting ms to frames per second
* Basic tools for measuring performance
* Displaying statistics for each subsystem, like occlusion culling or particles
* Recording performance metrics and reading them with a profiler

If you prefer a video version of this lesson, you can [{{ icon_link }} watch it on YouTube](https://www.youtube.com/watch?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&v=SXLYy6D1y80).

{% capture tutorialvideo %}SXLYy6D1y80?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

## Are frames per second the ultimate metric?

In hardware reviews, competing products are compared by running benchmarks based on the same games. The results are expressed in frames per second. Graphics card A is supposed to outperform B, because it produces twice as many frames in the same period of time. Game C is more demanding than D, because the same hardware is able to calculate only 40 frames each second compared to 60 in the other title.

Frames per second seem to be the most popular metric for performance. So why do game developers measure in _milliseconds_ instead?

Here's a quote from an article about NVidia HairWorks[^eurogamer]:

> As you might expect from a technology that is said to render tens of thousands of tessellated strands of hair, the performance hit to the game is substantial - whether you are running an Nvidia or AMD graphics card. In our test case, the GTX 970 lost 24 per cent of its performance when HairWorks was enabled, dropping from an average of 51.9fps to 39.4fps. However, AMD suffers an even larger hit, losing around 47 per cent of its average frame-rate - its 49.6fps metric slashed to just 26.3fps.

The conclusion we can draw from these numbers is that enabling HairWorks-simulated hair in Witcher 3 can make the game lose up to 23 frames per second, depending on the card. But let's assume we have another feature, with an identical computational cost --  for example Depth of Field. We enable both hair and DoF at the same time. Can we calculate final fps from this information, without using milliseconds?

Let's try subtracting these costs from the original frame rate.

__49.6__ fps - __23__ fps (HW) - __23__ fps (DoF) = __3.6__ fps?
{: .notice .text-center}

Now what if there was a third feature of such high requirements, let's say, HBAO? What we should end up with is:

__49.6__ fps - __23__ fps (HW) - __23__ fps (DoF)- __23__ fps (HBAO) = <span style="color: red;">__-19.4__ fps?</span>
{: .notice .text-center}

This apparent paradox of negative frame rate arises from the fact that we didn't compare costs here, even if this was our intention. The actual cost of using these features was a _period of time_, not a number of frames. Our calculations were wrong.

## Calculating feature costs with milliseconds

Let's try milliseconds. Running the game without additional features allowed to complete about 50 frames in a second on average. How do we get the cost of each frame in milliseconds?

<div class="notice--info" markdown="1">
There are 1000 milliseconds (ms) in a second. If the game was able to finish 50 frames during 1000 ms, it means that an average frame took 1000/50 = _20 ms_ to render.

After enabling HairWorks, the game produced 26 frames per second. That shows us that the time cost of a single frame rose to 1000/26 = _38.5_ ms.
</div>

Now we're working with actual time periods, which can be added to one another. The feature added 18.5 ms to the time it took to produce a frame. After enabling three features, we end up with:

__20__ ms + __18.5__ ms (HW) + __18.5__ ms (DoF) + __18.5__ ms (HBAO) = __75.5__ ms
{: .notice .text-center}

Enabling two more features lead to _75.5 ms_ per frame. This means the game was running at about _13 fps_.

## Common fps rates in milliseconds

It's useful to memorize these three frame duration values - for 30, 60 and 90 fps:

<div class="notice text-center" markdown="1">
Time in milliseconds = 1000 ms / frames per second

* 30 FPS = 1000 / 30 = __33.33__ ms
* 60 FPS = 1000 / 60 = __16.67__ ms
* 90 FPS = 1000 / 90 = __11.11__ ms
</div>

These represent common expected refresh rates for various game categories: a "cinematic" adventure (30 fps), a fast-paced action game (60 fps) and a VR product (90 fps or more for reducing motion sickness). Each frame rate translates to a maximum time it should take to render a frame. If the software exceeds the time limit, it falls below the desired refresh rate (or even risks losing a V-Sync window).

## Breakdown of a frame

The values above show us that every game has a certain limit of time per frame. It should never be exceeded. Otherwise the player will experience fps drops.

Aiming for 30 fps leaves the engine with over 33 ms to complete all gameplay code, audio code and graphics rendering. The work on graphics one is split between CPU and GPU(s). The CPU has a role of a "manager", preparing data and sending commands to the GPU. The latter does the majority of calculations. The exact pipeline is explained in detail in ["GPU and rendering pipelines"]({{ site.baseurl }}{% link book/pipelines/index.md %}). They work (mostly) simultaneously, allowing the CPU to return to the gameplay code, while the GPU begins processing meshes and shaders. This means that the graphics card can use most of this 33 ms time window.

Following screenshots show the output of a tool built into Unreal, [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}). It breaks down the work done by the GPU on a single frame into specific sections, like shadows or transparency. Several scenes were measured, each optimized for different refresh rates: 30, 60 and 90 fps.

### The richness at 30 fps

The 30 fps scene can allow for many costly features to be used at once. It has a lot of transparency, heavy lighting (multiple dynamic light sources with big radius). In the right part of the colored bar, you can also see the post process using 3 ms.

{% include figure image_path="/assets/images/ch01/gpuvis_32ms.png" alt="" caption="__Figure:__ GPU Visualizer showing a 33 ms frame." %}

### Careful but fine at 60 fps

Now please take look at this graph. This scene has to render at 60 fps.

{% include figure image_path="/assets/images/ch01/gpuvis_16ms.png" alt="" caption="__Figure:__ GPU Visualizer showing a 16 ms frame." %}

Now the engine has to fit in half of the time - 16.67 ms. The post process time didn't change (because its cost is [pixel-bound]({{ site.baseurl }}{% link book/pipelines/pixel.md %})). So we still have a cost of 3 ms here -- but now it's a significantly bigger chunk of the time we can spare.

You can easily assume that by disabling the post process entirely we'll end up with 12 ms per frame. But it still won't be enough to land at the target of 11 ms -- the 90 fps recommended for a VR kit like Oculus or Vive.

### The harsh austerity of 90 fps

{% include figure image_path="/assets/images/ch01/gpuvis_11ms.png" alt="" caption="__Figure:__ GPU Visualizer showing an 11 ms frame." %}

This scene could run with a VR kit. As you can see, even if the cost of post process is static, now at 11 ms it's much more significant (compared to other passes). It was necessary to remove all translucent materials and particle effects. These cuts allowed to do full post processing and still keep a decent number of lights.

Each [rendering pass]({{ site.baseurl }}{% link book/profiling/passes.md %}) has a certain cost that depends on the scene's content. It's up to us to balance the content and settings to reach a desired frame rate. As we've seen in this chapter, measuring in milliseconds allows us to do math pretty easily. The time it takes to render a frame is just a sum of its ingredients.

## Console commands for diagnostics

Now that we know how milliseconds can be useful, we can proceed to actually measure something. There's a multitude of commands to enter in a game or the editor. They can show frame times, object and texture statistics, memory usage and even record the data into a file for later inspection.

### Disabling "Smooth Framerate"

Before we begin, to measure all this accurately, we have to stop Unreal from snapping the frame rate to stable values like 30, 45, 60 fps.

Go to __Project Settings → General Settings → Framerate__ → and disable __Smooth Frame Rate__. The __Smooth Frame Rate__ option tries to avoid momentary spikes in frame rate. It's good for end user's experience, so you can enable it again for the final game. But now let's disable it to get precise measurements.

### Entering commands

The basic console commands are entered by pressing `[~]` (tilde) in the viewport window or in a standalone development build.

The key `[~]` is located on the left from key `[1]` on keyboard. Use it and then type the command, for example: `stat fps` or `stat unitgraph`. The case of letters doesn't matter. Then press Enter.

To toggle the command (e.g. hide the information it shows), just use the command again.

### Stat FPS and Stat Unit

`Stat fps` shows us both the final number of fps and the time it took to render the last frame. It's the total time. But we still don't know if the cost was caused by the CPU or the GPU. As explained before, one has to wait for the other. Fast rendering on the graphics card won't help, if the CPU needs more time to finish the work on gameplay, drawing (managing the GPU) or physics.

We can get more specific info by using the `stat unit` command. The time of a last frame is shown as 4 numbers.

![image]({{ site.baseurl }}/assets/images/stat_fps.png) ![image]({{ site.baseurl }}/assets/images/stat_unit.png)

* __Frame__ is the same as __FPS__, the final cost.
* __Game__ is the work of the CPU on the gameplay code.
* __Draw__ is the work of the CPU on preparing data for the graphics card.
* __GPU__ is the raw time it took to render a frame on the graphics card.

Work on the next frame can't begin until the current frame is finished and displayed. If one of the components takes more time than others, causing __Frame__ to exceed the desired limit, it becomes a bottleneck[^bottle].

### Stat UnitGraph

`Stat unitgraph` prints the same information, but also starts displaying a graph. It's useful when moving or flying through the scene, because it makes it easier to locate heaviest places or situations. The drop in frame rate will be clearly visible.

### Stat GPU

`Stat GPU` splits the time of rendering a frame into specific passes. It's like a simplified, text version of [GPU Visualizer]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}) (the graphical tool invoked with  `Ctrl Shift ,`). The [chapter about passes]({{ site.baseurl }}{% link book/profiling/passes.md %}) explains the meaning of each pass, along with tips for optimization.

{% include figure image_path="/assets/images/stat_gpu.png" alt="" caption="__Figure:__ Output of `stat gpu`, showing the cost of rendering passes." %}

<div class="notice--warning" markdown="1">
_Warning:_ If you're getting an empty window from `stat gpu` on an older NVidia card, you'll have to use a workaround.

Open `C:\Program Files\Epic Games\your-version\Engine\Config\ConsoleVariables.ini`. Then go to the end of the file and add these 2 lines:

```text
r.NVIDIATimestampWorkaround=0
r.gpustatsenabled=1
```

Probably there's a reason why Epic Games disabled it on older cards. Maybe it can lead to stability issues. If you want to use the workaround, please remember that you're doing it at your own risk.
</div>

### Stat InitViews

It helps with measuring the cost and effectiveness of occlusion and frustum culling. Both of them are techniques used by game engines to improve performance. They dynamically hide meshes that would be invisible from the camera's position anyway. Contents of this stat output, as well as the culling itself, are discussed thoroughly in [{{ icon_link }} an article by Tim Hobson](http://timhobsonue4.snappages.com/culling-visibilityculling.htm).

* __View Visibility__, __Occlusion Cull__. Cost of performing culling. 
* __Processed primitives__. All objects that were considered, before culling.
* __Frustum Culled primitives__. Object that were out of camera's cone of view.
* __Occluded primitives__. Objects concealed from camera's view by other bigger objects.

### Stat RHI

RHI in `stat RHI` stands for Rendering Hardware Interface[^rhi]. This command displays several unique statistics:

* __Render target memory__. Shows the total weight of render targets like the GBuffer (which stores the final information about lighting and materials) or shadow maps. Buffers' size depends on game's rendering resolution, while shadows are controlled by shadow quality settings. It's useful to check this value periodically on systems with various amounts of video RAM, then adjust your project's quality presets accordingly.
* __Triangles drawn__. This is the final number of triangles. It's after frustum and occlusion culling. It may seem too big compared to your meshes' polycount. It's because the real number includes shadows (that "copy" meshes to draw shadow maps) and tessellation. In the editor it's also affected by selection.
* __DrawPrimitive calls__. Draw calls can be a serious bottleneck in DirectX 11 and OpenGL programs[^pcper]. They are the commands issued by the CPU for the GPU and, unfortunately, they have to be translated by the driver[^renderhelldrawcalls]. This line in `stat RHI` shows the amount of draw calls issued in current frame (excluding only the Slate UI). This is the total value, so besides geometry (usually the biggest number) it also includes decals, shadows, translucent lighting volumes, post processing and more.

### Other stat commands

These stat commands can also be very useful:

* `stat Foliage`. Stats related to _all instanced static meshes_ (not only foliage). Shows number of instances and total triangle count.
* `stat Landscape`. Number of triangles and draw calls used to render all landscape actors.
* `stat Particles`. Number of particle sprites, among other info.
* `stat LightRendering`. Cost of lighting, number of lights affecting translucency lighting grid, shadow-casting and unshadowed lights.
* `stat ShadowRendering`. Cost of shadow casting. Total memory used by shadow maps.

## Recording performance metrics

We can record all the metrics into a log. We'll be able to analyze it later on a graph. `Stat startfile` is the command that starts recording the data[^ue4docs]. A message pops up about log's duration in the upper left corner. To finish recording, enter `stat stopfile`. Now exit the game, go to __Window → Developer Tools → Session Frontend__ and load the latest file from `your project\Saved\Profiling\UnrealStats\`.

{% include figure image_path="/assets/images/session_frontend_profiler.png" alt="" caption="__Figure:__ Profiler tab in Session Frontend window. Panels not related to GPU profiling were minimized." %}{: .align-left}

This is a profile of the GPU and the CPU at the same time. If we're interested in graphics profiling, not gameplay, we should look for items in the "GPU" category in the sidebar. Double-clicking them will plot their values as new line on the graph. Horizontal lines shows targets for common fps values.

[Next chapter →]({{ site.baseurl }}{% link book/process/diagnosis.md %}){: .btn .btn--primary}

## Footnotes

Footnotes for this chapter:

[^bottle]: ["High-Performance Games: Addressing Performance Bottlenecks with DirectX*, GPUs, and CPUs", David Conger, intel.com](https://software.intel.com/en-us/articles/high-performance-games-addressing-performance-bottlenecks-with-directx-gpus-and-cpus)
[^eurogamer]: ["Does Nvidia HairWorks really "sabotage" AMD Witcher 3 performance?", Richard Leadbetter, eurogamer.net](http://www.eurogamer.net/articles/digitalfoundry-2015-does-nvidia-hairworks-really-sabotage-amd-performance)
[^ue4docs]: ["GPU Profiling", Unreal Engine documentation, docs.unrealengine.com](https://docs.unrealengine.com/latest/INT/Engine/Performance/GPU/)
[^rhi]: [Accepted answer to "What is RHI?", Tim Sweeney, answers.unrealengine.com](https://answers.unrealengine.com/questions/37056/what-is-rhi.html)
[^pcper]: ["What Exactly Is a Draw Call (and What Can It Do)?", Scott Michaud, pcper.com](https://www.pcper.com/reviews/Editorial/What-Exactly-Draw-Call-and-What-Can-It-Do)
[^renderhelldrawcalls]: [Subchapter 1. "Many Draw Calls" in "Render Hell – Book III", Simon Schreibt, simonschreibt.de](https://simonschreibt.de/gat/renderhell-book3/)