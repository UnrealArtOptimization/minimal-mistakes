---
title: "Forward vs Deferred Shading"
excerpt: ""
permalink: "/book/pipelines/forward-vs-deferred/"
---

{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* -

{% include custom/wip-warning.md %}

# Deferred

Deferred shading is the default method of rendering lights and materials in Unreal (the other being forward rendering). _Deferred_ means that the work is moved to a separate pass, instead of being done in each object's shaders. This kind of lighting waits for the [base pass]({{ site.baseurl }}{% link book/profiling/passes.md %}#basepass) to accumulate the information about opaque objects and their materials into a G-Buffer. Then it resolves the lighting in screen space, in a single pass.

This approach reduces the performance hit of having multiple overlapping light sources, which is a typical issue in forward rendering (though things are better in the "clustered" method, used for UE4's forward renderer). However, this doesn't mean that lights are free in deferred. Their contribution is just easier to predict and doesn't depend on the number of objects in the scene. The cost of a single light is directly dependent on the area the light covers in screen space.

Spot lights are usually the cheapest type to render, because their screen-space area comes from a cone, not a full sphere. Point lights are often unecessarily used by artists in interiors, where a spot light would be enough. Still, what matters the most is the range. Deferred rendering is great at dealing with a multitude of small lights. Sometimes it's even feasible to attach tiny light sources to particles like sparks.

# Forward

When using a forward renderer, all work on dynamic lighting is done in the [base pass]({{ site.baseurl }}{% link book/profiling/passes.md %}#basepass), instead of a separate __Lights__ pass. That's because the lighting is no longer _deferred_ until later. It's done immediately on a shader level of every object, right after final material's atributes are calculated. This approach gets rid of the G-Buffer, saving GPU memory and making several thing easier (especially anti-aliasing). Don't be suprised, though, that the cost of the base pass is significantly higher with forward.

