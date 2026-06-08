# FFmpeg Project Framework

源码快照：

- 本机路径：`D:/github/FFmpeg`
- Git describe：`n6.0.1-24-gdc02ba2637-dirty`
- Commit：`dc02ba263755b981b809ad2708b77c82586669d9`
- 文档日期：2026-06-01

## 文档

- [整体架构](architecture.md)
- [编解码与硬件加速](codec-and-hwaccel.md)
- [Dolby Vision 支持](dolby-vision.md)
- [播放器高级音视频数据流与工程经验](2.player-av-dataflow.md)
- [HEVC hvcC / AnnexB / BSF 诊断](1.hevc-hvcc-annexb-bsf.md)
- [工程问题手册](engineering-playbook.md)
- [面试问答](interview-qa.md)
- [缺陷与不完善场景](gaps-and-risks.md)

## 图示索引

- 整体分层架构图：见 `architecture.md`。
- ffmpeg 转码主流程图：见 `architecture.md`。
- 解码 API 调用链图：见 `codec-and-hwaccel.md`。
- 硬解选择流程图：见 `codec-and-hwaccel.md`。
- 硬件后端关系图：见 `codec-and-hwaccel.md`。
- Dolby Vision 元数据流图：见 `dolby-vision.md`。
- RPU 解析结构图：见 `dolby-vision.md`。
- 播放器容器、packet、decoder、frame 数据流和高级工程经验：见 `player-av-dataflow.md`。
- HEVC hvcC、AnnexB、BSF 转换与失败链路图：见 `hevc-hvcc-annexb-bsf.md`。
- MP4 起播和参数集链路图：见 `engineering-playbook.md`。
- HLS/m3u8 收流与分片读取图：见 `engineering-playbook.md`。
- 快速 seek 调用链图：见 `engineering-playbook.md`。
- MP4 音视频交错起播图：见 `engineering-playbook.md`。
- 音画同步时钟模型图：见 `engineering-playbook.md`。
- 音频 filter/render 链路图：见 `engineering-playbook.md`。
- 缺陷分类图：见 `gaps-and-risks.md`。

## 快速定位

- CLI：`fftools/ffmpeg.c`、`fftools/ffmpeg_demux.c`、`fftools/ffmpeg_mux.c`、`fftools/ffmpeg_filter.c`。
- 输入封装：`libavformat/demux.c`。
- 输出封装：`libavformat/mux.c`。
- 编解码调度：`libavcodec/decode.c`、`libavcodec/encode.c`。
- 硬解配置：`libavcodec/hwconfig.h`、`libavcodec/hwaccels.h`。
- Dolby Vision：`libavcodec/dovi_rpu.c`、`libavcodec/hevcdec.c`、`libavformat/mov.c`、`libavformat/mpegts.c`。
- HEVC MP4/AnnexB：`libavcodec/hevc_mp4toannexb_bsf.c`、`libavcodec/hevcdec.c`、`libavcodec/h2645_parse.c`、`libavformat/mov.c`。
