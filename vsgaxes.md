# vsgaxes

## Overview

The `vsgaxes` example demonstrates how to create an orientation gizmo (axes indicator) that shows the current view orientation. This is commonly seen in 3D applications as a small coordinate system display in the corner of the viewport that rotates with the main camera but maintains a fixed position.

## Key VSG Features Used

- **Custom ViewMatrix** - RotationTrackingMatrix extracts rotation
- **vsg::Orthographic** - Fixed-size overlay rendering
- **vsg::Builder** - Creating geometric primitives
- **Matrix decomposition** - Extracting rotation from view matrix
- **Multiple views** - Main scene and overlay gizmo
- **Shared viewport** - Both views render to same window

## What the Example Demonstrates

1. **Rotation Tracking**
   - Custom ViewMatrix that extracts rotation component
   - Decomposing matrices into translation, rotation, scale
   - Synchronizing gizmo orientation with main camera

2. **Gizmo Creation**
   - Building coordinate axes with cylinders and cones
   - Color-coded axes (Red=X, Green=Y, Blue=Z)
   - Center sphere for origin reference

3. **Overlay Positioning**
   - Using orthographic projection for fixed size
   - Positioning gizmo in corner of viewport
   - Maintaining aspect ratio

## Code Highlights

```cpp
// Custom view matrix that tracks rotation only
class RotationTrackingMatrix : public vsg::Inherit<vsg::ViewMatrix, RotationTrackingMatrix>
{
    vsg::dmat4 transform(const vsg::dvec3&) const override
    {
        vsg::dvec3 translation, scale;
        vsg::dquat rotation;
        
        // Decompose parent transform to extract rotation
        vsg::decompose(parentTransform_->transform(),
                      translation,
                      rotation,
                      scale);
        
        // Return only the rotation component
        return vsg::rotate(rotation);
    }
};

// Create arrow for one axis
vsg::ref_ptr<vsg::Node> createArrow(vsg::vec3 pos, vsg::vec3 dir, vsg::vec4 color)
{
    vsg::Builder builder;
    vsg::GeometryInfo geomInfo;
    vsg::StateInfo stateInfo;
    
    // Cylinder for arrow shaft
    geomInfo.color = vsg::vec4{1, 1, 1, 1};
    geomInfo.position = pos;
    geomInfo.transform = vsg::translate(0.0f, 0.0f, 0.5f);
    
    // Rotate to point in direction
    if (vsg::length(vsg::cross(vsg::vec3{0, 0, 1}, dir)) > 0.0001)
    {
        vsg::vec3 axis = vsg::cross(vsg::vec3{0, 0, 1}, dir);
        float angle = acos(vsg::dot(vsg::vec3{0, 0, 1}, dir));
        geomInfo.transform = vsg::rotate(angle, axis) * geomInfo.transform;
    }
    
    // Scale cylinder
    geomInfo.transform = geomInfo.transform * vsg::scale(0.1f, 0.1f, 1.0f);
    auto cylinder = builder.createCylinder(geomInfo, stateInfo);
    
    // Cone for arrow head
    geomInfo.color = color;  // Axis color
    geomInfo.transform = vsg::scale(0.3f, 0.3f, 0.3f) * axisTransform * 
                        vsg::translate(0.0f, 0.0f, 1.0f / 0.3f);
    auto cone = builder.createCone(geomInfo, stateInfo);
    
    auto arrow = vsg::Group::create();
    arrow->addChild(cylinder);
    arrow->addChild(cone);
    return arrow;
}

// Create complete gizmo with three axes
vsg::ref_ptr<vsg::Node> createGizmo()
{
    auto gizmo = vsg::Group::create();
    
    // X axis - Red
    gizmo->addChild(createArrow({0,0,0}, {1,0,0}, {1,0,0,1}));
    // Y axis - Green  
    gizmo->addChild(createArrow({0,0,0}, {0,1,0}, {0,1,0,1}));
    // Z axis - Blue
    gizmo->addChild(createArrow({0,0,0}, {0,0,1}, {0,0,1,1}));
    
    // Origin sphere
    vsg::Builder builder;
    vsg::GeometryInfo geomInfo;
    geomInfo.color = {1, 1, 1, 1};
    geomInfo.transform = vsg::scale(0.1f, 0.1f, 0.1f);
    gizmo->addChild(builder.createSphere(geomInfo, {}));
    
    return gizmo;
}

// Create axes view overlay
vsg::ref_ptr<vsg::View> createAxesView(vsg::ref_ptr<vsg::Camera> camera,
                                      double aspectRatio)
{
    // Track rotation of main camera
    auto viewMat = RotationTrackingMatrix::create(camera->viewMatrix);
    
    // Position gizmo in corner using orthographic projection
    double camWidth = 10;
    double camXOffs = -8;
    double camYOffs = -8;
    auto ortho = vsg::Orthographic::create(
        camXOffs, camXOffs + camWidth,                              // left, right
        camYOffs / aspectRatio, (camYOffs + camWidth) / aspectRatio, // bottom, top
        -1000, 1000                                                  // near, far
    );
    
    auto gizmoCamera = vsg::Camera::create(ortho, viewMat, camera->viewportState);
    
    auto view = vsg::View::create(gizmoCamera);
    view->addChild(vsg::createHeadlight());
    view->addChild(createGizmo());
    return view;
}
```

## Running the Example

```bash
# Basic usage
./vsgaxes model.vsgt

# Window options
./vsgaxes model.vsgt --window 1920 1080

# Debug options
./vsgaxes model.vsgt --debug
./vsgaxes model.vsgt --api
```

### Visual Result

- Main scene fills the viewport
- Small coordinate axes appear in lower-left corner
- Axes rotate to match main camera orientation
- Axes maintain fixed size regardless of zoom
- Red arrow = X axis
- Green arrow = Y axis  
- Blue arrow = Z axis
- White sphere = origin

## Key Takeaways

- Custom ViewMatrix implementations can track specific transform components
- Matrix decomposition extracts translation, rotation, and scale separately
- Orthographic projection is ideal for fixed-size overlays
- The Builder class simplifies creating geometric primitives
- Multiple views can share the same viewport with different cameras
- Gizmos help users maintain spatial orientation in 3D scenes
- This pattern is useful for any orientation indicator or compass