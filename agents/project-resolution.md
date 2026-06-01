# Project Path Resolution

目标：在不同电脑上使用 AI Agent 时，尽量只说项目逻辑名，不反复描述绝对路径。

## 约定

- 文档仓库内维护 `projects/registry.local.json`，字段结构参考 `projects/registry.example.json`。
- `registry.local.json` 记录本机路径，不提交版本库。
- 文档正文只写项目逻辑名和源码相对路径。
- Agent 在执行任务前先解析逻辑名，例如 `ffmpeg` -> `D:/github/FFmpeg`。

## Agent 查找顺序

1. 如果用户给了绝对路径，直接使用该路径，并可提醒是否写入注册表。
2. 如果用户给了逻辑名，优先读取 `projects/registry.local.json`。
3. 如果没有 local 文件，读取 `projects/registry.example.json` 作为提示，但必须验证路径存在。
4. 如果注册表没有该项目，尝试在常见目录查找：`D:/github`、`D:/work`、`C:/github`、用户主目录下的 `github`、`work`。
5. 找到后用 `git rev-parse --show-toplevel` 验证仓库根目录。

## 文档引用规则

- 代码引用格式：`libavcodec/decode.c:1147`。
- 一篇文档开头写明源码版本：commit、tag/describe、是否 dirty。
- 不把 `D:/github/FFmpeg/libavcodec/decode.c` 这种绝对路径写进项目文档正文。
- 只有注册表和任务记录可以保存绝对路径。

## 建议提交策略

- 提交 `registry.example.json`。
- 忽略 `registry.local.json`。
- 每个项目文档目录保留 `index.md` 作为入口。
