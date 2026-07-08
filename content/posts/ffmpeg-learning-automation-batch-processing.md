+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：批处理工作流'
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

批处理不是把同一条 FFmpeg 命令复制很多遍，而是把“输入发现、前置探测、统一处理、输出命名、结果验收、失败处理”固定成流程。这样人和 AI 都能稳定复用。

## 推荐流程

1. 明确输入目录和输出目录，二者不要相同。
2. 用 `find` 列出将要处理的文件。
3. 用 `ffprobe` 抽查输入 codec、resolution、duration 和 audio stream。
4. 先处理 1 到 2 个样例文件。
5. 验收输出 codec、pixel format、resolution、audio codec 和 duration。
6. 批量运行全部文件。
7. 保存命令、日志和失败文件列表。

## 命名规则

输出文件名应表达处理目标，例如：

```text
input.mp4 -> input-h264-720p.mp4
input.mp4 -> input-compressed-crf28.mp4
input.mp4 -> input-watermarked.mp4
```

不要覆盖源文件。输出目录建议固定为 `examples/output/` 或业务项目中的专门产物目录。

## 日志和失败处理

简单脚本可以让失败直接停止，避免静默跳过问题。真实批处理可以为每个文件记录日志：

```bash
ffmpeg ... > "logs/${base}.log" 2>&1
```

如果需要继续处理其他文件，应记录失败文件列表，最后统一复查。

## AI 使用建议

让 AI 写批处理脚本时，不要只给“把目录里视频都压缩一下”。至少提供：

- 输入目录和输出目录。
- 是否允许覆盖输出。
- 目标 codec、resolution、CRF、audio bitrate。
- 一份样例文件的 `ffprobe` 输出。
- 失败时停止还是继续。
- 输出验收命令。
