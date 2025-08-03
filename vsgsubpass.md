# vsgsubpass

## Overview

The `vsgsubpass` example demonstrates Vulkan subpass functionality in VSG, showing how to create render passes with multiple subpasses and coordinate rendering operations within a single render pass. This technique allows for efficient GPU memory bandwidth usage by keeping intermediate results in tile memory rather than writing to main memory.

## Key VSG Features Used

- **Custom render passes** - Multi-subpass render pass creation
- **Subpass dependencies** - Proper synchronization between subpasses
- **vsg::NextSubPass** - Transition between subpasses
- **Multiple graphics pipelines** - Different pipelines per subpass
- **Subpass-specific pipeline states** - Pipeline targeting specific subpass indices

## What the Example Demonstrates

1. **Multi-Subpass Render Pass**
   - Creating render passes with multiple subpass descriptions
   - Configuring subpass dependencies for proper synchronization
   - Attachment usage across different subpasses

2. **Subpass Transitions**
   - Using NextSubPass commands to move between subpasses
   - Maintaining render pass state across transitions
   - Coordinating different rendering operations

3. **Pipeline Subpass Targeting**
   - Creating pipelines that target specific subpass indices
   - Different rendering configurations per subpass
   - Resource sharing between subpasses

## Code Highlights

```cpp
// Create custom render pass with multiple subpasses
vsg::ref_ptr<vsg::RenderPass> createRenderPass(vsg::Device* device)
{
    VkFormat imageFormat = VK_FORMAT_B8G8R8A8_UNORM;
    VkFormat depthFormat = VK_FORMAT_D24_UNORM_S8_UINT;

    // Attachment descriptions
    vsg::AttachmentDescription colorAttachment = {};
    colorAttachment.format = imageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;

    vsg::AttachmentDescription depthAttachment = {};
    depthAttachment.format = depthFormat;
    depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    vsg::RenderPass::Attachments attachments{colorAttachment, depthAttachment};

    // Attachment references for subpasses
    vsg::AttachmentReference colorAttachmentRef = {};
    colorAttachmentRef.attachment = 0;  // Index into attachments array
    colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    vsg::AttachmentReference depthAttachmentRef = {};
    depthAttachmentRef.attachment = 1;  // Index into attachments array
    depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    // First subpass - depth testing enabled
    vsg::SubpassDescription depth_subpass;
    depth_subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    depth_subpass.colorAttachments.emplace_back(colorAttachmentRef);
    depth_subpass.depthStencilAttachments.emplace_back(depthAttachmentRef);

    // Second subpass - no depth testing (classic rendering)
    vsg::SubpassDescription classic_subpass;
    classic_subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    classic_subpass.colorAttachments.emplace_back(colorAttachmentRef);
    // Note: No depth attachment - disables depth testing

    vsg::RenderPass::Subpasses subpasses{depth_subpass, classic_subpass};

    // Subpass dependency - ensures proper ordering
    vsg::SubpassDependency classic_dependency = {};
    classic_dependency.srcSubpass = 0;  // First subpass
    classic_dependency.dstSubpass = 1;  // Second subpass
    classic_dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    classic_dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    classic_dependency.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT;
    classic_dependency.dstAccessMask = 0;
    classic_dependency.dependencyFlags = 0;

    vsg::RenderPass::Dependencies dependencies{classic_dependency};

    return vsg::RenderPass::create(device, attachments, subpasses, dependencies);
}

// Create graphics pipeline targeting specific subpass
vsg::ref_ptr<vsg::GraphicsPipeline> createPipelineForSubpass(
    uint32_t subpassIndex,
    vsg::ref_ptr<vsg::PipelineLayout> pipelineLayout,
    const vsg::ShaderStages& shaders,
    const vsg::GraphicsPipelineStates& states)
{
    // Key: Specify subpass index when creating pipeline
    auto graphicsPipeline = vsg::GraphicsPipeline::create(
        pipelineLayout, 
        shaders, 
        states, 
        subpassIndex  // Target specific subpass
    );
    
    return graphicsPipeline;
}

// Setup pipelines for different subpasses
vsg::ref_ptr<vsg::GraphicsPipeline> graphicsPipelinePass1;
{
    // Standard pipeline states
    vsg::GraphicsPipelineStates pipelineStates{
        vsg::VertexInputState::create(vertexBindings, vertexAttributes),
        vsg::InputAssemblyState::create(),
        vsg::RasterizationState::create(),
        vsg::MultisampleState::create(),
        vsg::ColorBlendState::create(),
        vsg::DepthStencilState::create()  // Depth testing enabled
    };

    auto pipelineLayout = vsg::PipelineLayout::create(layouts, pushConstants);
    
    // Target subpass 0 (depth subpass)
    graphicsPipelinePass1 = vsg::GraphicsPipeline::create(
        pipelineLayout, 
        vsg::ShaderStages{vertexShader, fragmentShader}, 
        pipelineStates,
        0  // Subpass index 0
    );
}

vsg::ref_ptr<vsg::GraphicsPipeline> graphicsPipelinePass2;
{
    // Same pipeline states configuration
    vsg::GraphicsPipelineStates pipelineStates{
        vsg::VertexInputState::create(vertexBindings, vertexAttributes),
        vsg::InputAssemblyState::create(),
        vsg::RasterizationState::create(),
        vsg::MultisampleState::create(),
        vsg::ColorBlendState::create(),
        vsg::DepthStencilState::create()
    };

    auto pipelineLayout = vsg::PipelineLayout::create(layouts, pushConstants);
    
    // Target subpass 1 (classic subpass)
    graphicsPipelinePass2 = vsg::GraphicsPipeline::create(
        pipelineLayout, 
        vsg::ShaderStages{vertexShader, fragmentShader}, 
        pipelineStates,
        1  // Subpass index 1
    );
}

// Setup scene graph with subpass transitions
auto scenegraph = vsg::StateGroup::create();

// First subpass content
auto scenegraph1 = vsg::StateGroup::create();
scenegraph1->add(vsg::BindGraphicsPipeline::create(graphicsPipelinePass1));
scenegraph1->add(vsg::BindDescriptorSets::create(/* ... */));

// Geometry for first subpass (cube with depth testing)
auto drawCommands1 = vsg::Commands::create();
drawCommands1->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{vertices1, colors1, texcoords1}));
drawCommands1->addChild(vsg::BindIndexBuffer::create(indices1));
drawCommands1->addChild(vsg::DrawIndexed::create(12, 1, 0, 0, 0));  // 12 indices for cube

auto transform = vsg::MatrixTransform::create();
transform->addChild(drawCommands1);
scenegraph1->addChild(transform);

// Second subpass content
auto scenegraph2 = vsg::StateGroup::create();
scenegraph2->add(vsg::BindGraphicsPipeline::create(graphicsPipelinePass2));
scenegraph2->add(vsg::BindDescriptorSets::create(/* ... */));

// Geometry for second subpass (triangle without depth testing)
auto drawCommands2 = vsg::Commands::create();
drawCommands2->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{vertices2, colors2, texcoords2}));
drawCommands2->addChild(vsg::BindIndexBuffer::create(indices2));
drawCommands2->addChild(vsg::DrawIndexed::create(3, 1, 0, 0, 0));   // 3 indices for triangle

scenegraph2->addChild(drawCommands2);

// Combine with subpass transition
scenegraph->addChild(scenegraph1);                                    // First subpass
scenegraph->addChild(vsg::NextSubPass::create(VK_SUBPASS_CONTENTS_INLINE));  // Transition
scenegraph->addChild(scenegraph2);                                    // Second subpass

// Set custom render pass on window
window->setRenderPass(createRenderPass(device));

// Animation in main loop
while (viewer->advanceToNextFrame())
{
    // Animate the transform (affects first subpass only)
    float time = std::chrono::duration<float>(viewer->getFrameStamp()->time - viewer->start_point()).count();
    transform->matrix = vsg::rotate(time * vsg::radians(90.0f), vsg::vec3(0.0f, 0.0, 1.0f));
    
    viewer->update();
    viewer->recordAndSubmit();
    viewer->present();
}
```

## Running the Example

```bash
# Basic usage
./vsgsubpass

# Window configuration
./vsgsubpass --window 1920 1080

# Debug options
./vsgsubpass --debug
./vsgsubpass --api
```

### Visual Output

The example renders:
1. **First subpass**: A rotating textured cube with depth testing enabled
2. **Subpass transition**: Automatic transition between subpasses
3. **Second subpass**: A static triangle rendered without depth testing (appears on top)

The triangle from the second subpass will always appear in front of the cube because the second subpass doesn't use depth testing.

## Key Concepts

### Subpass Benefits

- **Memory Bandwidth**: Intermediate results stay in tile memory
- **Performance**: Avoid expensive memory writes between passes
- **Mobile Optimization**: Particularly beneficial on tile-based renderers
- **Synchronization**: Automatic dependency handling within render pass

### Subpass Dependencies

```cpp
vsg::SubpassDependency dependency = {};
dependency.srcSubpass = 0;          // Source subpass index
dependency.dstSubpass = 1;          // Destination subpass index
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT;
dependency.dstAccessMask = 0;
```

### Pipeline Subpass Targeting

- Pipelines must specify which subpass they target
- Different subpasses can have different attachment configurations
- Depth testing availability depends on subpass depth attachment usage

## Use Cases

1. **Deferred Rendering**: G-buffer generation → lighting passes
2. **Post-Processing**: Scene rendering → effect passes
3. **UI Overlay**: 3D content → 2D UI elements
4. **Shadow Mapping**: Shadow generation → lit scene rendering
5. **Multi-pass Effects**: Multiple rendering techniques in sequence

## Key Takeaways

- Subpasses enable efficient multi-pass rendering within a single render pass
- NextSubPass commands coordinate transitions between subpasses
- Pipelines must target specific subpass indices during creation
- Subpass dependencies ensure proper synchronization without manual barriers
- This technique is essential for mobile and tile-based GPU optimization
- Attachment usage can vary between subpasses (depth testing on/off)
- VSG simplifies subpass management compared to raw Vulkan