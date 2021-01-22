---
layout: post
title: Nifty hidden shapes in a Codejam problem
date: 2018-10-13
---
I was trying to solve a problem called [_How Big Are the Pockets?_](https://codejam.withgoogle.com/codejam/contest/32002/dashboard) from Codejam.

Basically they want you to count the number of shaded squares in a figure like follows.
![](https://codejam.withgoogle.com/codejam/contest/images/?image=pockets02.png&p=24444&c=32002)

They give you as input a sequence of movements (right, left, or forward):
```
9
F 6 R 1 F 4 RFF 2 LFF 1
LFFFR 1 F 2 R 1 F 5
```

In order to debug my solution I made a [program](https://github.com/tuket/challenges/blob/master/codejam/08/3/a_view.cpp) that draws the line path and a trace of my solution.

I found some cool figures in the input test cases.

![3.png](/img/nifty_shapes_cj/3.png)

This one looks like a face:

![2.png](/img/nifty_shapes_cj/2.png)


