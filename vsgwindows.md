# vsgwindows

## Overview

The `vsgwindows` example demonstrates how to create and manage multiple windows in a VSG application. It shows window creation, device sharing between windows, per-window event handling, and independent camera controls for each window.

## Key VSG Features Used

- **Multiple windows** - Creating and managing multiple render windows
- **Device sharing** - Sharing Vulkan devices between windows
- **Per-window cameras** - Independent camera for each window
- **Per-window event handling** - Window-specific input processing
- **vsg::CommandGraph** - Per-window command graphs
- **vsg::RenderGraph** - Per-window render configuration
- **Multi-threading** - Optional multi-threaded rendering
- **Clear values** - Custom background colors per window

## What the Example Demonstrates

1. **Window Creation Options**
   - Shared device mode (default) - windows share same Vulkan instance/device
   - Separate device mode - each window has its own device
   - Different window sizes and properties

2. **Independent Window Control**
   - Each window has its own camera
   - Separate trackball controls per window
   - Different scenes can be loaded in each window

3. **Event Handling**
   - Window-specific event handlers
   - Coordinated close handling across windows

4. **Rendering Configuration**
   - Per-window command graphs
   - Different clear colors
   - Independent render graphs

## Code Highlights

```cpp
// Create two windows with different properties
auto windowTraits = vsg::WindowTraits::create();
windowTraits->windowTitle = "Multiple Windows - first window";
windowTraits->width = 800;
windowTraits->height = 600;

auto windowTraits2 = vsg::WindowTraits::create();
windowTraits2->windowTitle = "Multiple Windows - second window";
windowTraits2->width = 640;
windowTraits2->height = 480;

// Option 1: Share device between windows (default)
if (!separateDevices)
{
    windowTraits2->device = window1->getOrCreateDevice();
    std::cout << "Sharing vsg::Instance and vsg::Device between windows." << std::endl;
}

// Create windows
auto window1 = vsg::Window::create(windowTraits);
auto window2 = vsg::Window::create(windowTraits2);

// Add windows to viewer
viewer->addWindow(window1);
viewer->addWindow(window2);

// Create separate cameras for each window
auto main_camera = createCameraForScene(scenegraph, 0, 0, 
    window1->extent2D().width, window1->extent2D().height);
auto secondary_camera = createCameraForScene(scenegraph2, 0, 0, 
    window2->extent2D().width, window2->extent2D().height);

// Window-specific event handlers
auto main_trackball = vsg::Trackball::create(main_camera);
main_trackball->addWindow(window1);  // Only responds to window1 events
viewer->addEventHandler(main_trackball);

auto secondary_trackball = vsg::Trackball::create(secondary_camera);
secondary_trackball->addWindow(window2);  // Only responds to window2 events
viewer->addEventHandler(secondary_trackball);

// Create render graphs with different settings
auto main_RenderGraph = vsg::RenderGraph::create(window1, main_view);
auto secondary_RenderGraph = vsg::RenderGraph::create(window2, secondary_view);

// Custom clear color for second window
secondary_RenderGraph->clearValues[0].color = 
    vsg::sRGB_to_linear(0.2f, 0.2f, 0.2f, 1.0f);

// Create command graphs for each window
auto commandGraph1 = vsg::CommandGraph::create(window1);
commandGraph1->addChild(main_RenderGraph);

auto commandGraph2 = vsg::CommandGraph::create(window2);
commandGraph2->addChild(secondary_RenderGraph);

// Assign both command graphs to viewer
viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph1, commandGraph2});
```

## Running the Example

```bash
# Basic two-window setup with same model
./vsgwindows model.vsgt

# Different models in each window
./vsgwindows model1.vsgt model2.vsgt

# Custom window sizes
./vsgwindows model.vsgt --window 1920 1080 --window2 800 600

# Separate devices (no sharing)
./vsgwindows model.vsgt --no-shared-window

# Multi-threaded rendering
./vsgwindows model.vsgt --mt

# Performance testing
./vsgwindows model.vsgt --test --fps

# Debugging
./vsgwindows model.vsgt --debug
./vsgwindows model.vsgt --profiler
```

### Window Controls

- **Window 1**: Use mouse in first window to control its camera
- **Window 2**: Use mouse in second window to control its camera
- **ESC or Close**: Closing any window exits the application
- **Independent**: Each window's camera moves independently

## Key Takeaways

- VSG supports multiple windows with shared or separate Vulkan devices
- Each window can have its own camera, scene, and rendering settings
- Event handlers can be window-specific using the addWindow() method
- Device sharing is more efficient but separate devices provide isolation
- Command graphs organize rendering commands per window
- The viewer manages all windows in a single render loop
- Multi-window applications are useful for CAD tools, editors, and comparison views
- Clear values and other render settings can be customized per window