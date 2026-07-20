+++
date = '2026-07-13T12:00:04+08:00'
draft = false
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程（四）：裁剪、拼接与变速'
+++

本篇处理常见时间线操作，包括截取片段、规范化后拼接视频，以及同步调整音视频速度。

## 系列目录

1. [FFmpeg 视频处理教程（一）：入门、环境与核心概念](/posts/ffmpeg-learning-00-guide/)
2. [FFmpeg 视频处理教程（二）：媒体探测与封装转换](/posts/ffmpeg-learning-01-media-basics/)
3. [FFmpeg 视频处理教程（三）：转码、压缩与分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
4. [FFmpeg 视频处理教程（四）：裁剪、拼接与变速](/posts/ffmpeg-learning-03-editing/)
5. [FFmpeg 视频处理教程（五）：提取和替换音频](/posts/ffmpeg-learning-04-audio/)
6. [FFmpeg 视频处理教程（六）：截图、字幕与水印](/posts/ffmpeg-learning-05-visual-elements/)
7. [FFmpeg 视频处理教程（七）：批处理、验收与 AI 辅助](/posts/ffmpeg-learning-06-batch-and-ai/)
8. [FFmpeg 视频处理教程（八）：API、生态与流媒体](/posts/ffmpeg-learning-07-advanced/)

## 裁剪片段

### 任务目标

使用 FFmpeg 从视频中截取指定时间范围，并理解快速关键帧裁剪与重新编码精确裁剪的差异。

### 适用场景

- 需要从长视频中截取一段用于预览、分享或后续处理。
- 需要快速得到片段，能接受起点落在附近关键帧。
- 需要更精确的起止时间，愿意用重新编码换取准确性。
- 需要让 AI 在 copy 裁剪和重新编码裁剪之间做出明确选择。

### 前置探测

先准备素材并查看时长、编码格式和 keyframe 分布：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,r_frame_rate -of default=nw=1 examples/input/sample-720p.mp4
```

如果使用复制流裁剪，输入文件的编码格式、容器和关键帧位置都会影响结果。起点不在关键帧上时，copy 裁剪通常只能从附近关键帧开始，画面起点可能与预期有偏差。

### 基础命令

快速裁剪，不重新编码：

```bash
ffmpeg -y -ss 00:00:02 -to 00:00:06 -i examples/input/sample-720p.mp4 -c copy examples/output/sample-trim-copy.mp4
```

更精确裁剪，重新编码：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -c:v libx264 -c:a aac examples/output/sample-trim-accurate.mp4
```

第一条把 `-ss` 放在 `-i` 前，FFmpeg 会先 seek 到接近目标时间的关键帧，再复制流，速度快但起点可能不精确。第二条把 `-ss` 放在输入后，并重新编码输出，通常能获得更准确的截取边界，但耗时更长且会产生一次有损重新编码。

### 常用变体

用起点加持续时长表达裁剪范围：

```bash
ffmpeg -y -ss 00:00:02 -i examples/input/sample-720p.mp4 -t 4 -c copy examples/output/sample-trim-4s-copy.mp4
```

只保留视频，不保留音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -an -c:v libx264 examples/output/sample-trim-video-only.mp4
```

重新编码并指定兼容 pixel format：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -ss 00:00:02 -to 00:00:06 -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k examples/output/sample-trim-compatible.mp4
```

### 参数解释

- `-ss 00:00:02`：设置裁剪起点。放在 `-i` 前偏向快速 seek，放在 `-i` 后偏向精确解码后截取。
- `-to 00:00:06`：设置裁剪终点，表示输出到输入时间线的 6 秒位置。
- `-t 4`：设置输出持续时长。使用时与 `-to` 二选一，避免同时指定造成混淆。
- `-c copy`：复制 video stream 和 audio stream，不重新编码，速度快、质量无损，但依赖 keyframe 和 container 兼容性。
- `-c:v libx264 -c:a aac`：重新编码视频和音频，裁剪边界更可控，但会增加耗时和质量损失。
- `-an`：禁用音频输出。

### 结果验收

检查快速裁剪输出：

```bash
ffprobe -hide_banner examples/output/sample-trim-copy.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-trim-copy.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-copy.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-copy.mp4
```

预期输出时长接近 4 秒，且能看到视频流和音频流的 `codec_name` 字段。copy 裁剪的输出时长可能因关键帧位置、时间戳处理和容器行为出现轻微偏差，属于正常现象。

检查精确裁剪输出：

```bash
ffprobe -hide_banner examples/output/sample-trim-accurate.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-trim-accurate.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-accurate.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of default=nw=1 examples/output/sample-trim-accurate.mp4
```

预期时长同样接近 4 秒，且能确认输出包含视频流和音频流。完成上述检查后，还应播放输出文件，拖到开头和结尾，确认画面切换和声音是否与预期裁剪位置一致。

### 常见错误

- 起点不精确：使用 `-c copy` 且起点不在 keyframe。改用重新编码，或接受从附近 keyframe 开始。
- 输出前几帧黑屏或音画不同步：copy 裁剪直接复制原始压缩帧和时间戳时可能出现。尝试改用重新编码。
- 把 `-to` 理解为输出时长：`-to` 是结束时间点，`-t` 才是持续时长。
- 裁剪后文件无法播放：容器或复制出的流不兼容。改用 `-c:v libx264 -c:a aac` 重新编码输出 MP4。
- 重新编码质量下降：显式设置 `-crf`、`-preset` 和 `-b:a`，并实际播放检查画面清晰度和声音是否满足需求。

### AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 裁剪命令。必须先根据 ffprobe 信息分析输入的 duration、container、video/audio codec 和 keyframe 风险。目标是截取 00:00:02 到 00:00:06 的片段。请分别给出快速 keyframe copy 裁剪和更精确的重新编码裁剪方案，并说明速度、准确性和质量损失的取舍。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 duration 与 video/audio stream codec 的验收命令。

输入文件：<input>
输出文件：<examples/output/output.mp4>
是否要求精确起点：<是/否>
ffprobe 信息：
<粘贴 format、stream、duration、codec 信息>
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行快速裁剪命令生成 `examples/output/sample-trim-copy.mp4`。
3. 运行重新编码裁剪命令生成 `examples/output/sample-trim-accurate.mp4`。
4. 用 `ffprobe` 分别检查两个文件的 duration 和 codec。
5. 播放两个输出文件，对比开头是否更接近 2 秒位置，并记录速度与准确性的差异。

## 拼接视频

### 任务目标

使用 FFmpeg 拼接多个视频。核心是建立一套可靠流程：先用 `ffprobe` 对比输入文件的 stream 参数；如果不兼容，先标准化为统一格式，再用 concat demuxer 配合 `-c copy` 直接拼接。

### 适用场景

- 需要把多个片段顺序合成为一个 MP4。
- 输入片段分辨率、帧率、音频采样率等参数不完全一致。
- 需要给视频前后添加标题卡、片头或片尾。
- 需要让 AI 先判断 stream 兼容性，再决定直接复制流拼接还是先统一转码。

### 前置探测

先准备素材并对比两个输入文件的关键 stream 字段：

```bash
test -f examples/input/sample-360p.mp4 || ./scripts/generate-samples.sh
test -f examples/input/title-card.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/input/sample-360p.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/input/sample-360p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/input/title-card.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/input/title-card.mp4
```

用 `-c copy` 拼接时，各输入文件的相关参数需要兼容。video 侧包括 codec、分辨率、帧率、time_base、pix_fmt；audio 侧包括 codec、sample_rate、channels、channel_layout。当前样例中 `sample-360p.mp4` 是 640x360、24 FPS、44100 Hz 音频；`title-card.mp4` 是 1280x720、30 FPS、48000 Hz 音频，参数不同。因此，直接用这两个原始文件做 `-c copy` concat 只适合作为风险演示，不应作为推荐主流程。

### 基础命令

推荐的主流程是先把两个片段转成同一套参数：1280x720、30 FPS、`yuv420p`、H.264、AAC、48000 Hz。

```bash
ffmpeg -y -i examples/input/sample-360p.mp4 -vf "scale=1280:720,fps=30,setpts=PTS-STARTPTS" -af "aresample=48000:first_pts=0" -c:v libx264 -pix_fmt yuv420p -c:a aac -ac 1 examples/output/concat-normalized-1.mp4
ffmpeg -y -i examples/input/title-card.mp4 -vf "scale=1280:720,fps=30,setpts=PTS-STARTPTS" -af "aresample=48000:first_pts=0" -c:v libx264 -pix_fmt yuv420p -c:a aac -ac 1 examples/output/concat-normalized-2.mp4
printf "file 'concat-normalized-1.mp4'\nfile 'concat-normalized-2.mp4'\n" > examples/output/concat-normalized-list.txt
ffmpeg -y -f concat -safe 0 -i examples/output/concat-normalized-list.txt -c copy examples/output/sample-concat.mp4
```

列表中的路径按列表文件所在目录解析。这里把 list 放在 `examples/output/`，所以列表中直接写同目录下的 normalized 文件名。

### 常用变体

风险演示：直接对当前两个原始样例做 copy concat。命令可能返回成功，但不代表输出在切换点、播放器兼容性和时长表现上可靠。

```bash
printf "file '../input/sample-360p.mp4'\nfile '../input/title-card.mp4'\n" > examples/input/concat-list.txt
ffmpeg -y -f concat -safe 0 -i examples/input/concat-list.txt -c copy examples/output/sample-concat-risk-demo.mp4
```

如果输入已经由同一编码流程生成，且探测字段一致，可以直接创建列表并 copy concat：

```bash
printf "file 'part-1.mp4'\nfile 'part-2.mp4'\n" > examples/output/concat-list-compatible.txt
ffmpeg -y -f concat -safe 0 -i examples/output/concat-list-compatible.txt -c copy examples/output/sample-concat-compatible.mp4
```

也可以用 concat filter 一步完成重新编码拼接，适合输入参数不一致、不想先生成中间文件的场景：

```bash
ffmpeg -y -i examples/input/sample-360p.mp4 -i examples/input/title-card.mp4 -filter_complex "[0:v]scale=1280:720,fps=30,setsar=1[v0];[1:v]scale=1280:720,fps=30,setsar=1[v1];[0:a]aresample=48000[a0];[1:a]aresample=48000[a1];[v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" -c:v libx264 -pix_fmt yuv420p -c:a aac examples/output/sample-concat-filter.mp4
```

### 参数解释

- `ffprobe ... codec_name,width,height,r_frame_rate,time_base,pix_fmt`：对比 video stream 是否适合 copy concat。
- `ffprobe ... codec_name,sample_rate,channels,channel_layout`：对比 audio stream 是否适合 copy concat。
- `scale=1280:720,fps=30,setpts=PTS-STARTPTS`：统一视频分辨率和帧率，同时将输出时间戳重置为 0。
- `-c:v libx264 -pix_fmt yuv420p`：统一 video codec 和 pixel format。
- `-af "aresample=48000:first_pts=0" -c:a aac -ac 1`：统一音频采样率、编码格式和声道数，同时将音频时间戳重置为 0。
- `-f concat`：使用 concat demuxer，从文本列表读取待拼接的文件。
- `-safe 0`：允许列表中使用相对路径、绝对路径或包含特殊字符的路径。处理不可信列表时要谨慎。
- `-c copy`：直接复制已标准化的流，避免对最终输出重新编码。

### 结果验收

先检查标准化片段字段：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/output/concat-normalized-1.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/output/concat-normalized-1.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/output/concat-normalized-2.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/output/concat-normalized-2.mp4
```

预期两个 normalized 片段都是 H.264、1280x720、30 FPS、`yuv420p`、AAC、48000 Hz、相同 channels。

再检查拼接输出：

```bash
ffprobe -hide_banner examples/output/sample-concat.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt -of default=nw=1 examples/output/sample-concat.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/output/sample-concat.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/concat-normalized-1.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/concat-normalized-2.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-concat.mp4
```

预期输出仍为 H.264、1280x720、30 FPS、`yuv420p`、AAC、48000 Hz，duration 接近两个 normalized 片段时长之和。最后播放输出文件，确认两个片段按列表顺序出现，切换点没有明显卡顿或音画错位。

### 常见错误

- `Impossible to open`：列表路径相对关系写错。路径是相对 concat list 文件所在目录，而不是当前 shell 目录。
- 把不兼容素材直接 `-c copy` 当作主流程：命令可能成功，但输出不可靠。先标准化，再 concat。
- 只有视频没有音频：某个输入缺少 audio stream，或 concat filter 的 stream 映射不完整。需要统一每个输入的音频结构，或明确只拼接视频。
- 时间戳警告：不同文件的 time base 或时间戳不连续。先 normalize，或用 concat filter 重新编码。
- MP4/AAC 拼接时出现 `Non-monotonous DTS` 警告：可能来自 AAC priming 或片段边界时间戳。先确认输出字段、duration 和播放效果是否正常。如果业务要求严格消除该类警告，可以改用 concat filter 重新编码最终输出，或使用更适合中间拼接的 container 流程。
- 把 concat demuxer 和 concat filter 混用：demuxer 读取列表并适合 copy compatible streams，filter 读取多个输入并重新编码，命令结构不同。

### AI 提示词模板

```text
请基于以下输入文件列表生成 FFmpeg 拼接命令。必须先根据 ffprobe 信息分析每个输入的 video codec、width/height、fps、time_base、pix_fmt、audio codec、sample_rate、channels/channel_layout。当前输入如果不兼容，不要把原始文件 -c copy concat 作为推荐主流程；请先统一转码到 H.264/AAC、1280x720、30fps、yuv420p、48000Hz，再用 concat demuxer 和 -c copy 拼接 normalized 文件。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 width/height/r_frame_rate/pix_fmt、audio sample_rate/channels、duration 的验收命令。

输入文件：
<逐行列出文件>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴每个输入的 video/audio stream 信息>
```

### 可复现实验

1. 运行 `test -f examples/input/sample-360p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 用 `ffprobe` 对比 `sample-360p.mp4` 和 `title-card.mp4` 的 video/audio 字段，确认它们参数不同。
3. 运行两条 normalized 命令生成 `examples/output/concat-normalized-1.mp4` 和 `examples/output/concat-normalized-2.mp4`。
4. 生成 `examples/output/concat-normalized-list.txt`，再运行 concat demuxer 命令生成 `examples/output/sample-concat.mp4`。
5. 用字段级 `ffprobe` 命令检查输出参数，并确认 duration 接近两个 normalized 片段时长之和。

## 调整播放速度

### 任务目标

使用 FFmpeg 同时调整视频和音频速度，理解 `setpts` 控制 video timestamp、`atempo` 控制 audio tempo 的基本方法与限制。

### 适用场景

- 需要把视频加速生成 2x 版本。
- 需要缩短教程、演示或素材预览时长。
- 需要同时处理音频，避免只有画面加速而声音不同步。
- 用 AI 生成变速命令时，需要让它区分视频和音频两条独立的处理链。

### 前置探测

先准备素材并确认输入包含 video stream 和 audio stream：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
```

如果输入没有音频，需要移除音频 filter 和 audio map，只处理视频。

### 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v];[0:a]atempo=2.0[a]" -map "[v]" -map "[a]" examples/output/sample-2x.mp4
```

`setpts=0.5*PTS` 会把视频时间戳压缩为原来的一半，所以播放速度变为 2x。`atempo=2.0` 会把音频速度变为 2x，并尽量保持音调稳定。

### 常用变体

0.5x 慢放：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=2.0*PTS[v];[0:a]atempo=0.5[a]" -map "[v]" -map "[a]" examples/output/sample-0_5x.mp4
```

4x 加速时可串联多个 `atempo`，因为单个 filter 的常用范围是 0.5 到 2.0：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.25*PTS[v];[0:a]atempo=2.0,atempo=2.0[a]" -map "[v]" -map "[a]" examples/output/sample-4x.mp4
```

无音频视频只调整画面速度：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v]" -map "[v]" -an examples/output/sample-2x-video-only.mp4
```

### 参数解释

- `-filter_complex`：同时定义视频和音频 filter graph，并给输出流命名。
- `[0:v]setpts=0.5*PTS[v]`：读取第一个输入的视频流，把 presentation timestamp 乘以 0.5，输出命名为 `[v]`。
- `[0:a]atempo=2.0[a]`：读取第一个输入的音频流，把音频 tempo 调整为 2.0 倍，输出命名为 `[a]`。
- `-map "[v]" -map "[a]"`：明确选择 filter 输出的视频和音频，避免 FFmpeg 自动选择错误 stream。
- `setpts`：只改变视频帧的时间戳，不会自动处理音频。
- `atempo`：调整音频 tempo。常用单个值范围是 `0.5` 到 `2.0`；更大或更小倍率建议串联多个 `atempo`。

### 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-2x.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,duration -of default=nw=1 examples/output/sample-2x.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,duration -of default=nw=1 examples/output/sample-2x.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/input/sample-720p.mp4
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 examples/output/sample-2x.mp4
```

预期输出包含 video stream 和 audio stream，format duration 接近输入的一半。示例输入约 8 秒，2x 输出应接近 4 秒。`stream=duration` 在某些 container 或 stream 上可能缺失，此时改用 format duration 并播放检查同步。如果 video 和 audio stream 的 duration 差异明显，需要重新检查 `setpts`、`atempo` 和 `-map`。

### 常见错误

- 只用了 `setpts`：画面速度改变，但音频仍是原速度，导致音画不同步或输出时长不符合预期。
- `atempo` 倍率过大：单个 `atempo` 常用范围为 `0.5` 到 `2.0`，超过时应串联多个 filter，例如 `atempo=2.0,atempo=2.0`。
- 倍率方向写反：2x 加速视频用 `setpts=0.5*PTS`，0.5x 慢放用 `setpts=2.0*PTS`。
- 输入没有音频：`[0:a]` 会报错。移除音频 filter 和 `-map "[a]"`，并添加 `-an`。
- 没有显式 `-map`：复杂 filter 的输出可能不会被正确写入，建议始终显式指定命名输出。

### AI 提示词模板

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

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础命令生成 `examples/output/sample-2x.mp4`。
3. 用 `ffprobe` 分别读取输入和输出 duration。
4. 确认输出时长接近输入的一半。
5. 把命令改为 `setpts=2.0*PTS` 和 `atempo=0.5`，生成慢放版本并比较时长。
