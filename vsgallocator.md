# vsgallocator

## Overview

The `vsgallocator` example demonstrates VSG's advanced memory allocation system, which provides efficient memory management for scene graph applications. It shows how to configure different allocator types, monitor memory usage, and optimize allocation patterns for different workloads.

## Key VSG Features Used

- **vsg::Allocator** - VSG's memory allocator system
- **vsg::IntrusiveAllocator** - High-performance allocator with custom alignment
- **AllocatorAffinity** - Different allocation pools for objects, nodes, and data
- **Memory reporting** - Detailed memory usage statistics
- **Block-based allocation** - Configurable memory block sizes
- **Scene loading and rendering** - Complete application with memory monitoring

## What the Example Demonstrates

1. **Allocator Configuration**
   - Setting custom alignment requirements
   - Choosing between standard and intrusive allocators
   - Configuring block sizes for different allocation types

2. **Memory Pool Affinity**
   - `ALLOCATOR_AFFINITY_OBJECTS` - General objects
   - `ALLOCATOR_AFFINITY_NODES` - Scene graph nodes
   - `ALLOCATOR_AFFINITY_DATA` - Large data arrays

3. **Memory Monitoring**
   - Real-time memory usage reporting
   - Empty block deletion
   - Total memory statistics

4. **Performance Testing**
   - Scene loading with memory tracking
   - Frame rate measurements
   - Memory cleanup timing

## Code Highlights

```cpp
// Configure allocator with custom alignment
if (size_t alignment; arguments.read("--alignment", alignment)) 
    vsg::Allocator::instance().reset(new vsg::IntrusiveAllocator(
        std::move(vsg::Allocator::instance()), alignment));

// Set block sizes for different allocation types
vsg::Allocator::instance()->setBlockSize(
    vsg::ALLOCATOR_AFFINITY_OBJECTS, objectsBlockSize);
vsg::Allocator::instance()->setBlockSize(
    vsg::ALLOCATOR_AFFINITY_NODES, nodesBlockSize);
vsg::Allocator::instance()->setBlockSize(
    vsg::ALLOCATOR_AFFINITY_DATA, dataBlockSize);

// Report memory usage
vsg::Allocator::instance()->report(std::cout);

// Clean up empty memory blocks
auto memoryDeleted = vsg::Allocator::instance()->deleteEmptyMemoryBlocks();
```

## Running the Example

```bash
# Basic usage with a 3D model
./vsgallocator model.vsgt

# Configure allocator settings
./vsgallocator model.vsgt --alignment 16
./vsgallocator model.vsgt --std  # Use standard allocator
./vsgallocator model.vsgt --allocator 1  # Set allocator type

# Set block sizes for different allocation types
./vsgallocator model.vsgt --objects 65536 --nodes 131072 --data 1048576

# Enable memory reporting
./vsgallocator model.vsgt --report

# Performance testing
./vsgallocator model.vsgt -f 1000  # Run for 1000 frames
./vsgallocator model.vsgt --stats  # Collect scene statistics
```

### Expected Output

```
Before end of Viewer scope.
IntrusiveAllocator::report() 
  totalAvailableSize = 1234567
  totalReservedSize = 2345678
  totalMemorySize = 3580245

After end of Viewer scope.
IntrusiveAllocator::report() 
  totalAvailableSize = 3456789
  totalReservedSize = 123456
  totalMemorySize = 3580245

After delete of empty memory blocks, where 2456789 was freed.
IntrusiveAllocator::report() 
  totalAvailableSize = 1000000
  totalReservedSize = 123456
  totalMemorySize = 1123456

load duration = 123.45ms
release duration  = 12.34ms
delete duration  = 1.23ms
Average frame rate = 60.5fps
```

## Key Takeaways

- VSG provides a sophisticated memory allocation system optimized for scene graphs
- Different allocation pools (objects, nodes, data) can be tuned independently
- The intrusive allocator provides better performance than standard allocation
- Memory blocks are reused efficiently, reducing allocation overhead
- Empty blocks can be explicitly freed when memory needs to be returned to the OS
- The allocator is thread-safe and supports multi-threaded applications
- Memory usage can be monitored and reported for optimization purposes