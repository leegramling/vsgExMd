# vsgortho

## Overview

The `vsgortho` example demonstrates orthographic projection in VSG. Unlike perspective projection where objects appear smaller with distance, orthographic projection maintains object sizes regardless of depth, making it ideal for CAD applications, 2D games, technical drawings, and architectural visualizations.

## Key VSG Features Used

- **vsg::Orthographic** - Orthographic projection matrix
- **Aspect ratio handling** - Proper scaling for different window sizes
- **View volume calculation** - Setting orthographic bounds
- **Scene bounds computation** - Automatic camera positioning

## What the Example Demonstrates

1. **Orthographic Projection Setup**
   - Creating orthographic projection matrix
   - Setting left, right, bottom, top bounds
   - Configuring near and far planes

2. **Aspect Ratio Management**
   - Adjusting projection bounds based on window aspect
   - Maintaining correct proportions
   - Handling both landscape and portrait orientations

3. **Scene Fitting**
   - Computing scene bounds
   - Calculating appropriate view volume
   - Positioning camera for optimal view

## Code Highlights

```cpp
// Compute scene bounds for camera setup
vsg::ComputeBounds computeBounds;
vsg_scene->accept(computeBounds);
vsg::dvec3 centre = (computeBounds.bounds.min + computeBounds.bounds.max) * 0.5;
double radius = vsg::length(computeBounds.bounds.max - computeBounds.bounds.min) * 0.6;

// Calculate orthographic bounds based on aspect ratio
double aspectRatio = static_cast<double>(window->extent2D().width) / 
                    static_cast<double>(window->extent2D().height);
double halfDim = radius * 1.1;  // Add some padding
double halfHeight, halfWidth;

if (window->extent2D().width > window->extent2D().height)
{
    // Landscape orientation
    halfHeight = halfDim;
    halfWidth = halfDim * aspectRatio;
}
else
{
    // Portrait orientation
    halfWidth = halfDim;
    halfHeight = halfDim / aspectRatio;
}

// Create orthographic projection
auto projection = vsg::Orthographic::create(
    -halfWidth, halfWidth,      // left, right
    -halfHeight, halfHeight,    // bottom, top
    nearFarRatio * radius,      // near plane
    radius * 4.5                // far plane
);

// Set up camera view
auto lookAt = vsg::LookAt::create(
    centre + vsg::dvec3(0.0, -radius * 3.5, 0.0),  // eye
    centre,                                         // center
    vsg::dvec3(0.0, 0.0, 1.0)                      // up
);

auto camera = vsg::Camera::create(projection, lookAt, 
                                 vsg::ViewportState::create(window->extent2D()));
```

## Orthographic vs Perspective

### Orthographic Projection
```cpp
// Fixed view volume in world space
vsg::Orthographic::create(
    left, right,    // X bounds
    bottom, top,    // Y bounds
    near, far       // Z bounds
);
```

### Perspective Projection (for comparison)
```cpp
// Field of view based
vsg::Perspective::create(
    fieldOfViewY,   // Vertical FOV in degrees
    aspectRatio,    // Width/height ratio
    near, far       // Clipping planes
);
```

## Running the Example

```bash
# Basic usage
./vsgortho model.vsgt

# Window options
./vsgortho model.vsgt --window 800 600
./vsgortho model.vsgt --fullscreen

# Debug options
./vsgortho model.vsgt --debug
./vsgortho model.vsgt --api
```

### Visual Characteristics

- **Parallel lines remain parallel** - No convergence at distance
- **Objects maintain size** - No perspective foreshortening
- **Equal scale** - Measurements are preserved
- **No vanishing points** - Ideal for technical drawings

### Use Cases

1. **CAD Applications** - Accurate measurements and dimensions
2. **2D Games** - Side-scrollers, top-down views
3. **Technical Drawings** - Blueprints, schematics
4. **UI Overlays** - HUD elements, menus
5. **Isometric Views** - Strategy games, architectural visualization

## Key Takeaways

- Orthographic projection preserves parallel lines and relative sizes
- The view volume is a rectangular box in world space
- Proper aspect ratio handling prevents distortion
- Scene bounds calculation helps set appropriate view volume
- Near and far planes still clip geometry outside the view volume
- Orthographic projection is essential for technical and 2D applications
- The same camera controls (trackball) work with orthographic projection