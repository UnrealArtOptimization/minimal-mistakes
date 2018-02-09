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

To display a frame, both calculations on the CPU and the GPU have to be finished. So all the game code has to be computed, all the pixels have to be shaded and so on, and only then we can dispatch a frame to the screen. That's why the cost of a frame, the time it takes to show a frame, is the bigger number of either CPU or the GPU. If the GPU finished first, then it would wait for the CPU to dispatch new commands for the GPU. While, if the GPU is taking too long, the CPU has to wait for it. If you press `~` and enter: STAT UNIT you'll be shown these values you can see here.

# Parallelism and pipeline

On the GPU we have concepts of parallelism and the pipeline. If something is parallel, it means that multiple cores, hundreds or thousands of cores, work on the same task simultaneously. So obviously, cores are best utilized when they are working on the same task. For example, the same big triangle, that is shaded with a single material. This is important because all cores assigned for a specific task on the GPU have to be exactly in the same place, for example: exactly the same moment in the shader.

Now, pipeline is like an assembly line. So the frame is pided into steps and for example, the vertex shader passes the data further to the pixel shader where the pixels are computed. and only then it's passed onto the screen. So this means that the pixel shader can't continue before it's fed with data from the vertex shader. Now, the simplified pipeline in OpenGL and DirectX 11 looks like that: We start with providing vertices to the vertex shader, it does it's transformations, then comes the tessallation phase, the geometry shader - which is the only one we have direct access to as an artist in Unreal Engine - and last comes the pixel shader which is responsible for materials, lighting and postprocess.

# Draw calls

An important thing, that can really affect your framerate are the draw calls. So draw calls are the commands sent by the CPU to control the GPU. So, for example, commands like: "change my mesh" or: "change my material" because if you want to draw a triangle set with a different material, different shader, you have to dispatch a command from the CPU first, which then goes through the driver, only then is translated and only then is submitted to the GPU. So having a lot of materials, a lot of different, separate objects, is a lot of work for the CPU. You can check your amount of draw calls by pressing `~` (tilde) and entering: STAT SceneRendering.

Are draw calls batched in UE4? No. There was a tweet about "fast path" https://twitter.com/joatski/status/679302537393119232