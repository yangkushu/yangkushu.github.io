+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：音频提取和替换'
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

## 提取音频


## 任务目标

从视频文件中提取 audio stream，生成原编码音频文件或重新编码后的常见音频格式。

## 适用场景

- 需要从视频中单独导出声音。
- 需要把视频里的 AAC 音频保存为 `.m4a`。
- 需要生成 MP3 供更广泛的播放器、剪辑软件或平台使用。
- 需要让 AI 判断应该 stream copy 还是重新编码。

## 前置探测

先准备素材并查看输入是否包含 audio stream：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/input/sample-720p.mp4
```

如果输入没有 audio stream，提取音频命令会失败。真实文件可能包含多个音轨，需要先确认要提取的是 `a:0`、`a:1` 还是其他音轨。

## 基础命令

不重新编码，直接复制原 audio stream：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vn -c:a copy examples/output/sample-audio.m4a
```

重新编码为 MP3：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vn -c:a libmp3lame -b:a 192k examples/output/sample-audio.mp3
```

第一条适合输入音频本身就是 AAC，并希望无损、快速导出到 M4A。第二条会重新编码，适合需要 MP3 兼容性的场景。

## 常用变体

提取并转为 WAV，适合后续语音识别、降噪或音频分析：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vn -c:a pcm_s16le -ar 16000 -ac 1 examples/output/sample-audio-16k-mono.wav
```

只提取第二条音轨：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -map 0:a:1 -vn -c:a copy examples/output/sample-audio-track2.m4a
```

如果第二条音轨不存在，这条命令会失败。先用 `ffprobe -show_streams` 或 `ffmpeg -i` 查看音轨列表。

## 参数解释

- `-vn`：不输出 video stream，只处理音频。
- `-c:a copy`：复制 audio stream，不重新编码，速度快且不损失质量，但输出容器必须支持该 codec。
- `.m4a`：常用于保存 AAC 音频的音频容器。
- `-c:a libmp3lame`：使用 MP3 encoder 重新编码音频。
- `-b:a 192k`：设置 audio bitrate。语音可以更低，音乐通常需要更高码率。
- `-ar 16000`：设置 audio sample rate 为 16 kHz。
- `-ac 1`：转为单声道 mono。
- `-map 0:a:1`：选择第一个输入中的第二条 audio stream。

## 结果验收

检查 M4A 输出：

```bash
ffprobe -hide_banner examples/output/sample-audio.m4a
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/output/sample-audio.m4a
```

检查 MP3 输出：

```bash
ffprobe -hide_banner examples/output/sample-audio.mp3
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/output/sample-audio.mp3
```

预期 M4A 输出包含 audio stream，示例素材中 `codec_name` 通常为 `aac`。MP3 输出的 `codec_name` 应为 `mp3`，并能看到 sample rate、channels 和 bitrate 信息。

## 常见错误

- `Output file #0 does not contain any stream`：输入没有音频，或使用了错误的 `-map`。
- `Could not write header`：输出容器不支持 copied audio codec。改用兼容容器，或重新编码为 AAC/MP3/WAV。
- 多音轨提取错了：没有先查看 stream index。先用 `ffprobe -show_streams` 确认音轨顺序和语言标签。
- MP3 encoder 不可用：当前 FFmpeg 构建缺少 `libmp3lame`。可以安装完整 FFmpeg，或改用 AAC/WAV。
- 音质变差：重新编码 bitrate 太低。提高 `-b:a`，或使用 `-c:a copy` 保留原始音频。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 提取音频命令。必须先根据 ffprobe 信息分析输入是否包含 audio stream、audio codec、sample_rate、channels、bit_rate，以及是否存在多个音轨。请分别判断是否适合 -c:a copy 导出 M4A，以及是否需要重新编码为 MP3 或 WAV。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 audio codec、sample_rate、channels、bit_rate 的验收命令。

输入文件：<input>
输出格式：<m4a/mp3/wav>
目标音轨：<例如 a:0>
ffprobe 信息：
<粘贴 audio stream 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行 M4A 提取命令生成 `examples/output/sample-audio.m4a`。
3. 运行 MP3 转码命令生成 `examples/output/sample-audio.mp3`。
4. 用 `ffprobe` 检查两个输出文件的 audio stream 字段。
5. 对比 `-c:a copy` 和重新编码的速度、体积和 codec 差异。

## 替换音频


## 任务目标

把视频中的原音频替换为新的 audio stream，并保持视频画面不重新编码。

## 适用场景

- 需要给视频替换背景音乐、旁白或测试音频。
- 需要保留原视频画质，只更换音频。
- 需要让 AI 明确选择 video stream 来自输入 0、audio stream 来自输入 1。
- 需要处理音频长度和视频长度不一致的问题。

## 前置探测

先准备视频素材，并确认视频和新音频的时长、codec：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
```

本章先用 FFmpeg 生成一段 220 Hz 的替换音频，避免依赖外部素材。

## 基础命令

生成替换音频：

```bash
ffmpeg -y -f lavfi -i sine=frequency=220:sample_rate=48000 -t 8 -c:a aac examples/input/replacement-audio.m4a
```

用新音频替换原视频音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -shortest examples/output/sample-replaced-audio.mp4
```

`-map 0:v:0` 表示使用第一个输入的视频流，`-map 1:a:0` 表示使用第二个输入的音频流。`-c:v copy` 不重新编码视频，`-c:a aac` 确保输出 MP4 的音频兼容性。

## 常用变体

保留视频并让音频循环到视频结束：

```bash
ffmpeg -y -stream_loop -1 -i examples/input/replacement-audio.m4a -i examples/input/sample-720p.mp4 -map 1:v:0 -map 0:a:0 -c:v copy -c:a aac -shortest examples/output/sample-replaced-audio-loop.mp4
```

替换音频并调整音量：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -af "volume=0.6" -c:a aac -shortest examples/output/sample-replaced-audio-volume.mp4
```

如果新音频已经是 MP4 兼容的 AAC，也可以 copy 音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/replacement-audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -c:a copy -shortest examples/output/sample-replaced-audio-copy.mp4
```

使用 `-c:a copy` 前必须先用 `ffprobe` 确认新音频 codec 被目标 container 和播放器支持；不确定时转 AAC 更稳。

## 参数解释

- `-f lavfi -i sine=...`：用 FFmpeg filter 生成测试音频。
- `-t 8`：生成 8 秒音频。
- `-map 0:v:0`：选择第一个输入的第一条 video stream。
- `-map 1:a:0`：选择第二个输入的第一条 audio stream。
- `-c:v copy`：复制视频流，不重新编码，速度快且不损失视频质量。
- `-c:a aac`：把新音频编码为 AAC，适合 MP4 输出。
- `-shortest`：当视频和音频时长不一致时，以较短的 stream 结束输出。
- `-stream_loop -1`：无限循环后面的输入文件，通常和 `-shortest` 配合使用。

## 结果验收

检查替换后的输出：

```bash
ffprobe -hide_banner examples/output/sample-replaced-audio.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/sample-replaced-audio.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/output/sample-replaced-audio.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-replaced-audio.mp4
```

预期输出包含 video stream 和 audio stream。video stream 应保持 H.264、1280x720；audio stream 应为 AAC。播放输出文件，确认听到 220 Hz 测试音，而不是原来的 1000 Hz 音频。

## 常见错误

- 替换后仍是原音频：`-map` 写错，或没有明确选择第二个输入的 audio stream。
- 输出没有音频：第二个输入没有 audio stream，或 `-map 1:a:0` 指向不存在的流。
- 输出时长不符合预期：视频和音频长度不同，且没有理解 `-shortest` 的行为。
- 视频被重新编码：没有使用 `-c:v copy`，或使用了 video filter。
- 音频 copy 后无法播放：目标 MP4 或播放器不支持该 audio codec。改用 `-c:a aac -b:a 128k`。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 替换音频命令。必须先根据 ffprobe 信息分析视频输入的 video/audio stream、新音频输入的 audio codec、sample_rate、channels 和两个输入的 duration。请明确使用 -map 0:v:0 和 -map 1:a:0，保留视频不重新编码，并判断音频应该转 AAC 还是可以 copy。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 video/audio stream、codec、duration 的验收命令；如果新音频比视频短或长，必须说明 -shortest、循环或裁剪策略。

视频输入：<video input>
新音频输入：<audio input>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴两个输入的 format 和 stream 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备视频。
2. 运行 sine 命令生成 `examples/input/replacement-audio.m4a`。
3. 运行替换音频命令生成 `examples/output/sample-replaced-audio.mp4`。
4. 用 `ffprobe` 检查输出包含 video stream 和 audio stream。
5. 播放输出文件，确认音频已经变为 220 Hz 测试音。
