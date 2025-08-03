# vsgoverlay

## Overview

The `vsgoverlay` example demonstrates how to render overlays in VSG by using depth buffer clearing between render passes. This technique allows rendering a second scene on top of the first, useful for HUDs, UI overlays, annotations, and foreground objects that should always appear in front.

## Key VSG Features Used

- **vsg::ClearAttachments** - Selective buffer clearing
- **Multiple views** - Rendering multiple scenes in order
- **Depth buffer management** - Clearing depth between passes
- **Render order control** - Explicit rendering sequence
- **Same viewport** - Both views use full window

## What the Example Demonstrates

1. **Overlay Rendering Technique**
   - Render main scene with depth testing
   - Clear only depth buffer (preserve color)
   - Render overlay scene on top

2. **Depth Buffer Clearing**
   - Selective clear of depth attachment only
   - Color buffer remains untouched
   - Enables overlay to render on top

3. **Multiple Scene Management**
   - Two independent scene graphs
   - Shared camera for both views
   - Each view can have different content

## Code Highlights

```cpp
// Create render graph
auto renderGraph = vsg::RenderGraph::create(window);

// View 1: Main scene (rendered first)
auto camera = createCameraForScene(scenegraph, 0, 0, width, height);
auto view1 = vsg::View::create(camera, scenegraph);
renderGraph->addChild(view1);

// Clear depth buffer between views
VkClearValue clearValue{};
clearValue.depthStencil = {0.0f, 0};  // Clear to far depth

VkClearAttachment attachment{
    VK_IMAGE_ASPECT_DEPTH_BIT,  // Only clear depth
    1,                          // Attachment index (1 = depth)
    clearValue
};

VkClearRect rect{
    VkRect2D{VkOffset2D{0, 0}, VkExtent2D{width, height}},  // Full viewport
    0,  // baseArrayLayer
    1   // layerCount
};

auto clearAttachments = vsg::ClearAttachments::create(
    vsg::ClearAttachments::Attachments{attachment},
    vsg::ClearAttachments::Rects{rect}
);
renderGraph->addChild(clearAttachments);

// View 2: Overlay scene (rendered second, on top)
auto secondary_camera = createCameraForScene(scenegraph2, 0, 0, width, height);
auto view2 = vsg::View::create(secondary_camera, scenegraph2);
renderGraph->addChild(view2);

// Add lighting to both views
auto headlight = vsg::createHeadlight();
view1->addChild(headlight);
view2->addChild(headlight);

// Single trackball controls both cameras
viewer->addEventHandler(vsg::Trackball::create(camera));
```

## Running the Example

```bash
# Basic usage - same model as overlay
./vsgoverlay model.vsgt

# Different models for main scene and overlay
./vsgoverlay background.vsgt foreground.vsgt

# Window options
./vsgoverlay model.vsgt --window 1920 1080

# Debug options
./vsgoverlay model.vsgt --debug
./vsgoverlay model.vsgt --api
```

### Visual Result

1. First model renders normally with depth testing
2. Depth buffer is cleared
3. Second model renders on top, always visible
4. Both models are affected by the same camera movement

### Use Cases

1. **HUD Elements** - Health bars, minimaps, scores
2. **UI Overlays** - Menus, dialogs, tooltips
3. **Annotations** - Labels that should never be occluded
4. **Selection Highlights** - Selected objects rendered on top
5. **Debug Visualization** - Wireframes, bounds, helpers
6. **Foreground Objects** - Important items always visible

## Alternative Approaches

```cpp
// Method 1: Disable depth testing (this example's approach)
// Clear depth buffer between passes

// Method 2: Depth range manipulation
camera->projectionMatrix->depthRange = {0.0, 0.1};  // Background
overlayCamera->projectionMatrix->depthRange = {0.0, 0.0};  // Overlay

// Method 3: Separate render passes with different depth settings
// Use different RenderGraph instances

// Method 4: Stencil buffer for selective rendering
// More complex but allows selective overlays
```

## Key Takeaways

- Clearing depth buffer between views enables overlay rendering
- The render order in the RenderGraph determines layering
- Color buffer is preserved when clearing only depth
- Both views can share the same camera for synchronized movement
- This technique is simpler than managing multiple render passes
- Useful for any content that must appear on top
- The same viewport is used for both views ensuring alignment