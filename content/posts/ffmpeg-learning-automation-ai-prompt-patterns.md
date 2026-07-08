+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：AI 提示词模式'
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


## 总体原则

AI 可以辅助生成 FFmpeg 命令，但不能跳过输入探测和输出验收。一个稳定的提示词应包含任务目标、输入信息、输出位置、安全约束、命令要求和验收方式。

## 通用命令生成模板

```text
请先根据以下 ffprobe 信息判断输入文件的 container、video stream、audio stream、duration、resolution 和 frame rate。然后生成一个 FFmpeg 命令。要求：

- 不覆盖源文件。
- 输出到 examples/output/。
- 如果输入路径和输出路径相同，必须拒绝。
- 说明每个关键参数。
- 给出执行后的 ffprobe 验收命令。
- 如果命令失败，根据错误日志重新分析。

任务目标：<目标>
输入文件：<input>
输出文件：<examples/output/output.ext>
ffprobe 信息：
<粘贴 ffprobe 输出>
```

## 错误修复模板

```text
下面是 FFmpeg 命令和错误日志。请先解释错误原因，再给出修正后的命令。不要改变任务目标，不要覆盖源文件，输出仍然写入 examples/output/。如果需要更多输入信息，请明确列出需要我运行的 ffprobe 命令。

原命令：
<command>

错误日志：
<stderr>
```

## 批处理模板

```text
请为以下目录生成 sh/bash 批处理脚本。脚本需要先检查输入目录，遍历 MP4 文件，输出到独立输出目录，不覆盖源文件。每个输出统一为 H.264/AAC、yuv420p、最大宽度 1280，并在脚本后给出 ffprobe 抽查命令。

输入目录：<input dir>
输出目录：<output dir>
是否遇错停止：<是/否>
样例 ffprobe 信息：
<probe>
```

## 审核 AI 输出

执行 AI 给出的命令前，至少检查：

- 输入路径是否存在。
- 输出路径是否在 `examples/output/` 或指定输出目录。
- 输入路径和输出路径是否相同。
- 是否有 `-y` 覆盖已有重要文件的风险。
- 是否解释了关键参数。
- 是否提供了 `ffprobe` 验收命令。
