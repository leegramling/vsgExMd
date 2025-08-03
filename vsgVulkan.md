# VSG Vulkan Initialization Guide

A comprehensive guide to understanding how the Vulkan Scene Graph (VSG) initializes and manages Vulkan resources, from basic setup to advanced configuration.

## Table of Contents

1. [Introduction](#introduction)
2. [VSG Vulkan Initialization Overview](#vsg-vulkan-initialization-overview)
3. [WindowTraits: The Configuration Foundation](#windowtraits-the-configuration-foundation)
4. [Instance Management](#instance-management)
5. [Physical Device Selection](#physical-device-selection)
6. [Logical Device Creation](#logical-device-creation)
7. [Swapchain and Surface Management](#swapchain-and-surface-management)
8. [Context and Resource Management](#context-and-resource-management)
9. [Command Graph Creation](#command-graph-creation)
10. [Advanced Configuration and Extensions](#advanced-configuration-and-extensions)
11. [Debugging and Validation](#debugging-and-validation)
12. [Common Patterns and Best Practices](#common-patterns-and-best-practices)

## Introduction

Vulkan is a powerful but complex graphics API that requires extensive boilerplate code for initialization. The Vulkan Scene Graph (VSG) abstracts this complexity while maintaining full access to Vulkan's capabilities when needed.

### Raw Vulkan vs VSG Comparison

```cpp
// Raw Vulkan: 100+ lines of initialization code
VkInstance instance;
VkPhysicalDevice physicalDevice;
VkDevice device;
VkSurfaceKHR surface;
VkSwapchainKHR swapchain;
// ... extensive setup code ...

// VSG: Clean, simple initialization
auto windowTraits = vsg::WindowTraits::create();
auto window = vsg::Window::create(windowTraits);
auto viewer = vsg::Viewer::create();
viewer->addWindow(window);
```

### Key VSG Benefits

- **Automatic Resource Management**: Reference counting eliminates manual cleanup
- **Intelligent Defaults**: Sensible default configurations that work out-of-the-box
- **Validation Integration**: Built-in validation layer support
- **Cross-Platform**: Platform-specific details handled automatically
- **Fine-Grained Control**: Access to low-level Vulkan features when needed

## VSG Vulkan Initialization Overview

VSG follows a structured initialization pipeline that mirrors Vulkan's requirements while hiding complexity:

```cpp
// VSG Initialization Pipeline
WindowTraits → Window → Instance → PhysicalDevice → LogicalDevice → Context → CommandGraph
```

### Core VSG Classes

| VSG Class | Vulkan Equivalent | Purpose |
|-----------|-------------------|---------|
| `vsg::Instance` | `VkInstance` | Vulkan instance management |
| `vsg::PhysicalDevice` | `VkPhysicalDevice` | GPU enumeration and capabilities |
| `vsg::Device` | `VkDevice` | Logical device and queues |
| `vsg::Surface` | `VkSurfaceKHR` | Window surface for presentation |
| `vsg::Context` | Command pools + queues | Resource compilation and submission |
| `vsg::WindowTraits` | N/A | Configuration container |

### Basic Initialization Pattern

```cpp
int main(int argc, char** argv)
{
    // 1. Configure window and Vulkan settings
    auto windowTraits = vsg::WindowTraits::create();
    windowTraits->windowTitle = "My VSG Application";
    windowTraits->debugLayer = true;  // Enable validation
    
    // 2. Create window (automatically initializes Vulkan)
    auto window = vsg::Window::create(windowTraits);
    if (!window) {
        std::cerr << "Failed to create window" << std::endl;
        return 1;
    }
    
    // 3. Create viewer and assign window
    auto viewer = vsg::Viewer::create();
    viewer->addWindow(window);
    
    // 4. Setup scene and camera
    auto scene = createScene();
    auto camera = createCamera(window);
    
    // 5. Create command graph
    auto commandGraph = vsg::createCommandGraphForView(window, camera, scene);
    viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph});
    
    // 6. Compile resources
    viewer->compile();
    
    // 7. Main loop
    while (viewer->advanceToNextFrame()) {
        viewer->handleEvents();
        viewer->update();
        viewer->recordAndSubmit();
        viewer->present();
    }
    
    return 0;
}
```

## WindowTraits: The Configuration Foundation

`WindowTraits` is VSG's central configuration class that controls all aspects of window and Vulkan initialization.

### Basic Window Configuration

```cpp
auto windowTraits = vsg::WindowTraits::create();

// Window properties
windowTraits->windowTitle = "Advanced VSG Application";
windowTraits->x = 100;              // Window position X
windowTraits->y = 100;              // Window position Y
windowTraits->width = 1920;         // Window width
windowTraits->height = 1080;        // Window height
windowTraits->fullscreen = false;   // Windowed mode
windowTraits->decoration = true;    // Window decorations (title bar, etc.)
windowTraits->overrideRedirect = false;  // X11 override redirect
```

### Display and Screen Selection

```cpp
// Multi-monitor setup
windowTraits->screenNum = 1;        // Use second monitor
windowTraits->display = ":0.1";     // X11 display specification

// Query available screens
for (uint32_t i = 0; i < vsg::getNumScreens(); ++i) {
    auto screenProperties = vsg::getScreenProperties(i);
    std::cout << "Screen " << i << ": " 
              << screenProperties.width << "x" << screenProperties.height << std::endl;
}
```

### Vulkan-Specific Configuration

```cpp
// Vulkan API version
windowTraits->vulkanVersion = VK_API_VERSION_1_2;

// Queue requirements
windowTraits->queueFlags = VK_QUEUE_GRAPHICS_BIT | VK_QUEUE_COMPUTE_BIT;

// Multisampling
windowTraits->samples = VK_SAMPLE_COUNT_8_BIT;  // 8x MSAA

// Depth buffer format
windowTraits->depthFormat = VK_FORMAT_D32_SFLOAT;
```

### Debug and Validation

```cpp
// Validation layers
windowTraits->debugLayer = true;           // Enable Khronos validation layer
windowTraits->apiDumpLayer = false;        // Disable API call logging
windowTraits->synchronizationLayer = false; // Disable sync validation

// GPU debugging
windowTraits->debugUtils = true;           // Enable debug utils extension
```

### Device Features

```cpp
// Create and assign device features
auto deviceFeatures = vsg::DeviceFeatures::create();
windowTraits->deviceFeatures = deviceFeatures;

// Core features
deviceFeatures->get().samplerAnisotropy = VK_TRUE;
deviceFeatures->get().geometryShader = VK_TRUE;
deviceFeatures->get().tessellationShader = VK_TRUE;
deviceFeatures->get().fillModeNonSolid = VK_TRUE;  // For wireframe
deviceFeatures->get().wideLines = VK_TRUE;         // For line width > 1.0

// Vulkan 1.1 features
auto& vulkan11Features = deviceFeatures->get<VkPhysicalDeviceVulkan11Features>();
vulkan11Features.shaderDrawParameters = VK_TRUE;

// Vulkan 1.2 features
auto& vulkan12Features = deviceFeatures->get<VkPhysicalDeviceVulkan12Features>();
vulkan12Features.bufferDeviceAddress = VK_TRUE;
vulkan12Features.descriptorIndexing = VK_TRUE;
```

### Extension Management

```cpp
// Instance extensions (added automatically for platform)
windowTraits->instanceExtensionNames.push_back("VK_EXT_debug_utils");

// Device extensions
windowTraits->deviceExtensionNames.push_back("VK_KHR_swapchain");
windowTraits->deviceExtensionNames.push_back("VK_EXT_line_rasterization");

// Check extension availability
auto physicalDevice = window->getOrCreatePhysicalDevice();
if (physicalDevice->supportsDeviceExtension("VK_EXT_mesh_shader")) {
    windowTraits->deviceExtensionNames.push_back("VK_EXT_mesh_shader");
}
```

## Instance Management

VSG's `Instance` class manages the Vulkan instance with automatic platform integration and validation.

### Automatic Instance Creation

```cpp
// VSG automatically creates instance through window
auto window = vsg::Window::create(windowTraits);
auto instance = window->getOrCreateInstance();

// Access underlying VkInstance
VkInstance vkInstance = *instance;
```

### Manual Instance Creation

```cpp
// Collect required extensions
vsg::Names instanceExtensions;
instanceExtensions.push_back(VK_KHR_SURFACE_EXTENSION_NAME);

#ifdef VK_USE_PLATFORM_WIN32_KHR
    instanceExtensions.push_back(VK_KHR_WIN32_SURFACE_EXTENSION_NAME);
#elif defined(VK_USE_PLATFORM_XCB_KHR)
    instanceExtensions.push_back(VK_KHR_XCB_SURFACE_EXTENSION_NAME);
#elif defined(VK_USE_PLATFORM_XLIB_KHR)
    instanceExtensions.push_back(VK_KHR_XLIB_SURFACE_EXTENSION_NAME);
#endif

// Setup validation layers
vsg::Names requestedLayers;
if (enableValidation) {
    instanceExtensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    requestedLayers.push_back("VK_LAYER_KHRONOS_validation");
}

// Validate layer availability
vsg::Names validatedLayers = vsg::validateInstancelayerNames(requestedLayers);

// Create instance
auto instance = vsg::Instance::create(
    instanceExtensions,
    validatedLayers,
    VK_API_VERSION_1_2
);
```

### Instance Layer Enumeration

```cpp
// List all available instance layers
auto availableLayers = vsg::enumerateInstanceLayerProperties();
std::cout << "Available Instance Layers:" << std::endl;
for (const auto& layer : availableLayers) {
    std::cout << "  " << layer.layerName << " (spec: " << layer.specVersion 
              << ", impl: " << layer.implementationVersion << ")" << std::endl;
    std::cout << "    " << layer.description << std::endl;
}

// List available instance extensions
auto availableExtensions = vsg::enumerateInstanceExtensionProperties();
std::cout << "Available Instance Extensions:" << std::endl;
for (const auto& extension : availableExtensions) {
    std::cout << "  " << extension.extensionName 
              << " (spec: " << extension.specVersion << ")" << std::endl;
}
```

## Physical Device Selection

VSG provides flexible physical device selection with automatic fallback and preference handling.

### Automatic Device Selection

```cpp
// VSG automatically selects best available device
auto window = vsg::Window::create(windowTraits);
auto physicalDevice = window->getOrCreatePhysicalDevice();

// Access device properties
auto properties = physicalDevice->getProperties();
std::cout << "Selected Device: " << properties.deviceName << std::endl;
std::cout << "Device Type: " << properties.deviceType << std::endl;
std::cout << "API Version: " << VK_VERSION_MAJOR(properties.apiVersion) 
          << "." << VK_VERSION_MINOR(properties.apiVersion) 
          << "." << VK_VERSION_PATCH(properties.apiVersion) << std::endl;
```

### Device Enumeration and Selection

```cpp
auto instance = window->getOrCreateInstance();
auto surface = window->getOrCreateSurface();

// Enumerate all physical devices
auto physicalDevices = instance->getPhysicalDevices();
std::cout << "Available Physical Devices:" << std::endl;

for (size_t i = 0; i < physicalDevices.size(); ++i) {
    auto device = physicalDevices[i];
    auto properties = device->getProperties();
    auto features = device->getFeatures();
    auto memoryProperties = device->getMemoryProperties();
    
    std::cout << "Device " << i << ": " << properties.deviceName << std::endl;
    std::cout << "  Type: ";
    switch (properties.deviceType) {
        case VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU:
            std::cout << "Discrete GPU"; break;
        case VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU:
            std::cout << "Integrated GPU"; break;
        case VK_PHYSICAL_DEVICE_TYPE_VIRTUAL_GPU:
            std::cout << "Virtual GPU"; break;
        case VK_PHYSICAL_DEVICE_TYPE_CPU:
            std::cout << "CPU"; break;
        default:
            std::cout << "Other"; break;
    }
    std::cout << std::endl;
    
    // Check queue family support
    auto queueFamilies = device->getQueueFamilyProperties();
    for (size_t j = 0; j < queueFamilies.size(); ++j) {
        const auto& queueFamily = queueFamilies[j];
        std::cout << "  Queue Family " << j << ": " << queueFamily.queueCount << " queues" << std::endl;
        
        if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) 
            std::cout << "    Graphics support" << std::endl;
        if (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT) 
            std::cout << "    Compute support" << std::endl;
        if (queueFamily.queueFlags & VK_QUEUE_TRANSFER_BIT) 
            std::cout << "    Transfer support" << std::endl;
        
        // Check presentation support
        VkBool32 presentSupport = device->getSurfaceSupport(j, surface);
        if (presentSupport) 
            std::cout << "    Present support" << std::endl;
    }
    
    // Memory heaps
    std::cout << "  Memory Heaps:" << std::endl;
    for (uint32_t j = 0; j < memoryProperties.memoryHeapCount; ++j) {
        const auto& heap = memoryProperties.memoryHeaps[j];
        std::cout << "    Heap " << j << ": " << (heap.size >> 20) << " MB";
        if (heap.flags & VK_MEMORY_HEAP_DEVICE_LOCAL_BIT)
            std::cout << " (Device Local)";
        std::cout << std::endl;
    }
}
```

### Manual Device Selection

```cpp
// Select device by index
if (size_t deviceIndex = 0; arguments.read("--device", deviceIndex)) {
    auto physicalDevices = instance->getPhysicalDevices();
    if (deviceIndex < physicalDevices.size()) {
        window->setPhysicalDevice(physicalDevices[deviceIndex]);
    }
}

// Select device by preference
auto physicalDevice = instance->getPhysicalDevice(
    VK_QUEUE_GRAPHICS_BIT,  // Required queue flags
    surface,                // Surface for presentation support
    {VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU, VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU}  // Preference order
);
```

### Device Capability Checking

```cpp
auto physicalDevice = window->getOrCreatePhysicalDevice();

// Check feature support
auto features = physicalDevice->getFeatures();
if (features.geometryShader) {
    std::cout << "Geometry shaders supported" << std::endl;
}

// Check extension support
if (physicalDevice->supportsDeviceExtension("VK_EXT_mesh_shader")) {
    std::cout << "Mesh shaders supported" << std::endl;
}

// Check format support
auto formatProperties = physicalDevice->getFormatProperties(VK_FORMAT_R8G8B8A8_SRGB);
if (formatProperties.optimalTilingFeatures & VK_FORMAT_FEATURE_COLOR_ATTACHMENT_BIT) {
    std::cout << "sRGB color attachment supported" << std::endl;
}

// Surface capabilities
auto surfaceCapabilities = physicalDevice->getSurfaceCapabilities(surface);
std::cout << "Surface capabilities:" << std::endl;
std::cout << "  Min images: " << surfaceCapabilities.minImageCount << std::endl;
std::cout << "  Max images: " << surfaceCapabilities.maxImageCount << std::endl;
std::cout << "  Current extent: " << surfaceCapabilities.currentExtent.width 
          << "x" << surfaceCapabilities.currentExtent.height << std::endl;
```

## Logical Device Creation

VSG's `Device` class manages logical device creation, queue setup, and resource sharing.

### Basic Device Creation

```cpp
// VSG creates device automatically with sensible defaults
auto window = vsg::Window::create(windowTraits);
auto device = window->getOrCreateDevice();

// Access underlying VkDevice
VkDevice vkDevice = *device;
```

### Custom Device Creation

```cpp
auto instance = window->getOrCreateInstance();
auto physicalDevice = window->getOrCreatePhysicalDevice();
auto surface = window->getOrCreateSurface();

// Find queue families
auto [graphicsFamily, presentFamily] = physicalDevice->getQueueFamily(
    VK_QUEUE_GRAPHICS_BIT, surface);

if (graphicsFamily < 0 || presentFamily < 0) {
    throw std::runtime_error("Required queue families not found");
}

// Setup queue creation
vsg::QueueSettings queueSettings;
if (graphicsFamily == presentFamily) {
    // Same queue family for graphics and present
    queueSettings.push_back(vsg::QueueSetting{graphicsFamily, {1.0f}});
} else {
    // Separate queue families
    queueSettings.push_back(vsg::QueueSetting{graphicsFamily, {1.0f}});
    queueSettings.push_back(vsg::QueueSetting{presentFamily, {1.0f}});
}

// Configure device features
auto deviceFeatures = vsg::DeviceFeatures::create();
deviceFeatures->get().samplerAnisotropy = VK_TRUE;
deviceFeatures->get().fillModeNonSolid = VK_TRUE;

// Device extensions
vsg::Names deviceExtensions;
deviceExtensions.push_back(VK_KHR_SWAPCHAIN_EXTENSION_NAME);

// Validation layers for device (if enabled)
vsg::Names validationLayers;
if (enableValidation) {
    validationLayers.push_back("VK_LAYER_KHRONOS_validation");
}

// Create logical device
auto device = vsg::Device::create(
    physicalDevice,
    queueSettings,
    validationLayers,
    deviceExtensions,
    deviceFeatures
);

// Assign to window
window->setDevice(device);
```

### Multi-Window Device Sharing

```cpp
// First window creates device
auto windowTraits1 = vsg::WindowTraits::create();
windowTraits1->windowTitle = "Primary Window";
auto window1 = vsg::Window::create(windowTraits1);

// Second window shares device
auto windowTraits2 = vsg::WindowTraits::create();
windowTraits2->windowTitle = "Secondary Window";
windowTraits2->device = window1->getOrCreateDevice();  // Share device
auto window2 = vsg::Window::create(windowTraits2);

// Both windows use same device, reducing resource overhead
auto device = window1->getOrCreateDevice();
std::cout << "Shared device: " << device.get() << std::endl;
std::cout << "Window1 device: " << window1->getOrCreateDevice().get() << std::endl;
std::cout << "Window2 device: " << window2->getOrCreateDevice().get() << std::endl;
```

### Queue Management

```cpp
auto device = window->getOrCreateDevice();

// Get queues by family index
auto graphicsQueue = device->getQueue(graphicsFamily);
auto presentQueue = device->getQueue(presentFamily);

// Get queue by type (VSG convenience method)
auto computeQueue = device->getQueue(VK_QUEUE_COMPUTE_BIT);
auto transferQueue = device->getQueue(VK_QUEUE_TRANSFER_BIT);

// Queue properties
std::cout << "Graphics queue family: " << graphicsQueue->queueFamilyIndex() << std::endl;
std::cout << "Graphics queue index: " << graphicsQueue->queueIndex() << std::endl;
```

## Swapchain and Surface Management

VSG provides comprehensive swapchain management with automatic format selection and fallback.

### Surface Format Configuration

```cpp
// Automatic format selection (recommended)
auto windowTraits = vsg::WindowTraits::create();
// VSG selects best available format

// Manual format specification
windowTraits->swapchainPreferences.surfaceFormat = {
    VK_FORMAT_B8G8R8A8_SRGB,           // sRGB format
    VK_COLOR_SPACE_SRGB_NONLINEAR_KHR  // sRGB color space
};

// Alternative formats
windowTraits->swapchainPreferences.surfaceFormat = {
    VK_FORMAT_B8G8R8A8_UNORM,          // Linear format
    VK_COLOR_SPACE_SRGB_NONLINEAR_KHR
};
```

### Present Mode Configuration

```cpp
auto windowTraits = vsg::WindowTraits::create();

// FIFO (V-Sync) - Guaranteed to be available
windowTraits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;

// Mailbox (Low latency) - Preferred for games
windowTraits->swapchainPreferences.presentMode = VK_PRESENT_MODE_MAILBOX_KHR;

// Immediate (No V-Sync) - Lowest latency, possible tearing
windowTraits->swapchainPreferences.presentMode = VK_PRESENT_MODE_IMMEDIATE_KHR;

// FIFO Relaxed - Allows late frames to bypass V-Sync
windowTraits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_RELAXED_KHR;
```

### Buffer Count Configuration

```cpp
auto windowTraits = vsg::WindowTraits::create();

// Double buffering
windowTraits->swapchainPreferences.imageCount = 2;

// Triple buffering (recommended for smooth animation)
windowTraits->swapchainPreferences.imageCount = 3;

// Automatic (let driver decide optimal count)
windowTraits->swapchainPreferences.imageCount = 0;
```

### Multi-Format Window Creation

```cpp
// Create windows for each supported format
auto instance = vsg::Instance::create();
auto physicalDevice = instance->getPhysicalDevice();
auto surface = vsg::Surface::create(instance, windowTraits);
auto swapChainSupport = vsg::querySwapChainSupport(physicalDevice->vk(), surface->vk());

for (const auto& format : swapChainSupport.formats) {
    auto localTraits = vsg::WindowTraits::create();
    localTraits->windowTitle = vsg::make_string(
        "Format: ", format.format, " ColorSpace: ", format.colorSpace
    );
    localTraits->swapchainPreferences.surfaceFormat = format;
    localTraits->x = windowIndex * 400;  // Offset windows
    
    auto window = vsg::Window::create(localTraits);
    if (window) {
        windows.push_back(window);
        windowIndex++;
    }
}
```

### Surface Capability Querying

```cpp
auto physicalDevice = window->getOrCreatePhysicalDevice();
auto surface = window->getOrCreateSurface();

// Query surface capabilities
auto capabilities = physicalDevice->getSurfaceCapabilities(surface);
std::cout << "Surface Capabilities:" << std::endl;
std::cout << "  Min image count: " << capabilities.minImageCount << std::endl;
std::cout << "  Max image count: " << capabilities.maxImageCount << std::endl;
std::cout << "  Current extent: " << capabilities.currentExtent.width 
          << "x" << capabilities.currentExtent.height << std::endl;
std::cout << "  Min extent: " << capabilities.minImageExtent.width 
          << "x" << capabilities.minImageExtent.height << std::endl;
std::cout << "  Max extent: " << capabilities.maxImageExtent.width 
          << "x" << capabilities.maxImageExtent.height << std::endl;

// Query supported formats
auto formats = physicalDevice->getSurfaceFormats(surface);
std::cout << "Supported Surface Formats:" << std::endl;
for (const auto& format : formats) {
    std::cout << "  Format: " << format.format 
              << ", Color Space: " << format.colorSpace << std::endl;
}

// Query present modes
auto presentModes = physicalDevice->getSurfacePresentModes(surface);
std::cout << "Supported Present Modes:" << std::endl;
for (auto mode : presentModes) {
    std::cout << "  ";
    switch (mode) {
        case VK_PRESENT_MODE_IMMEDIATE_KHR:
            std::cout << "IMMEDIATE"; break;
        case VK_PRESENT_MODE_MAILBOX_KHR:
            std::cout << "MAILBOX"; break;
        case VK_PRESENT_MODE_FIFO_KHR:
            std::cout << "FIFO"; break;
        case VK_PRESENT_MODE_FIFO_RELAXED_KHR:
            std::cout << "FIFO_RELAXED"; break;
        default:
            std::cout << "UNKNOWN"; break;
    }
    std::cout << std::endl;
}
```

## Context and Resource Management

VSG's `Context` class manages resource compilation, command pools, and GPU memory allocation.

### Basic Context Setup

```cpp
auto device = window->getOrCreateDevice();
auto queueFamily = device->getQueueFamily(VK_QUEUE_GRAPHICS_BIT);

// Create context
auto context = vsg::Context::create(device);
context->commandPool = vsg::CommandPool::create(device, queueFamily);
context->graphicsQueue = device->getQueue(queueFamily);
```

### Resource Requirement Collection

```cpp
// VSG automatically analyzes scene graph for resource requirements
vsg::CollectResourceRequirements collector;
sceneGraph->accept(collector);

// Access collected requirements
const auto& requirements = collector.requirements;
std::cout << "Resource Requirements:" << std::endl;
std::cout << "  Descriptor sets: " << requirements.externalNumDescriptorSets << std::endl;
std::cout << "  Vertex attributes: " << requirements.numVertexAttributes << std::endl;
std::cout << "  Vertex bindings: " << requirements.numVertexBindings << std::endl;

// Descriptor types needed
for (const auto& [type, count] : requirements.descriptorTypeMap) {
    std::cout << "  Descriptor type " << type << ": " << count << " descriptors" << std::endl;
}

// Reserve resources in context
context->reserve(requirements);
```

### Resource Compilation

```cpp
// Compile individual resources
auto descriptorImage = vsg::DescriptorImage::create(sampler, textureData);
descriptorImage->compile(*context);

// Compile entire scene graph
sceneGraph->accept(vsg::CompileTraversal(*context));

// Record command buffers
context->record();

// Alternative: Let viewer handle compilation
viewer->compile();
```

### Memory Allocation Strategies

```cpp
// Configure memory allocator
auto allocator = vsg::Allocator::create();

// Memory pools for different object types
allocator->setBlockSize(vsg::ALLOCATOR_AFFINITY_OBJECTS, 1024 * 1024);      // 1MB for general objects
allocator->setBlockSize(vsg::ALLOCATOR_AFFINITY_DATA, 4 * 1024 * 1024);     // 4MB for data arrays
allocator->setBlockSize(vsg::ALLOCATOR_AFFINITY_NODES, 512 * 1024);         // 512KB for scene graph nodes

// GPU memory allocation
auto bufferInfo = vsg::BufferInfo::create();
bufferInfo->size = dataSize;
bufferInfo->usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;

auto buffer = vsg::createBufferAndMemory(device, bufferInfo);
```

## Command Graph Creation

VSG's command graph system provides both high-level convenience and low-level control.

### Basic Command Graph

```cpp
// Simple command graph creation
auto commandGraph = vsg::createCommandGraphForView(window, camera, scene);
viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph});
```

### Custom Command Graph

```cpp
auto device = window->getOrCreateDevice();
auto queueFamily = device->getQueueFamily(VK_QUEUE_GRAPHICS_BIT);

// Create command graph
auto commandGraph = vsg::CommandGraph::create(device, queueFamily);

// Create render graph
auto renderGraph = vsg::RenderGraph::create(window);
renderGraph->addChild(vsg::View::create(camera, scene));

// Add render graph to command graph
commandGraph->addChild(renderGraph);

// Assign to viewer
viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph});
```

### Secondary Command Buffers

```cpp
// Create secondary command graph for shared rendering
auto secondaryCommandGraph = vsg::createSecondaryCommandGraphForView(
    window, camera, scene, 0  // subpass index
);

// Create ExecuteCommands for reusing secondary buffers
auto executeCommands1 = vsg::ExecuteCommands::create();
executeCommands1->connect(secondaryCommandGraph);

auto executeCommands2 = vsg::ExecuteCommands::create();
executeCommands2->connect(secondaryCommandGraph);

// Create primary command graphs that execute secondary commands
auto primaryCommandGraph1 = vsg::createCommandGraphForView(
    window1, camera, sceneWithExecuteCommands1, 
    VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS
);

auto primaryCommandGraph2 = vsg::createCommandGraphForView(
    window2, camera, sceneWithExecuteCommands2, 
    VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS
);

// All command graphs share the secondary command buffer
viewer->assignRecordAndSubmitTaskAndPresentation({
    secondaryCommandGraph, 
    primaryCommandGraph1, 
    primaryCommandGraph2
});
```

### Multi-Window Command Sharing

```cpp
// Efficient multi-window rendering with shared commands
if (useExecuteCommands) {
    auto secondaryCommandGraph = vsg::createSecondaryCommandGraphForView(
        window1, camera, scene, 0);
    
    auto scenegraph1 = vsg::Group::create();
    auto pass1 = vsg::ExecuteCommands::create();
    pass1->connect(secondaryCommandGraph);
    scenegraph1->addChild(pass1);
    
    auto scenegraph2 = vsg::Group::create();
    auto pass2 = vsg::ExecuteCommands::create();
    pass2->connect(secondaryCommandGraph);
    scenegraph2->addChild(pass2);
    
    auto commandGraph1 = vsg::createCommandGraphForView(window1, camera, scenegraph1, 
                                                       VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
    auto commandGraph2 = vsg::createCommandGraphForView(window2, camera, scenegraph2, 
                                                       VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
    
    viewer->assignRecordAndSubmitTaskAndPresentation({
        secondaryCommandGraph, commandGraph1, commandGraph2
    });
} else {
    // Separate command graphs (more memory, less sharing)
    auto commandGraph1 = vsg::createCommandGraphForView(window1, camera, scene);
    auto commandGraph2 = vsg::createCommandGraphForView(window2, camera, scene);
    
    viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph1, commandGraph2});
}
```

## Advanced Configuration and Extensions

### Custom Extension Loading

```cpp
// Check for line rasterization extension
auto windowTraits = vsg::WindowTraits::create();
windowTraits->vulkanVersion = VK_API_VERSION_1_1;
windowTraits->deviceExtensionNames.push_back(VK_EXT_LINE_RASTERIZATION_EXTENSION_NAME);

auto window = vsg::Window::create(windowTraits);
auto physicalDevice = window->getOrCreatePhysicalDevice();

if (!physicalDevice->supportsDeviceExtension(VK_EXT_LINE_RASTERIZATION_EXTENSION_NAME)) {
    std::cout << "Line Rasterization Extension not supported" << std::endl;
    return 1;
}

// Query extension features
auto lineFeatures = physicalDevice->getFeatures<
    VkPhysicalDeviceLineRasterizationFeaturesEXT,
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_LINE_RASTERIZATION_FEATURES_EXT
>();

std::cout << "Line Rasterization Features:" << std::endl;
std::cout << "  Rectangular lines: " << lineFeatures.rectangularLines << std::endl;
std::cout << "  Bresenham lines: " << lineFeatures.bresenhamLines << std::endl;
std::cout << "  Smooth lines: " << lineFeatures.smoothLines << std::endl;
std::cout << "  Stippled lines: " << lineFeatures.stippledRectangularLines << std::endl;

// Enable extension features
auto deviceFeatures = vsg::DeviceFeatures::create();
auto& requestedLineFeatures = deviceFeatures->get<
    VkPhysicalDeviceLineRasterizationFeaturesEXT,
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_LINE_RASTERIZATION_FEATURES_EXT
>();

if (lineFeatures.stippledRectangularLines) {
    requestedLineFeatures.stippledRectangularLines = VK_TRUE;
}

windowTraits->deviceFeatures = deviceFeatures;
```

### Advanced Present Mode Configuration

```cpp
// Performance-oriented present mode selection
void configurePresentMode(vsg::ref_ptr<vsg::WindowTraits> traits, const std::string& mode) {
    if (mode == "immediate") {
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_IMMEDIATE_KHR;
        std::cout << "Using IMMEDIATE present mode (lowest latency, possible tearing)" << std::endl;
    } else if (mode == "mailbox") {
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_MAILBOX_KHR;
        std::cout << "Using MAILBOX present mode (low latency, no tearing)" << std::endl;
    } else if (mode == "fifo") {
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        std::cout << "Using FIFO present mode (V-Sync)" << std::endl;
    } else if (mode == "fifo-relaxed") {
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_RELAXED_KHR;
        std::cout << "Using FIFO_RELAXED present mode (adaptive V-Sync)" << std::endl;
    }
}

// Runtime present mode switching (requires swapchain recreation)
void switchPresentMode(vsg::ref_ptr<vsg::Window> window, VkPresentModeKHR newMode) {
    auto traits = window->traits();
    if (traits->swapchainPreferences.presentMode != newMode) {
        traits->swapchainPreferences.presentMode = newMode;
        window->clear();  // Force swapchain recreation on next frame
    }
}
```

### Multi-GPU Configuration

```cpp
// Check if multiple devices are available
if (screensToUse.size() > vsg::Device::maxNumDevices()) {
    std::cout << "VSG built with VSG_MAX_DEVICES = " << VSG_MAX_DEVICES
              << ", insufficient for " << screensToUse.size() << " screens" << std::endl;
    return 1;
}

// Create window for each screen/GPU
std::vector<vsg::ref_ptr<vsg::Window>> windows;
for (size_t i = 0; i < screensToUse.size(); ++i) {
    auto localTraits = vsg::WindowTraits::create(*windowTraits);
    localTraits->screenNum = screensToUse[i];
    localTraits->windowTitle = vsg::make_string("Screen ", screensToUse[i]);
    
    auto window = vsg::Window::create(localTraits);
    if (window) {
        windows.push_back(window);
        viewer->addWindow(window);
        
        // Each window gets its own command graph
        auto commandGraph = vsg::createCommandGraphForView(window, camera, scene);
        commandGraphs.push_back(commandGraph);
    }
}

viewer->assignRecordAndSubmitTaskAndPresentation(commandGraphs);
```

### Advanced Buffer Configuration

```cpp
// Custom buffer management
class AdvancedBufferConfig {
public:
    static void configureBuffering(vsg::ref_ptr<vsg::WindowTraits> traits, 
                                  const std::string& mode) {
        if (mode == "single") {
            traits->swapchainPreferences.imageCount = 1;
            std::cout << "Single buffering (not recommended)" << std::endl;
        } else if (mode == "double") {
            traits->swapchainPreferences.imageCount = 2;
            std::cout << "Double buffering" << std::endl;
        } else if (mode == "triple") {
            traits->swapchainPreferences.imageCount = 3;
            std::cout << "Triple buffering (recommended)" << std::endl;
        } else if (mode == "auto") {
            traits->swapchainPreferences.imageCount = 0;  // Let driver decide
            std::cout << "Automatic buffer count selection" << std::endl;
        }
    }
    
    static void optimizeForLatency(vsg::ref_ptr<vsg::WindowTraits> traits) {
        traits->swapchainPreferences.imageCount = 2;  // Double buffer for low latency
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_MAILBOX_KHR;
        std::cout << "Configured for low latency" << std::endl;
    }
    
    static void optimizeForSmoothness(vsg::ref_ptr<vsg::WindowTraits> traits) {
        traits->swapchainPreferences.imageCount = 3;  // Triple buffer for smoothness
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        std::cout << "Configured for smooth animation" << std::endl;
    }
};
```

## Debugging and Validation

### Validation Layer Setup

```cpp
auto windowTraits = vsg::WindowTraits::create();

// Enable validation layers
windowTraits->debugLayer = true;
windowTraits->apiDumpLayer = false;           // Disable API logging (verbose)
windowTraits->synchronizationLayer = false;   // Disable sync validation (performance impact)

// Manual validation layer setup
vsg::Names instanceExtensions;
vsg::Names requestedLayers;

instanceExtensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
requestedLayers.push_back("VK_LAYER_KHRONOS_validation");

// Optional additional layers
if (enableApiDump) {
    requestedLayers.push_back("VK_LAYER_LUNARG_api_dump");
}

if (enableSyncValidation) {
    requestedLayers.push_back("VK_LAYER_KHRONOS_synchronization2");
}

// Validate layer availability
vsg::Names validatedLayers = vsg::validateInstancelayerNames(requestedLayers);
if (validatedLayers.size() != requestedLayers.size()) {
    std::cout << "Some validation layers not available" << std::endl;
}
```

### GPU Debug Annotations

```cpp
// Enable GPU debugging
windowTraits->debugUtils = true;

// Create GPU annotation instrumentation
auto gpu_instrumentation = vsg::GpuAnnotation::create();
gpu_instrumentation->labelType = vsg::GpuAnnotation::SourceLocation_name;

// Use with viewer
viewer->instrumentation = gpu_instrumentation;

// Manual debug labels in code
auto commandBuffer = context->getCommandBuffer();
vsg::DebugLabel::create("Scene Rendering")->record(*commandBuffer);
// ... rendering commands ...
vsg::EndDebugLabel::create()->record(*commandBuffer);
```

### Error Handling Patterns

```cpp
// VSG exception handling
try {
    auto window = vsg::Window::create(windowTraits);
    if (!window) {
        throw std::runtime_error("Failed to create window");
    }
    
    viewer->compile();
    
    while (viewer->advanceToNextFrame()) {
        viewer->handleEvents();
        viewer->update();
        viewer->recordAndSubmit();
        viewer->present();
    }
}
catch (const vsg::Exception& ve) {
    std::cerr << "VSG Exception: " << ve.message 
              << " (VkResult: " << ve.result << ")" << std::endl;
    return 1;
}
catch (const std::exception& e) {
    std::cerr << "Standard Exception: " << e.what() << std::endl;
    return 1;
}
```

### Diagnostic Information

```cpp
// Print comprehensive system information
void printSystemInfo(vsg::ref_ptr<vsg::Window> window) {
    auto instance = window->getOrCreateInstance();
    auto physicalDevice = window->getOrCreatePhysicalDevice();
    auto device = window->getOrCreateDevice();
    
    std::cout << "=== VSG System Information ===" << std::endl;
    
    // Instance information
    std::cout << "Instance layers:" << std::endl;
    for (const auto& layer : instance->layers) {
        std::cout << "  " << layer << std::endl;
    }
    
    std::cout << "Instance extensions:" << std::endl;
    for (const auto& extension : instance->extensions) {
        std::cout << "  " << extension << std::endl;
    }
    
    // Physical device information
    auto properties = physicalDevice->getProperties();
    std::cout << "Physical Device: " << properties.deviceName << std::endl;
    std::cout << "Driver Version: " << properties.driverVersion << std::endl;
    std::cout << "API Version: " << VK_VERSION_MAJOR(properties.apiVersion)
              << "." << VK_VERSION_MINOR(properties.apiVersion)
              << "." << VK_VERSION_PATCH(properties.apiVersion) << std::endl;
    
    // Device extensions
    std::cout << "Device extensions:" << std::endl;
    for (const auto& extension : device->extensions) {
        std::cout << "  " << extension << std::endl;
    }
    
    // Memory information
    auto memProps = physicalDevice->getMemoryProperties();
    std::cout << "Memory Heaps:" << std::endl;
    for (uint32_t i = 0; i < memProps.memoryHeapCount; ++i) {
        std::cout << "  Heap " << i << ": " << (memProps.memoryHeaps[i].size >> 20) << " MB";
        if (memProps.memoryHeaps[i].flags & VK_MEMORY_HEAP_DEVICE_LOCAL_BIT) {
            std::cout << " (Device Local)";
        }
        std::cout << std::endl;
    }
}
```

## Common Patterns and Best Practices

### Resource Lifecycle Management

```cpp
class VSGApplication {
public:
    bool initialize() {
        // 1. Create window traits with validation
        auto windowTraits = vsg::WindowTraits::create();
        windowTraits->windowTitle = "VSG Application";
        windowTraits->debugLayer = true;
        windowTraits->samples = VK_SAMPLE_COUNT_4_BIT;
        
        // 2. Create window (initializes Vulkan)
        window = vsg::Window::create(windowTraits);
        if (!window) {
            std::cerr << "Failed to create window" << std::endl;
            return false;
        }
        
        // 3. Create viewer
        viewer = vsg::Viewer::create();
        viewer->addWindow(window);
        
        // 4. Setup scene
        scene = createScene();
        camera = createCamera();
        
        // 5. Create command graph
        auto commandGraph = vsg::createCommandGraphForView(window, camera, scene);
        viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph});
        
        // 6. Compile resources
        viewer->compile();
        
        return true;
    }
    
    void run() {
        while (viewer->advanceToNextFrame()) {
            viewer->handleEvents();
            updateScene();
            viewer->update();
            viewer->recordAndSubmit();
            viewer->present();
        }
    }
    
    void cleanup() {
        // VSG automatically handles cleanup through reference counting
        // No manual Vulkan cleanup required
        viewer = nullptr;
        window = nullptr;
        scene = nullptr;
        camera = nullptr;
    }
    
private:
    vsg::ref_ptr<vsg::Window> window;
    vsg::ref_ptr<vsg::Viewer> viewer;
    vsg::ref_ptr<vsg::Node> scene;
    vsg::ref_ptr<vsg::Camera> camera;
};
```

### Performance Best Practices

```cpp
class PerformanceOptimizedSetup {
public:
    static vsg::ref_ptr<vsg::WindowTraits> createOptimizedTraits() {
        auto traits = vsg::WindowTraits::create();
        
        // Optimize for performance
        traits->swapchainPreferences.imageCount = 3;  // Triple buffer
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_MAILBOX_KHR;  // Low latency
        
        // Enable performance features
        auto deviceFeatures = vsg::DeviceFeatures::create();
        deviceFeatures->get().multiDrawIndirect = VK_TRUE;  // For batching
        deviceFeatures->get().drawIndirectFirstInstance = VK_TRUE;
        traits->deviceFeatures = deviceFeatures;
        
        // Disable validation in release builds
        #ifdef NDEBUG
        traits->debugLayer = false;
        traits->apiDumpLayer = false;
        #else
        traits->debugLayer = true;
        #endif
        
        return traits;
    }
    
    static void optimizeContext(vsg::ref_ptr<vsg::Context> context) {
        // Pre-allocate command buffers
        context->commandPool->allocateCommandBuffers(4);
        
        // Reserve descriptor pools
        vsg::ResourceRequirements requirements;
        requirements.externalNumDescriptorSets = 1000;
        requirements.descriptorTypeMap[VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER] = 500;
        requirements.descriptorTypeMap[VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER] = 500;
        context->reserve(requirements);
    }
};
```

### Error Recovery Patterns

```cpp
class RobustVSGSetup {
public:
    static vsg::ref_ptr<vsg::Window> createWindowWithFallback(
        vsg::ref_ptr<vsg::WindowTraits> preferredTraits) {
        
        // Try preferred settings first
        auto window = vsg::Window::create(preferredTraits);
        if (window) return window;
        
        // Fallback 1: Reduce MSAA
        auto fallback1 = vsg::WindowTraits::create(*preferredTraits);
        fallback1->samples = VK_SAMPLE_COUNT_1_BIT;
        window = vsg::Window::create(fallback1);
        if (window) {
            std::cout << "Fallback: Disabled MSAA" << std::endl;
            return window;
        }
        
        // Fallback 2: Change present mode
        auto fallback2 = vsg::WindowTraits::create(*fallback1);
        fallback2->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        window = vsg::Window::create(fallback2);
        if (window) {
            std::cout << "Fallback: Using FIFO present mode" << std::endl;
            return window;
        }
        
        // Fallback 3: Basic setup
        auto basicTraits = vsg::WindowTraits::create();
        basicTraits->windowTitle = preferredTraits->windowTitle;
        basicTraits->width = preferredTraits->width;
        basicTraits->height = preferredTraits->height;
        window = vsg::Window::create(basicTraits);
        if (window) {
            std::cout << "Fallback: Using basic configuration" << std::endl;
            return window;
        }
        
        return nullptr;  // Complete failure
    }
};
```

### Configuration Templates

```cpp
namespace VSGPresets {
    // Development preset - maximum validation
    vsg::ref_ptr<vsg::WindowTraits> development() {
        auto traits = vsg::WindowTraits::create();
        traits->debugLayer = true;
        traits->synchronizationLayer = true;
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        traits->samples = VK_SAMPLE_COUNT_1_BIT;  // Faster validation
        return traits;
    }
    
    // Gaming preset - low latency
    vsg::ref_ptr<vsg::WindowTraits> gaming() {
        auto traits = vsg::WindowTraits::create();
        traits->debugLayer = false;
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_MAILBOX_KHR;
        traits->swapchainPreferences.imageCount = 2;  // Lower latency
        traits->samples = VK_SAMPLE_COUNT_4_BIT;
        return traits;
    }
    
    // Visualization preset - high quality
    vsg::ref_ptr<vsg::WindowTraits> visualization() {
        auto traits = vsg::WindowTraits::create();
        traits->debugLayer = false;
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        traits->swapchainPreferences.imageCount = 3;
        traits->samples = VK_SAMPLE_COUNT_8_BIT;  // High quality
        
        auto deviceFeatures = vsg::DeviceFeatures::create();
        deviceFeatures->get().samplerAnisotropy = VK_TRUE;
        traits->deviceFeatures = deviceFeatures;
        
        return traits;
    }
    
    // Mobile preset - power efficient
    vsg::ref_ptr<vsg::WindowTraits> mobile() {
        auto traits = vsg::WindowTraits::create();
        traits->debugLayer = false;
        traits->swapchainPreferences.presentMode = VK_PRESENT_MODE_FIFO_KHR;
        traits->swapchainPreferences.imageCount = 2;
        traits->samples = VK_SAMPLE_COUNT_1_BIT;  // Power efficient
        return traits;
    }
}
```

---

This comprehensive guide covers all aspects of VSG's Vulkan initialization and management. VSG provides an elegant abstraction over Vulkan's complexity while maintaining access to advanced features when needed. The library's design patterns ensure both ease of use for common scenarios and detailed control for specialized applications.

For more specific examples and advanced techniques, refer to the individual VSG example applications and the detailed API documentation.