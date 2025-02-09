---
title: "vector.back()引发的指针错误"
tags:
    - 指针
    - Vulkan
    - Bug
date: "2025-02-06"
thumbnail: "https://www.kawabangga.com/wp-content/uploads/2015/04/logo-cpp.jpg"
bookmark: true
---


>在使用 Vulkan 进行图形编程时，我们经常需要构建 DescriptorSet 来管理资源绑定。本文将通过一个具体示例，讲解在构建 DescriptorSet 时可能遇到的一个指针失效问题，并探讨一种看似“解决”问题但实际上存在隐患的局部变量用法。

  

# 问题背景
在 DescriptorSetBuilder 类中，其成员函数 addImageDescriptor 用于添加图像描述符：

```C++
DescriptorSetBuilder &DescriptorSetBuilder::addImageDescriptor(
    uint32_t binding, VkImageView imageView, VkSampler sampler,
    VkImageLayout imageLayout, VkDescriptorType type) {

  VkDescriptorImageInfo imageInfo{};
  imageInfo.imageLayout = imageLayout;
  imageInfo.imageView = imageView;
  if (sampler != VK_NULL_HANDLE) {
    imageInfo.sampler = sampler;
  }
  m_imageInfos.push_back(imageInfo);

  VkWriteDescriptorSet write{};
  write.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
  write.dstBinding = binding;
  write.dstArrayElement = 0;
  write.descriptorType = type;
  write.descriptorCount = 1;
  write.pImageInfo = &m_imageInfos.back();

  m_writes.push_back(write);
  return *this;
}
```

在这段代码中，我们将 VkDescriptorImageInfo 对象存储到 m_imageInfos 这个 std::vector 中，然后将 **write.pImageInfo 指向 m_imageInfos 中的最后一个元素**，再把 VkWriteDescriptorSet 对象存入 m_writes。
看似简单的做法，却在函数返回后出现了指针失效的问题，导致 pImageInfo 指向的地址变成了无意义的值。

  

# 问题解析
**1. std::vector 的内存重分配**
当我们调用 m_imageInfos.push_back(imageInfo) 时，std::vector 有可能因为容量不足而重新分配内存。这会将已有的数据复制到新的内存块中，从而导致原来数据的地址改变。

• **指针失效**：当重新分配发生时，之前通过 &m_imageInfos.back() 获取的指针将指向已释放或不再使用的内存区域，这就是指针失效问题的根源。

  
**2. 局部变量的生命周期**
有一种不太严谨的“解决方法”是将 pImageInfo 指向一个局部变量，例如：

```
VkDescriptorImageInfo imageInfo{};
...
write.pImageInfo = &imageInfo;
```
这样做可以避免内存重新分配带来的不确定行为，看上去能达到目的，但实际问题在于：

• **局部变量生命周期短**：局部变量 imageInfo 的生命周期仅限于函数内部。当函数返回后，这块栈内存就会被释放或被后续的调用覆盖，导致 pImageInfo 指向了无效的内存区域。


# 正确的解决方法
为了解决上述问题，我们需要确保传递给 Vulkan 的 VkDescriptorImageInfo 数据在 Vulkan 调用期间始终有效。以下是几种推荐的做法：

**1. 使用稳定的容器存储数据**

继续使用 std::vector 存储 VkDescriptorImageInfo 对象：
• **预先分配足够空间**：在添加数据之前调用 m_imageInfos.reserve(预估数量)，以避免 push_back 过程中发生内存重分配。
• **管理好容器生命周期**：确保 m_imageInfos 的生命周期足够长，至少要覆盖整个 Vulkan 调用期间，直到调用 vkUpdateDescriptorSets 后再对其进行修改或释放。

  
**2. 避免使用局部变量的地址**
切记不要将 pImageInfo 指向一个局部变量，因为其生命周期过短，可能会在函数返回后导致未定义行为。即使当前测试时看似正常工作，也不能依赖这种方法，因为它的安全性无法得到保证。
