---
title: "Base pass"
excerpt: ""
permalink: "/book/profiling/passes-base/"
---

{% capture icon_settings %}<i class="fa fa-sliders fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_resolution %}<i class="fa fa-television fa-fw" style="color: #ab131c" aria-hidden="true"></i>{% endcapture %}
{% capture icon_number %}<i class="fa fa-tags fa-fw" style="color: #485cbe" aria-hidden="true"></i>{% endcapture %}
{% capture icon_triangles %}<i class="fa fa-cube fa-fw" style="color: #72b4e6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_area %}<i class="fa fa-dot-circle-o fa-fw" style="color: #42ad82" aria-hidden="true"></i>{% endcapture %}
{% capture icon_overdraw %}<i class="fa fa-database fa-fw" style="color: #ddbd3b" aria-hidden="true"></i>{% endcapture %}
{% capture icon_complexity %}<i class="fa fa-gears fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}

{% include toc icon="columns" title=page.title %}

# Base pass

{% include figure image_path="/assets/images/passes_gbuffer_all.jpg" alt="" caption="__Figure:__ _Top left:_ Final look of the scene, after lighting and post processes. _Top right:_ World-space normals in G-Buffer. _Bottom left:_ Base color (aka albedo) in G-Buffer. _Bottom right:_ Roughness in G-Buffer." %}

**Responsible for:**

* Rendering final attributes of __Opaque__ or __Masked__ materials to the G-Buffer
* Reading static lighting and saving it to the G-Buffer
* Applying DBuffer decals
* Applying fog
* Calculating final velocity (from packed 3D velocity)
* In forward renderer: dynamic lighting

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution
* {{ icon_complexity }} Shader complexity
* {{ icon_number }} Number of objects
* {{ icon_number }} Number of decals
* {{ icon_triangles }} Triangle count

The base pass is one of the most important ones. It takes your shaders -- the part that you created in the material editor -- then runs their code. In deferred rendering (the default mode), it also saves the resulting values: final roughness, base color, metalness and so on into the G-Buffer ("geometry buffer"). It actually is a container for multiple full-screen images, through [channel packing]({{ site.baseurl }}{% link book/pipelines/memory.md %}) helps to reduce the number. This data is used later by various passes, most notably by screen-space techniques: [deferred lighting]({{ site.baseurl }}{% link book/profiling/passes-lighting.md %}#lights--nonshadowedlights), dynamic reflections and ambient occlusion.

But that's just a part of this pass' responsibilities. It reads static lighting -- from lightmaps, indirect lighting caches -- and adds them into the G-Buffer too (blending it immediately with the base color).

The base pass also applies DBuffer decals. The decals in view are projected onto objects, their shaders are run and the results blended with the contents of the G-Buffer. Fog is also mixed in during this pass.

When using a forward renderer, the work on all dynamic lighting is also done here (instead of a separate __Lights__ pass). That's because the lighting is no longer _deferred_ until later. It's done on a shader level of every object, immediately after the material's atributes are calculated. This approach gets rid of the G-Buffer, saving memory and making several thing easier (especially anti-aliasing). Don't be suprised, though, that the cost of the base pass is significantly higher with forward.

**Optimization**

Shader complexity is obviously the most important factor here. You can check it roughly using the **Shader Complexity** view mode. It's also useful to look at the number displayed in the **Stats** window in **Material Editor** -- just be sure to check this number after you press **Apply**. The "good" value of shader instructions depends on the rendering resolution of your game and the area covered by the material on screen. To test how much the performance is affected by a shader, replace it temporarily with a basic material and compare miliseconds spent in the base pass.

Watch out for memory bandwith. Textures used by materials, as well as lightmaps, need to be read every frame. Even if you re-use textures a lot (which is a good thing), it only helps to save on the GPU memory and on reading from the CPU RAM. The data still needs to be transferred to the local cache and then to shader cores. A dedicated chapter explains the [memory-related costs]({{ site.baseurl }}{% link book/pipelines/memory.md %}).

Increasing rendering resolution directly affects the weight of the G-Buffer and other full-screen textures. The memory size of the G-Buffer can be read with the command `stat RHI`. You can temporarily scale the rendering resolution with `r.ScreenPercentage XX`, e.g. `20` or `150`. This will allow you to see how much additional VRAM is needed for a given resolution.

Decals are an oft-overlooked source of trouble. This pass can take a serious hit when there are too many decals in view. Keep their material complexity low as well, though this is rarely an issue with decals.

The costs (and optimization advice) which you'll normally see in **Lights** passes apply here instead when using forward rendering. In addition to the general rules, keep light overlap (aka overdraw) in check. Forward renderers are genetically allergic to multiple lights hitting the same object.