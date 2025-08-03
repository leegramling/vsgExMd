# vsgcolorspace

## Overview

The `vsgcolorspace` example demonstrates color space management in VSG, specifically showing the differences between linear and sRGB color spaces. It creates multiple windows with different surface formats to showcase how various Vulkan color formats affect rendering and provides visual comparison of color space conversions.

## Key VSG Features Used

- **Multiple windows** - Different surface formats per window
- **Surface format enumeration** - Query supported formats
- **Color space conversions** - sRGB/linear transformations
- **vsg::Builder** - Geometric primitive creation
- **Text rendering** - Labels for visual comparison
- **Image format manipulation** - Runtime format conversion

## What the Example Demonstrates

1. **Color Space Concepts**
   - Visual difference between linear and sRGB color spaces
   - Impact of surface format on color reproduction
   - Color space conversion functions

2. **Multiple Surface Formats**
   - Automatic window creation for each supported format
   - Side-by-side comparison of different formats
   - Surface format capability querying

3. **Runtime Format Conversion**
   - Converting images between sRGB and linear formats
   - Demonstrating format transformation effects
   - Proper handling of different color encodings

## Code Highlights

```cpp
// Query all supported surface formats
auto physicalDevice = window->getOrCreatePhysicalDevice();
auto surface = window->getOrCreateSurface();
auto swapChainSupportDetails = vsg::querySwapChainSupport(physicalDevice->vk(), surface->vk());

// Create a window for each supported surface format
for(auto& format : swapChainSupportDetails.formats)
{
    auto local_windowTraits = vsg::WindowTraits::create();
    local_windowTraits->windowTitle = vsg::make_string(
        "VkSurfaceFormatKHR{ VkFormat format = ", format.format, 
        ", VkColorSpaceKHR colorSpace = ", format.colorSpace, "}"
    );
    local_windowTraits->swapchainPreferences.surfaceFormat = format;
    
    auto local_window = vsg::Window::create(local_windowTraits);
    if (local_window) windows.push_back(local_window);
}

// Create spheres showing linear color progression
auto createSphere = [&](float intensity) -> vsg::ref_ptr<vsg::Node>
{
    vsg::GeometryInfo geomInfo;
    geomInfo.color.set(intensity, intensity, intensity, 1.0f);
    return builder->createSphere(geomInfo, stateInfo);
};

// Linear color values row
Row linearRow;
linearRow.push_back(ObjectLabel(createSphere(0.0f), "linear(0.0f)"));
linearRow.push_back(ObjectLabel(createSphere(0.25f), "linear(0.25f)"));
linearRow.push_back(ObjectLabel(createSphere(0.5f), "linear(0.5f)"));
linearRow.push_back(ObjectLabel(createSphere(0.75f), "linear(0.75f)"));
linearRow.push_back(ObjectLabel(createSphere(1.0f), "linear(1.0f)"));

// sRGB to linear conversion row
Row sRGBToLinearRow;
sRGBToLinearRow.push_back(ObjectLabel(createSphere(vsg::sRGB_to_linear(0.0f)), "sRGB_to_linear(0.0f)"));
sRGBToLinearRow.push_back(ObjectLabel(createSphere(vsg::sRGB_to_linear(0.25f)), "sRGB_to_linear(0.25f)"));
sRGBToLinearRow.push_back(ObjectLabel(createSphere(vsg::sRGB_to_linear(0.5f)), "sRGB_to_linear(0.5f)"));
sRGBToLinearRow.push_back(ObjectLabel(createSphere(vsg::sRGB_to_linear(0.75f)), "sRGB_to_linear(0.75f)"));
sRGBToLinearRow.push_back(ObjectLabel(createSphere(vsg::sRGB_to_linear(1.0f)), "sRGB_to_linear(1.0f)"));

// Linear to sRGB conversion row
Row linearToSRGBRow;
linearToSRGBRow.push_back(ObjectLabel(createSphere(vsg::linear_to_sRGB(0.0f)), "linear_to_sRGB(0.0f)"));
linearToSRGBRow.push_back(ObjectLabel(createSphere(vsg::linear_to_sRGB(0.25f)), "linear_to_sRGB(0.25f)"));
linearToSRGBRow.push_back(ObjectLabel(createSphere(vsg::linear_to_sRGB(0.5f)), "linear_to_sRGB(0.5f)"));
linearToSRGBRow.push_back(ObjectLabel(createSphere(vsg::linear_to_sRGB(0.75f)), "linear_to_sRGB(0.75f)"));
linearToSRGBRow.push_back(ObjectLabel(createSphere(vsg::linear_to_sRGB(1.0f)), "linear_to_sRGB(1.0f)"));

// Image format conversion demonstration
auto image = vsg::read_cast<vsg::Data>("textures/lz.vsgb", options);
if (image)
{
    // Convert to linear format
    auto image_uNorm = vsg::clone(image);
    image_uNorm->properties.format = vsg::sRGB_to_uNorm(image->properties.format);
    
    // Convert to sRGB format
    auto image_sRGB = vsg::clone(image);
    image_sRGB->properties.format = vsg::uNorm_to_sRGB(image->properties.format);
    
    // Create textured quads for comparison
    Row imageRow;
    imageRow.push_back(ObjectLabel(createTextureQuad(image, options), 
        vsg::make_string("image default loaded format: ", image->properties.format)));
    imageRow.push_back(ObjectLabel(createTextureQuad(image_uNorm, options), 
        vsg::make_string("image treated as linear: ", image_uNorm->properties.format)));
    imageRow.push_back(ObjectLabel(createTextureQuad(image_sRGB, options), 
        vsg::make_string("image treated as sRGB: ", image_sRGB->properties.format)));
}

// Create labeled subgraphs with text
vsg::ref_ptr<vsg::Node> createLabelledSubgraph(
    const vsg::dvec3& position, 
    const vsg::dvec3& dimensions, 
    vsg::ref_ptr<vsg::Object> object, 
    const std::string& label, 
    vsg::ref_ptr<vsg::Options> options)
{
    // Create text label
    auto text = vsg::Text::create();
    text->font = vsg::read_cast<vsg::Font>("fonts/times.vsgb", options);
    
    auto layout = vsg::StandardLayout::create();
    layout->position = position;
    layout->color = vsg::vec4(1.0, 1.0, 1.0, 1.0);
    layout->horizontalAlignment = vsg::StandardLayout::CENTER_ALIGNMENT;
    text->layout = layout;
    text->text = vsg::stringValue::create(label);
    
    // Combine object and label
    auto group = vsg::Group::create();
    group->addChild(transform);
    group->addChild(text);
    return group;
}
```

## Running the Example

```bash
# Basic usage - shows color space comparison
./vsgcolorspace

# Load additional models for comparison
./vsgcolorspace model1.vsgt model2.gltf

# Window options
./vsgcolorspace --window 800 600
./vsgcolorspace --fullscreen

# Performance testing
./vsgcolorspace --small-test
./vsgcolorspace --test

# Debug options
./vsgcolorspace --debug
./vsgcolorspace --api

# Color space specific options
./vsgcolorspace --rgb  # Map RGB to RGBA hint disabled
```

### Visual Output

The example creates multiple windows showing:

1. **First window**: Default surface format
2. **Additional windows**: Each supported surface format
3. **Scene content**: Grid of objects demonstrating:
   - Row 1: Linear color progression (0.0, 0.25, 0.5, 0.75, 1.0)
   - Row 2: sRGB to linear converted values
   - Row 3: Linear to sRGB converted values
   - Row 4: Image with different format treatments
   - Row 5+: User-loaded models

### Color Space Formats

Common surface formats the example demonstrates:
- **VK_FORMAT_B8G8R8A8_SRGB** - sRGB color space
- **VK_FORMAT_B8G8R8A8_UNORM** - Linear color space
- **VK_FORMAT_R8G8B8A8_SRGB** - sRGB with different channel order
- **VK_FORMAT_A2B10G10R10_UNORM_PACK32** - High precision linear

## Key Takeaways

- Color space significantly affects visual appearance of rendered content
- sRGB and linear color spaces require different handling
- VSG provides automatic color space conversion functions
- Surface format choice impacts color reproduction
- Multiple windows can display different format interpretations simultaneously
- Text labels help identify differences that might be subtle
- Proper color space management is crucial for consistent rendering
- VSG's format conversion utilities simplify color space workflows
- Different monitors and graphics cards may support different format sets