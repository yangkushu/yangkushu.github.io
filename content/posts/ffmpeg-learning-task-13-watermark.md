+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：添加水印'
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

使用 FFmpeg 的 drawtext filter 添加文字水印，并理解文字渲染、字体可用性和重新编码的影响。

## 适用场景

- 需要给演示视频、预览文件或内部素材添加文字标识。
- 不想依赖外部图片水印文件，优先使用 text watermark。
- 需要在视频右下角添加固定文字。
- 需要让 AI 生成可复现的水印命令，并提醒 drawtext/font 风险。

## 前置探测

先准备素材并确认 FFmpeg 支持 drawtext：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffmpeg -filters | grep drawtext
ffprobe -hide_banner examples/input/sample-720p.mp4
```

`drawtext` 依赖 FFmpeg 构建时启用相关 filter，运行环境还需要能通过 fontconfig 或默认字体配置找到可用字体。某些精简构建可能没有 drawtext，某些容器环境可能缺少字体。

## 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "drawtext=text='Demo':fontcolor=white:fontsize=36:x=w-tw-24:y=h-th-24" -c:a copy examples/output/sample-watermark.mp4
```

这条命令在画面右下角绘制白色 `Demo` 文字。因为修改了 video frame，视频必须重新编码；音频未被修改，所以示例使用 `-c:a copy` 直接复制。实际处理自己的文件时，先用 `ffprobe` 查看 audio codec；只有目标 MP4 container 和播放器支持该 codec 时才 copy，否则改用 `-c:a aac -b:a 128k` 重新编码音频。

## 常用变体

添加半透明黑色底框，提高可读性：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "drawtext=text='Demo':fontcolor=white:fontsize=36:box=1:boxcolor=black@0.5:boxborderw=12:x=w-tw-24:y=h-th-24" -c:a copy examples/output/sample-watermark-box.mp4
```

放到左上角：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "drawtext=text='Demo':fontcolor=white:fontsize=36:x=24:y=24" -c:a copy examples/output/sample-watermark-top-left.mp4
```

指定字体文件，适合默认字体不可用的环境：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "drawtext=fontfile=/path/to/font.ttf:text='Demo':fontcolor=white:fontsize=36:x=w-tw-24:y=h-th-24" -c:a copy examples/output/sample-watermark-fontfile.mp4
```

## 参数解释

- `-vf "drawtext=..."`：启用 video filter，在每帧画面上绘制文字。
- `text='Demo'`：水印文字。包含特殊字符时需要注意 shell quoting 和 FFmpeg filter escaping。
- `fontcolor=white`：文字颜色。
- `fontsize=36`：文字字号。
- `x=w-tw-24`：横向位置。`w` 是画面宽度，`tw` 是文字宽度，表示距离右边 24 pixels。
- `y=h-th-24`：纵向位置。`h` 是画面高度，`th` 是文字高度，表示距离底部 24 pixels。
- `-c:a copy`：音频直接复制，因为水印只改变视频画面。复制前需要确认 audio codec 被目标 container 和播放器支持；不确定时改用 `-c:a aac -b:a 128k`。

## 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-watermark.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/sample-watermark.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels -of default=nw=1 examples/output/sample-watermark.mp4
```

预期输出文件包含 video stream 和 audio stream，分辨率仍为 1280x720，并能看到音频 codec、sample_rate 和 channels。还需要播放或截图检查右下角是否出现 `Demo` 文字：

```bash
ffmpeg -y -ss 00:00:03 -i examples/output/sample-watermark.mp4 -frames:v 1 examples/output/sample-watermark-check.jpg
```

人工打开检查图，确认水印位置、大小和颜色符合预期。

## 常见错误

- `No such filter: 'drawtext'`：当前 FFmpeg 构建不包含 drawtext，需要安装启用 freetype/fontconfig 的版本。
- 找不到字体或文字不显示：运行环境缺少默认字体。指定 `fontfile=`，或安装字体和 fontconfig 配置。
- 特殊字符导致命令解析失败：水印文字中的冒号、单引号、反斜杠需要转义，复杂文本建议先写入脚本或使用稳定 quoting。
- 使用 `-c:v copy` 后失败：水印会改变视频帧，video stream 必须重新编码。
- 水印被裁切或位置不对：检查 `x`、`y` 表达式，确认使用了 `tw`、`th` 留出边距。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 文字水印命令。必须先根据 ffprobe 信息分析输入 video/audio codec、width/height，以及音频是否适合在 MP4 中直接 copy。目标是在视频右下角添加白色 Demo 文本，距离右边和底部各 24px。请说明 drawtext 对 FFmpeg 构建和字体可用性的依赖，并给出截图验收命令。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 video/audio stream、width/height 和截图检查的验收命令；如果 audio codec 不适合 copy 到目标 MP4，请改用 -c:a aac -b:a 128k。

输入文件：<input>
输出文件：<examples/output/output.mp4>
水印文字：<text>
位置：<右下角/左上角/自定义>
ffprobe 信息：
<粘贴 codec、width、height、audio codec 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行 `ffmpeg -filters | grep drawtext` 确认 filter 可用。
3. 运行基础命令生成 `examples/output/sample-watermark.mp4`。
4. 截取 `sample-watermark-check.jpg` 并人工检查右下角文字。
5. 修改 `x`、`y` 为 `24:24`，重新生成左上角版本并比较位置变化。
