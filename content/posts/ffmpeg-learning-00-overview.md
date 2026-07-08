+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：教程总览'
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
