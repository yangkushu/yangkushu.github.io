+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：添加字幕'
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

使用 FFmpeg 为视频添加字幕，理解 soft subtitles 和 burned subtitles 的区别。

## 适用场景

- 需要给 MP4 添加可开关的外挂式字幕轨。
- 需要把字幕烧录进画面，保证所有播放器都能看到字幕。
- 需要让 AI 判断字幕文件格式、目标平台和播放器兼容性。
- 需要基于可复现的 SRT 文件演示字幕处理流程。

## 前置探测

先准备素材，并创建一个简单的 SRT 字幕文件：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
printf "1\n00:00:00,000 --> 00:00:03,000\nHello FFmpeg\n\n2\n00:00:03,000 --> 00:00:06,000\nSubtitle demo\n" > examples/input/demo.srt
ffprobe -hide_banner examples/input/sample-720p.mp4
```

SRT 时间格式是 `时:分:秒,毫秒 --> 时:分:秒,毫秒`。字幕时间需要落在视频时长范围内。

## 基础命令

添加 soft subtitles，可在播放器中开关：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/demo.srt -c:v copy -c:a copy -c:s mov_text examples/output/sample-soft-subtitle.mp4
```

烧录字幕到画面：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:a copy examples/output/sample-burned-subtitle.mp4
```

soft subtitles 会增加 subtitle stream，画面本身不变；burned subtitles 会修改 video frame，因此必须重新编码视频，字幕不能关闭。

## 常用变体

指定字幕语言元数据：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/demo.srt -c:v copy -c:a copy -c:s mov_text -metadata:s:s:0 language=eng examples/output/sample-soft-subtitle-eng.mp4
```

烧录字幕并指定兼容 pixel format：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:v libx264 -pix_fmt yuv420p -c:a copy examples/output/sample-burned-subtitle-compatible.mp4
```

如果输出 MP4 的音频兼容性不确定，把音频转 AAC：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k examples/output/sample-burned-subtitle-aac.mp4
```

## 参数解释

- `-i examples/input/demo.srt`：把 SRT 字幕作为第二个输入。
- `-c:v copy -c:a copy -c:s mov_text`：复制已有 video/audio stream，并把字幕转为 MP4 支持的 `mov_text` subtitle codec。
- `-metadata:s:s:0 language=eng`：给第一条 subtitle stream 设置语言元数据。
- `-vf "subtitles=..."`：使用 subtitles filter，把字幕渲染到 video frame 上。
- `-c:v libx264 -pix_fmt yuv420p`：烧录字幕时重新编码视频，并输出兼容性较好的 H.264/yuv420p。
- `-c:a copy`：复制音频。若目标 container 或播放器不支持输入音频 codec，改用 `-c:a aac -b:a 128k`。

## 结果验收

检查 soft subtitle 输出：

```bash
ffprobe -hide_banner examples/output/sample-soft-subtitle.mp4
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name,codec_type -of default=nw=1 examples/output/sample-soft-subtitle.mp4
ffprobe -v error -show_streams examples/output/sample-soft-subtitle.mp4
```

预期包含 subtitle stream，`codec_name` 通常为 `mov_text`。

检查 burned subtitle 输出：

```bash
ffprobe -hide_banner examples/output/sample-burned-subtitle.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/sample-burned-subtitle.mp4
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-burned-subtitle.mp4
```

burned subtitle 输出通常不再包含 subtitle stream，因为字幕已经变成画面的一部分。需要播放视频或截帧检查字幕是否出现在画面上：

```bash
ffmpeg -y -ss 00:00:01 -i examples/output/sample-burned-subtitle.mp4 -frames:v 1 examples/output/sample-burned-subtitle-check.jpg
```

## 常见错误

- MP4 中看不到 soft subtitles：播放器不显示或未开启字幕轨。换播放器验证，或使用 burned subtitles。
- `Subtitle encoding currently only possible from text to text or bitmap to bitmap`：字幕 codec/container 不匹配。MP4 中常用 `-c:s mov_text`。
- `subtitles` filter 不可用：当前 FFmpeg 构建缺少 libass。安装完整 FFmpeg，或只使用 soft subtitles。
- 字幕乱码：SRT 编码不是 UTF-8。先转换字幕文件编码。
- 字幕时间不对：SRT 时间轴和视频时间轴不匹配。检查字幕起止时间和视频 duration。
- 烧录字幕后无法关闭字幕：这是 burned subtitles 的特性。如果需要可开关字幕，使用 soft subtitles。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 添加字幕命令。必须先根据 ffprobe 信息分析视频输入的 container、video/audio codec、duration，以及字幕文件格式和时间范围。请分别说明 soft subtitles 和 burned subtitles 的方案、兼容性和取舍。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；soft subtitles 必须给出 subtitle stream 验收命令；burned subtitles 必须给出 video stream 验收命令和截图检查命令。

视频输入：<video input>
字幕文件：<subtitle input>
输出方式：<soft/burned>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴视频 stream 信息和字幕文件摘要>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备视频。
2. 运行 `printf` 命令创建 `examples/input/demo.srt`。
3. 运行 soft subtitle 命令生成 `examples/output/sample-soft-subtitle.mp4`。
4. 运行 burned subtitle 命令生成 `examples/output/sample-burned-subtitle.mp4`。
5. 用 `ffprobe` 检查 soft subtitle 的 subtitle stream，并截帧检查 burned subtitle 是否显示在画面中。
