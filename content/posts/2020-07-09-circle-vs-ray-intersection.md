---
layout: post
published: true
date: 2020-07-09
title: Circle vs Ray intersection
---
A simple way to compute the intersection of a ray vs a circle.

<img src="/img/ray_circle_intersect.svg" alt="ray_circle_intersect.svg" width="100%"/>

Code [snippet](https://gist.github.com/tuket/7f604c2ea16f62af5a0d4778c80ac8c9 "snippet"):
```cpp
bool rayVsCircle(float& depth,
    glm::vec2 rayOrig, glm::vec2 rayDir,
    glm::vec2 circlePos, float circleRad)
{
    const float R = circleRad;
    const glm::vec2 op = circlePos - rayOrig;
    if(dot(op, op) < R*R) { // is rayOrigin inside the circle ?
        depth = 0;
        return true;
    }

    const float D = dot(rayDir, op);
    const float H2 = dot(op, op) - D*D;
    const float K2 = R*R - H2;

    if(K2 >= 0) {
        depth = D - sqrt(K2);
        if(depth > 0)
            return true;
    }
    return false;
}
```

Demo here: [https://github.com/tuket/demo_ray_vs_circle/blob/master/README.md](https://github.com/tuket/demo_ray_vs_circle/blob/master/README.md)


