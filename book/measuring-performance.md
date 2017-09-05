---
title: "Measuring Performance"
excerpt: ""
permalink: "/book/measuring-performance/"
---

{% include toc icon="columns" title=page.title %}

In this chapter we'll discover various ways to measure performance, expressed in miliseconds. Then we'll start looking for bottlenecks and other optimization issues.

## Video

If you prefer a video version of this lesson, you can watch it below.

_Note:_ Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

{% capture tutorialvideo %}SXLYy6D1y80?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

## Importance of miliseconds

In hardware reviews, competing products are compared by running benchmarks based on the same games. The results are expressed in frames per second. Graphics card A is supposed to outperform B, because it produces twice as many frames in the same period of time. Game C is more demanding than D, because the same hardware is able to calculate only 40 frames each second compared to 60 in the other title. Frames per second seem to be the most popular metric for performance.

So why do game developers measure in miliseconds instead?

Here's a quote from an article about NVidia HairWorks:

> As you might expect from a technology that is said to render tens of thousands of tessellated strands of hair, the performance hit to the game is substantial - whether you are running an Nvidia or AMD graphics card. In our test case, the GTX 970 lost 24 per cent of its performance when HairWorks was enabled, dropping from an average of 51.9fps to 39.4fps. However, AMD suffers an even larger hit, losing around 47 per cent of its average frame-rate - its 49.6fps metric slashed to just 26.3fps.

The conclusion we can draw from these numbers is that enabling HairWorks-simulated hair in Witcher 3 can make the game lose up to 23 frames per second, depending on the card. Now let's assume we have another feature, with an identical computational cost, for example Depth of Field. We enable both hair and DoF the same time. Can this information help us calculate final frames per second?

Let's try subtracting these costs from the original frame rate.

__49.6__ fps - __23__ fps (HW) - __23__ fps (DoF) = __3.6__ fps?
{: .notice .text-center}

Now what if there was a third feature of such high requirements, let's say, HBAO? What we should end up with is:

__49.6__ fps - __23__ fps (HW) - __23__ fps (DoF)- __23__ fps (HBAO) = <span style="color: red;">__-19.4__ fps?</span>
{: .notice .text-center}

This apparent paradox of negative frame rate arises from the fact that we didn't compare costs here, even if this was our intention. The actual cost of doing calculations for these features was a _period of time_, not some number of frames. Running the game without additional features allowed to complete about 50 frames in a second on average. How do we get the cost of each frame in miliseconds?

<div class="notice--info" markdown="1">
There are 1000 miliseconds (ms) in a second. If the game was able to finish 50 frames during 1000 ms, it means that an average frame took 1000/50 = _20 ms_ to render.

After enabling HairWorks, the game produced 26 frames per second. That shows us that the time cost of a single frame rose to 1000/26 = _38.5_ ms.
</div>

Now we're working with actual time periods, which can be added to one another. The feature added 18.5 ms to the time it took to produce a frame. Enabling two more features of the same kind will lead to 20 + 3 * 18.5 = _75.5 ms_ per frame. This is about _13 fps_.

## Cost of a frame

It's useful to memorize three common frame time values:

<div class="notice--info" markdown="1">
Time in miliseconds = 1000 ms / frames per second

* 30 FPS = 1000 / 30 = __33.33__ ms
* 60 FPS = 1000 / 60 = __16.67__ ms
* 90 FPS = 1000 / 90 = __11.11__ ms
</div>

These represent common expected refresh rates for various game categories: a "cinematic" adventure (30 fps), a fast-paced action game (60 fps) and a VR product (90 fps or more for reducing motion sickness). Each frame rate translates to a maximum time it should take to render a frame. If the software exceeds the value, it falls below the desired refresh rate (or even risks losing a V-Sync window).




Now here, trying to achieve 30 fps, we have a scene that renders in 30 ms. It has a lot of transparency, a lot of heavy lighting and in the right part of the image you can see that the post process is using 3 miliseconds. It's also shown here, in the GPU Visualizer.

{% include figure image_path="/assets/images/ch01/gpuvis_32ms.png" alt="" caption="__Fig. 1.2__ GPU Visualizer showing a 32 ms frame." %}

This bar is the time it takes to render the entire scene. The cost of rendering a single frame. Now please look at this graph. This is a scene that has to render in 60 fps.

{% include figure image_path="/assets/images/ch01/gpuvis_16ms.png" alt="" caption="__Fig. 1.3__ GPU Visualizer showing a 16 ms frame." %}

The post process time didn't change. We still have a cost of 3 ms here - and here, but the cost of 3 ms is a significantly bigger chunk of the time we have to render the scene at twice the framerate. We have to fit in 15 ms total. So you can easily assume that by disabling the post process entirely we'll have 12 ms per frame, which is not enough to land at the target of 11 ms, that is 90 fps, for example for a VR kit like Oculus or Vive.

{% include figure image_path="/assets/images/ch01/gpuvis_11ms.png" alt="" caption="__Fig. 1.4__ GPU Visualizer showing an 11 ms frame." %}

As you can see, even if the cost of post process is static, now when rendering for a VR kit it's much more significant, compared to other things. So we had to disable translucency at all, get rid of all particles and stuff which left us with place to do this post process and to have still a medium number of lights.

Using miliseconds instead of raw fps can allow us to break the entire scene into separate passes and compare their cost not just look at the final result, the final number of fps.
{: .notice--info}

## Console commands for diagnostics

### Disabling "Smooth Framerate"

Before we begin, to measure all this accurately, we have to stop Unreal from snapping the frame rate to stable values like 30, 45, 60 fps. Enter Project Settings → General Settings → Framerate → and disable Smooth Frame Rate. "Smooth Frame Rate" tries to avoid momentary spikes in frame rate. By disabling it, we'll be getting precise measurements.

### Entering commands

The basic console commands are entered by pressing `[~]` (tilde) in your game. The key `[~]` is located on the left from key `[1]` on keyboard. Use it and then type the command, for example: `stat fps`, `stat unitgraph`. The case of letters doesn't matter. Then press Enter.

### Stat FPS and Stat Unit(Graph)

`Stat fps` shows us both the final number of fps we achieved and the time it takes to render a frame. The total time. But we don't know yet, if the cost is caused by the CPU or the GPU? Because one will have to wait for another. The GPU will have to wait for the CPU, if the CPU needs more time to finish a frame. The gameplay code, drawing code and stuff.

So by using `stat unit` or `stat unitgraph`, we can divide this information into 4 specific parts.
* "Frame" is the same as "FPS", the final cost.
* "Game" is the work of the CPU on the gameplay code.
* "Draw" is the work of the CPU on preparing data for the GPU.
* "GPU" is the raw time it took to render a frame on the GPU.

### Stat GPU

Stat GPU splits the time of rendering a frame into specific passes. For example, base pass - which gathers information like roughness, base color and stuff into buffers. The [chapter about passes]({{ site.baseurl }}{% link book/profiling/passes.md %}) explains the meaning of each one of them. Then we have translucency Then post processing is, of course, the... all the screen-space stuff that's happening, like bloom like motion blur but also screen-space reflections. Now we have dynamic lights and shadow projection, which is the shadow component of lighting.

<div class="notice--warning" markdown="1">
If you're getting empty information from `stat gpu` on an older NVidia card, you'll have to use a workaround. Open `C:\Program Files\Epic Games\your-version\Engine\Config\ConsoleVariables.ini`. Go to the end of the file and add these 2 lines:

```
r.NVIDIATimestampWorkaround=0
r.gpustatsenabled=1
```

But probably there's a reason why Epic Games disabled it on older cards. Maybe it can lead to some stability issues. So if you want to do it, please remember that you're doing it at your own risk.
</div>

You can also achieve the same kind of information by pressing: `Ctrl Shift ,` (comma). Then the GPU Visualizer pops up and you also have your scene split into passes. The tool is explained in detail in [another section of this book]({{ site.baseurl }}{% link book/profiling/gpu-visualizer.md %}). The upside is you don't need to enable realtime GPU statistics in engine settings. It works on older configurations as well.

## Recording performance metrics

We can record all the metrics into a log. `stat startfile` is the command that starts recording the data[^ue4docs]. A message pops up about StatFile duration in the upper left corner. To finish recording, enter `stat stopfile`. Now exit the game, go to Window → Developer Tools → Session Frontend → and load a file from `your project\Saved\Profiling\` and the latest file.

This is a profile of the GPU and the CPU at the same time. We're interested in graphics profiling now, not gameplay so I will hide this part and this. And from here I enter the GPU and by double-clicking I enable the passes I'm interested in. And here on the graph you can see how the game performs in various moments. The horizontal lines show you targets for 60 fps, 30 fps, 20 fps. We are above the target for 60, unfortunately heading for 30 and these are your basic tools to just enter the game play it for a while and check how it performs in terms of raw fps and miliseconds values.

### Quote test

I will start with a quote from UE documentation page:
> "Lit translucency gets most of its lighting through a series of cascaded volume textures oriented around the view frustum".

## Footnotes

[^ue4docs]: ["GPU Profiling", Unreal Engine documentation](https://docs.unrealengine.com/latest/INT/Engine/Performance/GPU/)
