---
layout: ../layouts/WikiLayout.astro
title: Features
description: Features
link: https://github.com/learndaxa/Wiki/blob/main/docs/features.md
---

## Easy Resource Lifetimes

- Most resources are reference counted in Daxa (Instance, Device, Swapchain, Semaphore, ...) for convenience.
- For shader resources (Buffer, Image, ImageView, Sampler) however, Daxa does not reference count, because :
  - Bindless shader resource IDs/addresses can be stored in buffers and other memory, making Daxa unable to consistently track them.
  - In API calls, they are mentioned with much higher frequency compared to other objects, making their tracking with reference counts much more expensive.
  - In contrast to other objects, it is very common to build other more specialized lifetime management for shader resources (eg. static, cached, streamed, dynamically created, ...).
- The destruction of any resource is deferred until all submits, currently running at the time of the destroy call, are finished. This makes it safe to call destroy on resources in most cases.
- Extensive debug checks for any number of cases with sensible error messages detailing what could have gone wrong.
- Thread safety can optionally be disabled via a preprocessor macro definition.
  - Calls to CommandRecorder from multiple threads are NOT threadsafe.
  - Utils generally are not threadsafe.
  - All Other objects are generally threadsafe to access from multiple threads simultaneously.

## Ergonomic Swapchain

- Swapchain contains its semaphores for timing:
  - acquire semaphores, one for each frame in flight,
  - present semaphores, one per swapchain image,
  - a timeline semaphore, used to limit frames in flight.
- The correct set of semaphores can be queried with member functions, they are changed after every acquire call.
- The swapchain integrates frame in flight limiting, as it is natural for it to handle this.
- Easy-to-use interface for recreating the swapchain.

## Simple Renderpasses

- Daxa does not have the concept of a pre-created renderpass for framebuffers.
- To render, you begin and end a renderpass section directly within a command list.
- This can be done very efficiently, Daxa does not cache or lazy create anything for this behind the user's back, as Vulkan 1.3 has a feature mapping directly to this interface.
- This drastically simplifies rendering and removes a lot of object coupling. Pipelines are completely decoupled of framebuffers or renderpass objects.

## Powerful CommandRecorders

- The vulkan command pool is abstracted. Daxa maintains a pool pool inside each device.
- CommandRecorders make command buffers and command pools easier to use and much safer per design.
- Each recorder is a pool + buffer. A command buffer can be completed at any time, a new command buffer is used with the same pool after completing current commands.
- PipelineBarrier API is made more ergonomic:
  - CommandRecorder does not take arrays of memory barriers and image memory barriers, it takes them individually.
  - As long as only pipeline barrier commands are recorded in sequence, they will be batched into arrays.
  - As soon as a non-pipeline barrier command is recorded, all memory barriers are recorded to the command buffer.
  - This effectively is mostly syntax sugar, but can be quite nice to reduce Vulkan API calls if the barrier insertion is segmented into multiple functions.
- As mentioned above in the lifetime section, CommandRecorders can record a list of shader resources to destroy after the command list has finished execution on the GPU.
- CommandRecorder changes type in render passes to ensure correct command use by static type safety!
- CommandRecorders have a convenience function to defer the destruction of a shader resource until after the command list has finished execution on the GPU.
  - Very helpful to destroy things like staging or scratch buffers.
  - Necessary as it is not legal to destroy objects that are used in commands that are not yet submitted to the GPU.

## Tiny Pipelines

- Pipelines are much simpler in Daxa compared to normal Vulkan, they are created with parameter structs that have sensible defaults set for all parameters.
- Because of the full bindless interface and full descriptor set and descriptor set layout abstraction, you do not need to specify anything about descriptors at pipeline creation.
- Daxa again does this very efficiently with no on-the-fly creation, caching or similar, all cost is fixed and on creation time.
- As Renderpasses and framebuffers are abstracted, those also do not need to be mentioned, pipelines are decoupled.
- The pipeline manager utility makes pipeline management very simple and convenient.

## Effective use of modern Language features

- Nearly all functions take in structs. Each of these info structs has defaults. This allows for named parameters and out-of-order defaults!

## Safety and Debuggability

- IDs and reference-counted handles are fully threadsafe
- all Vulkan function return values are checked and propagated
- lots of custom checks are performed and propagated
- lots of internal validation to ensure correct Vulkan use
- preventing wrong use by design
- Using type safety for correct command recording
- Integrated object naming (used by tooling validation layers and Daxa itself).

## Other Features

- Automatically managed memory allocations with VMA. Optionally exposed manual management.
- Daxa stores and lets you get the info structs used in object creation. Very useful, as you do not need to store the object's metadata yourself.
- Abstraction of image aspect. In 99% of cases (Daxa has no sparse image support) Daxa can infer the perfect image aspect for an operation, which is why it's abstracted to reduce boilerplate.
- Next to bindless, Daxa also provides simplified uniform buffer bindings, as some hardware can still profit from these greatly.
- FSR 2.1 integration
- ImGUI backend
- Staging memory allocator utility. Provides a fast linear staging memory allocator for small per-frame uploads.
