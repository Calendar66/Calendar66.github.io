---
title: "Vulkan渲染通道"
tags:
    - Vulkan
    - EasyVulkan
    - RenderPass
    - Subpass
    - Attachment
    - Subpass Dependency
date: "2025-01-29"
thumbnail: "https://docs.vulkan.org/guide/latest/_images/memory_allocation_sub_allocation.png"
bookmark: true
---

# 引言
在Vulkan图形API的渲染管线中，渲染通道（Render Pass）是构建高效渲染流程的核心组件,它不仅描述了一次渲染操作中的**渲染目标（attachments）的使用方式**，还决定了多个渲染阶段（subpass）之间的**执行顺序和数据依赖关系**。本文将深入探讨Vulkan渲染通道的工作原理及其关键要素的实现细节。


## 渲染通道的本质
渲染通道（VkRenderPass）定义了渲染操作期间使用的帧缓冲附件集合及其使用方式。它通过明确指定附件的生命周期和依赖关系，允许驱动进行深层次优化。相较于传统图形API的隐式状态管理，Vulkan的显式声明机制可降低内存带宽消耗达30%以上。
渲染通道主要负责描述：
- **渲染目标**（Attachments） 的格式、加载/存储操作、采样数等属性；
- **子通道**（Subpasses） 中每个阶段如何使用这些渲染目标；
- 子通道间的**依赖关系**（Pipeline Dependencies），用于保证数据正确性和同步。


## 设计哲学
显式控制：开发者必须明确指定所有附件和子流程
执行优化：提前声明渲染流程使驱动能优化资源布局
依赖管理：精确控制子流程间的内存和执行顺序


# Attachment简介与创建方法
Attachment 通常指的是帧缓冲区中的渲染目标，比如颜色缓冲、深度缓冲或模板缓冲。每个attachment都需要在创建渲染通道时进行详细的描述，主要包括以下几个方面：
- 格式（Format）：如 VK_FORMAT_B8G8R8A8_UNORM、VK_FORMAT_D32_SFLOAT 等。
- 采样数（Samples）：多重采样时使用的采样数。
- 加载/存储操作（Load/Store Operations）：如在渲染开始时是清除还是保留已有数据，在渲染结束时是存储还是丢弃数据。
- 初始与最终布局（InitialLayout/FinalLayout）：表明attachment在渲染开始前和结束后的内存布局状态，便于Vulkan内部进行布局转换。

在创建渲染通道时，需要通过一个 VkAttachmentDescription 数组来描述所有的attachment。例如：

```C++
VkAttachmentDescription colorAttachment = {};
colorAttachment.format = swapchainImageFormat; // 指定Format
colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT; // 指定采样数
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR; // 指定加载操作
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE; // 指定存储操作
// For depth/stencil attachments
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE; // 指定模板加载操作
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE; // 指定模板存储操作
// End for depth/stencil attachments
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED; // 指定初始布局
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR; // 指定最终布局
```

# Subpass简介与创建方法
Subpass 是渲染通道中的一个阶段，每个subpass描述了在该阶段中如何使用和依赖attachment。

在创建渲染通道时，需要使用 VkSubpassDescription 结构体来描述每个subpass。一个subpass通常至少需要描述以下信息：
- Pipeline Bind Point：通常为 VK_PIPELINE_BIND_POINT_GRAPHICS，指明当前subpass将用于图形管线。
- 颜色附件引用（Color Attachments）：指明渲染阶段中将写入颜色数据的attachment。
- 输入附件引用（Input Attachments）：在一个subpass中可以读取之前subpass生成的数据。
- 深度/模板附件引用（Depth/Stencil Attachment）：如果需要使用深度或模板测试，则需要指定对应的attachment。
如下代码创建了一个简单的subpass：

```
VkAttachmentReference colorAttachmentRef = {};
colorAttachmentRef.attachment = 0;  // 引用上面定义的第一个attachment
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```
在这个例子中，我们创建了一个渲染阶段，该阶段会把渲染结果写入第0号attachment，并且要求该attachment处于 VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL 布局状态。


## 为什么需要subpass？
Vulkan采用subpass的设计模式，除了在API层面给出了更明确的渲染流程以外，还带来了以下好处：
1. 利用On-chip memory，减少内存带宽消耗。
    在移动端等功耗敏感型设备上，GPU普遍采用了TBR设计，减少对内存带宽的占用，其主要思想是在小块（tile）区域内完成大部分渲染操作，然后统一写回到内存。subpass 非常适合这种架构：在同一个 render pass 内的多个 subpass 可以在同一tile 的生命周期内连续处理(数据传输发生在On-chip memory上)，不必频繁地将数据在片上和内存之间来回传输。
2. 避免全局同步。
    传统渲染流水线中，可能需要使用全局的内存屏障来确保数据一致性，而在 subpass 内部，由于数据依赖关系已被明确定义，驱动和硬件就可以局部地处理同步，减少不必要的等待。

# Subpass Dependency简介与创建方法
在一个渲染通道中，多个subpass之间或subpass与外部操作之间往往存在数据依赖关系。为了确保数据的正确性和避免竞态条件，需要在渲染通道中明确声明这些依赖关系。这就是管线Dependency（Pipeline Dependency）的作用。

管线Dependency 允许开发者在subpass之间定义内存屏障和执行屏障，确保：
- 某个subpass的写操作完成后，下一个subpass读取数据时能够获得最新的结果；
- 在执行特定渲染操作前，所有前置的操作已经完成并且内存访问已经同步。

在Vulkan中，这种依赖关系通过 VkSubpassDependency 结构体进行描述。

## 如何使用管线依赖
假设有两个subpass：subpass0写入颜色数据，而subpass1需要读取这些数据作为输入attachment。在这种场景下，我们需要确保subpass0的写操作在subpass1开始读取之前已经完成。可以通过如下方式定义一个依赖：

```C++
VkSubpassDependency dependency = {};
dependency.srcSubpass = 0;  // 依赖源subpass
dependency.dstSubpass = 1;  // 依赖目标subpass

// 指定依赖的阶段与访问类型
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependency.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dependency.dstAccessMask = VK_ACCESS_INPUT_ATTACHMENT_READ_BIT;

// 对于跨subpass依赖，通常设置dependency.flags为0或VK_DEPENDENCY_BY_REGION_BIT
dependency.dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;
```

在上述例子中：
- srcSubpass 指定了依赖的来源，即subpass0；
- dstSubpass 指定了依赖的目标，即subpass1；
- srcStageMask 和 dstStageMask 指明了涉及的管线阶段；
- srcAccessMask 和 dstAccessMask 则描述了内存访问的类型。

此外，对于一些特殊情况（如初始状态与最终状态的同步），也可以将 srcSubpass 或 dstSubpass 设置为 VK_SUBPASS_EXTERNAL，以描述与渲染通道外部的依赖关系。

例如，定义一个计算预处理 -> 图形渲染的依赖：

```C++
VkSubpassDependency compToGraphic = {
    .srcSubpass = VK_SUBPASS_EXTERNAL, // 表示计算阶段
    .dstSubpass = 0,                   // 图形子流程索引
    .srcStageMask = VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    .dstStageMask = VK_PIPELINE_STAGE_VERTEX_INPUT_BIT |
                   VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT |
                    VK_ACCESS_SHADER_READ_BIT,
    .dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT
};
```
又或者定义一个图形渲染 -> 计算后处理的依赖：

```C++
VkSubpassDependency graphicToComp = {
    .srcSubpass = 1,                    // 最后一个图形子流程
    .dstSubpass = VK_SUBPASS_EXTERNAL,  // 后续计算阶段
    .srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,
    .dstStageMask = VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_SHADER_READ_BIT,
    .dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT
};
```
# EasyVulkan中的RenderPass