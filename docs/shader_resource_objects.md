---
title: Shader Ressource Objects
description: Shader Ressource Objects
slug: shader-ressource-objects
---

## Id vs. Handle

Two Daxa object categories are shader resource objects (SRO) and regular objects.

Shader resource objects are Daxas buffer, image, image view, and sampler type. Any other object is a regular object.

These two object categories are treated differently, as explained here.

### Regular Object

Most Daxa objects are regular objects.

Regular objects are represented by atomically reference-counted handles because reference counting ...

- ... prevents classes of errors
  - Reduces the need for much internal validation
  - Simplifies validation and internals
  - Makes debugging less frequent and easier
- ... is convenient
  - Removes the need to think about the end of the lifetime
  - Well-known comfortable concept for most programmers

But there are also downsides to reference counting:

- Copying handles has significant overhead. This can lead to unnecessary performance loss for objects that are high in number or often used in typical applications.
- Lifetimes need to be clarified. It can lead to hogging memory and a general uncertainty about when objects get destroyed.

These downsides are **not** a problem for most objects. Specifically for objects that are used in large numbers, these downsides become much more problematic. Because of this, Daxa objects are split into two classes based on their typical number and usage frequency.

An example of a regular object:

```cpp
daxa::Device device = instance.create_device({.name = "example device"});
// All Daxa objects store metadata that can be queried with an info function:
daxa::DeviceInfo const& device_info = device.info();
```

### Shader Resource Object

What makes SROs different from regular objects?

- They are all accessible directly in shaders
- Typically occurs in much larger numbers
- Typically accessed and used in greater frequency
- Having apparent lifetimes has a greater importance

As buffers, images, and samplers typically come in much larger numbers than other objects, the previously mentioned downsides of ref counting become a problem. Buffers and images are also tied to typically large regions of memory, which makes potential memory hogging much worse as well. Because of this, Daxa does **not** reference count SROs! Instead, Daxa gives the user manual lifetime management over these objects with a create-and-destroy function.

SROs are represented by an ID on the user side. When creating, using, and destroying SROs, they are exclusively referred to by their IDs. These IDs are trivially copyable; they all have weak reference lifetime semantics. The IDs work very similarly to how entity IDs work in an ECS. There is no way to use the object without the ID **and** another Daxa object verifying its use. For example, getting information about an object, like the name, **must** be done via a device function taking in the ID.

These IDs are much safer than, for example, a raw pointer to the resource. I won't go into specifics here, but these are some key advantages:

- The user has no way to access an object without the device
- Ability to efficiently verify object access
- Threadsafety and validation, even in the case of misuse
- Close to zero overhead for validation

Another significant advantage of IDs for SROs is that Daxa uses descriptor indexing to access SROs in shaders, and the IDs can, therefore, be used in CPU AND GPU shader code! This simplifies the API even more.

Examples of SROs:

```cpp
daxa::Device device = ...;

daxa::BufferId buffer = device.create_buffer({
    .size = 64,
    .name = "example buffer",
});

// For SROs, the info is returned as a value to prevent race conditions.
//device also stores metadata about its SROs that can be queried:
daxa::BufferInfo buffer_info = device.info_buffer(buffer).value();

//device can tell you if an ID is valid or not:
const bool id_valid = device.is_id_valid(buffer);

daxa::ImageId image = device.create_image({
    .format = daxa::Format::R8G8B8A8_SRGB,
    .size = {1024, 1024, 1},
    .usage = daxa::ImageUsageFlagBits::SHADER_SAMPLED |
        daxa::ImageUsageFlagBits::TRANSFER_DST,
    .name "example texture image",
});

daxa::ImageViewId image_view = device.create_image_view({
    .format = Format::R8G8B8A8_SRGB;
    .image = image;
    .slice = {};
    .name = "example image view";
});

daxa::SamplerId sampler = device.create_image_sampler({});

//device is responsible for destroying SROs:
device.destroy_buffer(buffer);
device.destroy_image(image);
device.destroy_image_view(image_view);
device.destroy_sampler(sampler);
```
