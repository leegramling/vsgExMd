# vsgocclusionquery

## Overview

The `vsgocclusionquery` example demonstrates Vulkan occlusion queries in VSG, which enable GPU-based visibility testing to determine if rendered geometry is occluded by other objects. This technique is essential for performance optimization in complex 3D scenes, allowing applications to skip rendering or processing of objects that won't be visible to the viewer.

## Key VSG Features Used

- **Query pools** - Vulkan query object management
- **Occlusion queries** - GPU-based visibility testing
- **Query commands** - BeginQuery, EndQuery, ResetQueryPool
- **Command graph integration** - Embedding queries in rendering pipeline
- **Results retrieval** - Reading query results from GPU
- **Scene bounds computation** - Automatic camera positioning
- **Camera animation** - Path-based camera movement support

## What the Example Demonstrates

1. **GPU Occlusion Testing**
   - Determining pixel visibility during rendering
   - Counting rendered fragments that pass depth testing
   - Hardware-accelerated visibility determination
   - Real-time occlusion feedback for optimization

2. **Query Pool Management**
   - Creating and configuring Vulkan query pools
   - Resetting queries before use
   - Proper query lifecycle management
   - Efficient query resource allocation

3. **Integration with Rendering Pipeline**
   - Embedding queries within command graphs
   - Coordinating queries with render passes
   - Proper synchronization between query operations
   - Command ordering for correct results

4. **Performance Optimization Applications**
   - Visibility-based culling decisions
   - Level-of-detail selection based on visibility
   - Conditional rendering based on occlusion
   - Real-time performance monitoring

## Code Highlights

```cpp
// Query pool creation for occlusion testing
auto query_pool = vsg::QueryPool::create();
query_pool->queryType = VK_QUERY_TYPE_OCCLUSION;
query_pool->queryCount = 1;  // Single query for this example
```

```cpp
// Command graph setup with embedded queries
auto commandGraph = vsg::CommandGraph::create(window);

// Reset query pool before use (required for consistent results)
auto reset_query = vsg::ResetQueryPool::create(query_pool);
commandGraph->addChild(reset_query);

// Begin occlusion query before rendering
commandGraph->addChild(vsg::BeginQuery::create(query_pool, 0, 0));

// Render the scene - fragments that pass depth test will be counted
commandGraph->addChild(vsg::createRenderGraphForView(window, camera, vsg_scene));

// End query after rendering
commandGraph->addChild(vsg::EndQuery::create(query_pool, 0));
```

```cpp
// Retrieve and display query results each frame
while (viewer->advanceToNextFrame() && (numFrames < 0 || (numFrames--) > 0))
{
    viewer->handleEvents();
    viewer->update();
    viewer->recordAndSubmit();
    viewer->present();
    
    // Read occlusion query results
    std::vector<uint64_t> results(1);
    if (query_pool->getResults(results) == VK_SUCCESS)
    {
        std::cout << "results[0] = " << results[0] << std::endl;  // Number of visible fragments
    }
}
```

```cpp
// Automatic scene bounds computation for camera positioning
vsg::ComputeBounds computeBounds;
vsg_scene->accept(computeBounds);
vsg::dvec3 centre = (computeBounds.bounds.min + computeBounds.bounds.max) * 0.5;
double radius = vsg::length(computeBounds.bounds.max - computeBounds.bounds.min) * 0.6;

// Position camera based on computed bounds
auto lookAt = vsg::LookAt::create(centre + vsg::dvec3(0.0, -radius * 3.5, 0.0), centre, vsg::dvec3(0.0, 0.0, 1.0));
```

```cpp
// Flexible projection matrix selection
vsg::ref_ptr<vsg::ProjectionMatrix> perspective;
auto ellipsoidModel = vsg_scene->getRefObject<vsg::EllipsoidModel>("EllipsoidModel");

if (ellipsoidModel)
{
    // Use ellipsoid perspective for geospatial data
    perspective = vsg::EllipsoidPerspective::create(lookAt, ellipsoidModel, 30.0, aspectRatio, nearFarRatio, 0.0);
}
else
{
    // Standard perspective for regular 3D models
    perspective = vsg::Perspective::create(30.0, aspectRatio, nearFarRatio * radius, radius * 4.5);
}
```

```cpp
// Query result interpretation
uint64_t fragmentCount = results[0];

if (fragmentCount == 0)
{
    // Object is completely occluded - consider skipping detailed processing
    std::cout << "Object fully occluded" << std::endl;
}
else if (fragmentCount < threshold)
{
    // Object is partially visible - might use lower LOD
    std::cout << "Object partially visible: " << fragmentCount << " fragments" << std::endl;
}
else
{
    // Object is clearly visible - use full detail
    std::cout << "Object clearly visible: " << fragmentCount << " fragments" << std::endl;
}
```

## Running the Example

```bash
# Basic usage with 3D model (required)
./vsgocclusionquery model.vsgt

# Multiple models
./vsgocclusionquery model1.vsgt model2.gltf model3.obj

# Debug mode for Vulkan validation
./vsgocclusionquery model.vsgt --debug

# Different present modes for performance testing
./vsgocclusionquery model.vsgt --IMMEDIATE    # Immediate present mode
./vsgocclusionquery model.vsgt --FIFO         # FIFO present mode (VSync)
./vsgocclusionquery model.vsgt --MAILBOX      # Mailbox present mode

# Window configuration
./vsgocclusionquery model.vsgt --window 1024 768
./vsgocclusionquery model.vsgt --fullscreen
./vsgocclusionquery model.vsgt --no-frame

# Camera animation from file
./vsgocclusionquery model.vsgt -p camera_path.vsgb

# Limited frame count for testing
./vsgocclusionquery model.vsgt -f 100

# Small test window
./vsgocclusionquery model.vsgt --small-test

# Different depth buffer formats
./vsgocclusionquery model.vsgt --d32
```

### Expected Output

The console continuously displays occlusion query results:

```
results[0] = 245832
results[0] = 198456
results[0] = 0        # Object completely occluded
results[0] = 89234
results[0] = 312567
```

The numbers represent the count of fragments (pixels) that passed the depth test during rendering. Higher numbers indicate more visible geometry, while zero indicates complete occlusion.

## Occlusion Query Applications

### 1. Visibility-Based Culling

```cpp
class OcclusionCuller
{
public:
    bool isVisible(vsg::ref_ptr<vsg::Node> object, uint64_t threshold = 100)
    {
        // Render object with occlusion query
        // Return true if fragment count > threshold
        return fragmentCount > threshold;
    }
    
    void cullInvisibleObjects(vsg::ref_ptr<vsg::Group> scene)
    {
        for (auto& child : scene->children)
        {
            if (!isVisible(child))
            {
                // Skip detailed rendering for invisible objects
                child->setValue("culled", true);
            }
        }
    }
};
```

### 2. Level of Detail Selection

```cpp
enum class LODLevel { High, Medium, Low, Culled };

LODLevel selectLOD(uint64_t visibleFragments, double distance)
{
    if (visibleFragments == 0) return LODLevel::Culled;
    if (visibleFragments < 50 || distance > 1000.0) return LODLevel::Low;
    if (visibleFragments < 500 || distance > 100.0) return LODLevel::Medium;
    return LODLevel::High;
}
```

### 3. Conditional Compute Dispatch

```cpp
void conditionalProcessing(uint64_t visibilityResult)
{
    if (visibilityResult > 0)
    {
        // Object is visible - run expensive compute shaders
        dispatchParticleSimulation();
        updateAnimations();
        performPhysicsCalculations();
    }
    // Skip expensive operations for invisible objects
}
```

## Query Performance Considerations

### Synchronization Points

- Query results may not be immediately available
- Use multiple query pools to avoid stalling the pipeline
- Consider async result retrieval patterns

### Memory Bandwidth

- Occlusion queries consume GPU memory bandwidth
- Balance query frequency with performance needs
- Group multiple objects per query when appropriate

### False Positives/Negatives

- Conservative occlusion testing may miss small visible parts
- Temporal coherence: visibility often changes gradually
- Use hysteresis to avoid flickering between LOD levels

## Advanced Query Techniques

### Multiple Query Types

```cpp
// Occlusion query
auto occlusionPool = vsg::QueryPool::create();
occlusionPool->queryType = VK_QUERY_TYPE_OCCLUSION;

// Pipeline statistics query
auto pipelinePool = vsg::QueryPool::create();
pipelinePool->queryType = VK_QUERY_TYPE_PIPELINE_STATISTICS;

// Timestamp query
auto timestampPool = vsg::QueryPool::create();
timestampPool->queryType = VK_QUERY_TYPE_TIMESTAMP;
```

### Hierarchical Queries

```cpp
// Query parent bounding volumes first
if (queryBoundingBox(parentObject) > 0)
{
    // Only query children if parent is visible
    for (auto& child : parentObject->children)
    {
        queryOcclusion(child);
    }
}
```

## Key Takeaways

- Occlusion queries provide GPU-based visibility testing for performance optimization
- Results represent fragment counts that passed depth testing during rendering
- Query pools must be reset before use and properly synchronized
- Integration with command graphs enables seamless query execution
- Results enable intelligent culling, LOD selection, and conditional processing
- Performance benefits are most significant in complex scenes with high occlusion
- Consider temporal coherence and hysteresis for stable visibility-based decisions
- VSG simplifies Vulkan query management while providing full functionality access
- This technique is essential for scalable rendering in large, complex 3D environments