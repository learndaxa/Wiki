## Getting Started

Before you can build Daxa, you need to first walk through the [dependency installation](../01_Installing_dependencies.md) step mentioned [here](../01_Installing_dependencies.md).

## Building Daxa

### Windows

```batch
cmake --preset=cl-x86_64-windows-msvc
cmake --build --preset=cl-x86_64-windows-msvc-debug
```

### Linux

```shell
cmake --preset=gcc-x86_64-linux-gnu
cmake --build --preset=gcc-x86_64-linux-gnu-debug
```

## Running a sample

### Windows

```batch
./build/cl-x86_64-windows-msvc/tests/Debug/daxa_test_2_daxa_api_5_swapchain
```

### Linux

```shell
./build/gcc-x86_64-linux-gnu/tests/Debug/daxa_test_2_daxa_api_5_swapchain
```

## Custom Validation

> Note:  This step is not necessary and only applicable to Daxa maintainers!!!

```git clone https://github.com/KhronosGroup/Vulkan-ValidationLayers```
You must build this repo (Debug is fine, you get symbols)

Open up Vulkan Configurator and add a new layer profile
![Screenshot 2022-10-02 110620](https://user-images.githubusercontent.com/28205981/193466792-96e243a4-ee97-440e-8617-b01fce8af100.png)

Add a user-defined path
![Screenshot 2022-10-02 110800](https://user-images.githubusercontent.com/28205981/193466859-19dc5cdc-6dce-4a0f-bf67-aabd36a55003.png)

For me, it's at `C:/dev/projects/cpp/Vulkan-ValidationLayers/build/debug-windows/layers`
![Screenshot 2022-10-02 110934](https://user-images.githubusercontent.com/28205981/193466910-7e0c6be9-7eb2-4d99-b60e-2fe5b38b64bb.png)

And then force override the validation layer
![Screenshot 2022-10-02 111055](https://user-images.githubusercontent.com/28205981/193467005-4fa15b24-0f77-4eee-a0b5-0f19e7fb5876.png)

And that should be it!
