+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：调整播放速度'
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

使用 FFmpeg 同时调整视频和音频速度，理解 `setpts` 控制 video timestamp、`atempo` 控制 audio tempo 的基本方法与限制。

## 适用场景

- 需要把视频加速生成 2x 版本。
- 需要缩短教程、演示或素材预览时长。
- 需要同时处理音频，避免只有画面加速而声音不同步。
- 需要让 AI 明确视频变速和音频变速是两条不同处理链。

## 前置探测

先准备素材并确认输入包含 video stream 和 audio stream：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
```

如果输入没有音频，需要移除音频 filter 和 audio map，只处理视频。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v];[0:a]atempo=2.0[a]" -map "[v]" -map "[a]" examples/output/sample-2x.mp4
```

`setpts=0.5*PTS` 会把视频时间戳压缩为原来的一半，所以播放速度变为 2x。`atempo=2.0` 会把音频速度变为 2x，并尽量保持音调稳定。

## 常用变体

0.5x 慢放：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=2.0*PTS[v];[0:a]atempo=0.5[a]" -map "[v]" -map "[a]" examples/output/sample-0_5x.mp4
```

4x 加速时串联多个 `atempo`，避免超过单个 filter 的稳定范围：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.25*PTS[v];[0:a]atempo=2.0,atempo=2.0[a]" -map "[v]" -map "[a]" examples/output/sample-4x.mp4
```

无音频视频只调整画面速度：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v]" -map "[v]" -an examples/output/sample-2x-video-only.mp4
```

## 参数解释

- `-filter_complex`：同时定义视频和音频 filter graph，并给输出流命名。
- `[0:v]setpts=0.5*PTS[v]`：读取第一个输入的视频流，把 presentation timestamp 乘以 0.5，输出命名为 `[v]`。
- `[0:a]atempo=2.0[a]`：读取第一个输入的音频流，把音频 tempo 调整为 2.0 倍，输出命名为 `[a]`。
- `-map "[v]" -map "[a]"`：明确选择 filter 输出的视频和音频，避免 FFmpeg 自动选择错误 stream。
- `setpts`：只改变视频帧的时间戳，不会自动处理音频。
- `atempo`：调整音频 tempo。常用单个值范围是 `0.5` 到 `2.0`；更大或更小倍率建议串联多个 `atempo`。

## 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-2x.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,duration -of default=nw=1 examples/output/sample-2x.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,duration -of default=nw=1 examples/output/sample-2x.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-2x.mp4
```

预期输出包含 video stream 和 audio stream，format duration 接近输入的一半。示例输入约 8 秒，2x 输出应接近 4 秒。`stream=duration` 在某些 container 或 stream 上可能缺失，此时至少使用 format duration 并播放检查同步；如果 video/audio stream duration 差异明显，需要重新检查 `setpts`、`atempo` 和 `-map`。

## 常见错误

- 只用了 `setpts`：画面速度改变，但音频仍是原速度，导致音画不同步或输出时长不符合预期。
- `atempo` 倍率过大：单个 `atempo` 常用范围为 `0.5` 到 `2.0`，超过时应串联多个 filter，例如 `atempo=2.0,atempo=2.0`。
- 倍率方向写反：2x 加速视频用 `setpts=0.5*PTS`，0.5x 慢放用 `setpts=2.0*PTS`。
- 输入没有音频：`[0:a]` 会报错。移除音频 filter 和 `-map "[a]"`，并添加 `-an`。
- 未显式 `-map`：复杂 filter 输出可能没有被正确写入，建议始终 map 命名输出。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 变速命令。必须先根据 ffprobe 信息分析输入 duration、video/audio stream 是否存在，以及是否需要同时处理音频。目标是生成 2x 加速 MP4，视频使用 setpts 调整 timestamp，音频使用 atempo 调整 tempo，并显式 map filter 输出。请解释 setpts 与 atempo 的倍率关系，以及 atempo 超过 0.5-2.0 常用范围时如何串联。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 video/audio stream 存在性、format duration、video/audio duration 或同步风险说明的验收命令。

输入文件：<input>
输出文件：<examples/output/output.mp4>
目标速度：<例如 2x>
是否包含音频：<是/否>
ffprobe 信息：
<粘贴 stream 和 duration 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础命令生成 `examples/output/sample-2x.mp4`。
3. 用 `ffprobe` 分别读取输入和输出 duration。
4. 确认输出时长接近输入的一半。
5. 把命令改为 `setpts=2.0*PTS` 和 `atempo=0.5`，生成慢放版本并比较时长。
