---
title: "Working with device memory using Vulkan"
date: 2018-09-16T21:48:34+02:00
---

[Vulkan] (and other modern low-level APIs, such as ***DirectX 12***) moves most of the complexity related to memory handling to the user's code.  
The API provides a very small set of functions.  
While this post focuses on the [Vulkan API], most ideas are applicable to ***DirectX 12*** as well.

# Heaps and types

*Device visible memory* is provided by [Vulkan] implementation in few *memory heaps*.
Each heap backs one or more *memory types*.
One memory type is backed by only one heap.
When the memory is allocated from particular type it utilize capacity of backing heap.
To obtain memory types and heaps of the physical device one should call [`vkGetPhysicalDeviceMemoryProperties`] function.

* **Note**: All logical devices created from same physical device will share memory heaps

Memory type defines set of properties, that all memory objects allocated from the type will have.
There are following memory properties defined in [Vulkan API]:

* [`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`] = `0x00000001`
  Memory with this property is fast for device access.
  Only heap with [`VK_MEMORY_HEAP_DEVICE_LOCAL_BIT`] flag backs memory with this property.
  Normally this flag should be set for memory that is physically on device.
  At least one memory type with this property is guaranteed to exist.

* [`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`] = `0x00000002`
  Memory with this property set may be accessed by the host (CPU) via mapping mechanism.
  At least one memory type with this property is guaranteed to exist.
  Not all devices will have memory types which are both device-local and host-visible.

* [`VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`] = `0x00000004`
  Memory with this property is always host-visible.
  The property signals that host access to the memory is coherent.
  Which in tuns means that there is no need to manually invalidate and flush mapped range.
  In many implementations all host visible is coherent.

* [`VK_MEMORY_PROPERTY_HOST_CACHED_BIT`] = `0x00000008`
  Memory with this property is always host-visible.
  This property signals that memory is cached on the host.
  Host access to the uncached memory is slower.

* [`VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT`] = `0x00000010`
  Memory types with device-only access can have this property.
  Which means that memory type can't have both [`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`] and [`VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT`] bits set.
  This flags signals that backing memory **may** be provided lazily.

* [`VK_MEMORY_PROPERTY_PROTECTED_BIT`] = `0x00000020`
  Memory types with device-only access can have this property.
  Which means that memory type can't have both [`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`] and [`VK_MEMORY_PROPERTY_PROTECTED_BIT`] bits set.
  This property specifies that protected queue operations to access the memory (whatever this means).

# Allocation

User allocates memory objects with [`vkAllocateMemory`] function specifying memory type and size of the memory required. The function returns memory in form of an opaque handle to the memory object.

The task to choose best memory type for each resource is now put on shoulders of the user.  
*Don't panic.*  
It is not as hard as it may seem.  
After creating resource user can obtain subset of memory types that can be used for resource using
[`vkGetBufferMemoryRequirements`] or [`vkGetImageMemoryRequirements`].  
Then it just a matter of picking memory type with all required properties, with as many as possible preferred properties, and
with as little as possible irrelevant properties. Also user should consider how much memory left in heaps.  
Simple solution would look like this:

1. *Filter out memory types that doesn't support resource.*
1. *Filter out memory types which heap is oversubscribed.*
1. *Filter out memory types without required properties.*
1. *For memory types left find one with maximum fitting value that can be calculated as weighted sum of properties.*
  * *Weight for desired property can be specified by user or positive preset value.*
  * *Weight for irrelevant property can be negative preset value.*

[`vkAllocateMemory`] is really expensive and implementation may allocate small amount of memory objects.
One can obtain that amount but it only guaranteed to be at least 2048.
It is strongly advised to allocate big chunks of memory with this function and use it to bind to the multiple buffers and images.
For start it can be allocations of 256 MiB (or 1/8 of heap size for small heaps).
Dedicated memory objects could be used for resources that require > 32 MiB.

Manual allocation of memory opens possibility to utilize higher-level knowledge of memory usage pattern to use allocation strategy that fits best.
For example one can use linear allocator for short-lived objects.

To free memory object user should call [`vkFreeMemory`]. User must ensure that device won't access freed memory.

# Host access

As mentioned before [`vkAllocateMemory`] returns opaque handle to newly allocated memory object.
The actual physical memory may not be CPU accessible RAM.
But if memory type has [`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`] it may be mapped for direct CPU access.
[`vkMapMemory`] function maps specified range of the host visible memory to the virtual memory accessible by CPU.
Memory range remains mapped until [`vkUnmapMemory`] is called for the memory object.

Only one range of the memory object can be mapped at a time.
This leads to two primary mapping strategy:

* Map - use - unmap
  Region of host visible memory is mapped to perform write or read by CPU and immediately unmapped afterwards.
  This strategy has performance cost as mapping memory may require pages to be allocated.
  But reduces RAM usage.

* Persistent mapping
  Whole memory object is mapped after allocation and unmapped before deallocation.
  This strategy doesn't require unnecessary ticks to map-unmap constantly.
  But greatly increase RAM usage.

In general user should pick suitable approach for each use case.
It is better to avoid map-unmap approach for uniforms as it may lead to thousands mapping operations per frame.
Short-lived mapping can also prove difficult for multi-threaded usage as application will require to ensure
that no other thread tries to map another memory region.

If host visible memory doesn't have [`VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`] property then
[`vkInvalidateMappedMemoryRanges`] function must be used on memory range before read by CPU,
otherwise writes made by GPU are not guaranteed to be visible by CPU.
Similarly [`vkFlushMappedMemoryRanges`] must be used after CPU writes.
More about this in following post about synchronization.

# Usage

Allocation and mapping strategies together gives us many different combinations.
But most use cases fall in 4 major categories.

* **`Data`**
  Memory that is only accessed by device. This includes framebuffers, textures, meshes etc.
  For those resources one should use device local memory types. Avoid using host visible memory type as it is useful for another usage.

* **`Upload`**
  To initialize textures and meshes staging technique requires host visible memory type.
  Avoid using device local and cached memory as it is useful for another usage. Host coherency is not really important here.

* **`Download`**
  Sometimes application needs to get result of computation done by device.
  It can be anything from captured frame to result of physics simulation done in compute shader.

* **`Dynamic`**
  Some memory is expected to be rewritten by CPU each frame.
  Common use case is uniform buffers.
  For this usage host visible memory is required and device local memory preferred.
  Not all GPUs has a memory type that is both host visible and device local.
  And even if such memory type exists its heap is relatively small, so it shouldn't be wasted for usages that doesn't require both properties.

# Binding and aliasing

To associate memory range with particular resource it must be bound.
[`vkBindBufferMemory`] function binds specified memory range to the buffer.
[`vkBindImageMemory`] function binds specified memory range to the image.
Once bound to memory range resource can't be rebound to another.
*This restriction doesn't apply to sparse resources. We'll talk about them later.*

Yet it is possible to bind overlapping ranges *(and even same range)* to the multiple resources.
When two resources are bound to overlapped ranges they are ***alias*** intersection of ranges.
Usage if ***aliased*** memory is subject to several constraints.
The only reason to ***alias*** memory is to reduce total memory usage.
Unless [*specific constrains*] are met then data written through one alias cannot be consistently read through another.

Common and simple use case for aliased memory is *transient resources*, e.g. resources that get rewritten each frame without trying to read it first.
This is true for most framebuffer attachments as they usually cleared before first use.
If last read from one transient resource is done before rewrite of second resource then it is safe to ***alias*** that pair of resources.
Properties of transient resources is known at setup time so it must be possible to allocate them in memory object with size smaller than total sum of size required by all transient resources.

# Existing solutions

For those who don't want to reinvent the wheel there are libraries to solve the task.

If you write in C or C++, or just want to use library with C API you can try [VulkanMemoryAllocator].

Rust folks who use [gfx-hal] can try [gfx-memory].
For [ash] users and those who like new shiny stuff [Rendy]'s memory manager could be of interest.

# What's left out of scope

* Multi-device
  Memory allocation for multiple physical devices is not a topic for first post :)

* Sparse resources
  require sophisticated memory binding while this topic is more about allocation.

* Synchronization
  requires setting up correct memory dependency.  
  Will be further explained in post about synchronization.

[Vulkan]: https://www.khronos.org/vulkan/
[Vulkan API]: https://www.khronos.org/registry/[vulkan]/specs/1.1-extensions/html/vkspec.html
[`vkGetPhysicalDeviceMemoryProperties`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkGetPhysicalDeviceMemoryProperties.html
[`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`VK_MEMORY_PROPERTY_HOST_CACHED_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`VK_MEMORY_PROPERTY_PROTECTED_BIT`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkMemoryPropertyFlagBits.html
[`vkAllocateMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkAllocateMemory.html
[`vkGetBufferMemoryRequirements`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkGetBufferMemoryRequirements.html
[`vkGetImageMemoryRequirements`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkGetImageMemoryRequirements.html
[`vkFreeMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkFreeMemory.html
[`vkMapMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkMapMemory.html
[`vkUnmapMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkUnmapMemory.html
[`vkInvalidateMappedMemoryRanges`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkInvalidateMappedMemoryRanges.html
[`vkFlushMappedMemoryRanges`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkFlushMappedMemoryRanges.html
[`vkBindBufferMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkBindBufferMemory.html
[`vkBindImageMemory`]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/vkBindImageMemory.html
[*specific constrains*]: https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#resources-memory-aliasing
[VulkanMemoryAllocator]: https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator
[gfx-hal]: https://github.com/gfx-rs/gfx
[gfx-memory]: https://github.com/gfx-rs/gfx-memory
[ash]: https://github.com/MaikKlein/ash
[Rendy]: https://github.com/omni-viral/rendy
