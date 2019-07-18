---
title: "Memory Costs"
excerpt: ""
permalink: "/book/pipelines/memory/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

{% include custom/wip-warning.md %}

# Bandwidth

What issues can we encounter when dealing with the memory? The first thing is that too many texture samples in a material use up the bandwidth. And the bandwith is the amount of data than can that can be transferred between memory and the actual cores that perform the calculations. So compression helps a lot. Compression is supported directly on the GPU, so don't disable it in your textures, unless you need it. And the so-called "texture packing" helps too. It helps you to save on the amount of texture samplers and to optimize when it comes to memory. It means that you take, for example, 3 grayscale textures like roughness, metalness and ambient occlusion and pack them into specific channels of a single RGB texture. So each one occupies only a single channel then you can store them like a single texture.

# Cache coherency

The GPU tries to keep last accessed area of a texture in the cache. So the cache is a very fast memory that is located very close to the actual cores that perform the computations. So instead of going all the way to the VRAM memory, it can fetch the data it needs from the nearby cache. So if you want the GPU to use the cache, which is very small, keep the UV continuous. I know that some clever shader tricks can, for example, jump over the UVs for some noisy effects or something like that, but use it sparingly. Keep it in mind, because this is not what the GPU expects and it can waste the cache. But normally, with standard art and materials you should be fine by default.

# Streaming

There is also the streaming cache. It's not related to the the cache on the GPU. It's a cache in normal RAM that is maintained by Unreal. This is like a storage of textures that can be loaded at once. So too many textures in a level, and rember that lightmaps are textures too, can fill the cache. Unreal will try to load the lowest possible resolution, the lowest resolution that's needed. For example, some far away object doesn't need the biggest resolution to be loaded. But after the cache is full, You can encounter the same behavior even on close objects, because it can't load any more textures. So press [~] and enter: STAT streaming when you such behavior in your scenes.

# Texture Statistics window

Another way to check how well you're going with the amount of textures used is to go to "Window" → "Statistics" and this window pops up. This windows pops up, where you can see all the textures - and you can sort them by the amount of memory used. For example, this material is 2k x 2k (2048 px) but the current usage - the usage that was really loaded into the streaming pool - is just 64 x 64. Probably it's used only by some small object I don't know which, so I can check it here. Okay, some asteroid is using this texture in a very low resolution. Now, if I want to suppress the resolution anyway I can click on the texture name. It shows up in the content browser. And open it. Then in the "Compression" tab, I can expand this setting and say, for example, "Maximum Texture Size": 256. As you can see, the resolution lowered and this is a very good method to control your resolution without going to the actual texture editor. You can change it anytime. "Max In-Game" is shown as 512.

