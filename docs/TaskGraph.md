## TaskGraph

As Vulkan and Daxa require manual synchronization, using Daxa and Vulkan can become quite complex and error-prone.

A common way to abstract and improve synchronization with low-level APIs is using a RenderGraph. Daxa provides a render graph called TaskGraph.

With TaskGraph, you can create task resource handles and names for the resources you have in your program. You can then list a series of tasks.
Each task contains a list of used resources and a callback to the operations the task should perform.

A core idea of TaskGraph (and other render graphs) is that you record a high-level description of a series of operations and execute these operations later. In TaskGraph, you record tasks, "complete" (compile), and later run them. The callbacks in each task are called during execution.

This "two-phase" design allows render graphs to optimize the operations, unlike how a compiler would optimize a program before execution. It also allows render graphs to determine optimal synchronization automatically based on the declared resource used in each task.
In addition, task graphs are reusable. You can, for example, record your main render loop as a task graph and let the task graph optimize the tasks only once and then reuse the optimized execution plan every frame.
All in all, this allows for automatically optimized, low CPU cost synchronization generation.

Overview of the workflow for the task graph:

* Create tasks
* Create task resources
* Add tasks to graph
* Complete task graph
* Execute task graph
* (optional) Repeatedly reassign resources to task resources
* (optional) Repeatedly execute task graph

## Task Resources

When constructing a task graph, it's essential not to use the real resource IDs used in execution but virtual representatives at record time. This is the simple reason that the task graph is reusable between executions. Making the reusability viable is only possible when resources can change between executions. The graph takes virtual resources, TaskImage and TaskBuffer. ImageId and BufferIds can be assigned to these TaskImages and Taskbuffers and changed between executions of task graphs.

### Task Resource Views

Referring to only a part of an image or buffer is convenient. For example, to specify specific mip levels in an image in a mip map generator.

For this purpose, Daxa has TaskImageViews. A TaskImageView, similarly to an ImageView, contains a slice of the TaskImage, specifying the subresource.

All Tasks take in views instead of the resources themselves. Resources implicitly cast to the views but have the explicit conversion function `.view()`. Views also have a `.view()` function to create a new view from an existing one.

## Task

The core part of any render graph is the nodes in the graph. In the case of Daxa, these nodes are called tasks.

A task is a unit of work. It mainly consists of the used task resources (akin to parameters in a function) and a callback (akin to a function body). One could think of a task as a meta-function inside the graph.

To construct a task graph, you define a set of task resources (virtual representatives for the actual resources when recording the graph) and a list of tasks using those resources.

Each time you add a task, it is like a function call where you pass the virtual task resources into the task's uses. TaskGraph will use the input tasks and resources to analyze and generate optimal execution order and synchronization for a given graph.

Notably, the graph works in two phases: the recording and the execution. The callbacks of tasks are only ever called in the execution of the graph, not the recording.

There are two ways to declare a task. You can declare tasks inline, directly inside the add_task function:

```cpp
using namespace daxa::task_resource_uses; // For ImageTransfer(Read|Write)
daxa::TaskImage src = ...;
daxa::TaskImage dst = ...;
int blur_width = ...;
graph.add_task({
  .uses = {
    ImageTransferRead<>{src},
    ImageTransferWrite<>{dst},
  },
  .task = [=](daxa::TaskInterface ti)
  {
    auto cmd = ti.get_recorder();
    copy_image_to_image(ti.uses[src].image(), ti.uses[dst].image(), blur_width);
  },
  .name = "example task",
});
```

This is convenient for smaller tasks or quick additions that don't necessarily need shaders.

And the general way, the persistent declaration:

```cpp
struct MyTask
{
  // The 'uses' field will be statically reflected by Daxa. 
  struct Uses {
    ImageTransferRead<> src;
    ImageTransferWrite<> dst;
  } uses;  
  // The 'name' will be statically reflected by Daxa. 
  static constexpr std::string_view name = "example task"; 
  int blur_width = {};
  void callback(daxa::TaskInterface ti)
  {
    auto cmd = ti.get_recorder();
    copy_image_to_image(uses.src.image(), uses.dst.image(), blur_width);
  }
};
```

Daxa uses limited static reflection to analyze task structs with templates and concepts. The Uses field must be called uses, and the struct type needs to be called Uses.
Optionally, Daxa can also reflect the name field to give the task a name.

### Task Resource Uses

Resource use is declared with either a TaskImageUse or a TaskBufferUse. Daxa predefines a set of shortened names for these uses under the namespace `daxa::task_resource_uses`. It is advised to use these.

A declared resource use describes a pipeline stage, the access (read and/or write), and, for images, optionally, an image view type.
Uses may be predeclared for persistent task structs or listed in a vector for inline tasks.

When a task is added to the graph, a resource must be assigned to each use of the added task.
These are immediately assigned for inline tasks as listed in the uses. For persistent resources, the uses struct must be instantiated, then all resources are assigned to the uses and passed to the add task function.

Inside the tasks callback, the uses provide an interface to access the underlying resource at execution time and metadata about it.
For inline tasks, the uses can be retrieved with the task interface:

```cpp
  .task = [=](daxa::TaskInterface ti)
  {
    ImageTransferRead<> const & img_use = ti.uses[src];
  },
```

While with persistent tasks, the uses are stored in the `Uses` struct and can be directly accessed in the callback:

```cpp
  void callback(daxa::TaskInterface ti)
  {
    ImageTransferRead<> const & img_use = uses.src;
  },
```

ImageUse provides a runtime interface:

```cpp
auto access()             const -> TaskImageAccess;
auto view_type()          const -> ImageViewType;
auto image(u32 index = 0) const -> ImageId;
auto view(u32 index = 0)  const -> ImageViewId;
auto layout()             const -> ImageLayout;
```

If the use's slice and image view type do not fit the default view, Daxa will create and cache image views that match the image view type and a slice of the use.
As the layout can change between tasks at any time, it is hard to know what layout the image slice of the use is currently in.
Daxa allows you to query the image layout via the interface. The image layout is guaranteed to stay the same for the given use-slice for the task's duration on the GPU timeline. 

BufferUse's have a similar interface:

```cpp
auto access() const -> TaskBufferAccess;
auto buffer(usize index = 0) const -> BufferId;
```

With this interface, you can query all necessary information about the images and buffers inside the callbacks with a similar interface for inline and persistently declared tasks.

### TaskInterface

The task interface is the primary way to interact with the task graph from within a task callback.

It provides access to task uses, command lists, a linear memory allocator, and a few other graph-related resources:

```cpp

```

### TaskHead

The resource uses are duplicated in C++ and the shaders/shared files. 
This is because in C++, there is a declaration of a struct of uses, and in shader/shared files, there is a declaration of the IDs/pointers to those used resources.
The IDs/pointers to the resources must also be manually written to some shared struct and pushed to the GPU.

Task heads improve this workflow. They are a unified declaration of uses and a shared struct of idIDss/pointers. The task graph can also analyze these and reduce the boilerplate of uploading IDs.

Task heads are declared within shaders or shared files. These list the resource uses with macros. They hold the necessary information for both the C++ and shader side of each resource use. 
* In shaders, it will declare a struct of IDs/pointers for the given uses. 
* In C++, it declares a struct that is an opaque blob of bytes; this blob is the same size as the shader struct. This is useful for declaring shared structs.
* In C++, it also declares an associated `Uses` struct within the task head struct. This is usable directly as used for a task.

Example:

```c
// within the shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead)
DAXA_TH_IMAGE_ID( COMPUTE_SHADER_READ,  daxa_BufferPtr(daxa_u32), src_buffer)
DAXA_TH_BUFFER_ID(COMPUTE_SHADER_WRITE, REGULAR_2D,               dst_image)
DAXA_DECL_TASK_HEAD_END
```

It is translated to the following in glsl:

```c
struct MyTaskHead
{
    daxa_BufferPtr(daxa_u32)  src_buffer;
    daxa_ImageViewId          dst_buffer;
};
```
It is translated to the following in C++:

```c++
struct MyTaskHead
{
    static inline constexpr std::string_view NAME = "MyTaskHead";
    struct Uses
    {
        daxa::TaskBufferUse<daxa::TaskBufferAccess::COMPUTE_SHADER_READ> src_buffer;
        daxa::TaskImageUse<
            daxa::TaskImageAccess::COMPUTE_SHADER_WRITE, 
            daxa::ImageViewType::REGULAR_2D
        > dst_image;
    };
    // daxa::detail::task_head_shader_blob_size<Uses>()> performs simple 
    /// static reflection on the uses to determine the size of this array.
    // As this struct should be abi compatible, the array is made up 
    // of u64's to match the alignment of ids and ptrs in shader files.
    std::array<
        daxa::u64, 
        daxa::detail::task_head_shader_blob_size<Uses>()
    > shader_byte_blob = {};
};
```

The example task head above is now usable like this:

```c
// within shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead)
DAXA_TH_IMAGE_ID( COMPUTE_SHADER_READ,  daxa_BufferPtr(daxa_u32), src_buffer)
DAXA_TH_BUFFER_ID(COMPUTE_SHADER_WRITE, REGULAR_2D,               dst_image)
DAXA_DECL_TASK_HEAD_END

struct MyPushStruct
{
    daxa_u32vec2 size;
    daxa_u32 settings_bitfield;
    // Head struct can be used within shared structs:
    MyTaskHead head;
};
```

```c++
// within c++
task_graph.add_task({          
    // Head contains Uses struct usable within inline AND static tasks:  
    .uses = daxa::generic_uses_cast(TestShaderTaskHead::Uses{
        .src_buffer = src_view,
        .dst_image = dst_view,
    }),
    .task = [](daxa::TaskInterface ti)
    {
        auto rec = ti.get_command_recorder();
        rec.set_pipeline(...);
        auto push = MyPushStruct{
            .size = ...,
            .settings_bitfield = ...,
        };
        // TaskInterface has function to copy the head shader struct to any memory:
        ti.copy_task_head_to(&push.head);
        // Alternatively, allocate and write the head shader struct to a buffer section:
        // This might be nessecary when the head shader struct is too large.
        // auto alloc = ti.allocate_task_head().value();
        rec.push_constant(push);
    },
    // The string-ified name of the head is also immediately available:
    .name = TestShaderTaskHead::NAME,
});

// Alternatively, the head can be used to declare a static task:
struct MyTask
{
    TestShaderTaskHead::Uses uses;
    static inline constexpr std::string_view NAME = TestShaderTaskHead::NAME;
    daxa_u32vec2 size;
    daxa_u32 settings_bitfield;
    std::shared_ptr<daxa::ComputePipeline> pipeline;
    void callback(daxa::TaskInterface ti)
    {
        auto rec = ti.get_command_recorder();
        rec.set_pipeline(*pipeline);
        auto push = MyPushStruct{
            .size = size
            .settings_bitfield = settings_bitfield,
        };
        // TaskInterface has function to copy the head shader struct to any memory:
        ti.copy_task_head_to(&push.head);
        rec.push_constant(push);
        // Alternatively, allocate and write the head shader struct to a buffer section:
        // This might be necessary when the head shader struct is too large.
        [[maybe_unused]] auto alloc = ti.allocate_task_head().value();
        // The uses work just as manually declared uses:
        [[maybe_unused]] daxa::BufferId = ti.uses.src_buffer.buffer();
        [[maybe_unused]] daxa::ImageViewId = ti.uses.dst_image.view();
    }
};
```

Due to C++ language and preprocessor limitations it's not easily possible to also declare a real struct of ids/ptrs within C++. The head in C++ is just a byteblob, so the ids/ptrs can not directly be set by name. The task interface has a quick way to write the ids/ptrs to some memory location like above. 

#### Specifics

There are multiple ways to declare how a resource is used within the shader:

```c
DAXA_TH_IMAGE_NO_SHADER(    TASK_ACCESS,            NAME)
DAXA_TH_IMAGE_ID(           TASK_ACCESS, VIEW_TYPE, NAME)
DAXA_TH_IMAGE_ID_ARRAY(     TASK_ACCESS, VIEW_TYPE, NAME, SIZE)
DAXA_TH_BUFFER_NO_SHADER(   TASK_ACCESS,            NAME)
DAXA_TH_BUFFER_ID(          TASK_ACCESS,            NAME)
DAXA_TH_BUFFER_PTR(         TASK_ACCESS, PTR_TYPE,  NAME)
DAXA_TH_BUFFER_ID_ARRAY(    TASK_ACCESS,            NAME, SIZE)
DAXA_TH_BUFFER_PTR_ARRAY(   TASK_ACCESS, PTR_TYPE,  NAME, SIZE)
```

* `_ID` postfix delcares that the resource is represented as id in the head shader struct.
* `_PTR` postfix declares the resource is represented as a pointer in the head shader struct.
* `_ARRAY` postfix declares the resource is represented as an array of ids/ptrs. Each array slot is matching a runtime resource within the runtime array of images/buffers of the TaskImage/TaskBuffer.
* `_NO_SHADER` postfix declares that the resource is not accessable within the shader at all. This can be useful for example when declaring a use of `COLOR_ATTACHMENT` in rasterization.

### Task Graph Valid Use Rules

- A task may use the same image multiple times, as long as the TaskImagView's slices don't overlap.
- A task may only ever have one use of a TaskBuffer
- All task uses must have a valid TaskResource or TaskResourceView assigned to them when adding a task.
- All task resources must have valid image and buffer IDs assigned to them on execution.
