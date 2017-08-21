---
title: "Measuring Performance"
excerpt: ""
---

{% include toc icon="columns" title=page.title %}

In this chapter we'll discover various ways to measure performance, expressed in miliseconds. Then we'll start looking for bottlenecks and other optimization issues.

## Video

If you prefer a video version of this lesson, you can watch it below.

*Note:* Every chapter of this book is extended compared with the original video. It's also regularly updated, while videos stay unchanged since their upload.
{: .notice--info}

{% capture tutorialvideo %}SXLYy6D1y80?list=PLF8ktr3i-U4A7vuQ6TXPr3f-bhmy6xM3S&amp;showinfo=1{% endcapture %}
{% include video id=tutorialvideo provider="youtube" %}

## Importance of miliseconds

So - why miliseconds? This is the most important question for this tutorial.

Basically it comes to this: you have to measure and compare your results for example, when checking if your changes affected the scene in a positive or negative way. And it's very hard to compare certain, specific features in fps. The fps are very useful for benchmarks they're very common, as you know, for testing if a game runs in 30 or 60 fps on a certain hardware.

So why it is quite useless for us? Well... It comes to how hard it is to express a cost of a certain feature in fps. Let's look at it that way: We have a light, that when we put it on the scene, we drop from 60 to 50 fps, for example. A very heavy, shadowed light. So if you assume that the cost of this certain light was "10 fps" then placing 5 another lights of this kind will leave us with... 0 fps? No. It certainly doesn't work that way.

{% include figure image_path="/assets/images/ch01/lights_cost.png" alt="" caption="*Fig. 1.1* Measuring the increasing cost of lights can't be done by subtracting fps." %}

<div class="notice--info" markdown="1">
Time in miliseconds = 1000 ms / frames per second

For example:

* 30 FPS = 1000 / 30 = 33.33 ms
* 60 FPS = 1000 / 60 = 16.67 ms
* 90 FPS = 1000 / 90 = 11.11 ms
</div>

We can see it leaves us with 33 ms to render a frame. If we exceed this number when doing calculations on the GPU or the CPU, we won't hit the target for 30 fps. In the same manner, 1000 ms / 60 fps leaves us with 16 ms to render a frame.

Now here, trying to achieve 30 fps, we have a scene that renders in 30 ms. It has a lot of transparency, a lot of heavy lighting and in the right part of the image you can see that the post process is using 3 miliseconds. It's also shown here, in the GPU Visualizer.

{% include figure image_path="/assets/images/ch01/gpuvis_32ms.png" alt="" caption="*Fig. 1.2* GPU Visualizer showing a 32 ms frame." %}

This bar is the time it takes to render the entire scene. The cost of rendering a single frame. Now please look at this graph. This is a scene that has to render in 60 fps.

{% include figure image_path="/assets/images/ch01/gpuvis_16ms.png" alt="" caption="*Fig. 1.3* GPU Visualizer showing a 16 ms frame." %}

The post process time didn't change. We still have a cost of 3 ms here - and here, but the cost of 3 ms is a significantly bigger chunk of the time we have to render the scene at twice the framerate. We have to fit in 15 ms total. So you can easily assume that by disabling the post process entirely we'll have 12 ms per frame, which is not enough to land at the target of 11 ms, that is 90 fps, for example for a VR kit like Oculus or Vive.

{% include figure image_path="/assets/images/ch01/gpuvis_11ms.png" alt="" caption="*Fig. 1.3* GPU Visualizer showing an 11 ms frame." %}

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

Stat GPU splits the time of rendering a frame into specific passes. For example, base pass - which gathers information like roughness, base color and stuff into buffers. *Chapter 3* explains the meaning of each pass. Then we have translucency Then post processing is, of course, the... all the screen-space stuff that's happening, like bloom like motion blur but also screen-space reflections. Now we have dynamic lights and shadow projection, which is the shadow component of lighting.

<div class="notice--warning" markdown="1">
If you're getting empty information from `stat gpu` on an older NVidia card, you'll have to use a workaround. Open `C:\Program Files\Epic Games\your-version\Engine\Config\ConsoleVariables.ini`. Go to the end of the file and add these 2 lines:

```
r.NVIDIATimestampWorkaround=0
r.gpustatsenabled=1
```

But probably there's a reason why Epic Games disabled it on older cards. Maybe it can lead to some stability issues. So if you want to do it, please remember that you're doing it at your own risk.
</div>

You can also achieve the same kind of information by pressing: `Ctrl Shift ,` (comma). Then the GPU Visualizer pops up and you also have your scene split into passes. The tool is explained in detail in *Chapter 3*. The upside is you don't need to enable realtime GPU statistics in engine settings. It works on older configurations as well.

## Recording performance metrics

Now, we can record all this. I will disable STAT SceneRendering and go to STAT GPU again and I can tell STAT StartFile As you can see, a message pops up about StatFile duration in the upper left corner. It started recording data. Okay, it'll be enough and... STAT StopFile. Now I exit the game I go to Window → Developer Tools → Session Frontend → and I can load a file from your project, Saved, Profiling, and the latest file.

This is a profiling of the GPU and the CPU at the same time. We're interested in graphics profiling now, not gameplay so I will hide this part and this. And from here I enter the GPU and by double-clicking I enable the passes I'm interested in. And here on the graph you can see how the game performs in various moments. The horizontal lines show you targets for 60 fps, 30 fps, 20 fps. We are above the target for 60, unfortunately heading for 30 and these are your basic tools to just enter the game play it for a while and check how it performs in terms of raw fps and miliseconds values.

### Quote test

I will start with a quote from UE documentation page:
> "Lit translucency gets most of its lighting through a series of cascaded volume textures oriented around the view frustum".
