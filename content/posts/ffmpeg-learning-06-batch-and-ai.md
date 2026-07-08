+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：批处理、验收和 AI 提示词'
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

## 批量处理目录


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

## 批处理工作流


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

## ffprobe 验收方法


## 总体思路

FFmpeg 命令执行成功不等于输出正确。每个任务都应使用 `ffprobe` 检查输出文件的关键字段，再结合必要的人工播放或截图检查。

## 前置检查

处理前先查看输入：

```bash
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_format -show_streams examples/input/sample-720p.mp4
```

脚本中建议使用 `-v error`，减少 banner、warning 和诊断日志噪声，方便解析字段。

## 常用字段

检查视频：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt,duration -of default=nw=1 output.mp4
```

检查音频：

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,bit_rate,duration -of default=nw=1 output.mp4
```

检查字幕：

```bash
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name,codec_type,duration -of default=nw=1 output.mp4
```

检查整体时长：

```bash
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 output.mp4
```

## 不同任务的验收重点

- 转封装：codec 是否保持不变，container 是否改变。
- 转码：video/audio codec、pixel format、bitrate 是否符合目标。
- 压缩：文件体积、CRF 输出质量、codec、pix_fmt、audio bitrate。
- 缩放：width、height、DAR/SAR。
- 裁剪：duration、stream 是否存在、边界是否符合预期。
- 拼接：输出字段是否统一，duration 是否接近片段之和，播放切换点。
- 字幕：soft subtitle 检查 subtitle stream；burned subtitle 需要截图或播放检查。

## AI 使用建议

让 AI 生成 FFmpeg 命令时，要求它同时生成验收命令。验收命令至少应覆盖：

- 输出文件是否存在。
- video stream 是否存在及 codec/resolution/pix_fmt。
- audio stream 是否存在及 codec/sample_rate/channels。
- format duration 是否接近预期。
- 任务特有字段，例如 subtitle stream、thumbnail image、watermark screenshot。

## AI 提示词模式


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
