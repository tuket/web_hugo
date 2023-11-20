---
layout: page
title: Vulkan Concepts
date: 2022-08-23
published: true
toc: true
---

## Host & Device

**Host** refers to the CPU.

**Device** refers to the GPU.

In Vulkan, the **host** send commands to be executed by the **device**. The execution is asynchronous, so you need to be careful and use the explicit synchronization mechanisms that Vulkan provides.

## Vulkan instance

Out first interaction with the Vulkan API is creating an **instance** ([VkCreateInstance](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkCreateInstance.html)).

The Vulkan **instance** stores the Vulkan state for our application.

Some interesting info you can provide when creating an **instance**:
- The version of Vulkan we need to use
- The extensions we want to use
- The layers we wanto use

More info about the Vulkan instance:
- [LunarG's create Vulkan instance tutorial](https://vulkan.lunarg.com/doc/view/1.2.154.1/windows/tutorial/html/01-init_instance.html)
- [Spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/vkspec.html#initialization-instances)

## Devices

In Vulkan we have 2 abstractions regarding devices ([spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkCreateInstance.html)):
- **Physical device**: actual HW device 
- **Logical device**: instantiation of the device for our application

Find out which Vulkan-compatible physical devices are available in your system: [vkEnumeratePhysicalDevices](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkEnumeratePhysicalDevices.html).

Interesting **functions** associated with VKPhysicalDevice:
- [vkGetPhysicalDeviceProperties](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkGetPhysicalDeviceProperties.html).
- [VkPhysicalDeviceMemoryProperties](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceMemoryProperties.html).
- [vkGetPhysicalDeviceQueueFamilyProperties](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkGetPhysicalDeviceQueueFamilyProperties.html)
- [vkGetPhysicalDeviceSurfaceSupportKHR](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkGetPhysicalDeviceSurfaceSupportKHR.html)

## Queues

**Execution engines**: actual HW that processes work in the device.

**Queues** are the interface of the device that we can use to send work to the **execution engines**.

**Queue family**: queues are grouped in the device by queue families. All the queues in a family have the same capabilities (i.e. they can process the same kind of work). Queues may run asynchronously from the others.

In practice, you want to use the least amount of queues. You can even use only one.

When you create the logical device you specify the number of queues of each family type you want to create.

## Memory

Device memory allocation is explicit ([vkAllocateMemory](VkMemoryAllocateInfo) / [vkFreeMemory](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkFreeMemory.html))

## Buffer

## RenderPass

Is a collection of:
- Attachments. Description of the attachment properties:
  - Format: expected format of the image views attached
  - Number of MASAA samples
  - loadOp: specifies how the contents of the color and depth attachments are treated at the beginning of the first subpass where it's used.
  - storeOp: specifies how the contents of the color and depth attachments are treated at the and of the last subpass where it's used.
  - stencilLoadOp, stencirStoreOp: same but for the stencil attachment
  - initialLayout:
  - finalLayout: 
- Subpasses. Description of the subpasses:
  - pieplineBindPoint: GRAPHICS/COMPUTE
  - inputAttachments: references to attachments that are read
  - colorAttachments: references to attachments that are written
  - resolveAttachments: multisampling resolve attachments for colorAttachments
  - depthStencilAttachment: reference to depth stencil attachment to be written
  - preserveAttachments: if an attachment is not referenced in one of the previous arrays, the driver assument that the contens are not needed. Use preserveAttachments to prevent that
- Dependencies between subpases
  - src pass (index)
  - dst pass (index)
  - srcStageMask (precisely which pipeline stage)
  - dstStageMask
  - dependencyFlags: VK_DEPENDENCY_BY_REGION_BIT is interesting for tile-based GPUs because it indicates that you don't need to finish all tiles before stating next subpass

### RenderPass compatibility
Framebuffers and graphics pipelines are created based on a specific render pass object.

They must only be used with that render pass object, or one compatible with it.

Two attachment references are compatible if they have matching format and sample count, or are both VK_ATTACHMENT_UNUSED or the pointer that would contain the reference is NULL.

Two arrays of attachment references are compatible if all corresponding pairs of attachments are
compatible. If the arrays are of different lengths, attachment references not present in the smaller
array are treated as VK_ATTACHMENT_UNUSED.

Two render passes are compatible if their corresponding color, input, resolve, and depth/stencil
attachment references are compatible and if they are otherwise identical except for:
• Initial and final image layout in attachment descriptions
• Load and store operations in attachment descriptions
• Image layout in attachment references

As an additional special case, if two render passes have a single subpass, the resolve attachment
reference and depth/stencil resolve mode compatibility requirements are ignored.

A framebuffer is compatible with a render pass if it was created using the same render pass or a
compatible render pass.

The **host** and the **device** can access memory though buses.

## Memory availability and visibility

```goat
+-----------+           |          +-----------+
| AVAILABLE |<----------+--------->| AVAILABLE |
+-----------+           |          +-----------+
   |     ^              |                
   v     |              |
+-----------+           |
|  VISIBLE  |           |
+-----------+           |
      ^                 |
      |                 |
      v                 |
+-----------+           |
|    HOST   |           |
+-----------+           |

```

## Links

- [What is the pipeline a vkCmdPipelineBarrier applies to?](https://stackoverflow.com/questions/49237258/what-is-the-pipeline-a-vkcmdpipelinebarrier-applies-to)
- [Host write guarantees source access scope](https://stackoverflow.com/questions/61342339/host-write-guarantees-source-access-scope)
- [Yet another blog explaining Vulkan synchronization - Maister](https://themaister.net/blog/2019/08/14/yet-another-blog-explaining-vulkan-synchronization/)
- [Vulkan: vkCmdPipelineBarrier for data coherence](https://stackoverflow.com/questions/40577047/vulkan-vkcmdpipelinebarrier-for-data-coherence)

## Images

Q: Let say I allocated memory for a `VkImage` and it turns out to be `HOST_VISIBLE`. Can I just `vkMapMemory` and write directly to it? Even if the image is `TILING_OPTIMAL`?

A:

According to the spec:

>Upon creation, all image subresources of an image are initially in the same layout, where that
layout is selected by the VkImageCreateInfo::initialLayout member. The initialLayout must be either
VK_IMAGE_LAYOUT_UNDEFINED or VK_IMAGE_LAYOUT_PREINITIALIZED. If it is VK_IMAGE_LAYOUT_PREINITIALIZED, then the image data can be preinitialized by the host while using this layout, and the transition away from this layout will preserve that data. If it is VK_IMAGE_LAYOUT_UNDEFINED, then the contents of the data are considered to be undefined, and the transition away from this layout is not guaranteed to preserve that data. For either of these initial layouts, any image subresources must be transitioned to another layout before they are accessed by the device.
>
> Host access to image memory is only well-defined for linear images and for image subresources of those images which are currently in either the VK_IMAGE_LAYOUT_PREINITIALIZED or VK_IMAGE_LAYOUT_GENERAL layout. Calling vkGetImageSubresourceLayout for a linear image returns a subresource layout mapping that is valid for either of those image layouts.

So not possible for `TILING_OPTIMAL`, but possible for `TILING_LINEAR`.

Additionally, the image layout needs to be `PREINITIALIZED` or `GENERAL`.

However, for textures, we want to use `TILING_OPTIMAL` because:
- Has better performance
- Linear tiling might not even be supported for sampling

In practice, this means that it's not possible to fill a texture without a staging buffer.

You could use a staging `TILING_LINEAR` image to fill a `TILING_OPTIMAL` image. But this is not a good idea because this is not supported for compressesed format. Using a staging buffer supports both compressed and non-compressed formats, so it's a better approach for code simplicity.

We can use the `VK_EXT_external_memory_host` extension to bind a host data pointer directly to a `VkBuffer`. That would allow us to save a `memcpy`! ([link](https://stackoverflow.com/questions/71626199/can-you-transfer-directly-to-an-image-without-using-an-intermediary-buffer))

## Pipeline layout

To crate a pipeline layout you need:
- []VkDescriptorSetLayout
- []VkPushConstantRange

## Descriptor set layout

To create a VkDescriptorSetLayout you need:
- flags: 
  - CREATE_UPDATE_AFTER_BIND_POOL_BIT (1.2)
  - CREATE_PUSH_DESCRIPTOR_BIT_KHR (VK_KHR_push_descriptor)
- []VkDescriptorSetLayoutBinding

```c++
struct VkDescriptorSetLayoutBinding {
    uint32_t binding;
    VkDescriptorType descriptorType;
    uint32_t descriptorCount;
    VkShaderStageFlags stageFlags;
    const VkSampler* pImmutableSamplers;
}
```

## Memory alignment rules

- We can consider matrices as column-major

![](https://upload.wikimedia.org/wikipedia/commons/thumb/4/4d/Row_and_column_major_order.svg/180px-Row_and_column_major_order.svg.png)

- In GLSL, a matrix type `matCxR` has `C` columns (vectors) and `R` rows
- For alignment porposes, `matCxR` is equivalent to an array `vecR[C]`
- bool has size 4, just like float or int!

We have 3 types of alignment:
- Scalar alignment
- Base alignment (std430)
- Extended alignment (std140, the default for uniform buffers)
- Relaxed alignment

### Scalar Alignment
1) Basic types such as bool, float, int, or double have scalar alignment equal to it's size.
2) A vector type has a scalar alignment equal to that of its component type
3) An array type has a scalar alignment equal to that of its element type
4) A structure has a scalar alignment equal to the largest scalar alignment of any of its members

All these points are pretty obvious, except for 4 maybe. Point 4 implies that if our struct has a `double` then the scalar alignment becomes `8`.

For using scalar alignment you need the `VK_EXT_scalar_block_layout` extension, which was promoted to core in Vulkan 1.2.
In GLSL you also need to enable the corresponding extension:
```glsl
#extension GL_EXT_scalar_block_layout : enable
layout (scalar, binding = 0) buffer block { }
```

### Base alignment
1) A scalar has a base alignment equal to its scalar alignment
2) A 2-component vector has a base alignment equal to x2 its scalar alignment
3) A 3- or 4-component vector has a base alignment equal to four times its scalar alignment
4) An array has a base alignment equal to the base alignment of its element type
5) A structure has a base alignment equal to the largest base alignment of any of its members.

From these set of rules, probably 3 and 4 are the most interesting.

Rule 3 tells us that 3-component vectors are actually aligned as if they were 4-component vectors. Some people even argue that one should [avoid using vec3 altogether](https://stackoverflow.com/q/38172696/1754322) in interface blocks.

Base alignment is the one used by default "push constants" and "storage buffers". In GLSL this type of alignment is know as std430.

```glsl
layout(set = 0, binding = 0) uniform MyUniforms {
  // "uniform" uses scalar "extended alignment" by default (std140)
  float myUniform0;
  vec2 myUniforms1;
};

layout(set = 0, binding = 1) buffer MyStorageBuffer {
  // push_constant uses "base alignment" by default (std140)
  uint myStorageBuffer0;
};

layout(push_constant) uniform MyPushConstants {
  // push_constant uses "base alignment" by default (std140)
  mat4 myPushConstant;
};

```

You can also use "base alignment" for uniform buffer through an extension `VK_KHR_uniform_buffer_standard_layout`. This extension has gone into `core` as for Vulkan 1.2.

```
layout(std430, set = 0, binding = 0) uniform MyUniforms {
  float myUniform0;
  vec2 myUniforms1;
};
```

### Extended aligment
1) A scalar or vector type has an extended alignment equal to its base alignment.
2) An array or structure type has an extended alignment equal to the largest extended alignment of any of its members, rounded up to a multiple of 16.

So basicaly, extended alignment is like "base alignment" but with that weird rounding rule. Which actually changes everything!

And "extended alignment" (aka std140) is the default for uniform buffers!

Example:

```glsl
struct MyData {
  float x;
};

layout (set = 0, binding = 0) uniform MyUniforms {
  MyData myData;
  float y; // offset 16
};
```

In the previous example, eventhough the struct has only one float (due to the rounding rule) it takes the space of a full vec4!

### Relaxed alignment

It's a small improvement to "base aligment" (std430) introduced in Vulkan 1.1. You don't need to do anything special to use "relaxed alignement" is automatically applies is you intend to use std430 and are using Vulkan 1.1+.

In "base aligment" if you had a struct with a vec3 followed by a float:

```glsl
struct MyData {
  vec3 abc;
  float d;
};
```

The two variables would be packed together nicely as if the where a vec4. But, due to the alignment rules, that wouldn't be the case id the order is reversed:

```
struct MyData {
  float a;
  vec3 bcd; // offset 16
}
```

Relaxed alignement addressed this precisely this issue. So the 2 variables are packed nicely in a vec4 as well. Just a small quality of life improvement.



Links about memory layout in Vulkan:
- https://docs.vulkan.org/guide/latest/shader_memory_layout.html
- https://registry.khronos.org/vulkan/specs/1.3-extensions/html/vkspec.html#interfaces-resources-layout
- https://www.reddit.com/r/vulkan/comments/wc0428/on_shader_memory_layout/
- https://stackoverflow.com/questions/38172696/should-i-ever-use-a-vec3-inside-of-a-uniform-buffer-or-shader-storage-buffer-o
- https://fvcaputo.github.io/2019/02/06/memory-alignment.html

## Pipelines

