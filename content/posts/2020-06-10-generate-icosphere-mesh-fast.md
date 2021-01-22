---
layout: post
title: Generate icosphere mesh fast
date: 2020-06-10
---
If you seach in the Internet for algorithms to generate icosphere meshes, most algorithms out there take an icosahedron and keep tessellating the triangular faces recursivelly ([link](https://www.researchgate.net/figure/Spherical-tessellation-grids-and-parent-daughter-triangle-relationships-The-initial_fig5_253252594 "link")).

<img src="https://www.researchgate.net/profile/Nathan_Simmons/publication/253252594/figure/fig5/AS:668851566043151@1536478048938/Spherical-tessellation-grids-and-parent-daughter-triangle-relationships-The-initial.png" alt="recursive icosphere tesellation" width="50%"/>

I have made an algorithm that outputs the result, in a couple of flat arrays(vertices and indices), in a single pass, so it's faster and very cache friendly. It also has the advantage that you can query the exact memory you need to allocate.

https://gist.github.com/tuket/32951040ca0a4b31395175ee3e075d46

```cpp
// example usage
int numVerts, numInds;
generateIcosphere(numVerts, numInds, nullptr, nullptr, 8);
vec3* verts = new vec3[numVerts];
int* inds = new int[numInds];
generateIcosphere(numVerts, numInds, &verts, &inds, 8);
```

![generate_icosphere.png](/img/generate_icosphere.png)
