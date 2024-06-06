---
layout: ../layouts/WikiLayout.astro
title: Pipeline Manager
description: Pipeline Manager
link: https://github.com/learndaxa/Wiki/blob/main/docs/pipeline_manager.md
---

## PipelineManager

As Daxa is designed to be a GPU-driven centric API, we provide some code to be used within your shaders. This is in the form of `.inl` files, such as the core `daxa/daxa.inl` and the newer `daxa/utils/task_graph.inl`.

As such, Daxa provides the PipelineManager util, which is meant to be used in the development phase of your app to iterate on your GPU code very quickly.

PipelineManager is mainly designed on top of [Khronos' glslang library](https://github.com/khronosGroup/glslang), providing GLSL to SPIR-V compilation. However, PipelineManager doesn't generate SPIR-V for you to feed to Daxa's Pipeline API. PipelineManager ultimately manages the pipelines for you. It does this because this way, it can do much more for you, such as:

- Hot reloading (with #include dependency tracking)
- #includes files with line-numbered error messages
- Virtual files

### Usage

To use the PipelineManager, include the corresponding util header and construct one with a Daxa device.

```cpp
#include <daxa/utils/pipeline_manager.hpp>

// ...

daxa::PipelineManager pipeline_manager = daxa::PipelineManager({
    .device = device,
    .name = "pipeline_manager",
});
```

Once you have a pipeline manager, you can start making pipelines! The pipeline manager's minimum input is the `.source` field of the `.shader_info` field. Let's create a compute pipeline since those are simpler than raster pipelines, and the additional configuration for raster pipelines is identical to what's necessary in the core Daxa API. So, it's not unique to the PipelineManager.

```cpp
auto compilation_result = pipeline_manager.add_compute_pipeline(/* daxa::ComputePipelineCompileInfo */{
    .shader_info = /* daxa::ShaderCompileInfo */ {.source = /* ... */},
    .name = "compute_pipeline",
});
```

This `.source` field is a variant. It can be a path to a file `daxa::ShaderFile`, a raw string of code `daxa::ShaderCode`, or raw SPIR-V binary `daxa::ShaderByteCode`. Providing a SPIR-V binary negates most of the utility of the PipelineManager but is available if you need it. The other two are extremely useful. First, we'll start by just passing in a string of code.

```cpp
auto compilation_result = pipeline_manager.add_compute_pipeline({
    .shader_info = {.source = daxa::ShaderCode{.string = R"glsl(

        layout(local_size_x = 1, local_size_y = 1, local_size_z = 1) in;
        void main()
        {
        }

    )glsl"}},
    .name = "compute_pipeline",
});
```

This is the simplest compute shader that will compile in GLSL. Once this compiles, `pipeline_manager` will construct a `daxa::ComputePipeline` for you, store it internally, and return a `daxa::Result<std::shared_ptr<daxa::ComputePipeline>>`. This is wrapped in a result since the compilation may fail, which we may want to check.

```cpp
if (compilation_result.is_err())
{
    std::cerr << "Failed to compile the compute_pipeline!\n";
    std::cerr << compilation_result.message() << std::endl;
    return -1;
}
```

Now, we can store our shared pointer in the compute pipeline and use it however we like!

```cpp
std::shared_ptr<daxa::ComputePipeline> compute_pipeline = compilation_result.value();
```

PipelineManager is the container of all these pipelines, so we only give the user a shared pointer to the underlying object. PipelineManager can modify the pipeline without the user needing to change each pipeline explicitly.

> Note: This design decision is mainly due to pipeline manager being a utility designed around developing your application, not for shipping it!

#### Hot Reloading

The most significant feature of PipelineManager is the hot-reloading. When the shader code is changed and saved, it will recompile the pipeline. You can even `#include` headers in your shaders, and when the code in those files is updated, the PipelineManager will automatically recompile the affected pipelines for you.

To use hot-reloading, we could demonstrate this by adding a `#include` to our `ShaderCode` string. Instead, to show them more directly, we will use real or virtual files. We'll review both in this document, but let's start with the actual files. We'll create a simple Daxa project structured like so:

```text
my_daxa_project/
  |- src/
  |  |- main.cpp
  |  |- main.glsl
  |- CMakeLists.txt
  | ...
```

> Note: We'll also be using the `my_daxa_project/` directory as the CWD when launching the application. This is important since all relative paths will be relative to the CWD. To be extra clear, with this CWD, we can address the CMakeLists.txt file by saying `./CMakeLists.txt`.

Now, to use `main.glsl` as our shader source file, we need to give its path to the `ShaderCompileInfo` when adding a new pipeline, so we'll do that instead. Since we want to use a relative path, we'll say `.source = daxa::ShaderFile{"src/main.glsl"}`. If we didn't specify the full relative path, then our error check from earlier would print the following:

```text
Failed to compile the compute_pipeline!
Could not find file: "main.glsl"
```

Alternatively, to provide the entire relative path, we can modify our PipelineManager to use our `src/` folder as a root look-up path for both source files and #includes by filling the `.root_paths` field in the PipelineManager creation info.

```cpp
daxa::PipelineManager pipeline_manager = daxa::PipelineManager({
    .device = device,
    .shader_compile_options = {
        // src is now a root look-up path!
        .root_paths = {
            "src",
        },
    },
    .name = "pipeline_manager",
});

auto compilation_result = pipeline_manager.add_compute_pipeline({
    //So now we can say
    .shader_info = {.source = daxa::ShaderFile{"main.glsl"}},
    .name = "compute_pipeline",
});
```

Now that we have a pipeline built on a file, we can look at the hot-reloading. In our application loop, we need to call `.reload_all()` on our PipelineManager.

```cpp
while (true) {
    // ...

    auto reloaded_result = pipeline_manager.reload_all();

    // ...
}
```

This `.reload_all()` function returns a result variant, which you can use to check the reload result. This function doesn't necessarily do anything except check the timestamps of the files in the tracked dependency graph, so it can be the case that it returns a `daxa::NoPipelineChanged` value.

```cpp
if (auto reload_err = std::get_if<daxa::PipelineReloadError>(&reloaded_result))
    std::cout << "Failed to reload " << reload_err->message << '\n';
if (auto _ = std::get_if<daxa::PipelineReloadSuccess>(&reloaded_result))
    std::cout << "Successfully reloaded!\n";
```

If we were to modify our `main.glsl` shader file while this application was running, the pipeline manager would automatically recompile `compute_pipeline` for us, with no developer intervention. This is extremely useful for iteration times since you can change your shaders as much as you like while the application runs. If the shader fails to compile, then the pipeline will not be modified and thus will continue to use the old _working_ version.

Now is a good time to mention the Daxa shader files, which you can and should #include in your shaders for ease of development. These are in the Daxa include directory, but this can be hard to find when using Daxa as a vcpkg dependency. To remedy this, the Daxa CMake package provides a C++ #define which has the full path to the Daxa include directory: `DAXA_SHADER_INCLUDE_DIR`. We can add this to our `.root_paths` to allow us to `#include` the Daxa headers in our shaders.

```cpp
.root_paths = {
    DAXA_SHADER_INCLUDE_DIR,
    "src",
},
```

Once we have this root path, we can change our `main.glsl` file like this!

```cpp
#include <daxa/daxa.inl>

layout(local_size_x = 1, local_size_y = 1, local_size_z = 1) in;
void main()
{
}
```

More about Daxa's shader integration (how this header is useful) can be found [here](https://github.com/Ipotrick/Daxa/tree/master/wiki/ShaderIntegration.md).

The last thing to mention for PipelineManager is the ability to register virtual files.

You can call `.add_virtual_file()` on a PipelineManager, providing a name and contents.

```cpp
pipeline_manager.add_virtual_file({
    .name = "my_file",
    .contents = R"glsl(
        // ...
    )glsl",
});
```

To update the virtual file's contents, all you need to do is `.add_virtual_file()` with the exact name string.

Here's what it would look like if we had both a virtual file for the main source file (`my_file`) and a virtual include file (`my_include`)!

```cpp
pipeline_manager.add_virtual_file({
    .name = "my_include",
    .contents = R"glsl(
        #pragma once
        #define MY_INCLUDE_DEFINE
    )glsl",
});

pipeline_manager.add_virtual_file({
    .name = "my_file",
    .contents = R"glsl(
        // Here we can
        #include <my_include>

        #ifndef MY_INCLUDE_DEFINE
        #error This should NOT happen
        #endif

        layout(local_size_x = 1, local_size_y = 1, local_size_z = 1) in;
        void main() {
        }
    )glsl",
});

auto compilation_result = pipeline_manager.add_compute_pipeline({
    // Here, we supply the path to the file, but our virtual file look-up
    // matches, and so the virtual file is used instead!
    .shader_info = {.source = daxa::ShaderFile{"my_file"}},
    .name = APPNAME_PREFIX("compute_pipeline"),
});
```
