# vsgrendertotexture

## Overview

The `vsgrendertotexture` example demonstrates offscreen rendering in VSG. It renders a 3D scene to a texture, then uses that texture on the faces of rotating quads. This technique is fundamental for effects like mirrors, security cameras, portals, and post-processing.

## Key VSG Features Used

- **Offscreen rendering** - Rendering to texture instead of screen
- **Custom render passes** - Configuring attachments and dependencies
- **Framebuffer creation** - Color and depth attachments
- **Subpass dependencies** - Synchronizing render passes
- **Texture sampling** - Using rendered image as texture
- **Multiple render graphs** - Offscreen and onscreen passes
- **Image layout transitions** - Proper Vulkan synchronization

## What the Example Demonstrates

1. **Offscreen Render Setup**
   - Creating images for color and depth attachments
   - Configuring render pass with proper layouts
   - Setting up framebuffer with attachments
   - Subpass dependencies for synchronization

2. **Render to Texture Pipeline**
   - First pass: Render scene to texture
   - Second pass: Use texture on geometry
   - Proper synchronization between passes

3. **Vulkan Concepts**
   - Image layout transitions
   - Pipeline barriers
   - Attachment descriptions
   - Subpass dependencies

## Code Highlights

```cpp
// Create offscreen render target
auto colorImage = vsg::Image::create();
colorImage->format = VK_FORMAT_R8G8B8A8_SRGB;
colorImage->extent = {extent.width, extent.height, 1};
colorImage->usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | 
                   VK_IMAGE_USAGE_SAMPLED_BIT;
colorImage->initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;

// Create image view and sampler for texture access
auto colorImageView = createImageView(context, colorImage, 
                                     VK_IMAGE_ASPECT_COLOR_BIT);
auto colorSampler = vsg::Sampler::create();
colorSampler->magFilter = VK_FILTER_LINEAR;
colorSampler->minFilter = VK_FILTER_LINEAR;

// Configure render pass attachments
vsg::RenderPass::Attachments attachments(2);
// Color attachment
attachments[0].format = VK_FORMAT_R8G8B8A8_SRGB;
attachments[0].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attachments[0].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[0].finalLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

// Critical: Subpass dependencies for synchronization
vsg::RenderPass::Dependencies dependencies(2);

// Dependency 1: External to subpass 0
dependencies[0].srcSubpass = VK_SUBPASS_EXTERNAL;
dependencies[0].dstSubpass = 0;
dependencies[0].srcStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[0].dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependencies[0].srcAccessMask = VK_ACCESS_SHADER_READ_BIT;
dependencies[0].dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

// Dependency 2: Subpass 0 to external (critical for texture usage)
dependencies[1].srcSubpass = 0;
dependencies[1].dstSubpass = VK_SUBPASS_EXTERNAL;
dependencies[1].srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependencies[1].dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[1].srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dependencies[1].dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

// Create framebuffer with attachments
auto framebuffer = vsg::Framebuffer::create(renderPass, 
    vsg::ImageViews{colorImageView, depthImageView}, extent);

// Create offscreen render graph
auto offscreenRenderGraph = vsg::RenderGraph::create();
offscreenRenderGraph->framebuffer = framebuffer;
offscreenRenderGraph->renderPass = renderPass;
offscreenRenderGraph->renderArea.extent = extent;
offscreenRenderGraph->clearValues = {
    {{{0.2f, 0.2f, 0.4f, 1.0f}}},  // Clear color
    {{{1.0f, 0}}}                   // Clear depth
};

// Use rendered texture in main scene
auto texture = vsg::DescriptorImage::create(colorImage, 0, 0, 
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER);
auto descriptorSet = vsg::DescriptorSet::create(descriptorSetLayout, 
    vsg::Descriptors{texture});
```

## Running the Example

```bash
# Basic usage - renders teapot to texture on rotating quads
./vsgrendertotexture

# Load custom model for offscreen rendering
./vsgrendertotexture models/teapot.vsgt

# Command graph options
./vsgrendertotexture model.vsgt --nested    # Nested command graph
./vsgrendertotexture model.vsgt -s         # Separate command graph

# Multi-threaded rendering
./vsgrendertotexture model.vsgt --mt

# Debugging
./vsgrendertotexture --debug               # Validation layers
./vsgrendertotexture --api                 # API dump
./vsgrendertotexture --sync                # Synchronization validation
```

### Visual Result

The example creates:
1. An offscreen render of the loaded model(s)
2. Rotating quads that display the rendered texture
3. The texture updates each frame showing the animated scene

## Key Takeaways

- Render-to-texture requires careful setup of image usage flags
- Subpass dependencies are crucial for synchronization
- Image layouts must transition properly between rendering and sampling
- The rendered image needs both COLOR_ATTACHMENT and SAMPLED usage
- Framebuffers define the render targets for offscreen rendering
- Multiple render graphs can be chained in a command graph
- This technique is essential for many advanced rendering effects
- Proper synchronization prevents race conditions in Vulkan