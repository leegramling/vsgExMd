# vsginterleaved

## Overview

The `vsginterleaved` example demonstrates how to use interleaved vertex data in VSG, where multiple vertex attributes (position, color, texture coordinates) are stored consecutively in a single array rather than in separate arrays. This approach can improve memory performance by increasing cache locality and reducing the number of vertex buffer bindings required during rendering.

## Key VSG Features Used

- **Interleaved vertex arrays** - Single array containing multiple vertex attributes
- **Vertex input state configuration** - Multiple attributes from single binding
- **Memory layout optimization** - Cache-friendly data organization
- **Vertex buffer binding** - Single buffer for all vertex data
- **Graphics pipeline setup** - Vertex input layout specification
- **Indexed rendering** - Efficient geometry rendering with index buffers

## What the Example Demonstrates

1. **Interleaved Vertex Data Layout**
   - Position, color, and texture coordinates in single array
   - Proper stride and offset calculations for vertex attributes
   - Memory-efficient vertex data organization
   - Cache-optimal data access patterns

2. **Vertex Input State Configuration**
   - Single vertex binding for multiple attributes
   - Correct attribute offset specification
   - Stride calculations for interleaved data
   - Vulkan vertex input descriptor setup

3. **Performance Benefits**
   - Reduced memory bandwidth through better cache locality
   - Fewer vertex buffer binding commands
   - Improved GPU memory access patterns
   - Streamlined vertex processing pipeline

## Code Highlights

```cpp
// Interleaved vertex data - position (3), color (3), texcoord (2) = 8 floats per vertex
auto attributeArray = vsg::floatArray::create({
    // x,    y,    z,     r,    g,    b,     s,    t
    -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,  0.0f, 0.0f, // Vertex 0: position, red color, texcoord
     0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,  1.0f, 0.0f, // Vertex 1: position, green color, texcoord
     0.5f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f,  1.0f, 1.0f, // Vertex 2: position, blue color, texcoord
    -0.5f,  0.5f, 0.0f,  1.0f, 1.0f, 1.0f,  0.0f, 1.0f, // Vertex 3: position, white color, texcoord
    -0.5f, -0.5f, -0.5f, 1.0f, 0.0f, 0.0f,  0.0f, 0.0f, // Back face vertices...
     0.5f, -0.5f, -0.5f, 0.0f, 1.0f, 0.0f,  1.0f, 0.0f,
     0.5f,  0.5f, -0.5f, 0.0f, 0.0f, 1.0f,  1.0f, 1.0f,
    -0.5f,  0.5f, -0.5f, 1.0f, 1.0f, 1.0f,  0.0f, 1.0f
});
```

```cpp
// Vertex input state - single binding for all attributes
vsg::VertexInputState::Bindings vertexBindingsDescriptions{
    VkVertexInputBindingDescription{
        0,                              // binding index
        32,                             // stride: 8 floats * 4 bytes = 32 bytes per vertex
        VK_VERTEX_INPUT_RATE_VERTEX     // per-vertex data (not per-instance)
    }
};

// Multiple attributes from the same binding with different offsets
vsg::VertexInputState::Attributes vertexAttributeDescriptions{
    VkVertexInputAttributeDescription{0, 0, VK_FORMAT_R32G32B32_SFLOAT, 0},  // position: offset 0
    VkVertexInputAttributeDescription{1, 0, VK_FORMAT_R32G32B32_SFLOAT, 12}, // color: offset 12 (3 floats)
    VkVertexInputAttributeDescription{2, 0, VK_FORMAT_R32G32_SFLOAT, 24},    // texcoord: offset 24 (6 floats)
};
```

```cpp
// Single vertex buffer binding for all attributes
auto drawCommands = vsg::Commands::create();
drawCommands->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{attributeArray}));
drawCommands->addChild(vsg::BindIndexBuffer::create(indices));
drawCommands->addChild(vsg::DrawIndexed::create(12, 1, 0, 0, 0));
```

```cpp
// Data layout visualization per vertex (32 bytes total):
// [0-11]   Position (3 floats): x, y, z
// [12-23]  Color (3 floats): r, g, b  
// [24-31]  TexCoord (2 floats): s, t

// Memory layout example for first vertex:
// Bytes:  0   4   8   12  16  20  24  28
// Data:  [-0.5|-0.5|0.0 |1.0 |0.0 |0.0 |0.0 |0.0]
//         ^position^     ^color^     ^texcoord^
```

```cpp
// Comparison with separate arrays approach:
// SEPARATE ARRAYS (vsgdraw style):
auto vertices = vsg::vec3Array::create({{-0.5f, -0.5f, 0.0f}, ...});
auto colors = vsg::vec3Array::create({{1.0f, 0.0f, 0.0f}, ...});
auto texcoords = vsg::vec2Array::create({{0.0f, 0.0f}, ...});
drawCommands->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{vertices, colors, texcoords}));

// INTERLEAVED ARRAYS (vsginterleaved style):
auto attributeArray = vsg::floatArray::create({-0.5f, -0.5f, 0.0f, 1.0f, 0.0f, 0.0f, 0.0f, 0.0f, ...});
drawCommands->addChild(vsg::BindVertexBuffers::create(0, vsg::DataList{attributeArray}));
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
# Basic usage - displays rotating textured quads with interleaved vertex data
./vsginterleaved

# Debug mode with Vulkan validation layers
./vsginterleaved --debug

# API dump mode for detailed Vulkan call inspection
./vsginterleaved --api

# Custom window size
./vsginterleaved --window 1280 720

# Combined options
./vsginterleaved --debug --window 1024 768
```

### Expected Output

The example renders two textured quads (front and back faces) that continuously rotate around the Z-axis, similar to vsgdraw but using interleaved vertex data. Each vertex contains:
- **Position**: 3D coordinates (x, y, z)
- **Color**: RGB values creating colored corners
- **Texture coordinates**: UV mapping for texture application

## Memory Layout Analysis

### Interleaved Layout Benefits

1. **Cache Locality**
   ```
   Traditional (separate arrays):    Interleaved:
   Position Array: [x₁,y₁,z₁][x₂,y₂,z₂]...    [x₁,y₁,z₁,r₁,g₁,b₁,s₁,t₁][x₂,y₂,z₂,r₂,g₂,b₂,s₂,t₂]...
   Color Array:    [r₁,g₁,b₁][r₂,g₂,b₂]...    
   TexCoord Array: [s₁,t₁][s₂,t₂]...          
   ```

2. **Memory Access Pattern**
   - **Interleaved**: All vertex attributes loaded in single cache line
   - **Separate**: Multiple memory locations accessed per vertex
   - **Result**: Better cache utilization and reduced memory bandwidth

3. **Vulkan Binding Efficiency**
   - **Interleaved**: 1 vertex buffer binding call
   - **Separate**: 3 vertex buffer binding calls
   - **Result**: Reduced driver overhead and command buffer size

### Performance Considerations

**When Interleaved is Better**:
- Vertex shaders use all attributes for every vertex
- Memory bandwidth is the limiting factor
- Frequent vertex buffer binding changes are expensive
- Cache misses are common with separate arrays

**When Separate Arrays are Better**:
- Only subset of attributes used in specific shaders
- Different update frequencies for different attributes
- Easier data management and modification
- Better for dynamic vertex data updates

## Vertex Input State Deep Dive

### Binding Description

```cpp
VkVertexInputBindingDescription{
    0,                              // binding - which vertex buffer binding point
    32,                             // stride - bytes between consecutive vertices
    VK_VERTEX_INPUT_RATE_VERTEX     // inputRate - per-vertex (not per-instance)
}
```

### Attribute Descriptions

```cpp
// Format: {location, binding, format, offset}
VkVertexInputAttributeDescription{0, 0, VK_FORMAT_R32G32B32_SFLOAT, 0},   // vec3 position
VkVertexInputAttributeDescription{1, 0, VK_FORMAT_R32G32B32_SFLOAT, 12},  // vec3 color  
VkVertexInputAttributeDescription{2, 0, VK_FORMAT_R32G32_SFLOAT, 24},     // vec2 texcoord
```

**Key Points**:
- All attributes use `binding = 0` (same buffer)
- Offsets are cumulative: 0, 12 (3×4), 24 (6×4)
- Formats match shader input types exactly
- Total stride must accommodate all attributes

## Comparison with Other Examples

| Example | Vertex Layout | Bindings | Use Case |
|---------|---------------|----------|----------|
| **vsgdraw** | Separate arrays | 3 bindings | General-purpose, flexible |
| **vsginterleaved** | Single interleaved array | 1 binding | Cache-optimized, performance |
| **vsgexecutecommands** | Separate arrays | 3 bindings | Multi-window sharing |

## Key Takeaways

- Interleaved vertex data improves cache locality and memory bandwidth utilization
- Single vertex buffer binding reduces driver overhead
- Proper stride and offset calculations are critical for interleaved layouts
- Memory layout choice depends on access patterns and performance requirements
- VSG makes interleaved vertex data setup straightforward with clear attribute descriptions
- This approach is especially beneficial for static geometry with high vertex processing load
- Consider interleaved layouts when memory bandwidth is a performance bottleneck
- The example demonstrates optimal memory organization for vertex-intensive applications