+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：截图、字幕和水印'
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

## 截取缩略图


## 任务目标

使用 FFmpeg 从视频指定时间点截取一帧图片，生成可用于封面、预览或质量检查的 thumbnail。

## 适用场景

- 需要为视频生成封面图或列表预览图。
- 需要从固定时间点抽取画面用于人工审核。
- 需要在自动化流程中验证视频画面是否正常。
- 需要让 AI 生成单帧截图命令，而不是导出连续图片序列。

## 前置探测

先准备素材并确认视频时长大于截图时间点：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

如果 `-ss` 指定的时间超过视频时长，可能生成空文件、失败，或得到非预期帧。

## 基础命令

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 examples/output/thumbnail.jpg
```

这条命令在输入前使用 `-ss` 快速 seek 到 3 秒附近，并输出 1 帧 JPEG 图片。对封面预览来说通常足够；如果需要逐帧级精确截图，可以把 `-ss` 放在 `-i` 后。

## 常用变体

更精确地按时间点解码后取帧：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:03 -frames:v 1 examples/output/thumbnail-accurate.jpg
```

生成指定宽度的缩略图，保持 aspect ratio：

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 -vf "scale=320:-2" examples/output/thumbnail-320w.jpg
```

输出 PNG，减少 JPEG 压缩损失：

```bash
ffmpeg -y -ss 00:00:03 -i examples/input/sample-720p.mp4 -frames:v 1 examples/output/thumbnail.png
```

## 参数解释

- `-ss 00:00:03`：定位到 3 秒时间点。放在输入前速度更快，放在输入后通常更精确。
- `-i examples/input/sample-720p.mp4`：指定输入视频。
- `-frames:v 1`：只输出 1 个 video frame，避免生成图片序列。
- `thumbnail.jpg`：根据扩展名选择图片封装和编码，JPEG 适合通用预览图。
- `-vf "scale=320:-2"`：截图时顺便缩放，`-2` 自动计算偶数高度并保持 aspect ratio。

## 结果验收

运行：

```bash
test -s examples/output/thumbnail.jpg
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/thumbnail.jpg
```

预期 `test -s` 返回成功，`ffprobe` 能看到图片 stream 信息，宽度为 1280，高度为 720。也可以用图片查看器打开，确认画面不是空白或错误帧。

## 常见错误

- 输出很多图片：使用了图片序列格式但没有 `-frames:v 1`。单张缩略图必须限制帧数。
- 截图时间点没有画面：`-ss` 超过视频时长，或输入视频损坏。先用 `ffprobe` 检查 duration。
- 截到的帧不够精确：把 `-ss` 放到 `-i` 后，代价是速度可能变慢。
- 图片太大：添加 `-vf "scale=目标宽:-2"` 生成较小尺寸。
- JPEG 画质不符合预期：可使用 `-q:v` 调整质量，或改用 PNG。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 缩略图命令。必须先根据 ffprobe 信息分析输入 duration、video stream 是否存在、截图时间点是否在时长范围内。目标是在 00:00:03 截取 1 帧 JPEG。请说明 -ss 放在 -i 前后的速度和精确度差异，并给出 ffprobe 或文件大小验收命令。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出文件非空和图片 width/height 的验收命令。

输入文件：<input>
输出图片：<examples/output/output.jpg>
截图时间点：<HH:MM:SS>
目标宽度：<原始尺寸/指定宽度>
ffprobe 信息：
<粘贴 duration、width、height 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础命令生成 `examples/output/thumbnail.jpg`。
3. 运行 `test -s examples/output/thumbnail.jpg` 确认文件非空。
4. 用 `ffprobe` 检查输出图片宽高。
5. 把 `-ss` 改成 `00:00:01` 和 `00:00:05`，对比不同时间点的画面。

## 添加字幕


## 任务目标

使用 FFmpeg 为视频添加字幕，理解 soft subtitles 和 burned subtitles 的区别。

## 适用场景

- 需要给 MP4 添加可开关的外挂式字幕轨。
- 需要把字幕烧录进画面，保证所有播放器都能看到字幕。
- 需要让 AI 判断字幕文件格式、目标平台和播放器兼容性。
- 需要基于可复现的 SRT 文件演示字幕处理流程。

## 前置探测

先准备素材，并创建一个简单的 SRT 字幕文件：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
printf "1\n00:00:00,000 --> 00:00:03,000\nHello FFmpeg\n\n2\n00:00:03,000 --> 00:00:06,000\nSubtitle demo\n" > examples/input/demo.srt
ffprobe -hide_banner examples/input/sample-720p.mp4
```

SRT 时间格式是 `时:分:秒,毫秒 --> 时:分:秒,毫秒`。字幕时间需要落在视频时长范围内。

## 基础命令

添加 soft subtitles，可在播放器中开关：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/demo.srt -c:v copy -c:a copy -c:s mov_text examples/output/sample-soft-subtitle.mp4
```

烧录字幕到画面：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:a copy examples/output/sample-burned-subtitle.mp4
```

soft subtitles 会增加 subtitle stream，画面本身不变；burned subtitles 会修改 video frame，因此必须重新编码视频，字幕不能关闭。

## 常用变体

指定字幕语言元数据：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -i examples/input/demo.srt -c:v copy -c:a copy -c:s mov_text -metadata:s:s:0 language=eng examples/output/sample-soft-subtitle-eng.mp4
```

烧录字幕并指定兼容 pixel format：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:v libx264 -pix_fmt yuv420p -c:a copy examples/output/sample-burned-subtitle-compatible.mp4
```

如果输出 MP4 的音频兼容性不确定，把音频转 AAC：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "subtitles=examples/input/demo.srt" -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k examples/output/sample-burned-subtitle-aac.mp4
```

## 参数解释

- `-i examples/input/demo.srt`：把 SRT 字幕作为第二个输入。
- `-c:v copy -c:a copy -c:s mov_text`：复制已有 video/audio stream，并把字幕转为 MP4 支持的 `mov_text` subtitle codec。
- `-metadata:s:s:0 language=eng`：给第一条 subtitle stream 设置语言元数据。
- `-vf "subtitles=..."`：使用 subtitles filter，把字幕渲染到 video frame 上。
- `-c:v libx264 -pix_fmt yuv420p`：烧录字幕时重新编码视频，并输出兼容性较好的 H.264/yuv420p。
- `-c:a copy`：复制音频。若目标 container 或播放器不支持输入音频 codec，改用 `-c:a aac -b:a 128k`。

## 结果验收

检查 soft subtitle 输出：

```bash
ffprobe -hide_banner examples/output/sample-soft-subtitle.mp4
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name,codec_type -of default=nw=1 examples/output/sample-soft-subtitle.mp4
ffprobe -v error -show_streams examples/output/sample-soft-subtitle.mp4
```

预期包含 subtitle stream，`codec_name` 通常为 `mov_text`。

检查 burned subtitle 输出：

```bash
ffprobe -hide_banner examples/output/sample-burned-subtitle.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height -of default=nw=1 examples/output/sample-burned-subtitle.mp4
ffprobe -v error -select_streams s:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-burned-subtitle.mp4
```

burned subtitle 输出通常不再包含 subtitle stream，因为字幕已经变成画面的一部分。需要播放视频或截帧检查字幕是否出现在画面上：

```bash
ffmpeg -y -ss 00:00:01 -i examples/output/sample-burned-subtitle.mp4 -frames:v 1 examples/output/sample-burned-subtitle-check.jpg
```

## 常见错误

- MP4 中看不到 soft subtitles：播放器不显示或未开启字幕轨。换播放器验证，或使用 burned subtitles。
- `Subtitle encoding currently only possible from text to text or bitmap to bitmap`：字幕 codec/container 不匹配。MP4 中常用 `-c:s mov_text`。
- `subtitles` filter 不可用：当前 FFmpeg 构建缺少 libass。安装完整 FFmpeg，或只使用 soft subtitles。
- 字幕乱码：SRT 编码不是 UTF-8。先转换字幕文件编码。
- 字幕时间不对：SRT 时间轴和视频时间轴不匹配。检查字幕起止时间和视频 duration。
- 烧录字幕后无法关闭字幕：这是 burned subtitles 的特性。如果需要可开关字幕，使用 soft subtitles。

## AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 添加字幕命令。必须先根据 ffprobe 信息分析视频输入的 container、video/audio codec、duration，以及字幕文件格式和时间范围。请分别说明 soft subtitles 和 burned subtitles 的方案、兼容性和取舍。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；soft subtitles 必须给出 subtitle stream 验收命令；burned subtitles 必须给出 video stream 验收命令和截图检查命令。

视频输入：<video input>
字幕文件：<subtitle input>
输出方式：<soft/burned>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴视频 stream 信息和字幕文件摘要>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备视频。
2. 运行 `printf` 命令创建 `examples/input/demo.srt`。
3. 运行 soft subtitle 命令生成 `examples/output/sample-soft-subtitle.mp4`。
4. 运行 burned subtitle 命令生成 `examples/output/sample-burned-subtitle.mp4`。
5. 用 `ffprobe` 检查 soft subtitle 的 subtitle stream，并截帧检查 burned subtitle 是否显示在画面中。

## 添加水印


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
