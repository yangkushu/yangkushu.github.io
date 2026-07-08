+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：流媒体和发布概览'
+++

## 系列目录

- [目录](/posts/ffmpeg-learning-index/)
- 入门：[教程总览](/posts/ffmpeg-learning-00-overview/) / [环境准备](/posts/ffmpeg-learning-01-setup/) / [必要音视频概念](/posts/ffmpeg-learning-02-concepts/)
- 核心任务：[01 查看媒体信息](/posts/ffmpeg-learning-task-01-inspect-media/) / [02 生成测试素材](/posts/ffmpeg-learning-task-02-generate-samples/) / [03 转封装](/posts/ffmpeg-learning-task-03-remux/) / [04 转码](/posts/ffmpeg-learning-task-04-transcode/) / [05 压缩视频](/posts/ffmpeg-learning-task-05-compress/)
- 核心任务：[06 调整分辨率](/posts/ffmpeg-learning-task-06-resize/) / [07 裁剪片段](/posts/ffmpeg-learning-task-07-trim/) / [08 拼接视频](/posts/ffmpeg-learning-task-08-concat/) / [09 提取音频](/posts/ffmpeg-learning-task-09-extract-audio/) / [10 替换音频](/posts/ffmpeg-learning-task-10-replace-audio/)
- 核心任务：[11 截取缩略图](/posts/ffmpeg-learning-task-11-thumbnail/) / [12 添加字幕](/posts/ffmpeg-learning-task-12-subtitles/) / [13 添加水印](/posts/ffmpeg-learning-task-13-watermark/) / [14 调整播放速度](/posts/ffmpeg-learning-task-14-speed/) / [15 批量处理目录](/posts/ffmpeg-learning-task-15-batch-processing/)
- 自动化：[批处理工作流](/posts/ffmpeg-learning-automation-batch-processing/) / [ffprobe 验收方法](/posts/ffmpeg-learning-automation-ffprobe-validation/) / [AI 提示词模式](/posts/ffmpeg-learning-automation-ai-prompt-patterns/)
- 进阶：[FFmpeg API 概览](/posts/ffmpeg-learning-advanced-ffmpeg-api-overview/) / [生态工具概览](/posts/ffmpeg-learning-advanced-ecosystem/) / [流媒体和发布概览](/posts/ffmpeg-learning-advanced-streaming/)

---


## Web 兼容 MP4

面向网页、移动端和常见播放器发布时，最稳妥的基础格式通常是 MP4 container、H.264 video、AAC audio、`yuv420p` pixel format，并加上 `-movflags +faststart`。

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -crf 23 -preset medium -pix_fmt yuv420p -c:a aac -b:a 128k -movflags +faststart examples/output/web-compatible.mp4
```

这类输出适合直接上传、网页预览和普通文件分发。它不是自适应码率流媒体，但足够覆盖很多日常发布场景。

## HLS 基础

HLS 会把视频切成多个 `.ts` 或 `.m4s` segment，并生成一个 `.m3u8` playlist。播放器先读取 playlist，再按需拉取 segment。

```bash
mkdir -p examples/output/hls
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -c:a aac -hls_time 4 -hls_playlist_type vod examples/output/hls/index.m3u8
```

验收：

```bash
test -f examples/output/hls/index.m3u8
find examples/output/hls -maxdepth 1 -type f
```

`-hls_time 4` 表示目标 segment 时长约 4 秒。实际 segment 边界通常受 keyframe 影响。

## DASH 基础

DASH 和 HLS 类似，也是 manifest 加 segment 的结构。DASH 常见 manifest 是 `.mpd`。FFmpeg 能生成基础 DASH 输出，但更复杂的多码率、自适应和加密场景通常会使用 Shaka Packager 或 GPAC。

```bash
mkdir -p examples/output/dash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -c:a aac -f dash examples/output/dash/manifest.mpd
```

## 关键帧和切片

切片边界通常应落在 keyframe 上。若 keyframe 间隔太长，HLS/DASH segment 可能不稳定或不接近目标时长。常见做法是明确 GOP：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -g 60 -keyint_min 60 -sc_threshold 0 -c:a aac -hls_time 2 -hls_playlist_type vod examples/output/hls/index.m3u8
```

30 FPS 视频中 `-g 60` 大约是 2 秒一个 keyframe。真实项目还要结合平台规格、编码 ladder 和播放器行为测试。

## 平台上传规格

不同平台对 codec、resolution、bitrate、frame rate、audio codec 和 loudness 可能有不同要求。不要只让 AI 猜参数；应先收集目标平台规格，再生成命令。

通用检查项：

- container：MP4、HLS、DASH 或平台指定格式。
- video codec：H.264、H.265、AV1。
- audio codec：AAC 通常兼容性较好。
- resolution 和 frame rate：是否符合平台推荐。
- bitrate 或 CRF：是否在质量和体积之间平衡。
- subtitle：是否需要 soft subtitles 或 burned subtitles。
