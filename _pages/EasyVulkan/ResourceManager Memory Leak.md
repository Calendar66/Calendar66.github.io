
---
title: "EasyVulkan资源管理中的内存泄漏问题"
tags:
    - Vulkan
    - EasyVulkan
    - Debug
date: "2025-07-07"
thumbnail: "https://obsidian-picture-1313051055.cos.ap-nanjing.myqcloud.com/Obsidian/LOGO.png"
bookmark: true
---
>近期的某项目中需要在每一帧动态创建新的资源。然而程序执行时内存占用逐渐增加，怀疑出现了内存泄漏问题。因此重新回顾了EasyVulkan的ResourceManager逻辑并进行了优化。

# 内存泄漏问题
之前的资源创建方式：
```
ShaderModuleBuilder& ResourceManager::createShaderModule() {
    return *new ShaderModuleBuilder(m_device,m_context);
}
```
这种方式会导致严重的内存泄漏问题：
- new ComputePipelineBuilder(...) 在自由存储区（堆）上创建了一个对象，并返回指向该对象的指针。
* 操作符解引用该指针，得到对象本身。
* 函数返回这个**堆上**对象的引用。

## 问题分析

```
ComputePipelineBuilder& builder = resourceManager.createComputePipeline();
// ... 使用 builder ...

// 更差的情况，创建了一个副本
ComputePipelineBuilder builder = resourceManager.createComputePipeline();
```
在这两种情况下，**都丢失了 new 返回的原始指针**。因为没有指针，**永远无法调用 delete 来释放这块在堆上分配的内存**。每次调用 createComputePipeline() 都会导致一块无法回收的内存，程序运行时间越长，消耗的内存就越多，最终可能导致程序崩溃。

**结论：绝对不要返回一个由 new 在函数内部创建的对象的引用。**

## 解决方法
```
ComputePipelineBuilder ResourceManager::createComputePipeline() {
    // 1. 在函数内部创建一个 ComputePipelineBuilder 临时对象
    // 2. 将这个临时对象作为返回值返回
    return ComputePipelineBuilder(m_device, m_context);
}
```
这是现代 C++ 中实现**工厂函数（Factory Function**的正确、安全且高效的方式。
- ComputePipelineBuilder(m_device, m_context) 在函数内创建了一个临时对象。
- 函数签名表明它将按值返回一个 ComputePipelineBuilder 对象。

返回对象的成本：
- C++中的RVO机制（返回值优化 Return Value Optimization）：**编译器会识别出这种情况**，并避免创建中间的临时对象。它会**直接在调用方的内存空间**（即接收返回值的那个对象的内存位置）上构造这个对象。这样一来，就完全跳过了任何拷贝或移动操作。从效果上看，几乎和返回引用一样快：
```
// 由于 RVO，ComputePipelineBuilder 对象会直接在 `builder` 的内存上构造
// 没有临时对象，没有拷贝，没有移动
ComputePipelineBuilder builder = resourceManager.createComputePipeline();
```

-  移动语义 (Move Semantics): 即使在少数 RVO 无法生效的情况下（例如，函数内有多个返回路径），C++11 的移动语义也会介入。**如果 ComputePipelineBuilder 有移动构造函数**，那么返回时会调用移动构造函数而非拷贝构造函数。移动通常非常廉价，它只是“窃取”临时对象的内部资源（如指针、句柄），而不需要深拷贝数据。

### 安全性与所有权
这种方式非常安全。**调用者会得到一个全新的、自己拥有的对象**。当这个对象离开其作用域时（例如函数结束、{} 块结束），**它的析构函数会被自动调用**，符合 RAII (Resource Acquisition Is Initialization) 原则。


# VMA资源对象管理
任何通过 VMA Create 函数创建的资源，都必须通过与之对应的 VMA Destroy 函数来清理。

-----

## vmaCreateImage

当调用 `vmaCreateImage()` 时，VMA实现如下操作：
1.  **分配内存 (Allocate Memory)**：VMA 从它管理的内存池中找到一块合适的 `VkDeviceMemory`，并处理所有复杂的内存类型选择和对齐问题。这个内存块由一个 `VmaAllocation` 对象来代表。
2.  **创建映像 (Create Image)**：VMA 调用标准的 Vulkan 函数 `vkCreateImage()` 来创建 `VkImage` 句柄。
3.  **绑定内存 (Bind Memory)**：VMA 调用 `vkBindImageMemory()` 将前面分配的内存绑定到新创建的映像上。

`vmaCreateImage` 将这三个步骤封装成了一个原子操作，极大地简化了开发。

因此，当需要销毁这个映像时，也必须执行相反的、对应的操作：解绑内存、销毁映像、释放内存。这正是 `vmaDestroyImage()` 函数的作用。

`vmaDestroyImage(allocator, image, allocation)` 会完成：
1.  **销毁映像句柄** (内部调用 `vkDestroyImage()`)。
2.  **释放内存块 `VmaAllocation`**，将其归还给 VMA 的内存池，以便后续的分配可以重新使用它。

-----

## 如果不使用 VMA 
如果用标准 Vulkan 函数来清理：
  * **只调用 `vkDestroyImage(device, image, nullptr)`：**

      * 成功销毁了 `VkImage` 句柄本身。
      * 但是，VMA 分配给它的那块 `VkDeviceMemory` (`VmaAllocation`) **完全没有被释放**。VMA 仍然认为这块内存正在被一个（现在已经不存在的）映像使用。
      * **结果：严重的内存泄漏。** VMA 的可用内存池会随着程序运行越来越小，最终可能导致内存耗尽。

  * **只调用 `vmaFreeMemory(allocator, allocation)`：**

      * 成功地将 `VmaAllocation` 归还给了 VMA 的内存池。
      * 但是，`VkImage` 句柄 **没有被销毁**。
      * **结果：严重的 Vulkan 资源泄漏。** Vulkan 驱动仍然保留着这个映像句柄的相关资源。Vulkan 的验证层（Validation Layers）会立即报错，提示有一个未被销毁的 `VkImage` 对象。

**结论：只有 `vmaDestroyImage()` 能够同时、正确地清理映像句柄和它所占用的内存。**

-----
## 正确的生命周期管理
```cpp
#include <vma/vk_mem_alloc.h>

// ... 假设已有 VmaAllocator allocator 和 VkDevice device ...

VkImage image;
VmaAllocation allocation;

// 1. 创建 Image
VkImageCreateInfo imageInfo = { ... };
VmaAllocationCreateInfo allocInfo = { };
allocInfo.usage = VMA_MEMORY_USAGE_AUTO; // 让VMA自动选择内存类型

VkResult result = vmaCreateImage(
    allocator,
    &imageInfo,
    &allocInfo,
    &image,       // 输出 VkImage 句柄
    &allocation,  // 输出 VmaAllocation 句柄
    nullptr       // 可选的 VmaAllocationInfo
);

if (result == VK_SUCCESS) {
    // ... 使用 image ...
}


// 2. 清理 Image (例如在程序退出或资源不再需要时)
// 必须同时传入 image 和 allocation 句柄
if (image != VK_NULL_HANDLE && allocation != VK_NULL_HANDLE) {
    vmaDestroyImage(allocator, image, allocation);
}
```

-----

### VMA 的通用配对规则

这个原则适用于 VMA 管理的所有主要资源类型：

  * `vmaCreateImage()`  -\> `vmaDestroyImage()`
  * `vmaCreateBuffer()`  -\> `vmaDestroyBuffer()`
  * `vmaAllocateMemory()` (如果只分配内存) -\> `vmaFreeMemory()`
  * `vmaCreatePool()` -\> `vmaDestroyPool()`
  * `vmaCreateAllocator()` -\> `vmaDestroyAllocator()`

始终确保资源创建和销毁调用是成对出现的，这样才能保证Vulkan 应用程序没有资源泄漏。