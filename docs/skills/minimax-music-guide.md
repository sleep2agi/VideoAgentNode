# MiniMax Music 2.6 BGM 生成指南

> AI生成的BGM用于视频背景音乐。免费额度充足，质量足够用于短视频。

---

## CLI 命令

### 基本用法

```bash
mmx music generate \
  --prompt "Clean minimal electronic ambient, tech product demo background music, subtle and professional" \
  --instrumental \
  --genre "Electronic" \
  --mood "Calm" \
  --bpm 100 \
  --model music-2.6 \
  --out bgm.mp3
```

### 参数说明

| 参数 | 说明 | 必须 |
|------|------|------|
| --prompt | 音乐描述(英文效果更好) | 是 |
| --instrumental | 纯音乐无人声 | **必须加** |
| --genre | 流派 | 否 |
| --mood | 情绪 | 否 |
| --bpm | 节拍速度 | 否 |
| --model | music-2.6 或 music-2.6-free | 否 |
| --out | 输出文件路径 | 否 |
| --duration | 时长(秒) 30-300 | 否 |
| --instruments | 指定乐器 | 否 |
| --avoid | 排除元素 | 否 |

---

## Prompt 模板库

### 科技 Ambient (机智流标配)

```bash
mmx music generate \
  --prompt "Clean minimal electronic ambient, tech product demo background music, subtle synthesizer pads, soft hi-hats, professional and modern, no vocals" \
  --instrumental \
  --genre "Electronic" \
  --mood "Calm" \
  --bpm 100 \
  --model music-2.6 \
  --out bgm_tech_ambient.mp3
```

适用: AI资讯、产品介绍、技术解说

### Lo-fi 轻松

```bash
mmx music generate \
  --prompt "Lo-fi ambient background music, calm tech news podcast feel, soft piano chords, subtle electronic beats, vinyl crackle texture, relaxing" \
  --instrumental \
  --genre "Lo-fi" \
  --mood "Relaxed" \
  --bpm 90 \
  --model music-2.6 \
  --out bgm_lofi.mp3
```

适用: 轻松话题、日常分享

### Future Bass 高能

```bash
mmx music generate \
  --prompt "Future bass energetic tech anthem, powerful drops, euphoric build-ups, soaring synth leads, punchy drums, festival energy" \
  --instrumental \
  --genre "Future Bass" \
  --mood "Energetic" \
  --bpm 130 \
  --model music-2.6 \
  --out bgm_futurebass.mp3
```

适用: 产品发布、重磅新闻、高潮段落

### Synthwave 复古未来

```bash
mmx music generate \
  --prompt "Synthwave cinematic retro-futuristic, nostalgic but modern, neon city vibes, analog synth arpeggios, 80s inspired, driving bassline" \
  --instrumental \
  --genre "Synthwave" \
  --mood "Epic" \
  --bpm 120 \
  --model music-2.6 \
  --out bgm_synthwave.mp3
```

适用: 回顾类、发展史、对比类视频

### 洗脑循环 (短视频优化)

```bash
mmx music generate \
  --prompt "Catchy energetic electronic loop, tech product promo, driving beat, memorable hook, simple but addictive melody, designed for short video" \
  --instrumental \
  --genre "Electronic" \
  --mood "Uplifting" \
  --bpm 125 \
  --model music-2.6 \
  --out bgm_catchy.mp3
```

适用: 15-30s短视频、需要高完播率的内容

### 悬疑/紧张

```bash
mmx music generate \
  --prompt "Dark atmospheric tension, mysterious synth drones, subtle heartbeat pulse, building anxiety, cinematic suspense" \
  --instrumental \
  --genre "Cinematic" \
  --mood "Tense" \
  --bpm 80 \
  --model music-2.6 \
  --out bgm_suspense.mp3
```

适用: 问题引入、痛点描述、悬念制造

### 温暖/正能量

```bash
mmx music generate \
  --prompt "Warm uplifting acoustic feel with modern electronic production, gentle guitar, soft pads, hopeful and inspiring, sunrise energy" \
  --instrumental \
  --genre "Indie Electronic" \
  --mood "Hopeful" \
  --bpm 110 \
  --model music-2.6 \
  --out bgm_warm.mp3
```

适用: 总结、展望、正面结论

---

## 参数详解

### --genre 常用值

| Genre | 风格特点 |
|-------|----------|
| Electronic | 电子、合成器为主 |
| Ambient | 氛围、背景感强 |
| Lo-fi | 低保真、轻松 |
| Future Bass | 高能量、drop |
| Synthwave | 80s复古电子 |
| Cinematic | 电影配乐感 |
| Hip Hop | 节奏感强 |
| Classical | 古典器乐 |
| Jazz | 爵士 |
| Indie Electronic | 独立电子 |

### --mood 常用值

| Mood | 情感 |
|------|------|
| Calm | 平静 |
| Relaxed | 放松 |
| Energetic | 高能 |
| Epic | 史诗 |
| Uplifting | 积极向上 |
| Tense | 紧张 |
| Mysterious | 神秘 |
| Hopeful | 充满希望 |
| Melancholic | 忧郁 |
| Powerful | 有力量 |

### --instruments 常用值

```
synthesizer, piano, guitar, drums, bass, strings, brass,
hi-hat, pad, arpeggio, violin, cello, flute, percussion
```

### --avoid 常用值

```
vocals, singing, lyrics, guitar solo, drum solo, distortion, harsh
```

### --bpm 参考

| BPM范围 | 感觉 | 适用 |
|---------|------|------|
| 60-80 | 很慢，沉思 | 悬疑、深度分析 |
| 80-100 | 中慢，平稳 | 解说、教程 |
| 100-120 | 中等，舒适 | **日常首选** |
| 120-140 | 偏快，有活力 | 产品展示、新闻 |
| 140-160 | 快，高能 | 高潮、转场 |

---

## 免费额度

| 模型 | 限制 | 质量 |
|------|------|------|
| music-2.6-free | 无限制 | 略低(可接受) |
| music-2.6 | 700次/周 | 更好 |

**策略**: 日常用music-2.6(700次/周足够)，如果某周产量大就切music-2.6-free。

---

## 与配音混音标准参数

### 核心公式

```bash
ffmpeg -y -i video.mp4 -i narration.mp3 -i bgm.mp3 \
  -filter_complex "[1:a]volume=2.0[voice];[2:a]volume=0.06[bgm];[voice][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.7[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k -ac 2 final.mp4
```

### 参数解释

| 参数 | 值 | 含义 |
|------|------|------|
| voice volume | 2.0 | 配音放大2倍(MiniMax输出偏小) |
| bgm volume | 0.06 | BGM压到6%(只做背景氛围) |
| amix duration | first | 以配音长度为准(BGM自动截断) |
| normalize | 0 | 关闭自动归一化(避免音量跳变) |
| alimiter limit | 0.7 | 限幅器防止削顶(max不超-3dB) |
| -ac 2 | stereo | 强制双声道(忘了会单声道) |

### 为什么这些值?

- **voice=2.0**: MiniMax TTS输出的音量普遍偏小(mean约-35dB)，需要放大
- **bgm=0.06**: BGM只是氛围，不能抢配音。0.1就会干扰听感
- **alimiter=0.7**: 放大后可能削顶(-2.3dB max)，limiter保证不超-3dB
- **normalize=0**: amix默认会归一化，导致音量不可预测

### 验证混音结果

```bash
# 检查音量
ffmpeg -i final.mp4 -af volumedetect -f null - 2>&1 | grep -E "mean|max"
# 目标: mean_volume: -25 ~ -20 dB, max_volume: ≥ -6 dB

# 听一下开头和结尾
ffplay -ss 0 -t 5 final.mp4    # 开头5秒
ffplay -ss -5 final.mp4         # 结尾5秒
```

---

## BGM循环处理

### 问题: BGM太短(30s)但视频3分钟

```bash
# 方法1: 生成时指定时长
mmx music generate --prompt "..." --duration 200 --out bgm_long.mp3

# 方法2: 循环拼接
ffmpeg -stream_loop 5 -i bgm_30s.mp3 -t 180 -c:a libmp3lame -b:a 192k bgm_3min.mp3

# 方法3: 淡入淡出循环(更自然)
ffmpeg -stream_loop 5 -i bgm_30s.mp3 -t 180 \
  -af "afade=t=in:st=0:d=2,afade=t=out:st=178:d=2" \
  -c:a libmp3lame -b:a 192k bgm_3min.mp3
```

---

## BGM选择决策树

```
视频类型?
├── 技术解说/教程 → 科技Ambient (bpm 100)
├── AI新闻/资讯 → Lo-fi (bpm 90) 或 科技Ambient
├── 产品发布/重磅 → Future Bass (bpm 130)
├── 对比/评测 → Synthwave (bpm 120)
├── 短视频(<30s) → 洗脑循环 (bpm 125)
├── 问题/悬念开头 → 悬疑 (bpm 80)
└── 总结/展望结尾 → 温暖 (bpm 110)
```

---

## 多段BGM拼接(情绪变化)

对于3分钟以上的视频，可以用不同BGM配不同段落:

```python
# segments定义
segments = [
    {"bgm": "bgm_suspense.mp3", "start": 0, "end": 30},     # 开头悬念
    {"bgm": "bgm_tech.mp3", "start": 30, "end": 120},       # 主体解说
    {"bgm": "bgm_futurebass.mp3", "start": 120, "end": 150}, # 高潮
    {"bgm": "bgm_warm.mp3", "start": 150, "end": 180},      # 结尾
]
```

```bash
# 截取各段
ffmpeg -y -i bgm_suspense.mp3 -ss 0 -t 30 -af "afade=t=out:st=28:d=2" seg1.mp3
ffmpeg -y -i bgm_tech.mp3 -ss 0 -t 90 -af "afade=t=in:d=2,afade=t=out:st=88:d=2" seg2.mp3
ffmpeg -y -i bgm_futurebass.mp3 -ss 0 -t 30 -af "afade=t=in:d=1,afade=t=out:st=28:d=2" seg3.mp3
ffmpeg -y -i bgm_warm.mp3 -ss 0 -t 30 -af "afade=t=in:d=2,afade=t=out:st=28:d=2" seg4.mp3

# concat
printf "file 'seg1.mp3'\nfile 'seg2.mp3'\nfile 'seg3.mp3'\nfile 'seg4.mp3'\n" > bgm_concat.txt
ffmpeg -y -f concat -safe 0 -i bgm_concat.txt -c:a libmp3lame -b:a 192k bgm_full.mp3
```

---

## Prompt写作技巧

### 好的prompt特征

1. **具体乐器**: "soft piano", "analog synth arpeggios" 比 "nice music" 好
2. **场景化**: "tech product demo background" 比 "electronic" 具体
3. **负面排除**: "no vocals, no guitar solo" 避免不想要的元素
4. **英文优先**: 音乐描述用英文效果显著优于中文

### 差的prompt

```
# 太笼统
"好听的背景音乐"

# 中文描述
"科技感的电子音乐，适合AI新闻视频"

# 矛盾
"calm and energetic" (两个情绪冲突)
```

### 好的prompt

```
# 具体+场景
"Clean minimal electronic ambient, tech news background, soft synth pads with subtle hi-hat rhythm, professional modern feel, no vocals"

# 乐器明确
"Gentle piano arpeggios over warm pad, subtle electronic beats, hopeful and uplifting, morning energy"
```

---

## 故障排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 生成有人声 | 忘了--instrumental | 必须加--instrumental |
| 风格不对 | prompt太笼统 | 加具体乐器和场景 |
| 太短 | 默认30s | 加--duration参数 |
| 质量差 | 用了free模型 | 切换music-2.6 |
| BGM太吵 | volume太高 | 用0.06不要超过0.1 |
| 循环断裂感 | 直接loop | 加crossfade |
