## Description

Different device criteria.

## Values

| Value | Description |
| --- | --- |
| NONE |  |
| BUFFER_DEVICE_ADDRESS_CAPTURE_REPLAY_BIT | |
| CONSERVATIVE_RASTERIZATION | Whether this device supports a new rasterization mode called conservative rasterization. [There are two modes of conservative rasterization; overestimation and underestimation.](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_conservative_rasterization.html) |
| MESH_SHADER | Whether the device supports mesh shaders |
| SHADER_ATOMIC64 | Whether the device allows 64-bit atomic operations on (un-)signed integers |
| IMAGE_ATOMIC64 | Whether the device allows 64-bit atomic operations on (un-)signed integer images |
| VK_MEMORY_MODEL | The Vulkan Memory Model formally defines how to synchronize memory accesses to the same memory locations performed by multiple shader invocations |
| RAY_TRACING | Whether the device has ray tracing capabilities |
