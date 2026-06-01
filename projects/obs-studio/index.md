# OBS Studio Project Framework

源码快照：

- 本机路径：`D:/github/obs-studio`
- Git describe：`32.0.3-1-gea9e4ca06`
- Commit：`ea9e4ca06ee0004132fc0a4dbcdb142198a95018`
- 文档日期：2026-06-01

## 文档

- [整体架构](architecture.md)
- [平台采集：视频与音频](capture.md)
- [音频混音与多音轨](audio-mixing.md)
- [编码、硬编与推流](encoding-and-streaming.md)
- [工程问题手册](engineering-playbook.md)
- [面试问答](interview-qa.md)

## 图示索引

- OBS 分层与插件架构图：见 `architecture.md`。
- 启动、模块注册与渲染主循环图：见 `architecture.md`。
- 平台视频采集矩阵图：见 `capture.md`。
- 音频采集入口图：见 `capture.md`。
- 多音轨混音链路图：见 `audio-mixing.md`。
- 输出、编码、交错、RTMP 推流图：见 `encoding-and-streaming.md`。
- 硬编后端关系图：见 `encoding-and-streaming.md`。
- 工程问题定位图：见 `engineering-playbook.md`。

## 快速定位

- Core：`libobs/obs.c`、`libobs/obs-source.c`、`libobs/obs-video.c`、`libobs/obs-audio.c`、`libobs/obs-encoder.c`、`libobs/obs-output.c`。
- Source API：`libobs/obs-source.h`。
- Encoder API：`libobs/obs-encoder.h`。
- Output API：`libobs/obs-output.h`。
- Windows 采集：`plugins/win-capture`、`plugins/win-dshow`、`plugins/win-wasapi`。
- macOS 采集：`plugins/mac-avcapture`、`plugins/mac-capture`、`plugins/mac-syphon`。
- Linux 采集：`plugins/linux-pipewire`、`plugins/linux-v4l2`、`plugins/linux-capture`、`plugins/linux-pulseaudio`、`plugins/linux-alsa`。
- 推流输出：`plugins/obs-outputs/rtmp-stream.c`、`plugins/rtmp-services`。
- 硬编：`plugins/obs-nvenc`、`plugins/obs-ffmpeg/texture-amf.cpp`、`plugins/obs-ffmpeg/obs-ffmpeg-vaapi.c`、`plugins/obs-qsv11`、`plugins/mac-videotoolbox/encoder.c`。
