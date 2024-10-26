# Learn Vulkan

[TOC]

![](picture\p1.png)

1. 挑选物理设备(也就是GPU)
   - 查询物理设备扩展与特性
   - 查询物理设备队列族
2. 创建逻辑设备
   - 设置逻辑设备特性
   - 获取队列句柄
3. 窗口表面与交换链
   - GLFW创建窗口表面
4. 管线设置与提交
   - 着色器设置
   - 管线状态设置
5. 描述布局
6. 图像载入

# 第一步：挑选物理设备

- 首先需要创建一个vulkan instance。挑选物理设备、创建窗口表面和交换链都需要这个instance。

- 电脑上可能有众多显卡(我的笔记本就有一张集成显卡和3060)，于是我们需要选择一个适合的显卡。

- 我们需要根据我们的需求去挑选显卡，比如我们想要meshshader的**扩展**，或者是geometryShader的**特性**。就可以提前设置好需要的东西去挑选显卡。**需要注意的是：特性和扩展是两个东西。**

  - VkPhysicalDeviceFeatures/VkPhysicalDeviceMeshShaderFeaturesNV
  - 物理设备特性/物理设备的英伟达扩展

- 

- 利用vkInstance查询物理设备，这里会介绍五个函数，分别时挑选物理设备、判断物理设备是否合适、查询物理设备的队列、查询物理设备支持的扩展、交换链细节查询。

- 具体代码如下

  - **挑选物理设备**

  - ```C++
     //挑选物理设备
    void PCBDevice::pickPhysicalDevice() {
    
            //两次调用vkEnumeratePhysicalDevices函数获取设备信息。
            //这个应该是c语言风格的代码，
            //第一次获取数量，第二次根据这个数量获取对应数量的信息
            uint32_t deviceCount = 0;
            vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
            if (deviceCount == 0) {
                throw std::runtime_error("failed to find GPUs with Vulkan support!");
            }
            std::cout << "Device count: " << deviceCount << std::endl;
            std::vector<VkPhysicalDevice> devices(deviceCount);
            vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
    
            //遍历设备中的显卡，挑选一块合适的显卡
            for (const auto& device : devices) {
                if (isDeviceSuitable(device)) {
                    physicalDevice = device;
                    break;
                }
            }
    
            //如果没有选好显卡就输出错误
            if (physicalDevice == VK_NULL_HANDLE) {
                throw std::runtime_error("failed to find a suitable GPU!");
            }
    
            //这一句时我为了输出显卡名字额外加上去的
            vkGetPhysicalDeviceProperties(physicalDevice, &properties);
            std::cout << "physical device: " << properties.deviceName << std::endl;
        }
    
    ```

  - **判断某个物理设备是否合适**

  - ```c++
    //选择适合的物理设备 
    bool PCBDevice::isDeviceSuitable(VkPhysicalDevice device) {
    
            //查询物理设备的队列
            QueueFamilyIndices indices = findQueueFamilies(device);
    
            //判断物理设备是否支持需要的扩展
            bool extensionsSupported = checkDeviceExtensionSupport(device);
    
            //查询物理设备是否支持交换链的某些操作
            bool swapChainAdequate = false;
            if (extensionsSupported) {
                SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
                swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
            }
            //获取物理设备的特性
            VkPhysicalDeviceFeatures supportedFeatures;
            vkGetPhysicalDeviceFeatures(device, &supportedFeatures);
            
            //我们需要的功能都支持的物理设备，就是我们需要的物理设备
            return indices.isComplete() && extensionsSupported && swapChainAdequate &&
                supportedFeatures.samplerAnisotropy;
        }
    ```

  - **查找队列**

  - ```c++
    //队列索引
    //我们调好的队列的索引就放在这里面
    struct QueueFamilyIndices {
    
            //我们只需要两个队列就行了，一个支持图形渲染，一个支持图形呈现
          uint32_t graphicsFamily;
          uint32_t presentFamily;
          bool graphicsFamilyHasValue = false;
          bool presentFamilyHasValue = false;
          bool isComplete() { return graphicsFamilyHasValue &&presentFamilyHasValue; }
    };   
    //寻找物理设备的队列
    QueueFamilyIndices PCBDevice::findQueueFamilies(VkPhysicalDevice device) {
    
            QueueFamilyIndices indices;
    
            //还是和查询物理设备的数量和信息一样的方式去查询物理设备的队列数量和信息
            uint32_t queueFamilyCount = 0;
            vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);
    
            std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
            vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
    
            int i = 0;
            //遍历物理设备的每一个队列
            for (const auto& queueFamily : queueFamilies) {
                //判断这个队列是否大于0，可能是因为队列族里面有空队列？
                //判断这个队列是否支持图形队列
                if (queueFamily.queueCount > 0 && queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
                    indices.graphicsFamily = i;
                    indices.graphicsFamilyHasValue = true;
                }
                VkBool32 presentSupport = false;
                //判断这个队列是否支持呈现队列
                vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface_, &presentSupport);
                if (queueFamily.queueCount > 0 && presentSupport) {
                    indices.presentFamily = i;
                    indices.presentFamilyHasValue = true;
                }
                //如果已经有了我们想要的图形队列和显示队列，那么我们就退出循环，就挑选好了队列
                if (indices.isComplete()) {
                    break;
                }
                //要是没有选好，那就往下遍历
                i++;
            }
            //返回队列信息
            return indices;
        }
    ```

  - **物理设备是否支持我们需要的扩展**

  - ```c++
    //预设我们需要的扩展，交换链是肯定要的，后续的meshshader也是要的
    const std::vector<const char*> deviceExtensions = { VK_KHR_SWAPCHAIN_EXTENSION_NAME,// 确保启用了 swapchain
    VK_NV_MESH_SHADER_EXTENSION_NAME  // 启用 Mesh Shader 扩展 
    };
    
    //查找物理设备是否有我们需要的扩展
        bool PCBDevice::checkDeviceExtensionSupport(VkPhysicalDevice device) {
            uint32_t extensionCount;
            //依旧是用两次调用的方式去获取物理设备扩展的信息
            vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);
    
            std::vector<VkExtensionProperties> availableExtensions(extensionCount);
            vkEnumerateDeviceExtensionProperties(
                device,
                nullptr,
                &extensionCount,
                availableExtensions.data());
           
            std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());
    
            //很有意思的挑选方式，
            //如果物理设备里用我们需要的扩展就把对应扩展的名字擦除
            //如果看看最后有没有requiredExtensions空，要是空了说明都擦除完了，都有
            for (const auto& extension : availableExtensions) {
                requiredExtensions.erase(extension.extensionName);
            }
            return requiredExtensions.empty();
        }
    ```

  - ```c++
    
    ```

    

# 第二步：创建逻辑设备

