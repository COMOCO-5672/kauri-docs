# Kauri Project Docs

这个仓库用于保存多个开源/内部项目的框架文档。每个项目使用一个逻辑名，例如 `ffmpeg`、`obs`、`webrtc`，文档内尽量引用逻辑名和相对源码路径，而不是绑定某台电脑的绝对路径。

## 目录

- `projects/registry.example.json`: 项目逻辑名到本机源码路径的配置样例。
- `projects/ffmpeg/`: FFmpeg 示例文档。
- `projects/obs-studio/`: OBS Studio 文档。
- `projects/webrtc/`: WebRTC Native/C++ 与 RTC 架构文档。
- `agents/project-resolution.md`: AI Agent 如何在不同电脑上解析项目路径。
- `agents/visual-doc-standard.md`: 所有项目文档的图文并茂规范。
- `skills/project-framework-docs/SKILL.md`: 项目框架文档整理 skill 草案。

## 图文并茂要求

所有项目都必须配图，不允许只有文字和表格。每个项目至少包含：

- 一张整体架构图。
- 一张主流程图，展示输入、核心处理、输出。
- 一张关键模块关系图。
- 对复杂专题补充专题图，例如 FFmpeg 的硬解选择流程、Dolby Vision 元数据流。

默认使用 Mermaid 图，因为它可维护、可 diff、可随源码变化快速修改。需要展示 UI、渲染效果、协议帧结构或硬件拓扑时，可以补充 SVG/PNG。

## 推荐工作流

1. 在自己的电脑上复制 `projects/registry.example.json` 为 `projects/registry.local.json`。
2. 把每个项目的本机源码路径填入 `registry.local.json`。
3. 对 AI Agent 只说逻辑名，例如“整理 `ffmpeg` 的硬解架构”，Agent 先读注册表找到源码路径。
4. 文档中引用源码统一写相对路径，例如 `libavcodec/decode.c:1147`。

`registry.local.json` 应该只保存本机路径，不建议提交到版本库。
