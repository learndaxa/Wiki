## Description

One of Daxa's goals is to make shader development easier.

Daxa achieves this in multiple ways, for example, the bindless integration on the CPU and GPU side. But as a basis for this Daxa also provides tools for easy code sharing between shaders and C++.

The details on code sharing and general shader integration will be described in this file.

## C++/Shader Code Sharing

The main motivation behind code sharing is defining things like structs stored in buffers a single time and then using them in C++ and shaders seamlessly. This concept extends to other parts of the API as well, for example, TaskGraph uses lists that can be declared in shared files to reduce code duplication as well.

Code is shared by creating header files that get included in C++ and GLSL. Any shared files must include the <daxa/daxa.inl> header. This file contains variations of macros that are switched based on the compiled language. So in essence the code sharing in Daxa relies on the shared GLSL and C++ preprocessor translating Daxa's makros to the correct type per language.

The macros in <daxa/daxa.inl> are mainly primitive data types as well as Daxa structs and helper utilities.

An example of a Daxa shared file:

```c
#include <daxa/daxa.inl>

struct MyStruct
{
    daxa_u32vec2 myvec;
};
```

daxa_u32 in this struct is a macro defined by daxa. This macro is translated to either uvec2 in GLSL, uint2 in Hlsl or daxa::types::f32vec2 in C++.

The full list of defined data types in Daxa can be found in the daxa.inl file.

Daxa always declares the alignment of buffer references to be in a scalar block layout. This layout causes shared structs containing daxa_ types to have the alignment rules C++ and GLSL. This means that the C++ and GLSL versions of the struct will be compatible. You do not need to worry about padding or the behavior of alignment with the usual GLSL format layouts like std140 or std430.

## Shader Constants

For the shared structs to properly work in other situations, Daxa also provides some utility macros to make it easier to write shaders. Please use these macros instead of making your own Glsl push constants and uniform buffer declarations, as Daxa may change the required layout specifiers later. Not using the Daxa-provided macros may break your code in the future.

As Daxa does not expose descriptor sets, there are only two ways to present data to the shader directly: push constants and uniform buffer bindings.

### Push Constants

`DAXA_DECL_PUSH_CONSTANT(STRUCT, NAME)` takes a struct and declares a push constant with the given name in the global scope.

This can ONLY be used within .glsl files, as it generates Glsl code.

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

## Always Enabled SPIR-V/GLSL Extensions

- [GL_KHR_memory_scope_semantics](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_memory_scope_semantics.txt)
  - removes/replaces coherent as a buffer and image decoration in SpirV and Glsl.
  - replaces coherent modifiers with more specific memory barriers and atomic op function calls. These new calls specify a more granular memory and execution scope for memory access (for example perthread/subgroup/workgroup/device).
  - old coherent was practically ub. 
  - new scopes give very granular, well-defined memory and execution scopes that can even increase performance.
- [GL_EXT_scalar_block_layout](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_scalar_block_layout.txt)
  - allows for full parity of C++ and Glsl structs.
  - allows for the shared file format to contain uniform buffers and structs that have the same memory layout in C++ and Glsl without additional padding
  - much more intuitive and less error-prone compared to old layout specifications
- [GL_EXT_samplerless_texture_functions](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_samplerless_texture_functions.txt):
  - adds sampler-less functions for texture functions.
  - Vulkan initially had some strange OpenGl'isms for texture functions.
  - this extension adds function overloads for texture functions that don't need or don't even use the sampler in them at all.
- [GL_EXT_shader_image_int64](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GLSL_EXT_shader_image_int64.txt)
  - Daxa requires 64-bit atomics.
  - it's 2023, platforms that don't support this are not worth thinking about (sorry Intel and Apple)
- [GL_EXT_nonuniform_qualifier](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt)
  - allows the use of the qualifier `nonuniformEXT(x)`
  - convenient for diverging access to descriptors within a warp.
- [GL_EXT_shader_explicit_arithmetic_types_int64](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GL_EXT_nonuniform_qualifier.txt)
  - defines fixed-size primitive types like uint32_t
  - needed to create the well-defined struct code sharing in Daxa shared files.
- [GL_EXT_shader_image_load_formatted](https://github.com/KhronosGroup/OpenGL-Registry/blob/main/extensions/EXT/EXT_shader_image_load_formatted.txt)
  - enables storage image access with no format specification in the type declaration. 
  - modern desktop GPUs don't need the format annotation at all, as they store the format in the descriptor.
  - reduces shader bloat generated by the Daxa shader preamble considerably (would be over 10x larger without it).
  - simplifies storage image access.
- [GL_EXT_buffer_reference](https://github.com/KhronosGroup/GLSL/blob/master/extensions/ext/GLSL_EXT_buffer_reference.txt)
  - needed to make BufferPtr and RWBufferPtr possible.
  - allows declaration of buffer references which makes it possible to interpret a 64-bit integer as a memory address, then use the reference as a pointer to it.
