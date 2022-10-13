---
layout: page
title: Vulkan Concepts
date: 2022-08-23
published: false
---

## Host & Device

**Host** refers to the CPU.

**Device** refers to the GPU.

In Vulkan, the **host** send commands to be executed by the **device**. The execution is asynchronous, so you need to be careful  and use the explicit synchronization mechanisms that Vulkan provides.

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

