+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：生态工具概览'
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
