# libplacebo Project Framework

源码快照：

- 本机路径：`D:/github/libplacebo`
- Git describe：`v7.351.0-145-g1dcaea8b-dirty`
- Commit：`1dcaea8b601aa969ffd5bfa70088957ce3eaa273`
- 文档日期：2026-06-08

## 文档

- [Windows D3D 硬解直通 libplacebo 渲染链路](1.libplacebo-d3d-gpu-directx-render.md)
- [整体架构](architecture.md)
- [渲染、色彩与 HDR 管线](rendering-color-hdr.md)
- [GPU 后端与交换链](gpu-backends-and-swapchain.md)
- [工程问题手册](engineering-playbook.md)
- [缺陷与风险清单](gaps-and-risks.md)
- [面试问答](interview-qa.md)

## 阅读顺序

```mermaid
flowchart LR
    A["architecture.md\n先看边界"]
    B["1.libplacebo-d3d-gpu-directx-render.md\n看 Windows 硬解接入"]
    C["rendering-color-hdr.md\n理解渲染数据合同"]
    D["gpu-backends-and-swapchain.md\n理解平台后端"]
    E["engineering-playbook.md\n真实问题排查"]
    F["gaps-and-risks.md\n风险分类"]

    A --> B --> C --> D --> E --> F
```

libplacebo 是一个面向播放器和视频应用的 GPU 渲染库。它不负责 demux、decode、A/V sync，也不直接决定播放器策略；它负责把调用方提供的 `pl_frame`、色彩信息、纹理和目标 surface，转换为一组 GPU pass，完成缩放、色彩转换、HDR tone mapping、gamut mapping、dithering、film grain、deinterlace/custom shader 等渲染工作。

> [!IMPORTANT]
> 对播放器开发来说，libplacebo 的核心价值不是“显示一张纹理”，而是维护从解码输出到显示设备之间的图像数据合同：plane 布局、采样位置、色彩表示、HDR 元数据、目标色域、GPU 后端能力、swapchain 输出能力必须一致。

## 图示索引

- 整体分层架构图：见 `architecture.md`。
- `pl_render_image()` 主流程图：见 `rendering-color-hdr.md`。
- 色彩和 HDR 元数据流：见 `rendering-color-hdr.md`。
- GPU 后端关系图：见 `gpu-backends-and-swapchain.md`。
- swapchain 生命周期图：见 `gpu-backends-and-swapchain.md`。
- Windows D3D11VA/DXVA2 硬解直通渲染链路：见 `1.libplacebo-d3d-gpu-directx-render.md`。
- 风险分类图：见 `gaps-and-risks.md`。

## 快速定位

- 公共 API：`src/include/libplacebo/*.h`。
- 渲染入口：`src/renderer.c`，尤其 `pl_render_image()`、`pl_render_image_mix()`。
- GPU 抽象：`src/include/libplacebo/gpu.h`、`src/gpu.c`。
- shader 构建和 dispatch：`src/shaders.c`、`src/dispatch.c`。
- 色彩和 HDR：`src/colorspace.c`、`src/tone_mapping.c`、`src/gamut_mapping.c`、`src/shaders/colorspace.c`。
- Vulkan 后端：`src/vulkan/*`。
- D3D11 后端：`src/d3d11/*`。
- OpenGL 后端：`src/opengl/*`。
- FFmpeg/DAV1D 辅助映射：`src/include/libplacebo/utils/libav*.h`、`src/include/libplacebo/utils/dav1d*.h`。
- 帧队列和插帧/混帧：`src/utils/frame_queue.c`。
