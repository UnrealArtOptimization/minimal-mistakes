---
title: "Geometry passes"
excerpt: ""
permalink: "/book/profiling/passes-geometry/"
---

{% capture icon_settings %}<i class="fa fa-sliders fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_resolution %}<i class="fa fa-television fa-fw" style="color: #ab131c" aria-hidden="true"></i>{% endcapture %}
{% capture icon_number %}<i class="fa fa-tags fa-fw" style="color: #485cbe" aria-hidden="true"></i>{% endcapture %}
{% capture icon_triangles %}<i class="fa fa-cube fa-fw" style="color: #72b4e6" aria-hidden="true"></i>{% endcapture %}
{% capture icon_area %}<i class="fa fa-dot-circle-o fa-fw" style="color: #42ad82" aria-hidden="true"></i>{% endcapture %}
{% capture icon_overdraw %}<i class="fa fa-database fa-fw" style="color: #ddbd3b" aria-hidden="true"></i>{% endcapture %}
{% capture icon_complexity %}<i class="fa fa-gears fa-fw" style="color: #bb72d6" aria-hidden="true"></i>{% endcapture %}

{% include toc icon="columns" title=page.title %}

## PrePass

**Responsible for:**

* Early rendering of depth (Z) from non-translucent meshes

**Cost affected by:**

* {{ icon_triangles }} Triangle count of meshes with __Opaque__ materials
* {{ icon_overdraw }} Depending on __Early Z__ setting: Triangle count and complexity of __Masked__ materials

Description TODO. Its results are required by DBuffer decals. The pre-pass may also be used by occlusion culling.

__Engine → Rendering → Optimizations → Early Z-pass__

## HZB (Setup Mips)

**Responsible for:**

* Generating the Hierarchical Z-Buffer

**Cost affected by:**

* {{ icon_resolution }} Rendering resolution

The HZB is used by an occlusion culling method[^hzbocclusion] and by screen-space techniques for ambient occlusion and reflections[^hzbuse].

Warning: Time may be extreme in editor, but don't worry. It's usually a false alarm, due to an bug in Unreal. Just check the value in an actual build. The bug is hopefully fixed since UE 4.18[^hzbbug].
{: .notice--warning}

## ParticleSimulation, ParticleInjection

**Responsible for:**

* Particle simulation on the GPU (only of __GPU Sprites__ particle type)

**Cost affected by:**

* {{ icon_number }} Number of particles spawned by __GPU Sprites__ emitters
* {{ icon_settings }} __Collision (Scene Depth)__ module

If you enabled __GPU Sprites__ in a particle emitter, then their physics simulation is done here. The cost depends on the number of particles spawned by such emitters. Enabling the __Collision__ module increases the complexity of the simulation. The collision of GPU sprites with the rest of the scene is tested against screen-space data (for example the Z-depth). This makes it faster than the traditional, CPU-based collision of particles against actual 3D meshes. Still, it's not entirely free -- and the cost is moved to the GPU.

**Optimization**

The bigger the number of particles that have to be simulated (especially collide), the bigger the cost. Remember that you can use particle level of detail (__LOD__) to limit the number of particles being spawned. You may get away with much smaller numbers if the emitter is viewed from far distance.

## RenderVelocities

**Responsible for:**

* Saving velocity of each vertex (used later by motion blur and temporal anti-aliasing)

**Cost affected by:**

* {{ icon_number }} Number of moving objects
* {{ icon_triangles }} Triangle count of moving objects

Description TODO. Takes the velocity of every moving vertex and saves it into the motion blur velocity buffer.

# Footnotes

[^hzbocclusion]: [Accepted answer to "How does object occlusion and culling work in UE4?", Tim Hobson, answers.unrealengine.com](https://answers.unrealengine.com/questions/312646/how-does-object-occlusion-and-culling-work-in-ue4.html)
[^hzbuse]: ["Screen Space Reflections in Killing Floor 2", Sakib Saikia, sakibsaikia.github.io](https://sakibsaikia.github.io/graphics/2016/12/25/Screen-Space-Reflection-in-Killing-Floor-2.html)
[^hzbbug]: [Bug UE-3448HZB, "Setup Mips taking considerable time in GPU Visualizer", issues.unrealengine.com](https://issues.unrealengine.com/issue/UE-33448)