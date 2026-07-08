+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：调整分辨率'
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

使用 FFmpeg 的 scale filter 调整视频分辨率。本章示例把 720p 输入缩放到宽度 640，并通过 `-2` 自动计算高度，保持 aspect ratio 且保证输出高度为偶数。

## 适用场景

- 需要生成较小分辨率的视频版本，降低文件体积或带宽压力。
- 需要统一输出宽度，同时保持原始画面比例。
- 目标 encoder 或播放器要求视频宽高为偶数。
- 需要让 AI 生成 resize 命令时避免画面拉伸和奇数尺寸错误。

## 前置探测

先准备素材并查看输入分辨率：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/input/sample-720p.mp4
```

确认输入文件存在 video stream。没有视频流的文件不能使用 `scale` 调整画面尺寸。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=640:-2" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-640w.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4
```

`scale=640:-2` 表示输出宽度固定为 640，高度由 FFmpeg 根据输入 aspect ratio 自动计算，并调整为能被 2 整除的偶数。H.264 和 `yuv420p` 等常见组合通常要求或更偏好偶数尺寸。

这里使用 `-c:a copy` 是因为示例素材的 audio codec 是 AAC，且目标输出仍为 MP4。处理自己的文件时，先用 `ffprobe` 查看 audio codec；只有目标 container 和播放环境支持该 codec 时才 copy，否则转 AAC，例如 `-c:a aac -b:a 128k`。

## 常用变体

固定高度为 360，自动计算偶数宽度：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=-2:360" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-360h.mp4
```

固定宽高，可能改变画面比例，不建议用于普通 resize：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=640:360" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-640x360.mp4
```

只在输入宽度大于 640 时缩小，不放大小视频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale='min(640,iw)':-2" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-max-640w.mp4
```

以上输出到 MP4 且使用 `-c:a copy` 的变体都需要先确认 audio codec 兼容性。如果输入音频不是目标环境稳定支持的 codec，改用 `-c:a aac -b:a 128k` 重新编码音频。

## 参数解释

- `-vf "scale=640:-2"`：启用 video filter。`scale` 的第一个值是输出宽度，第二个值是输出高度。
- `640`：固定输出宽度为 640 pixels。
- `-2`：让 FFmpeg 根据 aspect ratio 自动计算高度，并把结果调整为偶数。相比 `-1`，`-2` 更适合 H.264、YUV 4:2:0 等要求偶数尺寸的输出。
- `-c:v libx264`：scale 后必须重新编码 video stream，不能对视频使用 `-c:v copy`。
- `-crf 23 -preset medium`：用 CRF 和 preset 控制重新编码的质量与速度。
- `-c:a copy`：音频不受 resize 影响，可以直接复制，避免音质损失和额外耗时。复制前需要确认 audio codec 被目标 container 和播放环境支持；否则转为 AAC，例如 `-c:a aac -b:a 128k`。
- `ffprobe ... width,height`：只输出视频宽高字段，便于脚本验收。

## 结果验收

运行：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4
```

预期输出包含：

```text
width=640
height=360
```

也可以只读取宽度：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width -of default=nw=1:nk=1 examples/output/sample-640w.mp4
```

预期输出：

```text
640
```

## 常见错误

- `width not divisible by 2` 或 encoder 报尺寸错误：输出宽高为奇数。使用 `-2` 自动得到偶数尺寸，或手动指定偶数宽高。
- 画面被拉伸：固定宽高与原始 aspect ratio 不一致。使用 `scale=目标宽:-2` 或 `scale=-2:目标高`。
- 使用 `-c:v copy` 后失败：resize 会改变 video frame，必须重新编码视频。
- 文件不一定变小：resize 降低像素数量，但 CRF、内容复杂度和 audio bitrate 仍会影响最终体积。
- 字幕或叠加图形位置异常：硬字幕或 overlay 已在画面中会跟随缩放；软字幕需要单独处理。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg resize 命令。目标是把视频宽度缩放到 640，保持 aspect ratio，并确保输出宽高为偶数。视频使用 H.264/libx264 重新编码，音频直接 copy。请解释 scale 中 -2 的作用，并给出 ffprobe 验收命令。

输入文件：<input>
输出文件：<output.mp4>
目标尺寸：宽 640，高度自动
ffprobe 信息：
<粘贴 width、height、codec、pix_fmt 等信息>
限制：不要拉伸画面；不要重新编码音频，除非 container 不支持。
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础 resize 命令生成 `examples/output/sample-640w.mp4`。
3. 运行 `ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4` 检查宽高。
4. 运行只读取宽度的验收命令，确认输出为 `640`。
5. 把 `scale=640:-2` 改为 `scale=-2:360`，观察宽度也会按比例自动计算。
