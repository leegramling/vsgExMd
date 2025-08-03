# vsgmemory

## Overview

The `vsgmemory` example demonstrates the performance characteristics of VSG's reference-counted memory management system, particularly when copying large containers of objects.

## Key VSG Features Used

- **vsg::ref_ptr** - Reference-counted smart pointers
- **vsg::Node** - Basic scene graph node
- **vsg::CommandLine** - Command-line argument parsing
- **Container operations** - Copying vectors of ref_ptr objects

## What the Example Demonstrates

1. **Reference Counting Performance**
   - Creates a large number of objects (default: 1 million)
   - Measures the time to copy containers of ref_ptr objects
   - Shows that copying ref_ptr is efficient (only increments reference count)

2. **Memory Efficiency**
   - Objects are shared, not duplicated when copying containers
   - Reference counting overhead is minimal

3. **Scalability Testing**
   - Tests VSG's ability to handle large numbers of objects
   - Demonstrates performance characteristics at scale

## Code Highlights

```cpp
// Create a large number of objects
using Objects = std::vector<vsg::ref_ptr<vsg::Object>>;
Objects objects;
objects.reserve(numObjects);
for (unsigned int i = 0; i < numObjects; ++i)
{
    objects.push_back(vsg::Node::create());
}

// Copy the container - only reference counts are updated
copy_objects = objects;  // Efficient: increments ref counts
copy_objects.clear();    // Decrements ref counts
```

## Running the Example

```bash
# Default: 1 million objects
./vsgmemory

# Custom number of objects
./vsgmemory --num-objects 5000000
./vsgmemory -n 10000000
```

### Expected Output

```
Time to copy container with 1000000 objects : 0.0123 seconds.
```

## Key Takeaways

- Copying `ref_ptr` objects is very efficient - it only updates reference counts
- VSG can handle millions of objects without significant overhead
- The intrusive reference counting design scales well
- Container operations with ref_ptr are fast because no deep copying occurs
- Memory is automatically managed - objects are deleted when their reference count reaches zero