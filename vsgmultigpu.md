# vsgmultigpu

## Overview

The `vsgmultigpu` example demonstrates multi-GPU rendering in VSG, showcasing how to create multiple windows across different screens/displays and coordinate rendering between them. It supports both power wall configurations (adjacent screens) and surround configurations (screens arranged around viewer).

## Key VSG Features Used

- **Multiple windows** - Create windows on different screens/displays
- **Multi-threading** - Parallel rendering across multiple GPUs
- **RelativeProjection** - Offset projections for power wall setups
- **RelativeViewMatrix** - Rotated views for surround configurations
- **CPU affinity** - Thread pinning for performance optimization
- **Device limits** - Respect VSG_MAX_DEVICES compile-time limit
- **Shared scene graphs** - Option to share or duplicate scene data

## What the Example Demonstrates

1. **Multi-Screen Rendering**
   - Create windows on multiple screens simultaneously
   - Coordinate camera views across displays
   - Handle different display configurations

2. **Power Wall Configuration**
   - Multiple adjacent screens showing continuous view
   - Horizontal projection offsets for seamless display
   - Large format visualization support

3. **Surround Configuration**
   - Screens arranged around viewer position
   - Rotated camera views for 360-degree coverage
   - Immersive visualization environments

4. **Performance Optimization**
   - Multi-threaded rendering across GPUs
   - CPU affinity control for thread performance
   - Optional scene graph sharing vs duplication

## Code Highlights

```cpp
// Multi-screen window creation
std::vector<int> screensToUse;
int screen = -1;
while (arguments.read({"--screen", "-s"}, screen))
{
    screensToUse.push_back(screen);
}

// Check device limits
if (screensToUse.size() > vsg::Device::maxNumDevices())
{
    std::cout << "VulkanSceneGraph built with VSG_MAX_DEVICES = " << VSG_MAX_DEVICES
              << ", which is insufficient for the number of screens desired." << std::endl;
    return 1;
}

// Create window for each screen
for (size_t i = 0; i < screensToUse.size(); ++i)
{
    int screenNum = screensToUse[i];
    
    auto local_windowTraits = vsg::WindowTraits::create(*windowTraits);
    local_windowTraits->screenNum = screenNum;
    
    auto window = vsg::Window::create(local_windowTraits);
    
    vsg::ref_ptr<vsg::Camera> camera;
    if (powerWall)
    {
        // Power wall: horizontal projection offset
        auto relative_perspective = vsg::RelativeProjection::create(
            vsg::translate(double(numScreens - 1) - 2.0 * double(i), 0.0, 0.0), 
            perspective
        );
        camera = vsg::Camera::create(relative_perspective, lookAt, viewport);
    }
    else
    {
        // Surround: rotated views around viewer
        double fovY = 30.0;
        double fovX = atan(tan(vsg::radians(fovY) * 0.5) * aspectRatio) * 2.0;
        double angle = fovX * (double(i) - double(numScreens - 1) / 2.0);
        
        auto relative_view = vsg::RelativeViewMatrix::create(
            vsg::rotate(angle, 0.0, 1.0, 0.0), 
            lookAt
        );
        camera = vsg::Camera::create(perspective, relative_view, viewport);
    }
    
    // Choose between shared or independent scene graphs
    auto local_scene = sharedScene ? vsg_scene : createScene(filename, options);
    
    viewer->assignRecordAndSubmitTaskAndPresentation(
        {vsg::createCommandGraphForView(window, camera, local_scene)}
    );
    viewer->addWindow(window);
}

// Setup multi-threading
if (multiThreading)
{
    viewer->setupThreading();
    
    // CPU affinity configuration
    if (affinity)
    {
        auto cpu_itr = affinity.cpus.begin();
        
        // Set main thread affinity
        if (cpu_itr != affinity.cpus.end())
        {
            vsg::setAffinity(vsg::Affinity(*cpu_itr++));
        }
        
        // Set worker thread affinities
        for (auto& thread : viewer->threads)
        {
            if (thread.joinable() && cpu_itr != affinity.cpus.end())
            {
                vsg::setAffinity(thread, vsg::Affinity(*cpu_itr++));
            }
        }
    }
}

// Unified trackball camera across all windows
auto trackball = vsg::Trackball::create(master_camera);

uint32_t totalWidth = 0;
uint32_t maxHeight = 0;
for (auto& window : viewer->windows())
{
    trackball->addWindow(window, vsg::ivec2(totalWidth, 0));
    totalWidth += window->extent2D().width;
    if (window->extent2D().height > maxHeight) 
        maxHeight = window->extent2D().height;
}

// Set combined viewport for trackball calculations
master_camera->viewportState = vsg::ViewportState::create(0, 0, totalWidth, maxHeight);
```

## Running the Example

```bash
# Basic usage - single screen
./vsgmultigpu model.vsgt

# Multi-screen setup
./vsgmultigpu model.vsgt --screen 0 --screen 1 --screen 2

# Power wall configuration (adjacent screens)
./vsgmultigpu model.vsgt --screen 0 --screen 1 --power-wall

# Enable multi-threading
./vsgmultigpu model.vsgt --screen 0 --screen 1 --mt

# CPU affinity control
./vsgmultigpu model.vsgt --screen 0 --screen 1 --mt --cpu 0 --cpu 2 --cpu 4

# Scene graph options
./vsgmultigpu model.vsgt --screen 0 --screen 1 --no-shared  # Duplicate scene per window

# Window configuration
./vsgmultigpu model.vsgt --screen 0 --screen 1 --fullscreen
./vsgmultigpu model.vsgt --screen 0 --screen 1 --window 1920 1080

# Animation support
./vsgmultigpu model.vsgt --screen 0 --screen 1 -p camera_path.vsgb
```

### Display Configuration Examples

**Power Wall (3 screens horizontally)**:
```bash
./vsgmultigpu model.vsgt --screen 0 --screen 1 --screen 2 --power-wall --fullscreen
```

**Surround Setup (3 screens around viewer)**:
```bash
./vsgmultigpu model.vsgt --screen 0 --screen 1 --screen 2 --fullscreen
```

**Cave/Immersive Environment**:
```bash
./vsgmultigpu model.vsgt --screen 0 --screen 1 --screen 2 --screen 3 --fullscreen --mt
```

## Configuration Requirements

### Compile-Time Limits

VSG must be built with sufficient `VSG_MAX_DEVICES`:
```cmake
cmake -DVSG_MAX_DEVICES=8 ..
```

### Hardware Requirements

- Multiple displays/screens
- Multiple GPUs (recommended for performance)
- Sufficient system memory for scene duplication (if not shared)

### Software Requirements

- Display drivers supporting multiple screens
- Vulkan drivers for each GPU
- Proper screen configuration in OS

## Key Takeaways

- VSG supports sophisticated multi-display configurations out of the box
- Power wall and surround configurations require different camera setups
- Multi-threading significantly improves performance with multiple GPUs
- CPU affinity can optimize thread performance on NUMA systems
- Scene graph sharing reduces memory usage but may limit GPU independence
- Camera coordination across displays requires careful projection/view calculations
- The trackball handler can span multiple windows for unified navigation
- VSG_MAX_DEVICES compile-time limit must accommodate your display count
- This approach scales to large visualization installations and VR/AR systems