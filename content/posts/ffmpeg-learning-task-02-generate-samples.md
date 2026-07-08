+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：生成测试素材'
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

生成可复现的视频和音频素材，用于后续章节实验。

## 适用场景

- 需要统一的输入素材来验证转封装、转码、压缩、裁剪、拼接等命令。
- 需要在没有真实业务素材时，用 FFmpeg 的 lavfi 输入快速生成测试视频。
- 需要让 AI 或 agent 在可控素材上复现问题，避免依赖本地私有文件。

## 前置探测

先确认 FFmpeg 可用：

```bash
ffmpeg -version
ffprobe -version
```

如果命令不存在，需要先安装 FFmpeg，并确认 `ffmpeg` 与 `ffprobe` 在 `PATH` 中。

## 基础命令

```bash
./scripts/generate-samples.sh
```

脚本会创建 `examples/input/` 和 `examples/output/`，并生成三个 MP4 测试文件：

- `examples/input/sample-720p.mp4`
- `examples/input/sample-360p.mp4`
- `examples/input/title-card.mp4`

## 常用变体

生成更短的素材时，可以调整 `-t` 参数：

```bash
ffmpeg -y \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -t 3 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  examples/input/sample-720p-short.mp4
```

生成不同分辨率素材时，可以调整 `testsrc` 或 `testsrc2` 的 `size`：

```bash
ffmpeg -y \
  -f lavfi -i testsrc2=size=1920x1080:rate=30 \
  -f lavfi -i sine=frequency=440:sample_rate=48000 \
  -t 5 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  examples/input/sample-1080p.mp4
```

## 参数解释

- `-f lavfi`：使用 libavfilter 虚拟输入，不依赖外部媒体文件。
- `testsrc` / `testsrc2`：生成测试图案视频，便于观察画面尺寸、帧率和编码结果。
- `sine=frequency=...`：生成指定频率的正弦波音频。
- `-t`：限制输出时长。
- `-c:v libx264`：使用 H.264 video encoder。
- `-pix_fmt yuv420p`：使用兼容性更好的 pixel format。
- `-c:a aac -b:a ...`：使用 AAC audio encoder，并设置 audio bitrate。
- `drawtext`：在纯色画面上绘制文字，用于生成标题卡。这个 filter 依赖 FFmpeg 启用 drawtext 支持，运行环境还需要能通过 fontconfig 或默认字体配置找到可用字体。

## 结果验收

```bash
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -hide_banner examples/input/sample-360p.mp4
ffprobe -hide_banner examples/input/title-card.mp4
```

进一步检查 `sample-720p.mp4` 的时长和视频流参数：

```bash
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

预期结果是时长接近 8 秒，视频流宽度为 1280，高度为 720，帧率为 30 FPS。

## 常见错误

- `ffmpeg: command not found`：FFmpeg 未安装，或可执行文件不在 `PATH` 中。
- `Unknown encoder 'libx264'`：当前 FFmpeg 构建不包含 libx264，需要安装带 H.264 encoder 的版本，或改用可用的视频编码器。
- `No such filter: 'drawtext'`：当前 FFmpeg 构建不包含 drawtext filter，需要安装包含 freetype 支持的版本。
- 标题卡生成失败或提示找不到字体：运行环境可能缺少可用字体或 fontconfig 配置。可以先使用 `sample-720p.mp4` 和 `sample-360p.mp4` 继续后续基础实验，或按本机字体路径改写 `drawtext` 的 `fontfile=` 参数。
- 输出文件已经存在但内容不符合预期：脚本使用 `-y` 覆盖目标测试素材，重新运行脚本后再用 `ffprobe` 验收。

## AI 提示词模板

请基于 FFmpeg 的 lavfi 输入生成一个 8 秒、1280x720、30 FPS、带 1kHz 音频的 H.264/AAC MP4 测试文件。输出到 examples/input/，不要覆盖源素材以外的文件。

## 可复现实验

1. 运行 `./scripts/generate-samples.sh` 生成素材。
2. 使用 `ffprobe` 检查三个输出文件是否都有 video stream 和 audio stream。
3. 记录 `sample-720p.mp4` 的 duration、width、height、r_frame_rate。
4. 删除生成的 MP4 文件后重新运行脚本，确认输出参数保持一致。
