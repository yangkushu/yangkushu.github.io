+++
date = '2026-07-08T10:00:00+08:00'
draft = true
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程：拼接视频'
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

使用 FFmpeg 拼接多个视频，并建立一个可靠流程：先用 `ffprobe` 对比输入参数，不兼容时先标准化，再使用 concat demuxer 和 `-c copy` 拼接标准化后的片段。

## 适用场景

- 需要把多个片段顺序合成为一个 MP4。
- 输入片段分辨率、帧率、音频采样率等参数不完全一致。
- 需要给视频前后添加标题卡、片头或片尾。
- 需要让 AI 先判断 stream compatibility，再决定直接 copy concat 还是先统一转码。

## 前置探测

先准备素材并对比两个输入文件的关键 stream 字段：

```bash
test -f examples/input/sample-360p.mp4 || ./scripts/generate-samples.sh
test -f examples/input/title-card.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/input/sample-360p.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/input/sample-360p.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,width,height,r_frame_rate,time_base,pix_fmt -of default=nw=1 examples/input/title-card.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,sample_rate,channels,channel_layout -of default=nw=1 examples/input/title-card.mp4
```

直接使用 `-c copy` 拼接时，输入文件的 video codec、width/height、fps、time_base、pix_fmt、audio codec、sample_rate、channels/channel_layout 等参数需要兼容。当前样例中 `sample-360p.mp4` 是 640x360、24 FPS、44100 Hz 音频；`title-card.mp4` 是 1280x720、30 FPS、48000 Hz 音频，参数不同。因此直接对这两个原始文件做 `-c copy` concat 只可作为风险演示，不作为推荐主流程。

## 基础命令

推荐主流程是先把两个片段标准化为同一套参数：1280x720、30 FPS、`yuv420p`、H.264、AAC、48000 Hz。

```bash
ffmpeg -y -i examples/input/sample-360p.mp4 -vf "scale=1280:720,fps=30,setpts=PTS-STARTPTS" -af "aresample=48000:first_pts=0" -c:v libx264 -pix_fmt yuv420p -c:a aac -ac 1 examples/output/concat-normalized-1.mp4
ffmpeg -y -i examples/input/title-card.mp4 -vf "scale=1280:720,fps=30,setpts=PTS-STARTPTS" -af "aresample=48000:first_pts=0" -c:v libx264 -pix_fmt yuv420p -c:a aac -ac 1 examples/output/concat-normalized-2.mp4
printf "file 'concat-normalized-1.mp4'\nfile 'concat-normalized-2.mp4'\n" > examples/output/concat-normalized-list.txt
ffmpeg -y -f concat -safe 0 -i examples/output/concat-normalized-list.txt -c copy examples/output/sample-concat.mp4
```

列表中的路径按列表文件所在目录解析。这里把 list 放在 `examples/output/`，所以列表中直接写同目录下的 normalized 文件名。

## 常用变体

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

也可以使用 concat filter 直接重新编码拼接，适合输入参数不一致且不想先生成中间文件的场景：

```bash
ffmpeg -y -i examples/input/sample-360p.mp4 -i examples/input/title-card.mp4 -filter_complex "[0:v]scale=1280:720,fps=30,setsar=1[v0];[1:v]scale=1280:720,fps=30,setsar=1[v1];[0:a]aresample=48000[a0];[1:a]aresample=48000[a1];[v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" -c:v libx264 -pix_fmt yuv420p -c:a aac examples/output/sample-concat-filter.mp4
```

## 参数解释

- `ffprobe ... codec_name,width,height,r_frame_rate,time_base,pix_fmt`：对比 video stream 是否适合 copy concat。
- `ffprobe ... codec_name,sample_rate,channels,channel_layout`：对比 audio stream 是否适合 copy concat。
- `scale=1280:720,fps=30,setpts=PTS-STARTPTS`：把视频统一到相同 resolution 和 frame rate，并让输出时间戳从 0 开始。
- `-c:v libx264 -pix_fmt yuv420p`：统一 video codec 和 pixel format。
- `-af "aresample=48000:first_pts=0" -c:a aac -ac 1`：统一 audio sample rate、audio codec 和 channels，并让音频时间戳从 0 开始。
- `-f concat`：使用 concat demuxer，从文本列表读取要拼接的文件。
- `-safe 0`：允许列表中使用相对路径、绝对路径或包含特殊字符的路径。处理不可信列表时要谨慎。
- `-c copy`：复制 normalized streams，不重新编码最终拼接文件。

## 结果验收

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

## 常见错误

- `Impossible to open`：列表路径相对关系写错。路径是相对 concat list 文件所在目录，而不是当前 shell 目录。
- 把不兼容素材直接 `-c copy` 当作主流程：命令可能成功，但输出不可靠。先标准化，再 concat。
- 只有视频没有音频：某个输入缺少 audio stream，或 concat filter 的 stream 映射不完整。需要统一每个输入的音频结构，或明确只拼接视频。
- 时间戳警告：不同文件的 time base 或时间戳不连续。先 normalize，或用 concat filter 重新编码。
- MP4/AAC 拼接时出现 `Non-monotonous DTS`：可能来自 AAC priming 或片段边界时间戳。先确认输出字段、duration 和播放效果；如果业务要求严格消除该类警告，改用 concat filter 重新编码最终输出，或使用更适合中间拼接的 container 流程。
- 把 concat demuxer 和 concat filter 混用：demuxer 读取列表并适合 copy compatible streams，filter 读取多个输入并重新编码，命令结构不同。

## AI 提示词模板

```text
请基于以下输入文件列表生成 FFmpeg 拼接命令。必须先根据 ffprobe 信息分析每个输入的 video codec、width/height、fps、time_base、pix_fmt、audio codec、sample_rate、channels/channel_layout。当前输入如果不兼容，不要把原始文件 -c copy concat 作为推荐主流程；请先统一转码到 H.264/AAC、1280x720、30fps、yuv420p、48000Hz，再用 concat demuxer 和 -c copy 拼接 normalized 文件。

硬性要求：输出必须写入 examples/output/；不要覆盖源文件；如果输入路径和输出路径相同，必须拒绝并说明原因；必须给出 width/height/r_frame_rate/pix_fmt、audio sample_rate/channels、duration 的验收命令。

输入文件：
<逐行列出文件>
输出文件：<examples/output/output.mp4>
ffprobe 信息：
<粘贴每个输入的 video/audio stream 信息>
```

## 可复现实验

1. 运行 `test -f examples/input/sample-360p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 用 `ffprobe` 对比 `sample-360p.mp4` 和 `title-card.mp4` 的 video/audio 字段，确认它们参数不同。
3. 运行两条 normalized 命令生成 `examples/output/concat-normalized-1.mp4` 和 `examples/output/concat-normalized-2.mp4`。
4. 生成 `examples/output/concat-normalized-list.txt`，再运行 concat demuxer 命令生成 `examples/output/sample-concat.mp4`。
5. 用字段级 `ffprobe` 命令检查输出参数，并确认 duration 接近两个 normalized 片段时长之和。
