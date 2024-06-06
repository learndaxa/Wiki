---
layout: ../layouts/WikiLayout.astro
title: Object Lifetimes
description: Documentation on the lifetime of objects in Daxa
link: https://github.com/learndaxa/Wiki/blob/main/docs/object_lifetimes.md
---

## Ids vs. reference counting

Why have two different lifetime systems in Daxa?

In the past Daxa had only reference counted objects. This was a great convenience; reference counting is simple, solves all lifetime tracking issues, and makes use and implementation very easy.

The downsides of reference counting are performance and unclear lifetimes.

It is straightforward to forget about a reference somewhere and start to hog a ton of memory accidentally by never releasing temporary resources.

Daxa tracks all used objects in the command recording for validation. The overhead of reference counting for this is around 5-15%! On top of that, the ref counting can cause performance problems on the user side, too.

Object types that are high in numbers per application make the issues of ref counting much worse.

In opposition to that is using IDs for object references. These have the upside that they are trivial to copy and have proper lifetimes.

The downside is that they require extra checking; this checking is **very** cheap on average and far less than a reference count increment.

Daxa strikes a compromise. "low frequency" objects, in relatively low numbers, have uninteresting lifetimes, are not passed around often, and are reference counted for convenience.
Any object that typically appears in very high frequency (shader resource objects) is managed by IDs.

## Parent-child dependencies

Per default, Daxa will track parent-child dependencies between objects and keep the parents alive until all the children die.

This can be very convenient but also worsens the problem of unclear lifetimes.

So Daxa also has an optional instance flag (`DAXA_INSTANCE_FLAG_PARENT_MUST_OUTLIVE_CHILD`) that forces parents to outlive their children. The children will **not** keep their parents alive but throw an error.

## Deferred destruction - Zombies?

When an object's reference count drops to 0 or gets manually destroyed, it is not immediately destroyed; it is zombified.

A zombie object is no longer usable on the user side. But zombies are still valid in Daxa's internals and on the GPU.

Daxa defers the destruction of zombies until the GPU catches up with the CPU at the time of zombification.

The real object destructions exclusively happen in `Device::collect_garbage`. This function will check all zombies' zombification time points and compare them with the GPU timeline, and if the GPU catches up, it will destroy the zombie.

> Note: the functions `Device::submit_commands` and `Device::present`, as well as any non-completed `CommandRecorder`, hold a shared read lock on object lifetimes. `Device::collect_garbage` will lock the object lifetimes exclusively, meaning it will block until all shared locks get unlocked! Make sure not to have open command lists while calling `Device::collect_garbage.`
