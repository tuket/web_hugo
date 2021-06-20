---
layout: page
title: Vulkan cheatsheet
date: 2021-05-15
published: false
---

## Enumerating layers

```cpp
constexpr u32 MAX_LAYERS = 32;
VkLayerProperties layersProps[MAX_LAYERS];
u32 numLayers = MAX_LAYERS;
vkEnumerateInstanceLayerProperties(&numLayers, layersProps);
printf("Available Layers\n");
for(u32 i = 0; i < numLayers; i++) {
    printf("%s: %s\n", layersProps[i].layerName, layersProps[i].description);
}
```

## Create a Vulkan instance

```cpp
VkApplicationInfo appInfo = {};
appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
appInfo.pApplicationName = "example";
appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.pEngineName = "none";
appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.apiVersion = VK_API_VERSION_1_0;

VkInstanceCreateInfo instanceCreateInfo = {};
instanceCreateInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
instanceCreateInfo.pApplicationInfo = &appInfo;
instanceCreateInfo.enabledLayerCount = 1;
static const char* const LAYER_NAMES[] = {"VK_LAYER_KHRONOS_validation"};
instanceCreateInfo.ppEnabledLayerNames = LAYER_NAMES;
VkInstance inst;
vkCreateInstance(&instanceCreateInfo, nullptr, &inst);
```

# Enumerate Physical Devices
```cpp
constexpr u32 MAX_PHYSICAL_DEVICES = 4;
VkPhysicalDevice physicalDevices[MAX_PHYSICAL_DEVICES];
u32 numPhysicalDevices = MAX_PHYSICAL_DEVICES;
vkEnumeratePhysicalDevices(inst, &numPhysicalDevices, physicalDevices);
```

# Descriptor Set



# Pipeline Layout




vkEnumeratePhysicalDevices

VkPhysicalDeviceProperties

vkGetPhysicalDeviceProperties

vkGetPhysicalDeviceQueueFamilyProperties

VkQueueFamilyProperties

VkDeviceQueueCreateInfo

VkPhysicalDeviceFeatures

VkDeviceCreateInfo

vkCreateDevice

VkPhysicalDeviceMemoryProperties

vkGetPhysicalDeviceMemoryProperties

VkBufferCreateInfo

vkCreateBuffer

VkMemoryRequirements

vkGetBufferMemoryRequirements

VkDeviceMemory

VkMemoryAllocateInfo

vkAllocateMemory

vkBindBufferMemory

VkShaderModule

VkShaderModuleCreateInfo

vkCreateShaderModule

VkPipelineShaderStageCreateInfo

VkComputePipelineCreateInfo

VkPipelineLayoutCreateInfo
