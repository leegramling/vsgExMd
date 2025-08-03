# vsgdeviceselection

## Overview

The `vsgdeviceselection` example demonstrates how to enumerate, inspect, and manually select Vulkan physical devices and logical devices in VSG. This is useful for applications that need specific GPU capabilities or want to provide users with device selection options.

## Key VSG Features Used

- **Physical device enumeration** - List all available GPUs
- **Device capability inspection** - Query device properties and queue families
- **Manual device selection** - Override VSG's automatic device selection
- **Queue family analysis** - Inspect graphics, compute, and present capabilities
- **Extension enumeration** - List supported device extensions
- **Custom device creation** - Manual device and queue setup

## What the Example Demonstrates

1. **Device Discovery**
   - Enumerate all available physical devices
   - Query device properties (name, type, API version)
   - Analyze queue family capabilities

2. **Manual Device Selection**
   - Select specific physical device by index
   - Override VSG's automatic device selection
   - Create custom logical devices

3. **Capability Inspection**
   - List supported instance and device extensions
   - Display queue family properties and capabilities
   - Show device matching criteria for surface compatibility

## Code Highlights

```cpp
// Custom Vulkan API version parsing
uint32_t makeVulkanApiVersion(const std::string& versionStr)
{
    char c;
    uint32_t vk_major = 1, vk_minor = 0;
    std::stringstream vk_version_str(versionStr);
    vk_version_str >> vk_major >> c >> vk_minor;
    
#if defined(VK_MAKE_API_VERSION)
    return VK_MAKE_API_VERSION(0, vk_major, vk_minor, 0);
#elif defined(VK_MAKE_VERSION)
    return VK_MAKE_VERSION(vk_major, vk_minor, 0);
#else
    return VK_API_VERSION_1_0;
#endif
}

// Enumerate all available physical devices
auto instance = window->getOrCreateInstance();
auto surface = window->getOrCreateSurface();
auto physicalDevices = instance->getPhysicalDevices();

for (auto& physicalDevice : physicalDevices)
{
    auto properties = physicalDevice->getProperties();
    
    // Check if device supports required queue families
    auto [graphicsFamily, presentFamily] = physicalDevice->getQueueFamily(
        windowTraits->queueFlags, surface);
    
    bool deviceSuitable = (graphicsFamily >= 0 && presentFamily >= 0);
    
    std::cout << (deviceSuitable ? "matched " : "not matched ")
              << physicalDevice << " " << properties.deviceName
              << ", deviceType = " << properties.deviceType
              << ", apiVersion = " << VK_API_VERSION_MAJOR(properties.apiVersion)
              << "." << VK_API_VERSION_MINOR(properties.apiVersion)
              << std::endl;
}

// Manual device selection by index
if (size_t pd_num = 0; arguments.read("--select", pd_num))
{
    auto physicalDevices = instance->getPhysicalDevices();
    if (pd_num < physicalDevices.size())
    {
        auto physicalDevice = physicalDevices[pd_num];
        auto properties = physicalDevice->getProperties();
        
        std::cout << "Selected Physical Device: " << properties.deviceName << std::endl;
        window->setPhysicalDevice(physicalDevice);
    }
}

// Analyze queue family properties
auto& queueFamilyProperties = physicalDevice->getQueueFamilyProperties();
for (size_t qfp = 0; qfp < queueFamilyProperties.size(); ++qfp)
{
    auto& prop = queueFamilyProperties[qfp];
    
    std::list<std::string> flags;
    if ((prop.queueFlags & VK_QUEUE_GRAPHICS_BIT)) flags.push_back("GRAPHICS");
    if ((prop.queueFlags & VK_QUEUE_COMPUTE_BIT)) flags.push_back("COMPUTE");
    if ((prop.queueFlags & VK_QUEUE_TRANSFER_BIT)) flags.push_back("TRANSFER");
    if ((prop.queueFlags & VK_QUEUE_SPARSE_BINDING_BIT)) flags.push_back("SPARSE_BINDING");
    if ((prop.queueFlags & VK_QUEUE_PROTECTED_BIT)) flags.push_back("PROTECTED");
    
    std::cout << "QueueFamily[" << qfp << "] flags = " << combined_flags
              << ", queueCount = " << prop.queueCount
              << ", timestampValidBits = " << prop.timestampValidBits << std::endl;
}

// Custom device creation with specific extensions
vsg::Names deviceExtensions;
deviceExtensions.push_back(VK_KHR_SWAPCHAIN_EXTENSION_NAME);
deviceExtensions.insert(deviceExtensions.end(), 
                       windowTraits->deviceExtensionNames.begin(), 
                       windowTraits->deviceExtensionNames.end());

auto [physicalDevice, queueFamily, presentFamily] = 
    instance->getPhysicalDeviceAndQueueFamily(windowTraits->queueFlags, surface);

vsg::QueueSettings queueSettings{
    vsg::QueueSetting{queueFamily, {1.0}}, 
    vsg::QueueSetting{presentFamily, {1.0}}
};

auto device = vsg::Device::create(
    physicalDevice, 
    queueSettings, 
    validatedNames, 
    deviceExtensions, 
    windowTraits->deviceFeatures, 
    instance->getAllocationCallbacks()
);

window->setDevice(device);

// Enumerate device extensions
auto physicalDevice = window->getOrCreatePhysicalDevice();
for (auto extension : physicalDevice->enumerateDeviceExtensionProperties())
{
    std::cout << "Extension: " << extension.extensionName 
              << ", spec = " << extension.specVersion << std::endl;
}
```

## Running the Example

```bash
# List all available physical devices and their capabilities
./vsgdeviceselection --list

# Display instance layers
./vsgdeviceselection --layers

# Show supported extensions
./vsgdeviceselection --extensions

# Select specific physical device by index
./vsgdeviceselection --select 0

# Use preferred device selection (discrete GPU preferred)
./vsgdeviceselection --PhysicalDevice

# Create custom logical device
./vsgdeviceselection --Device

# Specify Vulkan API version
./vsgdeviceselection --vulkan 1,3

# Load and render a model with selected device
./vsgdeviceselection model.vsgt --select 1
```

### Example Output

When running `--list`, you'll see output like:

```
physicalDevices.size() = 2

    matched vsg::PhysicalDevice GeForce RTX 3080, deviceType = 2
        apiVersion = 1.3.0, driverVersion = 528.49.0
        QueueFamilyProperties 3
            VkQueueFamilyProperties[0] queueFlags = GRAPHICS | COMPUTE | TRANSFER | SPARSE_BINDING
                queueCount = 16, timestampValidBits = 64
            VkQueueFamilyProperties[1] queueFlags = TRANSFER | SPARSE_BINDING
                queueCount = 2, timestampValidBits = 64
            VkQueueFamilyProperties[2] queueFlags = COMPUTE | TRANSFER | SPARSE_BINDING
                queueCount = 8, timestampValidBits = 64

    matched vsg::PhysicalDevice Intel(R) UHD Graphics 630, deviceType = 1
        apiVersion = 1.3.0, driverVersion = 101.4146.0
        QueueFamilyProperties 1
            VkQueueFamilyProperties[0] queueFlags = GRAPHICS | COMPUTE | TRANSFER
                queueCount = 1, timestampValidBits = 36
```

## Device Type Values

- **0**: VK_PHYSICAL_DEVICE_TYPE_OTHER
- **1**: VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU  
- **2**: VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU
- **3**: VK_PHYSICAL_DEVICE_TYPE_VIRTUAL_GPU
- **4**: VK_PHYSICAL_DEVICE_TYPE_CPU

## Key Takeaways

- VSG normally handles device selection automatically, preferring discrete GPUs
- Manual device selection allows targeting specific hardware capabilities
- Queue families determine what operations a device can perform
- Not all physical devices may support presentation to your window surface
- Device extensions enable additional Vulkan features
- Custom device creation provides fine-grained control over Vulkan setup
- This example is essential for applications requiring specific GPU features
- Device enumeration helps debug multi-GPU systems and compatibility issues