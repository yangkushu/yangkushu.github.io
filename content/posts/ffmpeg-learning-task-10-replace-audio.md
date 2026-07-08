+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：替换音频'
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


## 任务目标

把视频中的原音频替换为新的 audio stream，并保持视频画面不重新编码。

## 适用场景

- 需要给视频替换背景音乐、旁白或测试音频。
- 需要保留原视频画质，只更换音频。
- 需要让 AI 明确选择 video stream 来自输入 0、audio stream 来自输入 1。
- 需要处理音频长度和视频长度不一致的问题。

## 前置探测

先准备视频素材，并确认视频和新音频的时长、codec：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
```

本章先用 FFmpeg 生成一段 220 Hz 的替换音频，避免依赖外部素材。

## 基础命令

生成替换音频：

```bash
ffmpeg -y -f lavfi -i sine=frequency=220:sample_rate=48000 -t 8 -c:a aac examples/input/replacement-audio.m4a
```

用新音频替换原视频音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -shortest examples/output/sample-replaced-audio.mp4
```

`-map 0:v:0` 表示使用第一个输入的视频流，`-map 1:a:0` 表示使用第二个输入的音频流。`-c:v copy` 不重新编码视频，`-c:a aac` 确保输出 MP4 的音频兼容性。

## 常用变体

保留视频并让音频循环到视频结束：

```bash
ffmpeg -y -stream_loop -1 -i examples/input/replacement-audio.m4a -i examples/input/sample-720p.mp4 -map 1:v:0 -map 0:a:0 -c:v copy -c:a aac -shortest examples/output/sample-replaced-audio-loop.mp4
```

替换音频并调整音量：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -af "volume=0.6" -c:a aac -shortest examples/output/sample-replaced-audio-volume.mp4
```

如果新音频已经是 MP4 兼容的 AAC，也可以 copy 音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -c:a copy -shortest examples/output/sample-replaced-audio-copy.mp4
```

使用 `-c:a copy` 前必须先用 `ffprobe` 确认新音频 codec 被目标 container 和播放器支持；不确定时转 AAC 更稳。

## 参数解释

- `-f lavfi -i sine=...`：用 FFmpeg filter 生成测试音频。
- `-t 8`：生成 8 秒音频。
- `-map 0:v:0`：选择第一个输入的第一条 video stream。
- `-map 1:a:0`：选择第二个输入的第一条 audio stream。
- `-c:v copy`：复制视频流，不重新编码，速度快且不损失视频质量。
- `-c:a aac`：把新音频编码为 AAC，适合 MP4 输出。
- `-shortest`：当视频和音频时长不一致时，以较短的 stream 结束输出。
- `-stream_loop -1`：无限循环后面的输入文件，通常和 `-shortest` 配合使用。

## 结果验收

检查替换后的输出：

```bash
ffprobe -hide_banner examples/output/sample-replaced-audio.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/sample-replaced-audio.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/output/sample-replaced-audio.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-replaced-audio.mp4
```

预期输出包含 video stream 和 audio stream。video stream 应保持 H.264、1280x720；audio stream 应为 AAC。播放输出文件，确认听到 220 Hz 测试音，而不是原来的 1000 Hz 音频。

## 常见错误

- 替换后仍是原音频：`-map` 写错，或没有明确选择第二个输入的 audio stream。
- 输出没有音频：第二个输入没有 audio stream，或 `-map 1:a:0` 指向不存在的流。
- 输出时长不符合预期：视频和音频长度不同，且没有理解 `-shortest` 的行为。
- 视频被重新编码：没有使用 `-c:v copy`，或使用了 video filter。
- 音频 copy 后无法播放：目标 MP4 或播放器不支持该 audio codec。改用 `-c:a aac -b:a 128k`。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 替换音频命令。必须先根据 ffprobe 信息分析视频输入的 video/audio stream、新音频输入的 audio codec、sample_rate、channels 和两个输入的 duration。请明确使用 -map 0:v:0 和 -map 1:a:0，保留视频不重新编码，并判断音频应该转 AAC 还是可以 copy。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 video/audio stream、codec、duration 的验收命令；如果新音频比视频短或长，必须说明 -shortest、循环或裁剪策略。

视频输入：<video input>
新音频输入：<audio input>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴两个输入的 format 和 stream 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备视频。
2. 运行 sine 命令生成 `examples/input/replacement-audio.m4a`。
3. 运行替换音频命令生成 `examples/output/sample-replaced-audio.mp4`。
4. 用 `ffprobe` 检查输出包含 video stream 和 audio stream。
5. 播放输出文件，确认音频已经变为 220 Hz 测试音。
