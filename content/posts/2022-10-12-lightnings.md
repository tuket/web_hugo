---
layout: post
published: true
title: "Lightnings!"
date: 2022-10-12
---

I have made a weapon in [Evil Robots](https://twitter.com/robots_evil) that casts lightnings to nearby enemies.

&NewLine;

{{<youtube P_t0vi-utS0>}}

In this post I wanted to show you how I approached it.

I made a demo that you can find in [github](https://github.com/tuket/lightning_demo).

Eventhough my game uses OpenGL, for this demo I have used the [sokol](https://github.com/floooh/sokol) library. I chose to use this library because it's a very simple and easy to use library, and it can be compiled to most platforms (including web). Besides I wanted an excuse to give it a try out and learn something new :).

<canvas id="canvas" class="emscripten" tabindex="-1" width=780 height=600></canvas>
<script async="" src="/wasm/lightnings/lightning_demo.js"></script>

## Generate a tree

The first step is to create a tree such at this one:

![tree](https://i.stack.imgur.com/iq2Uk.gif)

I got the idea from this [StackOverflow answer](https://gamedev.stackexchange.com/a/71433/25615)

The idea of this technique is to start with a line that joins the origin and destination points. That line is split more or less at the middle, and then we apply a random offset to it perpendicular to the direction of the line.

<img src="/img/lightnings/mid_split.svg" width="80%"/>

Do that **recursively** and you will get a jagged line.

<img src="/img/lightnings/jagged_line.svg" width="95%"/>

We would like to also add ramifications to the lightnings. Each time we make a split, we might also add a ramification with some probability.

<img src="/img/lightnings/tree.svg" width="95%"/>

We store each "joint" or node in a linear array as a position. Then we have another array of indices, which connects the nodes as a tree data structure. Along with the position, you can also store a **power** or energy. The energy at the root node is maximum. Each time there is a ramification, the power is split such that sum of powers is preserved. We will use this power later to give proportional thickness to the lines.

## Generate a triangle mesh

The last step consist of giving these lines some thickness.

<img src="/img/lightnings/demo_lightning.png" width="95%"/>

This part turned out to be the most difficult. Eventhough the core idea is quite simple, I found lots of problems in the corner cases. I managed to handle most of those special cases, but still, the results are not always 100% satifying with very thick lines.

The basic idea is to cast parallel lines according to the thickness of the segment, and compute the intersection points.

<img src="/img/lightnings/intersections.svg" width="100%"/>

After that it's just a matter of joining the points to form the triangles.

The tricky part it handling the corner cases. What happens when two lines are parallel (or almost)? The intersections points go to infinity or nans. I tried to handle those but I think it's not perfect. Here's the [relevant code](https://github.com/tuket/lightning_demo/blob/22f81558785065c2295002e2048fc4ca35b02aee/src/lightning.cpp#L116).

Onother detail to take into account is orienting the triangles towards the camera, in order to achive a billboarding effect.

