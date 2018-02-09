---
title: "Geometry Passes"
excerpt: ""
permalink: "/book/profiling/passes-geometry/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

[← Back to all passes]({{ site.baseurl }}{% link book/profiling/passes.md %}){: .btn .btn--prev}

## PrePass

<div class="notice" markdown="1">
__Responsible for:__

* Early rendering of depth (Z) from non-translucent meshes

__Cost affected by:__

* {{ icon_triangles }} Triangle count of meshes with __Opaque__ materials
* {{ icon_overdraw }} Depending on __Early Z__ setting: Triangle count and complexity of __Masked__ materials
</div>

{% include figure image_path="/assets/images/passes_prepass.jpg" alt="" caption="__Figure:__ Depth buffer (aka z-buffer) - the result of pre-pass" %}

Description TODO. Its results are required by DBuffer decals. The pre-pass may also be used by occlusion culling.

Optimization: __Engine → Rendering → Optimizations → Early Z-pass__

## HZB (Setup Mips)

<div class="notice" markdown="1">
__Responsible for:__

* Generating the Hierarchical Z-Buffer

__Cost affected by:__

* {{ icon_resolution }} Rendering resolution
</div>

The HZB is used by an occlusion culling method[^hzbocclusion] and by screen-space techniques for ambient occlusion and reflections[^hzbuse].

Warning: Time may be extreme in editor, but don't worry. It's usually a false alarm, due to an bug in Unreal. Just check the value in an actual build. The bug is hopefully fixed since UE 4.18[^hzbbug].
{: .notice--warning}

## ParticleSimulation, ParticleInjection

<div class="notice" markdown="1">
__Responsible for:__

* Particle simulation on the GPU (only of __GPU Sprites__ particle type)

__Cost affected by:__

* {{ icon_number }} Number of particles spawned by __GPU Sprites__ emitters
* {{ icon_settings }} __Collision (Scene Depth)__ module
</div>

{% include figure image_path="/assets/images/passes_particles.jpg" alt="" caption="__Figure:__ GPU-simulated particles with screen-space collision enabled" %}

If you enabled __GPU Sprites__ in a particle emitter, then their physics simulation is done here. The cost depends on the number of particles spawned by such emitters. Enabling the __Collision__ module increases the complexity of the simulation. The collision of GPU sprites with the rest of the scene is tested against screen-space data (for example the Z-depth). This makes it faster than the traditional, CPU-based collision of particles against actual 3D meshes. Still, it's not entirely free -- and the cost is moved to the GPU.

__Optimization__

The bigger the number of particles that have to be simulated (especially collide), the bigger the cost. Remember that you can use particle level of detail (__LOD__) to limit the number of particles being spawned. You may get away with much smaller numbers if the emitter is viewed from far distance.

## RenderVelocities

<div class="notice" markdown="1">
__Responsible for:__

* Saving velocity of each vertex (used later by motion blur and temporal anti-aliasing)

__Cost affected by:__

* {{ icon_number }} Number of moving objects
* {{ icon_triangles }} Triangle count of moving objects
</div>

Takes the velocity of every moving vertex and saves it into the motion blur velocity buffer.

# Footnotes

[^hzbocclusion]: [Accepted answer to "How does object occlusion and culling work in UE4?", Tim Hobson, answers.unrealengine.com](https://answers.unrealengine.com/questions/312646/how-does-object-occlusion-and-culling-work-in-ue4.html)
[^hzbuse]: ["Screen Space Reflections in Killing Floor 2", Sakib Saikia, sakibsaikia.github.io](https://sakibsaikia.github.io/graphics/2016/12/25/Screen-Space-Reflection-in-Killing-Floor-2.html)
[^hzbbug]: [Bug UE-3448HZB, "Setup Mips taking considerable time in GPU Visualizer", issues.unrealengine.com](https://issues.unrealengine.com/issue/UE-33448)