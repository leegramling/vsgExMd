# vsgmultiviews

## Overview

The `vsgmultiviews` example demonstrates rendering multiple views in a single window using overlapping viewports. It shows how to create a main view with a picture-in-picture overlay, handle clearing of specific regions, and dynamically swap view rendering order.

## Key VSG Features Used

- **Multiple views in one window** - Main view with overlay
- **vsg::RenderGraph** - Managing multiple views
- **vsg::ClearAttachments** - Selective clearing of viewport regions
- **Dynamic view reordering** - Swapping rendering order at runtime
- **Per-view cameras** - Independent camera control
- **Viewport overlays** - Picture-in-picture rendering
- **Depth buffer clearing** - Proper layering of views

## What the Example Demonstrates

1. **Overlay Views**
   - Main view covering entire window
   - Secondary view as picture-in-picture
   - Proper depth clearing for overlay

2. **Clear Attachments**
   - Clearing specific viewport regions
   - Both color and depth clearing
   - Preventing depth conflicts between views

3. **Dynamic View Management**
   - Swapping view rendering order with 's' key
   - Updating viewport states
   - Adjusting projection matrices for swapped viewports

4. **Event Handling Priority**
   - Secondary view trackball has higher priority
   - Mouse events in overlay area control overlay camera
   - Main view receives events outside overlay

## Code Highlights

```cpp
// Create main view covering entire window
auto main_camera = createCameraForScene(scenegraph, 0, 0, width, height);
auto main_view = vsg::View::create(main_camera, scenegraph);

// Create overlay view in top-right corner
auto secondary_camera = createCameraForScene(
    scenegraph2, 
    (width * 3) / 4,  // x: 3/4 from left
    0,                // y: top
    width / 4,        // width: 1/4 of window
    height / 4        // height: 1/4 of window
);
auto secondary_view = vsg::View::create(secondary_camera, scenegraph2);

// Create render graph and add views
auto renderGraph = vsg::RenderGraph::create(window);
renderGraph->addChild(main_view);

// Clear overlay area before rendering secondary view
VkClearAttachment color_attachment{
    VK_IMAGE_ASPECT_COLOR_BIT, 0, 
    vsg::sRGB_to_linear(0.2f, 0.2f, 0.2f, 1.0f)
};
VkClearAttachment depth_attachment{
    VK_IMAGE_ASPECT_DEPTH_BIT, 1, 
    {0.0f, 0}
};

VkClearRect rect{secondary_camera->getRenderArea(), 0, 1};
auto clearAttachments = vsg::ClearAttachments::create(
    {color_attachment, depth_attachment}, 
    {rect, rect}
);
renderGraph->addChild(clearAttachments);
renderGraph->addChild(secondary_view);

// View swap handler
class ViewHandler : public vsg::Inherit<vsg::Visitor, ViewHandler>
{
    void apply(vsg::KeyPressEvent& keyPress) override
    {
        if (keyPress.keyBase == 's')
        {
            // Swap rendering order
            std::swap(renderGraph->children[views[0].first], 
                     renderGraph->children[views[1].first]);
            
            // Swap viewport states
            std::swap(views[0].second->camera->viewportState, 
                     views[1].second->camera->viewportState);
            
            // Update projection matrices for new aspect ratios
            view0->camera->projectionMatrix->changeExtent(extent0, extent1);
            view1->camera->projectionMatrix->changeExtent(extent1, extent0);
        }
    }
};

// Event handler order matters - secondary view first for overlay priority
viewer->addEventHandler(vsg::Trackball::create(secondary_camera));
viewer->addEventHandler(vsg::Trackball::create(main_camera));
```

## Running the Example

```bash
# Basic usage with same model in both views
./vsgmultiviews model.vsgt

# Different models in each view
./vsgmultiviews main_model.vsgt overlay_model.vsgt

# Window size
./vsgmultiviews model.vsgt --window 1920 1080

# Performance testing
./vsgmultiviews model.vsgt --test --fps

# Debugging
./vsgmultiviews model.vsgt --debug
./vsgmultiviews model.vsgt --profiler
```

### Controls

- **Mouse in main area**: Control main camera
- **Mouse in overlay**: Control overlay camera
- **S key**: Swap main and overlay views
- **ESC**: Exit application

### View Layout

```
┌─────────────────────────────┐
│                      ┌─────┐│
│                      │     ││
│     Main View        │ PiP ││
│                      └─────┘│
│                             │
│                             │
└─────────────────────────────┘
```

After pressing 'S':
```
┌─────────────────────────────┐
│ ┌─────┐                     │
│ │     │                     │
│ │ PiP │    Main View        │
│ └─────┘                     │
│                             │
│                             │
└─────────────────────────────┘
```

## Key Takeaways

- Multiple views can be rendered in a single window using one RenderGraph
- ClearAttachments allows selective clearing of viewport regions
- Proper depth clearing is essential for overlapping views
- View rendering order affects which view appears on top
- Event handler order determines input priority for overlapping areas
- Dynamic view management allows runtime layout changes
- Projection matrices must be updated when viewport aspect ratios change
- This pattern is useful for CAD applications, editors, and debugging tools