# vsgtimestamps

## Overview

The `vsgtimestamps` example demonstrates Vulkan timestamp queries in VSG for precise GPU timing measurement. This technique provides accurate timing information about rendering operations directly from the GPU, enabling detailed performance analysis, profiling, and optimization of rendering pipelines. Unlike CPU-based timing, GPU timestamps measure actual GPU execution time.

## Key VSG Features Used

- **Timestamp queries** - GPU-based timing measurement
- **Query pools** - Managing timestamp query resources
- **WriteTimestamp commands** - Recording timestamps at specific pipeline stages
- **Physical device properties** - Timestamp capability detection
- **Pipeline stage synchronization** - Timing specific rendering phases
- **Results retrieval** - Reading timing data from GPU

## What the Example Demonstrates

1. **GPU Performance Profiling**
   - Measuring actual GPU execution time for rendering operations
   - High-precision timing independent of CPU performance
   - Pipeline stage-specific timing measurements
   - Real-time performance monitoring and analysis

2. **Timestamp Query Implementation**
   - Creating and managing timestamp query pools
   - Writing timestamps at different pipeline stages
   - Converting raw timestamp values to meaningful time units
   - Handling device-specific timestamp properties

3. **Device Capability Detection**
   - Checking timestamp support availability
   - Querying timestamp precision and valid bits
   - Understanding device-specific timing limitations
   - Proper fallback handling for unsupported devices

4. **Performance Analysis Applications**
   - Frame time measurement for performance optimization
   - Identifying rendering bottlenecks within the GPU pipeline
   - Comparing performance across different rendering techniques
   - Automated performance regression detection

## Code Highlights

```cpp
// Check device timestamp support and capabilities
auto physicalDevice = window->getOrCreatePhysicalDevice();
std::cout << "physicalDevice = " << physicalDevice << std::endl;

for (auto& queueFamilyProperties : physicalDevice->getQueueFamilyProperties())
{
    std::cout << "    queueFamilyProperties.timestampValidBits = " << queueFamilyProperties.timestampValidBits << std::endl;
}

const auto& limits = physicalDevice->getProperties().limits;
std::cout << "    limits.timestampComputeAndGraphics = " << limits.timestampComputeAndGraphics << std::endl;
std::cout << "    limits.timestampPeriod = " << limits.timestampPeriod << " nanoseconds." << std::endl;

// Ensure timestamp support is available
if (!limits.timestampComputeAndGraphics)
{
    std::cout << "Timestamps not supported by device." << std::endl;
    return 1;
}

// Calculate scaling factor for converting to milliseconds
double timestampScaleToMilliseconds = 1e-6 * static_cast<double>(limits.timestampPeriod);
```

```cpp
// Create timestamp query pool
auto query_pool = vsg::QueryPool::create();
query_pool->queryType = VK_QUERY_TYPE_TIMESTAMP;
query_pool->queryCount = 2;  // Two timestamps: start and end
```

```cpp
// Embed timestamps in command graph for precise GPU timing
auto commandGraph = vsg::CommandGraph::create(window);

// Reset query pool before use
auto reset_query = vsg::ResetQueryPool::create(query_pool);
commandGraph->addChild(reset_query);

// Record timestamp at beginning of pipeline (top of pipe)
auto write1 = vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT, query_pool, 0);
commandGraph->addChild(write1);

// Render the scene (timed operation)
commandGraph->addChild(vsg::createRenderGraphForView(window, camera, vsg_scene));

// Record timestamp at end of pipeline (bottom of pipe)
auto write2 = vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT, query_pool, 1);
commandGraph->addChild(write2);
```

```cpp
// Retrieve and process timestamp results each frame
while (viewer->advanceToNextFrame() && (numFrames < 0 || (numFrames--) > 0))
{
    viewer->handleEvents();
    viewer->update();
    viewer->recordAndSubmit();
    viewer->present();
    
    // Read timestamp query results
    std::vector<uint64_t> timestamps(2);
    if (query_pool->getResults(timestamps) == VK_SUCCESS)
    {
        // Calculate elapsed time in milliseconds
        auto delta = timestampScaleToMilliseconds * static_cast<double>(timestamps[1] - timestamps[0]);
        std::cout << "delta = " << delta << "ms" << std::endl;
    }
}
```

```cpp
// Pipeline stage options for different timing measurements
VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT           // Very beginning of command execution
VK_PIPELINE_STAGE_VERTEX_INPUT_BIT          // After vertex input assembly
VK_PIPELINE_STAGE_VERTEX_SHADER_BIT         // After vertex shader execution
VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT       // After fragment shader execution
VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT // After color attachment writes
VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT        // Very end of command execution
```

```cpp
// Advanced timing analysis example
class GPUProfiler
{
private:
    vsg::ref_ptr<vsg::QueryPool> timestampPool;
    double timestampScale;
    std::vector<std::string> stageNames;
    
public:
    void setupProfiling(vsg::ref_ptr<vsg::CommandGraph> commandGraph)
    {
        // Create query pool for multiple timing points
        timestampPool = vsg::QueryPool::create();
        timestampPool->queryType = VK_QUERY_TYPE_TIMESTAMP;
        timestampPool->queryCount = 6;  // Multiple timing points
        
        stageNames = {"Start", "PostVertex", "PostGeometry", "PostFragment", "PostColorOutput", "End"};
        
        // Reset queries
        commandGraph->addChild(vsg::ResetQueryPool::create(timestampPool));
        
        // Add timestamps at key pipeline stages
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT, timestampPool, 0));
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_VERTEX_SHADER_BIT, timestampPool, 1));
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_GEOMETRY_SHADER_BIT, timestampPool, 2));
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, timestampPool, 3));
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT, timestampPool, 4));
        commandGraph->addChild(vsg::WriteTimestamp::create(VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT, timestampPool, 5));
    }
    
    void analyzeResults()
    {
        std::vector<uint64_t> timestamps(6);
        if (timestampPool->getResults(timestamps) == VK_SUCCESS)
        {
            std::cout << "GPU Pipeline Timing Breakdown:" << std::endl;
            for (size_t i = 1; i < timestamps.size(); ++i)
            {
                double stageTime = timestampScale * static_cast<double>(timestamps[i] - timestamps[i-1]);
                std::cout << "  " << stageNames[i-1] << " -> " << stageNames[i] << ": " << stageTime << "ms" << std::endl;
            }
            
            double totalTime = timestampScale * static_cast<double>(timestamps.back() - timestamps.front());
            std::cout << "  Total GPU Time: " << totalTime << "ms" << std::endl;
        }
    }
};
```

## Running the Example

```bash
# Basic usage with 3D model (required)
./vsgtimestamps model.vsgt

# Multiple models for increased GPU load
./vsgtimestamps model1.vsgt model2.gltf model3.obj

# Debug mode for detailed Vulkan validation
./vsgtimestamps model.vsgt --debug

# Different present modes for timing analysis
./vsgtimestamps model.vsgt --IMMEDIATE    # No VSync for pure GPU timing
./vsgtimestamps model.vsgt --FIFO         # VSync timing analysis
./vsgtimestamps model.vsgt --MAILBOX      # Triple buffering timing

# Window configuration for performance testing
./vsgtimestamps model.vsgt --window 1920 1080
./vsgtimestamps model.vsgt --fullscreen
./vsgtimestamps model.vsgt --small-test   # Small window for quick testing

# Camera animation for consistent timing scenarios
./vsgtimestamps model.vsgt -p camera_animation.vsgb

# Limited frame count for profiling sessions
./vsgtimestamps model.vsgt -f 1000

# Different depth buffer formats for performance comparison
./vsgtimestamps model.vsgt --d32
```

### Expected Output

The console displays detailed device information followed by continuous timing measurements:

```
physicalDevice = vsg::PhysicalDevice
    queueFamilyProperties.timestampValidBits = 64
    limits.timestampComputeAndGraphics = 1
    limits.timestampPeriod = 1.000000 nanoseconds.
delta = 2.345ms
delta = 2.412ms
delta = 1.987ms
delta = 2.156ms
```

The timing values represent actual GPU execution time for rendering the scene, measured in milliseconds.

## Timestamp Analysis Applications

### 1. Performance Bottleneck Identification

```cpp
struct PerformanceMetrics
{
    double vertexTime;
    double fragmentTime;
    double totalRenderTime;
    
    void identifyBottlenecks()
    {
        if (fragmentTime > vertexTime * 2.0)
        {
            std::cout << "Fragment shader bottleneck detected" << std::endl;
        }
        else if (vertexTime > fragmentTime * 2.0)
        {
            std::cout << "Vertex processing bottleneck detected" << std::endl;
        }
    }
};
```

### 2. Frame Time Budgeting

```cpp
class FrameBudget
{
private:
    double targetFrameTime = 16.67; // 60 FPS target
    double renderBudget = 12.0;     // 12ms render budget
    
public:
    bool withinBudget(double actualTime)
    {
        return actualTime <= renderBudget;
    }
    
    void adjustQuality(double actualTime)
    {
        if (actualTime > renderBudget * 1.2)
        {
            // Reduce render quality
            decreaseLOD();
            reduceShadowQuality();
        }
        else if (actualTime < renderBudget * 0.8)
        {
            // Increase render quality
            increaseLOD();
            improveShadowQuality();
        }
    }
};
```

### 3. Performance Regression Detection

```cpp
class PerformanceRegression
{
private:
    std::vector<double> recentTimes;
    double baselineTime;
    
public:
    void checkRegression(double currentTime)
    {
        recentTimes.push_back(currentTime);
        if (recentTimes.size() > 100) recentTimes.erase(recentTimes.begin());
        
        double averageTime = std::accumulate(recentTimes.begin(), recentTimes.end(), 0.0) / recentTimes.size();
        
        if (averageTime > baselineTime * 1.15) // 15% slower
        {
            std::cout << "Performance regression detected: " << averageTime << "ms vs baseline " << baselineTime << "ms" << std::endl;
        }
    }
};
```

## Device Timestamp Properties

### Timestamp Valid Bits

```cpp
uint32_t validBits = queueFamilyProperties.timestampValidBits;
// Common values: 32, 36, 64
// Higher values provide more precision and longer measurement range
```

### Timestamp Period

```cpp
float nanosecPerTick = limits.timestampPeriod;
// Device-specific time resolution
// Common values: 1.0ns (high precision), 52.08ns (some integrated GPUs)
```

### Timestamp Compute and Graphics Support

```cpp
bool supported = limits.timestampComputeAndGraphics;
// Must be VK_TRUE for timestamp queries to work on graphics queues
```

## Performance Considerations

### Query Overhead

- Timestamp queries have minimal GPU overhead
- CPU query result retrieval may introduce synchronization points
- Use multiple query pools to avoid pipeline stalls

### Synchronization

- Results may not be immediately available
- Consider double-buffering query pools for continuous profiling
- Use VK_QUERY_RESULT_WAIT_BIT carefully to avoid blocking

### Precision vs Range

- Higher timestampValidBits provide better precision
- Lower timestampPeriod values offer finer time resolution
- Consider device capabilities when designing profiling systems

## Key Takeaways

- GPU timestamps provide accurate timing measurement independent of CPU performance
- Device capability checking is essential before using timestamp queries
- Pipeline stage selection determines what operations are being timed
- Continuous monitoring enables real-time performance optimization
- Results must be scaled by device-specific timestamp period for meaningful values
- Timestamp queries are ideal for identifying GPU rendering bottlenecks
- VSG simplifies Vulkan timestamp query management while providing full control
- This technique is fundamental for performance-critical applications and automated testing
- Combining with other profiling techniques provides comprehensive performance analysis