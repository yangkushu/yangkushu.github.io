+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：裁剪片段'
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

使用 FFmpeg 从视频中截取指定时间范围，并理解快速关键帧裁剪与重新编码精确裁剪的差异。

## 适用场景

- 需要从长视频中截取一段用于预览、分享或后续处理。
- 需要快速得到片段，能接受起点落在附近 keyframe。
- 需要更精确的起止时间，愿意用重新编码换取准确性。
- 需要让 AI 明确选择 copy trimming 还是 re-encoding trimming。

## 前置探测

先准备素材并查看时长、编码格式和 keyframe 分布：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

如果要做 stream copy 裁剪，输入 codec、container 和 keyframe 位置都会影响结果。起点不在 keyframe 上时，copy 裁剪通常只能从附近关键帧开始，画面起点可能不完全精确。

## 基础命令

快速裁剪，不重新编码：

```bash
ffmpeg -y -ss 00:00:02 -to 00:00:06 -i examples/input/sample-720p.mp4 -c copy examples/output/sample-trim-copy.mp4
```

更精确裁剪，重新编码：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -c:v libx264 -c:a aac examples/output/sample-trim-accurate.mp4
```

第一条把 `-ss` 放在 `-i` 前，FFmpeg 会先 seek 到接近目标时间的 keyframe，再复制 stream，速度快但起点可能不精确。第二条把 `-ss` 放在输入后，并重新编码输出，通常能获得更准确的截取边界，但耗时更长且会产生一次 lossy re-encoding。

## 常用变体

用起点加持续时长表达裁剪范围：

```bash
ffmpeg -y -ss 00:00:02 -i examples/input/sample-720p.mp4 -t 4 -c copy examples/output/sample-trim-4s-copy.mp4
```

只保留视频，不保留音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -an -c:v libx264 examples/output/sample-trim-video-only.mp4
```

重新编码并指定兼容 pixel format：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k examples/output/sample-trim-compatible.mp4
```

## 参数解释

- `-ss 00:00:02`：设置裁剪起点。放在 `-i` 前偏向快速 seek，放在 `-i` 后偏向精确解码后截取。
- `-to 00:00:06`：设置裁剪终点，表示输出到输入时间线的 6 秒位置。
- `-t 4`：设置输出持续时长。与 `-to` 二选一更清晰，避免同时使用造成理解混乱。
- `-c copy`：复制 video stream 和 audio stream，不重新编码，速度快、质量无损，但依赖 keyframe 和 container 兼容性。
- `-c:v libx264 -c:a aac`：重新编码视频和音频，裁剪边界更可控，但会增加耗时和质量损失。
- `-an`：禁用音频输出。

## 结果验收

检查快速裁剪输出：

```bash
ffprobe -hide_banner examples/output/sample-trim-copy.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-trim-copy.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-copy.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-copy.mp4
```

预期输出时长接近 4 秒，且能看到 video stream 和 audio stream 的 `codec_name` 字段。copy 裁剪可能因为 keyframe、时间戳和 container 行为出现轻微偏差。

检查精确裁剪输出：

```bash
ffprobe -hide_banner examples/output/sample-trim-accurate.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-trim-accurate.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-accurate.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-accurate.mp4
```

预期时长也接近 4 秒，并能确认输出仍包含 video stream 和 audio stream。进一步验收时，应播放开头和结尾，确认画面和声音边界符合预期。

## 常见错误

- 起点不精确：使用 `-c copy` 且起点不在 keyframe。改用重新编码，或接受从附近 keyframe 开始。
- 输出前几帧黑屏或音画不同步：copy 裁剪保留原始压缩帧和时间戳时可能出现。尝试重新编码。
- 把 `-to` 理解为输出时长：`-to` 是结束时间点，`-t` 才是持续时长。
- 裁剪后文件无法播放：container 或 copied stream 不兼容。改用 `-c:v libx264 -c:a aac` 输出 MP4。
- 重新编码质量下降：显式设置 `-crf`、`-preset`、`-b:a`，并根据业务内容验收画质和音质。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 裁剪命令。必须先根据 ffprobe 信息分析输入的 duration、container、video/audio codec 和 keyframe 风险。目标是截取 00:00:02 到 00:00:06 的片段。请分别给出快速 keyframe copy 裁剪和更精确的重新编码裁剪方案，并说明速度、准确性和质量损失的取舍。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 duration 与 video/audio stream codec 的验收命令。

输入文件：<input>
输出文件：<examples/output/output.mp4>
是否要求精确起点：<是/否>
ffprobe 信息：
<粘贴 format、stream、duration、codec 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行快速裁剪命令生成 `examples/output/sample-trim-copy.mp4`。
3. 运行重新编码裁剪命令生成 `examples/output/sample-trim-accurate.mp4`。
4. 用 `ffprobe` 分别检查两个文件的 duration 和 codec。
5. 播放两个输出文件，对比开头是否更接近 2 秒位置，并记录速度与准确性的差异。
