---
layout: ../layouts/WikiLayout.astro
title: Types
description: Types
link: #
---

## Description

Type aliases used by Daxa. They need to be included using the `<daxa/types.hpp>` header using the C++ API. For simplicity, you can remove the need to use the Daxa namespace prefix when using these values by using the daxa::types namespace: `using namespace daxa::types;`

## Values

| Value         | Description                            | Alias          |
| ------------- | -------------------------------------- | -------------- |
| u8            | 8-bit unsigned integer                 | std::uint8_t   |
| u16           | 16-bit unsigned integer                | std::uint16_t  |
| u32           | 32-bit unsigned integer                | std::uint32_t  |
| u64           | 64-bit unsigned integer                | std::uint64_t  |
| usize         |                                        | std::size_t    |
| b32           |                                        | daxa::u32      |
| i8            | 8-bit signed integer                   | std::int8_t    |
| i16           | 16-bit signed integer                  | std::int16_t   |
| i32           | 32-bit signed integer                  | std::int32_t   |
| i64           | 64-bit signed integer                  | std::int64_t   |
| isize         |                                        | std::ptrdiff_t |
| f32           | 32-bit precision floating point number | float          |
| f64           | 64-bit precision floating point number | double         |
| DeviceAddress | 64-bit unsigned device address         | daxa::u64      |
