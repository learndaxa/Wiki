---
title: TaskGraph
description: TaskGraph
slug: taskgraph
editUrl: https://github.com/learndaxa/Wiki/edit/main/docs/taskgraph.md
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
    copy_image_to_image(ti.recorder, ti.id(src), ti.id(dst), blur_width);
  },
  .name = "example task",
});
```

> NOTE: There is a third defaulted parameter to inl_attachment, taking in the VIEW_TYPE for the image.
>       When filling this VIEW_TYPE parameter, task graph will create an image view that exactly fits
>       the dimensions of the attachments view slice.
>       When this parameter is defaulted, daxa will fill the image view id with 0.
>       How to access these tg generated image views is shown later.

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

The get function alone can make the code verbose. The TaskInterface provides many helper functions such as:
* `id`
* `view`
* `info`
* `device_address`

Aside from attachment information the interface also provides:

- a command recorder (automatically reused by the graph)
- a transfer memory allocator (super fast per execution linear allocator for mapped gpu memory)
- attachment shader data (generated from the list of attachments, can be send to shader)
- task metadata such as the name and index

### TaskHead

When using shader resources like buffers and images, one must transport the image id or buffer pointer to the shader. In traditional apis one would bind buffers and images to an index but in daxa these need to be in a struct that is either stored inside another buffer or directly within a push constant.

This means that in traditional apis you must list the attachments many times:
1. once in shaders, either as indices/pointers in a push constant OR direct bindings
2. once in the attachments of the task
3. when assigning the indices/bindings for the api
4. once when assigning task buffer/task image views to the attachments

Daxa can help you a lot here by reducing the reduncancy with task heads. Task heads allow you to declare a struct in shader containing all indices/pointers to resources AS WELL AS the attachments for a task in one go! With task heads you only need to:
1. list resource in attachment
2. assign view to attachment

Thats it. Daxa will do all the other logic for you.

But how do task heads work?

Essentially task head declarations consist of a set of macros that are valid in shaders as well as c/c++.
In each language the macros have different definitions. 
The task head declaration either describes a struct with indices/pointers in the shader OR
a namespace containing constexpr metadata about the attachments and their use in the shader.
The metadata is enoug to properly fill the shader struct in the task graph internals.

An example of a task head:

```c
// within the shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead)
DAXA_TH_IMAGE_ID( COMPUTE_SHADER_READ,  daxa_BufferPtr(daxa_u32), src_buffer)
DAXA_TH_BUFFER_ID(COMPUTE_SHADER_WRITE, REGULAR_2D,               dst_image)
DAXA_DECL_TASK_HEAD_END
```

> NOTE: COMPUTE_SHADER_READ is from the enum daxa::TaskBufferAccess

> NOTE: For each of these access enum values, there is also a shortened version, example: CS_READ


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

    // Generated Types:
    static inline constexpr auto ATTACHMENTS_T = {/* TEMPLATE MAGIC */};
    static inline constexpr auto VIEWS_T = {/* TEMPLATE MAGIC */};

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
        using AttachmentViews = ATTACHMENTS_T;
        using Views = VIEWS_T;
        static constexpr AttachmentsStruct<ATTACHMENT_COUNT> const & AT = ATTACHMENTS;
        static constexpr daxa::usize ATTACH_COUNT = ATTACHMENT_COUNT;
        static auto name() -> std::string_view { return std::string_view{NAME}; }
        static auto attachments() -> std::span<daxa::TaskAttachment const>
        {
            return AT._internal.values;
        }
        static auto attachment_shader_blob_size() -> daxa::u32
        {
            return sizeof(daxa::get_asb_size(AT));
        };
    };
}

```

Extended example using a task head:

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
    // Slang:
    MyTaskHead::AttachmentShaderBlob attachments;
    // Glsl:
    DAXA_TH_BLOB(MyTaskHead, attachments);
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
        ti.recorder.set_pipeline(...);
        ti.recorder.push_constant(MyPushStruct{
            .size = ...,
            .settings_bitfield = ...,
            // IMPORTANT:
            // You still HAVE TO assign the attachment shader blob within your push constant/buffer!
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
  .views = MyTask::Views{
    .src_buffer = some_img_view,
    .dst_image = some_img_view,
  },
  .other_stuff = ...,
});
```

Daxa automatically generates a struct type `MyTask::Views` for syntactic suggar when assigning views. Its a struct with one field for each declared attachment. Each field is of the type `TaskAttachmentViewWrapper<T>` which accept task resource views.

### Alternative Use Of TaskHead

Task heads can also be directly used in inline tasks without having to declare a struct inheriting the task:

```c++

daxa::ImageViewId some_img_view = ...;
daxa::BufferViewId some_buf_view = ...;

using MyTask = daxa::InlineTaskWithHead<MyTaskHead::Task>

task_graph.add_task(MyTask{
    .views = MyTask::Views{
      .src_buffer = some_img_view,
      .dst_image = some_img_view,
    },
    .task = [=](daxa::TaskInterface ti) { ... },
});

```


### TaskInterface and Attachment Information

The ATTACHMENTS or AT constants declared within the task head contain all metadata about the attachments.
But they also contain named indices for each attachment!

In the above code these named indices are used to refer to the attachments.
You can refer to any attachment with `HEAD_NAME::AT.attachment_name`.

Note that all these functions also take views directly instead of attachments indices in order to be compatible with inline tasks.

These indices can also be used to access information of attachments within the task callback:

```c++
void callback(daxa::TaskInterface ti)
{
    // There are two ways to get the info for any attachment:
    {
        // daxa::TaskBufferAttachmentIndex index:
        [[maybe_unused]] daxa::TaskBufferAttachmentInfo const & buffer0_attachment0 = ti.get(AT.src_buffer);
        // daxa::TaskBufferView assigned to the buffer attachment:
        [[maybe_unused]] daxa::TaskBufferAttachmentInfo const & buffer0_attachment1 = ti.get(buffer0_attachment0.view);
    }
    // The Buffer Attachment info contents:
    {
        // Information retrieved from convenience functions:
        [[maybe_unused]] daxa::BufferId id_ = ti.id(AT.src_buffer, /*optional*/0);
        [[maybe_unused]] daxa::DeviceAddress address_ = ti.device_address(AT.src_buffer, /*optional*/0).value();
        [[maybe_unused]] std::byte* host_address = ti.buffer_host_address(AT.src_buffer).value();
        [[maybe_unused]] daxa::BufferInfo info_ = ti.info(AT.src_buffer, /*optional*/0).value();

        // Information retrieved from the .get meta function:
        [[maybe_unused]] std::span<daxa::BufferId const> ids = ti.get(AT.src_buffer).ids;
        [[maybe_unused]] daxa::BufferId id = ti.get(AT.src_buffer).ids[0];
        [[maybe_unused]] char const * name = ti.get(AT.src_buffer).name;
        [[maybe_unused]] daxa::TaskBufferAccess access = ti.get(AT.src_buffer).access;
        [[maybe_unused]] u8 shader_array_size = ti.get(AT.src_buffer).shader_array_size;
        [[maybe_unused]] bool shader_as_address = ti.get(AT.src_buffer).shader_as_address;
        [[maybe_unused]] daxa::TaskBufferView view = ti.get(AT.src_buffer).view;
    }
    // The Image Attachment info contents:
    {
        // Information retrieved from convenience functions:
        [[maybe_unused]] daxa::ImageId id_ = ti.id(AT.dst_image, /*optional*/0);
        [[maybe_unused]] daxa::ImageViewId view_ = ti.view(AT.dst_image, /*optional*/0);
        [[maybe_unused]] daxa::ImageViewInfo info_ = ti.info(AT.dst_image, /*optional*/0).value();

        // Information retrieved from the .get meta function:
        [[maybe_unused]] char const * name = ti.get(AT.dst_image).name;
        [[maybe_unused]] daxa::TaskImageAccess access = ti.get(AT.dst_image).access;
        [[maybe_unused]] daxa::ImageViewType view_type = ti.get(AT.dst_image).view_type;
        [[maybe_unused]] u8 shader_array_size = ti.get(AT.dst_image).shader_array_size;
        [[maybe_unused]] daxa::TaskHeadImageArrayType shader_array_type = ti.get(AT.dst_image).shader_array_type;
        [[maybe_unused]] daxa::ImageLayout layout = ti.get(AT.dst_image).layout;
        [[maybe_unused]] daxa::TaskImageView view = ti.get(AT.dst_image).view;
        [[maybe_unused]] std::span<daxa::ImageId const> ids = ti.get(AT.dst_image).ids;

        /// WARNING: ImageViews are only filled out for attachments that set the VIEW_TYPE!
        ///          If you use an inline attachment and dont specify the VIEW_TYPE like for a transfer op,
        ///          there will be an empty daxa::ImageViewId{} put into this array for the attachment!
        [[maybe_unused]] std::span<daxa::ImageViewId const> view_ids = ti.get(AT.dst_image).view_ids;
    }
    // The attachment infos are also provided, directly via a span:
    for ([[maybe_unused]] daxa::TaskAttachmentInfo const & attach : ti.attachment_infos)
    {
    }
    // The tasks shader side struct of ids and addresses is automatically filled and serialized to a blob:
    [[maybe_unused]] auto generated_blob = ti.attachment_shader_blob;
    // The head also declared an aligned struct with the right size as a dummy on the c++ side.
    // This can be used to declare shader/c++ shared structs containing this blob :
    [[maybe_unused]] TestTaskHead::AttachmentShaderBlob blob = {};
    // The blob also declares a constructor and assignment operator to take in the byte span generated by the taskgraph:
    blob = generated_blob;
    [[maybe_unused]] TestTaskHead::AttachmentShaderBlob blob2{ti.attachment_shader_blob};
}
```

### TaskHead Attachment Declarations

There are multiple ways to declare how a resource is used within the shader:

```c
// CPU only attachments. These are not present in the attachment byte blob:
#define DAXA_TH_IMAGE(TASK_ACCESS, VIEW_TYPE, NAME)
#define DAXA_TH_BUFFER(TASK_ACCESS, NAME)
#define DAXA_TH_BLAS(TASK_ACCESS, NAME)
#define DAXA_TH_TLAS(TASK_ACCESS, NAME)

// _ID Attachments will be represented by the first id.
#define DAXA_TH_IMAGE_ID(TASK_ACCESS, VIEW_TYPE, NAME) 
#define DAXA_TH_BUFFER_ID(TASK_ACCESS, NAME) 
#define DAXA_TH_TLAS_ID(TASK_ACCESS, NAME)

// _INDEX Attachments will be represented by the index of hte first id.
// This is useful for having lots of image attachments.
// Index attachments take only 4 bytes, id attachments need 8 bytes.
#define DAXA_TH_IMAGE_INDEX(TASK_ACCESS, VIEW_TYPE, NAME) 

// _TYPED Attachments will be represented either as a (RW)TextureXId<T> or (RW)TextureXIndex<T>.
// These Typed id/index handles are Slang only.
#define DAXA_TH_IMAGE_TYPED(TASK_ACCESS, TEX_TYPE, NAME)

// _MIP_ARRAY Attachments will be represented as an array of ids/indices where each array element
// views a mip level of the first image in the runtime array.
// This can be useful for mip map generation, as storage image views can only see one mip at a time.
// It is allowed to have an image bound to the attachment that has less mips then the array is in size,
// The remaining array elements will be filled with 0s.
#define DAXA_TH_IMAGE_ID_MIP_ARRAY(TASK_ACCESS, VIEW_TYPE, NAME, SIZE)
#define DAXA_TH_IMAGE_INDEX_MIP_ARRAY(TASK_ACCESS, VIEW_TYPE, NAME, SIZE)
#define DAXA_TH_IMAGE_TYPED_MIP_ARRAY(TASK_ACCESS, TEX_TYPE, NAME, SIZE)

// Ptr Attachments are represented by a device address.
#define DAXA_TH_BUFFER_PTR(TASK_ACCESS, PTR_TYPE, NAME)
#define DAXA_TH_TLAS_PTR(TASK_ACCESS, NAME)

// _MIP_ARRAY Attachments will be represented as an array of ids/indices where each array element
// refers to a RUNTIME resource in the runtime resource array PER resource.
#define DAXA_TH_BUFFER_ID_ARRAY(TASK_ACCESS, NAME, SIZE) 
#define DAXA_TH_BUFFER_PTR_ARRAY(TASK_ACCESS, PTR_TYPE, NAME, SIZE)
#define DAXA_TH_IMAGE_ID_ARRAY(TASK_ACCESS, VIEW_TYPE, NAME, SIZE) 
#define DAXA_TH_IMAGE_INDEX_ARRAY(TASK_ACCESS, VIEW_TYPE, NAME, SIZE) 
#define DAXA_TH_IMAGE_TYPED_ARRAY(TASK_ACCESS, TEX_TYPE, NAME, SIZE)
```

> Note: Some permutations are missing here. BLAS for example has no \_ID, \_INDEX or \_PTR version. This is intentional, as some resources can not be used in certain ways inside shaders.

### There are some additional valid usage rules

- A task may use the same image multiple times, as long as the TaskImagView's slices don't overlap.
- A task may only ever have one use of a TaskBuffer
- All task uses must have a valid TaskResource or TaskResourceView assigned to them when adding a task.
- All task resources must have valid image and buffer IDs assigned to them on execution.
