# libplacebo 面试问答

这些问题用于检验候选人是否真正理解 libplacebo 的项目边界、渲染数据合同、GPU 后端差异和播放器工程问题。

源码快照：

- 本机路径：`D:/github/libplacebo`
- Git describe：`v7.351.0-145-g1dcaea8b-dirty`
- Commit：`1dcaea8b601aa969ffd5bfa70088957ce3eaa273`
- 文档日期：2026-06-08

## 架构

**Q：libplacebo 在播放器里负责什么，不负责什么？**

A：负责 GPU 图像处理和渲染：plane 采样、缩放、色彩转换、HDR tone mapping、gamut mapping、dither、shader/pass 调度和 backend 输出。不负责 demux、decode、packet 管理和 A/V sync。源码入口是 `src/renderer.c:3493` `pl_render_image()`，底层 GPU 抽象在 `src/include/libplacebo/gpu.h:233` `struct pl_gpu_t`。

**Q：`pl_renderer`、`pl_dispatch`、`pl_gpu` 的关系是什么？**

A：`pl_renderer` 是高层视频渲染编排；`pl_dispatch` 把 shader 组合变成 pass 并缓存；`pl_gpu` 是后端无关资源和 pass 执行抽象。对应入口：`src/renderer.c:107`、`src/dispatch.c:1199`、`src/gpu.c:1109`。

## 数据合同

**Q：为什么只传 GPU texture 不够？**

A：texture 只有像素存储，不描述 YUV/RGB、range、transfer、primaries、HDR metadata、crop 和采样位置。libplacebo 需要 `pl_frame`、`pl_color_repr` 和 `pl_color_space` 才能正确渲染。结构定义在 `src/include/libplacebo/renderer.h:528`、`src/include/libplacebo/colorspace.h:154`、`src/include/libplacebo/colorspace.h:462`。

**Q：FFmpeg `AVFrame` 如何接入 libplacebo？**

A：可用 utils 层把 AVFrame 映射成 `pl_frame`。普通 CPU frame 可能走 `pl_upload_plane()` 上传；硬件 frame 会尝试 backend 相关导入，失败后 fallback。入口包括 `src/include/libplacebo/utils/libav_internal.h:733` `pl_frame_from_avframe()`、`:1273` `pl_map_avframe_ex()`、`src/utils/upload.c:225` `pl_upload_plane()`。

## 色彩和 HDR

**Q：HDR 画面发灰，你会怎么排查？**

A：先查 decoder/AVFrame 是否有正确 color fields 和 side data，再查 `pl_color_space` 和 `pl_color_repr` 映射，再查 tone mapping 参数，最后查 swapchain 是否以 HDR 色彩空间输出。入口：`src/include/libplacebo/utils/libav_internal.h:390`、`src/colorspace.c:514`、`src/renderer.c:2151`、`src/d3d11/swapchain.c:291` 或 `src/vulkan/swapchain.c:298`。

**Q：tone mapping 和 gamut mapping 有什么区别？**

A：tone mapping 处理亮度动态范围，gamut mapping 处理色域边界。前者入口 `src/tone_mapping.c:751` `pl_tone_map_functions[]`，后者入口 `src/gamut_mapping.c:979` `pl_gamut_map_functions[]`。

## GPU 后端

**Q：为什么 Vulkan 能 zero-copy，不代表 D3D11/OpenGL 也能？**

A：zero-copy 依赖外部纹理/内存导入、同步对象、format caps 和后端 API 兼容。每个后端实现不同：Vulkan `src/vulkan/gpu_tex.c:1256` `pl_vulkan_wrap()`，D3D11 `src/d3d11/gpu_tex.c:350` `pl_d3d11_wrap()`，OpenGL `src/opengl/gpu_tex.c:581` `pl_opengl_wrap()`。

**Q：如何判断后端是否真的可用？**

A：不能只看头文件。要看 Meson 选项、是否编译 stubs、运行时 create 是否成功、`pl_gpu` caps 和 swapchain 创建是否成功。Vulkan/D3D11/OpenGL 构建入口分别是 `src/vulkan/meson.build:1`、`src/d3d11/meson.build:1`、`src/opengl/meson.build:1`。

## 工程判断

**Q：首帧卡顿可能来自哪里？**

A：shader 编译、pass cache miss、纹理上传、GPU 初始化、swapchain 创建都可能。应拆分 `pl_pass_create()`、`pl_dispatch_finish()`、`pl_upload_plane()`、`pl_pass_run()` 和 present 时间。源码入口：`src/gpu.c:1025`、`src/dispatch.c:1199`、`src/utils/upload.c:225`、`src/gpu.c:1109`。

**Q：`pl_render_image()` 返回成功，为什么用户仍然看到黑屏？**

A：可能 render 到了错误/过期 target，swapchain submit/present 失败，窗口 backbuffer resize 后未重取，或 OS/display HDR 模式切换导致输出不可见。应继续检查 `pl_swapchain_submit_frame()`、`pl_swapchain_swap_buffers()` 和 backend swapchain 状态。入口：`src/swapchain.c:82`、`src/swapchain.c:88`、`src/vulkan/swapchain.c:1032`。

