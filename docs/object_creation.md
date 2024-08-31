---
title: Object Creation
description: Documentation on the creation of objects in Daxa
slug: object-creation
editUrl: https://github.com/learndaxa/Wiki/edit/main/docs/object_creation.md
---

## Initialization

For nearly all functions, Daxa uses structs as parameters. They follow the naming convention of `<Thing>Info`.

There are several significant advantages of struct parameters in conjunction with c++20 designated initialization:

- Default function parameters
- Out-of-order default function parameters
- Named function parameters

Here is an example of the creation of a `daxa::Instance`:

```cpp
daxa::Instance instance = daxa::create_instance(daxa::InstanceInfo{
    .app_name = "example instance",
});
```

InstanceInfo looks like this:

```cpp
struct InstanceInfo
{
    InstanceFlags flags =
        InstanceFlagBits::DEBUG_UTILS |
        InstanceFlagBits::PARENT_MUST_OUTLIVE_CHILD;
    SmallString engine_name = "daxa";
    SmallString app_name = "daxa app";
};
```

As you can see, all fields have default values. In the example above, we only initialize app_name. This reduces the boilerplate of initialization. Daxa also infers as much as possible for Vulkan object creation, making the info structs smaller and initialization easier.

Also, every object in Daxa has a `name` field in its info struct. These names are provided to Vulkan via the debug utils extension. This allows the driver, the Vulkan validation layers, and tooling like RenderDoc to give each resource its given name in debugging and profiling. Daxa also uses these names internally for error messages.
