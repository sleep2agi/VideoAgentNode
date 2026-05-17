# 视频制作全流程 SOP

> 基于 anet.sh 产品视频 20 轮迭代实战经验（5.85 → 8.18/10）

## 流程概览

```
选题 → 脚本 → TTS 配音 → 视觉素材 → Remotion 排版 → 渲染 → 合并音频+BGM → 质检 → 上传
```

## 1. 选题

- 在 GitHub Issue 中创建选题，标注视频类型（宣传/资讯/知识点）
- 明确目标受众、时长、发布平台
- **单篇专题比填充式多条新闻效果好**（Vincent 实测反馈）
- 时长建议：15s（社交传播）/ 45-65s（官网）/ 90-120s（B站/YouTube）

## 2. 脚本撰写

### 口语化原则
- ❌ 流水账新闻播报、模板式开场
- ✅ 直接问问题开头、用人话解释术语、给出判断
- ✅ 结尾必须有**反问**（引发转发表态）

### 产品视频结构（8 段模板）
```
Hook(10s) → Install(15s) → Architecture(20s) → Features(20s)
→ Demo(15s) → Dashboard(15s) → Security(15s) → CTA(10s)
```

### 资讯视频结构（术语视觉化）
```
s00: "你知道 XXX 是什么吗？简单说，就是…"
s01: "具体发生了什么。这说明什么？"
s02: "某人说了一句话：不只是加速旧工作"
s03: "你今天还觉得 AI 只是帮你打字快一点吗？"
```

## 3. TTS 配音（MiniMax）

```python
# MiniMax speech-2.8-hd
payload = {
    'model': 'speech-2.8-hd',
    'text': '旁白文本',
    'voice_setting': { 'voice_id': 'vincent_wenxing_v1', 'speed': 1.0 },
    'audio_setting': { 'sample_rate': 32000, 'format': 'wav' }
}
# ⚠️ audio 是 hex 编码：bytes.fromhex(resp['data']['audio'])
# ⚠️ 输出用 wav（mp3 有解码问题）
```

各段 TTS 生成后用 ffmpeg 拼接（0.8-1.0s gap）：
```bash
printf 'file s00.mp3\nfile gap.mp3\nfile s01.mp3\n...' > concat.txt
ffmpeg -f concat -safe 0 -i concat.txt -c:a libmp3lame narration.mp3
```

用 silencedetect 找段落边界，确定 Remotion 场景时间轴：
```bash
ffmpeg -i narration.mp3 -af silencedetect=noise=-30dB:d=0.7 -f null -
```

## 4. 视觉素材

### 优先级（实测经验）
1. **真实截图**（Playwright HiDPI）— 最真实，无 AI 感 ⭐推荐
2. **即梦概念图**（Dreamina 5.0）— 适合抽象概念，但有 AI 感
3. **录屏**（Playwright recordVideo）— 最真实但依赖服务器在线

### Playwright HiDPI 截图
```javascript
const page = await browser.newPage({
  viewport: { width: 1920, height: 1080 },
  deviceScaleFactor: 2  // 关键！出图更清晰
});
await page.waitForTimeout(5000); // 等数据加载完
await page.screenshot({ path: 'output.png' });
```

### Dreamina 文生图
```bash
dreamina text2image \
  --prompt="描述（中文最佳，300-400字）" \
  --ratio=16:9 --model_version=5.0 --resolution_type=4k \
  --poll=60
```
⚠️ 必须加「不要文字 no text」否则出乱码

## 5. BGM 生成（MiniMax music）

```bash
mmx music generate \
  --prompt "Clean minimal electronic ambient, tech product demo" \
  --instrumental --genre "ambient electronic" \
  --mood "professional, clean" --bpm 100 \
  --model music-2.6 --out bgm.mp3
```

BGM 加入对评分提升 +0.4（D4 节奏感 + D5 完成品感）。

## 6. Remotion 排版

### 横屏双列布局（推荐）
```tsx
const DualCol = ({ left, right }) => (
  <div style={{ display: "flex", gap: 48, width: "100%", height: "100%", alignItems: "center" }}>
    <div style={{ flex: "0 0 60%", height: "100%" }}>{left}</div>
    <div style={{ flex: 1, display: "flex", flexDirection: "column", gap: 20 }}>{right}</div>
  </div>
);
```

### 关键注意事项
- 截图用 `objectFit: "contain"`（不要 cover，会裁切）
- Sequence durationInFrames **延伸到下一场景开始**（避免黑帧）
- 场景只做 fade-in（fo=0），**不要 fade-out**（会导致两场景同时透明）

## 7. 渲染

```bash
# ⚠️ 必须指定 video-bitrate！默认 CRF 暗色内容只出 200kbps
npx remotion render CompositionID out.mp4 --codec h264 --video-bitrate "4M"
```

## 8. 合并音频（ffmpeg）

```bash
# Narration + BGM + limiter（防削顶）
ffmpeg -y \
  -i video.mp4 -i narration.mp3 -i bgm.mp3 \
  -filter_complex "[1:a]volume=2.0[voice];[2:a]volume=0.06[bgm];[voice][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.7[aout]" \
  -map 0:v -map "[aout]" \
  -c:v copy -c:a aac -b:a 192k -ac 2 \
  final.mp4
```

⚠️ **必须 `-map 0:v -map 1:a`**（Remotion 输出有空音轨，不加 map 会选空音轨导致静音）

## 9. 质检

```bash
# 黑帧检测（暗色主题用 0.995）
ffmpeg -i final.mp4 -vf blackdetect=d=0.3:pic_th=0.995 -an -f null -

# 音量检测
ffmpeg -i final.mp4 -af volumedetect -f null -
# 目标：mean -25~-20dB, max ≥ -6dB
```

## 10. 字幕硬烧（社交平台必备）

```bash
ffmpeg -i final.mp4 \
  -vf "subtitles=subs.srt:force_style='FontName=Noto Sans SC,FontSize=32,Outline=3,Shadow=2,MarginV=50,BorderStyle=4'" \
  -c:v libx264 -b:v 4M -c:a copy \
  final_subs.mp4
```

大字幕对 D8 传播潜力提升 +1.0（静音友好）。

## 11. 交付物矩阵

| 版本 | 比例 | 时长 | 字幕 | 用途 |
|------|------|------|------|------|
| 完整版 | 16:9 | 60-120s | 硬烧 | B站/YouTube |
| 精简版 | 16:9 | 30-45s | 硬烧 | 朋友圈/视频号 |
| 15s 横版 | 16:9 | 15s | 无 | 信息流广告 |
| 15s 竖版 | 9:16 | 15s | 无 | 小红书/抖音 |

## 12. 经验教训（20 轮迭代总结）

| # | 教训 | 评分影响 |
|---|------|---------|
| 1 | 码率必须 ≥3Mbps（`--video-bitrate "4M"`） | P0 |
| 2 | 暗色主题 blackdetect 误报（用 pic_th=0.995） | 避免误判 |
| 3 | Sequence gap 导致黑帧（延伸 duration） | P0 |
| 4 | fade-out 导致透明帧（fo=0 只 fade-in） | P0 |
| 5 | BGM 加分 +0.4（MiniMax music-2.6） | D4+D5 |
| 6 | 大字幕加分 +0.2（静音友好） | D8 |
| 7 | 截图 contain > cover（cover 裁切内容） | P1 |
| 8 | 录屏依赖服务器（Connection failed 废片） | 风险 |
| 9 | HiDPI 截图（deviceScaleFactor: 2）更清晰 | D2 |
| 10 | 等页面数据加载（骨架屏 = P1 bug） | D2 |
