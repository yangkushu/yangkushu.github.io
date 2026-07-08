+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：提取音频'
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
