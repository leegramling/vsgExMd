# vsgturntablecamera

## Overview

The `vsgturntablecamera` example demonstrates how to create a custom camera controller in VSG by implementing a turntable-style camera system. This provides smooth, professional camera movement ideal for object inspection, product visualization, and architectural walkthroughs where the camera orbits around a central focal point.

## Key VSG Features Used

- **Custom event handlers** - Implementing vsg::Visitor for input handling
- **Camera animation** - Smooth interpolated camera movements
- **LookAt manipulation** - Direct viewmatrix control
- **Multi-window support** - Camera controller across multiple windows
- **Keyboard/mouse integration** - Comprehensive input device support
- **Touch input support** - Multi-touch gesture handling
- **Viewpoint management** - Predefined camera positions with smooth transitions

## What the Example Demonstrates

1. **Turntable Camera Mechanics**
   - Orbital rotation around a focal point
   - Smooth zoom in/out along the view vector
   - Constrained panning within the scene bounds
   - Gimbal lock avoidance for smooth rotation

2. **Advanced Input Handling**
   - Mouse button mapping for different operations
   - Keyboard shortcuts for common camera movements
   - Touch gesture recognition for mobile support
   - Throw momentum for natural camera motion

3. **Animation System Integration**
   - Interpolated viewpoint transitions
   - Duration-based smooth movement
   - Key-mapped preset viewpoints
   - Frame-based animation updates

4. **Professional Camera Features**
   - Multi-window coordinate mapping
   - Focus handling for window systems
   - Render area boundary checking
   - Configurable movement sensitivity

## Code Highlights

```cpp
// Custom turntable camera controller class
class Turntable : public vsg::Inherit<vsg::Visitor, Turntable>
{
public:
    explicit Turntable(vsg::ref_ptr<vsg::Camera> camera);

    // Core camera operations
    virtual void rotate(double angle, const vsg::dvec3& axis);
    virtual void zoom(double ratio);
    virtual void pan(const vsg::dvec2& delta);

    // Input event handling
    void apply(vsg::KeyPressEvent& keyPress) override;
    void apply(vsg::ButtonPressEvent& buttonPress) override;
    void apply(vsg::MoveEvent& moveEvent) override;
    void apply(vsg::ScrollWheelEvent& scrollWheel) override;
    void apply(vsg::TouchMoveEvent& touchMove) override;
    void apply(vsg::FrameEvent& frame) override;

    // Coordinate system conversions
    vsg::dvec2 ndc(const vsg::PointerEvent& event);  // Normalized device coordinates
    vsg::dvec3 ttc(const vsg::PointerEvent& event);  // Turntable coordinates

    // Multi-window support
    std::map<vsg::observer_ptr<vsg::Window>, vsg::ivec2> windowOffsets;
    void addWindow(vsg::ref_ptr<vsg::Window> window, const vsg::ivec2& offset = {});

    // Viewpoint management
    void addKeyViewpoint(vsg::KeySymbol key, vsg::ref_ptr<vsg::LookAt> lookAt, double duration = 1.0);
    void setViewpoint(vsg::ref_ptr<vsg::LookAt> lookAt, double duration = 1.0);

    // Input configuration
    vsg::ButtonMask rotateButtonMask = vsg::BUTTON_MASK_1;  // Left mouse button
    vsg::ButtonMask panButtonMask = vsg::BUTTON_MASK_2;     // Middle mouse button
    vsg::ButtonMask zoomButtonMask = vsg::BUTTON_MASK_3;    // Right mouse button

    // Keyboard controls
    vsg::KeySymbol turnLeftKey = vsg::KEY_a;
    vsg::KeySymbol turnRightKey = vsg::KEY_d;
    vsg::KeySymbol pitchUpKey = vsg::KEY_w;
    vsg::KeySymbol pitchDownKey = vsg::KEY_s;
    vsg::KeySymbol rollLeftKey = vsg::KEY_q;
    vsg::KeySymbol rollRightKey = vsg::KEY_e;

    // Movement sensitivity
    double zoomScale = 1.0;
    bool supportsThrow = true;  // Momentum after mouse release

private:
    enum UpdateMode { INACTIVE, ROTATE, PAN, ZOOM };
    UpdateMode _updateMode = INACTIVE;
    
    vsg::ref_ptr<vsg::Camera> _camera;
    vsg::ref_ptr<vsg::LookAt> _lookAt;
    
    // Animation state
    vsg::time_point _startTime;
    vsg::ref_ptr<vsg::LookAt> _startLookAt;
    vsg::ref_ptr<vsg::LookAt> _endLookAt;
    double _animationDuration = 0.0;
};

// Rotation with gimbal lock avoidance
void Turntable::applyRotationWithGimbalAvoidance(const vsg::dvec3& rot, 
                                                 double scaleRotation, 
                                                 const vsg::dvec3& globalUp)
{
    vsg::dvec3 horizontalAxis, verticalAxis;
    auto lookVector = vsg::normalize(_lookAt->center - _lookAt->eye);
    
    computeRotationAxesWithGimbalAvoidance(horizontalAxis, verticalAxis, lookVector, globalUp);
    
    // Apply rotation around computed axes
    auto horizontalRotation = vsg::rotate(rot.x * scaleRotation, horizontalAxis);
    auto verticalRotation = vsg::rotate(rot.y * scaleRotation, verticalAxis);
    
    // Combine rotations and apply to camera
    auto combinedRotation = horizontalRotation * verticalRotation;
    auto rotatedDirection = combinedRotation * lookVector;
    
    _lookAt->eye = _lookAt->center - rotatedDirection * distance;
}

// Smooth viewpoint transitions
void Turntable::setViewpoint(vsg::ref_ptr<vsg::LookAt> lookAt, double duration)
{
    if (duration <= 0.0)
    {
        // Instant transition
        *_lookAt = *lookAt;
        return;
    }

    // Setup animation
    _startTime = vsg::clock::now();
    _startLookAt = vsg::LookAt::create(*_lookAt);
    _endLookAt = lookAt;
    _animationDuration = duration;
}

// Frame-based animation updates
void Turntable::apply(vsg::FrameEvent& frame)
{
    if (_animationDuration > 0.0)
    {
        auto currentTime = vsg::clock::now();
        auto elapsed = std::chrono::duration<double>(currentTime - _startTime).count();
        
        if (elapsed >= _animationDuration)
        {
            // Animation complete
            *_lookAt = *_endLookAt;
            _animationDuration = 0.0;
        }
        else
        {
            // Interpolate between start and end positions
            double t = elapsed / _animationDuration;
            t = smoothstep(t);  // Apply easing curve
            
            _lookAt->eye = vsg::mix(_startLookAt->eye, _endLookAt->eye, t);
            _lookAt->center = vsg::mix(_startLookAt->center, _endLookAt->center, t);
            _lookAt->up = vsg::normalize(vsg::mix(_startLookAt->up, _endLookAt->up, t));
        }
    }
}

// Touch gesture handling
void Turntable::apply(vsg::TouchMoveEvent& touchMove)
{
    if (touchMove.touches.size() == 1)
    {
        // Single touch - rotation
        auto& touch = touchMove.touches[0];
        if (_previousTouches.contains(touch.id))
        {
            auto& prevTouch = _previousTouches[touch.id];
            auto delta = vsg::dvec2(touch.x - prevTouch->x, touch.y - prevTouch->y);
            
            // Convert to turntable coordinates and apply rotation
            double angle = vsg::length(delta) * 0.01;
            vsg::dvec3 axis = vsg::normalize(vsg::dvec3(-delta.y, delta.x, 0.0));
            rotate(angle, axis);
        }
    }
    else if (touchMove.touches.size() == 2)
    {
        // Two finger gesture - zoom
        auto& touch1 = touchMove.touches[0];
        auto& touch2 = touchMove.touches[1];
        
        double distance = vsg::length(vsg::dvec2(touch2.x - touch1.x, touch2.y - touch1.y));
        
        if (_prevZoomTouchDistance > 0.0)
        {
            double ratio = distance / _prevZoomTouchDistance;
            zoom(ratio);
        }
        
        _prevZoomTouchDistance = distance;
    }
}

// Integration with viewer
auto viewer = vsg::Viewer::create();
auto camera = vsg::Camera::create(perspective, lookAt, viewport);

// Use custom turntable controller instead of default trackball
auto cameraController = Turntable::create(camera);
viewer->addEventHandler(cameraController);

// Optional: Add preset viewpoints
cameraController->addKeyViewpoint(vsg::KEY_1, 
    vsg::LookAt::create(frontView_eye, center, up), 1.0);
cameraController->addKeyViewpoint(vsg::KEY_2, 
    vsg::LookAt::create(sideView_eye, center, up), 1.0);
cameraController->addKeyViewpoint(vsg::KEY_3, 
    vsg::LookAt::create(topView_eye, center, up), 1.0);
```

## Running the Example

```bash
# Basic usage with 3D model
./vsgturntablecamera model.vsgt

# Multiple models
./vsgturntablecamera model1.vsgt model2.gltf model3.obj

# Load image as textured quad
./vsgturntablecamera texture.jpg

# Performance and threading options
./vsgturntablecamera model.vsgt --mt         # Enable multi-threading
./vsgturntablecamera model.vsgt --fps        # Report frame rate
./vsgturntablecamera model.vsgt -c 0 -c 2    # CPU affinity

# Animation and camera options
./vsgturntablecamera model.vsgt -p camera_path.vsgb  # Camera animation
./vsgturntablecamera model.vsgt --no-auto-play       # Disable auto animation

# Window configuration
./vsgturntablecamera model.vsgt --window 1920 1080
./vsgturntablecamera model.vsgt --fullscreen

# Debug and profiling
./vsgturntablecamera model.vsgt --debug
./vsgturntablecamera model.vsgt --profiler --cpu 2 --gpu 2
./vsgturntablecamera model.vsgt --gpu-annotation
```

### Input Controls

**Mouse Controls**:
- **Left click + drag**: Rotate camera around focal point
- **Middle click + drag**: Pan view
- **Right click + drag**: Zoom in/out
- **Scroll wheel**: Zoom in/out

**Keyboard Controls**:
- **W/S**: Pitch up/down
- **A/D**: Turn left/right
- **Q/E**: Roll left/right
- **O/I**: Move forward/backward
- **Arrow keys**: Move left/right/up/down
- **Number keys**: Jump to preset viewpoints (if configured)

**Touch Controls**:
- **Single finger drag**: Rotate camera
- **Two finger pinch**: Zoom in/out
- **Two finger pan**: Pan view

## Key Features

### Gimbal Lock Avoidance

The turntable camera uses sophisticated rotation axis computation to prevent gimbal lock:

```cpp
void computeRotationAxesWithGimbalAvoidance(vsg::dvec3& horizontalAxis, 
                                           vsg::dvec3& verticalAxis,
                                           const vsg::dvec3& lookVector, 
                                           const vsg::dvec3& globalUp)
{
    // Horizontal axis is perpendicular to both look vector and global up
    horizontalAxis = vsg::normalize(vsg::cross(lookVector, globalUp));
    
    // Vertical axis is perpendicular to horizontal axis and look vector
    verticalAxis = vsg::normalize(vsg::cross(horizontalAxis, lookVector));
}
```

### Smooth Animation System

Viewpoint transitions use eased interpolation for professional camera movement:

```cpp
double smoothstep(double t)
{
    return t * t * (3.0 - 2.0 * t);  // Hermite interpolation
}
```

### Multi-Window Coordination

Camera controllers can handle input from multiple windows with coordinate offsets:

```cpp
cameraController->addWindow(window1, {0, 0});
cameraController->addWindow(window2, {window1->extent2D().width, 0});
```

## Key Takeaways

- Custom camera controllers provide specialized navigation paradigms
- Turntable cameras are ideal for object inspection and product visualization
- Gimbal lock avoidance ensures smooth rotation in all orientations
- Frame-based animation enables smooth camera transitions
- Multi-touch support makes applications mobile-friendly
- Viewpoint presets enhance user experience for common viewing angles
- Throw momentum creates natural camera motion feel
- Professional camera systems require careful coordinate space management
- VSG's event system makes custom camera controllers straightforward to implement