---
title: "Introduction"
excerpt: ""
permalink: "/book/"
---

{% include toc icon="columns" title=page.title %}

With popular commercial engines like Unreal Engine 4, understanding performance factors is at the same time hard and easy.

It's hard to intuitively pinpoint problems in a scene or project settings. That's because engine's renderer consists of dozens of systems. They are often dependent on each other, making it harder to isolate certain features for testing. By default, Unreal needs to go through 46 different [rendering passes](/book/profiling/lighting/) to process and draw a scene every frame. Usually, detailed explanation of these processes can only be found in engine programming documentation or by reading the source code. This makes the the knowledge obscure to people without background in engine programming, not just artists.

But understanding performance issues is also easy now. The engine provides us with detailed statistics about each rendering pass and subsystem. Immediately useful information, including a frames-per-second graph, can be displayed with basic console commands. Various built-in tools (like the [GPU Visualizer](/book/profiling/gpu-visualizer/)) allow us to make a performance analysis - the step called _profiling_. And if we ever a need a precise breakdown of a single frame, we can turn to powerful [standalone applications](/book/profiling/external/). This book walks you through the full spectrum of tools that can be used to analyze performance.

This book's goal is to help you like a good technical artist would - having a deep knowledge, but telling you the most relevant pieces first. The book should be a complete guide to graphics profiling without the need to dive into code. Of course, whenever you want to learn more or fact-check the text (please do), there are sources and links at the end of every chapter.

There is also a lot of advice for modelling and texturing from optimization point of view. In-engine features like lighting are covered as well. This is a universally useful knowledge, which I think makes you a better digital artist.

Please don't worry that you can't grasp all the information at once. I tried to divide the book into clear chapters and sub-chapters. You can probably read most of them separately if you're in a hurry. Each chapter's introduction is also a summary, which should make it easier to return to the book, when you're stuck on a specific problem in your project.

Enjoy the book - onwards to [Chapter 1](/book/measuring-performance/)!