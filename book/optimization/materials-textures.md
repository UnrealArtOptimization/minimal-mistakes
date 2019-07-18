---
title: "Materials and Textures"
excerpt: ""
permalink: "/book/optimization/materials-textures/"
---

{% include custom/wip-warning.md %}

# Dependent texture reads

If the shader modifies UVs just to do tiling, you can safely move the calculation to the vertex shader. You can do it by putting a VertexInterpolator node after the operation.

# Breaking tiling

# Atlases, RGB packing

Good cases for packing and atlassing:

* Spatially related textures - maximizes the re-use of data among nearby assets, which helps to do less streaming
* Logically related textures - makes them easy to find, use and modify

Actually the best case may be to pack PBR channels of the same material. Additional effort is often low, as programs like Substance Painter and Designer can handle such packing almost by default. Both positive cases stated above are met, which means good performance and ease of use.

