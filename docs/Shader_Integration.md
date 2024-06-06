---
layout: ../layouts/WikiLayout.astro
title: Shader Integration
description: Shader Integration
link: #
---

## Description

One of Daxa's goals is to make shader development easier.

Daxa achieves this in multiple ways, for example, the bindless integration on the CPU and GPU side. But as a basis for this, Daxa also provides tools for easy code sharing between shaders and C++.

The details on code sharing and general shader integration will be described on this page.

## C++/Shader Code Sharing

The primary motivation behind code sharing is defining things like structs stored in buffers a single time and then seamlessly using them in C++ and shaders. This concept extends to other parts of the API as well. For example, TaskGraph uses lists that can be declared in shared files to reduce code duplication.

Code is shared by creating header files included in C++ and Glsl. Any shared files must include the <daxa/daxa.inl> header. This file contains variations of macros switched based on the compiled language. The code sharing in Daxa relies on the shared Glsl and C++ preprocessor translating Daxa's macros to the correct type per language.

The macros in <daxa/daxa.inl> are mainly primitive data types, Daxa structs, and helper utilities.

An example of a Daxa shared file:

```c
#include <daxa/daxa.inl>

struct MyStruct
{
    daxa_u32vec2 my_vec;
};
```

daxa_u32 in this struct is a macro defined by daxa. This macro is translated to uvec2 in Glsl, uint2 in Hlsl, or daxa::types::f32vec2 in C++.

The complete list of defined data types in Daxa can be found in the `daxa.inl` file.

Daxa always declares the alignment of buffer references to be in a scalar block layout. This layout causes shared structs containing daxa\_ types to have the alignment rules C++ and Glsl. This means that the C++ and Glsl versions of the struct will be compatible. You need not worry about padding or alignment behavior with the usual Glsl format layouts like std140 or std430.

## Shader Constants

For the shared structs to work correctly in other situations, Daxa also provides some utility macros to make it easier to write shaders. Please use these macros instead of making your own Glsl push constants and uniform buffer declarations, as Daxa may change the required layout specifiers later. Not using the Daxa-provided macros may break your code in the future.

As Daxa does not expose descriptor sets, there are only two ways to directly present data to the shader: push constants and uniform buffer bindings.

### Push Constants

`DAXA_DECL_PUSH_CONSTANT(STRUCT, NAME)` takes a struct and declares a push constant with the given name in the global scope.

This can **only** be used within `.glsl` files, as it generates Glsl code.

Example:

```c
struct Push
{
    daxa_u32 field;
};

DAXA_DECL_PUSH_CONSTANT(Push, push)

layout(local_size_x = 1)
void main()
{
    uint value = push.field;
}
```

## Bindless Shader Integration

In order to make bindless work seamlessly within shaders, Daxa needs to provide Glsl and Hlsl headers that define abstractions for shaders to conveniently access the SROs bindlessly.

The headers provided by Daxa include shader types for the SRO ids: `daxa_BufferId`, `daxa_ImageViewId` and `daxa_SamplerId` in Glsl and `daxa::BufferId`, `daxa::ImageViewId` and `daxa::SamplerId` in Hlsl. They also define functions to use these IDs within shaders.
As everything in the provided headers is implemented with either macros or standard Glsl or Hlsl syntax, this all works out of the box without the need for any language extensions or custom compilers.

### Images

All images are created with internal default image views. In fact, their IDs can be trivially converted to an image view ID:

```cpp
daxa::ImageId image_id = device.create_image({...});
daxa::ImageViewId image_view_id = image_id.default_view();
```

This default view covers the full mip and layer dimensions of the image.
This can significantly reduce boilerplate, as for most images, only the default view is necessary for all uses.

> Daxa only supports separate images and samplers, so no combined image samplers. This simplifies the API and allows for more consistent HLSL support.

### Image Access in Glsl

The shader access works by transforming a `daxa_ImageViewId` with or without a `daxa_SamplerId` into a Glsl `texture`, `image`, or `sampler` locally.

Examples of transforming image and sampler IDs into glsl objects locally:

```glsl
#include <daxa/daxa.glsl>
...
daxa_ImageViewId img = ...;
daxa_SamplerId smp = ...;
ivec4 v = texture(daxa_isampler3D(img,smp), vec3(...));

daxa_ImageViewId img2 = ...;
imageStore(daxa_image2D(img2), ivec2(...), vec4(...));

daxa_ImageViewId img3 = ...;
uvec2 size = textureSize(daxa_texture1DArray(img3));
...
```

> Note that they can not be treated as local variables and can only be used IN PLACE of the usage, as shown below. You CAN, however, pass the IDs as value types to functions structs and buffers!

> Daxa default enables many glsl extensions like [GL_EXT_samplerless_texture_functions](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GL_EXT_samplerless_texture_functions.txt). It is worth to check those out as they can be a bit unknown but handy.

### Glsl Annotations For Images

In glsl, it is possible to annotate image variables with custom [qualifiers](<https://www.khronos.org/opengl/wiki/Type_Qualifier_(Glsl)>) and [image formats](<https://www.khronos.org/opengl/wiki/Layout_Qualifier_(Glsl)>). Such annotations can be: `coherent` or `readonly` and `r32ui` or `rgba16f`.
Custom qualifiers are really useful and can provide better performance and more possibilities in some cases. Image formats, on the other hand, are sometimes required by some glsl functions (`imageAtomicOr` for example).

To provide the image accessor macros, Daxa pre-defines image tables without any annotations. These make the access macros such as `daxa_image2D` possible to use.

Pre-defining all possible permutations of qualifiers for all image types would be thousands of LOC, destroying compile times. Because of this, Daxa only pre-defines the tables used in the macros without qualifiers.

To still provide a nice way to gain access to Daxa image views with the benefits of the annotations, Daxa tries to offer the middle ground by allowing the user to declare new accessors for images with annotations when needed.
These custom accessors declare a new table with these annotations. Each user-defined accessor must have a unique `ACCESSOR_NAME`. This name is used to identify the accessor when using it with `daxa_access(ACCESSOR_NAME, image_view_id)`.

```glsl
DAXA_DECL_IMAGE_ACCESSOR(TYPE, ANNOTATIONS, ACCESSOR_NAME) // Declares new accessor.
DAXA_DECL_IMAGE_ACCESSOR_WITH_FORMAT(TYPE, FORMAT, ANNOTATIONS, ACCESSOR_NAME) // Declares new accessor with format
daxa_access(ACCESSOR_NAME, image_view_id) // Uses the accessor by name to convert an image view id to the given glsl type.
```

Example:

```glsl
DAXA_DECL_IMAGE_ACCESSOR(image2D, coherent restrict, RWCoherRestr)
DAXA_DECL_IMAGE_ACCESSOR(iimage2DArray, writeonly restrict, WORestr)
DAXA_DECL_IMAGE_ACCESSOR_WITH_FORMAT(uimage2D, r32ui, , r32uiImage)
...
void main() {
    daxa_ImageViewId img0, img1, img2 = ...;
    vec4 v = imageLoad(daxa_access(3WCoherRestr, img0), ivec2(0,0));
    imageStore(daxa_access(WORestr, img1), ivec2(0,0), 0, ivec4(v));
    imageAtomicOr(daxa_access(r32uiImage, img2), ivec2(0,0), 1 << 31);
}
```

### Image Access in Hlsl

To get access to images in Hlsl, you create a local Hlsl texture object in the shader from the image ID.

Constructing a texture handle in Hlsl is done with macro constructors similar to glsl. These constructors look like this: `daxa_##HLSL_TEXTURE_TYPE(TEX_RET_TYPE, IMAGE_VIEW_ID)`.

> Note: In contrast to glsl, you can treat the returned Hlsl texture handles as local variables.

> Note: Currently, only 4 component return types for texture functions are implemented; this is done to reduce the header bloat. The generated code will be of the same quality.

Example:

```hlsl
#include <daxa/daxa.hlsl>
...
daxa::ImageViewId img = ...;
daxa::SamplerId smp = ...;
// Alternative one: using a macro to construct a texture handle locally. Used in place.
int4 v = daxa_Texture3D(int4, img).Sample(smp, float3(...));
int4 v = t.Sample(smp, float3(...));

daxa::ImageViewId img2 = ...;
daxa_RWTexture2D(float4, img2)[int2(...)] = float4(...);

daxa::ImageViewId img3 = ...;
// Alternative two: As you can treat them as local variables in Hlsl, the following is also possible:
Texture1DArray<float4> t = daxa_Texture1DArray(float4, img3);
uint mips; uint width; uint elements; uint levels;
t.GetDimensions(mips, width, elements, levels);
...
```

### Buffers

Each buffer is created with a buffer device address and optionally a mapped host pointer, as long as the memory requirements allow for it.

The host and device pointers can be retrieved:

```cpp
void* host_ptr                              = device.get_buffer_host_address(buffer_id).value();
daxa::types::DeviceAddress device_ptr = device.get_buffer_device_address(buffer_id).value();
```

### Buffer Access in Glsl

The general way to access buffers in Daxa is via buffer device address and Glsl's [buffer reference](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GLSL_EXT_buffer_reference.txt).

> In order for other features to work correctly, Daxa requires very specific glsl layout specifiers for buffer references. Thus, it is necessary to use Daxas macros for buffer reference declarations!

Daxa provides four ways to declare a new buffer reference:

- `DAXA_DECL_BUFFER_REFERENCE_ALIGN(ALIGNMENT)`: declares head for new buffer reference block with a given alignment
- `DAXA_DECL_BUFFER_REFERENCE`: declares head for new buffer reference block with default alignment (4)
- `DAXA_DECL_BUFFER_PTR_ALIGN(STRUCT, ALIGNMENT)`: declares readonly and read/write buffer pointers to given struct type with given alignment
- `DAXA_DECL_BUFFER_PTR(STRUCT)`: declares readonly and read/write buffer pointers to given struct type with default alignment (4)

Usage examples:

```glsl
DAXA_DECL_BUFFER_REFERENCE MyBufferReference
{
    uint field;
};

struct MyStruct { uint i; };
DAXA_DECL_BUFFER_PTR(MyStruct)

...
void main()
{
    // You can also get the address of a buffer id inside all shaders: daxa_u64 daxa_id_to_address(BUFFER_ID)
    daxa_u64 address = ...;
    MyBufferReference my_ref = MyBufferReference(address);
    my_ref.field = 1;
    daxa_BufferPtr(MyStruct) my_readonly_ptr = daxa_BufferPtr(MyStruct)(address);
    uint read_value = deref(my_readonly_ptr).i;
    daxa_RWBufferPtr(MyStruct) my_readwrite_ptr = daxa_RWBufferPtr(MyStruct)(address);
    deref(my_readwrite_ptr).i = 1;
}
```

In C++, the `daxa_BufferPtr(x)` and `daxa_RWBufferPtr` macros become `daxa::types::DeviceAddress`, so you can put them into structs, push constants and or buffer blocks. `DAXA_DECL_BUFFER_PTR_ALIGN` and `DAXA_DECL_BUFFER_PTR` become blank lines in C++. This makes them usable in shared files.

It is generally recommended to declare structs in shared files and then declare buffer pointers to the structs. Using structs and buffer pointers reduces redundancy and is less error-prone. The pointer-like syntax with structs is also quite convenient in general, as you gain value semantics to the pointee with the `deref(ptr)` macro.

Sometimes, it is necessary to use Glsl annotations/ qualifiers for fields within buffer blocks or to use Glsl features that are not available in C++. For example, the coherent annotation or unbound arrays are not valid in C++ or in Glsl/C++ structs, meaning in order to use those features, one must use a buffer reference instead of a buffer pointer.

> The Daxa buffer ptr types are simply buffer references containing one field named `value` of the given struct type.
> For the `BufferPtr` macro, the field is annotated with `readonly`, while it is not with `RWBufferPtr`.

### Buffer Access in Hlsl

As Hlsl has poor buffer device address support, hence Daxa relies on StructuredBuffer and ByteAddressBuffer for buffers in Hlsl.

These are constructed similarly to texture handles in Hlsl with a construction macro: `daxa_ByteAddressBuffer(BUFFER_ID)`, `daxa_RWByteAddressBuffer(BUFFER_ID)`, `daxa_StructuredBuffer(STRUCT_TYPE, BUFFER_ID)`.

Note that in order to use StructuredBuffer for a given struct type, you must use the `DAXA_DECL_BUFFER_PTR` macro for that struct. This is due to limitations of Hlsl and backward compatibility reasons with glsl.

Example:

```hlsl
#include <daxa.hlsl>
...
struct MyStruct
{
    uint field;
};
DAXA_DECL_BUFFER_PTR(MyStruct)

void main()
{
    daxa::BufferId buffer_id = ...;
    ByteAddressBuffer b = daxa_ByteAddressBuffer(buffer_id);
    b.Store(0, 1);
    uint read_value0 = b.Load<MyStruct>(0).field;
    StructuredBuffer<MyStruct> my_readonly_buffer =
        daxa_StructuredBuffer(MyStruct, buffer_id);
    uint read_value1 = my_readonly_buffer[0].field;
}
```

> Note: Hlsl provides atomic ops for ByteAddressBuffer as well as a sizeof operator for structs.

## Always Enabled SPIR-V/Glsl Extensions

- [GL_KHR_memory_scope_semantics](https://github.com/KhronosGroup/Glsl/blob/master/extensions/khr/GL_KHR_memory_scope_semantics.txt)
  - Removes/replaces coherent as a buffer and image decoration in SpirV and Glsl.
  - Replaces coherent modifiers with more specific memory barriers and atomic op function calls. These new calls specify a more granular memory and execution scope for memory access (for example, perthread/subgroup/workgroup/device).
  - The old coherent was practically ub.
  - New scopes give very granular, well-defined memory and execution scopes that can increase performance.
- [GL_EXT_scalar_block_layout](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GL_EXT_scalar_block_layout.txt)
  - Allows for full parity of C++ and Glsl structs.
  - Allows for the shared file format to contain uniform buffers and structs that have the same memory layout in C++ and Glsl without additional padding
  - much more intuitive and less error-prone compared to old layout specifications
- [GL_EXT_samplerless_texture_functions](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GL_EXT_samplerless_texture_functions.txt):
  - Adds sampler-less functions for texture functions.
  - Vulkan initially had some strange OpenGl'isms for texture functions.
  - This extension adds function overloads for texture functions that don't need or don't even use the sampler in them at all.
- [GL_EXT_shader_image_int64](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GLSL_EXT_shader_image_int64.txt)
  - Daxa requires 64-bit atomics.
  - It is 204, and platforms that don't support this are not worth thinking about (Mainly Intel and Apple)
- [GL_EXT_nonuniform_qualifier](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt)
  - Allows the use of the qualifier `nonuniformEXT(x)`
  - Convenient for diverging access to descriptors within a warp.
- [GL_EXT_shader_explicit_arithmetic_types_int64](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt)
  - Defines fixed-size primitive types like uint32_t
  - Needed to create the well-defined struct code sharing in Daxa shared files.
- [GL_EXT_shader_image_load_formatted](https://github.com/KhronosGroup/OpenGL-Registry/blob/main/extensions/EXT/EXT_shader_image_load_formatted.txt)
  - Enables storage image access with no format specification in the type declaration.
  - Modern desktop GPUs don't need the format annotation at all, as they store the format in the descriptor.
  - Reduces shader bloat generated by the Daxa shader preamble considerably (would be over 10x larger without it).
  - Simplifies storage image access.
- [GL_EXT_buffer_reference](https://github.com/KhronosGroup/Glsl/blob/master/extensions/ext/GLSL_EXT_buffer_reference.txt)
  - Needed to make BufferPtr and RWBufferPtr possible.
  - Allows declaration of buffer references, making it possible to interpret a 64-bit integer as a memory address and then use the reference as a pointer.
