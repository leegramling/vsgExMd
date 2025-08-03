# vsganaglyphicstereo

## Overview

The `vsganaglyphicstereo` example demonstrates how to create anaglyphic (red/cyan) 3D stereo rendering in VSG. This technique renders two slightly offset views with different color masks to create a stereoscopic effect when viewed with red/cyan 3D glasses.

## Key VSG Features Used

- **vsg::RelativeProjection** - Offset projection matrices for stereo views
- **vsg::RelativeViewMatrix** - Offset view matrices for eye separation
- **Color mask manipulation** - Channel-specific rendering (red vs cyan)
- **vsg::ColorBlendState** - Color channel write masking
- **Multiple views** - Left and right eye views
- **Depth buffer clearing** - Between stereo passes
- **Custom visitor pattern** - Pipeline state replacement

## What the Example Demonstrates

1. **Stereo Camera Setup**
   - Two cameras with horizontal eye separation
   - Convergence adjustment through projection shearing
   - Synchronized camera movement

2. **Color Channel Masking**
   - Left eye renders to red channel only
   - Right eye renders to green/blue channels
   - Pipeline state modification for color masks

3. **Rendering Order**
   - Left view renders first (red channel)
   - Depth buffer cleared between views
   - Right view renders second (cyan channels)

## Code Highlights

```cpp
// Custom visitor to replace color blend states
class ReplaceColorBlendState : public vsg::Visitor
{
public:
    ReplaceColorBlendState(vsg::Mask in_leftMask, vsg::Mask in_rightMask) :
        leftMask(in_leftMask), rightMask(in_rightMask) {}

    void apply(vsg::BindGraphicsPipeline& bgp) override
    {
        auto gp = bgp.pipeline;
        // Replace original ColorBlendState with stereo-specific ones
        
        auto left_colorBlendState = vsg::ColorBlendState::create(*colorBlendState);
        left_colorBlendState->mask = leftMask;
        left_colorBlendState->attachments[0].colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_A_BIT;

        auto right_colorBlendState = vsg::ColorBlendState::create(*colorBlendState);
        right_colorBlendState->mask = rightMask;
        right_colorBlendState->attachments[0].colorWriteMask = VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
    }
};

// Setup stereo cameras with eye separation
double eyeSeperation = 0.06;  // 6cm typical human eye separation
double shear = (eyeSeperation / screenWidth) * 0.8;

// Left eye camera - offset left, renders to red channel
auto left_relative_perspective = vsg::RelativeProjection::create(
    vsg::translate(-shear, 0.0, 0.0), perspective);
auto left_relative_view = vsg::RelativeViewMatrix::create(
    vsg::translate(-0.5 * eyeSeperation, 0.0, 0.0), lookAt);
auto left_camera = vsg::Camera::create(left_relative_perspective, left_relative_view, viewport);

// Right eye camera - offset right, renders to green/blue channels
auto right_relative_perspective = vsg::RelativeProjection::create(
    vsg::translate(shear, 0.0, 0.0), perspective);
auto right_relative_view = vsg::RelativeViewMatrix::create(
    vsg::translate(0.5 * eyeSeperation, 0.0, 0.0), lookAt);
auto right_camera = vsg::Camera::create(right_relative_perspective, right_relative_view, viewport);

// Render left view first (red channel)
auto left_view = vsg::View::create(left_camera, vsg_scene);
left_view->mask = leftMask;  // 0x1
renderGraph->addChild(left_view);

// Clear depth buffer between views
VkClearValue clearValue{};
clearValue.depthStencil = {0.0f, 0};
VkClearAttachment depth_attachment{VK_IMAGE_ASPECT_DEPTH_BIT, 1, clearValue};
auto clearAttachments = vsg::ClearAttachments::create(attachments, rects);
renderGraph->addChild(clearAttachments);

// Render right view second (cyan channels)
auto right_view = vsg::View::create(right_camera, vsg_scene);
right_view->mask = rightMask;  // 0x2
renderGraph->addChild(right_view);

// Dynamic eye separation adjustment based on view distance
double lookDistance = vsg::length(lookAt->center - lookAt->eye);
double horizontalSeperation = 0.5 * eyeSeperation;
horizontalSeperation *= (lookDistance / screenDistance);

left_relative_view->matrix = vsg::translate(horizontalSeperation, 0.0, 0.0);
right_relative_view->matrix = vsg::translate(-horizontalSeperation, 0.0, 0.0);
```

## Running the Example

```bash
# Basic usage with default model
./vsganaglyphicstereo

# Load specific 3D model
./vsganaglyphicstereo model.vsgt

# Stereo image pair mode
./vsganaglyphicstereo --stereo-pair left.jpg right.jpg

# Window options
./vsganaglyphicstereo --window 1920 1080
./vsganaglyphicstereo --fullscreen

# Offset adjustment for stereo pairs
./vsganaglyphicstereo --offset 0.1 0.0 --stereo-pair left.jpg right.jpg

# Debug options
./vsganaglyphicstereo --debug
./vsganaglyphicstereo --api
```

### Stereo Parameters

The example uses physiologically realistic stereo parameters:
- **Eye separation**: 6cm (typical human interpupillary distance)
- **Screen distance**: 75cm (typical viewing distance)
- **Screen width**: 55cm (physical screen width)
- **Convergence**: Adjusted via projection shearing

## Key Takeaways

- Anaglyphic stereo works by rendering two views with different color masks
- RelativeProjection and RelativeViewMatrix provide convenient stereo camera setup
- Color channel masking requires pipeline state modification
- Depth buffer must be cleared between stereo views to prevent interference
- Eye separation should scale with viewing distance for proper stereo perception
- The visitor pattern allows systematic modification of render states
- VSG's mask system enables selective rendering to different views
- This technique provides stereoscopic 3D with minimal hardware requirements