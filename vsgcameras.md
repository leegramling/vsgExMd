# vsgcameras

## Overview

The `vsgcameras` example demonstrates multiple camera views and camera management in VSG. It shows how to find cameras in a scene, create multiple viewports, implement camera switching, and use tracking view matrices for dynamic camera updates.

## Key VSG Features Used

- **vsg::Camera** - Camera objects with projection and view matrices
- **vsg::FindCameras** - Visitor to find cameras in scene graphs
- **Multiple viewports** - Rendering multiple views in one window
- **vsg::TrackingViewMatrix** - Dynamic view matrix tracking
- **vsg::RenderGraph** - Per-view render management
- **Camera switching** - Interactive camera selection
- **Named cameras** - Camera identification system
- **View transformation** - Computing transforms from node paths

## What the Example Demonstrates

1. **Camera Discovery**
   - Finding existing cameras in loaded scenes
   - Creating default cameras if none exist
   - Enumerating camera paths in scene graph

2. **Multiple Viewport Rendering**
   - Main interactive view
   - Secondary views for each camera
   - Viewport layout management

3. **Camera Switching**
   - Number keys (1-9) select cameras
   - Mouse scroll wheel navigation
   - Return to original view (0 key)

4. **Dynamic Camera Tracking**
   - TrackingViewMatrix follows scene graph transforms
   - Cameras can be animated within the scene
   - View matrices update automatically

## Code Highlights

```cpp
// Find all cameras in the scene
auto scene_cameras = vsg::visit<vsg::FindCameras>(scenegraph).cameras;

// Create default cameras if none exist
if (scene_cameras.empty())
{
    auto root = vsg::Group::create();
    root->addChild(scenegraph);
    
    auto perspective = vsg::Perspective::create(90.0, 1.0, nearFarRatio * radius, radius * 4.0);
    
    // Create orthogonal views
    auto lookAt = vsg::LookAt::create(
        centre + vsg::dvec3(radius * 2.0, 0.0, 0.0),  // Left view
        centre, 
        vsg::dvec3(0.0, 0.0, 1.0)
    );
    auto camera = vsg::Camera::create(perspective, lookAt);
    camera->name = "Left";
    root->addChild(camera);
}

// Camera selector event handler
class CameraSelector : public vsg::Inherit<vsg::Visitor, CameraSelector>
{
    void apply(vsg::KeyPressEvent& keyPress) override
    {
        if (keyPress.keyBase >= '1' && keyPress.keyBase <= '9')
        {
            // Select camera by index
            currentCameraIndex = keyPress.keyBase - '1';
            selectCameraByIndex(currentCameraIndex);
        }
    }
    
    void selectCameraByIndex(int index)
    {
        // Compute transform from node path
        auto matrix = vsg::visit<vsg::ComputeTransform>(begin, end).matrix;
        
        // Apply selected camera's view to main camera
        auto selected_lookAt = camera->viewMatrix.cast<vsg::LookAt>();
        auto main_lookAt = main_camera->viewMatrix.cast<vsg::LookAt>();
        if (main_lookAt)
        {
            *main_lookAt = *selected_lookAt;
            main_lookAt->transform(matrix);
        }
    }
};

// Create multiple viewports
auto main_RenderGraph = vsg::RenderGraph::create(window, main_view);
commandGraph->addChild(main_RenderGraph);

// Secondary views for each camera
for (auto& [nodePath, camera] : scene_cameras)
{
    // Dynamic tracking view matrix
    auto viewMatrix = vsg::TrackingViewMatrix::create(nodePath);
    auto viewportState = vsg::ViewportState::create(x, y, secondary_width, secondary_height);
    
    auto secondary_camera = vsg::Camera::create(projectionMatrix, viewMatrix, viewportState);
    auto secondary_view = vsg::View::create(secondary_camera, scenegraph);
    auto renderGraph = vsg::RenderGraph::create(window, secondary_view);
    
    commandGraph->addChild(renderGraph);
}
```

## Running the Example

```bash
# Load model with embedded cameras
./vsgcameras models/cameras.gltf

# Load model without cameras (creates default views)
./vsgcameras models/teapot.vsgt

# With options
./vsgcameras model.vsgt --debug    # Enable validation
./vsgcameras model.vsgt --window 1920 1080
./vsgcameras model.vsgt --profiler # Enable profiling
```

### Controls

- **1-9 keys**: Switch to camera 1-9
- **0 key**: Return to original view
- **Mouse wheel up**: Next camera
- **Mouse wheel down**: Previous camera
- **Mouse drag**: Control main camera (trackball)

### Expected Layout

```
┌─────────────────────────────────┐
│                                 │
│                          ┌─────┐│
│                          │Cam 1││
│        Main View         ├─────┤│
│                          │Cam 2││
│                          ├─────┤│
│                          │Cam 3││
│                          └─────┘│
└─────────────────────────────────┘
```

## Key Takeaways

- VSG supports multiple cameras and viewports in a single window
- Cameras can be discovered in loaded scenes or created programmatically
- TrackingViewMatrix enables dynamic camera updates based on scene transforms
- The RenderGraph/View system allows flexible viewport configurations
- Camera switching can be implemented with custom event handlers
- Named cameras help with identification and selection
- The example provides a foundation for multi-view applications like CAD tools