# vsgpointer

## Overview

The `vsgpointer` example demonstrates VSG's smart pointer system and memory management fundamentals. It compares VSG's `ref_ptr` with standard C++ `shared_ptr`, showing memory layout, alignment, and size characteristics of VSG objects.

## Key VSG Features Used

- **vsg::ref_ptr** - VSG's reference-counted smart pointer
- **vsg::Object** - Base class for all VSG objects
- **vsg::Auxiliary** - Auxiliary data storage mechanism
- **vsg::QuadGroup** - Specialized group node with exactly 4 children
- **Memory layout analysis** - Size and alignment characteristics

## What the Example Demonstrates

1. **Smart Pointer Comparison**
   - Shows the size difference between `vsg::ref_ptr` (8 bytes) and `std::shared_ptr` (16 bytes)
   - Demonstrates why VSG uses its own smart pointer implementation for efficiency

2. **Object Memory Layout**
   - Displays sizes of fundamental VSG types
   - Shows alignment requirements for different types
   - Checks standard layout properties

3. **Reference Counting**
   - VSG uses intrusive reference counting (counter is part of the object)
   - More memory efficient than `shared_ptr`'s external control block

## Code Highlights

```cpp
// VSG's ref_ptr usage
vsg::ref_ptr<vsg::QuadGroup> ref_node = vsg::QuadGroup::create();

// Size comparison
sizeof(vsg::ref_ptr) = 8 bytes    // Just a pointer
sizeof(std::shared_ptr) = 16 bytes // Pointer + control block pointer
```

## Running the Example

```bash
./vsgpointer
```

### Expected Output

```
sizeof(Object)=24
sizeof(atomic_uint)=4
sizeof(atomic_ushort)=2
sizeof(Auxiliary)=56
sizeof(vsg::ref_ptr)=8, sizeof(QuadGroup)=224
sizeof(std::shared_ptr)=16, sizeof(SharedPtrQuadGroup)=56
...
```

## Key Takeaways

- VSG's `ref_ptr` is more memory efficient than `std::shared_ptr`
- VSG objects have intrusive reference counting built into the base `Object` class
- The reference count uses atomic operations for thread safety
- VSG's design prioritizes memory efficiency and cache performance