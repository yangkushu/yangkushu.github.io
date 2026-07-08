+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：导读、环境和概念'
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

## 教程目录


这里是教程入口。建议按顺序先建立环境和概念，再进入任务章节；如果你已经熟悉 FFmpeg，也可以直接跳到需要的任务。

## 从这里开始

1. [教程总览](/posts/ffmpeg-learning-00-guide/)
2. [环境准备](/posts/ffmpeg-learning-00-guide/)
3. [必要音视频概念](/posts/ffmpeg-learning-00-guide/)
4. [查看媒体信息](/posts/ffmpeg-learning-01-media-basics/)
5. [生成测试素材](/posts/ffmpeg-learning-01-media-basics/)

完成前五步后，再根据目标选择后续任务。

## 任务目录

1. [查看媒体信息](/posts/ffmpeg-learning-01-media-basics/)
2. [生成测试素材](/posts/ffmpeg-learning-01-media-basics/)
3. [封装转换](/posts/ffmpeg-learning-02-transcode-compress/)
4. [视频转码](/posts/ffmpeg-learning-02-transcode-compress/)
5. [视频压缩](/posts/ffmpeg-learning-02-transcode-compress/)
6. [调整分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
7. [裁剪片段](/posts/ffmpeg-learning-03-editing/)
8. [拼接视频](/posts/ffmpeg-learning-03-editing/)
9. [提取音频](/posts/ffmpeg-learning-04-audio/)
10. [替换音频](/posts/ffmpeg-learning-04-audio/)
11. [截图和封面](/posts/ffmpeg-learning-05-visual-elements/)
12. [加字幕](/posts/ffmpeg-learning-05-visual-elements/)
13. [加水印](/posts/ffmpeg-learning-05-visual-elements/)
14. [变速](/posts/ffmpeg-learning-03-editing/)
15. [批量处理目录](/posts/ffmpeg-learning-06-batch-and-ai/)

## 自动化和 AI 辅助

1. [批处理思路](/posts/ffmpeg-learning-06-batch-and-ai/)
2. [ffprobe 验收](/posts/ffmpeg-learning-06-batch-and-ai/)
3. [AI prompt 模板](/posts/ffmpeg-learning-06-batch-and-ai/)

## 扩展主题

1. [FFmpeg API 概览](/posts/ffmpeg-learning-07-advanced/)
2. [生态和工具链](/posts/ffmpeg-learning-07-advanced/)
3. [流媒体基础](/posts/ffmpeg-learning-07-advanced/)

## 教程总览


## 定位

本教程关注视频处理实践，而不是 FFmpeg 参数百科。计划每个任务章节都从一个真实问题出发，给出可执行命令、参数解释、输出验收和 AI 辅助模板。

## 为什么先探测再处理

FFmpeg 命令依赖输入文件的容器、编码、流、时长、分辨率、帧率和音频参数。直接猜参数容易导致音画不同步、编码失败、输出不可播放或体积异常。

## 标准处理流程

1. 确认输入文件路径。
2. 使用 `ffprobe` 获取媒体信息。
3. 选择最小可用命令。
4. 根据质量、速度、兼容性调整参数。
5. 输出到新文件。
6. 验收输出文件。

## AI 辅助原则

AI 生成命令前必须先知道输入信息。AI 输出命令时必须说明输入、输出、是否覆盖、关键参数含义和验收方式。

## 环境准备


## 安装 FFmpeg

### macOS

推荐使用 Homebrew：

```bash
brew install ffmpeg
```

### Windows

推荐两种方式：

- 使用 WSL，然后在 Linux 环境中安装 FFmpeg。
- 使用 Git Bash 执行本教程中的 sh/bash 脚本，并使用 Windows 版 FFmpeg。

### 检查安装

```bash
ffmpeg -version
ffprobe -version
ffplay -version
```

如果 `ffplay` 不可用，不影响大部分教程任务；它主要用于快速预览媒体文件。

## 路径和覆盖规则

- 输入文件放在 `examples/input/`。
- 输出文件放在 `examples/output/`。
- 不直接覆盖源文件。
- 需要覆盖输出文件时显式使用 `-y`。
- 需要避免覆盖时使用 `-n`。

## 必要音视频概念


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
