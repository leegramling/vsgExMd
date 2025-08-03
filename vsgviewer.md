# vsgviewer

## Overview

The `vsgviewer` example is a comprehensive 3D model viewer that demonstrates advanced VSG features including multi-threading, profiling, animation playback, and extensive configuration options. It serves as a feature-rich reference implementation for VSG applications.

## Key VSG Features Used

- **Multi-format loading** - Supports 3D models and images via vsgXchange
- **Animation support** - Automatic animation playback
- **Camera animations** - Path-based camera movement
- **Multi-threading** - Optional multi-threaded rendering
- **Profiling** - CPU/GPU performance profiling
- **GPU annotations** - Debug labeling for tools like RenderDoc
- **LOD management** - PagedLOD loading and management
- **Ellipsoid support** - For geospatial/planetary rendering
- **Resource hints** - Memory and resource optimization
- **Shader debugging** - Debug info generation
- **Extensive configuration** - Command-line control

## What the Example Demonstrates

1. **Flexible File Loading**
   - Loads multiple files (3D models or images)
   - Creates texture quads for image files
   - Supports all vsgXchange formats

2. **Advanced Rendering Options**
   - Swapchain configuration (double/triple buffering)
   - Present modes (immediate, FIFO, mailbox)
   - Depth formats and color spaces
   - Multi-sample anti-aliasing

3. **Performance Features**
   - Multi-threaded rendering with CPU affinity
   - GPU and CPU profiling
   - Frame rate reporting
   - Memory statistics

4. **Debugging Support**
   - Vulkan validation layers
   - API dump layer
   - GPU annotations
   - Shader debug info

## Code Highlights

```cpp
// Multi-format file loading
for (int i = 1; i < argc; ++i)
{
    auto object = vsg::read(filename, options);
    if (auto node = object.cast<vsg::Node>())
    {
        group->addChild(node);
    }
    else if (auto data = object.cast<vsg::Data>())
    {
        // Create texture quad for images
        if (auto textureGeometry = createTextureQuad(data, options))
        {
            group->addChild(textureGeometry);
        }
    }
}

// Animation playback
if (autoPlay)
{
    auto animationGroups = vsg::visit<vsg::FindAnimations>(vsg_scene).animationGroups;
    for (auto ag : animationGroups)
    {
        if (!ag->animations.empty()) 
            viewer->animationManager->play(ag->animations.front());
    }
}

// Multi-threading setup
if (multiThreading)
{
    viewer->setupThreading();
    
    // Set CPU affinity
    for (auto& thread : viewer->threads)
    {
        if (thread.joinable() && cpu_itr != affinity.cpus.end())
        {
            vsg::setAffinity(thread, vsg::Affinity(*cpu_itr++));
        }
    }
}

// Profiling setup
if (arguments.read({"--profiler", "--pr"}))
{
    auto settings = vsg::Profiler::Settings::create();
    arguments.read("--cpu", settings->cpu_instrumentation_level);
    arguments.read("--gpu", settings->gpu_instrumentation_level);
    instrumentation = vsg::Profiler::create(settings);
    viewer->assignInstrumentation(instrumentation);
}

// GPU annotations
if (arguments.read({"--gpu-annotation", "--ga"}))
{
    windowTraits->debugUtils = true;
    auto gpu_instrumentation = vsg::GpuAnnotation::create();
    gpu_instrumentation->labelType = vsg::GpuAnnotation::SourceLocation_name;
    instrumentation = gpu_instrumentation;
}
```

## Running the Example

```bash
# Basic usage
./vsgviewer model.vsgt

# Multiple files
./vsgviewer model1.obj model2.gltf texture.png

# Performance testing
./vsgviewer model.vsgt --test              # Fullscreen immediate mode
./vsgviewer model.vsgt --fps               # Report frame rate
./vsgviewer model.vsgt --profiler --cpu 2  # Enable profiling

# Multi-threading with CPU affinity
./vsgviewer model.vsgt --mt -c 0 -c 1 -c 2

# Debugging
./vsgviewer model.vsgt --debug             # Validation layers
./vsgviewer model.vsgt --api               # API dump
./vsgviewer model.vsgt --gpu-annotation    # GPU debug labels

# Window configuration
./vsgviewer model.vsgt --window 1920 1080
./vsgviewer model.vsgt --fullscreen
./vsgviewer model.vsgt --MAILBOX           # Mailbox present mode
./vsgviewer model.vsgt --samples 8         # 8x MSAA

# Animation control
./vsgviewer animated.gltf --no-auto-play   # Don't auto-play animations
./vsgviewer model.vsgt -p camera_path.vsgt # Camera animation path

# Advanced options
./vsgviewer model.vsgt --load-levels 3     # Preload LOD levels
./vsgviewer model.vsgt --maxPagedLOD 100   # Max paged LODs
./vsgviewer model.vsgt --sRGB              # sRGB framebuffer
./vsgviewer model.vsgt --log-level 3       # Verbose logging
```

## Key Takeaways

- vsgviewer demonstrates how to build a production-ready VSG application
- Extensive command-line options provide runtime configuration
- Multi-threading can significantly improve performance
- Profiling tools help identify performance bottlenecks
- GPU annotations aid in debugging with graphics debuggers
- The viewer handles both 3D models and images
- Animation support includes both object and camera animations
- Resource hints can optimize memory usage
- The example shows best practices for error handling and robustness