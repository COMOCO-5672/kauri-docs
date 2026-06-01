# FFmpeg Interview Q&A

这些问题按工程面试组织，不追求背 API，而是看候选人能不能把源码、媒体格式、播放器问题和 FFmpeg 边界连起来。

## 架构与主流程

### Q1: ffmpeg 命令从输入到输出的核心链路是什么？

A: `fftools` 负责参数和编排，输入通过 `avformat_open_input()`、`avformat_find_stream_info()`、`av_read_frame()` 得到 `AVPacket`，送入 `avcodec_send_packet()`/`avcodec_receive_frame()` 解码，经过 `libavfilter`，再用 encoder 产出 packet，最后 `av_interleaved_write_frame()` 写出。源码入口见 `fftools/ffmpeg_demux.c:967`、`libavformat/demux.c:1439`、`libavcodec/decode.c:598`、`libavformat/mux.c:1241`。

### Q2: `AVPacket`、`AVFrame`、`extradata`、side data 分别是什么？

A: `AVPacket` 是压缩数据包，`AVFrame` 是解码后的音视频帧或硬件帧。`extradata` 保存 codec 初始化数据，例如 H.264 avcC 里的 SPS/PPS、HEVC hvcC 里的 VPS/SPS/PPS。side data 是附着在 packet/frame 上的扩展元数据，例如 `AV_PKT_DATA_DOVI_CONF`、`AV_FRAME_DATA_DOVI_METADATA`。

### Q3: 为什么 `avformat_find_stream_info()` 有时会触发解码？

A: 它需要读一些 packet 来补全 stream 参数，必要时会借助 parser/decoder 推断 codec 信息、时间基、帧率、extradata 等。入口是 `libavformat/demux.c:2425`。

## MP4 与起播

### Q4: MP4 起播慢最常见原因是什么？

A: `moov` 在文件尾。播放器需要 `moov` 里的 sample table、track metadata、codec extradata 才能定位和解码 `mdat`。普通 progressive download 如果先拿不到 `moov`，就要等待下载尾部或额外 seek。相关代码：`libavformat/mov.c:8448` `mov_read_header()`，`libavformat/mov.c:1170` `mov_read_moov()`。

### Q5: `-movflags +faststart` 做了什么？

A: mux 完成后第二遍把 `moov` atom 移到文件开头，改善 HTTP 点播起播。选项定义在 `libavformat/movenc.c:81`，写尾处理在 `libavformat/movenc.c:7557` 附近。

### Q6: 普通 MP4 和 fragmented MP4 对起播有什么差异？

A: 普通 MP4 依赖完整 `moov` 和 sample table；fragmented MP4 可以用 `moof+mdat` 分片逐段提供索引和媒体数据，更适合直播、低延迟和边录边播。相关 movflags 包括 `empty_moov`、`delay_moov`、`separate_moof`。

## SPS/PPS/VPS

### Q7: MP4 里的 SPS/PPS 从哪里来？

A: H.264 通常在 `avcC`，HEVC 在 `hvcC`。MOV demuxer 读取 sample entry 的 extra atoms 后写入 `AVCodecParameters.extradata`，注释见 `libavformat/mov.c:2618`。

### Q8: SPS/PPS 怎么送进解码器？

A: 应用层通常把 `AVCodecParameters` 拷到 `AVCodecContext`，decoder init 时读 `avctx->extradata`。H.264 在 `libavcodec/h264dec.c:388` 调用 `ff_h264_decode_extradata()`；HEVC 在 `libavcodec/hevcdec.c:3667` 调用 `hevc_decode_extradata()`。

### Q9: 如果 packet 里也带 SPS/PPS，会发生什么？

A: decoder 在 `decode_nal_units()` 里遇到 SPS/PPS 会更新参数集。H.264 见 `libavcodec/h264dec.c:680` 和 `:700`；HEVC 见 `libavcodec/hevcdec.c:2973`、`:2986`、`:3000`。

### Q10: `h264_mp4toannexb` 的作用是什么？

A: 把 MP4 length-prefixed NAL 转成 Annex B start-code 格式，并在 IDR 前补 SPS/PPS，常用于 TS、RTP、裸流或硬件 decoder 输入。源码是 `libavcodec/h264_mp4toannexb_bsf.c:65` 和 `:169`。

### Q11: “non-existing PPS 0 referenced” 一般怎么排查？

A: 查 extradata 是否传入 decoder、packet 是否从关键帧开始、SPS/PPS 是否被切片器丢掉、MP4 转 TS 是否缺少 `h264_mp4toannexb`。H.264 slice 读取 `pps_id` 在 `libavcodec/h264_slice.c:1703`，找 PPS 在 `:1714`。

## Dolby Vision 与颜色

### Q12: FFmpeg 支持 Dolby Vision 到什么程度？

A: 支持 DOVI 配置记录、HEVC RPU NAL 识别、RPU 解析，并通过 frame side data 暴露；不等于完整 Dolby Vision 渲染。关键代码：`libavcodec/hevcdec.c:3208`、`libavcodec/dovi_rpu.c:194`、`libavcodec/dovi_rpu.c:91`。

### Q13: Dolby Vision P5 偏色通常是什么原因？

A: P5 不能简单当普通 HDR10/BT.2020/PQ 画面显示。若渲染器不消费 `AV_FRAME_DATA_DOVI_METADATA` 或不做 DOVI 映射，只按普通颜色标签显示，就可能偏色。FFmpeg 能解析 metadata，但渲染侧要负责应用。

### Q14: 怎么判断是 FFmpeg 没解析 metadata，还是播放器没用 metadata？

A: 先用 `ffprobe -show_frames` 或代码检查 frame side data 是否有 `AV_FRAME_DATA_DOVI_METADATA`/`AV_FRAME_DATA_DOVI_RPU_BUFFER`。如果有，问题多半在渲染/tonemap；如果没有，再查 demux 的 `AV_PKT_DATA_DOVI_CONF` 和 HEVC RPU 解析。

### Q15: 普通颜色标签从哪里来？

A: 主要来自 codec bitstream VUI/metadata 和容器。H.264 SPS VUI 写入 `AVCodecContext.color_range/color_primaries/color_trc/colorspace` 的代码在 `libavcodec/h264_slice.c:1091` 附近。

## 硬解

### Q16: FFmpeg 硬解是怎么接入的？

A: decoder 构造候选 `pix_fmts`，调用 `ff_get_format()` 选择硬件像素格式，再匹配 codec 的 `hw_configs`，初始化 `AVHWAccel` 和 `hw_frames_ctx`。入口是 `libavcodec/decode.c:1147`、`:1099`、`:998`。

### Q17: `hw_device_ctx` 和 `hw_frames_ctx` 区别是什么？

A: `hw_device_ctx` 表示硬件设备，`hw_frames_ctx` 表示基于设备的硬件帧池和格式约束。解码时如果只有 device，FFmpeg 可能通过 `ff_decode_get_hw_frames_ctx()` 创建 frames ctx，但复杂 pipeline 最好明确管理 frames ctx。

### Q18: 为什么硬解后 filter 报不支持格式？

A: 很多普通 filter 只接受软件帧。硬件帧需要 `hwdownload` 到 CPU，或使用对应硬件 filter，例如 VAAPI filter。`fftools/ffmpeg_filter.c:928` 还有 TODO 提到自动插入硬件 filter 未完成。

## 音频 filter 与渲染

### Q19: 音频 filter graph 的基本结构是什么？

A: `abuffer -> filters -> abuffersink`。ffmpeg CLI 创建入口见 `fftools/ffmpeg_filter.c:991`、`:687`，推帧用 `av_buffersrc_add_frame()`，配置用 `avfilter_graph_config()`。

### Q20: ffplay 音频渲染链路可以参考哪些函数？

A: `fftools/ffplay.c:2334` `audio_decode_frame()`，`:2368` `swr_alloc_set_opts2()`，`:2406` `swr_convert()`，`:2443` `sdl_audio_callback()`，`:2486` `audio_open()`。

### Q21: 音频变速、爆音、声道错乱怎么排查？

A: 先确认 sample rate、sample format、channel layout 在 decoder 输出、filter graph、resampler、设备端是否一致。看 `aformat`、`aresample`、`pan/channelmap` 是否显式约束，避免隐式转换。

### Q22: 音画同步 FFmpeg 能提供什么参考？

A: ffplay 的同步逻辑值得参考：`synchronize_audio()` 根据主时钟调整样本数，必要时通过 `swr_set_compensation()` 微调。入口是 `fftools/ffplay.c:2286` 和 `:2397`。

## 边界与发散

### Q23: FFmpeg 能不能解决播放器所有问题？

A: 不能。FFmpeg 主要解决容器、codec、filter、转换、部分硬件加速。起播策略、网络缓冲、ABR、DRM、安全解码、平台渲染、Dolby Vision 最终显示映射通常在播放器/平台层。

### Q24: 遇到线上花屏，应该怎么拆问题？

A: 先分容器、bitstream、decoder、hwaccel、渲染。确认 packet 是否完整、extradata 是否正确、是否从关键帧开始、软解是否正常、硬解是否 profile 不兼容、渲染是否误用 stride/format/color metadata。

### Q25: 如何设计一个基于 FFmpeg 的播放器解码模块？

A: 把 demux、decode、filter、render 解耦。packet 队列和 frame 队列分开；decoder 负责 extradata/flush/seek/参数变化；filter 负责格式约束；render 负责时钟和颜色管理。硬解路径要明确 device/frames ctx 和 fallback。

### Q26: 如何评价候选人是否真的懂 FFmpeg？

A: 不只看能否调用 API，而看是否能解释 `extradata`、time_base、PTS/DTS、SPS/PPS、filter graph、hw_frames_ctx、side data、BSF、容器差异，以及能否把一个现象定位到正确层级。

## 播放器工程问题

### Q27: FFmpeg 处理 m3u8/HLS 的主链路是什么？

A: HLS demuxer 在 `hls_read_header()` 中解析 m3u8，调用 `parse_playlist()` 建立 variant、playlist、segment 信息，再由 `select_cur_seq_no()` 决定起始 segment。播放过程中 `hls_read_packet()` 进入 `read_data()`，必要时 reload live playlist，再通过 `open_input()` 打开 segment 并交给内部 demuxer 解析。源码入口是 `libavformat/hls.c:1932`、`:1953`、`:1725`、`:2292`、`:1471`、`:1276`。

### Q28: HLS 直播起播慢应该怎么排查？

A: 先看起播点是不是离 live edge 太远：`live_start_index`、`prefer_x_start`、`#EXT-X-START` 会影响 `select_cur_seq_no()`。再看 playlist reload、segment 是否过期、首个 segment 是否含关键帧。FFmpeg 能负责 m3u8/segment 拉取和 demux；ABR、CDN 切换、buffer 水位和首帧策略通常在播放器层。

### Q29: HLS seek 为什么经常不准？

A: HLS seek 本质上先定位 segment，再从 segment 内解码。`hls_read_seek()` 会用 `find_timestamp_in_playlist()` 找目标时间所在片段；如果不是 `AVSEEK_FLAG_ANY`，会倾向回到 segment start 或关键帧，源码在 `libavformat/hls.c:2437`、`:2476`、`:2480`。所以 seek 落点偏前是正常策略，播放器要 decode 后丢帧到目标 PTS。

### Q30: 快速 seek 和精确 seek 怎么取舍？

A: 快速 seek 优先找关键帧，允许落点偏前，体验上更稳；精确 seek 要从关键帧开始解码并丢弃目标前帧，GOP 越长越慢。通用入口是 `av_seek_frame()` 和 `avformat_seek_file()`，在 `libavformat/seek.c:636`、`:659`。`fflags=fastseek` 快但不准，`seek2any` 可到非关键帧但 H.264/HEVC 可能缺参考帧。

### Q31: MP4 已经 `faststart` 了，为什么仍然起播慢？

A: `faststart` 只把 `moov` 移到文件头，解决 metadata 先到的问题；如果 `mdat` 内音视频 sample 交错很差，播放器仍要发多次 Range 请求或等待远处 chunk 才能拿齐首帧音视频。mux 侧要看 `av_interleaved_write_frame()`、`ff_interleave_packet_per_dts()`、`max_interleave_delta`，源码在 `libavformat/mux.c:1241`、`:923`、`:953`。

### Q32: MP4 音视频交错问题能通过改 FFmpeg 解决吗？

A: 如果你控制打包链路，优先用 FFmpeg mux 参数或改 muxer：限制首段 A/V offset 距离、输出首个关键帧 offset 和首个音频 offset 诊断、调整 `max_interleave_delta`。如果文件已经在线上，播放器 demuxer 很难“消除”物理布局带来的 Range 成本，只能预取或异步缓冲。

### Q33: 音画同步应该由 FFmpeg 还是播放器负责？

A: FFmpeg 提供时间戳、time_base、demux 修正和 ffplay 参考实现，但同步策略属于播放器。ffplay 里 `get_master_clock()`、`compute_target_delay()`、`synchronize_audio()`、`swr_set_compensation()` 分别处理主时钟、视频显示延迟、音频样本微调，源码在 `fftools/ffplay.c:1428`、`:1514`、`:2286`、`:2397`。

### Q34: 音画不同步时第一步看什么？

A: 先看 PTS/DTS 是否连续、`time_base` 是否正确、seek 后队列和 decoder 是否 flush、音频设备缓冲是否计入 audio clock。FFmpeg 的 `compute_pkt_fields()` 会计算 packet 字段，`fflags=genpts` 可在缺 PTS 时尝试生成，入口是 `libavformat/demux.c:926`、`:1439` 和 `libavformat/options_table.h:45`。

### Q35: 低延迟播放可以只靠 `fflags=nobuffer` 吗？

A: 不够。`fflags=nobuffer`、`avioflags=direct`、`max_delay`、`probesize`、`analyzeduration` 能减少 FFmpeg 输入侧额外等待，但低延迟还依赖协议、segment/part 时长、网络 jitter buffer、解码速度、渲染调度和 ABR。相关选项在 `libavformat/options_table.h`。

### Q36: “起播慢”要怎么分层定位？

A: 先分网络、容器 metadata、媒体数据布局、关键帧位置、解码初始化、渲染首帧。MP4 看 `moov`、首个关键帧和 A/V offset；HLS 看起始 segment、playlist reload、HTTP 建连；解码看 extradata 和硬解初始化；渲染看像素格式、颜色和首帧提交。

### Q37: 哪些播放器问题适合改 FFmpeg？

A: 适合改的是格式解析 bug、特定 demuxer 的 timestamp 修正、HLS seek 诊断、segment retry 行为、muxer 交错策略、硬解后端能力识别、side data 透传。不要把 ABR、CDN 调度、业务缓冲策略、DRM 安全链路、平台渲染和 Dolby Vision 最终 tone mapping 硬塞进通用 FFmpeg。

### Q38: 如果要给 FFmpeg 加播放器诊断能力，优先加什么？

A: 优先加不会改变行为的观测信息：HLS 命中的 `seq_no` 和 `seg_start_ts`、MP4 seek 的目标 PTS/实际 DTS/关键帧 sample index、首个音视频 sample offset 距离、DOVI side data 是否生成、硬解选择和 fallback 原因。这些信息能直接缩短线上排障时间。
