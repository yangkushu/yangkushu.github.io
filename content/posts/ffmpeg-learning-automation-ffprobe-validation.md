+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：ffprobe 验收方法'
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


## 总体思路

FFmpeg 命令执行成功不等于输出正确。每个任务都应使用 `ffprobe` 检查输出文件的关键字段，再结合必要的人工播放或截图检查。

## 前置检查

处理前先查看输入：

```bash
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_format -show_streams examples/input/sample-720p.mp4
```

脚本中建议使用 `-v error`，减少 banner、warning 和诊断日志噪声，方便解析字段。

## 常用字段

检查视频：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt,duration -of default=nw=1 output.mp4
```

检查音频：

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate,duration -of default=nw=1 output.mp4
```

检查字幕：

```bash
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name,codec_type,duration -of default=nw=1 output.mp4
```

检查整体时长：

```bash
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 output.mp4
```

## 不同任务的验收重点

- 转封装：codec 是否保持不变，container 是否改变。
- 转码：video/audio codec、pixel format、bitrate 是否符合目标。
- 压缩：文件体积、CRF 输出质量、codec、pix_fmt、audio bitrate。
- 缩放：width、height、DAR/SAR。
- 裁剪：duration、stream 是否存在、边界是否符合预期。
- 拼接：输出字段是否统一，duration 是否接近片段之和，播放切换点。
- 字幕：soft subtitle 检查 subtitle stream；burned subtitle 需要截图或播放检查。

## AI 使用建议

让 AI 生成 FFmpeg 命令时，要求它同时生成验收命令。验收命令至少应覆盖：

- 输出文件是否存在。
- video stream 是否存在及 codec/resolution/pix_fmt。
- audio stream 是否存在及 codec/sample_rate/channels。
- format duration 是否接近预期。
- 任务特有字段，例如 subtitle stream、thumbnail image、watermark screenshot。
