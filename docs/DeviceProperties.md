## Description

Different device properties.

## Values

| Name | Description | Value type |
| --- | --- | --- |
| vulkan_api_version |  | [daxa::u32](Types.md) |
| driver_version |  | [daxa::u32](Types.md) |
| vendor_id |  | [daxa::u32](Types.md) |
| device_id |  | [daxa::u32](Types.md) |
| device_type |  | daxa::DeviceType |
| device_name |  | [daxa::u8[256U]](Types.md) |
| pipeline_cache_uuid |  | [daxa::u8[16U]](Types.md) |
| limits |  | daxa::DeviceLimits |
| mesh_shading_properties |  | Optional\<MeshShaderDeviceProperties> |
| ray_tracing_properties |  | Optional\<RayTracingPipelineProperties> |
| acceleration_structure_properties |  | Optional\<AccelerationStructureProperties> |
