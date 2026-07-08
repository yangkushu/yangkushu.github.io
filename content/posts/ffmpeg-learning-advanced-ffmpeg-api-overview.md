+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：FFmpeg API 概览'
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
