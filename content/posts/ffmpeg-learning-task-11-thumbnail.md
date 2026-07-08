+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：截取缩略图'
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

使用 FFmpeg 从视频指定时间点截取一帧图片，生成可用于封面、预览或质量检查的 thumbnail。

## 适用场景

- 需要为视频生成封面图或列表预览图。
- 需要从固定时间点抽取画面用于人工审核。
- 需要在自动化流程中验证视频画面是否正常。
- 需要让 AI 生成单帧截图命令，而不是导出连续图片序列。

## 前置探测

先准备素材并确认视频时长大于截图时间点：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

如果 `-ss` 指定的时间超过视频时长，可能生成空文件、失败，或得到非预期帧。

## 基础命令

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 examples/output/thumbnail.jpg
```

这条命令在输入前使用 `-ss` 快速 seek 到 3 秒附近，并输出 1 帧 JPEG 图片。对封面预览来说通常足够；如果需要逐帧级精确截图，可以把 `-ss` 放在 `-i` 后。

## 常用变体

更精确地按时间点解码后取帧：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:03 -frames:v 1 examples/output/thumbnail-accurate.jpg
```

生成指定宽度的缩略图，保持 aspect ratio：

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 -vf "scale=320:-2" examples/output/thumbnail-320w.jpg
```

输出 PNG，减少 JPEG 压缩损失：

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 examples/output/thumbnail.png
```

## 参数解释

- `-ss 00:00:03`：定位到 3 秒时间点。放在输入前速度更快，放在输入后通常更精确。
- `-i examples/input/sample-720p.mp4`：指定输入视频。
- `-frames:v 1`：只输出 1 个 video frame，避免生成图片序列。
- `thumbnail.jpg`：根据扩展名选择图片封装和编码，JPEG 适合通用预览图。
- `-vf "scale=320:-2"`：截图时顺便缩放，`-2` 自动计算偶数高度并保持 aspect ratio。

## 结果验收

运行：

```bash
test -s examples/output/thumbnail.jpg
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/thumbnail.jpg
```

预期 `test -s` 返回成功，`ffprobe` 能看到图片 stream 信息，宽度为 1280，高度为 720。也可以用图片查看器打开，确认画面不是空白或错误帧。

## 常见错误

- 输出很多图片：使用了图片序列格式但没有 `-frames:v 1`。单张缩略图必须限制帧数。
- 截图时间点没有画面：`-ss` 超过视频时长，或输入视频损坏。先用 `ffprobe` 检查 duration。
- 截到的帧不够精确：把 `-ss` 放到 `-i` 后，代价是速度可能变慢。
- 图片太大：添加 `-vf "scale=目标宽:-2"` 生成较小尺寸。
- JPEG 画质不符合预期：可使用 `-q:v` 调整质量，或改用 PNG。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 缩略图命令。必须先根据 ffprobe 信息分析输入 duration、video stream 是否存在、截图时间点是否在时长范围内。目标是在 00:00:03 截取 1 帧 JPEG。请说明 -ss 放在 -i 前后的速度和精确度差异，并给出 ffprobe 或文件大小验收命令。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出文件非空和图片 width/height 的验收命令。

输入文件：<input>
输出图片：<examples/output/output.jpg>
截图时间点：<HH:MM:SS>
目标宽度：<原始尺寸/指定宽度>
ffprobe 信息：
<粘贴 duration、width、height 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础命令生成 `examples/output/thumbnail.jpg`。
3. 运行 `test -s examples/output/thumbnail.jpg` 确认文件非空。
4. 用 `ffprobe` 检查输出图片宽高。
5. 把 `-ss` 改成 `00:00:01` 和 `00:00:05`，对比不同时间点的画面。
