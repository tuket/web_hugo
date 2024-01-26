---
layout: page
title: Partial custom mipmap levels in OpenGL
date: 2021-02-22
published: true
---

OpenGL allows to automatically create mipmaps for you with `glGenMipmap`.

But you can upload you own mipmaps using `glTexImage2D`.

```c
for(int lvl = 0; numMips; lvl++) {
    glTexImage2D(GL_TEXTURE_2D,
        lvl,
        GL_RGBA8,
        texture[lvl].w, texture[lvl].h, 0,
        GL_RGBA, GL_UNSIGNED_BYTE, texture[lvl].data
    );
}
```

However, I wanted to provide my own images only for the upper levels, and let OpenGL compute the rest.
This is possible to do because OpenGL will generate mipmaps starting from `GL_TEXTURE_BASE_LEVEL`.
So you can do it like this:

```c
// upload custom images
for(int i = 1; i < numCustomMips; i++) {
    glTexImage2D(GL_TEXTURE_2D,
        lvl,
        GL_RGBA8,
        texture[lvl].w, texture[lvl].h, 0,
        GL_RGBA, GL_UNSIGNED_BYTE, texture[lvl].data
    );
}
// set the base level to the smallest mip we provided
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_BASE_LEVEL, numCustomMips-1);
glGenerateMipmap(GL_TEXTURE_2D);
// restore base level
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_BASE_LEVEL, 0);
```

If you find some textures look tilted, it could be that your texture rows are not 4-byte aligned for higher mips. Try if doing this fixes your problem:

```c
glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
```
