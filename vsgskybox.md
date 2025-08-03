# vsgskybox

## Overview

The `vsgskybox` example demonstrates how to create a skybox (environment map) in VSG using cube mapping. A skybox provides an infinite background by rendering a textured cube around the camera that moves with the view but doesn't translate, creating the illusion of a distant environment.

## Key VSG Features Used

- **Cube map textures** - 6-sided environment textures
- **Custom shaders** - Skybox-specific vertex and fragment shaders
- **Depth testing configuration** - Render behind everything
- **Front-face culling** - Render inside of cube
- **View matrix manipulation** - Remove translation component
- **samplerCube** - Cube map sampling in shaders

## What the Example Demonstrates

1. **Skybox Rendering Technique**
   - Creating a cube that surrounds the camera
   - Removing translation from view matrix
   - Setting depth to maximum (render behind everything)

2. **Cube Map Usage**
   - Loading cube map texture data
   - Using samplerCube in fragment shader
   - 3D texture coordinates for sampling

3. **Rendering Order**
   - Depth test with GREATER_OR_EQUAL
   - Depth write disabled
   - Front face culling (render inside of cube)

## Code Highlights

```cpp
// Create skybox geometry - a simple cube
auto vertices = vsg::vec3Array::create({
    // Back face
    {-1.0f, -1.0f, -1.0f}, {1.0f, -1.0f, -1.0f},
    {-1.0f,  1.0f, -1.0f}, {1.0f,  1.0f, -1.0f},
    // ... other faces
});

// Skybox-specific render state
auto rasterState = vsg::RasterizationState::create();
rasterState->cullMode = VK_CULL_MODE_FRONT_BIT;  // Render inside of cube

auto depthState = vsg::DepthStencilState::create();
depthState->depthTestEnable = VK_TRUE;
depthState->depthWriteEnable = VK_FALSE;  // Don't write to depth buffer
depthState->depthCompareOp = VK_COMPARE_OP_GREATER_OR_EQUAL;  // Behind everything

// Load cube map texture
auto data = vsg::read_cast<vsg::Data>(filename, options);
auto sampler = vsg::Sampler::create();
sampler->maxLod = data->properties.maxNumMipmaps;

auto texture = vsg::DescriptorImage::create(
    sampler, data, 0, 0, 
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER
);

// Vertex shader - removes translation from view matrix
const auto skybox_vert = R"(
    layout(push_constant) uniform PushConstants {
        mat4 projection;
        mat4 modelView;
    } pc;
    
    layout(location = 0) in vec3 vsg_Vertex;
    layout(location = 0) out vec3 UVW;
    
    void main()
    {
        UVW = vsg_Vertex;  // Use position as texture coordinate
        
        // Remove translation component
        mat4 modelView = pc.modelView;
        modelView[3] = vec4(0.0, 0.0, 0.0, 1.0);
        
        vec4 pos = pc.projection * modelView * vec4(vsg_Vertex, 1.0);
        gl_Position = vec4(pos.xy, 0.0, pos.w);  // Set z to 0 (maps to 1 after perspective divide)
    }
)";

// Fragment shader - samples cube map
const auto skybox_frag = R"(
    layout(binding = 0) uniform samplerCube envMap;
    layout(location = 0) in vec3 UVW;
    layout(location = 0) out vec4 outColor;
    
    void main()
    {
        outColor = textureLod(envMap, UVW, 0);
    }
)";

// Integrate skybox with scene
auto commandGraph = vsg::CommandGraph::create(window);

// First: Render skybox
auto skyboxView = vsg::View::create(camera, skybox);
auto skyboxRenderGraph = vsg::RenderGraph::create(window, skyboxView);
commandGraph->addChild(skyboxRenderGraph);

// Second: Render main scene (will appear in front of skybox)
auto sceneView = vsg::View::create(camera, scene);
auto sceneRenderGraph = vsg::RenderGraph::create(window, sceneView);
commandGraph->addChild(sceneRenderGraph);
```

## Running the Example

```bash
# Basic usage with default skybox
./vsgskybox

# Load custom cube map
./vsgskybox --cubemap path/to/cubemap.ktx

# Load a 3D model to render in front of skybox
./vsgskybox model.vsgt --cubemap skybox.ktx

# Window options
./vsgskybox --window 1920 1080
./vsgskybox --fullscreen

# Debug options
./vsgskybox --debug
./vsgskybox --api
```

### Cube Map Texture Format

The example expects a cube map texture with 6 faces in the order:
1. Positive X (Right)
2. Negative X (Left)
3. Positive Y (Top)
4. Negative Y (Bottom)
5. Positive Z (Front)
6. Negative Z (Back)

## Key Takeaways

- Skyboxes create infinite backgrounds using cube mapping
- The view matrix translation must be removed to keep skybox centered
- Depth settings ensure skybox renders behind all other geometry
- Front-face culling renders the inside of the skybox cube
- Vertex positions double as 3D texture coordinates
- The z component is set to 0 in clip space (becomes 1 after perspective divide)
- Cube maps can be stored in formats like KTX that support all 6 faces
- Skyboxes are essential for realistic outdoor environments and reflections