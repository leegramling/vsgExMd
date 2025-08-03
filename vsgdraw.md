# vsgdraw

## Overview

The `vsgdraw` example demonstrates low-level Vulkan command buffer creation and manual draw commands in VSG. Originally developed as a port of the VulkanTutorial's depth buffering stage, this example shows how to create graphics pipelines, bind vertex data, and issue draw commands directly. At just 192 lines of code, it's significantly more concise than the original 1530-line VulkanTutorial equivalent while providing the same functionality.

## Key VSG Features Used

- **Graphics pipeline creation** - Manual Vulkan graphics pipeline setup
- **Command buffer management** - Direct draw command recording
- **Vertex buffer binding** - Multiple vertex attribute streams
- **Index buffer usage** - Indexed geometry rendering
- **Descriptor sets** - Texture binding and sampling
- **Push constants** - Efficient matrix data transfer
- **State groups** - Pipeline state management
- **Matrix transforms** - Object transformation and animation

## What the Example Demonstrates

1. **Manual Graphics Pipeline Setup**
   - Vertex input state configuration with multiple attribute streams
   - Shader stage creation and binding
   - Pipeline layout with descriptor sets and push constants
   - Complete graphics pipeline state object creation

2. **Direct Draw Command Recording**
   - Manual vertex buffer binding commands
   - Index buffer binding and indexed drawing
   - Command graph construction for rendering
   - Low-level command buffer management

3. **Multi-Stream Vertex Data**
   - Separate vertex, color, and texture coordinate streams
   - Proper vertex input binding descriptions
   - Multiple vertex buffer binding in single draw call

4. **Texture Integration**
   - Texture loading and descriptor creation
   - Sampler configuration and binding
   - Fragment shader texture sampling

## Code Highlights

```cpp
// Graphics pipeline setup with multiple vertex streams
vsg::VertexInputState::Bindings vertexBindingsDescriptions{
    VkVertexInputBindingDescription{0, sizeof(vsg::vec3), VK_VERTEX_INPUT_RATE_VERTEX}, // vertex data
    VkVertexInputBindingDescription{1, sizeof(vsg::vec3), VK_VERTEX_INPUT_RATE_VERTEX}, // colour data
    VkVertexInputBindingDescription{2, sizeof(vsg::vec2), VK_VERTEX_INPUT_RATE_VERTEX}  // tex coord data
};

vsg::VertexInputState::Attributes vertexAttributeDescriptions{
    VkVertexInputAttributeDescription{0, 0, VK_FORMAT_R32G32B32_SFLOAT, 0}, // vertex data
    VkVertexInputAttributeDescription{1, 1, VK_FORMAT_R32G32B32_SFLOAT, 0}, // colour data
    VkVertexInputAttributeDescription{2, 2, VK_FORMAT_R32G32_SFLOAT, 0}     // tex coord data
};

// Complete graphics pipeline creation
vsg::GraphicsPipelineStates pipelineStates{
    vsg::VertexInputState::create(vertexBindingsDescriptions, vertexAttributeDescriptions),
    vsg::InputAssemblyState::create(),
    vsg::RasterizationState::create(),
    vsg::MultisampleState::create(),
    vsg::ColorBlendState::create(),
    vsg::DepthStencilState::create()
};

auto pipelineLayout = vsg::PipelineLayout::create(vsg::DescriptorSetLayouts{descriptorSetLayout}, pushConstantRanges);
auto graphicsPipeline = vsg::GraphicsPipeline::create(pipelineLayout, vsg::ShaderStages{vertexShader, fragmentShader}, pipelineStates);
```

```cpp
// Manual draw command creation
auto drawCommands = vsg::Commands::create();
drawCommands->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{vertices, colors, texcoords}));
drawCommands->addChild(vsg::BindIndexBuffer::create(indices));
drawCommands->addChild(vsg::DrawIndexed::create(12, 1, 0, 0, 0));

// Scene graph structure with state management
auto scenegraph = vsg::StateGroup::create();
scenegraph->add(bindGraphicsPipeline);
scenegraph->add(bindDescriptorSet);

auto transform = vsg::MatrixTransform::create();
scenegraph->addChild(transform);
transform->addChild(drawCommands);
```

```cpp
// Vertex data definition - cube geometry
auto vertices = vsg::vec3Array::create(
    {{-0.5f, -0.5f, 0.0f},
     {0.5f, -0.5f, 0.0f},
     {0.5f, 0.5f, 0.0f},
     {-0.5f, 0.5f, 0.0f},
     {-0.5f, -0.5f, -0.5f},
     {0.5f, -0.5f, -0.5f},
     {0.5f, 0.5f, -0.5f},
     {-0.5f, 0.5f, -0.5f}});

auto colors = vsg::vec3Array::create(
    {{1.0f, 0.0f, 0.0f},  // Red
     {0.0f, 1.0f, 0.0f},  // Green
     {0.0f, 0.0f, 1.0f},  // Blue
     {1.0f, 1.0f, 1.0f},  // White
     {1.0f, 0.0f, 0.0f},  // Red
     {0.0f, 1.0f, 0.0f},  // Green
     {0.0f, 0.0f, 1.0f},  // Blue
     {1.0f, 1.0f, 1.0f}}); // White

auto texcoords = vsg::vec2Array::create(
    {{0.0f, 0.0f}, {1.0f, 0.0f}, {1.0f, 1.0f}, {0.0f, 1.0f},
     {0.0f, 0.0f}, {1.0f, 0.0f}, {1.0f, 1.0f}, {0.0f, 1.0f}});

// Index buffer for two textured quads
auto indices = vsg::ushortArray::create(
    {0, 1, 2, 2, 3, 0,    // Front face
     4, 5, 6, 6, 7, 4});  // Back face
```

```cpp
// Texture loading and descriptor setup
auto textureData = vsg::read_cast<vsg::Data>(vsg::findFile(textureFile, searchPaths));
auto texture = vsg::DescriptorImage::create(vsg::Sampler::create(), textureData, 0, 0, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER);

auto descriptorSet = vsg::DescriptorSet::create(descriptorSetLayout, vsg::Descriptors{texture});
auto bindDescriptorSet = vsg::BindDescriptorSet::create(VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, descriptorSet);
```

```cpp
// Animation in main loop
while (viewer->advanceToNextFrame())
{
    viewer->handleEvents();
    
    // Animate the transform - continuous rotation around Z-axis
    float time = std::chrono::duration<float, std::chrono::seconds::period>(viewer->getFrameStamp()->time - viewer->start_point()).count();
    transform->matrix = vsg::rotate(time * vsg::radians(90.0f), vsg::vec3(0.0f, 0.0, 1.0f));
    
    viewer->update();
    viewer->recordAndSubmit();
    viewer->present();
}
```

## Running the Example

**Prerequisites**: Set the `VSG_FILE_PATH` environment variable to the vsgExamples/data directory:

```bash
# Linux/macOS
export VSG_FILE_PATH=/path/to/vsgExamples/data

# Windows
set VSG_FILE_PATH=C:\path\to\vsgExamples\data
```

```bash
# Basic usage - displays two rotating textured quads
./vsgdraw

# Debug mode with Vulkan validation layers
./vsgdraw --debug

# API dump mode - logs all Vulkan calls
./vsgdraw --api

# Custom window size
./vsgdraw --window 1024 768

# Combined options
./vsgdraw --debug --window 1280 720
```

### Expected Output

The example renders two textured quads that continuously rotate around the Z-axis. Each quad has:
- Different colored vertices (red, green, blue, white corners)
- Texture mapping from the loaded texture file
- Depth buffering for proper 3D rendering
- Smooth rotation animation at 90 degrees per second

## Key Low-Level Concepts

### Manual Command Buffer Creation

Unlike higher-level VSG examples, vsgdraw shows direct command buffer management:

```cpp
// Explicit command creation and binding
auto drawCommands = vsg::Commands::create();
drawCommands->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{vertices, colors, texcoords}));
drawCommands->addChild(vsg::BindIndexBuffer::create(indices));
drawCommands->addChild(vsg::DrawIndexed::create(12, 1, 0, 0, 0));
```

### Pipeline State Management

Complete control over Vulkan pipeline state:

```cpp
vsg::GraphicsPipelineStates pipelineStates{
    vsg::VertexInputState::create(vertexBindingsDescriptions, vertexAttributeDescriptions),
    vsg::InputAssemblyState::create(),        // Triangle assembly
    vsg::RasterizationState::create(),        // Rasterization settings
    vsg::MultisampleState::create(),          // MSAA configuration
    vsg::ColorBlendState::create(),           // Color blending
    vsg::DepthStencilState::create()          // Depth testing
};
```

### Push Constants for Matrices

Efficient matrix data transfer using push constants:

```cpp
vsg::PushConstantRanges pushConstantRanges{
    {VK_SHADER_STAGE_VERTEX_BIT, 0, 128} // projection, view, and model matrices
};
```

## VulkanTutorial Comparison

| Aspect | VulkanTutorial (1530 lines) | vsgdraw (192 lines) |
|--------|----------------------------|---------------------|
| **Setup** | Manual Vulkan initialization | VSG automatic setup |
| **Memory Management** | Manual buffer/image creation | VSG automatic management |
| **Pipeline Creation** | Verbose state specification | Concise VSG objects |
| **Command Recording** | Manual command buffer recording | VSG command graph |
| **Resource Management** | Manual cleanup required | Automatic via smart pointers |

## Key Takeaways

- VSG dramatically simplifies Vulkan development while maintaining low-level control
- Manual draw commands provide fine-grained control over rendering
- Multi-stream vertex data enables complex geometry with separate attributes
- Graphics pipeline creation in VSG is much more concise than raw Vulkan
- State groups provide hierarchical state management
- Push constants offer efficient matrix transfer for transformations
- VSG's command system bridges high-level scene graphs with low-level Vulkan commands
- This example serves as an excellent foundation for understanding VSG's Vulkan integration