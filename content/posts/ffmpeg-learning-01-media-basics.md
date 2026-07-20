+++
date = '2026-07-13T12:00:01+08:00'
draft = false
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程（二）：媒体探测与封装转换'
+++

本篇从 ffprobe 探测开始，学习识别容器、编码格式和流，并在兼容边界内完成不重新编码的封装转换。

## 系列目录

1. [FFmpeg 视频处理教程（一）：入门、环境与核心概念](/posts/ffmpeg-learning-00-guide/)
2. [FFmpeg 视频处理教程（二）：媒体探测与封装转换](/posts/ffmpeg-learning-01-media-basics/)
3. [FFmpeg 视频处理教程（三）：转码、压缩与分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
4. [FFmpeg 视频处理教程（四）：裁剪、拼接与变速](/posts/ffmpeg-learning-03-editing/)
5. [FFmpeg 视频处理教程（五）：提取和替换音频](/posts/ffmpeg-learning-04-audio/)
6. [FFmpeg 视频处理教程（六）：截图、字幕与水印](/posts/ffmpeg-learning-05-visual-elements/)
7. [FFmpeg 视频处理教程（七）：批处理、验收与 AI 辅助](/posts/ffmpeg-learning-06-batch-and-ai/)
8. [FFmpeg 视频处理教程（八）：API、生态与流媒体](/posts/ffmpeg-learning-07-advanced/)

## 查看媒体信息

### 任务目标

使用 `ffprobe` 查看媒体文件的 container、video stream 和 audio stream 信息，为后续转封装、转码、压缩、裁剪、拼接等任务准备可靠输入参数。

### 适用场景

- 不确定输入文件是否包含 video stream 或 audio stream。
- 需要确认编码格式、分辨率、帧率、pixel format、采样率、声道数和 bitrate。
- 需要把媒体信息提供给 AI，让 AI 基于真实输入生成 FFmpeg 命令。
- 需要在脚本中读取稳定字段，避免从完整日志中手工复制信息。

### 前置探测

先确认 `ffprobe` 可用且测试素材存在：

```bash
ffprobe -version
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
```

如果使用自己的文件，把下面示例中的 `examples/input/sample-720p.mp4` 替换为实际路径。路径包含空格时需要加引号。

### 基础命令

确认环境就绪后，从最常用的探测命令开始。

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

### 常用变体

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

输出 JSON，方便 Python、Node.js 等工具直接解析：

```bash
ffprobe -v error -show_format -show_streams -of json examples/input/sample-720p.mp4
```

### 参数解释

- `-hide_banner`：隐藏 FFmpeg 构建版本和编译配置，让人工查看时聚焦输入文件本身。适合临时诊断和教程演示；如需排查 FFmpeg 构建能力，再去掉它查看完整 banner。
- `-v error`：只输出 error 级别日志，减少噪声。脚本中使用该选项可以让 stdout 更稳定，便于解析字段，也能避免 warning、banner 和进度信息混入结果。
- `-show_format`：显示 container 层信息，例如 format name、duration、size 和 overall bitrate。
- `-show_streams`：显示每条 stream 的详细信息，包括 video、audio、subtitle 等。
- `-select_streams v:0`：只选择第一个 video stream。`v` 表示 video，`:0` 表示同类型 stream 中的第一条。
- `-select_streams a:0`：只选择第一个 audio stream。`a` 表示 audio，其余部分含义同上。
- `-show_entries stream=...`：只输出指定 stream 字段，适合把结果作为 AI 或脚本的输入。
- `-of default=nw=1`：使用 default 输出格式，关闭 section wrapper，让结果成为 `key=value` 形式。
- `-of default=nw=1:nk=1`：进一步隐藏 key，只输出 value，适合读取单个字段。

### 结果验收

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

### 常见错误

- `ffprobe: command not found`：FFmpeg 未安装，或 `ffprobe` 不在 `PATH` 中。
- `No such file or directory`：输入路径错误，或测试素材尚未生成。先运行 `./scripts/generate-samples.sh`。
- 输出为空：可能选择了不存在的 stream，例如对无音频文件使用 `-select_streams a:0`。
- 字段值是分数，例如 `r_frame_rate=30/1`：这是正常输出。脚本需要按 rational number 处理，不要直接当作整数。
- `bit_rate=N/A` 或字段缺失：部分容器或 stream 不保存准确 bitrate，需要结合 file size、duration 或重新编码参数判断。
- Windows 路径解析失败：建议在 WSL 或 Git Bash 中使用相对路径，或给包含空格的路径加引号。

### AI 提示词模板

请先根据以下 ffprobe 信息生成 FFmpeg 命令，不要猜测输入参数。你需要关注这些字段：输入文件路径、container format、duration、video codec、width、height、frame rate、pixel format、audio codec、sample rate、channels、audio bitrate，以及目标输出格式、是否允许覆盖输出文件、质量优先还是速度优先。

```text
任务：<描述你要做的视频处理，例如压缩、转码、裁剪、提取音频>
输入文件：examples/input/sample-720p.mp4
ffprobe 输出：
<粘贴 ffprobe -v error -show_format -show_streams 的结果，或分别粘贴 video/audio stream 字段>
输出要求：<目标路径、格式、清晰度、体积、兼容性要求>
限制：不要覆盖源文件；命令需要说明参数含义；给出验收命令。
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备测试素材。
2. 运行 `ffprobe -hide_banner examples/input/sample-720p.mp4`，观察完整概览适合人工阅读。
3. 运行 `ffprobe -v error -show_format -show_streams examples/input/sample-720p.mp4`，保存结构化字段。
4. 分别运行 video stream 和 audio stream 探测命令，确认可以独立获取视频和音频参数。
5. 把探测结果交给 AI，要求它生成后续 FFmpeg 命令，检查它是否使用了真实字段而不是猜测。

## 转封装

### 任务目标

使用 FFmpeg 将媒体文件从一种容器（container）转换为另一种容器，同时保持已有的编码流不变。本章示例把 MP4 转成 MKV，只改变封装格式，不重新编码视频流和音频流。

### 适用场景

- 输入文件的视频编码格式和音频编码格式已经符合目标要求，只需要换容器。
- 需要在 MP4、MOV、MKV 等格式之间快速转换，避免重新编码带来的画质损失。
- 需要修复或重建容器元数据（metadata），但不想改变原始音视频编码。
- 需要让 AI 明确区分 remux 和 transcode，避免不必要的重编码。

### 前置探测

先确认输入文件存在，并查看原始容器和流信息：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
```

如果使用自己的文件，先确认目标容器是否支持输入文件中的编码格式。例如 MKV 通常支持范围较广，MP4 对字幕流（subtitle）、部分音频编码和元数据的限制更明显。

### 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c copy examples/output/sample-remux.mkv
ffprobe -hide_banner examples/output/sample-remux.mkv
```

`-c copy` 表示 stream copy：直接复制输入中的编码流，不经过解码器和编码器。输出文件的容器从 MP4 变为 MKV，但视频编码格式、音频编码格式、分辨率、帧率、采样率等流参数保持不变。

### 常用变体

把 MKV 转为 MP4，前提是其中的编码格式被 MP4 容器支持：

```bash
ffmpeg -y -i input.mkv -c copy output.mp4
```

只保留视频流和音频流，丢弃字幕流或数据流：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -map 0:v:0 -map 0:a:0 -c copy examples/output/sample-remux-video-audio.mkv
```

复制所有流：

```bash
ffmpeg -y -i input.mkv -map 0 -c copy output.mkv
```

### 参数解释

- `-y`：允许覆盖已有输出文件，适合可重复运行的实验。处理真实文件时可以去掉，避免误覆盖。
- `-i examples/input/sample-720p.mp4`：指定输入文件。
- `-c copy`：对所有选中的流使用 copy 模式，不重新编码。也可以写成 `-c:v copy -c:a copy`，分别指定视频和音频。
- `examples/output/sample-remux.mkv`：输出路径。FFmpeg 通常根据扩展名选择 muxer，这里使用 Matroska/MKV 容器。
- `ffprobe -hide_banner`：查看输出文件的容器和流信息，用来确认容器已改变而编码流未改变。
- `-map`：手动选择要写入输出文件的流。默认映射通常会在每类流中选择一条，复杂文件建议显式指定。

### 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-remux.mkv
```

预期结果：

- 输出文件的容器格式（format）显示为 Matroska 或 WebM 相关格式。
- 视频流仍为原输入的 H.264，不应出现重新编码后的编码格式变化。
- 音频流仍为原输入的 AAC，不应出现重新编码后的编码格式变化。
- 命令执行速度通常明显快于转码（transcode），因为没有重新编码。

### 常见错误

- `Could not write header`：目标容器不支持某个输入流，需要换容器、去掉该流，或改用转码。
- 输出文件无法播放：可能是播放器对容器或编码格式组合支持不足。先用 `ffprobe` 确认流信息，再换播放器或改用兼容性更好的容器。
- 文件大小变化但画质没变：容器元数据、索引和封装开销不同会导致大小变化，这不代表发生了重新编码。
- 以为转封装能压缩文件：remux 不改变编码数据，通常不能显著降低体积。需要压缩时使用 CRF、码率（bitrate）或缩放（scale）等重新编码参数。
- 没有音频或字幕：默认的流选择可能未包含所有流，复杂文件请使用 `-map 0` 或明确的 `-map` 参数。

### AI 提示词模板

```text
请为以下输入文件生成 FFmpeg 转封装命令。目标是只改变 container，不重新编码 video/audio stream。请使用 stream copy，并给出 ffprobe 验收命令。

输入文件：<input>
目标 container：<例如 mkv 或 mp4>
ffprobe 信息：
<粘贴 ffprobe -hide_banner 或 -show_streams 输出>
限制：不要改变 codec、分辨率、帧率、采样率；如果目标 container 不支持某个 stream，请说明风险和替代方案。
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备测试素材。
2. 运行 `ffprobe -hide_banner examples/input/sample-720p.mp4` 记录输入容器和编码格式。
3. 运行 `ffmpeg -y -i examples/input/sample-720p.mp4 -c copy examples/output/sample-remux.mkv` 生成 MKV。
4. 运行 `ffprobe -hide_banner examples/output/sample-remux.mkv` 检查输出容器已改变。
5. 对比输入和输出的视频/音频编码格式，确认流仍为 H.264/AAC。
