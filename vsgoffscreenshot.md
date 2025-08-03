# vsgoffscreenshot

## Overview

The `vsgoffscreenshot` example demonstrates advanced offscreen rendering techniques for capturing high-quality screenshots at arbitrary resolutions. It shows how to render to offscreen framebuffers, transfer image data to CPU memory, and save screenshots while maintaining the interactive display.

## Key VSG Features Used

- **Offscreen rendering** - Render to custom framebuffers
- **Image transfer operations** - Copy/blit between GPU images  
- **Memory mapping** - Access GPU memory from CPU
- **Custom render passes** - Specialized for image capture
- **Multi-sampling support** - MSAA for high-quality captures
- **Format conversion** - Handle different image formats
- **Dynamic viewport management** - Runtime resolution changes
- **Pipeline barriers** - Synchronize GPU operations

## What the Example Demonstrates

1. **Offscreen Framebuffer Creation**
   - Custom render passes for image transfer
   - Framebuffer setup with proper attachments
   - Multi-sampling and resolve operations

2. **Image Data Transfer**
   - GPU to CPU memory transfer
   - Format conversion and blitting
   - Optimal vs linear tiling handling

3. **Interactive Screenshot Capture**
   - Keyboard-triggered screenshot saving
   - Dynamic resolution synchronization
   - Independent offscreen and display rendering

4. **Advanced Memory Management**
   - Mapped device memory access
   - Row pitch handling for image data
   - Proper memory barriers and synchronization

## Code Highlights

```cpp
// Check if device supports format blitting
bool supportsBlit(vsg::ref_ptr<vsg::Device> device, VkFormat format)
{
    auto physicalDevice = device->getPhysicalDevice();
    VkFormatProperties srcFormatProperties;
    vkGetPhysicalDeviceFormatProperties(*(physicalDevice), format, &srcFormatProperties);
    VkFormatProperties destFormatProperties;
    vkGetPhysicalDeviceFormatProperties(*(physicalDevice), VK_FORMAT_R8G8B8A8_SRGB, &destFormatProperties);
    
    return ((srcFormatProperties.optimalTilingFeatures & VK_FORMAT_FEATURE_BLIT_SRC_BIT) != 0) &&
           ((destFormatProperties.linearTilingFeatures & VK_FORMAT_FEATURE_BLIT_DST_BIT) != 0);
}

// Create capture image with host-visible memory
vsg::ref_ptr<vsg::Image> createCaptureImage(
    vsg::ref_ptr<vsg::Device> device,
    VkFormat sourceFormat,
    const VkExtent2D& extent)
{
    // Use RGBA if blitting is supported, otherwise use source format
    auto targetFormat = supportsBlit(device, sourceFormat) ? VK_FORMAT_R8G8B8A8_SRGB : sourceFormat;

    auto image = vsg::Image::create();
    image->format = targetFormat;
    image->extent = {extent.width, extent.height, 1};
    image->tiling = VK_IMAGE_TILING_LINEAR;  // For CPU access
    image->usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT;
    image->compile(device);

    // Allocate host-visible memory
    auto memReqs = image->getMemoryRequirements(device->deviceID);
    auto memFlags = VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT;
    auto deviceMemory = vsg::DeviceMemory::create(device, memReqs, memFlags);
    image->bind(deviceMemory, 0);

    return image;
}

// Create transfer commands with proper synchronization
vsg::ref_ptr<vsg::Commands> createTransferCommands(
    vsg::ref_ptr<vsg::Device> device,
    vsg::ref_ptr<vsg::Image> sourceImage,
    vsg::ref_ptr<vsg::Image> destinationImage)
{
    auto commands = vsg::Commands::create();

    // Transition destination image to transfer layout
    auto transitionBarrier = vsg::ImageMemoryBarrier::create(
        0,                                    // srcAccessMask
        VK_ACCESS_TRANSFER_WRITE_BIT,        // dstAccessMask
        VK_IMAGE_LAYOUT_UNDEFINED,           // oldLayout
        VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, // newLayout
        VK_QUEUE_FAMILY_IGNORED,
        VK_QUEUE_FAMILY_IGNORED,
        destinationImage,
        VkImageSubresourceRange{VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1}
    );

    commands->addChild(vsg::PipelineBarrier::create(
        VK_PIPELINE_STAGE_TRANSFER_BIT,
        VK_PIPELINE_STAGE_TRANSFER_BIT,
        0, transitionBarrier));

    if (sourceImage->format == destinationImage->format && 
        sourceImage->extent.width == destinationImage->extent.width &&
        sourceImage->extent.height == destinationImage->extent.height)
    {
        // Direct copy - same format and size
        VkImageCopy region{};
        region.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        region.srcSubresource.layerCount = 1;
        region.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        region.dstSubresource.layerCount = 1;
        region.extent = destinationImage->extent;

        auto copyImage = vsg::CopyImage::create();
        copyImage->srcImage = sourceImage;
        copyImage->srcImageLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
        copyImage->dstImage = destinationImage;
        copyImage->dstImageLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
        copyImage->regions.push_back(region);
        commands->addChild(copyImage);
    }
    else if (supportsBlit(device, destinationImage->format))
    {
        // Blit for format/size conversion
        VkImageBlit region{};
        region.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        region.srcSubresource.layerCount = 1;
        region.srcOffsets[1] = VkOffset3D{
            static_cast<int32_t>(sourceImage->extent.width),
            static_cast<int32_t>(sourceImage->extent.height), 1};
        region.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        region.dstSubresource.layerCount = 1;
        region.dstOffsets[1] = VkOffset3D{
            static_cast<int32_t>(destinationImage->extent.width),
            static_cast<int32_t>(destinationImage->extent.height), 1};

        auto blitImage = vsg::BlitImage::create();
        blitImage->srcImage = sourceImage;
        blitImage->srcImageLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
        blitImage->dstImage = destinationImage;
        blitImage->dstImageLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
        blitImage->regions.push_back(region);
        blitImage->filter = VK_FILTER_NEAREST;
        commands->addChild(blitImage);
    }

    // Transition to general layout for CPU access
    auto finalTransition = vsg::ImageMemoryBarrier::create(
        VK_ACCESS_TRANSFER_WRITE_BIT,        // srcAccessMask
        VK_ACCESS_MEMORY_READ_BIT,           // dstAccessMask
        VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, // oldLayout
        VK_IMAGE_LAYOUT_GENERAL,             // newLayout
        VK_QUEUE_FAMILY_IGNORED,
        VK_QUEUE_FAMILY_IGNORED,
        destinationImage,
        VkImageSubresourceRange{VK_IMAGE_ASPECT_COLOR_BIT, 0, 1, 0, 1}
    );

    commands->addChild(vsg::PipelineBarrier::create(
        VK_PIPELINE_STAGE_TRANSFER_BIT,
        VK_PIPELINE_STAGE_TRANSFER_BIT,
        0, finalTransition));

    return commands;
}

// Custom render pass for offscreen rendering
vsg::ref_ptr<vsg::RenderPass> createTransferRenderPass(
    vsg::ref_ptr<vsg::Device> device,
    VkFormat imageFormat,
    VkFormat depthFormat,
    bool requiresDepthRead)
{
    auto colorAttachment = vsg::defaultColorAttachment(imageFormat);
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL; // Key difference
    
    auto depthAttachment = vsg::defaultDepthAttachment(depthFormat);
    if (requiresDepthRead)
    {
        depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    }

    vsg::RenderPass::Attachments attachments{colorAttachment, depthAttachment};
    // ... subpass and dependency setup
    
    return vsg::RenderPass::create(device, attachments, subpasses, dependencies);
}

// Access image data from GPU memory
vsg::ref_ptr<vsg::Data> getImageData(
    vsg::ref_ptr<vsg::Viewer> viewer, 
    vsg::ref_ptr<vsg::Device> device, 
    vsg::ref_ptr<vsg::Image> captureImage)
{
    constexpr uint64_t waitTimeout = 1000000000; // 1 second
    viewer->waitForFences(0, waitTimeout);

    // Get subresource layout information
    VkImageSubresource subResource{VK_IMAGE_ASPECT_COLOR_BIT, 0, 0};
    VkSubresourceLayout subResourceLayout;
    vkGetImageSubresourceLayout(*device, captureImage->vk(device->deviceID), 
                               &subResource, &subResourceLayout);

    auto deviceMemory = captureImage->getDeviceMemory(device->deviceID);
    size_t destRowWidth = captureImage->extent.width * sizeof(vsg::ubvec4);

    if (destRowWidth == subResourceLayout.rowPitch)
    {
        // Contiguous memory - direct mapping
        return vsg::MappedData<vsg::ubvec4Array2D>::create(
            deviceMemory, subResourceLayout.offset, 0,
            vsg::Data::Properties{captureImage->format},
            captureImage->extent.width, captureImage->extent.height);
    }
    else
    {
        // Non-contiguous - copy row by row
        auto mappedData = vsg::MappedData<vsg::ubyteArray>::create(
            deviceMemory, subResourceLayout.offset, 0,
            vsg::Data::Properties{captureImage->format},
            subResourceLayout.rowPitch * captureImage->extent.height);
            
        auto imageData = vsg::ubvec4Array2D::create(
            captureImage->extent.width, captureImage->extent.height,
            vsg::Data::Properties{captureImage->format});
            
        for (uint32_t row = 0; row < captureImage->extent.height; ++row)
        {
            std::memcpy(
                imageData->dataPointer(row * captureImage->extent.width),
                mappedData->dataPointer(row * subResourceLayout.rowPitch),
                destRowWidth);
        }
        return imageData;
    }
}

// Interactive screenshot handler
class ScreenshotHandler : public vsg::Inherit<vsg::Visitor, ScreenshotHandler>
{
public:
    bool do_sync_extent = false;
    bool do_image_capture = false;

    void apply(vsg::KeyPressEvent& keyPress) override
    {
        if (keyPress.keyBase == 'e')
        {
            do_sync_extent = true;  // Sync offscreen size to display
        }
        if (keyPress.keyBase == 's')
        {
            do_image_capture = true;  // Trigger screenshot
        }
    }
};
```

## Running the Example

```bash
# Basic usage
./vsgoffscreenshot model.vsgt

# Specify capture filename
./vsgoffscreenshot model.vsgt --capture-file output.png

# Enable multi-sampling for high quality
./vsgoffscreenshot model.vsgt --msaa

# Command graph variations
./vsgoffscreenshot model.vsgt --nested      # Nested command graphs
./vsgoffscreenshot model.vsgt -s            # Separate command graphs

# Debug options
./vsgoffscreenshot model.vsgt --debug
./vsgoffscreenshot model.vsgt --api
```

### Interactive Controls

- **'s' key**: Capture screenshot and save to file
- **'e' key**: Sync offscreen rendering resolution to display resolution

### Command Graph Options

1. **Default**: Single command graph with offscreen and display render graphs
2. **Nested (`--nested`)**: Offscreen rendering nested within main command graph
3. **Separate (`-s`)**: Independent command graphs for offscreen and display

## Key Features

### Multi-Sampling Support

When `--msaa` is enabled:
- Uses VK_SAMPLE_COUNT_8_BIT for high-quality rendering
- Requires Vulkan 1.2 for proper depth resolve
- Creates resolve attachments for final image output

### Format Handling

- Attempts to use RGBA format for consistency
- Falls back to source format if blitting unsupported
- Handles format conversion during transfer operations

### Memory Management

- Host-visible memory for CPU access
- Proper row pitch handling for GPU memory layout
- Automatic memory mapping/unmapping

## Key Takeaways

- Offscreen rendering enables high-quality screenshot capture at any resolution
- Proper memory barriers and layout transitions are critical for image transfer
- Format support varies by GPU - always check blit capabilities
- Memory layout differences require careful handling of row pitch
- VSG's MappedData simplifies GPU memory access from CPU
- Interactive resolution changes require recreating framebuffers and images
- This technique is essential for batch rendering and automated image generation