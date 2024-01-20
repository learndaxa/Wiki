## Description

The device in Daxa is essentially a `VkDevice` in Vulkan, but it carries much more responsibility.

## Creation

### Sample

```cpp
daxa::Device device = instance.create_device({
    .selector = [](daxa::DeviceProperties const & device_props) -> daxa::i32
    {
        daxa::i32 score = 0;
        switch (device_props.device_type)
        {
        case daxa::DeviceType::DISCRETE_GPU: score += 10000; break;
        case daxa::DeviceType::VIRTUAL_GPU: score += 1000; break;
        case daxa::DeviceType::INTEGRATED_GPU: score += 100; break;
        default: break;
        }
        score += static_cast<daxa::i32>(device_props.limits.max_memory_allocation_count / 100000);
        return score;
    },
    .name = "my device",
});
```

## Graphics Queue

As all applications that use daxa will have a GPU with a graphics queue, daxa::Device also represents the graphics queue. This is not a problem regarding the multi-queue as no modern GPU has multiple hardware graphics queues; there is no benefit in having numerous VkQueues of the graphics queue family.

In Daxa, you use the device directly instead of choosing to create and then using a VkQueue for graphics command submission. Example of using the device for submitting commands:

```cpp
daxa::Device device = ...;
daxa::ExecutableCommandList cmd_list = ...;
daxa::BinarySemaphore sema = ...;
daxa::TimelineSemaphore team = ...;
uint64_t tsema_value = ...;
device.submit_commands({
    .command_lists = std::array{std::move(cmd_list)},
    .wait_binary_semaphores = std::array{sema},
    .signal_timeline_semaphores = std::array{std::pair{tsema, tsema_value}},
});
``` 

This generally reduces boilerplate and conveniently merges the VkDevice and VkQeueue concepts.

> Note that Daxa will support async compute and async transfer queues in the future. As no one has requested these up to this point, Daxa does not implement them.

Aside from submitting commands, daxa::Device is also responsible for presenting frames to a swapchain:

```cpp
daxa::Device device = ...;
daxa::Swapchain swapchain = ...;
daxa::BinarySemaphore sema = ...;
device.present_frame({
    .wait_binary_semaphores = std::array{sema},
    .swapchain = swapchain,
});
```

## Collecting Garbage

As mentioned in the lifetime section, Daxa defers the destruction of objects until collecting garbage.

```cpp
daxa::Device device = ...;
device.collect_garbage();
```

This is an essential function. It scans through the array of queued object destructions; it checks if the GPU caught up to the point on the timeline where the resource was "destroyed." If the GPU catches up, they get destroyed for real; if not, they stay in the queue. It is advised to call it once per frame at the end of the render loop.

> Note that collecting garbage is called automatically on device destruction; do not worry about potential leaking here!

This function has another essential caveat! It "locks" the lifetimes of SROs. This means that you may **not** record commands for this device in parallel to calling collect garbage. This is not a limitation in typical applications.

> Note: If fully async command recording is vital, I have plans for "software" command recorders that will work fully parallel with collect garbage, among other features.

Daxa will not error if you try to record commands at the same time as collecting garbage. Instead, it will simply block the thread. Each existing daxa::CommandRecorder holds a read lock on resources, and calling collect garbage will acquire an exclusive write lock.

This is necessary to ensure the efficiency of validation in command recording. Consider this case of Daxa's internals when recording a command:

```cpp
auto daxa_cmd_copy_buffer_to_buffer(
    daxa_CommandRecorder self, 
    daxa_BufferCopyInfo const * info) 
-> daxa_Result
{
    ...
    // We check the validity of the IDs here.
    _DAXA_CHECK_AND_REMEMBER_IDS(self, info->src_buffer, info->dst_buffer)            
    // After we checked the IDs, another thread could destroy the used buffers
    // and call collect garbage, fully deleting the data for the buffers internally.
    // This would cause the IDs to dangle; we would read garbage past here.
    auto vk_buffer_copy = std::bit_cast<VkBufferCopy>(info->src_offset);    
    vkCmdCopyBuffer(
        self->current_command_data.vk_cmd_buffer,
        // We read the internal data of the buffers here.
        self->device->slot(info->src_buffer).vk_buffer,                                 
        self->device->slot(info->dst_buffer).vk_buffer,
        1,
        vk_buffer_copy);
    return DAXA_RESULT_SUCCESS;
}
```

By locking the lifetimes on daxa::CommandRecorder creation and unlocking when they get destroyed, we only need to perform two atomic operations per command recorder instead of per recorded command.

## Metadata

The device stores all kinds of queriable metadata of the VkPhysicalDevice as well as itself and its SROs.

These can all be queried with device member functions. Here are some examples

```cpp
daxa::BufferId buffer = ...;
daxa::Device device = ...;
daxa::DeviceInfo const & device_info = device.info();
daxa::BufferInfo const & buffer_info = device.info_buffer(buffer);
daxa::DeviceProperties const & properties = device.properties();
```
