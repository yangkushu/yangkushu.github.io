+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：转码'
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

使用 FFmpeg 将输入媒体重新编码为指定 video codec、audio codec、质量参数和 pixel format。本章示例把输入视频转为兼容性较好的 H.264/AAC MP4，并使用 CRF 控制视频质量。

## 适用场景

- 输入 codec 不被目标平台、浏览器、播放器或剪辑软件支持。
- 需要统一输出为 H.264 video、AAC audio 和 MP4 container。
- 需要调整质量、码率、pixel format 或 encoder preset。
- 需要让 AI 明确指定转码参数，而不是依赖 FFmpeg 默认 encoder。

## 前置探测

先确认输入素材存在，并查看原始 stream：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
```

再确认当前 FFmpeg 支持 `libx264` 和 `aac`：

```bash
ffmpeg -hide_banner -encoders
```

如果 `libx264` 不存在，需要安装包含 x264 support 的 FFmpeg，或改用当前环境可用的 H.264 encoder。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 23 -c:a aac -b:a 128k -pix_fmt yuv420p examples/output/sample-transcoded.mp4
ffprobe -hide_banner examples/output/sample-transcoded.mp4
```

这条命令会重新编码 video stream 和 audio stream。输出文件使用 H.264 video codec、AAC audio codec、CRF 23、`medium` preset 和 `yuv420p` pixel format。

## 常用变体

提高视频质量，通常文件更大：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 18 -c:a aac -b:a 160k -pix_fmt yuv420p examples/output/sample-higher-quality.mp4
```

降低编码耗时，可能降低压缩效率：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset veryfast -crf 23 -c:a aac -b:a 128k -pix_fmt yuv420p examples/output/sample-fast.mp4
```

只转码视频，复制音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p -c:a copy examples/output/sample-video-transcoded.mp4
```

使用 `-c:a copy` 前先用 `ffprobe` 查看 audio codec。只有目标 container 和播放环境支持该 audio codec 时才直接复制；如果不确定，转为 AAC 更稳妥，例如改用 `-c:a aac -b:a 128k`。

## 参数解释

- `-c:v libx264`：使用 x264 encoder 输出 H.264 video codec。H.264 兼容性好，适合 Web、移动端和常见播放器。
- `-preset medium`：控制编码速度和压缩效率。越慢的 preset 通常压缩效率越高，但耗时越长；它不直接代表画质等级。
- `-crf 23`：Constant Rate Factor，控制视频质量。数值越小质量越高、文件通常越大；数值越大质量越低、文件通常越小。x264 常见范围是 18 到 28。
- `-c:a aac`：使用 AAC audio encoder，适合 MP4 container 和多数播放环境。
- `-b:a 128k`：设置 audio bitrate 为 128 kbps。语音可更低，音乐通常需要更高。
- `-pix_fmt yuv420p`：设置 pixel format 为 `yuv420p`，提升浏览器、播放器和移动设备兼容性。
- `examples/output/sample-transcoded.mp4`：输出 MP4 文件。MP4 container 搭配 H.264/AAC 是常见交付组合。

## 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-transcoded.mp4
```

预期结果：

- video stream 的 codec 为 H.264。
- audio stream 的 codec 为 AAC。
- video stream 的 pixel format 为 `yuv420p`。
- 输出文件可被常见播放器播放。

也可以只检查关键字段：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt,width,height -of default=nw=1 examples/output/sample-transcoded.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,bit_rate -of default=nw=1 examples/output/sample-transcoded.mp4
```

## 常见错误

- `Unknown encoder 'libx264'`：当前 FFmpeg 不包含 x264 encoder，需要安装完整版本或换用可用 encoder。
- 输出画面颜色或兼容性异常：缺少 `-pix_fmt yuv420p` 时，可能输出播放器兼容性较差的 pixel format。
- 文件比输入更大：转码不是必然压缩。输入本身可能已高度压缩，或 CRF 数值较低导致质量和体积更高。
- 编码速度太慢：使用更快的 `preset`，例如 `fast` 或 `veryfast`，但压缩效率可能下降。
- 音频码率不符合预期：部分 AAC encoder 对低码率或多声道输入有约束，先用 `ffprobe` 查看输入声道和采样率。

## AI 提示词模板

```text
请基于以下 ffprobe 信息生成 FFmpeg 转码命令。目标输出为 MP4，video codec 使用 H.264/libx264，audio codec 使用 AAC，pixel format 使用 yuv420p。请解释 CRF、preset、audio bitrate 和 pixel format 的作用，并给出验收命令。

输入文件：<input>
输出文件：<output.mp4>
质量偏好：<质量优先/体积优先/速度优先>
ffprobe 信息：
<粘贴输入文件信息>
限制：不要覆盖源文件；如需改变分辨率请单独说明。
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础转码命令生成 `examples/output/sample-transcoded.mp4`。
3. 运行 `ffprobe -hide_banner examples/output/sample-transcoded.mp4` 查看输出 stream。
4. 使用 `ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt -of default=nw=1 examples/output/sample-transcoded.mp4` 确认 video codec 和 pixel format。
5. 调整 `-crf` 为 18 和 28 各运行一次，对比画质、文件大小和编码耗时。
