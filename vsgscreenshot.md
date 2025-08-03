# vsgscreenshot

## Overview

The `vsgscreenshot` example demonstrates how to capture screenshots from a VSG application, including both color and depth buffer captures. It shows the proper way to read back GPU data, handle image format conversions, and save images to disk.

## Key VSG Features Used

- **Swapchain image access** - Reading from rendered frames
- **Image layout transitions** - Proper Vulkan synchronization
- **vsg::BlitImage/CopyImage** - GPU image transfer operations
- **Device memory mapping** - Reading GPU data to CPU
- **Format conversion** - Handling different image formats
- **Command buffer recording** - One-off commands for transfer
- **Event synchronization** - Optional GPU/CPU sync
- **Depth buffer readback** - Capturing depth information

## What the Example Demonstrates

1. **Color Buffer Capture**
   - Access previous frame's swapchain image
   - Check for blit support and format compatibility
   - Transfer image from GPU to CPU memory
   - Save as image file

2. **Depth Buffer Capture**
   - Read depth/stencil attachment
   - Convert depth values to grayscale
   - Handle platform-specific depth formats

3. **Synchronization**
   - Image layout transitions
   - Pipeline barriers
   - Optional event-based synchronization

4. **Format Handling**
   - Automatic format conversion via blit
   - Linear vs optimal tiling
   - Row pitch considerations

## Code Highlights

```cpp
// Screenshot handler responds to key presses
class ScreenshotHandler : public vsg::Inherit<vsg::Visitor, ScreenshotHandler>
{
    void apply(vsg::KeyPressEvent& keyPress) override
    {
        if (keyPress.keyBase == 's') do_image_capture = true;
        if (keyPress.keyBase == 'd') do_depth_capture = true;
    }
};

// Get previous frame's image (current frame not rendered yet)
auto sourceImage = window->imageView(window->imageIndex(1))->image;

// Check if blit is supported for format conversion
VkFormatProperties srcFormatProperties;
vkGetPhysicalDeviceFormatProperties(physicalDevice, sourceImageFormat, 
                                   &srcFormatProperties);
bool supportsBlit = (srcFormatProperties.optimalTilingFeatures & 
                    VK_FORMAT_FEATURE_BLIT_SRC_BIT) != 0;

// Create destination image in linear tiling for CPU access
auto destinationImage = vsg::Image::create();
destinationImage->format = targetImageFormat;
destinationImage->extent = {width, height, 1};
destinationImage->tiling = VK_IMAGE_TILING_LINEAR;
destinationImage->usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT;

// Image layout transitions
auto transitionSourceImage = vsg::ImageMemoryBarrier::create(
    VK_ACCESS_MEMORY_READ_BIT,              // srcAccessMask
    VK_ACCESS_TRANSFER_READ_BIT,            // dstAccessMask
    VK_IMAGE_LAYOUT_PRESENT_SRC_KHR,        // oldLayout
    VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,   // newLayout
    sourceImage
);

// Transfer operation (blit or copy)
if (supportsBlit)
{
    auto blitImage = vsg::BlitImage::create();
    blitImage->srcImage = sourceImage;
    blitImage->dstImage = destinationImage;
    blitImage->regions.push_back(region);
    commands->addChild(blitImage);
}
else
{
    auto copyImage = vsg::CopyImage::create();
    copyImage->srcImage = sourceImage;
    copyImage->dstImage = destinationImage;
    commands->addChild(copyImage);
}

// Map and read image data
VkSubresourceLayout subResourceLayout;
vkGetImageSubresourceLayout(device, destinationImage, 
                           &subResource, &subResourceLayout);

// Handle row pitch differences
if (destRowWidth == subResourceLayout.rowPitch)
{
    // Direct mapping
    imageData = vsg::MappedData<vsg::ubvec4Array2D>::create(
        deviceMemory, offset, 0, props, width, height);
}
else
{
    // Copy with row pitch handling
    auto mappedData = vsg::MappedData<vsg::ubyteArray>::create(...);
    // Copy row by row
}

// Save to file
vsg::write(imageData, colorFilename, options);
```

## Running the Example

```bash
# Basic usage
./vsgscreenshot model.vsgt

# Specify output filenames
./vsgscreenshot model.vsgt --color screenshot.png --depth depth.png

# Enable event synchronization
./vsgscreenshot model.vsgt --event

# Test event synchronization
./vsgscreenshot model.vsgt --event-test

# With debugging
./vsgscreenshot model.vsgt --debug
```

### Controls

- **S key**: Capture color buffer screenshot
- **D key**: Capture depth buffer screenshot
- **Mouse**: Navigate scene before capture

### Output Files

- **Color screenshot**: RGBA image of rendered scene
- **Depth screenshot**: Grayscale visualization of depth buffer

## Key Takeaways

- Screenshot capture requires accessing the previous frame's image
- Proper image layout transitions are crucial for Vulkan
- Blit operations can perform format conversion automatically
- Linear tiling is required for CPU access to image data
- Row pitch must be considered when mapping image memory
- The swapchain image must be transitioned back to present layout
- Depth buffer capture requires special handling and format conversion
- One-off command buffers are used for transfer operations