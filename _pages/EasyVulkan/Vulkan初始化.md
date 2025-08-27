---
title: "Vulkan初始化"
tags:
    - Vulkan
    - EasyVulkan
date: "2025-01-24"
thumbnail: "https://obsidian-picture-1313051055.cos.ap-nanjing.myqcloud.com/Obsidian/20250124230354.png"
bookmark: true
---

>最开始接触Vulkan时，通常会被其复杂的概念和庞大的API所吓到。无法理解window、instance、surface等概念的关系，不能区分物理设备和逻辑设备的区别。本文将介绍Vulkan的初始化过程，并解释各个概念之间的关系。最后，本文将介绍EasyVulkan项目的VulkanDevice和VulkanContext对这些概念的封装。

# 整体流程
1.  创建 Window
2.  创建 Instance
    1.  检查和启用必要的validation layers（如果在debug模式下）
    2.  设置必要的instance extensions，特别是GLFW要求的extensions
3.  创建 Window Surface
4.  获取物理设备
    1.  检查物理设备是否支持所需的features和extensions
    2.  检查物理设备是否适合（比如是否为独立显卡、是否支持所需的图形特性等）
5.  获取队列族索引
6.  创建逻辑设备
    1.  创建队列创建信息
    2.  启用必要的device extensions（比如VK_KHR_swapchain）
    3.  指定设备features
7.  创建命令池

# 1.创建Window
使用 GLFW 创建窗口，这是显示 Vulkan 渲染结果的基础：

```cpp
glfwInit();
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);  // 不创建 OpenGL 上下文
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);    // 暂时禁用窗口大小调整

window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```
由于历史原因，GLFW 最初是为 OpenGL 设计的窗口管理库，默认情况下，当你创建 GLFW 窗口时，它会自动创建一个 OpenGL 上下文。因此需要指定`GLFW_CLIENT_API`为`GLFW_NO_API`，只创建窗口，而不创建 OpenGL 上下文。

# 2.创建 Instance

Instance 是应用程序与 Vulkan 库之间的连接：

```cpp
VkApplicationInfo appInfo{};
appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
appInfo.pApplicationName = "Vulkan App";
appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.pEngineName = "No Engine";
appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
appInfo.apiVersion = VK_API_VERSION_1_0;

VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;

// 获取 GLFW 需要的 extension
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);
createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;

vkCreateInstance(&createInfo, nullptr, &instance);
```
Instance 代表了一个 Vulkan 应用程序的实例，它主要负责：

- 告诉 Vulkan 驱动程序我们要使用哪些全局扩展（比如与窗口系统的集成）
- 告诉驱动程序我们的应用程序信息（名称、版本等）
- 设置调试回调
- 枚举系统中可用的物理设备（GPU）


**Instance的创建过程恰恰说明了他的角色**：
```C++
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;

// 告诉 Vulkan 我们的应用程序信息
createInfo.pApplicationInfo = &appInfo;

// 告诉 Vulkan 我们需要哪些扩展
createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;

// 告诉 Vulkan 我们需要哪些验证层（用于调试）
createInfo.enabledLayerCount = validationLayers.size();
createInfo.ppEnabledLayerNames = validationLayers.data();
```

可以把 Instance 想象成一个"门户"或"接待员"：
```C++
// 没有 Instance 之前，我们无法调用大多数 Vulkan 函数
// 创建 Instance 后，我们可以做这些事：
vkEnumeratePhysicalDevices(instance, ...);  // 查询 GPU
vkCreateDebugUtilsMessengerEXT(instance, ...);  // 设置调试
// 等等
```

Instance可以被理解为一个“配置中心”，我们可以通过他告诉Vulkan：
- 这是我的应用程序
- 这是我需要的功能
- 这是我的调试需求


# 3. 创建 Window Surface

Window Surface 提供了 Vulkan 与窗口系统的连接：

```cpp
VkSurfaceKHR surface;
if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```
Vulkan 是与平台无关的图形 API，不直接处理窗口系统。Window Surface 是 `Vulkan` 和`窗口系统`之间的桥梁,它提供了一个可以渲染到的目标平面.
## 工作流程
>Vulkan 渲染流程 → Swapchain → Surface → 窗口系统 → 显示到屏幕

```C++
// 1. 创建 Surface
VkSurfaceKHR surface;
glfwCreateWindowSurface(instance, window, nullptr, &surface);

// 2. Surface 用于创建 Swapchain
VkSwapchainCreateInfoKHR createInfo{};
createInfo.surface = surface;  // Surface 告诉 Swapchain 渲染目标在哪里

// 3. 渲染时
vkAcquireNextImageKHR(...);    // 从 Swapchain 获取下一个可用的图像
// 渲染到图像
vkQueuePresentKHR(...);        // 通过 Surface 将渲染结果显示到窗口
```

## Surface的作用
- 提供图像呈现能力
  - 决定支持的图像格式
  - 决定支持的呈现模式
- 处理平台差异
  - Windows：使用 Win32 窗口系统
  - Linux：使用 X11 或 Wayland
  - macOS：使用 Metal 层

可以把 Surface 想象成一个"画布"：
- Vulkan 是画家（渲染器）
- Window 是画框（显示窗口）
- Surface 是画布，它把画家的作品（渲染结果）放在画框中展示


<div markdown="0" style="text-align: center;">

<img src="https://obsidian-picture-1313051055.cos.ap-nanjing.myqcloud.com/Obsidian/20250124230354.png" alt="image description" style="max-width: 100%; height: auto;">

<p style="text-align: center;">图 1：Surface 的作用。</p>

</div>

# 4. 获取物理设备

选择合适的物理设备（显卡）：

```cpp
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());

// 选择第一个适合的设备
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}
```

`isDeviceSuitable` 函数通常会检查以下几个关键方面来确定物理设备是否满足应用需求：
1. **基本设备信息检查**
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    // 获取设备基本属性
    VkPhysicalDeviceProperties deviceProperties;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    
    // 获取设备特性
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    // 检查是否为独立显卡
    bool isDiscreteGPU = deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU;
}
```

2. **队列族支持检查**
```cpp
bool checkQueueFamilySupport(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);
    
    // 检查是否支持所需的所有队列族
    // - 图形队列族
    // - 计算队列族
    // - 显示队列族
    return indices.isComplete();
}
```

3. **设备扩展支持检查**
```cpp
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    // 获取设备支持的扩展
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);
    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    // 检查必要的扩展是否被支持
    // 比如 VK_KHR_swapchain
    std::set<std::string> requiredExtensions = {
        VK_KHR_SWAPCHAIN_EXTENSION_NAME
    };
    
    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

4. **Swapchain 适配性检查**
```cpp
bool checkSwapChainAdequate(VkPhysicalDevice device) {
    // 检查 surface 格式
    uint32_t formatCount;
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);
    
    // 检查显示模式
    uint32_t presentModeCount;
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);
    
    return formatCount > 0 && presentModeCount > 0;
}
```

5. **内存属性检查**
```cpp
bool checkMemoryProperties(VkPhysicalDevice device) {
    VkPhysicalDeviceMemoryProperties memProperties;
    vkGetPhysicalDeviceMemoryProperties(device, &memProperties);
    
    // 检查是否有足够的显存
    // 检查是否支持所需的内存类型
    return true; // 根据具体需求判断
}
```

6. **综合评分系统（可选）**
```cpp
int rateDeviceSuitability(VkPhysicalDevice device) {
    int score = 0;
    
    // 基础分：独立显卡加分
    VkPhysicalDeviceProperties deviceProperties;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }
    
    // 性能分：根据最大纹理大小加分
    score += deviceProperties.limits.maxImageDimension2D;
    
    // 特性分：支持几何着色器加分
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
    if (deviceFeatures.geometryShader) {
        score += 100;
    }
    
    return score;
}
```

最终的设备选择函数如下：
```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    bool extensionsSupported = checkDeviceExtensionSupport(device);
    bool swapChainAdequate = false;
    
    if (extensionsSupported) {
        swapChainAdequate = checkSwapChainAdequate(device);
    }
    
    return checkQueueFamilySupport(device) && 
           extensionsSupported && 
           swapChainAdequate &&
           checkMemoryProperties(device) &&
           rateDeviceSuitability(device) > minRequiredScore;
}
```
## 5. 获取队列族索引

查找支持所需操作的队列族：

```cpp
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsAndComputeFamily; // 图形和计算共用一个队列族
    std::optional<uint32_t> presentFamily;
    
    bool isComplete() {
        return graphicsAndComputeFamily.has_value() && presentFamily.has_value();
    }
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    uint32_t queueFamilyCount = 0;
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);
    std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
    
    // 查找支持图形和计算的队列族
    // 查找支持显示的队列族
    // ... 具体实现略
    
    return indices;
}
```
## 队列族
队列族是一组具有相同功能的队列（Queue），可以理解为是物理设备的一部分，每个队列族支持特定类型的操作，比如：

- 图形操作（绘制命令）
- 计算操作（计算着色器）
- 传输操作（内存复制）
- 显示操作（显示到屏幕）

```
物理设备（GPU）
├── 队列族 0（支持图形+计算+传输）
│   ├── 队列 0
│   └── 队列 1
├── 队列族 1（仅支持传输）
│   └── 队列 0
└── 队列族 2（支持显示）
    └── 队列 0
```

可以查询每个队列族的队列数量、支持的特性，比如：

```C++
// 获取队列族属性
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueFamilyCount, nullptr);
std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueFamilyCount, queueFamilies.data());
// 遍历每个队列族，查看其中的队列数量
for (uint32_t i = 0; i < queueFamilyCount; i++) {
    const auto& queueFamily = queueFamilies[i];
    
    // queueCount 就是该队列族中的队列数量
    uint32_t numQueues = queueFamily.queueCount;
    
    // 打印队列族信息
    std::cout << "Queue Family " << i << ":\n";
    std::cout << "  Number of queues: " << numQueues << "\n";
    std::cout << "  Supports graphics: " << (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT ? "yes" : "no") << "\n";
    std::cout << "  Supports compute: " << (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT ? "yes" : "no") << "\n";
    std::cout << "  Supports transfer: " << (queueFamily.queueFlags & VK_QUEUE_TRANSFER_BIT ? "yes" : "no") << "\n";
}
```

## 队列族作用
在创建逻辑设备时需要制定使用的队列族，比如：
```C++
// 创建队列信息
float queuePriority = 1.0f;
VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = graphicsFamily;  // 指定队列族索引
queueCreateInfo.queueCount = 1;                     // 使用的队列数量
queueCreateInfo.pQueuePriorities = &queuePriority;  // 队列优先级
```

### 不同队列族
- 不同队列族的命令可以并行执行
- 专用队列族（如只支持传输的队列族）通常性能更好
- 需要在不同队列族之间同步操作时会有性能开销

### 相同队列族的不同队列
可以使用同一个队列族的两个队列提交命令：
```C++
// 获取同一队列族的两个队列
VkQueue queue1, queue2;
vkGetDeviceQueue(device, graphicsFamilyIndex, 0, &queue1);
vkGetDeviceQueue(device, graphicsFamilyIndex, 1, &queue2);

// 这两个队列可以并行执行命令
vkQueueSubmit(queue1, 1, &submitInfo1, fence1);  // 在队列1提交命令
vkQueueSubmit(queue2, 1, &submitInfo2, fence2);  // 在队列2提交命令

// 这两个提交会并行执行，不需要等待队列1完成
```

- 同一队列族的所有队列具有相同的能力（比如都支持图形操作）
- 每个队列都有**自己独立的命令流**
- 每个队列都可以独立提交命令缓冲区
- 队列之间的执行是异步的(**并行执行**，没有先后顺序），除非使用同步原语（使用同步原语，可以实现队列2等待队列1完成）

### 队列优先级和同步
```C++
// 创建队列时可以指定不同的优先级
float priorities[] = { 1.0f, 0.5f };  // 两个队列，不同优先级
VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.queueCount = 2;
queueCreateInfo.pQueuePriorities = priorities;
```
```C++
// 如果需要队列间同步，可以使用信号量
VkSubmitInfo submitInfo{};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = &waitSemaphore;    // 等待其他队列的信号量
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &signalSemaphore; // 发出完成信号
```

### 并行渲染举例
```C++
// 场景1：并行渲染多个对象
void renderScene() {
    // 队列1渲染地形
    vkQueueSubmit(queue1, 1, &terrainSubmitInfo, terrainFence);
    
    // 同时，队列2渲染角色
    vkQueueSubmit(queue2, 1, &characterSubmitInfo, characterFence);
    
    // 两个渲染任务并行执行
}

// 场景2：一个队列处理主要渲染，另一个处理后期效果
void render() {
    // 队列1执行主要渲染
    vkQueueSubmit(queue1, 1, &mainRenderSubmitInfo, mainRenderFence);
    
    // 设置依赖关系
    waitSemaphores = mainRenderComplete;
    
    // 队列2执行后期处理
    vkQueueSubmit(queue2, 1, &postProcessSubmitInfo, postProcessFence);
}
```


# 6. 创建逻辑设备
## 创建队列创建信息
如前文所说，逻辑设备的创建需要指定使用的队列族。为每个唯一的队列族创建创建信息结构体：

```C++
std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {
    indices.graphicsAndComputeFamily.value(),
    indices.presentFamily.value()
};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;   // 指定队列族索引
    queueCreateInfo.queueCount = 1;                    // 使用的队列数量
    queueCreateInfo.pQueuePriorities = &queuePriority;  // 队列优先级
    queueCreateInfos.push_back(queueCreateInfo);
}
```
## 创建逻辑设备
```cpp
VkDeviceCreateInfo deviceCreateInfo{};
deviceCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
deviceCreateInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
deviceCreateInfo.pQueueCreateInfos = queueCreateInfos.data();  // 指定队列创建信息

VkPhysicalDeviceFeatures deviceFeatures{};
deviceCreateInfo.pEnabledFeatures = &deviceFeatures;  // 指定设备特性

VkDevice device;
if (vkCreateDevice(physicalDevice, &deviceCreateInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

如前面提到的，队列族是物理设备的一部分，逻辑设备创建时需要指定使用的队列族和队列的数量，因此可以将逻辑设备理解为建立在队列族上对物理设备的抽象。
即：
```
物理设备（GPU）
    └── 逻辑设备（对GPU的抽象接口）
            ├── 队列族 0
            │   ├── 队列 0
            │   └── 队列 1
            └── 队列族 1
                └── 队列 0
```
上面展示了如何指定队列族和队列数量来创建逻辑设备，下面介绍如何从逻辑设备获取队列：
```C++
// 从逻辑设备获取队列
VkQueue graphicsQueue;
vkGetDeviceQueue(logicalDevice,  // 逻辑设备句柄
                 graphicsFamilyIndex,  // 队列族索引
                 0,  // 队列索引
                 &graphicsQueue);  // 获取到的队列
```

## 逻辑设备的功能
- 创建和管理各种 Vulkan 资源（缓冲区、图像等）
- 提供队列访问接口
- 启用设备特性和扩展
- 控制设备内存分配
例如：

```C++
// 使用逻辑设备创建资源
VkBuffer buffer;
vkCreateBuffer(logicalDevice, &bufferInfo, nullptr, &buffer);
// 使用逻辑设备分配内存
VkDeviceMemory memory;
vkAllocateMemory(logicalDevice, &allocInfo, nullptr, &memory);
```

## 不同的逻辑设备
- 一个应用程序可以创建多个逻辑设备
- 每个逻辑设备**都有自己的队列和资源**
- 不同逻辑设备间的资源**不能直接共享**
- 逻辑设备销毁时，其创建的所有资源也会被销毁


# 7. 创建命令池

创建用于管理命令缓冲区的命令池：

```cpp
VkCommandPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = indices.graphicsAndComputeFamily.value();  // 指定队列族索引
poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT; // 允许单独重置命令缓冲区

VkCommandPool commandPool;
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```
## 命令池的作用
- 命令池用于管理命令缓冲区的内存
- 每个命令池只能分配给特定的队列族使用
- 从同一个命令池分配的命令缓冲区只能提交到同一队列族的队列中(因为第二点指定了队列族的类型)

## 命令池标志
```C++
// 常用的命令池标志
VK_COMMAND_POOL_CREATE_TRANSIENT_BIT  // 提示命令缓冲区会频繁重录制，可以优化内存分配
VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT  // 允许单独重置命令缓冲区，而不是只能重置整个池
VK_COMMAND_POOL_CREATE_PROTECTED_BIT  // 创建受保护的命令缓冲区
```

## 分配命令缓冲
```C++
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;  // 指定命令池
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;  // 主要或次要命令缓冲区
allocInfo.commandBufferCount = 1;  // 分配数量

VkCommandBuffer commandBuffer;
vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
```

# 命令池（Command Pool）和命令缓冲区（Command Buffer）
- 命令池是内存池
- 命令缓冲区是从这个内存池分配的内存块
- 所有命令缓冲区共享命令池的属性（如队列族绑定）
- 命令池管理着所有命令缓冲区的生命周期
## 内存管理关系

```C++
// 命令池负责管理命令缓冲区的内存分配
VkCommandPool commandPool;
std::vector<VkCommandBuffer> commandBuffers;

// 从命令池分配命令缓冲区
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;              // 指定从哪个命令池分配
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = 1;                 // 分配数量

vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
```

## 生命周期关系
- 命令池控制着其分配的所有命令缓冲区的生命周期
- 销毁命令池时会自动销毁其分配的所有命令缓冲区
```C++
void cleanup() {
    // 不需要单独释放命令缓冲区
    vkDestroyCommandPool(device, commandPool, nullptr); // 会自动释放所有命令缓冲区
}
```
## 重置关系

```C++
// 重置整个命令池（影响所有命令缓冲区）
vkResetCommandPool(device, commandPool, VK_COMMAND_POOL_RESET_RELEASE_RESOURCES_BIT);

// 如果命令池创建时指定了 VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT
// 则可以单独重置命令缓冲区
vkResetCommandBuffer(commandBuffer, VK_COMMAND_BUFFER_RESET_RELEASE_RESOURCES_BIT);
```

## 队列族关系

```C++
// 命令池绑定到特定队列族
VkCommandPoolCreateInfo poolInfo{};
poolInfo.queueFamilyIndex = graphicsQueueFamily;  // 指定队列族

// 从该命令池分配的命令缓冲区只能提交到同一队列族的队列
VkSubmitInfo submitInfo{};
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;
vkQueueSubmit(graphicsQueue, 1, &submitInfo, fence);  // 队列必须属于同一队列族
```

# 实例扩展和设备扩展
## 实例扩展（Instance Extensions）：
**作用范围**：作用于整个 Vulkan 实例（VkInstance），影响全局功能
**主要用途**：
1. 提供跨平台功能，如窗口系统集成（WSI）
2. 添加调试和验证层支持
3. 提供实例级别的新功能

**加载时机**：在创建 VkInstance 时通过 vkCreateInstance 启用

**常见的实例扩展：**
VK_KHR_surface
- 最基础的窗口系统接口扩展
- 定义了创建和管理平台无关的窗口表面的基础功能
- 几乎所有需要显示的应用都会用到

平台特定的 surface 扩展：
- VK_KHR_win32_surface (Windows)
- VK_KHR_xlib_surface (X11/Linux) 
- VK_KHR_wayland_surface (Wayland/Linux)
- VK_KHR_android_surface (Android)
- VK_MVK_macos_surface (macOS)

VK_EXT_debug_utils
- 提供调试功能
- 允许为 Vulkan 对象添加标签和名称
- 支持调试信息的回调

VK_KHR_get_physical_device_properties2
- 获取物理设备的额外属性信息
- 常用于查询新功能的支持情况
## 设备扩展（Device Extensions）：
**作用范围**：作用于特定的物理设备（VkPhysicalDevice）和逻辑设备（VkDevice）
**主要用途**：
1. 提供特定硬件功能支持
2. 启用设备特定的渲染特性
3. 添加新的设备级API功能

**加载时机**：在创建 VkDevice 时通过 vkCreateDevice 启用

**常见的设备扩展：**
VK_KHR_swapchain
- 最基础的显示相关扩展
- 用于创建和管理交换链
- 实现帧缓冲和显示同步

VK_KHR_maintenance1/2/3
- 提供各种 API 改进和补充功能
- 修复早期版本的一些限制和问题

VK_KHR_dynamic_rendering
- 简化渲染流程
- 无需创建 render pass 对象
- 更灵活的渲染配置

VK_KHR_multiview
- 支持单次渲染传递到多个视图
- 用于 VR 等立体渲染场景

VK_KHR_shader_*系列：
```C++
VK_KHR_shader_float16_int8  // 支持16位浮点和8位整数
VK_KHR_shader_non_semantic_info  // 着色器附加信息
VK_KHR_shader_draw_parameters  // 绘制参数访问
```

# EasyVulkan的初始化设计
在EasyVulkan中，初始化主要由VulkanContext和VulkanDevice两个类完成。

## VulkanContext
主要用于管理各种Vulkan的对象和资源。
- 创建 Vulkan 实例，可选择设置验证层和调试回调。
- 拥有对 VulkanDevice、SwapchainManager、ResourceManager、CommandPoolManager 和可选的 SynchronizationManager 的引用。
- 协调高级生命周期（初始化、清理）。

## VulkanDevice
VulkanDevice类主要对物理设备和逻辑设备进行管理。
- 选择具有所需功能的物理设备，创建逻辑设备。
- 维护队列句柄（图形、计算、传输）。
- 集成 Vulkan 内存分配器（VMA）。

## 使用说明
借助VulkanDevice本身就是VulkanContext的成员，因此初始化可以简化为：
```C++
VulkanContext context;
context.initialize();
```
在VulkanContext中，我们可以借助Manager类来管理各种Vulkan的对象和资源。
例如，**使用SwapchainManager**：
- 创建和管理交换链
- 处理窗口调整事件
- 管理交换链图像和图像视图
- 提供图像获取和呈现功能

**使用ResourceManager**：
- 所有主要 Vulkan 资源的生成器接口（如BufferBuilder、ImageBuilder、ShaderModuleBuilder等）
- 自动资源跟踪和清理
- 基于名称的资源查找