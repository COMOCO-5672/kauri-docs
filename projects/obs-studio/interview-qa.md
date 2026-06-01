# OBS Studio Interview Q&A

这些问题按工程面试组织，重点看候选人能否把平台采集、libobs 抽象、音频混音、编码推流和硬编后端连起来。

## 架构

### Q1: OBS 的核心抽象是什么？

A: `libobs` 提供 source、scene、audio、video、encoder、output、service 抽象。插件通过 `obs_source_info`、`obs_encoder_info`、`obs_output_info` 注册能力，前端负责创建和编排。入口看 `libobs/obs-source.h:222`、`libobs/obs-encoder.h:197`、`libobs/obs-output.h:41`。

### Q2: OBS 插件是怎么注册 source/encoder/output 的？

A: 插件在 `obs_module_load()` 中调用 `obs_register_source()`、`obs_register_encoder()`、`obs_register_output()`。注册检查在 `libobs/obs-module.c:956` 和 `:1126`；例如 `plugins/obs-outputs/obs-outputs.c:63` 注册 RTMP output，`plugins/obs-nvenc/nvenc.c:1506` 注册 NVENC。

### Q3: OBS 的视频主循环在哪里？

A: `obs_reset_video()` 初始化视频，`obs_init_video()` 创建 `obs_graphics_thread()`。主循环在 `libobs/obs-video.c:1099` `obs_graphics_thread_loop()`，每帧调用 `output_frames()`，再到 `output_frame()` 和 `render_video()`。

## 视频采集

### Q4: Windows 游戏采集和窗口采集有什么区别？

A: 游戏采集通过 hook 目标进程拿渲染纹理，核心在 `plugins/win-capture/game-capture.c:914` `inject_hook()`、`:1632` `start_capture()`、`:1864` `game_capture_render()`。窗口采集走 BitBlt/WGC 等系统 API，核心在 `plugins/win-capture/window-capture.c:138` `choose_method()`、`:589` `wc_tick()`、`:775` `wc_render()`。

### Q5: Windows 显示器采集有哪些路径？

A: OBS 会在 DXGI Desktop Duplication、WGC 等方法之间选择。入口看 `plugins/win-capture/duplicator-monitor-capture.c:250` `choose_method()` 和 `:851` `duplicator_capture_info`。

### Q6: macOS 摄像头/采集卡怎么进 OBS？

A: `mac-avcapture` 使用 AVFoundation。`plugin-main.m:17` 创建 `OBSAVCapture`，`OBSAVCapture.m:1221` 接收 `CMSampleBuffer`，视频在 `:1400` 调 `obs_source_output_video()`，音频在 `:1464` 调 `obs_source_output_audio()`。

### Q7: Linux Wayland 下屏幕采集为什么主要看 PipeWire？

A: Wayland 不允许传统 X11 风格直接抓屏，OBS 通过 xdg-desktop-portal + PipeWire 获取授权 stream。入口看 `plugins/linux-pipewire/screencast-portal.c:154`、`:197` 和 `plugins/linux-pipewire/pipewire.c:1228`、`:1243`。

### Q8: V4L2 摄像头采集主链路是什么？

A: `plugins/linux-v4l2/v4l2-input.c:956` 创建 mmap buffer，`:971` 创建采集线程，`:160` `v4l2_thread()` 读取 frame，`:270` 调 `obs_source_output_video()`。

## 音频和多音轨

### Q9: OBS 音频采集最终进入哪里？

A: 平台插件构造 `obs_source_audio`，调用 `obs_source_output_audio()`。核心入口是 `libobs/obs-source.c:3964`。平台侧例子：WASAPI `win-wasapi.cpp:1007`、CoreAudio `mac-audio.c:452`、PulseAudio `pulse-input.c:216`、ALSA `alsa-input.c:569`。

### Q10: OBS 多音轨是怎么实现的？

A: OBS 有最多 6 个 audio mix，`MAX_AUDIO_MIXES` 在 `libobs/media-io/audio-io.h:28`。source 通过 mixer mask 决定进哪些 mix，audio encoder 通过 `mixer_idx` 订阅某一路 mix。`obs_encoder.c:351` 用 `audio_output_connect()` 连接。

### Q11: 多音轨没有声音怎么排查？

A: 先查 source mixer mask，再查对应 audio encoder 的 `mixer_idx`，再查 output/muxer 是否支持该轨。关键函数：`obs_source_get_audio_mixers()` 在 `libobs/obs-source.c:4775`，`obs_output_set_audio_encoder()` 在 `libobs/obs-output.c:1075`。

### Q12: 音频监听和推流音频是一回事吗？

A: 不是。监听是本地 monitor 链路，在 `libobs/audio-monitoring` 下；推流/录制音频走 `obs-audio.c` mix 和 encoder/output。重复采集问题可先看 `libobs/obs-audio.c:53` 起对 monitoring device 的处理。

## 编码推流

### Q13: OBS 推流从点击开始到发包的链路是什么？

A: `obs_output_start()` 调 output 插件 start，RTMP 是 `rtmp_stream_start()`，随后 `obs_output_begin_data_capture()` 连接 encoder，encoder 产出 `encoder_packet`，output 交错后在 `rtmp_stream_data()` 发送。入口：`libobs/obs-output.c:396`、`plugins/obs-outputs/rtmp-stream.c:1397`、`libobs/obs-output.c:2758`、`libobs/obs-output.c:2222`、`plugins/obs-outputs/rtmp-stream.c:1652`。

### Q14: OBS 如何把 encoder packet 交给 RTMP？

A: encoder 的 callback 先进入 output，音视频都有时通常走 `interleave_packets()`，然后 output 的 `.encoded_packet` 回调到 `rtmp_stream_data()`，最终 `send_packet()`/`send_packet_ex()` 发送。源码是 `libobs/obs-output.c:2222`、`plugins/obs-outputs/rtmp-stream.c:1652`、`:399`、`:426`。

### Q15: 硬编 texture path 为什么重要？

A: texture path 可以避免 GPU 渲染结果下载到 CPU 再上传/编码。libobs 在 `obs-video-gpu-encode.c:151` 判断 `encode_texture2`，`:158` 调用后端 texture 编码。NVENC、QSV、VAAPI、AMF 都有 texture encoder 变体。

### Q16: NVENC 的关键代码在哪里？

A: `plugins/obs-nvenc/nvenc.c:1391` H.264，`:1412` HEVC，`:1433` AV1。texture encode 在这些 `obs_encoder_info` 里设置，D3D11 和 CUDA/OpenGL 后端分别看 `nvenc-d3d11.c`、`nvenc-cuda.c`、`nvenc-opengl.c`。

### Q17: AMF/QSV/VAAPI/VideoToolbox 分别在哪里？

A: AMF 在 `plugins/obs-ffmpeg/texture-amf.cpp`，QSV 在 `plugins/obs-qsv11/obs-qsv11.c`，VAAPI 在 `plugins/obs-ffmpeg/obs-ffmpeg-vaapi.c`，VideoToolbox 在 `plugins/mac-videotoolbox/encoder.c:1439`。

### Q18: 编码过载和网络丢帧怎么区分？

A: 编码过载看 `obs-encoder.c:1390` `do_encode()` 和硬编插件耗时；网络丢帧看 `rtmp-stream.c` 发送和 output 统计。渲染过载看 `obs-video.c:870` `output_frame()` 和 `:887` `render_video()`。

### Q19: 为什么硬编选项存在但启动失败？

A: 常见原因是驱动版本、GPU 代际、并发 session 数、像素格式、10bit/HDR/AV1 能力不匹配。OBS 后端常有 fallback/reroute，例如 NVENC `nvenc.c:972` 到 `:976`，QSV `obs-qsv11.c:844` 起，VAAPI `obs-ffmpeg-vaapi.c:572`。

### Q20: 如果要给 OBS 增加一个采集源，需要实现什么？

A: 实现并注册 `obs_source_info`，至少包括 `id`、`type`、`output_flags`、`create`、`destroy`、`get_name`，视频源还要实现 `video_render` 或异步调用 `obs_source_output_video()`，音频源调用 `obs_source_output_audio()`。
