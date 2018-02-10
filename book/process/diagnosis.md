---
title: "The Diagnostic Process"
excerpt: ""
permalink: "/book/process/diagnosis/"
---

{% include custom/inline-icons.md %}
{% include toc icon="columns" title=page.title %}

In this chapter you'll learn about:

* Being your project's doctor
* Smashing down nice theories
* Hiding objects - the crucial step of performance analysis

# Take a scientific approach

You and your team just gave the scene another test run. The frame rate was a random mess. An average frame time have rarely stayed below your desired limit of 16 ms for a competitive multiplayer game. You noticed, however, that fragments of the playthrough where you had travelled through a dark underground maze performed much better than well-lit palace rooms. The worst case was a detailed golden throne hall.

You have a clue and some numbers that you scribbled in a notebook. Or maybe you're already past the chapter about measuring performance? Then you have some hard data to work with. What do you do?

* Search AnswerHub for _"ue4 big room huge frame drops"_?
* Hit people on forums with your burning question, "Why my detailed golden room is heavier than dark dungeon?"
* Go to YouTube, _"wtf is lighting performance"_?

Or you behave like a scientist. They have a tool that may come in handy -- the __scientific method__. It has much in common with an earlier, philosophical one, made famous by Francis Bacon. From an article about the [{{ icon_link }} inductive reasoning](http://changingminds.org/disciplines/argument/types_reasoning/induction.htm):

> Inference can be done in four stages:
> 
> Observation: collect facts, without bias.  
> Analysis: classify the facts, identifying patterns of regularity.  
> Inference: From the patterns, infer generalizations about the relations between the facts.  
> Confirmation: Testing the inference through further observation.

You _observed_ that time per frame is better in an underground maze than it is in a palace hall. Working without bias, you have to admit that you have no evidence yet about what factors mattered. OK, you're pretty sure it was the golden details and a multitude of lights that caused frame drops, but being certain is against inductive reasoning.

Knowing that, you can _analyze_ the facts and look for _patterns_. Don't care about the numbers, care about general clusters of numbers. The _majority_ of low fps was noticed in palace scenes, while the highest ones were seen in the maze. There were exceptions, but not so much to break the pattern. The underground hallways rendered quicker than palace rooms. You may now start to infer the reason. Was it because _palace rooms had more lights visible at once than the maze_? Or did they have _more varied materials, compared to bare dungeon walls_?

Now it's time to _confirm_ the predicted rules.

You play through the game again, this time with a specific goal in mind. You head straight for the palace and measure the metrics again. Then, back in the editor, you check the approximate cost of lighting in each place with the [Light Complexity view mode]({{ site.baseurl }}{% link book/profiling/view-modes.md %}#light-complexity). If both data corelate, it's very likely you've found an issue. Otherwise - back to start.

But we're not just philosophers, right? We're also mad scientists, examining our project. So let's add another, last step, heavily restricted on humans but much welcome on game projects: _an experiment_. That's the secret that separates true eggheads from white-coated impostors. We test our assumptions.

# Testing your assumptions with simple methods

What's the best process to confirm or discard a prediction? There's a method that nothing can beat in simplicity. Basically, _hide or disable_ the suspected cause of a problem. 

Doing so will allow you to instantly check if your predictions were right. Display a [numeric metric of choice]({{ site.baseurl }}{% link book/process/measuring-performance.md %}), like `stat UnitGraph`, which shows miliseconds per frame changing in time. Do the desired change in the scene and observe its immediate effect on the framerate.

There are several ways of disabling objects or effects from rendering. The easiest way is to prevent objects from being displayed in game. Select one or multiple objects, then go to __Properties → Rendering__ and check __Actor Hidden in Game__. You can do it also when playing the game in editor. Press `[F8]` to __Eject__ from a running game, select objects in __Outliner__ instead of the viewport and change the setting. Then return to the game by pressing `F8` in the viewport again.

Please remember that sometimes disabling an object can lead to a drop in performance, instead of improvement. This is because some meshes act as _occluders_ of objects behind them. When hiding a wall or a ceiling, you may reveal the whole part of the scene behind them.
{: .notice--info}

You can also hide a whole category at once. Press `[~]` in the viewport when __Playing in Editor__ and type `show`. This should propose a list of auto-completions, which you can browse with up and down arrows. Select a command and press `[Enter]`. Frequently used `show` commands include:
* `show DirectionalLights`
* `show DirectLighting`
* `show InstancedGrass`, `InstancedFoliage`
* `show InstancedStaticMeshes`,
* `show Landscape`
* `show PostProcessing`
* `show ScreenSpaceReflections`
* `show StaticMeshes`
* `show Translucency`
* `show VolumetricLightmap` (since UE 4.18)

{% include figure image_path="/assets/images/cmd_show.jpg" alt="" caption="__Figure:__ Entering a command in Unreal Engine" %}

## Scalability Settings

The __Settings__ menu in viewport's toolbar provides access to yet another tool: __Engine Scalability Settings__. The menu contains two options worth playing with when figuring out performance issues: __Post Processing__ and __Shadows__. Possible values range from __Low__ to __Cinematic__. Some of the remaining sliders (like __Foliage__) do not do anything until you had manually set up LOD (level of detail) in static meshes and emitters. These two, however, directly influence the quality of post process effects (which include screen-space ambient occlusion) and the resolution of shadows. In reality, more factors affect the cost of these effects, but let's leave it for later chapters -- these buttons are good enough for quick tests.

The quality of the shadows and post processes can also be quickly adjusted in-game, via console commands:

* `sg.ShadowQuality X` (where `X` is a number between 0 and 4)
* `sg.PostProcessQuality X` (where `X` is a number between 0 and 4)

We can disable them completely by using a value of `0`.

# Next steps

Now you're well equipped to locate general sources of trouble in your project. Further in the book you'll learn to use profiling, view modes and other tools. You'll be able to add them to _analysis_ and _confirmation_ parts of the process. So what's the nearest step? It's learning what's going on _between the engine and the hardware_. This is essential to understanding precise info that profiling provides. And then, to successfully proceed with optimization.

Luickily, the next chapter is all about Unreal's and GPU pipelines!

[Next chapter →]({{ site.baseurl }}{% link book/pipelines/index.md %}){: .btn .btn--primary}