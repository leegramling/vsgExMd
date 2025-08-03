# vsgheadless

## Overview

The `vsgheadless` example demonstrates headless (off-screen) rendering in VSG without requiring a window or display. This is essential for server-side rendering, automated testing, batch processing, and cloud-based rendering applications.

## Key VSG Features Used

- **Headless rendering** - No window or surface required
- **Custom framebuffer** - Manual render target creation
- **Off-screen image capture** - Direct rendering to memory
- **vsg::RenderGraph** - Rendering without a window
- **Manual device creation** - No window-based device
- **Image transfer operations** - GPU to CPU memory transfer
- **Format conversion** - Optional blit-based conversion

## What the Example Demonstrates

1. **Headless Setup**
   - Creating Vulkan instance without surface extensions
   - Manual device and queue selection
   - Custom render targets (color and depth)

2. **Framebuffer Creation**
   - Color attachment with transfer source capability
   - Depth attachment for proper rendering
   - Custom extent independent of any window

3. **Render and Capture Pipeline**
   - Render scene to off-screen framebuffer
   - Transfer rendered image to CPU memory
   - Save result to disk

4. **Memory Management**
   - Host-visible memory for CPU access
   - Proper image layout transitions
   - Efficient data transfer

## Code Highlights

```cpp
// Create color image for off-screen rendering
vsg::ref_ptr<vsg::ImageView> createColorImageView(
    vsg::ref_ptr<vsg::Device> device, 
    const VkExtent2D& extent, 
    VkFormat imageFormat, 
    VkSampleCountFlagBits samples)
{
    auto colorImage = vsg::Image::create();
    colorImage->format = imageFormat;
    colorImage->extent = {extent.width, extent.height, 1};
    colorImage->samples = samples;
    colorImage->tiling = VK_IMAGE_TILING_OPTIMAL;
    colorImage->usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | 
                       VK_IMAGE_USAGE_TRANSFER_SRC_BIT;
    
    return vsg::createImageView(device, colorImage, VK_IMAGE_ASPECT_COLOR_BIT);
}

// Create depth buffer
auto depthImageView = createDepthImageView(device, extent, depthFormat, samples);

// Set up framebuffer attachments
vsg::RenderPass::Attachments attachments{
    {// Color attachment
        imageFormat,                               // format
        samples,                                   // samples
        VK_ATTACHMENT_LOAD_OP_CLEAR,              // loadOp
        VK_ATTACHMENT_STORE_OP_STORE,             // storeOp
        VK_ATTACHMENT_LOAD_OP_DONT_CARE,          // stencilLoadOp
        VK_ATTACHMENT_STORE_OP_DONT_CARE,         // stencilStoreOp
        VK_IMAGE_LAYOUT_UNDEFINED,                // initialLayout
        VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL   // finalLayout
    },
    {// Depth attachment
        depthFormat,
        samples,
        VK_ATTACHMENT_LOAD_OP_CLEAR,
        VK_ATTACHMENT_STORE_OP_DONT_CARE,
        VK_ATTACHMENT_LOAD_OP_DONT_CARE,
        VK_ATTACHMENT_STORE_OP_DONT_CARE,
        VK_IMAGE_LAYOUT_UNDEFINED,
        VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL
    }
};

// Create framebuffer for off-screen rendering
auto framebuffer = vsg::Framebuffer::create(
    renderPass, 
    vsg::ImageViews{colorImageView, depthImageView}, 
    extent.width, extent.height, 1
);

// Create render graph without a window
auto renderGraph = vsg::RenderGraph::create();
renderGraph->framebuffer = framebuffer;
renderGraph->renderPass = renderPass;
renderGraph->renderArea.extent = extent;
renderGraph->clearValues = {
    {{{clearColor.r, clearColor.g, clearColor.b, clearColor.a}}},
    {{{1.0f, 0}}}
};

// Add scene to render
auto view = vsg::View::create(camera, scenegraph);
renderGraph->addChild(view);

// Create command graph without window
auto commandGraph = vsg::CommandGraph::create(device, queueFamily);
commandGraph->addChild(renderGraph);

// Record and submit commands
vsg::submitCommandGraphsToQueue(commandGraph, queue);

// Capture rendered image
auto [commands, destinationImage] = createColorCapture(
    device, extent, colorImageView->image, imageFormat
);

// Execute capture commands
vsg::submitCommandsToQueue(commandPool, fence, waitTime, queue, 
    [&](vsg::CommandBuffer& commandBuffer) {
        commands->record(commandBuffer);
    }
);

// Map and save image data
auto imageData = vsg::MappedData<vsg::ubvec4Array2D>::create(
    deviceMemory, offset, 0, 
    vsg::Data::Properties{targetImageFormat}, 
    width, height
);

vsg::write(imageData, outputFilename, options);
```

## Running the Example

```bash
# Basic usage - render teapot to image
./vsgheadless

# Custom output filename
./vsgheadless --output screenshot.png

# Custom resolution
./vsgheadless --extent 1920 1080

# Load specific model
./vsgheadless model.vsgt --output render.png

# Custom clear color
./vsgheadless --clear-color 0.2 0.2 0.3 1.0

# Enable validation
./vsgheadless --debug
```

### Output

The example produces an image file containing the rendered scene without displaying any window.

## Key Takeaways

- Headless rendering doesn't require a window or display surface
- Manual framebuffer creation replaces swapchain framebuffers
- The same rendering pipeline works for both windowed and headless
- Proper image usage flags enable both rendering and transfer
- This technique is essential for server-side and batch rendering
- No platform-specific window system integration is needed
- The rendered image can be captured and saved like screenshots
- Headless rendering is useful for automated testing and CI/CD