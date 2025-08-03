# vsghelloworld

## Overview

The `vsghelloworld` example is the fundamental VSG application template. It demonstrates the minimal code needed to load a 3D scene, create a window, set up a camera, and render with Vulkan. This is the starting point for most VSG applications.

## Key VSG Features Used

- **vsg::Viewer** - Main application class managing the render loop
- **vsg::Window** - Platform-independent window creation
- **vsg::Camera** - View and projection matrices
- **vsg::read_cast<>** - Loading scene files
- **vsg::CommandGraph** - Vulkan command buffer management
- **vsg::Trackball** - Camera manipulation
- **vsg::ComputeBounds** - Automatic scene bounds calculation
- **Event handlers** - User input processing

## What the Example Demonstrates

1. **Basic Application Structure**
   - Command line parsing
   - File loading with vsgXchange
   - Window creation
   - Main render loop

2. **Camera Setup**
   - Automatic camera positioning based on scene bounds
   - Support for both standard and ellipsoid (Earth) models
   - Perspective projection configuration

3. **Event Handling**
   - Close handler for window close and ESC key
   - Trackball for mouse-based camera control

4. **Render Loop**
   - Event processing
   - Scene updates
   - Command buffer recording and submission
   - Frame presentation

## Code Highlights

```cpp
// Load a scene file
auto options = vsg::Options::create(vsgXchange::all::create());
auto vsg_scene = vsg::read_cast<vsg::Node>(filename, options);

// Create window and viewer
auto windowTraits = vsg::WindowTraits::create();
windowTraits->windowTitle = "Hello World";
auto viewer = vsg::Viewer::create();
auto window = vsg::Window::create(windowTraits);
viewer->addWindow(window);

// Calculate scene bounds for camera positioning
vsg::ComputeBounds computeBounds;
vsg_scene->accept(computeBounds);
vsg::dvec3 centre = (computeBounds.bounds.min + computeBounds.bounds.max) * 0.5;
double radius = vsg::length(computeBounds.bounds.max - computeBounds.bounds.min) * 0.6;

// Set up camera
auto lookAt = vsg::LookAt::create(
    centre + vsg::dvec3(0.0, -radius * 3.5, 0.0),  // eye
    centre,                                         // center
    vsg::dvec3(0.0, 0.0, 1.0)                      // up
);
auto perspective = vsg::Perspective::create(
    30.0,                                          // field of view
    aspectRatio,                                   // aspect ratio
    nearFarRatio * radius,                         // near plane
    radius * 4.5                                   // far plane
);
auto camera = vsg::Camera::create(perspective, lookAt, vsg::ViewportState::create(window->extent2D()));

// Add event handlers
viewer->addEventHandler(vsg::CloseHandler::create(viewer));
viewer->addEventHandler(vsg::Trackball::create(camera));

// Create command graph and compile
auto commandGraph = vsg::createCommandGraphForView(window, camera, vsg_scene);
viewer->assignRecordAndSubmitTaskAndPresentation({commandGraph});
viewer->compile();

// Main render loop
while (viewer->advanceToNextFrame())
{
    viewer->handleEvents();
    viewer->update();
    viewer->recordAndSubmit();
    viewer->present();
}
```

## Running the Example

```bash
# Load default model
./vsghelloworld

# Load specific model
./vsghelloworld path/to/model.vsgt
./vsghelloworld models/teapot.obj

# With file search paths
VSG_FILE_PATH=/path/to/models ./vsghelloworld model.vsgt

# With file caching
VSG_FILE_CACHE=/tmp/vsg_cache ./vsghelloworld large_model.vsgb
```

## Key Takeaways

- VSG provides a clean, minimal API for 3D applications
- The viewer manages the complete render loop
- Scene bounds calculation helps with automatic camera setup
- Event handlers provide modular input processing
- Command graphs encapsulate Vulkan rendering commands
- The compile step transfers all data to the GPU
- Reference counting handles all cleanup automatically
- vsgXchange enables loading of various 3D file formats