---
layout: page
title: "Full-screen quad: easy and efficient trick"
date: 2021-03-19
---

The usual way would be to create a VBO with the geometry of a quad (2 triangles).

<img src="/img/fs_quad_trick/fs_quad.svg" alt="fs_quad.svg" width="50%"/>

However, there is a better way of doing it.
- Draw a single triangle that covers the whole screen (the triangle will extend outside of the screen).
- Don't use a VBO. Instead, use the vertex id (`gl_VertexID`) to figure out the position and texture coordinates.

<img src="/img/fs_quad_trick/fs_triangle.svg" alt="fs_quad.svg" width="60%"/>

The reason this is faster, is not because it's drawing less triangles. It's because, paradoxically, it's drawing less fragments!
The GPU generates fragments in tiles of 2x2 to compute partial derivatives. At the boundaries of primitives, there can be some fragment overdraw just to compute those devivatives, even though the fragment is discarded. When drawing two triangles, the diagonal in between will produce for sure some of those wasted fragments. More info here: https://stackoverflow.com/a/59739538/1754322.

<img src="/img/fs_quad_trick/fs_overdraw.svg" alt="fs_quad.svg" width="60%"/>

As an additional trick, we can avoid having to upload vertex data by using the vertex id in the vertex shader:

```glsl
out vec2 v_tc;

void main()
{
    const vec2 tc[3] = vec2[3](
        vec2(0, 0),
        vec2(2, 0),
        vec2(0, 2)
    );
    v_tc = tc[gl_VertexID];
    gl_Position =  vec4(2 * v_tc - 1 , 0, 1);   
}
```

And just draw it normally with:

```c
glDrawArrays(GL_TRIANGLES, 0, 3);
```

Important note: from OpenGL 3.3 the use VAOs is [mandatory](https://stackoverflow.com/questions/30057286/how-to-use-vbos-without-vaos-with-opengl-core-profile).
That means that, despite not needing VBOs, we would still have to create at least one VAO and bind it. No need to use a specific one though, any one that was previously bound will do.

