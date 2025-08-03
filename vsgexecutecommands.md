# vsgexecutecommands

## Overview

The `vsgexecutecommands` example demonstrates Vulkan secondary command buffers and command execution optimization through VSG's `ExecuteCommands` functionality. This example shows how to share rendering commands between multiple windows efficiently by recording once and executing multiple times, reducing CPU overhead and improving performance in multi-window or multi-view applications.

## Key VSG Features Used

- **Secondary command buffers** - Reusable command buffer recording
- **ExecuteCommands** - Command buffer execution and sharing
- **Multi-window rendering** - Shared rendering across multiple windows
- **Device sharing** - Single device across multiple windows
- **Command graph creation** - Primary and secondary command buffer setup
- **Subpass contents** - Secondary command buffer integration
- **Threading support** - Multi-threaded rendering capability

## What the Example Demonstrates

1. **Command Buffer Optimization**
   - Recording commands once into secondary command buffers
   - Executing the same commands across multiple render targets
   - Reducing CPU overhead through command reuse
   - Vulkan best practices for command buffer management

2. **Multi-Window Architecture**
   - Creating multiple windows sharing the same device
   - Coordinated rendering across different windows
   - Shared scene graph and camera between windows
   - Window-specific command graph setup

3. **Flexible Rendering Modes**
   - ExecuteCommands mode for optimized multi-window rendering
   - Traditional mode with separate command graphs per window
   - Runtime switching between rendering approaches
   - Performance comparison capabilities

4. **Advanced Command Graph Setup**
   - Primary command buffers with secondary command buffer contents
   - Proper subpass configuration for secondary buffers
   - Command graph connection and execution flow

## Code Highlights

```cpp
// Multi-window setup with shared device
auto window1 = vsg::Window::create(traits);
auto traits2 = vsg::WindowTraits::create(*traits);
traits2->windowTitle = "vsgexecutecommands window2";
traits2->device = window1->getOrCreateDevice();  // Share device between windows
auto window2 = vsg::Window::create(traits2);

viewer->addWindow(window1);
viewer->addWindow(window2);
```

```cpp
// ExecuteCommands mode - optimized for command reuse
if (useExecuteCommands)
{
    std::cout << "Using SecondaryCommandGraph and ExecuteCommands" << std::endl;
    
    // Create secondary command graph for reusable commands
    auto secondaryCommandGraph = vsg::createSecondaryCommandGraphForView(window1, camera, vsg_scene, 0);
    
    // Create ExecuteCommands nodes that reference the secondary command graph
    auto scenegraphwin1 = vsg::Group::create();
    auto pass1 = vsg::ExecuteCommands::create();
    pass1->connect(secondaryCommandGraph);
    
    auto scenegraphwin2 = vsg::Group::create();
    auto pass2 = vsg::ExecuteCommands::create();
    pass2->connect(secondaryCommandGraph);
    
    scenegraphwin1->addChild(pass1);
    scenegraphwin2->addChild(pass2);
    
    // Create primary command graphs that execute secondary commands
    auto commandGraphwin1 = vsg::createCommandGraphForView(window1, camera, scenegraphwin1, 
                                                          VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
    auto commandGraphwin2 = vsg::createCommandGraphForView(window2, camera, scenegraphwin2, 
                                                          VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
    
    // Assign all command graphs to viewer
    viewer->assignRecordAndSubmitTaskAndPresentation({secondaryCommandGraph, commandGraphwin1, commandGraphwin2});
}
```

```cpp
// Traditional mode - separate command graphs per window
else
{
    std::cout << "Using Windows with separate full CommandGraph." << std::endl;
    
    // Each window gets its own complete command graph
    auto commandGraphwin1 = vsg::createCommandGraphForView(window1, camera, vsg_scene);
    auto commandGraphwin2 = vsg::createCommandGraphForView(window2, camera, vsg_scene);
    
    viewer->assignRecordAndSubmitTaskAndPresentation({commandGraphwin1, commandGraphwin2});
}
```

```cpp
// Scene creation function with complete graphics pipeline
vsg::ref_ptr<vsg::Node> createScene(vsg::ref_ptr<const vsg::Options> options)
{
    // Load shaders
    auto vertexShader = vsg::ShaderStage::read(VK_SHADER_STAGE_VERTEX_BIT, "main", 
                                              vsg::findFile("shaders/vert_PushConstants.spv", options));
    auto fragmentShader = vsg::ShaderStage::read(VK_SHADER_STAGE_FRAGMENT_BIT, "main", 
                                                vsg::findFile("shaders/frag_PushConstants.spv", options));
    
    // Graphics pipeline setup
    vsg::GraphicsPipelineStates pipelineStates{
        vsg::VertexInputState::create(vertexBindingsDescriptions, vertexAttributeDescriptions),
        vsg::InputAssemblyState::create(),
        vsg::RasterizationState::create(),
        vsg::MultisampleState::create(),
        vsg::ColorBlendState::create(),
        vsg::DepthStencilState::create()
    };
    
    auto graphicsPipeline = vsg::GraphicsPipeline::create(pipelineLayout, 
                                                         vsg::ShaderStages{vertexShader, fragmentShader}, 
                                                         pipelineStates);
    
    // State group with pipeline and descriptor bindings
    auto scenegraph = vsg::StateGroup::create();
    scenegraph->add(vsg::BindGraphicsPipeline::create(graphicsPipeline));
    scenegraph->add(vsg::BindDescriptorSets::create(VK_PIPELINE_BIND_POINT_GRAPHICS, 
                                                   graphicsPipeline->layout, 0, 
                                                   vsg::DescriptorSets{descriptorSet}));
    
    return scenegraph;
}
```

```cpp
// Camera setup with automatic bounds computation
vsg::ComputeBounds computeBounds;
vsg_scene->accept(computeBounds);
vsg::dvec3 centre = (computeBounds.bounds.min + computeBounds.bounds.max) * 0.5;
double radius = vsg::length(computeBounds.bounds.max - computeBounds.bounds.min) * 0.6;

// Position camera based on scene bounds
auto lookAt = vsg::LookAt::create(centre + vsg::dvec3(0.0, -radius * 3.5, 0.0), centre, vsg::dvec3(0.0, 0.0, 1.0));

// Handle different projection types
if (auto ellipsoidModel = vsg_scene->getRefObject<vsg::EllipsoidModel>("EllipsoidModel"))
{
    perspective = vsg::EllipsoidPerspective::create(lookAt, ellipsoidModel, 30.0, aspectRatio, nearFarRatio, horizonMountainHeight);
}
else
{
    perspective = vsg::Perspective::create(30.0, aspectRatio, nearFarRatio * radius, radius * 4.5);
}
```

## Running the Example

```bash
# Basic usage with ExecuteCommands optimization (default)
./vsgexecutecommands

# Load specific model file
./vsgexecutecommands model.vsgt

# Disable ExecuteCommands to use separate command graphs
./vsgexecutecommands --no-ec

# Enable multi-threading
./vsgexecutecommands --mt

# Custom window size
./vsgexecutecommands --window 1024 768

# Debug mode with Vulkan validation
./vsgexecutecommands --debug

# Camera animation from file
./vsgexecutecommands -p camera_animation.vsgb

# Immediate present mode for testing
./vsgexecutecommands --test

# Limit number of frames
./vsgexecutecommands -f 100
```

### Expected Output

The example displays the same 3D scene in two separate windows:
- **Window 1**: "vsgexecutecommands window1"
- **Window 2**: "vsgexecutecommands window2"

Both windows show identical content and respond to the same camera controls. The console output indicates which rendering mode is active:
- `"Using SecondaryCommandGraph and ExecuteCommands"` (default, optimized)
- `"Using Windows with separate full CommandGraph"` (with `--no-ec`)

## Performance Benefits

### ExecuteCommands Mode Advantages

1. **Reduced CPU Overhead**
   - Commands recorded once, executed multiple times
   - Less CPU time spent on command buffer recording
   - Better scalability with more windows/views

2. **Memory Efficiency**
   - Shared command buffers reduce memory usage
   - Less duplication of rendering state
   - Improved cache locality

3. **Synchronization Benefits**
   - Coordinated rendering across multiple targets
   - Easier management of shared resources
   - Reduced complexity in multi-window scenarios

### When to Use Each Mode

**ExecuteCommands Mode (Default)**:
- Multiple windows or views showing the same content
- Performance-critical applications
- Large, complex scenes with heavy command recording overhead

**Separate Command Graphs Mode**:
- Windows need different rendering settings
- Different post-processing per window
- Debugging or comparison purposes

## Key Secondary Command Buffer Concepts

### Primary vs Secondary Command Buffers

```cpp
// Primary command buffer - can execute secondary command buffers
auto primaryCommandGraph = vsg::createCommandGraphForView(window, camera, sceneGraph, 
                                                         VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);

// Secondary command buffer - contains reusable rendering commands
auto secondaryCommandGraph = vsg::createSecondaryCommandGraphForView(window, camera, scene, 0);
```

### Command Buffer Inheritance

Secondary command buffers inherit state from primary command buffers:
- Render pass and subpass information
- Framebuffer configuration
- Pipeline state inheritance

### ExecuteCommands Connection

```cpp
auto executeCommands = vsg::ExecuteCommands::create();
executeCommands->connect(secondaryCommandGraph);  // Link to reusable commands
```

## Key Takeaways

- ExecuteCommands provides significant performance benefits for multi-window applications
- Secondary command buffers enable efficient command reuse across render targets
- Device sharing between windows reduces resource overhead
- VSG abstracts complex Vulkan secondary command buffer management
- Command graphs can be mixed and matched for different rendering strategies
- Multi-threading support scales well with ExecuteCommands architecture
- Proper subpass configuration is essential for secondary command buffer integration
- This pattern is ideal for applications with multiple views of the same content