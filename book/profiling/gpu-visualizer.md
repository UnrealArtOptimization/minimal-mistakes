---
title: "Built-in GPU Visualizer"
excerpt: ""
permalink: "/book/profiling/gpu-visualizer/"
---

{% include toc icon="columns" title=page.title %}

{% include custom/wip-warning.md %}

# Preparations for profiling

How do we prepare for profiling? I suggest to place several fixed cameras across the level and check a typical in-game scenario. So, place the cameras where typically the player would enter and what the player will have in view. Let me show you this in Unreal. What I do, besides of placing the camera actor. You can enter it here: __Perspective → Camera Actor__. There should be a list of all your cameras. Now, as you can see, the aspect ratio is fixed. And I also want to ensure that the resolution is fixed. Because in the viewport, depending on the layout, I can have vastly different resolutions, which, as you know from part 1 and part 2, can heavily affect your scene's performance. So to prevent that, I press the arrow near the Play button, enter __Advanced Settings__ and in __Play in New Window__ I can specify the resolution for the game to run in. Now, close the window, press __Play__.

# GPU Visualizer

To run __GPU Visualizer__ press __Ctrl Shift , (comma)__. A new window should pop up. It's tool built into Unreal. It has very precise categories like: BeginOcclusionTests, ShadowDepths, RenderVelocities. And its most significant limitation is that it won't tell you the cost of specific meshes or specific typical lights. Only lights with shadows can be found in the GPU Visualizer. So again, I run it by pressing Ctrl Shift , in a game running in editor. You can also press it in the viewport but probably it will have some overhead from the editor's interface. Sometimes even quite heavy overhead.

So what the GPU Visualizer does is showing a breakdown of a single frame on the GPU. This is a single frame that was captured, sort of. And the frame took 24... slightly above 24 ms to render. Now, in the upper part you have breakdown of the scene as a bar. This is the Base Pass, ShadowDepths and the heaviest parts was Lights. Here I also have Translucency because this is an intentionally unoptimized scene I used here, to be able to show you most of the examples in terms of categories. And then, where I have ShadowedDepths for example, I can expand it further and further and here I have particular lights that cost me most time in this frame when it comes to rendering shadow depths. And the top of each category is the sum of this category.

# Profiling in a build

An important thing to know is that the GPU Visualizer only shows up when you start the game from inside the editor. If you don't have access to the editor, then not everything is lost, because the build, when you run it, won't show you the GPU Visualizer (because we don't have the Unreal Engine editor UI) but if you press Ctrl Shift comma, it saves the information to the log. So let me exit the game and then you enter: `your project name\Saved\Logs` and the current log. What you can see here is the same information and even more that you'd get from the GPU Visualizer, sorted chronologically. So, all these categories, for example: __BeginOcclusionTests__ or __ComputeLightGrid__, here they're sorted as they happen in the game. The engine begins with PrePass, then it computes the light grid, then the Base Pass happens and so on. So this is an interesting information in itself. You can also see the amount of vertices that were rendered by each pass.

The same information that we have in the log can be also found in the editor, in __Window → Developer Tools → Output Log__. Or, which is very convenient, in __Window → Project Launcher__, where you launch your project and control it from inside Unreal Engine.
