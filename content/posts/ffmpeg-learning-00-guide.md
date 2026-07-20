+++
date = '2026-07-13T12:00:07+08:00'
draft = false
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程（一）：入门、环境与核心概念'
+++

本篇建立整套教程的学习方式，完成环境准备，理解核心音视频概念，并生成后续实验需要的测试素材。

## 系列目录

1. [FFmpeg 视频处理教程（一）：入门、环境与核心概念](/posts/ffmpeg-learning-00-guide/)
2. [FFmpeg 视频处理教程（二）：媒体探测与封装转换](/posts/ffmpeg-learning-01-media-basics/)
3. [FFmpeg 视频处理教程（三）：转码、压缩与分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
4. [FFmpeg 视频处理教程（四）：裁剪、拼接与变速](/posts/ffmpeg-learning-03-editing/)
5. [FFmpeg 视频处理教程（五）：提取和替换音频](/posts/ffmpeg-learning-04-audio/)
6. [FFmpeg 视频处理教程（六）：截图、字幕与水印](/posts/ffmpeg-learning-05-visual-elements/)
7. [FFmpeg 视频处理教程（七）：批处理、验收与 AI 辅助](/posts/ffmpeg-learning-06-batch-and-ai/)
8. [FFmpeg 视频处理教程（八）：API、生态与流媒体](/posts/ffmpeg-learning-07-advanced/)

## 教程总览

### 定位

本教程关注视频处理实践，而不是 FFmpeg 参数百科。每个任务章节都从一个真实问题出发，给出可执行命令、参数解释、输出验收和 AI 辅助模板。

### 为什么先探测再处理

FFmpeg 命令依赖输入文件的容器、编码、流、时长、分辨率、帧率和音频参数。直接猜参数容易导致音画不同步、编码失败、输出不可播放或体积异常。

### 标准处理流程

1. 确认输入文件路径。
2. 使用 `ffprobe` 获取媒体信息。
3. 选择最小可用命令。
4. 根据质量、速度、兼容性调整参数。
5. 输出到新文件。
6. 验收输出文件。

### AI 辅助原则

AI 生成命令前必须先知道输入信息。AI 输出命令时必须说明输入、输出、是否覆盖、关键参数含义和验收方式。

## 环境准备

开始学习 FFmpeg 教程之前，先在本机安装好 FFmpeg 并约定工作目录，后续所有练习都基于这套环境。

### 安装 FFmpeg

#### macOS

推荐使用 Homebrew：

```bash
brew install ffmpeg
```

#### Windows

推荐两种方式：

- 使用 WSL，然后在 Linux 环境中安装 FFmpeg。
- 使用 Git Bash 执行本教程中的 sh/bash 脚本，并使用 Windows 版 FFmpeg。

#### 检查安装

安装完成后，运行以下命令确认各工具是否可用：

```bash
ffmpeg -version
ffprobe -version
ffplay -version
```

即使 `ffplay` 不可用，也不影响大部分教程任务。它主要用于快速预览媒体文件。

### 路径和覆盖规则

本教程中所有练习都使用统一的目录结构，避免和原始文件混淆：

- 输入文件放在 `examples/input/`。
- 输出文件放在 `examples/output/`。
- 不直接覆盖源文件。
- 需要覆盖输出文件时，显式使用 `-y`。
- 需要避免覆盖时，使用 `-n`。

环境准备好后，就可以进入下一节了解 FFmpeg 的基本概念了。

## 必要音视频概念

读完这篇文章，你再看 FFmpeg 命令时能辨认出它在操作容器、编码格式、流、码率、分辨率、帧率和像素格式中的哪一个维度。

### 核心概念

**Container（封装格式）**

用于承载多个流和元数据，包括视频、音频、字幕以及章节信息。常见封装格式有 MP4、MOV、MKV。

比如把 MKV 重新封装为 MP4 时，如果不希望重新编码，可以用 `-c copy` 直接复制各个流（stream）。但要注意，不是所有编码格式都能放进任意容器格式，某些组合下 `-c copy` 会报错。

**Stream（媒体流）**

一个媒体文件可以包含多个流（stream）：一个视频流、多个音频流、多个字幕流。容器承载这些流，每个流的内容由对应的 codec 编码。

想保留原视频但换掉音频时，你实际上是在操作不同的流。`-map 0:v:0`、`-map 0:a:1` 就用来选择保留或丢弃哪些流。

**Codec（编码格式）**

定义音视频数据如何被压缩和还原，例如 H.264、H.265、AAC、Opus。

对同一个视频，可以用不同 codec 重新编码（改变编码格式），也可以在换容器时不重新编码（保留原 codec）。`-c:v libx264` 指定视频编码器，`-c:a aac` 指定音频编码器。

**Bitrate（码率）**

码率直接影响文件体积，也会影响画质。编码耗时更多取决于 codec、编码器参数、分辨率和 preset。视频压缩经常需要在码率、质量和编码速度之间取舍。

`-b:v 2M` 设置视频目标码率，`-b:a 128k` 设置音频目标码率。

**Frame Rate（帧率）**

常见值有 24、25、30、60 FPS。修改或重采样帧率可能影响流畅度、体积和音画同步。`-r 30` 设置输出帧率。

**Resolution（分辨率）**

例如 1920x1080。调整分辨率通常使用 `scale` 滤镜。`-vf scale=1280:-2` 把宽度缩放到 1280，高度按比例自动计算并保持偶数。

**Pixel Format（像素格式）**

描述像素存储方式。为兼容大多数播放器，H.264 MP4 输出通常使用 `yuv420p`，但它不是唯一可用的格式。`-pix_fmt yuv420p` 指定输出像素格式。

**Keyframe 和 GOP（关键帧和画面组）**

关键帧是可以独立解码的帧，GOP（Group of Pictures）是两个关键帧之间的一组画面。关键帧间隔主要影响拖动进度条的响应速度和流媒体的分片效率。

`-g 48` 设置 GOP 大小，通常表示关键帧最大间隔约为 48 帧；编码器仍可能因为场景切换插入额外关键帧。

### 常见误解

- **MP4 是编码格式？** 不是。MP4 是容器格式（container），它不决定数据如何压缩。一个 MP4 文件里的视频可能使用 H.264、H.265 或 AV1 等编码格式。
- **高码率一定等于高画质？** 不一定。画质还取决于 codec 类型、编码参数和源素材本身。一个高效 codec 在同样码率下可以产出比低效 codec 更好的画质。
- **`-c copy` 总是能用？** 不是。源文件的编码格式必须兼容目标容器格式，否则会报错。即使容器本身支持某种编码，目标播放器或平台也可能支持不好；例如 MP4 中的 Opus 音频在一些播放环境中兼容性有限。

## 生成测试素材

### 任务目标

生成可复现的视频和音频素材，供后续章节实验使用。

### 适用场景

- 验证转封装、转码、压缩、裁剪、拼接等命令的效果时，需要统一的输入素材做对比。
- 没有真实业务素材时，用 FFmpeg 的 lavfi 虚拟输入快速生成测试视频。
- 需要让 AI 或 agent 在参数可控的素材上复现问题，避免依赖本地私有文件。

### 前置探测

确认 FFmpeg 已安装且可用：

```bash
ffmpeg -version
ffprobe -version
```

如果命令不存在，需要先安装 FFmpeg，并确认 `ffmpeg` 与 `ffprobe` 在 `PATH` 中。

### 基础命令

```bash
./scripts/generate-samples.sh
```

脚本会创建 `examples/input/` 和 `examples/output/`，并生成三个 MP4 测试文件：

- `examples/input/sample-720p.mp4`
- `examples/input/sample-360p.mp4`
- `examples/input/title-card.mp4`

如果当前 FFmpeg 构建不包含 drawtext filter，脚本会生成不带文字的纯色标题卡。它仍保留拼接实验所需的分辨率、帧率和音频参数。

### 常用变体

脚本生成的是默认参数的文件。如果需要自定义时长、分辨率或音频参数，可以手动运行以下命令。

生成更短的素材时，调整 `-t` 参数：

```bash
ffmpeg -y \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -t 3 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  examples/input/sample-720p-short.mp4
```

生成不同分辨率的素材时，调整 `testsrc` 或 `testsrc2` 的 `size` 参数：

```bash
ffmpeg -y \
  -f lavfi -i testsrc2=size=1920x1080:rate=30 \
  -f lavfi -i sine=frequency=440:sample_rate=48000 \
  -t 5 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  examples/input/sample-1080p.mp4
```

### 参数解释

- `-f lavfi`：使用 libavfilter 虚拟输入，不依赖外部媒体文件。
- `testsrc` / `testsrc2`：生成测试图形视频流，便于观察画面尺寸、帧率和编码结果。
- `sine=frequency=...`：生成指定频率的正弦波音频。
- `-t`：限制输出时长。
- `-c:v libx264`：使用 H.264 视频编码器。
- `-pix_fmt yuv420p`：使用兼容性更好的 pixel format。
- `-c:a aac -b:a ...`：使用 AAC 音频编码器，并设置音频码率。
- `drawtext`：在纯色画面上绘制文字。这个 filter 依赖 FFmpeg 启用 drawtext 支持，运行环境还需要能通过 fontconfig 或默认字体配置找到可用字体。filter 不可用时，脚本会退回到不带文字的纯色标题卡。

### 结果验收

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

预期：时长约 8 秒，视频流 1280×720，帧率 30 FPS。

### 常见错误

- `ffmpeg: command not found`：FFmpeg 未安装，或可执行文件不在 `PATH` 中。
- `Unknown encoder 'libx264'`：当前 FFmpeg 构建不包含 libx264。需要安装带 H.264 编码器的版本，或改用其他可用的视频编码器。
- 脚本提示 drawtext 不可用：当前 FFmpeg 构建不包含 drawtext filter，脚本会继续生成不带文字的纯色标题卡。如果实验必须显示文字，需要安装启用 freetype 支持的版本。
- 标题卡提示找不到字体：运行环境可能缺少可用字体或 fontconfig 配置。可以按本机字体路径改写 `drawtext` 的 `fontfile=` 参数。
- 输出文件已存在但内容不符合预期：直接重新运行脚本（脚本使用 `-y` 会自动覆盖已有文件），然后用 `ffprobe` 验收。

### AI 提示词模板

请基于 FFmpeg 的 lavfi 输入生成一个 8 秒、1280x720、30 FPS、带 1kHz 音频的 H.264/AAC MP4 测试文件。输出到 examples/input/，不要覆盖源素材以外的文件。

### 可复现实验

1. 运行 `./scripts/generate-samples.sh` 生成素材。
2. 使用 `ffprobe` 检查三个输出文件是否都包含视频流和音频流。
3. 记录 `sample-720p.mp4` 的 duration、width、height、r_frame_rate。
4. 删除生成的 MP4 文件，重新运行脚本，确认两次运行的输出参数一致。

## 测试素材生成脚本

文章中的脚本命令对应以下完整内容：

```sh
#!/usr/bin/env sh
set -eu

SCRIPT_DIR=$(CDPATH= cd "$(dirname "$0")" && pwd)
REPO_ROOT=$(CDPATH= cd "$SCRIPT_DIR/.." && pwd)
INPUT_DIR="$REPO_ROOT/examples/input"
OUTPUT_DIR="$REPO_ROOT/examples/output"

mkdir -p "$INPUT_DIR" "$OUTPUT_DIR"

ffmpeg -y \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=48000 \
  -t 8 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  "$INPUT_DIR/sample-720p.mp4"

ffmpeg -y \
  -f lavfi -i testsrc2=size=640x360:rate=24 \
  -f lavfi -i sine=frequency=440:sample_rate=44100 \
  -t 6 \
  -c:v libx264 -pix_fmt yuv420p \
  -c:a aac -b:a 96k \
  "$INPUT_DIR/sample-360p.mp4"

if ffmpeg -hide_banner -filters 2>/dev/null | grep -q ' drawtext '; then
  ffmpeg -y \
    -f lavfi -i color=c=black:size=1280x720:rate=30 \
    -f lavfi -i sine=frequency=660:sample_rate=48000 \
    -t 5 \
    -vf "drawtext=text='FFmpeg Tutorial':fontcolor=white:fontsize=48:x=(w-text_w)/2:y=(h-text_h)/2" \
    -c:v libx264 -pix_fmt yuv420p \
    -c:a aac -b:a 128k \
    "$INPUT_DIR/title-card.mp4"
else
  printf '%s\n' 'drawtext is unavailable; generating a plain title card.' >&2
  ffmpeg -y \
    -f lavfi -i color=c=black:size=1280x720:rate=30 \
    -f lavfi -i sine=frequency=660:sample_rate=48000 \
    -t 5 \
    -c:v libx264 -pix_fmt yuv420p \
    -c:a aac -b:a 128k \
    "$INPUT_DIR/title-card.mp4"
fi
```
