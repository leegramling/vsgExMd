# vsgarrays

## Overview

The `vsgarrays` example demonstrates VSG's type-safe array classes, which are fundamental data containers used throughout the scene graph for storing vertices, colors, texture coordinates, and other bulk data.

## Key VSG Features Used

- **vsg::Array** - Template base class for type-safe arrays
- **vsg::floatArray** - Array of float values
- **vsg::vec2Array** - Array of 2D vectors
- **vsg::vec4Array** - Array of 4D vectors (often used for colors)
- **Initializer list construction** - Modern C++ initialization
- **STL-compatible iterators** - Range-based for loops and algorithms

## What the Example Demonstrates

1. **Array Creation and Initialization**
   - Creating arrays with specified sizes
   - Initializing arrays with initializer lists
   - Default initialization

2. **Array Manipulation**
   - Using STL algorithms with VSG arrays
   - Iterator-based access
   - Index-based access

3. **Type Safety**
   - Each array type is strongly typed
   - Compile-time type checking
   - Efficient storage for graphics data

## Code Highlights

```cpp
// Create a float array with 10 elements
auto floats = vsg::floatArray::create(10);

// Initialize with STL algorithm
float value = 0.0f;
std::for_each(floats->begin(), floats->end(), [&value](float& v) {
    v = value++;
});

// Create vec4 array (for colors) with 40 elements
auto colours = vsg::vec4Array::create(40);

// Index-based access
for (std::size_t i = 0; i < colours->size(); ++i)
{
    (*colours)[i] = colour;
}

// Create array with initializer list
auto texCoords = vsg::vec2Array::create(
    {{1.0f, 2.0f},
     {3.0f, 4.0f},
     {}});  // Default-initialized element

// Range-based for loop
for (auto p : *texCoords)
{
    std::cout << "    tc " << p.x << ", " << p.y << std::endl;
}
```

## Running the Example

```bash
./vsgarrays
```

### Expected Output

```
floats.size() = 10
   v[] = 0
   v[] = 1
   v[] = 2
   ...
   c[] = [0.25, 0.5, 0.75, 1]
   c[] = [0.5, 0.75, 1, 0.25]
   ...
texCoords.size() = 3
    tc 1, 2
    tc 3, 4
    tc 0, 0
col.size() = 1
    colour 0, 0, 0, 0
```

## Key Takeaways

- VSG arrays are reference-counted like all VSG objects
- Arrays provide STL-compatible interfaces (iterators, size(), etc.)
- Type-safe arrays prevent common errors in graphics programming
- Arrays can be initialized with C++ initializer lists
- Default initialization creates zero-initialized elements
- Arrays are the primary way to store vertex attributes, indices, and other bulk data
- Memory for arrays is managed by VSG's allocator system