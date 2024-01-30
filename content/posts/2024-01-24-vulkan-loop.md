---
layout: post
title: "Vulkan render loop synchronization"
date: 2024-01-24
published: true
---

One of the things I struggled the most when learning Vulkan is the **render loop synchronization**. For over 2 years I've been revisiting the topic many times. Every time, I though I had finally understood how to do it properly. But I see different approaches implemented in tutorials and examples out there, and some of them are buggy or sub-optimal.

In this post I wanted to write down the way I settled with. This way is correct (no validation errors), optimal (no unnecessary stalls), and simple.

## Be ready to support triple buffering

**Double buffering** is a good tradeoff for memory and performance. You just have two images: while one is being presented, you can keep drawing into the other. You could have more images in the swapchain in order to increase the amount of work that can be done concurrently, but many programs don't need to go as far as using triple buffering.

Still, when you create a swapchain in Vulkan, you provide a **minimum** amount of images to be created (`VkSwapchainCreateInfoKHR::minImageCount`). However, the driver could give you **more** images than you requested. So you need to be able to support that case. It has actually happened to me that I had some code that would work in Nvidia, but when I tested it in another AMD card it would crash because I wasn't expecting anything other than double buffering.

 ## vkAcquireNextImageKHR doesn't block as you might expect

```c
// Provided by VK_KHR_swapchain
VkResult vkAcquireNextImageKHR(
    VkDevice                                    device,
    VkSwapchainKHR                              swapchain,
    uint64_t                                    timeout,
    VkSemaphore                                 semaphore,
    VkFence                                     fence,
    uint32_t*                                   pImageIndex);
```

You see the `timeout` parameter?

This is what the documentation says about it:

```
timeout specifies how long the function waits, in nanoseconds, if no image is available
```

The function will return `VK_TIMEOUT` if the image could not be acquired within `timeout` nanoseconds. If `timeout` is 0, we try to acquire the image immediately. And the function returns `VK_NOT_READY` if the images could not be acquired at the time we call the function. If the image is successfully acquired the function returns `VK_SUCCESS`, and the index of the image is returned though the `pImageIndex` parameter.

Here is the confusing part: even if the function successfully returns after some waiting/blocking on the CPU, that doesn't mean you can already draw to the image. The presentation engine could be still reading that image. You would also need to wait on the `fence` for that. So why the timeout if we need to wait on the fence anyways? The function returns when it **knows** what the index of the next image will be. But that doesn't mean you can use that image yet. For that you can use the `fence` and `semaphore` parameters.

## Use at least one command buffer for each swapchain image

The whole point of double buffering (or N-buffering) is to be able to draw consecutive frames in parallel. If your swapchain has N images, you should have at least one **command buffer** for each of them. You could have N+1 command buffers, if you would like to start recording commands for the next frame while we are rendering the N previous frames. But I don't think this is necessary.

It's also worth considering using a **command pool** per swapchain image. This way you can reset the whole pool every frame. It should be quicker than resetting individual command buffers.

## Use a fences to limit the FPS

If the frames are quick to render, you will eventually run out of images to render to. All images will be queued in the presentation engine. Use fences to perform idle waiting.

## pseudo code

```cpp

VkSemaphore semaphore_imageAvailable[N + 1]; // notice the +1 (even if all N frames have been presented, we would like to query what will be the next image index)
u32 semaphoreInd_imageAvailable = 0;
VkSemaphore semaphore_drawFinished[N];
VkFence fence_drawFinished[N];
VkCommandBuffer cmdBuffers_draw[N];

while(true) // render loop
{
    u32 scImgInd = acquireNextImage(semaphore_imageAvailable[semaphoreInd_imageAvailable]);

    { // wait until drawing to scImgInd has finished. We need to wait because, otherwise, cmdBuffers_draw[scImgInd] would be in use
        VkFence& fence = fence_drawFinished[scImgInd];
        vkWaitForFences(device.device, 1, &fence, VK_FALSE, u64(-1));
        vkResetFences(device.device, 1, &fence);
    }

    auto& cmdBuffer_draw = cmdBuffers_draw[scImgInd];
    reset(cmdBuffer_draw);
    begin(cmdBuffer_draw);
        // record cmds
        // ...
    end(cmdBuffer_draw)

    submit(
        .cmdBuffer = cmdBuffer_draw,
        .waitSemaphore = semaphore_imageAvailable[semaphoreInd_imageAvailable],
        .signalSemaphore = semaphore_drawFinished[scImgInd],
        .signalFence = fence_drawFinished[scImgInd],
    );

    present(
        .waitSemaphore = semaphore_drawFinished[scImgInd],
        .imageInd = scImgInd,
    );

    semaphoreInd_imageAvailable = (semaphoreInd_imageAvailable + 1) % (N + 1);
}
```

This is essentially the same approach used in [Khronos Examples](https://github.com/KhronosGroup/Vulkan-Samples/blob/27d1c21f82be8c580349d3e19f85891be504eea5/samples/api/hello_triangle/hello_triangle.cpp). But they use a semaphore recycle queue, instead of using our simple [N+1] array.

## Delay the destruction of resources N frames

Whenever you need to destroy a Vulkan resource, you need to keep in mind it might still be in use by previous frames!

Without the need of any extra synchronization primitives, we can leverage our render loop to do safely destroy resources.

Every time we want to destroy a resource, we will instead push the resource into a per-frame queue. After N frames, we can be sure that the resource is not being used by any cmd buffer.

```cpp

// ... 
std::vector<VkBuffer> toDestroy_buffers[N];
std::vector<VkBuffer> toDestroy_images[N];

while(true)
{
    // ...

    { // wait until drawing to scImgInd has finished
        VkFence& fence = fence_drawFinished[scImgInd];
        vkWaitForFences(device.device, 1, &fence, VK_FALSE, u64(-1));
        vkResetFences(device.device, 1, &fence);
    }

    for(auto buffer : toDestroy_buffers[scImgInd])
        destroy(buffer);

    for(auto image : toDestroy_images[scImgInd])
        destroy(image);

    //...
}

```
