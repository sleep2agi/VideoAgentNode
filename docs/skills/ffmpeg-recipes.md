# ffmpeg 常用配方

> 视频制作中反复使用的ffmpeg命令。每一条都经过实际验证，注释标注了踩过的坑。

---

## 混音 (配音 + BGM)

### 标准混音命令

```bash
ffmpeg -y -i video.mp4 -i narration.mp3 -i bgm.mp3 \
  -filter_complex "[1:a]volume=2.0[voice];[2:a]volume=0.06[bgm];[voice][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.7[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k -ac 2 final.mp4
```

**关键参数说明:**
- `volume=2.0` 配音: MiniMax TTS输出偏小，需要放大
- `volume=0.06` BGM: 只做氛围背景，不能干扰语音
- `normalize=0`: 关闭amix自动归一化(否则音量不可控)
- `alimiter=limit=0.7`: 防止放大后削顶
- `-ac 2`: 强制立体声(忘了会出单声道)

### 踩坑: Remotion输出有空音轨

Remotion渲染的视频包含一个空的音频轨道。ffmpeg默认会选择这个空轨。

```bash
# 错误: 最终视频无声音
ffmpeg -i remotion_output.mp4 -i narration.mp3 -c:v copy -c:a aac output.mp4

# 正确: 显式指定音频来源
ffmpeg -i remotion_output.mp4 -i narration.mp3 \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 192k output.mp4
```

**铁律: 永远写 `-map 0:v -map 1:a`，不要依赖自动选择。**

### 只替换音频(不重编码视频)

```bash
ffmpeg -y -i video.mp4 -i new_audio.mp3 \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 192k -ac 2 \
  -shortest output.mp4
```

---

## 字幕硬烧

### 标准命令

```bash
ffmpeg -i input.mp4 \
  -vf "subtitles=subs.srt:force_style='FontName=Noto Sans SC,FontSize=32,Outline=3,Shadow=2,MarginV=50,BorderStyle=4,BackColour=&H80000000&'" \
  -c:v libx264 -b:v 4M -c:a copy output.mp4
```

### force_style 参数详解

| 参数 | 值 | 说明 |
|------|------|------|
| FontName | Noto Sans SC | 中文字体(需已安装) |
| FontSize | 32 | 字号(竖版用28-32) |
| Outline | 3 | 描边粗细 |
| Shadow | 2 | 阴影距离 |
| MarginV | 50 | 底部边距(像素) |
| BorderStyle | 4 | 4=背景框样式 |
| BackColour | &H80000000& | 半透明黑底 |
| PrimaryColour | &HFFFFFF& | 白色文字 |
| Bold | 1 | 加粗 |

### 竖版视频字幕(9:16)

```bash
ffmpeg -i input_vertical.mp4 \
  -vf "subtitles=subs.srt:force_style='FontName=Noto Sans SC,FontSize=28,Outline=2,Shadow=1,MarginV=120,BorderStyle=4,BackColour=&H80000000&,Alignment=2'" \
  -c:v libx264 -b:v 4M -c:a copy output.mp4
```

MarginV要大一些(120)，避免被播放器控件遮挡。

---

## 音量检测

### volumedetect

```bash
ffmpeg -i file.mp4 -af volumedetect -f null - 2>&1 | grep -E "mean|max"
```

输出:
```
[Parsed_volumedetect_0] mean_volume: -22.3 dB
[Parsed_volumedetect_0] max_volume: -4.1 dB
```

### 合格标准

| 指标 | 目标范围 | 不合格 |
|------|----------|--------|
| mean_volume | -25 ~ -20 dB | <-30(太小) >-15(太大) |
| max_volume | ≥ -6 dB | <-10(太小) >-1(削顶) |

### 修复音量过小

```bash
# 先检测当前音量
ffmpeg -i quiet.mp4 -af volumedetect -f null - 2>&1 | grep mean
# mean_volume: -35.2 dB → 需要+12dB

ffmpeg -y -i quiet.mp4 -af "volume=12dB,alimiter=limit=0.8" -c:v copy output.mp4
```

---

## 黑帧检测

### 标准命令

```bash
ffmpeg -i file.mp4 -vf blackdetect=d=0.3:pic_th=0.995 -an -f null - 2>&1 | grep black
```

### 参数说明

| 参数 | 值 | 说明 |
|------|------|------|
| d | 0.3 | 最小黑帧持续时间(秒) |
| pic_th | 0.995 | 像素黑阈值(暗色主题用0.995) |

### 关于 pic_th 的坑

**默认0.98会误报**: 暗色主题(深灰/深蓝)的合法画面会被误判为黑帧。

```bash
# 暗色主题视频: 用0.995
ffmpeg -i dark_theme.mp4 -vf blackdetect=d=0.3:pic_th=0.995 -an -f null -

# 亮色主题视频: 用0.98(默认)
ffmpeg -i light_theme.mp4 -vf blackdetect=d=0.3:pic_th=0.98 -an -f null -
```

### 合格标准

- 开头不能有>0.1s黑帧(首帧铁律)
- 中间不能有>0.5s连续黑帧(场景切换gap)
- 结尾不能有>1s黑帧(尾部留白)

---

## 拼接 (concat)

### 相同编码拼接(快速，不重编码)

```bash
# 创建列表文件
printf "file 'clip01.mp4'\nfile 'clip02.mp4'\nfile 'clip03.mp4'\n" > concat_list.txt

# 拼接
ffmpeg -y -f concat -safe 0 -i concat_list.txt -c copy output.mp4
```

### 不同编码拼接(重编码)

```bash
ffmpeg -y -f concat -safe 0 -i concat_list.txt \
  -c:v libx264 -b:v 4M -c:a aac -b:a 192k -ac 2 output.mp4
```

### Python生成concat列表

```python
import os

clips = sorted([f for f in os.listdir("clips/") if f.endswith(".mp4")])
with open("concat_list.txt", "w") as f:
    for clip in clips:
        f.write(f"file 'clips/{clip}'\n")
```

---

## 静音 Gap 生成

### 生成指定时长的静音

```bash
# 0.8秒立体声静音(用于段落间隔)
ffmpeg -y -f lavfi -i anullsrc=r=44100:cl=stereo -t 0.8 -q:a 2 gap.mp3

# 1.5秒静音(用于幕间)
ffmpeg -y -f lavfi -i anullsrc=r=44100:cl=stereo -t 1.5 -q:a 2 gap_long.mp3
```

### 配音拼接带间隔

```bash
# 列表文件
cat > narration_concat.txt << 'EOF'
file 'parts/scene01.mp3'
file 'parts/gap.mp3'
file 'parts/scene02.mp3'
file 'parts/gap.mp3'
file 'parts/scene03.mp3'
EOF

ffmpeg -y -f concat -safe 0 -i narration_concat.txt \
  -c:a libmp3lame -b:a 192k narration_full.mp3
```

---

## silencedetect 段落边界

### 检测配音中的自然停顿

```bash
ffmpeg -i narration.mp3 -af silencedetect=noise=-30dB:d=0.7 -f null - 2>&1 | grep silence
```

输出:
```
[silencedetect] silence_start: 12.456
[silencedetect] silence_end: 13.234 | silence_duration: 0.778
[silencedetect] silence_start: 28.901
[silencedetect] silence_end: 29.712 | silence_duration: 0.811
```

### 参数调优

| noise | d | 用途 |
|-------|---|------|
| -30dB | 0.7s | 标准段落检测 |
| -25dB | 0.5s | 更敏感(短句间停顿) |
| -35dB | 1.0s | 只检测明显大停顿 |

### 用途: 确定视频场景切换时间点

silence_end 时间点 = 配音段落结束 = 适合切换画面的时刻

---

## 图片转视频 (Ken Burns Effect)

### 缩放推进

```bash
ffmpeg -loop 1 -i img.png -t 5 \
  -vf "zoompan=z='min(zoom+0.001,1.05)':d=150:s=1920x1080:fps=30" \
  -c:v libx264 -b:v 4M -pix_fmt yuv420p out.mp4
```

### 从中心放大

```bash
ffmpeg -loop 1 -i img.png -t 8 \
  -vf "zoompan=z='min(zoom+0.0008,1.08)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=240:s=1080x1920:fps=30" \
  -c:v libx264 -b:v 4M -pix_fmt yuv420p out_vertical.mp4
```

### 竖版图片转视频(9:16)

```bash
ffmpeg -loop 1 -i vertical_img.png -t 6 \
  -vf "zoompan=z='min(zoom+0.0007,1.04)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=180:s=1080x1920:fps=30" \
  -c:v libx264 -b:v 4M -pix_fmt yuv420p vertical_video.mp4
```

### 平移(从左到右)

```bash
ffmpeg -loop 1 -i wide_img.png -t 8 \
  -vf "zoompan=z=1.1:x='if(eq(on,0),0,x+2)':y='ih/2-(ih/zoom/2)':d=240:s=1920x1080:fps=30" \
  -c:v libx264 -b:v 4M -pix_fmt yuv420p pan_lr.mp4
```

---

## 录屏处理

### 慢放

```bash
# 2倍慢放(录屏太快看不清)
ffmpeg -i recording.mp4 -filter_complex "[0:v]setpts=2.0*PTS[v]" \
  -map "[v]" -c:v libx264 -b:v 4M -an slow.mp4
```

### 慢放 + 循环

```bash
# 慢放2倍 + 循环2次
ffmpeg -stream_loop 2 -i recording.mp4 \
  -filter_complex "[0:v]setpts=2.0*PTS[v]" \
  -map "[v]" -c:v libx264 -b:v 4M -an looped_slow.mp4
```

### 裁剪录屏(去掉状态栏/dock)

```bash
# 裁剪: 从(0,50)开始，取1920x1000区域
ffmpeg -i recording.mp4 -vf "crop=1920:1000:0:50" -c:v libx264 -b:v 4M cropped.mp4
```

### 录屏缩放到视频分辨率

```bash
# 录屏2560x1440 → 视频1920x1080
ffmpeg -i recording.mp4 -vf "scale=1920:1080:flags=lanczos" \
  -c:v libx264 -b:v 4M -c:a copy scaled.mp4
```

---

## 音效合成

### Whoosh (转场音效)

```bash
ffmpeg -y -f lavfi \
  -i "sine=frequency=800:duration=0.3" \
  -af "afade=t=in:d=0.05,afade=t=out:st=0.15:d=0.15,aecho=0.6:0.3:20:0.5" \
  whoosh.wav
```

### Click (按钮点击)

```bash
ffmpeg -y -f lavfi \
  -i "sine=frequency=1200:duration=0.05" \
  -af "afade=t=out:d=0.04" \
  click.wav
```

### Notification (提示音)

```bash
ffmpeg -y -f lavfi \
  -i "sine=frequency=880:duration=0.2" \
  -af "afade=t=in:d=0.02,afade=t=out:st=0.1:d=0.1" \
  notification.wav
```

### 混入视频的特定时间点

```bash
# 在3.5秒处插入whoosh音效
ffmpeg -y -i video.mp4 -i whoosh.wav \
  -filter_complex "[1:a]adelay=3500|3500,volume=0.5[sfx];[0:a][sfx]amix=inputs=2:duration=first[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac output.mp4
```

---

## ffprobe 检查

### 完整信息

```bash
ffprobe -v quiet -print_format json -show_format -show_streams file.mp4
```

### 快速检查关键信息

```bash
# 时长
ffprobe -v quiet -show_entries format=duration -of csv=p=0 file.mp4

# 分辨率
ffprobe -v quiet -select_streams v:0 -show_entries stream=width,height -of csv=p=0 file.mp4

# 码率
ffprobe -v quiet -show_entries format=bit_rate -of csv=p=0 file.mp4

# 音频信息
ffprobe -v quiet -select_streams a:0 -show_entries stream=codec_name,channels,sample_rate -of csv=p=0 file.mp4
```

### 批量检查脚本

```bash
for f in clips/*.mp4; do
  dur=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$f")
  res=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=width,height -of csv=p=0 "$f")
  echo "$f: ${dur}s, ${res}"
done
```

---

## 格式转换

### MOV → MP4 (不重编码)

```bash
ffmpeg -i input.mov -c:v copy -c:a copy output.mp4
```

### WebM → MP4

```bash
ffmpeg -i input.webm -c:v libx264 -b:v 4M -c:a aac -b:a 192k output.mp4
```

### 提取音频

```bash
ffmpeg -i video.mp4 -vn -c:a libmp3lame -b:a 192k audio.mp3
```

### 提取首帧

```bash
ffmpeg -i video.mp4 -vframes 1 -q:v 2 first_frame.jpg
```

---

## 视频码率相关

### Remotion默认CRF的坑

Remotion默认使用CRF模式，对暗色主题视频只输出200kbps，画质极差。

```bash
# Remotion渲染时必须指定码率
npx remotion render --video-bitrate "4M" composition_id output.mp4

# 如果已经渲染了低码率，重编码
ffmpeg -i low_bitrate.mp4 -c:v libx264 -b:v 4M -c:a copy high_bitrate.mp4
```

### 码率参考

| 内容类型 | 推荐码率 | 文件大小参考(1min) |
|----------|----------|-------------------|
| 简单动效(Remotion) | 4M | ~30MB |
| 录屏/截图 | 4-6M | ~30-45MB |
| 即梦视频素材 | 6-8M | ~45-60MB |
| 最终输出(混合) | 4M | ~30MB |

---

## 实用组合命令

### 完整的后期处理 Pipeline

```bash
# 1. 合并视频+配音+BGM
ffmpeg -y -i remotion_output.mp4 -i narration.mp3 -i bgm.mp3 \
  -filter_complex "[1:a]volume=2.0[voice];[2:a]volume=0.06[bgm];[voice][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.7[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k -ac 2 merged.mp4

# 2. 硬烧字幕
ffmpeg -y -i merged.mp4 \
  -vf "subtitles=subs.srt:force_style='FontName=Noto Sans SC,FontSize=32,Outline=3,Shadow=2,MarginV=50'" \
  -c:v libx264 -b:v 4M -c:a copy final.mp4

# 3. 质检
ffmpeg -i final.mp4 -af volumedetect -f null - 2>&1 | grep -E "mean|max"
ffmpeg -i final.mp4 -vf blackdetect=d=0.3:pic_th=0.995 -an -f null - 2>&1 | grep black
ffprobe -v quiet -show_entries format=duration,bit_rate -show_entries stream=width,height,codec_name -of json final.mp4
```

### 快速预览(降低分辨率加速)

```bash
ffmpeg -i input.mp4 -vf scale=480:-2 -c:v libx264 -preset ultrafast -c:a copy preview.mp4
```
