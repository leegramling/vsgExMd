# vsgtypes

## Overview

The `vsgtypes` example demonstrates VSG's fundamental data types, including vectors, matrices, and their mapping to Vulkan formats. It shows type introspection, mathematical operations, and how VSG types correspond to GPU data formats.

## Key VSG Features Used

- **Vector types** - vec2, vec3, vec4 and their variants
- **Matrix types** - mat4 and transformation utilities
- **Type introspection** - Runtime type identification
- **Vulkan format mapping** - Mapping C++ types to VkFormat
- **Mathematical functions** - radians, rotate, translate, dot product
- **Template vector types** - t_vec2, t_vec3, t_vec4 templates

## What the Example Demonstrates

1. **Type System Overview**
   - Float vectors: vec2, vec3, vec4
   - Double vectors: dvec2, dvec3, dvec4
   - Integer vectors: ivec2, ivec3, ivec4
   - Unsigned vectors: uivec2, uivec3, uivec4
   - Byte vectors: ucvec4 (unsigned char)

2. **Vulkan Format Mapping**
   - Maps C++ types to corresponding Vulkan formats
   - Essential for vertex attribute descriptions
   - Ensures type safety between CPU and GPU

3. **Mathematical Operations**
   - Angle conversion (degrees to radians)
   - Matrix transformations (rotate, translate)
   - Vector operations (dot product, length)

## Code Highlights

```cpp
// Type introspection
std::cout << "typeid(vsg::vec3) = " << typeid(vsg::vec3).name() << std::endl;

// Vulkan format mapping
VkFormatTypeMap[std::type_index(typeid(vsg::vec3))] = VK_FORMAT_R32G32B32_SFLOAT;
VkFormatTypeMap[std::type_index(typeid(vsg::ucvec4))] = VK_FORMAT_R8G8B8A8_UINT;

// Template function to get format from type
template<typename T>
VkFormat VkFormatForType() {
    return VkFormatTypeMap[std::type_index(typeid(T))];
}

// Mathematical operations
constexpr float rad1 = vsg::radians(90.0f);  // Compile-time conversion
auto matrix = vsg::rotate(vsg::radians(90.0), 0.0, 0.0, 1.0);  // Rotation matrix

// Vector operations
constexpr vsg::dvec3 v = vsg::dvec3(1.0, 0.0, 0.0);
std::cout << "length(v) = " << vsg::length(v) << std::endl;
std::cout << "dot product = " << vsg::dot(v, vsg::dvec3(2.0, 0.0, 0.0)) << std::endl;
```

## Running the Example

```bash
./vsgtypes
```

### Expected Output

```
typeid(char) = c
typeid(unsigned char) = h
typeid(vsg::vec2) = N3vsg6t_vec2IfEE
typeid(vsg::vec3) = N3vsg6t_vec3IfEE
typeid(vsg::vec4) = N3vsg6t_vec4IfEE
VkFormat 49  // VK_FORMAT_R8_SINT for char
VkFormat 13  // VK_FORMAT_R8_UINT for unsigned char
VkFormat 100 // VK_FORMAT_R32_SFLOAT for float
VkFormat 103 // VK_FORMAT_R32G32_SFLOAT for vec2
VkFormat 106 // VK_FORMAT_R32G32B32_SFLOAT for vec3
VkFormat 109 // VK_FORMAT_R32G32B32A32_SFLOAT for vec4
rad1=1.5708
matrix = [[0, -1, 0, 0], [1, 0, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]
length(v) = 1
dot product = 2
```

## Key Takeaways

- VSG provides a complete set of vector and matrix types for graphics programming
- Types are templated, allowing different precision (float, double) and element types
- Direct mapping to Vulkan formats ensures efficient GPU data transfer
- Mathematical operations support both runtime and compile-time evaluation
- Type introspection allows runtime type queries and format lookups
- The type system is designed for both safety and performance