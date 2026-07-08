+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：必要音视频概念'
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


理解下面这些概念，至少能看懂大多数 FFmpeg 命令是在操作哪些维度。

## 核心概念

**Container（封装格式）**

用于承载多个流和元数据，例如视频、音频、字幕以及章节信息。常见封装格式包括 MP4、MOV、MKV。

常见相关操作：把 MKV 转成 MP4 时，如果只改封装，通常会配合 `-c copy`。

**Codec（编码格式）**

定义音视频数据如何被压缩和还原，例如 H.264、H.265、AAC、Opus。

转码会改变 codec；转封装在使用 stream copy 时通常不改变 codec。

常见相关参数：`-c:v libx264` 指定视频编码器，`-c:a aac` 指定音频编码器。

**Stream（媒体流）**

一个媒体文件可以包含多个 stream，例如一个视频流、多个音频流、多个字幕流。

可以把三者理解为：容器承载多个 stream，每个 stream 的内容由对应 codec 编码。

常见相关参数：`-map 0:v:0`、`-map 0:a:1` 用来选择具体的视频流或音频流。

**Bitrate（码率）**

通常与体积、画质和编码耗时相关，但三者的关系不总是线性的。视频压缩经常需要在码率、质量和编码速度之间取舍。

常见相关参数：`-b:v 2M` 设置视频目标码率，`-b:a 128k` 设置音频目标码率。

**Frame Rate（帧率）**

例如 24、25、30、60 FPS。修改或重采样帧率可能影响流畅度、体积和音画同步。

常见相关参数：`-r 30` 设置输出帧率。

**Resolution（分辨率）**

例如 1920x1080。调整分辨率通常使用 `scale` 滤镜。

常见相关参数：`-vf scale=1280:-2` 把宽度缩放到 1280，高度按比例自动计算并保持偶数。

**Pixel Format（像素格式）**

描述像素存储方式。为了兼容性，H.264 MP4 常见使用 `yuv420p`，但它不是唯一可用格式。

常见相关参数：`-pix_fmt yuv420p` 指定输出像素格式。

**Keyframe 和 GOP（关键帧和画面组）**

关键帧是可以独立解码的帧，GOP 是两个关键帧之间的一组画面。关键帧间隔会影响 seek、切片和流媒体播放；HLS 和 DASH 场景会更关注关键帧是否稳定对齐。

常见相关参数：`-g 48` 设置 GOP 大小，表示大约每 48 帧放一个关键帧。
