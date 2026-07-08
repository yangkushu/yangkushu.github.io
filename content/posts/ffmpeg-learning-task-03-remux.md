+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：转封装'
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

使用 FFmpeg 将媒体文件从一种 container 转换为另一种 container，同时保持已有 codec stream 不变。本章示例把 MP4 转成 MKV，只改变封装格式，不重新编码 video stream 和 audio stream。

## 适用场景

- 输入文件的 video codec 和 audio codec 已经符合目标要求，只需要换 container。
- 需要把 MP4、MOV、MKV 等格式之间快速转换，避免重新编码带来的画质损失。
- 需要修复或重建 container metadata，但不想改变原始音视频编码。
- 需要让 AI 明确区分 remux 和 transcode，避免不必要的重编码。

## 前置探测

先确认输入素材存在，并查看原始 container 与 stream 信息：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
```

如果使用自己的文件，先确认目标 container 支持输入文件中的 codec。例如 MKV 通常支持范围较广，MP4 对 subtitle、某些 audio codec 和 metadata 的限制更明显。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c copy examples/output/sample-remux.mkv
ffprobe -hide_banner examples/output/sample-remux.mkv
```

`-c copy` 表示 stream copy：直接复制输入中的编码流，不经过 decoder 和 encoder。输出文件的 container 从 MP4 变为 MKV，但 video codec、audio codec、分辨率、帧率、采样率等 stream 参数保持不变。

## 常用变体

把 MKV 转为 MP4，前提是其中的 codec 被 MP4 container 支持：

```bash
ffmpeg -y -i input.mkv -c copy output.mp4
```

只保留 video stream 和 audio stream，丢弃 subtitle 或 data stream：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -map 0:v:0 -map 0:a:0 -c copy examples/output/sample-remux-video-audio.mkv
```

复制所有 stream：

```bash
ffmpeg -y -i input.mkv -map 0 -c copy output.mkv
```

## 参数解释

- `-y`：允许覆盖已有输出文件，适合可重复运行实验。处理真实文件时可以去掉，避免误覆盖。
- `-i examples/input/sample-720p.mp4`：指定输入文件。
- `-c copy`：对所有选中的 stream 使用 copy 模式，不重新编码。也可以写成 `-c:v copy -c:a copy`，分别指定 video 和 audio。
- `examples/output/sample-remux.mkv`：输出路径。FFmpeg 通常根据扩展名选择 muxer，这里使用 Matroska/MKV container。
- `ffprobe -hide_banner`：查看输出文件的 container 和 stream 信息，用来确认 container 已改变，codec stream 未改变。
- `-map`：手动选择要写入输出文件的 stream。默认映射通常会选择每类 stream 中的一条，复杂文件建议显式指定。

## 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-remux.mkv
```

预期结果：

- 输出文件的 format/container 显示为 Matroska 或 WebM 相关格式。
- video stream 仍为原输入的 H.264，不应出现重新编码后的 codec 变化。
- audio stream 仍为原输入的 AAC，不应出现重新编码后的 codec 变化。
- 命令执行速度通常明显快于 transcode，因为没有重新编码。

## 常见错误

- `Could not write header`：目标 container 不支持某个输入 stream，需要换 container、去掉该 stream，或改用 transcode。
- 输出文件无法播放：可能是播放器对 container 或 codec 组合支持不足。先用 `ffprobe` 确认 stream，再换播放器或改用更兼容的 container。
- 文件大小变化但画质没变：container metadata、索引和封装开销不同会导致大小变化，这不代表发生了重新编码。
- 以为转封装能压缩文件：remux 不改变 codec 数据，通常不能显著降低体积。需要压缩时使用 CRF、bitrate 或 scale 等重新编码参数。
- 没有音频或字幕：默认 stream selection 可能未选择所有 stream，复杂文件使用 `-map 0` 或明确的 `-map` 参数。

## AI 提示词模板

```text
请为以下输入文件生成 FFmpeg 转封装命令。目标是只改变 container，不重新编码 video/audio stream。请使用 stream copy，并给出 ffprobe 验收命令。

输入文件：<input>
目标 container：<例如 mkv 或 mp4>
ffprobe 信息：
<粘贴 ffprobe -hide_banner 或 -show_streams 输出>
限制：不要改变 codec、分辨率、帧率、采样率；如果目标 container 不支持某个 stream，请说明风险和替代方案。
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备测试素材。
2. 运行 `ffprobe -hide_banner examples/input/sample-720p.mp4` 记录输入 container 和 codec。
3. 运行 `ffmpeg -y -i examples/input/sample-720p.mp4 -c copy examples/output/sample-remux.mkv` 生成 MKV。
4. 运行 `ffprobe -hide_banner examples/output/sample-remux.mkv` 检查输出 container 已改变。
5. 对比输入和输出的 video/audio codec，确认 stream 仍为 H.264/AAC。
