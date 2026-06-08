# Experience Documentation Standard

这个仓库的“经验文档”要体现高级工程判断，而不是只解释当前问题。经验文档必须让读者知道：系统架构是什么、数据合同是什么、正常路径如何工作、异常路径如何定位、真实工程如何降级。

## 适用范围

适用于：

- `player-*.md`
- `*-experience.md`
- `*-diagnosis.md`
- 用户明确要求“经验文档”“高级开发经验”“项目架构和实战问题”
- 媒体播放器、FFmpeg、mpv、OBS、WebRTC 等包含复杂数据流和平台差异的项目文档

## 文件命名

统一命名，避免和普通模块说明混淆：

| 文档类型 | 命名格式 | 示例 |
| --- | --- | --- |
| 顶层架构 | `architecture.md` | `architecture.md` |
| 高级经验 | `<domain>-experience.md` 或 `player-*.md` | `player-av-dataflow.md` |
| 专项诊断 | `<topic>-diagnosis.md` | `hevc-annexb-diagnosis.md` |
| 风险清单 | `gaps-and-risks.md` | `gaps-and-risks.md` |

媒体播放器相关经验文档优先使用 `player-` 前缀。

## 必须结构

经验文档建议按这个顺序组织：

1. **标题和定位**：说明文档是系统经验、专项诊断还是实现手册。
2. **源码快照**：项目路径、commit、describe、日期。
3. **阅读标记**：声明重点块含义。
4. **架构地图**：用图说明模块边界和依赖。
5. **数据合同**：列出核心结构、字段、生命周期和谁负责维护。
6. **正常流程**：从入口到输出的主路径。
7. **异常流程**：seek、切流、fallback、弱支持、参数变化。
8. **格式/平台差异**：不能只写一种容器、codec、硬件或平台。
9. **高级风险表**：症状、先查字段、源码入口、降级策略。
10. **日志/排查矩阵**：打开、读包、送解码、出帧、渲染/AO 各打印什么。
11. **经验规则汇总**：把关键判断压缩成可复用规则。

## 重点块格式

统一使用 GitHub admonition：

```markdown
> [!IMPORTANT]
> 核心合同、必须保持一致的数据关系、不能破坏的不变量。

> [!WARNING]
> 真实工程高风险点、弱支持格式、平台/驱动限制、数据不匹配。

> [!TIP]
> 实现策略、日志建议、fallback、调试命令。

> [!NOTE]
> 标准背景、术语解释、帮助理解但不一定每次处理的信息。
```

不要只靠普通段落表达重点。

## 媒体播放器经验文档要求

媒体文档必须分清这些层：

| 层 | 必须说明 |
| --- | --- |
| 容器 | MP4/MOV、MKV、TS/M2TS、HLS/DASH、raw stream 的差异 |
| 流参数 | `AVStream`、`AVCodecParameters`、extradata、coded side data |
| packet | payload 格式、PTS/DTS、duration、flags、packet side data |
| parser/BSF | 是否改变边界、格式或 extradata 合同 |
| decoder | `AVCodecContext`、private state、硬解上下文 |
| frame | `AVFrame` 数据、色彩、HDR/DV、audio samples、frame side data |
| 输出 | renderer/AO 能力、硬件 surface、PCM/透传、同步 |

必须覆盖常见高级风险：

- MP4 NALFF 和 AnnexB 混用。
- `hvcC/avcC` 缺参数集或数组为空。
- SPS/PPS/VPS 只在文件开头、seek 后缺失、关键帧前不重复。
- TS/M2TS 时间戳跳变、PAT/PMT 更新、PES 边界、蓝光 playlist 和 clip 拼接。
- HLS/DASH representation 切换导致 extradata、timebase、分辨率变化。
- Dolby Vision container config、RPU、profile fallback、BL/HDR10/SDR fallback。
- HDR10/HDR10+ metadata 从 packet side data 到 frame side data 的传递。
- Dolby 音频 PCM 解码和 HDMI/SPDIF passthrough 的差异。
- 硬解 profile/level/bit depth/输出 surface 不匹配。
- 音画同步、seek、缓冲、低延迟策略。

## 表格和图

- 复杂表格前必须先给一张图。
- 表格要服务于决策，不要只罗列名词。
- 风险表至少包含：问题、症状、先查什么、源码入口、工程处理。
- 容器/codec 差异表至少包含：参数来源、packet 形态、时间戳特点、风险。

## 源码引用

- 所有技术判断必须有源码入口支撑。
- 引用格式：`path/file.c:line` 加函数、结构或字段名。
- 不要把绝对源码路径写进正文，源码快照除外。
- 如果无法确认某个行为，写成“需要验证”，并列出验证命令或日志点。

## 质量门槛

完成前检查：

- 文档是否包含架构图、流程图或状态图。
- 是否明确核心数据合同。
- 是否写了异常路径和 fallback。
- 是否区分了容器、codec、decoder、硬件、输出设备层。
- 是否使用了 `IMPORTANT/WARNING/TIP/NOTE` 标记。
- 是否有日志矩阵或排查清单。
- 是否有源码引用和验证点。
