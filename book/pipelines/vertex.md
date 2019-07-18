---
title: "Vertex Costs"
excerpt: ""
permalink: "/book/pipelines/vertex/"
---

{% include toc icon="columns" title=page.title %}

{% include custom/wip-warning.md %}

# Triangle count

The nightmare of game artists since time immemorial -- the _triangle count_. Is it still a real problem? Does it remain an important factor? Most of the time, it's overshadowed by troubles with bandwidth and the cost of pixel shaders. However, keep in mind that it can explode after tessallation. When you don't control tessallation too well or assign too big values, you can end up with a lot of quad overdraw. And the amount of vertices matters for shadow casting. Because a lamp that has some meshes on its way needs to, sort of, make a copy of the mesh to draw a shadow map. Shadow casting basically multiplies the amount of vertices that fall into shadowed light's range.

# Splits

Now, this it a nice thing to keep in mind, but don't think about it too much - it's just good to know - that hard edges (or: "smooth groups") or splits on your UVs add to actual vertex count because the information has to be stored several times for a single vertex. But it only affects memory usage and disk space. Not the actual rendering performance.

# Shadows

{% include figure image_path="/assets/images/passes_shadow.jpg" alt="" caption="__Figure:__ Shadow-casting spot light" %}

So as said before, dynamic shadows can be a vertex-bound source of trouble. The heavier meshes the light has on its way, the bigger the performance cost. So to avoid it it's best to disable dynamic shadow wherever you can. Probably, for many of your lights you don't need dynamic shadows or you can change them to static. Then it's also a very good practice to keep the attenuation radius as small as you can. This will not only limit the cost of shadowing but also the cost to render the light in pixel shaders.
