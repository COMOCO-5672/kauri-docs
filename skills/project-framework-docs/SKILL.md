---
name: project-framework-docs
description: Build and update source-code-grounded architecture documentation for many projects stored in a docs repository. Use when the user asks to document a project such as FFmpeg, OBS, WebRTC, or another codebase, especially when references must be precise to source files, functions, hardware acceleration paths, codecs, formats, or known gaps.
---

# Project Framework Docs

## Resolve The Project

1. Treat names like `ffmpeg`, `obs`, and `webrtc` as logical project IDs.
2. Read `projects/registry.local.json` first. If it is absent, read `projects/registry.example.json`.
3. Verify the resolved repo path exists and is a git worktree with `git rev-parse --show-toplevel`.
4. Record `git rev-parse HEAD` and `git describe --tags --always --dirty` in the document.
5. Do not write absolute source paths in project docs except in a short source snapshot section. Prefer repo-relative paths.

## Documentation Rules

- Ground technical claims in source references: `path/file.c:line` plus function or structure name.
- Make every project visually documented. Do not produce text-only architecture docs.
- Add diagrams before dense tables. Use Mermaid for maintainable architecture, flow, sequence, state, and dependency diagrams.
- Every major document should contain at least one diagram, and every project should contain an overall architecture diagram, main flow diagram, key subsystem diagram, and gap/risk classification diagram.
- Prefer `rg` for symbol discovery.
- For architecture docs, identify public API entry points, internal dispatch functions, registration tables, and backend-specific implementations.
- Include an engineering playbook for real development issues, not only static architecture. Capture startup latency, metadata loss, color problems, filter/render chains, synchronization, seek, buffering, hardware fallback, and debugging commands when relevant.
- For player/media projects, cover receive/input, demux, decode, filter, render handoff, A/V sync, low latency, stream switching, seek, buffering, and packaging problems. For each major problem, state symptoms, source files/functions, operational fixes, and whether modifying the project source is appropriate.
- Include an interview Q&A document for major projects. Questions should test architecture, debugging judgment, source navigation, project boundaries, and practical tradeoffs.
- For gaps, distinguish source limitation, build-time option, platform/driver limitation, and usage/configuration issue.
- Do not claim a feature is unsupported only because it was not found in one search. Search by codec name, container name, option name, and side-data/API symbols.

## Suggested Output Layout

Use one folder per logical project:

```text
projects/<id>/
  index.md
  architecture.md
  codec-and-hwaccel.md
  dolby-vision.md
  engineering-playbook.md
  interview-qa.md
  gaps-and-risks.md
```

Adapt file names to the project. For non-media projects, replace codec-specific files with domain-specific files.

## Visual Standard

Follow `agents/visual-doc-standard.md` when available. For each diagram:

- Put a short sentence before the diagram explaining the question it answers.
- Put source entry points after the diagram.
- Keep node names tied to real modules, files, functions, or data structures.
- Prefer several focused diagrams over one oversized diagram.

## FFmpeg Focus Areas

When documenting FFmpeg, inspect at least:

- CLI orchestration: `fftools/ffmpeg*.c`, `fftools/ffmpeg*.h`.
- Demux/mux: `libavformat/demux.c`, `libavformat/mux.c`, relevant format files.
- Decode/encode dispatch: `libavcodec/decode.c`, `libavcodec/encode.c`, `libavcodec/allcodecs.c`.
- Hardware acceleration: `libavcodec/hwconfig.h`, `libavcodec/hwaccels.h`, `nvdec*`, `vaapi*`, `dxva2*`, `qsv*`, `videotoolbox*`, `mediacodec*`, `vdpau*`.
- Dolby Vision: `libavcodec/dovi_rpu.*`, `libavcodec/hevcdec.*`, `libavutil/dovi_meta.h`, `libavformat/mov*`, `libavformat/mpegts*`.
- Engineering issues: HLS/m3u8, fast seek, MP4 `moov`/`mdat`/A-V interleaving/`avcC`/`hvcC`, `h264_mp4toannexb`, `hevc_mp4toannexb`, audio `abuffer`/`abuffersink`/`aresample`, ffplay audio rendering, PTS/DTS/time_base, low-latency options, buffer strategy boundaries.

## Validation

Before finishing:

- Run `rg` against documented function names to confirm they still exist.
- Check markdown links and relative paths.
- Run `git status --short` in the docs repo and summarize created/changed files.
