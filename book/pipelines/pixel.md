---
title: "Pixel Costs"
excerpt: ""
permalink: "/book/pipelines/pixel/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

{% include custom/wip-warning.md %}

# Is a scene pixel-bound?

Pixels are most probably the slowest part in your pipeline. The bigger the resolution, the more pixels we have to be shaded! Unsurprisingly, I think... So heavy lighting, which is done per pixel heavy shaders and also post process effects depend on the resolution of the screen. And given the current FullHD and 4K resolutions, this can really mean a lot. So how to check if we're pixel-bound? While running our game, we can press `~` and enter: r.ScreenPercentage (for example) 25 or r.SetRes 480x270, for example. Then if your framerate improved a lot, then it means you're pixel-bound. Not for example vertex- bound or memory-bound. The pixels are the problem.

# Translucency

The biggest common problem with shading pixels is translucency. Opaque is very cheap because only the mesh closest to the camera is being rendered. While, when you have a translucent object, you have to draw everything: the translucent object, the next one behind it sometimes even the next one and only then the rest of the scene. So it's a big cost. And of course translucent particles can be an unexpected but very heavy source of computation. You can use not only level of detail on the meshes -- you can also do level of detail on particle emitters. You can have very detailed, translucent particles when you're nearby but as you go further away from the emitter, you can replace them with some more crude particle systems with fewer particles, to minimize the impact on performance.

# Quad overdraw

{% include figure image_path="/assets/images/viewmode_quad_overdraw.jpg" alt="" caption="__Figure:__ Quad Overdraw view mode" %}

An unexpected source of performance issues that I thinks is not so well known among artists, while it should be, is the so called _quad overdraw_. This is the reason why small polygons waste GPU time. In this case, a "quad" means a block of 4 pixels (2 by 2). And most of the operations on the GPU when it comes to pixel shading are done on full quads or even bigger tiles, like 8x8. Not on single inpidual pixels. It's easier for the GPU, or sometimes necessary, to perform operations on bigger tiles and only then discard unnecessary pixels. So the discared pixels are basically wasted. As you can imagine, the smaller the triangle, or more thin the triangle, the bigger the problem.

So the triangle count itself is actually not a problem very often. Quite contrary to the popular opinion, the triangle count by itself doesn't matter It's much more important to avoid small and thin and long triangles. Use level of detail for that to control this. Try to keep polygons big and even in screen space. So watch from your camera, from your game, not just in the 3D package. And empty pixels in foliage are extra-waste. Because you have translucency or some overdraw. So clip like crazy. Spend more polygons, but keep closer to the actual shape of the plant to be drawn.
