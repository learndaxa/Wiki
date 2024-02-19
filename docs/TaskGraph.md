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

A task is a unit of work. It might be a single compute dispatch or multiple dispatches/render passes/raytracing dispatches. What limits the size of a task is resource dependencies that require synchronization.

Synchronization is only inserted *between* tasks. If dispatch A writes an image and dispatch B needs to read the finished content, both dispatches *must* be within different tasks, so task graph is able to synchronize.

A Task consists of four parts:

1. A description of how graph resources are used, the so called "Attachments".
2. A task resource view for each attachment, telling the graph which resource belongs to which attachment.
3. User data, such as a pointer to some context, pipeline pointer and general parameters for the task. 
4. The callback, describing how the work should be recorded for the task.

Notably, the graph works in two phases: the recording and the execution. The callbacks of tasks are only ever called in the execution of the graph, not the recording.

There are two ways to declare a task. You can declare tasks inline, directly inside the add_task function:

```cpp
daxa::TaskImage src = ...;
daxa::TaskImage dst = ...;
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

And the general way, the persistent declaration:

```cpp
// Must declare the number of attachments (here 2).
struct MyTask : daxa::PartialTask<2, "example task">
{
  // Inherited field type from daxa::PartialTask<>.
  // Must be declared in child task to make designated initializer work.
  AttachmentViews views = {};
  // Macros declared within daxa/task_graph.types
  // Each Macro declares an attachment.
  DAXA_TH_IMAGE(TRANSFER_READ, REGULAR_2D, src)
  DAXA_TH_IMAGE(TRANSFER_WRITE, REGULAR_2D, dst)
  int blur_width = {};
  void callback(daxa::TaskInterface ti)
  {
    copy_image_to_image(ti.recorder, ti.get(src).ids[0], ti.get(dst).ids[0], blur_width);
  }
};
...
graph.add_task(MyTask{
  .views = std::array{
    daxa::attachment_view(MyTask::src, src),
    daxa::attachment_view(MyTask::dst, dst),
  },
  .blur_width = x,
});
```

### Task Attachments

Attachments describe a list of used graph resources that might require synchronization between tasks.

> Note: Any resource that is readonly for the execution of the task, like textures, do not need to be mentioned in the attachments.

Each attachment consists of:
* a `task resource access` (either `TaskBufferAccess` or `TaskImageAccess`),
* a description of how the resource is meant to be used in a shader,
* an attachment index

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
* views assigned to attachments
* runtime daxa resource ids
* runtime daxa resource view ids (these are created by the graph based on the attachment view type)
* image layout

Aside from attachment information the interface also provides:
* a command recorder (automatically reused by the graph)
* a transfer memory allocator (super fast per execution linear allocator for mapped gpu memory)
* attachment shader data (generated from the list of attachments, can be send to shader)

### TaskHead

When using shaders of any kind, one must transport the image and buffer ids to the shader via a push constant.

This means declaring a struct passed from cpu to gpu containing the runtime ids for each attachment view.

Task graph can partially automate/ deduplicate code here with task heads.

A task head is a partial task declaration that is valid within .inl files. Its made up of macros that are translated to the correct language at preprocessing time.

The partial task declaration can later be inherited within c++ to form a full task.

An example:

```c
// within the shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead, 2)
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
struct MyTaskHead : daxa::PartialTask<2, "MyTaskHead">
{
    static inline const daxa::TaskBufferAttachmentIndex src_buffer = 
        add_attachment(daxa::TaskBufferAttachment{             
        .name = "src_buffer",                                     
        .access = daxa::TaskBufferAccess::COMPUTE_SHADER_READ,     
        .shader_array_size = 1,
    });
    static inline const daxa::TaskImageAttachmentIndex dst_image = 
        add_attachment(daxa::TaskImageAttachment{             
            .name = "dst_image",                                    
            .access = daxa::TaskImageAccess::COMPUTE_SHADER_WRITE,     
            .view_type = daxa::ImageViewType::REGULAR_2D, 
            .shader_array_size = 1,
    });
};
```

The task graph is getting exactly the information it needs to create the gpu side struct and fill it with the correct data.

This data is given via the task interface in attachment_shader_data and can be directly put into a push constant or buffer.

Extended example:

```c
// within shared file
DAXA_DECL_TASK_HEAD_BEGIN(MyTaskHead, 2)
DAXA_TH_IMAGE_ID( COMPUTE_SHADER_READ,  daxa_BufferPtr(daxa_u32), src_buffer)
DAXA_TH_BUFFER_ID(COMPUTE_SHADER_WRITE, REGULAR_2D,               dst_image)
DAXA_DECL_TASK_HEAD_END

struct MyPushStruct
{
    daxa_u32vec2 size;
    daxa_u32 settings_bitfield;
    // This blob will become nothing in c++ and the head struct in glsl.
    DAXA_TH_BLOB(MyTaskHead,head)
};
```

```c++

struct MyTask : MyTaskHead
{
    AttachmentViews views = {};
    ...
    void callback(daxa::TaskInterface ti)
    {
        auto rec = ti.get_command_recorder();
        ti.recorder.set_pipeline(...);
        ti.recorder.push_constant(MyPushStruct{
            .size = ...,
            .settings_bitfield = ...,
        });
        // The graph automatically fills the attachment shader data.
        ti.recorder.push_constant_vptr({
            .data = ti.attachment_shader_data.data(),
            .size = ti.attachment_shader_data.size(),
            .offset = sizeof(MyPushStruct),
        });
        ti.dispatch(...);
    }
};

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
