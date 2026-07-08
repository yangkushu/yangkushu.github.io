+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：媒体信息和测试素材'
+++

## 系列目录

- [00 导读、环境和概念](/posts/ffmpeg-learning-00-guide/)
- [01 媒体信息和测试素材](/posts/ffmpeg-learning-01-media-basics/)
- [02 转封装、转码、压缩和分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
- [03 裁剪、拼接和变速](/posts/ffmpeg-learning-03-editing/)
- [04 音频提取和替换](/posts/ffmpeg-learning-04-audio/)
- [05 截图、字幕和水印](/posts/ffmpeg-learning-05-visual-elements/)
- [06 批处理、验收和 AI 提示词](/posts/ffmpeg-learning-06-batch-and-ai/)
- [07 API、生态和流媒体](/posts/ffmpeg-learning-07-advanced/)

---

## 查看媒体信息


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

## 生成测试素材


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
