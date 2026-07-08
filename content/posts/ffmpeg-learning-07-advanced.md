+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：API、生态和流媒体'
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

## FFmpeg API 概览


## CLI 适合什么

FFmpeg CLI 适合绝大多数文件处理任务，例如转码、裁剪、拼接、压缩、加字幕、水印、切片和批处理。它的优势是成熟、可组合、容易被 shell、脚本和 AI agent 调用。

第一版教程以 CLI 为主，因为它最贴近日常视频处理和 AI 辅助操作。

## 脚本封装适合什么

当任务需要重复执行、批量处理或统一参数时，可以用 sh/bash 封装 FFmpeg 命令。例如：

- 批量转码目录。
- 对每个输出运行 ffprobe 验收。
- 固定输入输出命名规则。
- 记录日志和失败文件。

脚本封装仍然调用 FFmpeg CLI，但能把操作流程固定下来，减少人工输入错误。

## 什么时候考虑 API

以下场景可能需要考虑 FFmpeg API：

- 需要把解码、滤镜、编码深度嵌入应用程序。
- 需要逐帧处理视频数据。
- 需要自定义 demux、mux、filter 或复杂实时流程。
- CLI 启动进程的开销、日志解析和错误处理已经成为瓶颈。
- 需要开发播放器、转码服务核心模块或自定义媒体处理引擎。

如果只是常见文件处理、批处理或服务端转码，CLI 加任务队列通常已经足够。

## libavformat

`libavformat` 负责 container 层处理，例如打开输入、读取 stream 信息、demux packet、写输出 container。它对应 CLI 中很多输入输出和封装相关能力。

常见概念：

- `AVFormatContext`
- input format / output format
- stream
- packet
- mux / demux

## libavcodec

`libavcodec` 负责 codec 层处理，例如 H.264、AAC、MP3 的解码和编码。

常见概念：

- `AVCodecContext`
- decoder / encoder
- packet
- frame
- pixel format
- sample format

## libavfilter

`libavfilter` 负责 filter graph，例如 scale、drawtext、subtitles、setpts、atempo、overlay 等。

CLI 中的 `-vf`、`-af`、`-filter_complex` 最终都对应 filter graph 的思想。

## 第一版不展开 C API 的原因

本教程第一版目标是建立视频处理任务能力，并让人和 AI 能稳定使用 FFmpeg。C API 学习成本高、代码量大、验证方式也不同。如果过早展开，会稀释核心目标。

后续可以把 FFmpeg API 单独做成进阶专题：

- 最小 demux 示例。
- 解码 frame。
- 编码输出。
- filter graph。
- 错误处理和资源释放。
- 与 CLI 工作流的取舍。

## 生态工具概览


## 总体思路

FFmpeg 是底层能力很强的音视频工具，但不是所有任务都应该只用 FFmpeg。实际工作中，经常会把 FFmpeg 和 OBS、HandBrake、Shaka Packager、GPAC、MediaInfo、Whisper、OCR 或视频理解模型组合使用。

## OBS

OBS 适合录屏、直播推流、场景合成和实时采集。需要做直播、录制桌面、摄像头混流时，OBS 通常比直接写 FFmpeg 命令更直观。

FFmpeg 更适合后处理，例如转码、裁剪、压缩、切片和批量处理。

## HandBrake

HandBrake 是图形化转码工具，适合人工本地转码、压缩和格式转换。它的 preset 对入门用户更友好。

如果任务需要脚本化、批处理、AI 调用或服务端运行，FFmpeg 更合适。

## Shaka Packager

Shaka Packager 适合 HLS/DASH 打包、多码率输出、DRM 和更专业的流媒体发布流程。FFmpeg 可以生成基础 HLS/DASH，但复杂发布工作流通常会交给专门 packager。

## GPAC

GPAC 提供 `MP4Box` 等工具，适合 MP4 box 操作、DASH 打包和某些封装层检查。需要深入检查 MP4 结构时，它可以补足 FFmpeg/ffprobe 的视角。

## MediaInfo

MediaInfo 适合快速查看媒体文件参数，输出对人更友好。`ffprobe` 更适合脚本和结构化字段检查。

常见组合是：人工排查时用 MediaInfo，自动化验收时用 ffprobe。

## Whisper 字幕生成

Whisper 适合从语音生成字幕文本。典型流程是：

1. 用 FFmpeg 提取音频。
2. 用 Whisper 生成 SRT/VTT。
3. 用 FFmpeg 添加 soft subtitles 或 burned subtitles。

## OCR 画面识别

OCR 可用于识别视频画面中的文字，例如课件、字幕、屏幕录制内容。FFmpeg 可以先截帧或按间隔抽帧，再交给 OCR 模型处理。

## 视频理解模型

视频理解模型适合做内容分析、镜头判断、场景摘要、违规检测或自动剪辑建议。FFmpeg 负责抽帧、裁剪、转码和生成最终视频；模型负责理解内容。

## 选择建议

- 实时采集和直播：优先 OBS。
- 人工本地转码：可以用 HandBrake。
- 自动化处理和批量任务：优先 FFmpeg/ffprobe。
- 专业 HLS/DASH/DRM：考虑 Shaka Packager 或 GPAC。
- 媒体信息人工查看：MediaInfo。
- 字幕生成：Whisper + FFmpeg。
- 内容理解：抽帧/OCR/视频模型 + FFmpeg。

## 流媒体和发布概览


## Web 兼容 MP4

面向网页、移动端和常见播放器发布时，最稳妥的基础格式通常是 MP4 container、H.264 video、AAC audio、`yuv420p` pixel format，并加上 `-movflags +faststart`。

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -crf 23 -preset medium -pix_fmt yuv420p -c:a aac -b:a 128k -movflags +faststart examples/output/web-compatible.mp4
```

这类输出适合直接上传、网页预览和普通文件分发。它不是自适应码率流媒体，但足够覆盖很多日常发布场景。

## HLS 基础

HLS 会把视频切成多个 `.ts` 或 `.m4s` segment，并生成一个 `.m3u8` playlist。播放器先读取 playlist，再按需拉取 segment。

```bash
mkdir -p examples/output/hls
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -c:a aac -hls_time 4 -hls_playlist_type vod examples/output/hls/index.m3u8
```

验收：

```bash
test -f examples/output/hls/index.m3u8
find examples/output/hls -maxdepth 1 -type f
```

`-hls_time 4` 表示目标 segment 时长约 4 秒。实际 segment 边界通常受 keyframe 影响。

## DASH 基础

DASH 和 HLS 类似，也是 manifest 加 segment 的结构。DASH 常见 manifest 是 `.mpd`。FFmpeg 能生成基础 DASH 输出，但更复杂的多码率、自适应和加密场景通常会使用 Shaka Packager 或 GPAC。

```bash
mkdir -p examples/output/dash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -c:a aac -f dash examples/output/dash/manifest.mpd
```

## 关键帧和切片

切片边界通常应落在 keyframe 上。若 keyframe 间隔太长，HLS/DASH segment 可能不稳定或不接近目标时长。常见做法是明确 GOP：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -g 60 -keyint_min 60 -sc_threshold 0 -c:a aac -hls_time 2 -hls_playlist_type vod examples/output/hls/index.m3u8
```

30 FPS 视频中 `-g 60` 大约是 2 秒一个 keyframe。真实项目还要结合平台规格、编码 ladder 和播放器行为测试。

## 平台上传规格

不同平台对 codec、resolution、bitrate、frame rate、audio codec 和 loudness 可能有不同要求。不要只让 AI 猜参数；应先收集目标平台规格，再生成命令。

通用检查项：

- container：MP4、HLS、DASH 或平台指定格式。
- video codec：H.264、H.265、AV1。
- audio codec：AAC 通常兼容性较好。
- resolution 和 frame rate：是否符合平台推荐。
- bitrate 或 CRF：是否在质量和体积之间平衡。
- subtitle：是否需要 soft subtitles 或 burned subtitles。
