---
title: "Base Pass"
excerpt: ""
permalink: "/book/profiling/passes-base/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

[← Back to all passes]({{ site.baseurl }}{% link book/profiling/passes.md %}){: .btn .btn--prev}

<div class="notice" markdown="1">
__Responsible for:__

* Rendering final attributes of __Opaque__ or __Masked__ materials to the G-Buffer
* Reading static lighting and saving it to the G-Buffer
* Applying DBuffer decals
* Applying fog
* Calculating final velocity (from packed 3D velocity)
* In forward renderer: dynamic lighting

__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
* {{ icon_complexity }} Shader complexity
* {{ icon_number }} Number of objects
* {{ icon_number }} Number of decals
* {{ icon_triangles }} Triangle count
</div>

{% include figure image_path="/assets/images/passes_gbuffer_all.jpg" alt="" caption="__Figure:__ _Top left:_ Final look of the scene, after lighting and post processes. _Top right:_ World-space normals in G-Buffer. _Bottom left:_ Base color (aka albedo) in G-Buffer. _Bottom right:_ Roughness in G-Buffer." %}

The base pass renders materials. Shaders serve many purposes, but by _materials_ I specifically mean the part that's created in the material editor. In foward rendering mode the base pass also calculates and applies lighting. On the contrary, deferred rendering (the default mode in UE4) saves only the resulting material properties: final roughness, base color, metalness and so on into the G-Buffer ("geometry buffer"). Then it leaves the lighting calculation for another pass (that's why it's called _deferred_). This material data is also used later by various other passes, most notably by screen-space techniques: dynamic reflections and ambient occlusion.

But that's just a part of this pass' responsibilities. It reads static lighting -- from lightmaps, indirect lighting caches -- and applies it, by blending it immediately with material's color.

The base pass also applies DBuffer decals. The decals in view are projected onto objects, their shaders are run and the results blended with the contents of the G-Buffer. Fog is also mixed in during this pass.

__Optimization__

Shader complexity is obviously the most important factor here. You can check it roughly using the __Shader Complexity__ view mode. It's also useful to look at the number displayed in the __Stats__ window in __Material Editor__ -- just be sure to check this number after you press __Apply__. The "good" value of shader instructions depends on the rendering resolution of your game and the area covered by the material on screen. To test how much the performance is affected by a shader, replace it temporarily with a basic material and compare miliseconds spent in the base pass.

Watch out for memory bandwith. Textures used by materials, as well as lightmaps, need to be read every frame. Even if you re-use textures a lot (which is a good thing), it only helps to save on the GPU memory and on reading from the CPU RAM. The data still needs to be transferred to the local cache and then to shader cores. A dedicated chapter explains the [memory-related costs]({{ site.baseurl }}{% link book/pipelines/memory.md %}).

Increasing rendering resolution directly affects the weight of the G-Buffer and other full-screen textures. The memory size of the G-Buffer can be read with the command `stat RHI`. You can temporarily scale the rendering resolution with `r.ScreenPercentage XX`, e.g. `20` or `150`. This will allow you to see how much additional VRAM is needed for a given resolution.

Decals are an oft-overlooked source of trouble. This pass can take a serious hit when there are too many decals in view. Keep their material complexity low as well, though this is rarely an issue with decals.

The costs (and optimization advice) which you'll normally see in __Lights__ passes apply here instead when using forward rendering. In addition to the general rules, keep light overlap (aka overdraw) in check. Forward renderers are genetically allergic to multiple lights hitting the same object.