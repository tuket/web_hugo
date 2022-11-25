---
layout: post
published: true
title: "ImGui gamma correction in OpenGL and Vulkan"
date: 2022-11-24
---

I had been using ImGui with OpenGL for a long time. Never encountered any problem with gamma corection.

Recently, I have started using ImGui wih Vulkan and noticed that the colors looked brighter. I immediately suspected it was a gamma issue.

Googling how to solve my problem, I found lots of people with similar experiences([^prob0], [^prob1], [^prob2]).

[SKIP BLAHBLAH AND GO TO THE GOOD SOLUTION](#he-actual-good-fix)

### Dumb solution
One of the solutions people suggested is to create the swapchain images in linear RGB, instead of sRGB. That's obviously not a good solution, because our own application beneffits from using sRGB compression to avoid banding issues. But it does indeed fix the issue with ImGui so it confirms out suspicions about gamma correction being the culprit.

### Why does it work in OpenGL?

This made me wonder: why is it working in OpenGL but not in Vulkan though? If could find what's different, maybe I could go down to the root cause and fix it. Most of the ImGui code is shared. The difference must be in the backends. However, I wasn't able to find anything different in there.

Even shaders seem to do the same. Some people in the internet were suggesting to change the vertex shader to do a pow(vertexColor, 2.2). But why does Vulkan need to do it, while OpenGL seems to work fine?

### Facepalm

After lots of research, FACEPALM.

![facepalm](https://i.imgflip.com/7202vq.jpg)

I had enabled gamma correction in GLFW with:

```c
glfwWindowHint(GLFW_SRGB_CAPABLE, GLFW_TRUE);
```

But that's not enough! 

You also need to enable it with:

```c
glEnable(GL_FRAMEBUFFER_SRGB);
```

When you do so, then ImGui looks wrong, just like the problem we were having in Vulkan.

Now everything looks wrong :D

### The quick OpenGL fix

In most OpenGL applications that use gamma correction, they use this quick workaround:

```
glEnable(GL_FRAMEBUFFER_SRGB);
// DRAW MY STUFF

glDisable(GL_FRAMEBUFFER_SRGB);
ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
```

In this way, we render our stuff with gamma compression, and ImGui without. Everything *seems* to work.

This is because ImGui is not designed taking sRGB into account[^prob1].

Downsides of this approach:
- Even with this trick, you will encounter issues displaying sRGB textures.
- There is no equivalent API in Vulkan for this.

### Demo

In order to understand what's going on I made a simple demo.

This application draws some simple stuff and then an ImGui window on top.

1. Clear the screen with 50% grey
2. Draw a big rectangle with a shader that outputs 50% grey (hard-coded in the shader)
3. Draw a smaller rectangle that samples a single-pixel, 50% grey, sRGB texture.

![layout](https://i.stack.imgur.com/x0h30.png)

This is what it looks like when we don't disable sRGB just before rendering ImGui:

![demo_0](/img/imgui_gamma/gamma_test_0.png)

As you can see, 1 and 2 look the same. That means doing `gl_FragColor = vec4(0.5, 0.5, 0.5, 1)` in the shader is the same as doing `glClearColor(0.5, 0.5, 0.5, 1)`.

The 3rd case looks different. That is because, even though the value 0.5 is stored in the texture, since the texture is sRGB, **decompression** is automatically performed when **sampling** from it. So it would be equivalent to `gl_FragColor = vec4(srgbToLinear(vec3(0.5)), 0)`.

Also, since we have gamma correction in the FBO, **compression** is performed automatically when **writing** to the FBO. So the value is decompressed when reading from the texture, and the compressed again when writing to the FBO. So we end up with the original value of 0.5 stored in the FBO.

![diagram_0](/img/imgui_gamma/diagram_0.svg)

For comparison, this is the 1-pixel 50% grey SRGB texture (it's actully a png of 1 pixel, but it shows scaled):

<img src="/img/imgui_gamma/grey_05.png" width="20%"/>

And this is the test image:

![tent](/img/imgui_gamma/tent.jpg)

As you can see, both images look correct in the previous screenshot. It's just the rest of the UI that looks too bright: window border, buttons etc.

If we use the workaround of disabling FBO's gamma correction:

![demo_0](/img/imgui_gamma/gamma_test_1.png)

Everything looks fine... except for the images which look too dark.

## The actual good fix

The actual fix involves changing the vextex shader.

As we already saw, sRGB decompression happens automatically when we sample the texture.

However, this is not the case for vertex colors!

In the original ImGui shaders, you won't see any special treatment for the vertex color attribute. But the colors are actually in sRGB format.

If we **disable** gamma correction, it's not a problem. The vertex color is 0.5 -> shader reads 0.5 -> shader writes 0.5 -> FBO stores the 0.5 value.

If we **enable** gamma correction: the vertex color is 0.5 -> shaders reads 0.5 -> shader writes 0.5 -> the value is compressed to sRGB [0.5^(1/2.2)] then stored in the FBO.

A thumb rule we use in computer graphics is: 1) store colors in sRGB space, 2) in shaders, work with colors in linear space.

So the best solution is to <u>decompress the vertex color attributes in the vertex shader</u>.

It's super easy:

```diff
precision highp float;
layout (location = 0) in vec2 Position;
layout (location = 1) in vec2 UV;
layout (location = 2) in vec4 Color;
uniform mat4 ProjMtx;
out vec2 Frag_UV;
out vec4 Frag_Color;
void main()
{
    Frag_UV = UV;
-   Frag_Color = Color;
+   Frag_Color = vec4(pow(Color.rgb, vec3(2.2)), Color.a);
    gl_Position = ProjMtx * vec4(Position.xy,0,1);
}
```

ImGui has a bunch of shaders for different versions of GLSL, so you should change all of them.

![demo_0](/img/imgui_gamma/gamma_test_2.png)

**Everything looks fine now!** But use a color picker to verify it - human eye is not reliable (some greys in the previous picture look slightly different to me. But using the color picker I verified they are actually identical).

This simple solution works for Vulkan as well!

NOTE that doing `pow(color, 2.2)` for converting from sRGB to linear is not 100% accurate. [Here](https://gamedev.stackexchange.com/questions/92015/optimized-linear-to-srgb-glsl) you will find the correct convertion. However, `^2.2` is a good cheap approximation.

## Links
[^prob0]: https://stackoverflow.com/questions/69327444/the-data-rendered-of-imgui-within-window-get-a-lighter-color
[^prob1]: https://github.com/ocornut/imgui/issues/578
[^prob2]: https://www.reddit.com/r/vulkan/comments/s3x6hq/help_with_srgb_surface_format_and_imgui/