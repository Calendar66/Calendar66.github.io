---
title: "Vulkan中的同步机制"
tags:
    - Vulkan
    - EasyVulkan
    - VMA
date: "2025-01-29"
thumbnail: "https://docs.vulkan.org/guide/latest/_images/memory_allocation_sub_allocation.png"
bookmark: true
---

>Vulkan 通过提供多样且细粒度的同步机制，为开发者在控制渲染和计算流程时带来了极大的灵活性。Barrier、Semaphore、Fence 以及 Subpass Dependencies 各有不同的适用场景和影响：
- Barrier 强调 GPU 内部流水线阶段及内存的同步，适合在单个队列内确保读写有序。
- Semaphore 强调队列之间的同步，用来连接多队列的工作流。
- Fence 强调 CPU 对 GPU 任务完成的可见性，用于资源回收和多帧并行调度。
- Subpass Dependencies 强调在同一 Render Pass 内分阶段进行渲染时的同步，更高效地处理共享附件。

# Vulkan中的同步机制详解
在传统图形API如OpenGL中，驱动程序会自动处理资源同步，开发者无需关心底层执行顺序。但这种"黑箱"机制带来了两个严重问题：**性能损耗不可控和多线程扩展困难。**
Vulkan 作为现代图形和计算的低层次API，其设计核心之一就是让开发者可以更细粒度地控制GPU和CPU之间的工作流程，以及不同GPU队列之间的执行顺序。而要实现稳定且高性能的渲染或计算，就必须要合理地利用好各种同步机制。本文将从几个常见的 Vulkan 同步原语（Barrier、Semaphore、Fence、Subpass Dependencies）入手，探讨它们各自的概念、适用场景、性能影响以及使用注意事项。希望通过本文，能为正在使用 Vulkan 或即将使用 Vulkan 的读者提供一些实践上的参考。

#  命令缓冲与队列：线性流但可乱序完成
在 Vulkan 中，所有命令都要先记录在 VkCommandBuffer 中，再提交到某个 VkQueue。在单个队列中，你提交的命令会按顺序进入 GPU 执行管线；但 GPU 可能在还没完成某个命令的写操作时，就已经开始处理后续命令的读阶段。

- 逻辑顺序：提交顺序一定依次排队
- 实际执行：可以重叠 / 并行 / 乱序完成

#  内存模型与缓存一致性
现代GPU采用分级缓存设计：
>DDR显存 → L2缓存 → L1缓存（每个SM） → 寄存器

当计算单元写入L1缓存后，数据不会立即同步到其他缓存层级。这就是**非一致性内存访问**的根源。
“可用（Available）”意味着数据已经被刷出到更大层级缓存或主存；“可见（Visible）”意味着后续阶段可以读取到最新的数据——需要对读取方无效化缓存或更新缓存。
或者说：
- **Available**：源阶段写完数据后，做缓存 Flush，让数据到了 L2 或更高层共享区域
- **Visible**：目的阶段需要读数据时，做缓存 Invalidate，从而迫使硬件从 L2 或更高层共享区域读取数据，避免读到陈旧缓存

例如：

```C++
VkMemoryBarrier memBarrier{
    .sType = VK_STRUCTURE_TYPE_MEMORY_BARRIER,
    .srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_SHADER_READ_BIT
};

vkCmdPipelineBarrier(cmd,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,
    0, 1, &memBarrier, 0, nullptr, 0, nullptr);
```
这个屏障完成：
- 刷新所有计算阶段的写入到L2缓存（可用性）
- 使片段着色器能读取最新数据（可见性）

由于非一致性内存访问问题的存在，*Vulkan还要求开发者显式地管理内存相关屏障，以确保数据在不同阶段之间的可见性和可用性。*

因此，Vulkan的同步机制主要有两个目的：
- 确保数据在不同阶段之间的可见性和可用性(主要借助内存相关屏障)
- 确保执行顺序(主要借助pipeline屏障、semaphore、fence等)


# Barrier（屏障）

## Barrier的概念

Vulkan 中的 Barrier 是一种细粒度的内存和执行顺序同步机制。Barrier 在 GPU 内部起到"分割线"的作用，确保某些阶段的操作在 Barrier 之前完成，才能进行后续的阶段。例如，在进行纹理的读写转换时，需要使用 Pipeline Barrier 来保证图像布局转换或访问掩码的更改已完成，才进行下一步的采样或写入。

Barrier 有多种类型，最常见的包括：
- Pipeline Barrier：用于指定源阶段（srcStageMask）到目标阶段（dstStageMask）的内存和执行依赖。
- Memory Barrier：作用在整个资源上，用于指定对内存可见性的限制与保证。
- Buffer Memory Barrier：只作用在特定的 Buffer 范围上。
- Image Memory Barrier：只作用在特定的图像资源上，可以指定图像布局转换（image layout transition）。

## Opengl中的Barrier和Vulkan对比
在Opengl中同样存在Barrier的概念，但是Vulkan中的barrier提供了更细粒度的控制。

```C++
// OpenGL隐式同步
glDispatchCompute(1024, 1, 1);  // 计算着色器写入数据
glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT); 
glDrawArrays(GL_TRIANGLES, 0, 3); // 读取计算数据

// Vulkan显式同步
vkCmdDispatch(computeCmd, 1024, 1, 1);
VkMemoryBarrier barrier{...};
vkCmdPipelineBarrier(computeCmd, 
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    VK_PIPELINE_STAGE_VERTEX_SHADER_BIT,
    0, 1, &barrier);
vkCmdDraw(graphicCmd, 0, 3);
```

## Barrier的适用场景
- 图像布局转换：如从 VK_IMAGE_LAYOUT_UNDEFINED 转为 VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL，或者从 VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL 转为 VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL，在开始或结束渲染通道时需要合适的图像布局。
- **内存可见性保证**：当一个操作写入资源，另一个操作要读取该资源时，需要添加Barrier确保写入可见并完成。
- **不同着色阶段间的同步**：例如，当顶点着色器阶段写入Buffer后，需要在片元着色器阶段进行读取，可通过Barrier来控制依赖顺序。

## 管线阶段分解
Vulkan将GPU工作分解为可组合的阶段：
- `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`: 表示管线的起始阶段。
- `VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT`: 间接绘制命令的阶段。
- `VK_PIPELINE_STAGE_VERTEX_INPUT_BIT`: 顶点输入操作的阶段。
- `VK_PIPELINE_STAGE_VERTEX_SHADER_BIT`: 顶点着色器执行的阶段。
- `VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT`: 片段着色器执行的阶段。
- `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT`: 写入颜色附件的阶段。
- `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT`: 计算着色器执行的阶段。
- `VK_PIPELINE_STAGE_TRANSFER_BIT`: 内存传输操作的阶段。
- `VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT`: 表示管线的结束阶段。


## 访问掩码
GPU 有多级缓存（L1、L2），不同阶段可能对同一块内存资源有不同的缓存策略。为了避免缓存不一致（Incoherent），Vulkan 提供了 VK_ACCESS_* 标志来精确说明某个阶段对资源的访问类型。通过source access 与 destination access 结合，可以告诉 Vulkan “我要保证前面写的数据，在后面读的时候一定可见（Visible）”。

具体包括：
- `VK_ACCESS_INDIRECT_COMMAND_READ_BIT`: 对间接命令数据的读取访问。
- `VK_ACCESS_INDEX_READ_BIT`: 对索引缓冲区的读取访问。
- `VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT`: 对顶点属性的读取访问。
- `VK_ACCESS_UNIFORM_READ_BIT`: 对统一缓冲区的读取访问。
- `VK_ACCESS_SHADER_READ_BIT`: 对着色器存储的读取访问。
- `VK_ACCESS_SHADER_WRITE_BIT`: 对着色器存储的写入访问。
- `VK_ACCESS_COLOR_ATTACHMENT_READ_BIT`: 对颜色附件的读取访问。
- `VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT`: 对颜色附件的写入访问。
- `VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT`: 对深度/模板附件的读取访问。
- `VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT`: 对深度/模板附件的写入访问。

## Barrier的使用
一个Barrier的定义如下：

```C++
void vkCmdPipelineBarrier(
    VkCommandBuffer                             commandBuffer,          // 记录barrier的命令缓冲区
    VkPipelineStageFlags                        srcStageMask,          // 源管线阶段掩码，指定哪些管线阶段必须在barrier之前完成
    VkPipelineStageFlags                        dstStageMask,          // 目标管线阶段掩码，指定哪些管线阶段必须等待barrier
    VkDependencyFlags                           dependencyFlags,        // 依赖标志，如VK_DEPENDENCY_BY_REGION_BIT表示区域依赖
    uint32_t                                    memoryBarrierCount,     // 全局内存屏障数量
    const VkMemoryBarrier*                      pMemoryBarriers,       // 全局内存屏障数组
    uint32_t                                    bufferMemoryBarrierCount, // 缓冲内存屏障数量
    const VkBufferMemoryBarrier*                pBufferMemoryBarriers,   // 缓冲内存屏障数组
    uint32_t                                    imageMemoryBarrierCount, // 图像内存屏障数量
    const VkImageMemoryBarrier*                 pImageMemoryBarriers    // 图像内存屏障数组
);
```

- `VkDependencyFlags`主要用于控制屏障的行为，设置为0表示默认行为，同步将在整个渲染区域上全局进行，对于所有区域(所有像素)，所有指定的源操作都必须在任何目标操作开始之前完成；设置为VK_DEPENDENCY_BY_REGION_BIT则允许基于区域的依赖，同步只在每个区域内进行，而不是整个渲染目标，不同区域之间可以并行处理，适合TBR架构。
- 如果不使用内存相关的屏障，该命令定义了一个执行屏障，即在srcStageMask和dstStageMask之间插入一个同步点，确保所有指定源操作都完成，目标操作才开始。
- 如果使用内存相关的屏障，则屏障会根据自身的srcAccessMask和dstAccessMask的值，在srcStageMask和dstStageMask之间插入一个同步点，确保所有指定源操作都完成，目标操作才开始。

## 全局内存屏障

```C++
// 全局内存屏障：影响所有内存访问
typedef struct VkMemoryBarrier {
    VkStructureType    sType;                // 结构体类型，必须是 VK_STRUCTURE_TYPE_MEMORY_BARRIER
    const void*        pNext;                // 扩展信息指针，通常为nullptr
    VkAccessFlags      srcAccessMask;        // 源访问掩码，指定在barrier之前必须完成的内存访问类型
    VkAccessFlags      dstAccessMask;        // 目标访问掩码，指定必须等待barrier的内存访问类型
} VkMemoryBarrier;
```

全局内存屏障由 VkMemoryBarrier 结构体描述，用于同步整个 GPU 内存中的所有数据。**它并不针对具体的缓冲区或图像**，而是作用于全局范围内的内存访问。通过设置源和目标访问掩码（srcAccessMask 和 dstAccessMask），开发者可以确保在某个阶段完成的所有内存写操作对后续阶段的所有内存读取操作可见。

## 缓冲区内存屏障

```C++
// 缓冲内存屏障：针对特定缓冲区的内存访问
typedef struct VkBufferMemoryBarrier {
    VkStructureType    sType;                // 结构体类型，必须是 VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER
    const void*        pNext;                // 扩展信息指针，通常为nullptr
    VkAccessFlags      srcAccessMask;        // 源访问掩码
    VkAccessFlags      dstAccessMask;        // 目标访问掩码
    uint32_t          srcQueueFamilyIndex;   // 源队列族索引，用于队列族所有权转移
    uint32_t          dstQueueFamilyIndex;   // 目标队列族索引，用于队列族所有权转移
    VkBuffer          buffer;                // 受影响的缓冲区对象
    VkDeviceSize      offset;                // 受影响区域的起始偏移
    VkDeviceSize      size;                  // 受影响区域的大小
} VkBufferMem
```

缓冲内存屏障由 VkBufferMemoryBarrier 结构体描述，专门用于同步对**特定 VkBuffer 的内存访问**。它不仅包含全局内存屏障的所有属性，还能指定屏障**所作用的缓冲区、起始偏移量和数据大小。**

srcQueueFamilyIndex和dstQueueFamilyIndex用于队列族所有权转移，当使用队列族所有权转移功能时，需要指定源队列族索引和目标队列族索引。如果不涉及队列族所有权转移，设置为VK_QUEUE_FAMILY_IGNORED。当计算队列族和图形队列族分离时，如果需要在图形队列中访问计算队列族的资源，需要使用队列族所有权转移(或者在资源创建时设置资源共享)。

## 图像内存屏障

```C++

// 图像内存屏障：针对特定图像的内存访问
typedef struct VkImageMemoryBarrier {
    VkStructureType            sType;               // 结构体类型，必须是 VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER
    const void*                pNext;               // 扩展信息指针，通常为nullptr
    VkAccessFlags             srcAccessMask;        // 源访问掩码
    VkAccessFlags             dstAccessMask;        // 目标访问掩码
    VkImageLayout             oldLayout;            // 转换前的图像布局
    VkImageLayout             newLayout;            // 转换后的图像布局
    uint32_t                  srcQueueFamilyIndex;  // 源队列族索引
    uint32_t                  dstQueueFamilyIndex;  // 目标队列族索引
    VkImage                   image;                // 受影响的图像对象
    VkImageSubresourceRange   subresourceRange;     // 受影响的图像子资源范围
} VkImageMemoryBarrier;

// 子资源范围
typedef struct VkImageSubresourceRange {
    VkImageAspectFlags    aspectMask;       // 图像方面(颜色、深度、模板等)
    uint32_t             baseMipLevel;      // 基础mip级别
    uint32_t             levelCount;        // mip级别数量
    uint32_t             baseArrayLayer;    // 基础数组层
    uint32_t             layerCount;        // 数组层数量
} VkImageSubresourceRange;
```

图像内存屏障由 `VkImageMemoryBarrier` 结构体描述，专门用于同步对**特定 VkImage 的内存访问**。它不仅包含全局内存屏障的所有属性，还能指定屏障**所作用的图像、子资源范围**。

`VkImageSubresourceRange` 用于指定图像的哪些部分受到影响。
- aspectMask：指定图像的方面，如颜色、深度、模板等。
- baseMipLevel：指定起始的mip级别。
- levelCount：指定mip级别的数量。
- baseArrayLayer：指定起始的数组层。
- layerCount：指定数组层的数量。

除此之外，`VkImageMemoryBarrier` 还包含`oldLayout`和`newLayout`，用于指定图像布局的转换。
>在 Vulkan 中，图像布局（Image Layout）用于描述 GPU 内部如何组织和访问图像数据。相比 OpenGL 的隐式管理，Vulkan 要求开发者显式指定和转换图像布局，方便GPU明确用途并进行对应的性能优化。

## Barrier的性能影响
- 过度使用导致性能损耗：每个 Barrier 都会在流水线上插入一个同步点，如果频繁地插入不必要的Barrier，会增加**GPU的停顿**并降低并行效率。
- 恰当使用可以避免错误和竞态：Barrier 在正确的位置使用能够让数据流安全地"串行化"，避免读写冲突。

## 使用注意事项
- 阶段掩码精确化：在指定 srcStageMask 和 dstStageMask 时，要尽量精确地指定真实会产生和需要依赖的着色阶段或管线阶段，以减少不必要的同步。
- 布局转换与访问掩码：Barrier 需要指定图像的布局转换和访问掩码（srcAccessMask/dstAccessMask），要确保与实际使用场景匹配。
- 批量Barrier：避免在不同的资源上反复调用单个Barrier，可以把多个资源的Barrier一起批量提交，减少命令开销。

# Events：更灵活的同步利器
Pipeline Barrier 适用于同一个命令缓冲里强制“先做完 A，再做 B”。但如果你希望在并行中更灵活地“给 GPU 发信号”，可以用 Events:
1. Set Event (vkCmdSetEvent)：在指定的 pipeline 阶段完成后，标记某个事件为已触发
2. Wait Event (vkCmdWaitEvent)：在指定管线阶段等事件触发后，再继续执行

例如：
```C++
// A 和 B 两次计算
vkCmdDispatch(...);
vkCmdDispatch(...);

// 设置事件，表示 A 和 B 都完成后触发
vkCmdSetEvent(event, VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT);

// D 是独立计算，可与 A、B 并行
vkCmdDispatch(...);

// 在绘制（C）前等待事件，确保 A、B 完成
vkCmdWaitEvents(1, &event, VK_PIPELINE_STAGE_VERTEX_SHADER_BIT, ...);

// 真正的绘制
vkCmdDraw(...);
```
这样便可让 D 与 A、B 并行，只有在需要结果依赖的绘制（C）时才去等待。


# Semaphore（信号量）

## 概念

Semaphore（信号量）在 Vulkan 中主要用于**队列之间**的执行顺序同步。它不能被 CPU 查询，CPU无法通过 Semaphore 判断什么时候 GPU 完成了某一条指令；它更适合在 GPU 内部或不同队列之间"串起"执行顺序。例如，我们可能在一个队列上进行图像后处理，然后在另一个队列上进行呈现；在这两个队列之间需要一个 Semaphore 让呈现等待后处理完成。

## Semaphore的适用场景
- 不同队列间依赖：渲染队列和呈现队列之间交换图像通常会用到 Semaphore 来协调。例如，vkAcquireNextImageKHR 返回的信号量会在图像可用时触发，渲染结束后再用一个信号量告知显示队列可以进行呈现。
- 多通道并行处理：如果使用多队列同时处理不同任务，在队列间需要明确地指定执行顺序时，使用 Semaphore 进行同步。

## Semaphore的性能影响
- 轻量但仅限 GPU 内部：相比Fence来说，Semaphore 更轻量，因为它无需让 CPU 轮询或等待，也不需要在 CPU 端可见。通常可以让不同队列并发工作，充分利用 GPU 资源。
- 等待延迟：如果在一个队列上等待另一个队列信号量，会引入一定延迟，要结合管线设计和命令流来优化。

# Fence（栅栏）

## 概念
Fence（栅栏）是另一种同步原语，与 Semaphore 不同的是，Fence 可以被 CPU 端查询。当一个命令缓冲区在 GPU 上执行完成后，Fence 会被置为已信号状态（signaled state），CPU 通过 vkWaitForFences 或 vkGetFenceStatus 等函数能够得知 GPU 的执行状态。

## Fence的适用场景
- GPU 任务结束的 CPU 检测：在需要 CPU 端等待 GPU 完成某些任务（比如更新一块缓冲区后，需要 CPU 端再做一些操作）时使用。
- 动态资源回收：如果想知道 GPU 什么时候真正使用完一块资源（例如上一帧的 Uniform Buffer），Fence 可以帮助我们在 CPU 上安全地回收或复用资源。
- 多缓冲策略中的帧同步：常用在多帧并行（triple buffering 或 double buffering）时，CPU需要知道当前帧是否可以安全地写入时，会查询对应的 Fence。

## Fence的性能影响
- CPU/GPU 同步开销：Fence 一旦被 CPU 等待（vkWaitForFences），就会造成 CPU 和 GPU 之间的同步停顿，影响并行度。
- 延迟增加：如果没有必要的地方使用 Fence，会导致 CPU 过度等待，引起帧率下降或延迟增大。

## 使用注意事项
- 批量等待：在需要等待多个Fence时，尽量使用批量等待，而不是一个个等待。
- 复用 Fence：Fence 在被 signaled 之后，可通过 vkResetFences 重置来重复使用，避免频繁创建销毁。
- 避免无谓等待：在管线设计中，如果可以让 CPU 继续做其他工作，就尽量不要阻塞 CPU，只有在需要保证资源一致性时再使用 Fence。

>Queue Submit & 信号量等待会带来隐式内存保证：
- 当一个队列提交被另一个队列通过信号量（Semaphore）等待时，Vulkan 会隐式地完成所有写入的 flush，使得内存可见给后续队列。
- 当一个队列提交完成并且 Fence 被置位，表示 GPU -> CPU 的所有工作也可见。

# Subpass Dependencies（子通道依赖）
## 概念
Vulkan 中的 Render Pass 可以由多个 Subpass 组成，每个 Subpass 是一个渲染阶段。Subpass Dependencies 用于指定 Subpass 之间的执行和内存依赖关系，从而在同一个 Render Pass 内实现图像的输入/输出同步。因此在同一个 Render Pass 内，Subpass Dependencies 可以减少外部 Barrier 的使用。

一般地，Subpass Dependencies 会指定：
- 源子通道（srcSubpass）和目标子通道（dstSubpass）
- 源阶段与目标阶段（srcStageMask, dstStageMask）
- 源访问掩码与目标访问掩码（srcAccessMask, dstAccessMask）
- 依赖标志（dependencyFlags）

## Subpass 依赖的适用场景
- 多渲染阶段共享同一图像：例如，在一个 Subpass 中写入颜色附件，接下来一个 Subpass 需要使用它作为输入附件（input attachment）。
- 分阶段渲染：如果想在一个 Render Pass 内连续执行多个着色阶段，而这几个阶段都在同一个 GPU 队列上运行，那么 Subpass Dependencies 就是最合适的同步方式。

## Subpass 依赖的性能影响
- 减少开销：使用 Subpass Dependencies 在同一个 Render Pass 内可以减少图像布局切换和相关命令的开销。
- 提高带宽利用率：Subpass Dependencies 可以帮助 Vulkan 在同一 Render Pass 内合理地利用附件。

## 使用注意事项
- 需要在创建 Render Pass 时指定：一旦 Render Pass 的依赖关系确定，就不能再动态修改。
- 适当规划多 Subpass 结构：过度的 Subpass 拆分会导致复杂的依赖管理，不是所有的场景都适合在一个 Render Pass 内解决。
- 配合 input attachment 使用：当一个 Subpass 的输出作为下一个 Subpass 的 input attachment 时，需要正确设置依赖，确保不会读写冲突。

# 同步方法对比
不同的同步机制往往对应着不同的使用层次和需求，下面简要对比：

| 同步原语 | 主要作用 | 适用场景 | CPU 可见性 | 性能影响 |
|---------|---------|----------|------------|----------|
| Barrier | GPU 内部执行与内存同步 | 管线阶段、内存访问控制、布局转换 | 不可见 | 需要精确指定，过多会影响并行性 |
| Semaphore | 队列间同步 | 多队列交互，如图像获取与提交 | 不可见 | 轻量级，主要在GPU端 |
| Fence | CPU 等待 GPU 完成 | 资源回收、GPU任务结束时需 CPU 介入 | 可见 | CPU 阻塞可能拖慢帧率 |
| Subpass | Render Pass 内部同步 | 多个渲染阶段共享附件，减少外部Barrier | 不可见 | 在同一 Render Pass 内更高效 |

# 设计与实践建议

1. 尽量减少无意义的Barrier
   - 保证数据访问安全的前提下，减少不必要的 Pipeline Barrier，可以合并多个资源的Barrier或者使用更精准的阶段掩码。

2. 巧用Subpass减少外部同步
   - 同一 Render Pass 内的多个阶段尽量用 Subpass Dependencies 处理，可以避免过多的图像布局切换和额外的 Barrier 开销。

3. 合理划分队列并使用Semaphore
   - 如果 GPU 拥有异步计算队列或传输队列，适当将工作分摊在不同队列。使用 Semaphore 进行队列间同步，充分利用 GPU 并行。

4. Fence用于 CPU/GPU 交互
   - 只有在需要 CPU 等待 GPU 的结果时才使用 Fence，避免无谓的阻塞。要注意等待方式（阻塞或轮询）的选择及资源回收。

5. 监控和调试
   - 使用 Vulkan 的验证层（Validation Layers）或 GPU 调试工具，来确认 Barrier 和同步的正确性，避免出现 GPU 死锁或数据争用。



