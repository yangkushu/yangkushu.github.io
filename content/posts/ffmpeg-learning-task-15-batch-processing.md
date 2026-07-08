+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：批量处理目录'
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

使用 sh 脚本批量处理目录中的 MP4 文件，把它们统一转为 H.264/AAC、最大宽度 1280、兼容性较好的 MP4。

## 适用场景

- 需要批量转码一个目录中的视频。
- 需要统一输出参数，便于后续上传、预览或分发。
- 需要让 AI 生成可重复运行的批处理流程，而不是一次性手写多条命令。
- 需要用 `ffprobe` 在处理前确认输入文件至少包含 video stream。

## 前置探测

先准备测试素材，并查看目录中的 MP4：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
find examples/input -maxdepth 1 -type f -name "*.mp4"
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/input/sample-720p.mp4
```

批处理前要先确认输入目录是否正确，避免把输出目录再次当作输入目录递归处理。

## 基础命令

```bash
./scripts/batch-transcode.sh examples/input examples/output/batch
find examples/output/batch -maxdepth 1 -type f -name "*.mp4"
```

脚本会遍历输入目录中的 `*.mp4`，输出到 `examples/output/batch/`，并为文件名添加 `-h264-720p` 后缀。

## 常用变体

使用默认输入输出目录：

```bash
./scripts/batch-transcode.sh
```

指定自定义目录：

```bash
./scripts/batch-transcode.sh /path/to/input /path/to/output
```

只先列出将要处理的文件：

```bash
find examples/input -maxdepth 1 -type f -name "*.mp4"
```

真实批处理前建议先对少量文件试跑，确认输出画质、体积和兼容性符合预期。

## 参数解释

- `INPUT_DIR="${1:-...}"`：第一个参数是输入目录；未传入时使用默认样例目录。
- `OUTPUT_DIR="${2:-...}"`：第二个参数是输出目录；未传入时使用默认批处理输出目录。
- `for input in "$INPUT_DIR"/*.mp4`：遍历输入目录中的 MP4 文件。
- `ffprobe ... "$input" >/dev/null`：处理前确认输入至少能探测到 video stream。
- `scale='min(1280,iw)':-2`：宽度最大 1280；如果原视频小于 1280，则保留原宽度；高度按比例计算并保持偶数。
- `-c:v libx264 -crf 23 -pix_fmt yuv420p`：输出兼容性较好的 H.264/yuv420p。
- `-c:a aac -b:a 128k`：统一音频为 AAC。
- `-movflags +faststart`：优化 MP4 网络播放启动。

## 结果验收

检查输出文件列表：

```bash
find examples/output/batch -maxdepth 1 -type f -name "*.mp4"
```

抽查一个输出文件：

```bash
ffprobe -hide_banner examples/output/batch/sample-720p-h264-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,pix_fmt -of default=nw=1 examples/output/batch/sample-720p-h264-720p.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 examples/output/batch/sample-720p-h264-720p.mp4
```

预期 video codec 为 `h264`，`pix_fmt` 为 `yuv420p`，宽度不超过 1280；audio codec 为 `aac`。

## 常见错误

- 输入目录没有 MP4：脚本不会生成输出。先用 `find` 确认输入文件。
- 输出覆盖旧文件：脚本使用 `-y`，同名输出会被覆盖。重要产物应输出到新目录。
- 某个文件失败后脚本停止：脚本使用 `set -eu`，遇到错误会停止，便于暴露问题。
- 文件名包含换行：简单 shell 循环不适合处理带换行的路径。真实生产流程应限制文件命名或使用更稳健的任务队列。
- 输出画质不符合预期：调整 `-crf`、`-preset`、分辨率和 audio bitrate。

## AI 提示词模板

```text
请基于以下输入目录信息生成 FFmpeg 批处理方案。必须先列出将处理的文件，并用 ffprobe 抽查输入的 video/audio stream。请生成 sh/bash 脚本，统一输出 H.264/AAC、yuv420p、最大宽度 1280、MP4 faststart。

硬性要求：输出必须写入 examples/output/ 或我指定的输出目录；不要覆盖源文件；如果输入目录和输出目录相同，必须拒绝并说明原因；必须给出 find 输出检查和 ffprobe 字段级验收命令。

输入目录：<input dir>
输出目录：<output dir>
目标参数：<codec/resolution/crf/audio bitrate>
ffprobe 抽查信息：
<粘贴样例文件探测结果>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行 `./scripts/batch-transcode.sh examples/input examples/output/batch`。
3. 用 `find` 查看 `examples/output/batch/` 中的输出文件。
4. 用 `ffprobe` 抽查 `sample-720p-h264-720p.mp4` 的 video/audio stream 字段。
5. 修改 CRF 或输出目录，重新运行并比较体积和画质。
