+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：压缩视频'
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

使用 FFmpeg 通过重新编码降低视频文件体积，并理解 CRF、preset、audio bitrate 和 `+faststart` 对输出文件的影响。本章示例使用较高 CRF 和较低 audio bitrate 生成更小的 MP4。

## 适用场景

- 需要降低视频上传、分发或归档成本。
- 需要在质量可接受的前提下缩小文件体积。
- 输入文件过大，需要生成便于分享或网页播放的版本。
- 需要向 AI 明确质量与体积的取舍，而不是只要求“压缩到最小”。

## 前置探测

先准备素材并查看输入文件大小：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ls -lh examples/input/sample-720p.mp4
ffprobe -hide_banner examples/input/sample-720p.mp4
```

压缩前建议确认输入的 duration、resolution、codec 和 bitrate。相同压缩参数在不同内容上的效果不同，运动复杂、噪点多或细节丰富的视频更难压缩。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 28 -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart examples/output/sample-compressed.mp4
ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4
```

这条命令会重新编码 video stream 和 audio stream。`-crf 28` 会比常见的 `-crf 23` 更偏向小体积，但画质也更容易下降。`-pix_fmt yuv420p` 面向 MP4/H.264/AAC/Web 播放场景，能提升浏览器、移动端和常见播放器兼容性。

## 常用变体

质量更高、文件通常更大：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p -c:a aac -b:a 128k -movflags +faststart examples/output/sample-compressed-crf23.mp4
```

体积更小、质量损失更明显：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 32 -pix_fmt yuv420p -c:a aac -b:a 64k -movflags +faststart examples/output/sample-compressed-crf32.mp4
```

先缩小分辨率再压缩：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=854:-2" -c:v libx264 -preset slow -crf 26 -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart examples/output/sample-compressed-480p.mp4
```

## 参数解释

- `-c:v libx264`：使用 H.264 encoder 重新编码视频。
- `-preset slow`：使用较慢的编码 preset。通常比 `medium` 或 `fast` 获得更好的压缩效率，但耗时更长。
- `-crf 28`：CRF 数值越高，视频质量越低、文件通常越小。压缩不是无损过程，较高 CRF 会丢弃更多视觉细节。
- `-pix_fmt yuv420p`：输出兼容性更好的 pixel format，适合 MP4/H.264/AAC/Web 交付。
- `-c:a aac -b:a 96k`：把音频编码为 AAC，并设置较低 audio bitrate，进一步降低体积。音乐或复杂音频可能需要更高码率。
- `-movflags +faststart`：把 MP4 的 moov atom 移到文件开头，便于网页或远程播放时更快开始播放。
- `ls -lh`：对比输入和输出文件大小，快速判断压缩是否生效。

## 结果验收

运行：

```bash
ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4
ffprobe -hide_banner examples/output/sample-compressed.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt,width,height,bit_rate -of default=nw=1 examples/output/sample-compressed.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,bit_rate -of default=nw=1 examples/output/sample-compressed.mp4
```

预期结果：

- 输出文件通常小于输入文件，但这不是对所有内容的绝对保证。
- video stream 的 `codec_name` 为 `h264`，`pix_fmt` 为 `yuv420p`，并能看到 `width`、`height` 和 `bit_rate` 字段。
- audio stream 的 `codec_name` 为 `aac`，并能看到 `bit_rate` 字段。
- 播放输出文件时，画面可能出现更明显的压缩痕迹，这是较小体积的代价。

`ffprobe -hide_banner` 可以确认输出 container 和 stream 概览，但不能直接证明 `+faststart` 已经把 moov atom 前置。`+faststart` 更适合通过 FFmpeg 编码日志中的 `moving the moov atom to the beginning of the file`、专门工具，或针对 MP4 box 位置的检查来确认。

进一步验收时，不只看文件大小和字段，也要人工抽查画面细节、字幕边缘、快速运动片段和音频质量。

## 常见错误

- 文件没有变小：输入已经很小或很高压缩，或者 CRF 过低、audio bitrate 过高。可以提高 CRF、降低分辨率或降低音频码率。
- 画质下降明显：CRF 太高，或输入内容本身细节多、运动快。降低 CRF，例如从 28 改为 23 或 24。
- 编码时间太长：`-preset slow` 更耗时。可改用 `medium` 或 `fast`，但同等 CRF 下文件可能更大。
- 只看码率不看内容：不同画面复杂度对压缩效率影响很大。压缩参数需要结合实际视频验收。
- 误以为压缩不损质量：使用 H.264/AAC 重新编码通常是 lossy compression，体积变小往往意味着部分细节被舍弃。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 压缩命令。目标是在可接受画质下减小文件体积，输出为 MP4/H.264/AAC，并使用 +faststart。请说明 CRF 与文件大小、画质之间的取舍，并给出文件大小和 ffprobe 验收命令。

输入文件：<input>
输出文件：<output.mp4>
优先级：<小体积/画质/编码速度>
可接受限制：<最大文件大小、目标分辨率、音频质量要求>
ffprobe 信息：
<粘贴输入信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础压缩命令生成 `examples/output/sample-compressed.mp4`。
3. 运行 `ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4` 对比文件大小。
4. 分别用 `-crf 23`、`-crf 28`、`-crf 32` 输出三个版本。
5. 对比三个版本的文件大小和画面质量，记录自己能接受的 CRF 范围。
