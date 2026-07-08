+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：查看媒体信息'
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

使用 `ffprobe` 查看媒体文件的 container、video stream 和 audio stream 信息，为后续转封装、转码、压缩、裁剪、拼接等任务准备可靠输入参数。

## 适用场景

- 不确定输入文件是否包含 video stream 或 audio stream。
- 需要确认编码格式、分辨率、帧率、pixel format、采样率、声道数和 bitrate。
- 需要把媒体信息提供给 AI，让 AI 基于真实输入生成 FFmpeg 命令。
- 需要在脚本中读取稳定字段，避免从完整日志中手工复制信息。

## 前置探测

先确认 `ffprobe` 可用，并确认测试素材存在：

```bash
ffprobe -version
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
```

如果使用自己的文件，把下面示例中的 `examples/input/sample-720p.mp4` 替换为实际路径。路径包含空格时需要加引号。

## 基础命令

快速查看媒体概览：

```bash
ffprobe -hide_banner examples/input/sample-720p.mp4
```

输出完整的 container 和 stream 结构，适合复制给 AI 或保存为脚本输入：

```bash
ffprobe -v error -show_format -show_streams examples/input/sample-720p.mp4
```

只查看第一个 video stream：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt -of default=nw=1 examples/input/sample-720p.mp4
```

只查看第一个 audio stream：

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/input/sample-720p.mp4
```

## 常用变体

查看文件时长：

```bash
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
```

查看第一个视频流的编码、尺寸和帧率：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

查看第一个音频流的编码、采样率和声道数：

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels -of default=nw=1 examples/input/sample-720p.mp4
```

输出 JSON，适合被 Python、Node.js 或其他自动化工具解析：

```bash
ffprobe -v error -show_format -show_streams -of json examples/input/sample-720p.mp4
```

## 参数解释

- `-hide_banner`：隐藏 FFmpeg 构建版本和编译配置，让人工查看时更专注于输入文件本身。适合临时诊断和教程演示；如果需要排查 FFmpeg 构建能力，再去掉它查看完整 banner。
- `-v error`：只输出 error 级别日志，减少噪声。脚本中使用它可以让 stdout 更稳定，便于解析字段，也能避免 warning、banner 和进度信息混入结果。
- `-show_format`：显示 container 层信息，例如 format name、duration、size 和 overall bitrate。
- `-show_streams`：显示每条 stream 的详细信息，包括 video、audio、subtitle 等。
- `-select_streams v:0`：只选择第一个 video stream。`v` 表示 video，`:0` 表示同类型 stream 中的第一条。
- `-select_streams a:0`：只选择第一个 audio stream。`a` 表示 audio。
- `-show_entries stream=...`：只输出指定 stream 字段，适合把结果作为 AI 或脚本的输入。
- `-of default=nw=1`：使用 default 输出格式，并关闭 section wrapper，让结果成为 `key=value` 形式。
- `-of default=nw=1:nk=1`：进一步隐藏 key，只输出 value，适合读取单个字段。

## 结果验收

运行视频流探测命令：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt -of default=nw=1 examples/input/sample-720p.mp4
```

预期输出至少包含：

```text
codec_name=h264
width=1280
height=720
```

运行音频流探测命令：

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/input/sample-720p.mp4
```

预期能看到 audio codec、sample rate 和 channels。不同 FFmpeg 版本或编码设置下，`bit_rate` 可能为空，脚本不要只依赖这个字段判断音频是否存在。

## 常见错误

- `ffprobe: command not found`：FFmpeg 未安装，或 `ffprobe` 不在 `PATH` 中。
- `No such file or directory`：输入路径错误，或测试素材还没有生成。先运行 `./scripts/generate-samples.sh`。
- 输出为空：可能选择了不存在的 stream，例如无音频文件上使用 `-select_streams a:0`。
- 字段值是分数，例如 `r_frame_rate=30/1`：这是正常输出。脚本需要按 rational number 处理，不要直接当作整数。
- `bit_rate=N/A` 或字段缺失：部分容器或 stream 不保存准确 bitrate，需要结合 file size、duration 或重新编码参数判断。
- Windows 路径解析失败：建议在 WSL 或 Git Bash 中使用相对路径，或给包含空格的路径加引号。

## AI 提示词模板

请先根据以下 ffprobe 信息生成 FFmpeg 命令，不要猜测输入参数。你需要关注这些字段：输入文件路径、container format、duration、video codec、width、height、frame rate、pixel format、audio codec、sample rate、channels、audio bitrate，以及目标输出格式、是否允许覆盖输出文件、质量优先还是速度优先。

```text
任务：<描述你要做的视频处理，例如压缩、转码、裁剪、提取音频>
输入文件：examples/input/sample-720p.mp4
ffprobe 输出：
<粘贴 ffprobe -v error -show_format -show_streams 的结果，或分别粘贴 video/audio stream 字段>
输出要求：<目标路径、格式、清晰度、体积、兼容性要求>
限制：不要覆盖源文件；命令需要说明参数含义；给出验收命令。
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备测试素材。
2. 运行 `ffprobe -hide_banner examples/input/sample-720p.mp4`，观察完整概览适合人工阅读。
3. 运行 `ffprobe -v error -show_format -show_streams examples/input/sample-720p.mp4`，保存结构化字段。
4. 分别运行 video stream 和 audio stream 探测命令，确认可以独立获取视频和音频参数。
5. 把探测结果交给 AI，再要求它生成后续 FFmpeg 命令，并检查它是否使用了真实字段而不是猜测。
