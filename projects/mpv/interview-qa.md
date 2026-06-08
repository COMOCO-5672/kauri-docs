# mpv 面试问答

## 架构理解

**Q：mpv 和 FFmpeg 的架构差异是什么？**

A：FFmpeg 主要提供 demux、decode、filter、encode、mux 等库和命令行工具；mpv 是播放器状态机。mpv 使用 FFmpeg 的 libavformat/libavcodec，但播放生命周期由 `player/main.c:446` `mpv_main()`、`player/loadfile.c:2080` `mp_play_files()` 和 `player/playloop.c:1256` `run_playloop()` 驱动。核心状态集中在 `player/core.h:235` `struct MPContext`。

**Q：mpv 播放一个文件的主调用链怎么走？**

A：命令行入口 `mpv_main()` 创建并初始化 `MPContext`，`mp_play_files()` 遍历播放列表，`play_current_file()` 打开单个文件，`open_demux_reentrant()` 调 `demux_open_url()`，然后 `add_demuxer_tracks()` 建 track，`reinit_video_chain()`/`reinit_audio_chain()` 建音视频链，最后 `run_playloop()` 不断调用 `write_video()` 和 `fill_audio_out_buffers()`。

关键位置：`player/main.c:446`、`player/loadfile.c:1630`、`player/loadfile.c:1216`、`demux/demux.c:3485`、`player/video.c:204`、`player/audio.c:546`、`player/playloop.c:1256`。

**Q：为什么读 mpv 源码经常从 `MPContext` 开始？**

A：`MPContext` 是播放器运行时上下文，保存 playlist、当前文件、demuxer、tracks、current_track、filter_root、VO/AO、client API、input、播放状态和同步状态。很多问题不是单个模块的问题，而是这些状态组合的问题。结构定义在 `player/core.h:235`。

## demux、decode 和 filter

**Q：mpv 的 demux 是否完全交给 FFmpeg？**

A：不是。mpv 有自己的 demux 抽象、packet 队列、cache 和 seek 状态；FFmpeg 是 `demux_lavf` 后端之一。打开入口是 `demux/demux.c:3485` `demux_open_url()`，lavf 后端是 `demux/demux_lavf.c:973` `demux_open_lavf()`，读包进入 `demux/demux_lavf.c:1217` `demux_lavf_read_packet()`。

**Q：decoder wrapper 解决什么问题？**

A：它把 demux packet、lavc decoder、codec 切换、丢帧、硬解状态和 filter graph 对接起来。入口是 `filters/f_decoder_wrapper.c:1204` `mp_decoder_wrapper_create()`，重初始化在 `filters/f_decoder_wrapper.c:480` `decoder_wrapper_reinit()`，lavc 推进在 `filters/f_decoder_wrapper.c:1302` `lavc_process()`。

**Q：filter graph 由谁推动？**

A：播放循环会调用 `filters/filter.c:211` `mp_filter_graph_run()`。音视频输出链使用 `filters/f_output_chain.c:683` `mp_output_chain_create()` 创建，lavfi graph 由 `filters/f_lavfi.c:903` `mp_lavfi_create_graph()` 创建。

## 同步、缓存和 seek

**Q：音画同步主要在哪里做？**

A：音频时钟来自 `player/audio.c:624` `playing_audio_pts()` 和 `audio/out/buffer.c:295` `ao_get_delay()`；视频侧在 `player/video.c:343` `adjust_sync()`、`player/video.c:590` `update_avsync_before_frame()` 和 `player/video.c:810` `handle_display_sync_frame()` 调整等待、补偿和 display sync。

**Q：seek 的入口和真正执行点分别在哪里？**

A：命令侧会调用 `player/playloop.c:455` `queue_seek()` 排队；播放循环在 `player/playloop.c:493` `execute_queued_seek()` 执行；核心 seek 逻辑在 `player/playloop.c:289` `mp_seek()`；demux 层入口是 `demux/demux.c:3798` `demux_seek()`，lavf 后端是 `demux/demux_lavf.c:1327` `demux_seek_lavf()`。

**Q：缓存卡顿应该看哪里？**

A：先看 demux cache 和播放循环。demux 层 `demux/demux.c:4167` `update_cache()` 更新缓存，`demux/demux.c:4513` `demux_get_reader_state()` 暴露状态；播放层 `player/playloop.c:706` `handle_update_cache()` 决定是否 paused-for-cache。

## 渲染和硬解

**Q：`--hwdec` 为什么不是只打开 decoder 就够？**

A：硬解需要 decoder 产出硬件帧，VO 提供对应设备和 interop，renderer 能把硬件帧 map 成 texture。相关入口包括 `video/hwdec.c:20` `hwdec_devices_create()`、`video/out/gpu/hwdec.c:250` `ra_hwdec_ctx_init()`、`video/out/gpu/hwdec.c:168` `ra_hwdec_mapper_map()` 和 `player/video.c:558` `check_for_hwdec_fallback()`。

**Q：`vo_gpu_next` 和 `vo_gpu` 的区别应该怎么概括？**

A：两者都是 GPU VO，但 `vo_gpu_next` 走 libplacebo 主渲染路径，入口在 `video/out/vo_gpu_next.c:998` `draw_frame()` 和 `video/out/vo_gpu_next.c:2565` `video_out_gpu_next`；旧 `vo_gpu` 使用 mpv 自己的 `gl_video` renderer，入口在 `video/out/vo_gpu.c:73` `draw_frame()` 和 `video/out/gpu/video.c:3407` `gl_video_render_frame()`。

**Q：libmpv 嵌入模式如何渲染？**

A：应用调用 `mpv_render_context_create()` 创建外部渲染上下文，再通过 update callback 驱动 `mpv_render_context_render()`，并在 swap 后调用 `mpv_render_context_report_swap()`。实现集中在 `video/out/vo_libmpv.c:164`、`:336`、`:429` 和 `include/mpv/render.h:578`、`:709`。

## 输入、脚本和 API

**Q：按键、IPC、脚本和 libmpv 命令是否走不同系统？**

A：入口不同，但最终汇入 command/property/event 机制。按键由 `input/input.c:1356` `mp_input_read_cmd()` 读取，命令解析在 `input/cmd.c:449` `mp_input_parse_cmd_str()`，执行在 `player/command.c:5532` `run_command()`，属性在 `player/command.c:4693` `mp_property_do()`。libmpv API 入口在 `player/client.c`。

**Q：脚本系统如何接到 core？**

A：`player/scripting.c:276` `mp_load_scripts()` 加载用户脚本，`player/scripting.c:262` `mp_load_builtin_scripts()` 加载内置脚本，Lua/JS backend 分别在 `player/lua.c:1351` `mp_scripting_lua` 和 `player/javascript.c:1253` `mp_scripting_js`。脚本通常通过 libmpv 风格的命令、属性、事件访问 core。

## 调试判断

**Q：遇到黑屏但有声音，怎么缩小范围？**

A：先用 `--no-config --load-scripts=no` 排除配置和脚本，再对比 `--hwdec=no`、`--vo=gpu-next`、`--vo=gpu`、`--vo=null`。如果 `--vo=null` 正常推进而 GPU VO 黑屏，看 `video/out/vo.c:921` `render_frame()`、具体 VO `draw_frame()` 和 `video/out/gpu/hwdec.c:168` `ra_hwdec_mapper_map()`。

**Q：遇到 seek 后卡住，怎么判断是 demux 还是播放循环？**

A：看 `seeking`、`paused-for-cache`、`demuxer-cache-state` 和日志。若 demux cache 没新包，查 `demux/demux.c:2419` `execute_seek()` 和 lavf seek；若已有包但不出图，查 `player/video.c:466` `video_output_image()` 的精确 seek 丢帧、`player/playloop.c:1149` `handle_playback_restart()` 的 ready 条件。

**Q：什么时候适合改 mpv 源码，什么时候应该改配置或依赖？**

A：`--no-config`、换 VO/AO、禁用硬解后仍可复现，且日志显示 core 状态错误，才优先改源码。只在某个构建包缺功能，先看 meson/依赖；只在某个 GPU 或 AO 后端复现，先看平台驱动和对应 `video/out/`、`audio/out/` 后端；只在一个文件复现，先看容器、时间戳和 FFmpeg demuxer。
