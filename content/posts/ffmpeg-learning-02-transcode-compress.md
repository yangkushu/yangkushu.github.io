+++
date = '2026-07-13T12:00:00+08:00'
draft = false
categories = ["教程"]
tags = ["FFmpeg", "视频处理"]
title = 'FFmpeg 视频处理教程（三）：转码、压缩与分辨率'
+++

本篇围绕视频重新编码展开，分别处理编码格式、文件体积和画面尺寸，并给出可验证的输出标准。

## 系列目录

1. [FFmpeg 视频处理教程（一）：入门、环境与核心概念](/posts/ffmpeg-learning-00-guide/)
2. [FFmpeg 视频处理教程（二）：媒体探测与封装转换](/posts/ffmpeg-learning-01-media-basics/)
3. [FFmpeg 视频处理教程（三）：转码、压缩与分辨率](/posts/ffmpeg-learning-02-transcode-compress/)
4. [FFmpeg 视频处理教程（四）：裁剪、拼接与变速](/posts/ffmpeg-learning-03-editing/)
5. [FFmpeg 视频处理教程（五）：提取和替换音频](/posts/ffmpeg-learning-04-audio/)
6. [FFmpeg 视频处理教程（六）：截图、字幕与水印](/posts/ffmpeg-learning-05-visual-elements/)
7. [FFmpeg 视频处理教程（七）：批处理、验收与 AI 辅助](/posts/ffmpeg-learning-06-batch-and-ai/)
8. [FFmpeg 视频处理教程（八）：API、生态与流媒体](/posts/ffmpeg-learning-07-advanced/)

## 转码

### 任务目标

使用 FFmpeg 将输入文件重新编码，按需指定输出的 video codec、audio codec、质量参数和 pixel format。本章示例将输入视频转为兼容性较好的 H.264/AAC MP4，用 CRF 控制视频质量。

### 适用场景

- 输入 codec 不被目标平台、浏览器、播放器或剪辑软件支持。
- 需要统一输出为 H.264 video、AAC audio 和 MP4 container。
- 需要调整质量、码率、pixel format 或 encoder preset。
- 需要通过 AI 工具显式指定转码参数，避免依赖 FFmpeg 自动选择的 encoder。

### 前置探测

先确认输入文件存在，并查看其媒体流：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -hide_banner examples/input/sample-720p.mp4
```

再确认当前 FFmpeg 支持 `libx264` 和 `aac`：

```bash
ffmpeg -hide_banner -encoders
```

如果 `libx264` 不存在，需要安装包含 x264 support 的 FFmpeg，或改用当前环境可用的 H.264 encoder。

### 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 23 -c:a aac -b:a 128k -pix_fmt yuv420p examples/output/sample-transcoded.mp4
ffprobe -hide_banner examples/output/sample-transcoded.mp4
```

这条命令同时重新编码视频流和音频流。输出为 H.264/AAC 编码的 MP4 文件，视频质量由 CRF 23 控制，编码速度由 `medium` preset 决定，pixel format 设为 `yuv420p`。

### 常用变体

追求更高画质时，文件通常更大：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 18 -c:a aac -b:a 160k -pix_fmt yuv420p examples/output/sample-higher-quality.mp4
```

追求更快编码时，压缩效率可能下降：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset veryfast -crf 23 -c:a aac -b:a 128k -pix_fmt yuv420p examples/output/sample-fast.mp4
```

只转码视频，复制音频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p -c:a copy examples/output/sample-video-transcoded.mp4
```

使用 `-c:a copy` 前先用 `ffprobe` 查看 audio codec。只有目标 container 和播放环境支持该 audio codec 时才直接复制；如果不确定，转为 AAC 更稳妥，例如改用 `-c:a aac -b:a 128k`。

### 参数解释

- `-c:v libx264`：使用 x264 encoder 输出 H.264 video codec。H.264 兼容性好，适合 Web、移动端和常见播放器。
- `-preset medium`：控制编码速度与压缩效率的平衡。preset 越慢，压缩效率通常越高，但耗时也越长；preset 本身不直接决定画质等级。
- `-crf 23`：Constant Rate Factor，控制视频质量。数值越小质量越高、文件通常越大；数值越大质量越低、文件通常越小。x264 常见范围是 18 到 28。
- `-c:a aac`：使用 AAC audio encoder，适合 MP4 container 和多数播放环境。
- `-b:a 128k`：设置 audio bitrate 为 128 kbps。语音可更低，音乐通常需要更高。
- `-pix_fmt yuv420p`：设置 pixel format 为 `yuv420p`，提升浏览器、播放器和移动设备兼容性。
- `examples/output/sample-transcoded.mp4`：输出 MP4 文件。MP4 container 搭配 H.264/AAC 是常见交付组合。

### 结果验收

运行：

```bash
ffprobe -hide_banner examples/output/sample-transcoded.mp4
```

预期结果：

- video stream 的 codec 为 H.264。
- audio stream 的 codec 为 AAC。
- video stream 的 pixel format 为 `yuv420p`。
- 输出文件可被常见播放器播放。

也可以只检查关键字段：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt,width,height -of default=nw=1 examples/output/sample-transcoded.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,bit_rate -of default=nw=1 examples/output/sample-transcoded.mp4
```

### 常见错误

- `Unknown encoder 'libx264'`：当前 FFmpeg 不包含 x264 encoder，需要安装完整版本或换用可用 encoder。
- 输出画面颜色或兼容性异常：缺少 `-pix_fmt yuv420p` 时，可能输出播放器兼容性较差的 pixel format。
- 文件比输入更大：转码不是必然压缩。输入本身可能已高度压缩，或 CRF 数值较低导致质量和体积更高。
- 编码速度太慢：可改用更快的 `preset`，例如 `fast` 或 `veryfast`；代价是压缩效率可能下降。
- 音频码率不符合预期：部分 AAC encoder 对低码率或多声道输入有约束。先用 `ffprobe` 查看输入声道和采样率，再调整音频码率。

### AI 提示词模板

```text
请基于以下 ffprobe 信息生成 FFmpeg 转码命令。目标输出为 MP4，video codec 使用 H.264/libx264，audio codec 使用 AAC，pixel format 使用 yuv420p。请解释 CRF、preset、audio bitrate 和 pixel format 的作用，并给出验收命令。

输入文件：<input>
输出文件：<output.mp4>
质量偏好：<质量优先/体积优先/速度优先>
ffprobe 信息：
<粘贴输入文件信息>
限制：不要覆盖源文件；如需改变分辨率请单独说明。
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础转码命令生成 `examples/output/sample-transcoded.mp4`。
3. 运行 `ffprobe -hide_banner examples/output/sample-transcoded.mp4` 查看输出 stream。
4. 使用 `ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt -of default=nw=1 examples/output/sample-transcoded.mp4` 确认 video codec 和 pixel format。
5. 调整 `-crf` 为 18 和 28 各运行一次，对比画质、文件大小和编码耗时。

## 压缩视频

### 任务目标

使用 FFmpeg 重新编码来降低视频文件体积，并理解 CRF、preset、音频码率（audio bitrate）和 `+faststart` 各自如何影响输出。本章示例选用较高 CRF 和较低音频码率，生成体积更小的 MP4。

### 适用场景

- 需要降低视频上传、分发或长期归档的成本。
- 希望在不明显影响观感的前提下缩小文件体积。
- 输入文件过大，需要生成方便分享或网页播放的轻量版本。
- 需要清楚表达“在多大画质损失下换取多大体积缩减”，而不是笼统地要求“压缩到最小”，尤其在向 AI 工具描述需求时。

### 前置探测

先准备素材并查看输入文件大小：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ls -lh examples/input/sample-720p.mp4
ffprobe -hide_banner examples/input/sample-720p.mp4
```

压缩前建议先确认输入文件的时长、分辨率、编码格式和码率。同一组压缩参数在不同内容上效果差异很大，运动复杂、噪点多或细节丰富的视频更难压到小体积。

### 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 28 -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart examples/output/sample-compressed.mp4
ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4
```

这条命令会重新编码视频流（video stream）和音频流（audio stream）。其中 `-crf 28` 比常见的 `-crf 23` 更偏向小体积，但画质也更容易出现可见损失；`-pix_fmt yuv420p` 适合 MP4/H.264/AAC 及 Web 播放场景，能提升浏览器、移动端和常见播放器的兼容性。

### 常用变体

质量更高、文件通常更大：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p -c:a aac -b:a 128k -movflags +faststart examples/output/sample-compressed-crf23.mp4
```

体积更小、质量损失更明显：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -c:v libx264 -preset slow -crf 32 -pix_fmt yuv420p -c:a aac -b:a 64k -movflags +faststart examples/output/sample-compressed-crf32.mp4
```

先缩小分辨率再压缩：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=854:-2" -c:v libx264 -preset slow -crf 26 -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart examples/output/sample-compressed-480p.mp4
```

### 参数解释

- `-c:v libx264`：使用 H.264 编码器重新编码视频。
- `-preset slow`：较慢的编码预设。通常比 `medium` 或 `fast` 压缩效率更高，但耗时也更长。
- `-crf 28`：CRF 数值越高，输出质量越低、文件通常越小。压缩不是无损过程，较高 CRF 会丢弃更多视觉细节。
- `-pix_fmt yuv420p`：输出兼容性更好的像素格式（pixel format），适合 MP4/H.264/AAC 及 Web 交付。
- `-c:a aac -b:a 96k`：把音频编码为 AAC，并采用较低的音频码率以进一步缩减体积。音乐或复杂音频可能需要更高码率。
- `-movflags +faststart`：把 MP4 的 moov atom 移到文件开头，便于网页或远程播放时更快开始播放。
- `ls -lh`：对比输入和输出文件大小，快速判断压缩是否生效。

### 结果验收

运行：

```bash
ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4
ffprobe -hide_banner examples/output/sample-compressed.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt,width,height,bit_rate -of default=nw=1 examples/output/sample-compressed.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name,bit_rate -of default=nw=1 examples/output/sample-compressed.mp4
```

预期结果：

- 输出文件通常比输入文件小，但也存在例外：某些内容本身已高度压缩，再编码反而可能增大。
- 视频流的 `codec_name` 为 `h264`，`pix_fmt` 为 `yuv420p`，并能看到 `width`、`height` 和 `bit_rate` 字段。
- 音频流的 `codec_name` 为 `aac`，并能看到 `bit_rate` 字段。
- 播放输出文件时，画面可能出现更明显的压缩痕迹，这是缩小体积的代价。

`ffprobe -hide_banner` 可以确认输出文件的容器（container）和流（stream）概览，但不能直接证明 `+faststart` 已经把 moov atom 前置。`+faststart` 更适合通过 FFmpeg 编码日志中的 `moving the moov atom to the beginning of the file`、专门工具，或针对 MP4 box 位置的检查来确认。

进一步验收时，不只看文件大小和探测字段，也要人工抽查画面细节、字幕边缘、快速运动片段和音频质量。

### 常见错误

- 文件没有变小：输入文件本身已经很小或已高度压缩，或者 CRF 过低、音频码率过高。可以提高 CRF、降低分辨率或降低音频码率。
- 画质下降明显：CRF 太高，或输入内容本身细节多、运动快。降低 CRF，例如从 28 改为 23 或 24。
- 编码时间太长：`-preset slow` 更耗时。可改用 `medium` 或 `fast`，但同等 CRF 下文件可能更大。
- 只看码率不看内容：不同画面复杂度对压缩效率影响很大。压缩参数需要结合实际视频验收。
- 误以为压缩不损失质量：使用 H.264/AAC 重新编码通常是有损压缩（lossy compression），体积变小往往意味着部分细节被舍弃。

### AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg 压缩命令。目标是在可接受画质下减小文件体积，输出为 MP4/H.264/AAC，并使用 +faststart。请说明 CRF 与文件大小、画质之间的取舍，并给出文件大小和 ffprobe 验收命令。

输入文件：<input>
输出文件：<output.mp4>
优先级：<小体积/画质/编码速度>
可接受限制：<最大文件大小、目标分辨率、音频质量要求>
ffprobe 信息：
<粘贴输入信息>
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础压缩命令生成 `examples/output/sample-compressed.mp4`。
3. 运行 `ls -lh examples/input/sample-720p.mp4 examples/output/sample-compressed.mp4` 对比文件大小。
4. 分别用 `-crf 23`、`-crf 28`、`-crf 32` 输出三个版本。
5. 对比三个版本的文件大小和画面质量，记录自己能接受的 CRF 范围。

## 调整分辨率

### 任务目标

使用 FFmpeg 的 scale 过滤器调整视频分辨率。本章示例：把 720p 输入缩放到宽度 640，通过 `-2` 让 FFmpeg 根据原始宽高比（aspect ratio）自动计算高度，同时保证输出高度为偶数。

### 适用场景

- 需要生成较小分辨率的视频版本，降低文件体积或带宽压力。
- 需要统一输出宽度，同时保持原始画面比例。
- 目标 encoder 或播放器要求视频宽高为偶数。
- 让 AI 生成 resize 命令时，需要明确的约束来避免画面拉伸和奇数尺寸错误。

### 前置探测

先准备素材并查看输入分辨率：

```bash
test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/input/sample-720p.mp4
```

确认输入文件存在 video stream。没有视频流的文件不能使用 `scale` 调整画面尺寸。

### 基础命令

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=640:-2" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-640w.mp4
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4
```

`scale=640:-2` 表示输出宽度固定为 640，高度由 FFmpeg 根据输入宽高比自动计算，并将结果调整为能被 2 整除的偶数。H.264 和 `yuv420p` 等常见组合通常要求或更偏好偶数尺寸。

这里使用 `-c:a copy` 是因为示例素材的音频编码是 AAC，目标输出也是 MP4，两者兼容。处理自己的文件时，先用 `ffprobe` 查看音频编码；只有目标容器和播放环境都支持该编码格式时才可以直接 copy，否则应转为 AAC，例如 `-c:a aac -b:a 128k`。

### 常用变体

固定高度为 360，自动计算偶数宽度：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=-2:360" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-360h.mp4
```

固定宽高，可能改变画面比例，不建议用于普通 resize：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale=640:360" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-640x360.mp4
```

只在输入宽度大于 640 时缩小，不放大小视频：

```bash
ffmpeg -y -i examples/input/sample-720p.mp4 -vf "scale='min(640,iw)':-2" -c:v libx264 -crf 23 -preset medium -c:a copy examples/output/sample-max-640w.mp4
```

以上变体都使用 `-c:a copy`，同样需要先确认音频编码与目标容器的兼容性。不兼容时改用 `-c:a aac -b:a 128k`。

### 参数解释

- `-vf "scale=640:-2"`：启用视频过滤器。`scale` 的第一个值是输出宽度，第二个值是输出高度。
- `640`：固定输出宽度为 640 像素。
- `-2`：让 FFmpeg 根据宽高比自动计算高度，并将结果调整为偶数。相比 `-1`，`-2` 更适合 H.264、YUV 4:2:0 等通常要求或更偏好偶数尺寸的输出。
- `-c:v libx264`：缩放后必须重新编码视频流，不能对视频使用 `-c:v copy`。
- `-crf 23 -preset medium`：用 CRF 和 preset 控制重新编码的质量与速度。
- `-c:a copy`：音频不受缩放影响，直接复制即可，避免音质损失和额外耗时。复制前需确认音频编码被目标容器和播放环境支持；否则转为 AAC，例如 `-c:a aac -b:a 128k`。
- `ffprobe ... width,height`：只输出视频的宽度和高度字段，方便脚本验证。

### 结果验收

运行：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4
```

预期输出包含：

```text
width=640
height=360
```

也可以只读取宽度：

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width -of default=nw=1:nk=1 examples/output/sample-640w.mp4
```

预期输出：

```text
640
```

### 常见错误

- `width not divisible by 2` 或编码器报尺寸错误：输出宽高为奇数。使用 `-2` 让 FFmpeg 自动计算偶数尺寸，或手动指定偶数宽高。
- 画面被拉伸：同时固定了宽高，且比例与原始画面不一致。使用 `scale=目标宽:-2` 或 `scale=-2:目标高`。
- 使用 `-c:v copy` 失败：缩放会改变视频帧数据，必须重新编码视频流。
- 文件不一定变小：缩放降低了像素数量，但 CRF 值、画面内容的编码难度和音频码率仍会影响最终体积。
- 字幕或叠加图形位置异常：已烧录到画面中的硬字幕或叠加层会跟随缩放；软字幕需单独处理。

### AI 提示词模板

```text
请基于以下输入信息生成 FFmpeg resize 命令。目标是把视频宽度缩放到 640，保持 aspect ratio，并确保输出宽高为偶数。视频使用 H.264/libx264 重新编码，音频直接 copy。请解释 scale 中 -2 的作用，并给出 ffprobe 验收命令。

输入文件：<input>
输出文件：<output.mp4>
目标尺寸：宽 640，高度自动
ffprobe 信息：
<粘贴 width、height、codec、pix_fmt 等信息>
限制：不要拉伸画面；不要重新编码音频，除非 container 不支持。
```

### 可复现实验

1. 运行 `test -f examples/input/sample-720p.mp4 || ./scripts/generate-samples.sh` 准备素材。
2. 运行基础 resize 命令生成 `examples/output/sample-640w.mp4`。
3. 运行 `ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of default=nw=1 examples/output/sample-640w.mp4` 检查宽高。
4. 运行只读取宽度的验收命令，确认输出为 `640`。
5. 把 `scale=640:-2` 改为 `scale=-2:360`，观察宽度也会按比例自动计算。
