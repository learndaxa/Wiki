---
layout: ../layouts/WikiLayout.astro
title: TaskGraph
description: TaskGraph
link: https://github.com/learndaxa/Wiki/blob/main/docs/taskgraph.md
---

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

- Create tasks
- Create task resources
- Add tasks to graph
- Complete task graph
- Execute task graph
- (optional) Repeatedly reassign resources to task resources
- (optional) Repeatedly execute task graph

## Task Resources

When constructing a task graph, it's essential not to use the real resource IDs used in execution but virtual representatives at record time. This is the simple reason that the task graph is reusable between executions. Making the reusability viable is only possible when resources can change between executions. The graph takes virtual resources, TaskImage and TaskBuffer. ImageId and BufferIds can be assigned to these TaskImages and Taskbuffers and changed between executions of task graphs.

### Task Resource Views

Referring to only a part of an image or buffer is convenient. For example, to specify specific mip levels in an image in a mip map generator.

For this purpose, Daxa has TaskImageViews. A TaskImageView, similarly to an ImageView, contains a slice of the TaskImage, specifying the subresource.

All Tasks take in views instead of the resources themselves. Resources implicitly cast to the views but have the explicit conversion function `.view()`. Views also have a `.view()` function to create a new view from an existing one.

## Task

The core part of any render graph is the nodes in the graph. In the case of Daxa, these nodes are called tasks.

A task is a unit of work. It might be a single compute dispatch or multiple dispatches/render passes/raytracing dispatches. What limits the size of a task is resource dependencies that require synchronization.

Synchronization is only inserted _between_ tasks. If dispatch A writes an image and dispatch B needs to read the finished content, both dispatches _must_ be within different tasks, so task graph is able to synchronize.

A Task consists of four parts:

1. A description of how graph resources are used, the so called "Attachments".
2. A task resource view for each attachment, telling the graph which resource belongs to which attachment.
3. User data, such as a pointer to some context, pipeline pointer and general parameters for the task.
4. The callback, describing how the work should be recorded for the task.

Notably, the graph works in two phases: the recording and the execution. The callbacks of tasks are only ever called in the execution of the graph, not the recording.

There are two ways to declare a task. You can declare tasks inline, directly inside the add_task function:

```cpp
daxa::TaskImageView src = ...;
daxa::TaskImageView dst = ...;
int blur_width = ...;
graph.add_task({
  .attachments = {
    daxa::inl_attachment(daxa::TaskImageAccess::TRANSFER_READ, src),
    daxa::inl_attachment(daxa::TaskImageAccess::TRANSFER_WRITE, dst),
  },
  .task = [=](daxa::TaskInterface ti)
  {
    copy_image_to_image(ti.recorder, ti.get(src).ids[0], ti.get(dst).ids[0], blur_width);
  },
  .name = "example task",
});
```

This is convenient for smaller tasks or quick additions that don't necessarily need shaders.

The other way to declare tasks (using "task heads") is shown later.

### Task Attachments

Attachments describe a list of used graph resources that might require synchronization between tasks.

> Note: Any resource that is readonly for the execution of the task, like textures, do not need to be mentioned in the attachments.

Each attachment consists of:

- a `task resource access` (either `TaskBufferAccess` or `TaskImageAccess`),
- a description of how the resource is meant to be used in a shader,
- an attachment index

For persistent tasks this is obvious, take `DAXA_TH_IMAGE` as an example:

`DAXA_TH_IMAGE(TaskImageAccess, ImageViewType, TaskImageAttachmentIndexName)`.

TaskGraph will use all this information to generate optimal synchronization and ordering of tasks, based on the attachments and assigned resource views.

Inline tasks omit some of these and set them do default values. When listing an inline attachment, one also directly assigns the view to the attachment as well.

### TaskInterface

The interface provides functions to query information about the graph, attachments and task itself.

For example to get the runtime information for a given attachment the interface has the `get` function.

It takes a resource view or an attachment index directly.

It returns a `TaskAttachmentInfo` (`TaskBufferAttachmentInfo` for buffers and `TaskImageAttachmentInfo` for images), this struct contains all data about the attachment given on construction as well as runtime data used by the graph.

This includes:

- views assigned to attachments
- runtime daxa resource ids
- runtime daxa resource view ids (these are created by the graph based on the attachment view type)
- image layout

Aside from attachment information the interface also provides:

- a command recorder (automatically reused by the graph)
- a transfer memory allocator (super fast per execution linear allocator for mapped gpu memory)
- attachment shader data (generated from the list of attachments, can be send to shader)

### TaskHead

When using shader resources like buffers and images, one must transport the image id or buffer pointer to the shader. In traditional apis one would bind buffers and images to an index but in daxa these need to be in a struct that is either stored inside another buffer or directly within a push constant.

With a little help, TaskGraph can declare AND fill a shader side struct containing ids and pointers for all resources mentioned in the list of attachments for that task.

To make task graph able to do that we need to use TaskHeads.

A task head is a partial task declaration that is valid within .inl files. Its made up of macros that are translated to the correct language at preprocessing time.

An example of a task head:

```c
// within the shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead)
DAXA_TH_IMAGE_ID( COMPUTE_SHADER_READ,  daxa_BufferPtr(daxa_u32), src_buffer)
DAXA_TH_BUFFER_ID(COMPUTE_SHADER_WRITE, REGULAR_2D,               dst_image)
DAXA_DECL_TASK_HEAD_END
```

This task head declaration will translate to the following glsl shader struct:

```c
struct MyTaskHead
{
    daxa_BufferPtr(daxa_u32)  src_buffer;
    daxa_ImageViewId          dst_buffer;
};
```

Or the following Slang-HLSL:

```c
struct MyTaskHead
{
    daxa::u32*         src_buffer;
    daxa::ImageViewId  dst_buffer;
};
```

In c++ this macro declares a namespace containing a few constexpr static variables.
In the following code i omittied some code as it is hard to read/understand on the spot:

```c++
namespace MyTaskHead
{
    /* TEMPALTE MAGIC */

    // Number of declared attachments:
    static inline constexpr daxa::usize ATTACHMENT_COUNT = {/* TEMPLATE MAGIC */};

    // Attachment meta information:
    static inline constexpr auto ATTACHMENTS = {/* TEMPLATE MAGIC */};

    // Short alias for attachment meta information:
    static inline constexpr auto const & AT = ATTACHMENTS;

    // Shader byte blob with the exact size and alignment of the equivalent shader struct:
    struct alignas(daxa::get_asb_alignment(AT)) AttachmentShaderBlob
    {
        std::array<daxa::u8, daxa::get_asb_size(AT)> value = {};
    };

    // Partially declared task, already defining some functions,
    // also getting some fields into the task structs namespace:
    struct Task : public daxa::IPartialTask
    {
        using AttachmentViews = daxa::AttachmentViews<ATTACHMENT_COUNT>;
        static constexpr AttachmentsStruct<ATTACHMENT_COUNT> const & AT = ATTACHMENTS;
        static constexpr daxa::usize ATTACH_COUNT = ATTACHMENT_COUNT;
        static auto name() -> std::string_view { return std::string_view{NAME}; }
        static auto attachments() -> std::span<daxa::TaskAttachment const>
        {
            return AT.attachment_decl_array;
        }
        static auto attachment_shader_blob_size() -> daxa::u32
        {
            return sizeof(daxa::get_asb_size(AT));
        };
    };
}

```

The partial task in this head namespace can be inherited from to form a full task.

The ATTACHMENTS constant contains all information on the attachments declared in the head.
That information is used within task graph to fill the shader struct with the proper data on execution.

This constant is also used to declare a byte array struct (AttachmentShaderBlob) as a placeholder for the shader struct.
This placeholder can be used to declare shared structs containing the shader shared struct.

This is necessary as with the current limits of macros, we can NOT generate the shader struct as it is in c++ as well. In c++ we only have the placeholder (AttachmentShaderBlob).

Extended example:

```c
// within shared file

DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead)
  // buffer attachment:   cpu side daxa::TaskBufferAccess:   shader side pointer type:       buffer name:
  DAXA_TH_BUFFER_PTR(     COMPUTE_SHADER_READ,               daxa_BufferPtr(daxa_u32),       src_buffer)
  // image attachment:    cpu side daxa::TaskImageAccess:    cpu side daxa::ImageViewType:   image name:
  DAXA_TH_IMAGE_ID(       COMPUTE_SHADER_WRITE,              REGULAR_2D,                     dst_image)
DAXA_DECL_TASK_HEAD_END

// This push constant is shared in shader and c++!
struct MyPushStruct
{
    daxa_u32vec2 size;
    daxa_u32 settings_bitfield;
    // The head field is an aligned byte array in c++ and the attachment struct in shader:
    MyTaskHead::AttachmentShaderBlob attachments;
};
```

```c++
// within c++ file
#include "shared.inl"

// Inherited from the partial task declared by the task head:
struct MyTask : MyTaskHead::Task
{
    // In order to allow for designated struct init in c++,
    // the views field can NOT be part of the partially declared task,
    // it must be declared here!
    AttachmentViews views = {};
    ...
    void callback(daxa::TaskInterface ti)
    {
        auto rec = ti.get_command_recorder();
        ti.recorder.set_pipeline(...);
        ti.recorder.push_constant(MyPushStruct{
            .size = ...,
            .settings_bitfield = ...,
            // Daxa declares convenience assignment operators.
            // You can directly use the graph generated byte blob (ti.attachment_shader_blob)
            // to the push constant byte blob:
            .attachments = ti.attachment_shader_blob,
        });
        ti.dispatch(...);
    }
};
```

Example usage of the above task:

```c++

daxa::ImageViewId some_img_view = ...;
daxa::BufferViewId some_buf_view = ...;

task_graph.add_task(MyTask{
  .views = {
    MyTaskHead::AT.src_buffer | some_img_view,
    MyTaskHead::AT.dst_image | some_img_view,
  },
  .other_stuff = ...,
});
```

The ATTACHMENTS or AT constants declared within the task head contain all metadata about the attachments.
But they also contain named indices for each attachment!

In the above code these named indices are used to refer to the attachments.
You can refer to any attachment with `HEAD_NAME::AT.attachment_name`.

These indices can also be used to access information of attachments within the task callback:

```c++
void callback(daxa::TaskInterface ti)
{
    // The daxa::TaskInterface::get() function is defined to work on
    // TaskView's as well as the indices directly:
    daxa::BufferId id = ti.get(AT.buffer_name).ids[0];

    // The attachment infos contain anything you might need:

    daxa::TaskBufferAttachmentInfo const & attach_info = ti.get(AT.buffer_name);
    char const * name               = attach_info.name;
    TaskBufferAccess access         = attach_info.access;
    u8 shader_array_size            = attach_info.shader_array_size;
    bool shader_as_address          = attach_info.shader_as_address;
    TaskBufferView view             = attach_info.view;
    TaskBufferView translated_view  = attach_info.translated_view;
    std::span<BufferId const> ids   = attach_info.ids;

    daxa::TaskImageAttachmentInfo const & img_attach_info = ti.get(AT.image_name);
    char const * name                     = img_attach_info.name;
    TaskImageAccess access                = img_attach_info.access;
    u8 shader_array_size                  = img_attach_info.shader_array_size;
    bool shader_as_index                  = img_attach_info.shader_as_index;
    TaskImageView view                    = img_attach_info.view;
    TaskImageView translated_view         = img_attach_info.translated_view;
    std::span<ImageId const> ids          = img_attach_info.ids;
    // These view ids are auto generated by the task graph,
    // based on the VIEW_TYPE and the task image view slice:
    std::span<ImageViewId const> view_ids = img_attach_info.view_ids;
}
```

### TaskHead Attachment Declarations

There are multiple ways to declare how a resource is used within the shader:

```c
DAXA_TH_IMAGE_NO_SHADER(    TASK_ACCESS,            NAME)
DAXA_TH_IMAGE_ID(           TASK_ACCESS, VIEW_TYPE, NAME)
DAXA_TH_IMAGE_INDEX(        TASK_ACCESS, VIEW_TYPE, NAME)
DAXA_TH_IMAGE_ID_ARRAY(     TASK_ACCESS, VIEW_TYPE, NAME, SIZE)
DAXA_TH_IMAGE_ID_MIP_ARRAY( TASK_ACCESS, VIEW_TYPE, NAME, SIZE)
DAXA_TH_BUFFER_NO_SHADER(   TASK_ACCESS,            NAME)
DAXA_TH_BUFFER_ID(          TASK_ACCESS,            NAME)
DAXA_TH_BUFFER_PTR(         TASK_ACCESS, PTR_TYPE,  NAME)
DAXA_TH_BUFFER_ID_ARRAY(    TASK_ACCESS,            NAME, SIZE)
DAXA_TH_BUFFER_PTR_ARRAY(   TASK_ACCESS, PTR_TYPE,  NAME, SIZE)
DAXA_TH_TLAS_PTR(           TASK_ACCESS,            NAME)
DAXA_TH_BLAS(               TASK_ACCESS,            NAME)
```

> Note: Some permutations are missing here. BLAS for example has no \_ID, \_INDEX or \_PTR version. This is intentional, as some resources can not be used in certain ways inside shaders.

- `PTR_TYPE` here refers to the shader pointer type of the buffer.
- `VIEW_TYPE` here refers to a daxa::ImageViewType. Task graph will create image views fitting exactly this view type AND the task image views slice.
- `NAME` here is used for the shader struct field names as well as the c++ side ATTACHMENT constants resource index names.
- `SIZE` always refers to the size of the array.

### There are multiple suffix modifiers for each resource type:

- `_ID` postfix delcares that the resource is represented as id in the head shader struct.
- `_INDEX` is similar to `_ID`, but instead of storing the full 64 bit id, it only stores the 32 bit index of the resource. This can save considerable push constant size.
- `_PTR` postfix declares the resource is represented as a pointer in the head shader struct.
- `_ARRAY` postfix declares the resource is represented as an array of ids/ptrs. Each array slot is matching a runtime resource within the runtime array of images/buffers of the TaskImage/TaskBuffer.
- `_MIP_ARRAY` is similar to the `_ARRAY` suffix. The difference the array is filled with generated image views for each mip of the FIRST runtime image of that task image view.
- `_NO_SHADER` postfix declares that the resource is not accessable within the shader at all. This can be useful for example when declaring a use of `COLOR_ATTACHMENT` in rasterization.

### There are some additional valid usage rules:

- A task may use the same image multiple times, as long as the TaskImagView's slices don't overlap.
- A task may only ever have one use of a TaskBuffer
- All task uses must have a valid TaskResource or TaskResourceView assigned to them when adding a task.
- All task resources must have valid image and buffer IDs assigned to them on execution.
