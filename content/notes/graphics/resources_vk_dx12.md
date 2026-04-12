---
layout: page
title: Resources in Vulkan vs DX12
date: 2026-04-10
toc: true
---


**Vulkan** and **DX12** are the first two modern explicit APIs that came to market. \
They have more or less the same functionality and, as HW provides new capabilities, the APIs add extensions to take advantage of them.

Now, despite both APIs can do more or less the same, the interfaces are quite different. \
Also, the differences in terminology can get confusing: sometimes they use different words to refer to the same thing, sometimes they use the same word to refer to different things, sometimes there is no exact equivalent.

<div class="row" align="middle">
  <img src="https://www.vulkan.org/user/themes/vulkan/images/logo/vulkan-logo.svg" width=300 style="vertical-align:middle">
  <img src="https://upload.wikimedia.org/wikipedia/commons/6/67/DirectX_12_Ultimate.png" width=200 style="vertical-align:middle">
</div>

I have been learning Vulkan for several years and, due to my work, now I've been recently exposed to DX12 too (although I'm a total noob in DX12 TBH). \
When you are learning something new, it's always helpful to find analogies and similarities to something you already know. \
For example, when I was reading about root signatures in DX12 I was having trouble understanding them until my brain clicked: hold on, this is actually very similar to `VkDescriptorSetLayout`! And then I was able to hold it in my brain more easily.

So, in this post, I wanted to lay out those similarities (and differences) I found between Vulkan and DX12 in relation to **resources**. \
I write this down to help others in the same situation: you already know one API; you want to learn the other. And also, since I'm very forgetful, I'm sure I will find this post helpful myself in the future :)

Disclaimer: \
I'm a noob in DX12, so please forgive me for any mistakes (and please let me know in the comments so I can fix them).

## Creating resources

The most basic resources are: buffers, images and samplers. \
Samplers are pretty similar in both APIs, so I will only talk about them briefly.

### Creating resources in Vulkan

In Vulkan, you have **VkBuffer** and **VkImage** as two separate objects. \
There are two functions for creating them: [vkCreateBuffer](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateBuffer.html) and [vkCreateImage](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateImage.html).

The creation of a VkBuffer or a VkImage doesn't assign memory to these resources. \
The memory needs to be assigned explicitly. We will cover this in chapter [Assigning Memory to Resources](#assigning-memory-to-resources).

### Creating resources in DX12
In DX12, `ID3D12Resource` is used for both buffers and textures. \
The same functions can be used to create buffers and images:
- [ID3D12Device::CreateCommittedResource](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource)
- [ID3D12Device::CreatePlacedResource](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createplacedresource)
- [ID3D12Device::CreateReservedResource](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createreservedresource)

As you can see, we don't have separate functions for creating buffers and images, like in Vulkan. \
In DX12, whether we are creating a buffer or an image is determined by the [D3D12_RESOURCE_DESC](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_desc). More precisely, `D3D12_RESOURCE_DESC::Dimension` is an enum ([D3D12_RESOURCE_DIMENSION](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_resource_dimension)):
```C
typedef enum D3D12_RESOURCE_DIMENSION {
  D3D12_RESOURCE_DIMENSION_UNKNOWN = 0,
  D3D12_RESOURCE_DIMENSION_BUFFER = 1,
  D3D12_RESOURCE_DIMENSION_TEXTURE1D = 2,
  D3D12_RESOURCE_DIMENSION_TEXTURE2D = 3,
  D3D12_RESOURCE_DIMENSION_TEXTURE3D = 4
} ;
```

Notice that in **DX12** the term "texture" is used instead of "image". \
In **Vulkan**, the word "texture" is used sometimes to refer to an image that is going to be sampled. But there is no formal definition of "texture" in the Vulkan spec. \
In **GLSL**, the keyword "texture" is a [built-in function](https://registry.khronos.org/OpenGL-Refpages/gl4/html/texture.xhtml) that allows sampling an image+sampler.

As we just saw, there are 3 different functions to create buffers and images in DX12: this has to do with the way memory is assigned to the resource. We will cover that in chapter [Assigning Memory to Resources](#assigning-memory-to-resources).

## Assigning Memory to Resources

Here there are important differences in terminology.

In **Vulkan**, the term used for a memory chunk allocation is "**memory**" and the handle type is `VkDeviceMemory`.\
In **DX12**, we call it "**heap**" and the handle type is `ID3D12Heap`.

Now, in Vulkan, the term "heap" means a different thing.\
In **Vulkan**, a **heap** refers to actual **physical** memory sources. The number of heaps and their properties depend on the particular system. Some of the most important properties of the different heaps are: whether the memory attached to CPU or GPU memory (could be both), whether it's visible by the CPU or the GPU (again could be both), the size, and the cache coherency model.

In **DX12**, there is no exact equivalent to what they call "heap" in Vulkan. The closest thing would be [D3D12_HEAP_PROPERTIES](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_heap_properties), which is passed as a parameter to [CreateCommittedResource](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource)`

### Assigning Memory to Resources - Vulkan
You need to query how much memory is needed by the resource, allocate a `VkDeviceMemory`, and assign a memory range to the resource with [vkBindBufferMemory](https://docs.vulkan.org/refpages/latest/refpages/source/vkBindBufferMemory.html) and [vkBindImageMemory](https://docs.vulkan.org/refpages/latest/refpages/source/vkBindImageMemory.html). \
Efficient implementations won't allocate a `VkDeviceMemory` for each resource... instead, they will allocate large chunks of memory and assign subranges of the chunk to multiple resources. As you can imagine, GPU memory management in Vulkan can be complicated, so people like to use the [VMA library](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator). VMA is a general-purpose GPU memory allocator for Vulkan; it should work pretty well for most applications, but you can always make your own allocation strategies to get better performance.

### Assigning Memory to Resources - DX12

In DX12, the memory used by a resource can be assigned automatically if you use `CreateCommittedResource`. \
In this regard, DX12 is much simpler than Vulkan, as you can just forget about memory management for resources.

However, if you want, you can go into *hardcore* mode and do manual memory management, just like in Vulkan (although it's recommended to start with the easy way and then optimize incrementally if needed). \
Instead of calling `CreateCommittedResource`, you would call [CreateHeap](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createheap), and then assign sub-ranges to your resources using [CreatePlacedResource](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createplacedresource). \
Read this tutorial if you want to learn more about manual memory management in DX12 (Ctrl+F: "Advanced Model"): https://logins.github.io/graphics/2020/07/31/DX12ResourceHandling.html \
There is the DX12 equivalent of VMA: https://gpuopen.com/d3d12-memory-allocator.


## Shaders

In this section we will see how resources are used in shaders.

Vulkan and DX12 don't recognize high-level shader code such as GLSL or HLSL; you need to provide them IR (intermediate representation) bytecode.\
**Vulkan** uses **SPIR-V** (Standard Portable Intermediate Representation - Vulkan).\
**DX12** uses **DXIL** (DirectX Intermediate Language).

| | Vulkan | DX12 |
|-|--------|------|
| High-Level Language| GLSL | HLSL
| Low-Level IR | SPIR-V | DXIL |

Even though GLSL and HLSL are respectively the official high-level languages for Vulkan and DX12, it's not mandatory to use them.

Both GLSL and HLSL can be compiled to SPIR-V.
- GLSL can be converted to SPIR-V using:
  - [glslang](https://github.com/KhronosGroup/glslang): it's the official compiler from Khronos. It comes with the Vulkan SDK.
  - [glslc](https://github.com/google/shaderc): it's a wrapper around glslang made by Google. `glslc` is the name of the CLI tool, and `shaderc` is the name of the library. They are also included in the Vulkan SDK. I prefer glslc over glslang as I find the interface easier to use, but both are perfectly valid.
- HLSL can be converted to SPIR-V using:
  - [DXC](https://github.com/microsoft/DirectXShaderCompiler): it's the official HLSL compiler from Microsoft. It comes with the Vulkan SDK. DXC can convert to both SPIR-V and DXIL.
  - You will see that glslang has a HLSL parser, but it's not complete. Just use DXC.

Having so many HW vendors, operating systems, and graphics APIs adds a lot of work for developers. Companies usually try to impose their own thing, and they want to be the ones that define "the standard", so we end up with many different competing standards.

<img src="https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fimgs.xkcd.com%2Fcomics%2Fstandards_2x.png&f=1&nofb=1&ipt=15ac6913ef4ee525444c310c196cb8e45dd6bf9458ece7878e4a7569dae51949" width=480>

In the realm of shaders, however, something wonderful (and surprising) happened: SPIR-V has become a true standard.

The first thing that contributed to this miracle is [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross). SPIRV-Cross is a tool/library made by Khronos that can convert SPIR-V to other languages such as GLSL, MSL, and HLSL.

SPIR-V contributed to the following scenario:
  - Take any high-level shader language you like: GLSL, HLSL, MSL, Slang
  - Compile it to SPIR-V
  - Convert the SPIR-V to anything!

The second thing that stablished SPIR-V is... when Microsoft anounced that they will be officially supporting SPIR-V in DX12: https://devblogs.microsoft.com/directx/directx-adopting-spir-v/

![spirv ecosystem](https://www.khronos.org/assets/uploads/apis/2024-spirv-language-ecosystem.jpg)

So now SPIR-V is the standard interchange format.

Ok, so SPIR-V is very cool, but you won't be programming directly in SPIR-V because it's designed for tools and drivers, not for humans.

In this post we will cover how resources are used in GLSL and HLSL.

## Shader Stages

Here's a table that matches the names of the shader stages in Vulkan and DX12.

<table>
<tr>
  <th>Vulkan</th>
  <th>DX12</th>
</tr>
<tr>
  <th colspan="2">Graphics Pipeline</th>
</tr>
<tr>
  <td>Vertex Shader</td>
  <td>Vertex Shader</td>
</tr>
<tr>
  <td>Fragment Shader</td>
  <td>Pixel Shader</td>
</tr>
<tr>
  <td>Geometry Shader</td>
  <td>Geometry Shader</td>
</tr>
<tr>
  <td>Tessellation Control Shader</td>
  <td>Hull Shader</td>
</tr>
<tr>
  <td>Tessellation Evaluation Shader</td>
  <td>Domain Shader</td>
</tr>
<tr>
  <th colspan="2">Compute Pipeline</th>
</tr>
<tr>
  <td>Compute Shader</td>
  <td>Compute Shader</td>
</tr>
<tr>
  <th colspan="2">Mesh Shading Pipeline</th>
</tr>
<tr>
  <td>Task Shader</td>
  <td>Amplification Shader</td>
</tr>
<tr>
  <td>Mesh Shader</td>
  <td>Mesh Shader</td>
</tr>
</table>

## Pipelines

|Vulkan|DX12|
|--|--|
|Pipeline|PSO (Pipeline State Object)|
|PipelineLayout|Root Signature|
|set|space|
|binding|slot|

Shaders are used to create Pipeline objects. \
Pipeline objects contain state that shaders need to know: the format of the vertex buffers, configuration of the rasterizer, configuration of color blending, configuration of the depth testing, the resources that can be bound, and [much much more](https://docs.vulkan.org/refpages/latest/refpages/source/VkGraphicsPipelineCreateInfo.html)

In **Vulkan** they use the term "**pipeline**"; in **DX12** they use the term "**PSO**" (pipeline state object).

## Resource layouts

For creating a pipeline object, a **resource layout** needs to be specified. \
The resource layout defines what resources can be bound to a pipeline, and what stages can access each of the bound resources. \
In **Vulkan** they use the term ["pipeline layout"](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreatePipelineLayout.html); in **DX12** they use the term ["root signature"](https://learn.microsoft.com/en-us/windows/win32/direct3d12/root-signatures-overview).

Resource layouts are organized in groups. \
In **Vulkan**, these groups are called ["descriptor sets"](https://docs.vulkan.org/spec/latest/chapters/descriptorsets.html). \
In **DX12**, these groups are called ["spaces"](https://learn.microsoft.com/en-us/windows/win32/direct3d12/resource-binding-in-hlsl).

Inside each of these groups (set/space), we have slots for our resource descriptors. \
We will cover descriptors extensively later. For now, you can just think of them as pointers to resources with some metadata. \
In **Vulkan** they use the term "**binding**" for the individual descriptor slots; in **DX12**, they just call it "**slot**".

The criteria for grouping descriptors into the same set/space is update frequency. \
For example, you could have 3 different sets:
- Per scene: with resources common to all objects of the same scene. Here we could have the lights, the environment map, or the shadowmaps.
- Per material: there might be multiple objects with the same material. They would share the albedo texture, the normal map, etc.
- Per object: the transformation matrices for each object.

This way, we only need to rebind the descriptor set that changes (better performance and code organization).

Sets/spaces and bindings/slots can be assigned arbitrary numbers, meaning you don't need to assign them consecutive numbers. Example resource layout: 

<table>
  <tr>
    <th>Set</th>
    <th>Binding</th>
    <th>Name</th>
  </tr>
  <tr>
    <td rowspan="3" scope="rowgroup">2</td>
    <td scope="row">1</td>
    <td>lightsBuffer</td>
  </tr>
  <tr>
    <td scope="row">5</td>
    <td>shadowMap</td>
  </tr>
  <tr>
    <td scope="row">2</td>
    <td>environmentMap</td>
  </tr>

  <tr>
    <td rowspan="3" scope="rowgroup">5</td>
    <td scope="row">0</td>
    <td>albedoTexture</td>
  </tr>
  <tr>
    <td scope="row">1</td>
    <td>normalTexture</td>
  </tr>
  <tr>
    <td scope="row">2</td>
    <td>pbrUniformBuffer</td>
  </tr>
</table>

Also, as you can see, you can have the same binding number in two different sets/spaces. For example, lightsBuffer and normalTexture both have binding number "1".

Here's an important distinction: in DX12 the slots are per resource type, meaning you can have a texture in slot 0 and, at the same time, a buffer in slot 0. In Vulkan that won't work; slots are shared for all resource types. \
Check out the following shader code:

```glsl
// GLSL
layout (set = 1, binding = 0) uniform texture2D u_albedoTexture;
layout (set = 1, binding = 1) uniform sampler u_albedoSampler; 
```

```hlsl
// HLSL
Texture2D<float4> u_albedoTexture : register(t0, space0);
SamplerState u_albedoSampler : register(s0, space0);
```

As you can see, in GLSL the sampler and the texture have different binding slots.\
In HLSL, both the texture and the sampler can be assigned to slot 0, because they are different types of resource. \
Also, you can see that, in HLSL, each type of resource has a different prefix:
- s: sampler
- t: texture
- b: constant buffer
- u: unordered access buffer

## Resource Views

Buffers and images can have **views** to them. \
Views can:
1) Reference parts of a resource.
2) Interpret the resource with a different format.

For example, you could have an ImageView that references a specific mip within an Image. \
Or you could have a buffer view that references a range of bytes within a buffer, and interprets them as RGB8 colors.

In Vulkan, we have just two types of view:
- VkBufferView
- VkImageView

In DX12, we have all these:
- Constant Buffer View (CBV)
- Shader Resource View (SRV)
- Unordered Access View (UAV)
- Render Target View (RTV)
- Depth Stencil View (DSV)
- Vertex Buffer View (VBV)
- Index Buffer View (IBV)
- Stream Output View (SOV)

When I initially learned about all the resource view types in DX12, I thought it was unnecessarily complex. Why so many types of view? \
It turns out that the term "view" doesn't mean exactly the same in Vulkan and DX12. \
In DX12, the concepts of "view" and "descriptor" are tied together.

## Descriptors

Descriptors are handles that can be bound to the slots of resource sets. \
Descriptors are lightweight objects that reference the resources we want to bind to pipelines/shaders.

In **Vulkan**, views and descriptors are two completely separate things. \
Descriptors don't reference buffers/images directly; descriptors reference views, and the view references the buffer/image.

In **DX12**, views and descriptors are closely related. \
Views can be used directly as descriptors. That's why in DX12 there are so many types of view: the view has information about how the resource is used, so it can be used as a descriptor.

These are the [types of descriptor](https://docs.vulkan.org/refpages/latest/refpages/source/VkDescriptorType.html) in **Vulkan**:
- **Sampler**: can be used to sample an image.
- **Combined Image-Sampler**: combines a sampled **image**, and a **sampler** into the same descriptor (Vulkan exclusive)
- **Sampled Image**: an **image** that can be sampled using a sampler.
- **Storage Image**: an **image** that can be used to load/store data in precise integer texel coordinates. It doesn't require the use of a sampler.
- **Uniform Texel Buffer**: a read-only **buffer** that can be used in the shader like a 1D image with a given format.
- **Storage Texel Buffer**: like a Uniform Texel Buffer, but can be written.
- **Uniform Buffer**: a read-only buffer (usually small) interpreted as a collection of values of different types. Has limitations in size, which enables them to be stored in fast, specific-purpose, memory banks.
- **Storage Buffer**: general purpose read/write buffer, with arbitrary type, and arbitrary size.
- **Input Attachment**: read-only image from previous render sub-passes (Vulkan exclusive). In Vulkan you can specify the sub-passes involved in a render pass, and the dependencies between them. This explicitness allows optimizations in tiled renderers (usually found in smartphones and other low-power devices). In DX12, since it's designed for desktop and XBOX, they don't have anything similar. As such, Vulkan is more complicated and verbose in this regard, and has been criticized for it. In response to that criticism, Vulkan made the [dynamic rendering](https://docs.vulkan.org/samples/latest/samples/extensions/dynamic_rendering/README.html) extension, which provides a simpler API. With this extension you can simplify your renderer if your application doesn't need to run on tiled HW.

In Vulkan and GLSL, the term "uniform" denotes that a binding is read-only.

These are the [types of descriptor](https://learn.microsoft.com/en-us/windows/win32/direct3d12/descriptors-overview) in **DX12**:
- Constant Buffer View (**CBV**): fast read-only buffer, with space limitations.
- Shader Resource View (**SRV**): read-only buffer or image.
- Unordered Access View (**UAV**): read/write buffer or image.
- **Sampler**: can be used to sample a texture.

As you can see, in DX12 there is a peculiar overlap between descriptors and views. For example:
- Constant Buffer View (CBV) is both a view and a descriptor.
- Sampler is a descriptor, but not a view.
- Index Buffer View (IBV) is a view, but not a descriptor.

The following table tries to match the descriptor types in Vulkan/DX12, and how they are used in GLSL/HLSL. \
After the table, there will be more detailed discussion.

<p></p>
<table>
<tr>
  <td> DX12<br>descriptor<br>type </td>
  <td> Vulkan<br>descriptor<br>type </td>
  <td> HLSL </td>
  <td> GLSL </td>
</tr>

<!----- SAMPLER ---------->
<tr>
  <td> sampler </td>
  <td> sampler </td>
<td>

```hlsl
SamplerState u_sampler
: register(...);
```

</td>

<td>

```glsl
layout (...)
uniform sampler u_sampler;
```

</td>
</tr>

<!----- SHADOW SAMPLER ---------->
<tr>
  <td> sampler </td>
  <td> sampler </td>
<td>

```hlsl
SamplerComparisonState
u_sampler : register(...);
```

</td>

<td>

```glsl
layout (...)
uniform samplerShadow u_sampler;
```

</td>
</tr>

<!----- COMBINED IMAGE-SAMPLER ---------->
<tr>
  <td> - </td>
  <td> combined<br>image+sampler </td>
<td> - </td>

<td>

```glsl
layout (...)
uniform sampler2D u_texture;
```

</td>
</tr>

<!------ CONSTANT BUFFER --------->
<tr>
  <td> CBV </td>
  <td> Uniform<br>Buffer </td>
<td>

```hlsl
cbuffer Uniforms :
register(...) {
  float4 A;
  float4 B;
};
```
or:
```hlsl
struct Uniforms {
  float4 A;
  float4 B;
};
ConstantBuffer<Uniforms>
uniforms : register(...);
```

</td>

<td>

```glsl
layout(...)
uniform Uniforms {
    vec4 A;
    vec4 B;
};
```
or:
```glsl
layout(...)
uniform Uniforms {
    vec4 A;
    vec4 B;
} uniforms;
```

</td>
</tr>

<!------ UNIFORM TEXEL BUFFER --------->
<tr>
  <td> SRV </td>
  <td> Uniform<br>Texel<br>Buffer </td>
<td>

```hlsl
Buffer<float3> colors
: register(...); 
```

</td>

<td>

```glsl
layout (...)
uniform samplerBuffer colors;
```

</td>
</tr>

<!------ STORAGE TEXEL BUFFER --------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Texel<br>Buffer </td>
<td>

```hlsl
RWBuffer<float3> colors
: register(...); 
```

</td>

<td>

```glsl
layout (...)
uniform imageBuffer colors;
```

</td>
</tr>

<!----- SAMPLED IMAGE ---------->
<tr>
  <td> SRV </td>
  <td> Sampled<br>Image </td>
<td>

```hlsl
Texture2D albedo
: register(...); 
```

</td>

<td>

```glsl
layout (...)
uniform texture2D albedo;
```

</td>
</tr>

<!----- STORAGE IMAGE ---------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Image </td>
<td>

```hlsl
RWTexture2D albedo
: register(...); 
```

</td>

<td>

```glsl
layout (...)
uniform image2D albedo;
```

</td>
</tr>

<!----- STRUCTURED BUFFER ---------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Buffer </td>
<td>

```hlsl
struct Particle {
  float4 position;
  float4 speed;
};
RWStructuredBuffer<Particle> particles
: register(...); 
```

</td>

<td>

```glsl
struct Particle {
  vec4 position;
  vec4 speed;
};
layout (std430, ...)
buffer Particles {
  Particle particles[];
};
```

</td>

<!----- BYTE ADDESS BUFFER ---------->
<tr>
  <td> SRV </td>
  <td> Storage<br>Buffer </td>
<td>

```hlsl
ByteAddressBuffer data
: register(...); 
```

</td>

<td>

```glsl
layout (std430, ...)
readonly buffer Data {
  uint data[];
};
```

</td>
</tr>

<!----- RW BYTE ADDESS BUFFER ---------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Buffer </td>
<td>

```hlsl
RWByteAddressBuffer data
: register(...); 
```

</td>

<td>

```glsl
layout (std430, ...)
buffer Data {
  uint data[];
};
```

</td>
</tr>

<!----- APPEND STRUCTURED BUFFER ---------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Buffer </td>
<td>

```hlsl
AppendStructuredBuffer<T> list
: register(...);

// ...
list.Append(X);
```

</td>

<td>

```glsl
layout (std430, set = 0, binding = 0)
buffer List {
    uint count;
    T elems[];
} list;

// ...
list.elems[atomicAdd(list.count, 1)] = X;
```

</td>
</tr>

<!----- CONSUME STRUCTURED BUFFER ---------->
<tr>
  <td> UAV </td>
  <td> Storage<br>Buffer </td>
<td>

```hlsl
ConsumeStructuredBuffer<T> list
: register(...);

// ...
X = list.Consume();
```

</td>

<td>

```glsl
layout (std430, set = 0, binding = 0)
buffer List {
    uint count;
    T elems[];
} list;

// ...
X = list.elems[atomicAdd(list.count, -1) - 1];
```

</td>
</tr>

</table>

I have deliberately skipped DX12's TextureBuffer/tbuffer: they are weird. They are basically the same as a constant buffer, but they are supposedly optimized for "[arbitrarily indexed data](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants)". There is very [little information](https://computergraphics.stackexchange.com/a/12967/6172) around about tbuffer; it doesn't look like people are using them.

### Sampler

Sampler descriptors are fairly simple.
In both APIs, these kinds of descriptors are handles that reference a sampler.

In HLSL:

```hlsl
// declare sampler
SamplerState u_sampler : register(s0, space0);
// use sampler
float4 color = u_sampler.Sample(myTexture, uv);
```

In GLSL:

```glsl
// declare sampler
uniform sampler u_sampler;
// use sampler
vec4 color = texture(sampler2D(myTexture, u_sampler), uv);
```

### Compare Sampler

Compare samplers are used to fetch texels and compare them against a reference value.\
It's commonly used in shadow mapping to figure out if a pixel is in shadow or not.\
When the texture is sampled we get 1.0 if the comparison is true, and 0.0 if the comparison is false. \
However, you can get values in between [0, 1] when using multiple samples (e.g when using linear filtering), then you get the percentage of comparisons that passed.

In HLSL:
```hlsl
// declaration
SamplerComparisonState u_sampler : register(s0, space0);
// usage
float visible = shadowMap.SampleCmpLevelZero(u_sampler, uv, referenceDepth);
```

In GLSL:
```glsl
// declaration
layout (set = 0, binding = 0) uniform samplerShadow u_sampler;
// usage
float visible = texture(sampler2DShadow(shadowMap, u_sampler), vec3(uv, referenceDepth));
```

### Combined Image-Sampler

Combined image-samplers are exclusive to Vulkan.

In GLSL:

```glsl
// declare sampler
layout (...) uniform sampler2D u_texture;
// use sampler
vec4 color = texture(u_texture, uv);
```

This also applies to shadow samplers:

In GLSL, with combined image-sampler:
```glsl
// declaration
layout (...) uniform sampler2DShadow u_shadowTex;
// usage
float visible = texture(u_shadowTex, vec3(uv, referenceDepth));
```

### Uniform Buffer

**Uniform Buffer** in Vulkan. \
**ConstantBuffer** in DX12.

A buffer that contains a set of constants that can be read from the shader. \
Usually, these kind of buffers are limited in size. Keeping them small can allow them to be placed in local, small, fast-to-access memory types.

In GLSL:
```glsl
// declaration
layout(...)
uniform Uniforms {
    vec4 A;
    vec4 B;
};

// usage
vec4 AB = A + B;
```
or:
```glsl
// declaration
layout(...)
uniform Uniforms {
    vec4 A;
    vec4 B;
} uniforms;

// usage
vec4 AB = uniforms.A + uniforms.B;
```

In HLSL:
```hlsl
// declaration
cbuffer Uniforms :
register(...) {
  float4 A;
  float4 B;
};

// usage
float4 AB = A + B;
```
or:
```hlsl
// declaration
struct Uniforms {
  float4 A;
  float4 B;
};
ConstantBuffer<Uniforms>
uniforms : register(...);

// usage
float4 AB = uniforms.A + uniforms.B;
```

Warning: in both APIs there are alignment rules that need to be taken into account. This applies to all kinds of buffers, but it's even more restrictive with uniforms buffers. I will not cover alignment rules here, as that could make it's own post.

### Uniform Texel Buffer

**Uniform Texel Buffer**s, [according to Vulkan](https://docs.vulkan.org/guide/latest/storage_image_and_texel_buffers.html#_texel_buffers), allow to access buffers with texture-like operations in shaders. I think this definition is misleading. To me they are just 1D arrays, but with a fixed color format for it's elements.

In GLSL:
```glsl
// declaration
layout (...) uniform samplerBuffer colors;
// usage
vec4 c = texelFetch(colors, 123);
```

In HLSL the equivalent would be `Buffer`:

```hlsl
// declaration
Buffer<float4> colors;
// usage
float4 c = colors[123];
```

This post talks about the difference between `Buffer` and `StructuredBuffer`: https://darkcorners.dev/buffers-vs-structuredbuffers

### Storage Texel Buffer

The same as uniform texel buffer, but allows for writing.

In GLSL:
```glsl
// declaration
layout (...) uniform imageBuffer colors;
// usage
imageStore(colors, 123, c);
```

In HLSL the equivalent would be `RWBuffer`:

```hlsl
// declaration
RWBuffer<float4> colors;
// usage
colors[123] = c;
```

### Sampled Image

An image which can be sampled using a sampler, or fetched using integer coordinates.

In GLSL:
```glsl
// declaration
layout(...) uniform texture2D albedo;
// usage
vec4 c = texelFetch(albedo, ivec2(12, 34));
```

In HLSL:
```hlsl
// declaration
Texture2D albedo : register(...);
// usage
float4 c = albedo.Load(int2(12, 34));
```

### Storage Image

An image that can be read & written.

In GLSL:
```glsl
// declaration
layout(...) uniform image2D albedo;
// usage (read)
vec4 c = imageLoad(albedo, ivec2(12, 34));
// usage (write)
imageStore(albedo, ivec2(12, 34), 0.5*c);
```

In HLSL:
```hlsl
// declaration
RWTexture2D albedo : register(...);
// usage (read)
float4 c = albedo.Load(int2(12, 34));
// usage (write)
albedo.Store(int2(12, 34), 0.5*c);
```

### Storage Buffer

Vulkan's Storage Buffer is very versatile. \
In GLSL, we can just use the "buffer" keyword. \
In HLSL, for convenience, there are multiple types for doing different things. But we can just simulate those in GLSL.

The thing that makes GLSL's `buffer` so versatile, is the fact that you can add an unspecified-length array as the last member.

Here follows DX12's buffer types that can be emulated with Vulkan's **Storage Buffer**s:

#### StructuredBuffer

A StructuredBuffer is a buffer that is interpreted as an array of a given type.\
The type can be any of the basic types (e.g int2, float4x4, etc), or a struct.\
In **HLSL**, there is `StructuredBuffer` and `RWStructuredBuffer`.
In **GLSL**, the `buffer` is read&write by default, but you can prepend the keyword `readonly` to make it... read-only.

In HLSL:
```hlsl
// declaration
struct Particle {
  float4 position;
  float4 speed;
};
RWStructuredBuffer<Particle> particles : register(...);

// usage
for (int i = 0; i < 1000; i++)
  particles[i].position += particles[i].speed * dt;
```

In GLSL:
```glsl
// declaration
struct Particle {
  vec4 position;
  vec4 speed;
};
layout (std430, ...) buffer Particles {
  Particle particles[];
};


// usage
for (int i = 0; i < 1000; i++)
  particles[i].position += particles[i].speed * dt;
```

#### ByteAddressBuffer

A buffer interpreted as an array of bytes.

In GLSL there isn't any direct equivalent, but it can be emulated with an array of uint and bitmasks. \
This is a bit cumbersome, yeah. \
However, it's a fairly good way to emulate it since, in HLSL, the ByteAddressBuffer needs to be 4-byte aligned anyway.

#### AppendStructuredBuffer

It's a type of StructuredBuffer that can be used to build a list by appending elements.

In GLSL, this can be emulated with an unspecified-length array, and an atomic counter.

In HLSL:

```hlsl
// declaration
AppendStructuredBuffer<T> list : register(...);

// usage
list.Append(X);

```

```glsl
// declaration
layout (std430, set = 0, binding = 0) buffer List {
    uint count;
    T elems[];
} list;

// usage
list.elems[atomicAdd(list.count, 1)] = X;
```

`atomicAdd` adds some number (1 in this case) to list.count, and returns the old value.

#### ConsumeStructuredBuffer

It's a type of StructuredBuffer that appears as a stream that you can consume values from.

In GLSL, this can be emulated with an unspecified-length array, and an atomic counter.

```hlsl
// declaration
ConsumeStructuredBuffer<T> list : register(...);

// usage
X = list.Consume();

```

```glsl
// declaration
layout (std430, set = 0, binding = 0) buffer List {
    uint count;
    T elems[];
} list;

// usage
X = list.elems[atomicAdd(list.count, -1) - 1];
```

## Conclusion

I hope this post has helped you in some way.\
Let me know in the comments if you have any feedback or questions.

# Links

- https://www.lei.chat/posts/hlsl-for-vulkan-resources/
- https://learn.microsoft.com/en-us/windows/win32/direct3d12/resource-binding-in-hlsl
- https://stefanpijnacker.nl/article/directx12-resources-key-concepts/
- https://logins.github.io/graphics/2020/07/31/DX12ResourceHandling.html