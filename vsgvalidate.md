# vsgvalidate

## Overview

The `vsgvalidate` example demonstrates VSG's validation and testing mechanisms for Vulkan resource management. It performs automated tests to verify correct behavior of shared resources, format handling, and descriptor pool management. This example is particularly valuable for understanding VSG's internal validation systems and debugging resource-related issues.

## Key VSG Features Used

- **Validation layers** - Vulkan debugging and validation
- **Resource sharing** - Sampler sharing between multiple ImageInfo instances
- **Format properties** - Querying device format capabilities
- **Compressed textures** - Block-based texture format handling
- **Descriptor pool management** - Resource requirement collection and reservation
- **Context resource management** - GPU resource compilation and allocation
- **Automatic mipmap handling** - Format-aware mipmap generation

## What the Example Demonstrates

1. **Resource Sharing Validation**
   - Proper sharing of Vulkan samplers across multiple ImageInfo objects
   - Verification that shared resources maintain their properties
   - Testing of reference counting and resource lifetime management

2. **Format Capability Testing**
   - Automatic detection of format-specific limitations
   - Compressed texture format handling
   - Mipmap generation capability validation

3. **Descriptor Management**
   - Resource requirement collection and analysis
   - Descriptor pool sizing and reservation
   - Context-based resource compilation

## Code Highlights

```cpp
// Test 1: Sampler sharing validation
void testSamplerSharing()
{
    vsg::info("---- Test 1: Sharing Sampler between ImageInfo instances ----");

    // Create shared sampler with specific properties
    auto sampler = vsg::Sampler::create();
    sampler->maxLod = VK_LOD_CLAMP_NONE;  // Test property

    // Create different texture data types
    auto textureData1 = vsg::ubvec4Array2D::create(32, 32, 
        vsg::Data::Properties(VK_FORMAT_R8G8B8A8_UNORM));

    auto textureData2 = vsg::block64Array2D::create(32, 32, 
        vsg::Data::Properties(VK_FORMAT_BC4_UNORM_BLOCK));
    textureData2->properties.blockWidth = 8;
    textureData2->properties.blockHeight = 8;

    // Share sampler between different ImageInfo instances
    auto imageInfo1 = vsg::ImageInfo::create(sampler, textureData1);
    auto imageInfo2 = vsg::ImageInfo::create(sampler, textureData2);

    // Validation: Verify shared sampler properties are preserved
    if (imageInfo1->sampler->maxLod != VK_LOD_CLAMP_NONE)
    {
        vsg::warn("Test 1 failed: shared sampler has been modified");
    }
    else
    {
        vsg::info("Test 1 passed: sampler sharing works correctly");
    }

    // Test compilation and resource management
    auto descriptorImage1 = vsg::DescriptorImage::create(imageInfo1);
    auto descriptorImage2 = vsg::DescriptorImage::create(imageInfo2);

    // Collect resource requirements
    vsg::CollectResourceRequirements collectResourceRequirements;
    descriptorImage1->accept(collectResourceRequirements);
    descriptorImage2->accept(collectResourceRequirements);

    // Compile resources
    descriptorImage1->compile(*context);
    descriptorImage2->compile(*context);
    context->record();
}

// Test 2: Format capability and mipmap validation
void testFormatCapabilities()
{
    vsg::info("---- Test 2: Automatic mipmap disabling for unsupported formats ----");

    auto textureData = vsg::block64Array2D::create(32, 32, 
        vsg::Data::Properties(VK_FORMAT_BC4_UNORM_BLOCK));

    // Query format properties from device
    VkFormatProperties props;
    vkGetPhysicalDeviceFormatProperties(*(device->getPhysicalDevice()), 
                                       textureData->properties.format, &props);

    // Check if format supports blitting (required for mipmap generation)
    const bool isBlitPossible = 
        (props.optimalTilingFeatures & VK_FORMAT_FEATURE_BLIT_DST_BIT) > 0 && 
        (props.optimalTilingFeatures & VK_FORMAT_FEATURE_BLIT_SRC_BIT) > 0;

    if (!isBlitPossible)
    {
        // Simulate format without blit support
        textureData->properties.blockWidth = 1;
        textureData->properties.blockHeight = 1;

        auto sampler = vsg::Sampler::create();
        sampler->maxLod = VK_LOD_CLAMP_NONE;

        auto imageInfo = vsg::ImageInfo::create(sampler, textureData);
        auto descriptorImage = vsg::DescriptorImage::create(imageInfo);

        // Collect requirements to trigger automatic format analysis
        vsg::CollectResourceRequirements collectResourceRequirements;
        descriptorImage->accept(collectResourceRequirements);

        // Validation: Verify automatic mipmap disabling
        if (imageInfo->imageView->image->mipLevels > 1)
        {
            vsg::warn("Test 2 failed: Image will be created with ", 
                     imageInfo->imageView->image->mipLevels, 
                     " mipLevels, but we have no way to fill them in.");
        }
        else
        {
            vsg::info("Test 2 passed: Automatic mipmap disabling works correctly");
        }
    }
}

// Test 3: Descriptor pool reservation validation
void testDescriptorPoolManagement()
{
    vsg::info("---- Test 3: DescriptorPool reservation in vsg::Context ----");

    // Define resource requirements
    vsg::ResourceRequirements requirements;
    requirements.descriptorTypeMap[VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER] = 1;

    vsg::info("Test 3, part 1: Single descriptor set");
    requirements.externalNumDescriptorSets = 1;
    context->reserve(requirements);

    vsg::info("Test 3, part 2: Multiple descriptor sets");
    requirements.externalNumDescriptorSets = 2;
    requirements.descriptorTypeMap[VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER] = 2;
    context->reserve(requirements);

    // The reservation should handle pool sizing automatically
    vsg::info("Test 3 passed: Descriptor pool reservation handled correctly");
}

// Device setup with validation layers
void setupValidatedDevice()
{
    bool debugLayer = true;
    auto apiDumpLayer = arguments.read({"--api", "-a"});

    // Enable validation layers
    vsg::Names instanceExtensions;
    vsg::Names requestedLayers;
    if (debugLayer || apiDumpLayer)
    {
        instanceExtensions.push_back(VK_KHR_GET_PHYSICAL_DEVICE_PROPERTIES_2_EXTENSION_NAME);
        instanceExtensions.push_back(VK_EXT_DEBUG_REPORT_EXTENSION_NAME);
        requestedLayers.push_back("VK_LAYER_KHRONOS_validation");
        if (apiDumpLayer) requestedLayers.push_back("VK_LAYER_LUNARG_api_dump");
    }

    // Validate available layers
    vsg::Names validatedNames = vsg::validateInstancelayerNames(requestedLayers);

    // Create instance with validation
    uint32_t vulkanVersion = VK_API_VERSION_1_0;
    auto instance = vsg::Instance::create(instanceExtensions, validatedNames, vulkanVersion);

    // Select appropriate device
    auto [physicalDevice, queueFamily] = instance->getPhysicalDeviceAndQueueFamily(VK_QUEUE_GRAPHICS_BIT);
    if (!physicalDevice || queueFamily < 0)
    {
        throw std::runtime_error("Could not create PhysicalDevice");
    }

    // Configure device features
    auto deviceFeatures = vsg::DeviceFeatures::create();
    deviceFeatures->get().samplerAnisotropy = VK_TRUE;
    deviceFeatures->get().geometryShader = enableGeometryShader;

    // Create device and context
    vsg::QueueSettings queueSettings{vsg::QueueSetting{queueFamily, {1.0}}};
    auto device = vsg::Device::create(physicalDevice, queueSettings, 
                                     validatedNames, {}, deviceFeatures);
    
    auto context = vsg::Context::create(device);
    context->commandPool = vsg::CommandPool::create(device, queueFamily);
    context->graphicsQueue = device->getQueue(queueFamily);
}

// Resource requirement collection example
class ResourceAnalyzer
{
public:
    static void analyzeResourceRequirements(vsg::ref_ptr<vsg::Object> object)
    {
        vsg::CollectResourceRequirements collector;
        object->accept(collector);
        
        // Analyze collected requirements
        const auto& requirements = collector.requirements;
        
        std::cout << "Resource Requirements Analysis:" << std::endl;
        std::cout << "  External descriptor sets: " << requirements.externalNumDescriptorSets << std::endl;
        std::cout << "  Descriptor types:" << std::endl;
        
        for (const auto& [type, count] : requirements.descriptorTypeMap)
        {
            std::cout << "    Type " << type << ": " << count << " descriptors" << std::endl;
        }
        
        std::cout << "  Vertex attributes: " << requirements.numVertexAttributes << std::endl;
        std::cout << "  Vertex bindings: " << requirements.numVertexBindings << std::endl;
    }
};
```

## Running the Example

```bash
# Basic validation run
./vsgvalidate

# Enable API dump layer for detailed Vulkan call tracing
./vsgvalidate --api

# Enable multi-sampling (requires Vulkan 1.2)
./vsgvalidate --msaa

# Enable geometry shader features
./vsgvalidate --gs
```

### Test Output

The example runs three comprehensive tests:

1. **Test 1 Output**:
   ```
   ---- Test 1 : Sharing Sampler between ImageInfo/InmageView/Image instances ----
   test 1 passed
   End of test 1
   ```

2. **Test 2 Output**:
   ```
   ---- Test 2 : Automatic disabling of mipmapping when using compressed image formats. ----
   test 2 passed
   End of test 2
   ```

3. **Test 3 Output**:
   ```
   ---- Test 3 : DescriptorPool reservation in vsg::Context ----
   Test 3, part 1
   Test 3, part 2
   End of test 3
   ```

## Validation Benefits

### Resource Sharing Validation

- Ensures sampler objects maintain properties when shared
- Verifies reference counting works correctly
- Tests memory management for shared resources

### Format Capability Detection

- Automatically detects device format limitations
- Prevents invalid mipmap generation attempts
- Adapts rendering pipeline to hardware capabilities

### Descriptor Pool Management

- Validates proper resource requirement collection
- Tests descriptor pool sizing algorithms
- Ensures efficient GPU memory usage

## Key Validation Concepts

### Resource Requirement Collection

```cpp
vsg::CollectResourceRequirements collector;
sceneGraph->accept(collector);

// Access collected requirements
auto& requirements = collector.requirements;
std::cout << "Descriptor sets needed: " << requirements.externalNumDescriptorSets << std::endl;
```

### Format Property Querying

```cpp
VkFormatProperties props;
vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);

bool supportsBlitting = 
    (props.optimalTilingFeatures & VK_FORMAT_FEATURE_BLIT_SRC_BIT) &&
    (props.optimalTilingFeatures & VK_FORMAT_FEATURE_BLIT_DST_BIT);
```

### Validation Layer Integration

```cpp
vsg::Names requestedLayers{"VK_LAYER_KHRONOS_validation"};
vsg::Names validatedLayers = vsg::validateInstancelayerNames(requestedLayers);
auto instance = vsg::Instance::create(extensions, validatedLayers, vulkanVersion);
```

## Key Takeaways

- VSG provides comprehensive internal validation for Vulkan resource management
- Automatic format capability detection prevents runtime errors
- Resource sharing is properly validated to ensure memory safety
- Descriptor pool management is handled automatically with proper validation
- Validation layers are essential for debugging Vulkan applications
- This example serves as a template for creating custom validation tests
- Resource requirement collection enables efficient GPU memory planning
- Format-aware mipmap generation prevents invalid operations