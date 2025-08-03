# vsgrendertotexturearray

## Overview

The `vsgrendertotexturearray` example demonstrates advanced render-to-texture techniques using 2D texture arrays in VSG. It renders the same scene from multiple viewpoints into different layers of a texture array, then displays all rendered textures in a grid layout. This technique is useful for shadow mapping, reflection probes, and multi-view rendering.

## Key VSG Features Used

- **2D Texture Arrays** - Multi-layer textures with array indexing
- **Layer-specific rendering** - Target individual array layers
- **Custom traversal nodes** - TraverseChildrenOfNode for scene reuse
- **Multiple command graphs** - Coordinated rendering pipeline
- **Shared view IDs** - Pipeline optimization across views
- **Event handling nodes** - EventNode for custom interaction
- **Performance statistics** - Pipeline and view counting

## What the Example Demonstrates

1. **Texture Array Creation**
   - Creating 2D texture arrays with multiple layers
   - Configuring usage flags for render targets and sampling
   - Managing memory allocation for array textures

2. **Per-Layer Rendering**
   - Individual framebuffers targeting specific array layers
   - Camera configuration for each rendering pass
   - Efficient scene graph reuse across layers

3. **Pipeline Optimization**
   - Shared view IDs to reduce pipeline duplication
   - Command graph ordering for dependencies
   - Statistics collection for performance analysis

4. **Results Visualization**
   - Grid layout for displaying all rendered textures
   - Second window showing texture array contents
   - Interactive navigation of results

## Code Highlights

```cpp
// Create 2D texture array for multiple layers
vsg::ref_ptr<vsg::Image> createImage(vsg::Context& context, uint32_t width, uint32_t height, 
                                    uint32_t layers, VkFormat format, VkImageUsageFlags usage)
{
    auto image = vsg::Image::create();
    image->imageType = VK_IMAGE_TYPE_2D;
    image->format = format;
    image->extent = VkExtent3D{width, height, 1};
    image->mipLevels = 1;
    image->arrayLayers = layers;  // Key: Multiple layers
    image->samples = VK_SAMPLE_COUNT_1_BIT;
    image->tiling = VK_IMAGE_TILING_OPTIMAL;
    image->usage = usage;
    image->initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    
    image->compile(context);
    return image;
}

// Create layer-specific image view and framebuffer
vsg::ref_ptr<vsg::RenderGraph> createOffscreenRendergraph(
    vsg::Context& context, const VkExtent2D& extent, uint32_t layer,
    vsg::ref_ptr<vsg::Image> colorImage, vsg::ImageInfo& colorImageInfo,
    vsg::ref_ptr<vsg::Image> depthImage, vsg::ImageInfo& depthImageInfo)
{
    // Create image view for specific layer
    auto colorImageView = vsg::ImageView::create(colorImage, VK_IMAGE_ASPECT_COLOR_BIT);
    colorImageView->subresourceRange.baseArrayLayer = layer;  // Target specific layer
    colorImageView->subresourceRange.layerCount = 1;         // Single layer view
    colorImageView->compile(context);

    // Create sampler for texture access
    auto colorSampler = vsg::Sampler::create();
    colorSampler->magFilter = VK_FILTER_LINEAR;
    colorSampler->minFilter = VK_FILTER_LINEAR;
    colorSampler->addressModeU = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
    colorSampler->addressModeV = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;

    // Setup image info for shader access
    colorImageInfo.imageView = colorImageView;
    colorImageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    colorImageInfo.sampler = colorSampler;

    // Create depth view for same layer
    auto depthImageView = vsg::ImageView::create(depthImage, VK_IMAGE_ASPECT_DEPTH_BIT);
    depthImageView->subresourceRange.baseArrayLayer = layer;
    depthImageView->subresourceRange.layerCount = 1;
    depthImageView->compile(context);

    // Create render pass with proper dependencies
    vsg::RenderPass::Attachments attachments(2);
    attachments[0].format = colorImage->format;
    attachments[0].finalLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    attachments[1].format = depthImage->format;
    attachments[1].finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    // Critical dependencies for render-to-texture
    vsg::RenderPass::Dependencies dependencies(2);
    
    // Dependency for using previous texture as input
    dependencies[0].srcSubpass = VK_SUBPASS_EXTERNAL;
    dependencies[0].dstSubpass = 0;
    dependencies[0].srcStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[0].dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[0].srcAccessMask = VK_ACCESS_SHADER_READ_BIT;
    dependencies[0].dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

    // Dependency for subsequent texture usage
    dependencies[1].srcSubpass = 0;
    dependencies[1].dstSubpass = VK_SUBPASS_EXTERNAL;
    dependencies[1].srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[1].dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[1].srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[1].dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    auto renderPass = vsg::RenderPass::create(context.device, attachments, subpasses, dependencies);
    auto framebuffer = vsg::Framebuffer::create(renderPass, 
                                               vsg::ImageViews{colorImageView, depthImageView}, 
                                               extent.width, extent.height, 1);

    auto renderGraph = vsg::RenderGraph::create();
    renderGraph->framebuffer = framebuffer;
    renderGraph->renderArea.extent = extent;
    renderGraph->clearValues = {
        VkClearValue{{{0.2f, 0.2f, 0.4f, 1.0f}}},  // Clear color
        VkClearValue{{{0.0f, 0}}}                   // Clear depth
    };

    return renderGraph;
}

// Custom traversal node to reuse scene content without view overhead
class TraverseChildrenOfNode : public vsg::Inherit<vsg::Node, TraverseChildrenOfNode>
{
public:
    explicit TraverseChildrenOfNode(vsg::ref_ptr<vsg::Node> in_node) : node(in_node) {}
    vsg::observer_ptr<vsg::Node> node;

    template<class N, class V>
    static void t_traverse(N& in_node, V& visitor)
    {
        if (auto ref_node = in_node.node.ref_ptr()) ref_node->traverse(visitor);
    }

    void traverse(Visitor& visitor) override { t_traverse(*this, visitor); }
    void traverse(ConstVisitor& visitor) const override { t_traverse(*this, visitor); }
    void traverse(RecordTraversal& visitor) const override { t_traverse(*this, visitor); }
};

// Setup multiple render-to-texture passes
auto colorImage = createImage(*context, targetExtent.width, targetExtent.height, 
                             numLayers, VK_FORMAT_R8G8B8A8_SRGB, 
                             VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT);
auto depthImage = createImage(*context, targetExtent.width, targetExtent.height, 
                             numLayers, VK_FORMAT_D32_SFLOAT, 
                             VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT);

// Use TraverseChildrenOfNode to avoid camera/ViewDependentState overhead
auto tcon = vsg::TraverseChildrenOfNode::create(view3D);

auto rtt_commandGraph = vsg::CommandGraph::create(window);
rtt_commandGraph->submitOrder = -1;  // Render before main

vsg::ref_ptr<vsg::View> first_rrt_view;
for (uint32_t layer = 0; layer < numLayers; ++layer)
{
    auto offscreenCamera = createCameraForScene(vsg_scene, targetExtent);
    auto colorImageInfo = vsg::ImageInfo::create();
    auto depthImageInfo = vsg::ImageInfo::create();
    
    auto rtt_RenderGraph = createOffscreenRendergraph(*context, targetExtent, layer, 
                                                     colorImage, *colorImageInfo, 
                                                     depthImage, *depthImageInfo);

    // Optimize pipelines by sharing view IDs
    vsg::ref_ptr<vsg::View> rtt_view;
    if (shareViewID && first_rrt_view)
    {
        rtt_view = vsg::View::create(*first_rrt_view);  // Copy view with same ID
        rtt_view->camera = offscreenCamera;              // Different camera
    }
    else
    {
        rtt_view = vsg::View::create(offscreenCamera, tcon);
        if (!first_rrt_view) first_rrt_view = rtt_view;
    }

    rtt_RenderGraph->addChild(rtt_view);
    rtt_commandGraph->addChild(rtt_RenderGraph);
    imageInfos.push_back(colorImageInfo);
}

// Create results window showing texture grid
auto resultsCommandGraph = createResultsWindow(device, width/2, height/2, imageInfos, presentMode);

// Performance statistics collection
struct CollectStats : public vsg::ConstVisitor
{
    std::map<const vsg::GraphicsPipeline*, uint32_t> pipelines;
    std::map<const vsg::View*, uint32_t> views;

    void apply(const vsg::BindGraphicsPipeline& bgp) override
    {
        ++pipelines[bgp.pipeline.get()];
    }

    void apply(const vsg::View& view) override
    {
        ++views[&view];
        view.traverse(*this);
    }
};

// Event handling with custom EventNode
class EventNode : public vsg::Inherit<vsg::Node, EventNode>
{
public:
    EventHandlers eventHandlers;
    
    explicit EventNode(ref_ptr<Visitor> eventHandler)
    {
        if (eventHandler) eventHandlers.push_back(eventHandler);
    }
};
```

## Running the Example

```bash
# Basic usage - renders to 4 texture layers
./vsgrendertotexturearray model.vsgt

# Specify number of layers
./vsgrendertotexturearray model.vsgt --numLayers 8

# Enable shared view IDs for pipeline optimization
./vsgrendertotexturearray model.vsgt -s

# Camera animation path
./vsgrendertotexturearray model.vsgt -p camera_path.vsgb

# Window configuration
./vsgrendertotexturearray model.vsgt --window 1920 1080

# Performance testing
./vsgrendertotexturearray model.vsgt --test
./vsgrendertotexturearray model.vsgt -f 1000

# Debug options
./vsgrendertotexturearray model.vsgt --debug
./vsgrendertotexturearray model.vsgt --api
```

### Visual Output

The example creates two windows:
1. **Main window**: Shows the 3D scene with interactive navigation
2. **Results window**: Grid display of all rendered texture layers

Each texture layer shows the scene from a slightly different viewpoint, demonstrating the array texture capabilities.

## Key Features

### Texture Array Benefits

- **Memory efficiency**: Single allocation for multiple textures
- **Reduced binding overhead**: One texture binding for all layers
- **Shader array indexing**: Access any layer with array index
- **Consistent format and size**: All layers have identical properties

### Pipeline Optimization

When `shareViewID` is enabled:
- Views share the same view ID
- Graphics pipelines are reused across views
- Significantly reduces Vulkan pipeline objects
- Improves performance and memory usage

### Render Dependencies

The example demonstrates proper render pass dependencies:
- Previous texture reads wait for writes to complete
- Color attachment writes finish before shader reads
- Critical for avoiding race conditions in GPU

## Key Takeaways

- Texture arrays enable efficient multi-target rendering with minimal overhead
- Layer-specific image views allow targeting individual array elements
- Shared view IDs dramatically reduce pipeline duplication
- TraverseChildrenOfNode provides efficient scene graph reuse
- Proper render pass dependencies are essential for render-to-texture
- Command graph ordering controls rendering sequence
- This technique is fundamental for shadow mapping, reflection probes, and VR rendering
- Performance statistics help identify pipeline optimization opportunities