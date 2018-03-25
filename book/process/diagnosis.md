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

You and your team just gave the scene another test run. The frame rate was a random mess. An average frame time have rarely stayed below your desired limit of 16 ms for a competitive multiplayer game. You noticed, however, that fragments of the playthrough where you had travelled through a dark underground maze performed much better than well-lit palace rooms. The worst case was a spacious marble throne hall.

You have some clues but there's too many factors that seem to matter for the final outcome. Or maybe you're already past the chapter about measuring performance? Then you have some hard data to work with. What do you do?

I encourage you to behave like a scientist. They have a tool that may come in handy -- the __scientific method__. It has much in common with an earlier, philosophical one, as made famous by Francis Bacon. From an article about the [{{ icon_link }} inductive reasoning](http://changingminds.org/disciplines/argument/types_reasoning/induction.htm):

> Inference can be done in four stages:
> 
> Observation: collect facts, without bias.  
> Analysis: classify the facts, identifying patterns of regularity.  
> Inference: From the patterns, infer generalizations about the relations between the facts.  
> Confirmation: Testing the inference through further observation.

You _observed_ that time per frame is better in an underground maze than it is in a palace hall. Knowing that, you can _analyze_ the facts and look for _patterns_. Don't care about the numbers - look for general trends instead. At this stage, put aside most optimization advice you've heared.

## Infer theories from observations

The majority of low fps was noticed in palace scenes, while the highest ones were seen in the maze. There were exceptions, but not so much to break the pattern. The underground hallways rendered quicker than palace rooms. Was it because palace rooms had more _lights_ visible at once than the maze? Or did they have more varied _materials_, compared to the bare dungeon walls? This is the _inference_ stage, where you define (and write down) the potential rules that you discovered.

How does an inference process look like? First, I always check whether the frame rate is _CPU- or GPU-bound_ in the scene I'm examining. Use the tips from [Measuring Performance]({{ site.baseurl }}{% link book/process/measuring-performance.md %}) to check that. Then you can look for these aspects:

__CPU__:

* Moments or places that involve a lot of physics interactions
* Blueprints that perform a lot of instructions every frame (aka every tick)
* Animations, both skeletal and Blueprint-based
* Places with a lot of objects and different materials visible at once (it leads to more draw calls)
* Small static objects with __Use as Occluder__ enabled (occlusion culling is meant to reduce the work for the GPU by testing visibility, but in this case the gain is smaller than the cost of testing)

__Graphics card__:

* Places with big number of lights, especially intersecting each other
* Lights and heavy objects that cast shadows
* A lot of tiny objects that cast shadows
* Quality settings (try __Medium__ instead of __Epic__ and see what happens)
* Big number and complexity of materials visible at once
* Big number and size of different textures visible at once
* Emitters that spawn a lot of particles or use lighting/shadowing
* Vistas with foliage

If you're not sure how to test these aspects, don't worry and just skip them. You'll learn about everything from the lists above in the next chapters.

## Confirm the predictions

After you wrote down the potential sources of trouble, it's time to _confirm_ or discard the predictions. To check the approximate cost of lighting in each place, you can visit each area with the __Light Complexity__  [view mode]({{ site.baseurl }}{% link book/profiling/view-modes.md %}) turned on. To check the number of instructions each material requires, you'll use the __Shader Complexity__ mode. Later in the book you'll learn a wide spectrum of tools to make the observations precise -- _profiling_ with the __GPU Visualizer__ being the most important one.

If any of these shows an excessive usage of lights or shader instructions, you confirmed your assumptions. Otherwise - back to start.


# Experiment to test your assumptions

But we're not just observers, right? We're also mad scientists examining the project. So let's add another, last step, frequently restricted on humans but so much welcome on game projects: _an experiment_. That's the secret that separates true eggheads from white-coated impostors. We test our assumptions!

## Hide objects

What's the best process to confirm or discard a prediction? There's a method that can't be surpassed in simplicity. Basically, _hide or disable_ the suspected cause of a problem.

Doing so will allow you to instantly check if your predictions were right. Display a [numeric metric of choice]({{ site.baseurl }}{% link book/process/measuring-performance.md %}), like `stat UnitGraph` (which shows miliseconds per frame changing in time). Make the desired change in the scene and observe its immediate effect on the framerate.

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

## Control feature quality

The __Settings__ menu in viewport's toolbar provides access to yet another tool: __Engine Scalability Settings__. Possible values range from __Low__ to __Cinematic__. Some of the  sliders (like __Foliage__) do not do anything until you had manually set up LOD (level of detail) in static meshes and emitters. Others, like __Post Processing__ and __Shadows__, directly influence the quality of the features. These buttons aren't very sophisticated, but they're perfect for quick tests.

The quality of the shadows and post processes can also be quickly adjusted in-game via console commands:

* `sg.ShadowQuality X` (where `X` is a number between 0 and 4)
* `sg.PostProcessQuality X` (where `X` is a number between 0 and 4)

We can disable a feature completely by using a value of `0`.

# Next steps

Now you're well equipped to locate general sources of trouble in your project. Further in the book you'll learn to use profiling, view modes and other tools. You'll be able to add them to _analysis_ and _confirmation_ parts of the process.

So what's the next step? It's learning what's going on _between the engine and the hardware_. This is essential to understanding the precise info that profiling provides. I also urge you to play with your projects, as you'll learn a lot from your own experiments!

[Next chapter →]({{ site.baseurl }}{% link book/pipelines/index.md %}){: .btn .btn--primary}