## Description

Traditionally, in graphics APIs like OpenGL or earlier versions of DirectX, resources like textures and buffers were bound to specific binding slots; the shaders had to reference these resources by their slot or binding point. This approach has limitations and could lead to performance bottlenecks, especially when dealing with many resources, as the GPU needs to switch between bindings frequently. Aside from performance, these bindings can be cumbersome and error-prone to use.

New APIs like Metal, Vulkan, and Dx12 still expose some form of binding points with a more direct descriptor management. This descriptor management can become much work for the user and hardware. It can also cause many hard-to-debug issues due to the complexity of descriptor management. 

Daxa's bindless approach eliminates these limitations by allowing shaders to access resources directly without needing explicit binding points. Instead of binding resources to specific slots, Daxa bindless resources are given unique handles that shaders can use to access the resources directly. This approach provides several advantages:

## Advantages

1. Improved performance: With bindless resources, shaders can access resources more efficiently, reducing the overhead associated with frequent binding and unbinding operations.
2. Simpler code: No management of descriptor pools, set-layouts, set-allocation, set-writes, set-binding points, or sync on set-allocations.
3. Flexibility: Bindless resources make working with dynamic and large datasets easier, as shaders can access any resource resources by handle.
4. Less error-prone: Compared to typical Vulkan and Dx12 descriptor management systems, Daxa's bindless has such a small API surface that misuse and errors are less likely.
