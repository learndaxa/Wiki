## Description

Traditionally, in graphics APIs like OpenGL or earlier versions of DirectX, resources like textures and buffers were bound to specific slots or locations in the GPU's memory, and the shaders (programs running on the GPU) had to reference these resources by their slot or binding point. This approach had limitations and could lead to performance bottlenecks, especially when dealing with many resources, as the GPU needed to switch between bindings frequently.

Daxa's bindless approach eliminates these limitations by allowing shaders to access resources directly without needing explicit binding points. Instead of binding resources to specific slots, Daxa bindless resources are given unique handles or descriptors that shaders can use to access the resources directly. This approach provides several advantages:

## Advantages

1. Improved performance: With bindless resources, shaders can access resources more efficiently, reducing the overhead associated with frequent binding and unbinding operations.
2. Flexibility: Bindless resources make working with dynamic and large datasets easier, as shaders can access resources without worrying about available binding points.
3. Reduced CPU overhead: Vulkan bindless reduces the CPU workload, as there's no need to manage resource bindings explicitly.
4. Better resource utilization: Bindless allows for better resource utilization, as the GPU can dynamically allocate and use resources more efficiently.
