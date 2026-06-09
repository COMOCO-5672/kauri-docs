# PortAudio Project Framework

源码快照：

- 本机路径：`D:/github/portaudio`
- Git describe：`v19.7.0-RC2-177-gcf218ed`
- Commit：`cf218ed8e3085ac3731106d3636c3c6396ec2d82`
- 文档日期：2026-06-09

## 先回答边界问题

了解 PortAudio 以后，不能说“完全知道跨平台音频渲染的全部功能”。更准确的说法是：你会理解跨平台音频 I/O 抽象中最核心的一层，包括设备枚举、Host API 选择、stream 生命周期、callback/blocking 两种数据交付方式、采样格式转换、通道交错处理、延迟和 underflow/overflow 风险。

它不负责媒体播放器里更上层的音频渲染策略：解码、重采样质量策略、混音图、音效链、音画同步主时钟、蓝牙/HDMI 路由策略、系统音量策略、音频焦点、环绕/透传策略等。这些通常在 FFmpeg、mpv、OBS、WebRTC 或应用自己的 audio engine 中完成。

> [!IMPORTANT]
> PortAudio 的价值是把不同平台的音频设备 API 收敛成统一 C API。它是跨平台音频输出/输入基础设施，不是完整播放器音频渲染引擎。

## 文档

- [解码后 PCM 音频渲染送帧](1.audio_render.md)
- [整体架构](architecture.md)
- [Host API 与 stream 调度](host-api-and-streaming.md)
- [工程问题手册](engineering-playbook.md)
- [缺陷与风险清单](gaps-and-risks.md)
- [面试问答](interview-qa.md)

## 阅读顺序

```mermaid
flowchart LR
    A["index.md\n先看边界"]
    B["1.audio_render.md\n解码后怎么送帧"]
    C["architecture.md\n公共 API 和分层"]
    D["host-api-and-streaming.md\n后端与数据流"]
    E["engineering-playbook.md\n真实问题排查"]
    F["gaps-and-risks.md\n风险分类"]
    G["interview-qa.md\n复盘与面试"]

    A --> B --> C --> D --> E --> F --> G
```

## 快速定位

- 公共 C API：`include/portaudio.h`。
- 前端分发：`src/common/pa_front.c`。
- Host API 合同：`src/common/pa_hostapi.h`。
- Stream vtable 和 opaque stream 表示：`src/common/pa_stream.h`、`src/common/pa_stream.c`。
- Buffer processor：`src/common/pa_process.h`、`src/common/pa_process.c`。
- 格式转换和 dithering：`src/common/pa_converters.c`、`src/common/pa_dither.c`。
- 无锁 ring buffer：`src/common/pa_ringbuffer.h`、`src/common/pa_ringbuffer.c`。
- Windows Host API 表：`src/os/win/pa_win_hostapis.c`。
- Unix/macOS Host API 表：`src/os/unix/pa_unix_hostapis.c`。
- 主要后端：`src/hostapi/wasapi/`、`src/hostapi/alsa/`、`src/hostapi/coreaudio/`、`src/hostapi/jack/`、`src/hostapi/pulseaudio/`、`src/hostapi/wmme/`、`src/hostapi/dsound/`。

## 图示索引

- 解码后 PCM 到设备的数据流：见 `1.audio_render.md`。
- 5.1 声道重排和 Windows channel mask：见 `1.audio_render.md`。
- 整体架构图：见 `architecture.md`。
- `Pa_OpenStream()` 主流程图：见 `architecture.md`。
- Host API 注册和设备索引图：见 `host-api-and-streaming.md`。
- callback/blocking 数据路径图：见 `host-api-and-streaming.md`。
- 工程排查决策树：见 `engineering-playbook.md`。
- 风险分类图：见 `gaps-and-risks.md`。
