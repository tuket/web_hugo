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
- Subpasses. Deescription of the subpasses:
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