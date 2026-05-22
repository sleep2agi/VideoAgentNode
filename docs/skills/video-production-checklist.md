# 视频制作完整 Checklist

> 20+轮视频制作迭代总结的铁律清单。每一条都是用废片换来的教训。

---

## 首帧铁律

**Frame 0 必须是完整的封面画面。绝不允许:**
- 黑帧开头
- fade-in从黑色渐显
- loading/skeleton状态
- 未渲染完的过渡状态

### 验证方法

```bash
# Remotion: 导出首帧检查
npx remotion still --frame=0 composition_id first_frame.png

# ffmpeg: 从视频提取首帧
ffmpeg -i output.mp4 -vframes 1 -q:v 2 first_frame.jpg

# 目视确认: 打开first_frame.jpg，是否是完整封面
```

### 为什么这条是铁律

- 平台封面抓取: B站/抖音/小红书都从frame 0截取封面
- 首帧决定点击率: 用户在feed流里看到的就是首帧
- 黑帧=废片: 封面是黑的没人会点进来

---

## 渲染前检查 (Pre-render)

### Remotion Composition 配置

- [ ] **durationInFrames**: 必须 = 配音时长(帧) + buffer(30-60帧)

```typescript
// 计算方法
const narrationDuration = 52.6; // 秒
const fps = 30;
const buffer = 30; // 1秒buffer
const durationInFrames = Math.ceil(narrationDuration * fps) + buffer;
// = Math.ceil(52.6 * 30) + 30 = 1578 + 30 = 1608
```

- [ ] **Sequence gap**: 所有Sequence之间不能有空隙

```typescript
// 错误: Scene2在frame 300开始，但Scene1只到frame 280，中间有20帧gap → 黑帧
<Sequence from={0} durationInFrames={280}>  {/* Scene 1 */}
<Sequence from={300} durationInFrames={200}>  {/* Scene 2 - GAP! */}

// 正确: 每个Scene的durationInFrames延伸到下一个Scene的from
<Sequence from={0} durationInFrames={300}>  {/* Scene 1 - 延伸到300 */}
<Sequence from={300} durationInFrames={200}>  {/* Scene 2 - 无gap */}
```

- [ ] **只 fade-in，不 fade-out**: 场景切换用下一个场景的fade-in覆盖

```typescript
// 错误: fade-out会导致透明→显示底层黑色
<Sequence>
  <div style={{opacity: interpolate(frame, [0, 10, 80, 90], [0, 1, 1, 0])}}>
    {/* fo到0时会露出黑底 */}
  </div>
</Sequence>

// 正确: 只做fade-in (fo=0)
<Sequence>
  <div style={{opacity: interpolate(frame, [0, 15], [0, 1], {extrapolateRight: 'clamp'})}}>
    {/* 始终保持不透明 */}
  </div>
</Sequence>
```

- [ ] **截图 objectFit: contain**: 不要用cover(会裁切重要内容)

```typescript
// 错误: cover会裁切截图边缘
<Img src={screenshot} style={{objectFit: 'cover', width: '100%', height: '100%'}} />

// 正确: contain保留完整截图
<Img src={screenshot} style={{objectFit: 'contain', width: '100%', height: '100%'}} />
```

- [ ] **视频码率**: 必须指定 `--video-bitrate "4M"`

```bash
# 正确
npx remotion render --video-bitrate "4M" CompositionId output.mp4

# 错误(默认CRF，暗色主题只出200kbps)
npx remotion render CompositionId output.mp4
```

---

## 合并后检查 (Post-merge)

### 音量检测

```bash
ffmpeg -i final.mp4 -af volumedetect -f null - 2>&1 | grep -E "mean|max"
```

- [ ] mean_volume: -25 ~ -20 dB
- [ ] max_volume: ≥ -6 dB (不能太小)
- [ ] max_volume: ≤ -1 dB (不能削顶)

### 黑帧检测

```bash
ffmpeg -i final.mp4 -vf blackdetect=d=0.3:pic_th=0.995 -an -f null - 2>&1 | grep black
```

- [ ] 无输出 = 通过(没有黑帧)
- [ ] 如果有输出，检查时间点是否在场景切换处
- [ ] **pic_th=0.995**: 暗色主题必须用这个值(0.98会误报)

### ffprobe 验证

```bash
ffprobe -v quiet -print_format json -show_format -show_streams final.mp4 | python3 -c "
import json, sys
data = json.load(sys.stdin)
fmt = data['format']
print(f\"Duration: {float(fmt['duration']):.1f}s\")
print(f\"Bitrate: {int(fmt['bit_rate'])//1000}kbps\")
for s in data['streams']:
    if s['codec_type'] == 'video':
        print(f\"Video: {s['width']}x{s['height']} {s['codec_name']}\")
    elif s['codec_type'] == 'audio':
        print(f\"Audio: {s['codec_name']} {s.get('channels','?')}ch {s.get('sample_rate','?')}Hz\")
"
```

- [ ] 分辨率正确 (1080x1920竖版 或 1920x1080横版)
- [ ] 时长与配音匹配 (误差<2s)
- [ ] 有音频轨道
- [ ] 音频是stereo (2ch)
- [ ] 码率 ≥ 2000kbps

### 首帧目视确认

```bash
ffmpeg -i final.mp4 -vframes 1 -q:v 2 /tmp/check_frame0.jpg
open /tmp/check_frame0.jpg  # macOS
```

- [ ] 画面完整，不是黑帧
- [ ] 文字清晰可读
- [ ] 没有AI文字artifacts
- [ ] 构图适合做封面

---

## 上传前检查 (Pre-upload)

- [ ] **文件大小合理**: 1分钟约10-30MB，3分钟约30-90MB

```bash
ls -lh final.mp4
# 期望: 1min ~15MB, 3min ~45MB
# 异常: <5MB(码率太低) 或 >200MB(需要压缩)
```

- [ ] **OSS上传后URL可访问**

```bash
# 上传
aliyun oss cp final.mp4 oss://vincent-lab/intern/videos/episode_name.mp4 -f

# 验证(用curl检查HTTP 200)
curl -I https://c.vansin.top/intern/videos/episode_name.mp4
```

- [ ] **不同设备播放测试**
  - 手机竖屏播放(竖版视频)
  - 电脑横屏播放(横版视频)
  - 检查字幕是否被播放器UI遮挡

---

## 踩过的所有坑 (20轮血泪总结)

### P0 级 (直接导致废片)

#### 1. 码率致命坑

**现象**: Remotion渲染的视频看起来模糊、有色块
**原因**: 默认CRF模式对暗色内容只分配200kbps
**解决**: 必须加 `--video-bitrate "4M"`

```bash
# 验证码率
ffprobe -v quiet -show_entries format=bit_rate -of csv=p=0 output.mp4
# 如果<1000000(1Mbps)就是有问题
```

#### 2. 静音bug

**现象**: 混音后最终视频完全没有声音
**原因**: Remotion输出包含空音轨，ffmpeg自动选了空的
**解决**: 永远显式指定 `-map 0:v -map 1:a`

```bash
# 检查输入有几条音轨
ffprobe -v quiet -show_entries stream=index,codec_type -of csv input.mp4
# 如果看到 audio 但实际无声，就是空音轨
```

#### 3. 黑帧 (Sequence Gap)

**现象**: 视频中间突然闪黑
**原因**: 两个Sequence之间有时间空隙，或者fade-out导致透明
**解决**:
- 每个Sequence的durationInFrames延伸到下一个from值
- 场景只做fade-in，不做fade-out

#### 4. 截图裁切

**现象**: 产品截图显示不全，边缘被切掉
**原因**: `objectFit: 'cover'` 会缩放裁切以填满容器
**解决**: 改用 `objectFit: 'contain'`，宁可留黑边也不裁切

### P1 级 (影响质量)

#### 5. 骨架屏/loading状态

**现象**: 截图中显示的是loading spinner或骨架屏
**原因**: 页面截图时数据还没加载完
**解决**: Puppeteer截图前加 `waitForTimeout(8000)` 或 `waitForSelector`

```javascript
await page.goto(url);
await page.waitForTimeout(8000);  // 等8秒确保加载完
await page.screenshot({path: 'screenshot.png'});
```

#### 6. 音频削顶

**现象**: 配音某些字出现"噗"的爆音
**原因**: volume放大后max超过0dB
**解决**: 加 `alimiter=limit=0.7`

```bash
# 检查是否削顶
ffmpeg -i file.mp4 -af volumedetect -f null - 2>&1 | grep max
# max_volume: -0.3 dB → 有削顶风险!
# max_volume: -2.3 dB → 加了limiter后正常
```

#### 7. 单声道

**现象**: 播放时只有左耳/右耳有声音，或某些设备无声
**原因**: 合成时忘了 `-ac 2`
**解决**: 每个ffmpeg音频输出命令都加 `-ac 2`

#### 8. 暗色主题误报黑帧

**现象**: blackdetect报了大量"黑帧"但实际画面正常(只是深色)
**原因**: pic_th=0.98(默认)对深色主题太敏感
**解决**: 暗色主题用 `pic_th=0.995`

#### 9. 录屏依赖外部服务

**现象**: 录屏中出现"Connection failed"/"502"等错误
**原因**: 录屏时目标服务恰好挂了
**解决**:
- 录屏前确认服务状态
- 准备备用静态截图方案
- 录屏后立即检查，有问题马上重录

#### 10. AI感太强

**现象**: 即梦生成的概念图一看就是AI生成的(过度光滑/色彩过饱和)
**原因**: AI生图的固有风格特征
**解决**:
- 产品视频: 用真实截图/录屏代替AI图
- 不可避免时: 降低饱和度/加噪点/用作转场而非主体

### P2 级 (体验细节)

#### 11. 字幕被遮挡

**现象**: 底部字幕被视频播放器的进度条遮挡
**解决**: `MarginV=50`(横版) 或 `MarginV=120`(竖版)

#### 12. 配音节奏不对

**现象**: 画面和配音对不上，配音说完了画面还没切
**解决**: 用silencedetect确定段落边界，视频场景切换对齐到静音点

#### 13. BGM太吵

**现象**: 背景音乐盖过了配音
**解决**: BGM volume不能超过0.06(对应voice volume=2.0时)

#### 14. 首帧不适合做封面

**现象**: 视频首帧是过渡动画中的某一帧，不适合做封面
**解决**: 第一个场景必须有>15帧的静态画面(0.5秒)

#### 15. 视频结尾突然截断

**现象**: 配音最后一个字还没说完就结束了
**解决**: durationInFrames加30-60帧buffer(1-2秒)

#### 16. 文字出现太快

**现象**: 打字机效果文字还没打完就切场景了
**解决**: 打字机速度根据文字量调整，确保完成后有1秒停留

#### 17. 色彩空间不一致

**现象**: 不同来源的素材拼在一起颜色跳变
**解决**: 统一用 `-pix_fmt yuv420p`

#### 18. 音频采样率不匹配

**现象**: concat时报错或出现杂音
**解决**: 所有音频统一44100Hz: `-ar 44100`

#### 19. 竖版视频横屏播放

**现象**: 竖版视频在某些播放器横着播放
**解决**: 确认输出的width<height (1080x1920不是1920x1080)

#### 20. 文件命名含中文/空格

**现象**: 脚本处理时路径解析错误
**解决**: 文件名只用英文+数字+下划线+连字符

---

## 快速质检脚本

### 一键全检

```bash
#!/bin/bash
# qa_check.sh - 视频质量一键检查

FILE=$1
if [ -z "$FILE" ]; then
    echo "Usage: qa_check.sh <video_file>"
    exit 1
fi

echo "=== File Info ==="
ls -lh "$FILE"

echo ""
echo "=== Format ==="
ffprobe -v quiet -show_entries format=duration,bit_rate -of default=noprint_wrappers=1 "$FILE"
ffprobe -v quiet -select_streams v:0 -show_entries stream=width,height,codec_name -of default=noprint_wrappers=1 "$FILE"
ffprobe -v quiet -select_streams a:0 -show_entries stream=codec_name,channels,sample_rate -of default=noprint_wrappers=1 "$FILE"

echo ""
echo "=== Volume ==="
ffmpeg -i "$FILE" -af volumedetect -f null - 2>&1 | grep -E "mean|max"

echo ""
echo "=== Black Frames ==="
BLACKS=$(ffmpeg -i "$FILE" -vf blackdetect=d=0.3:pic_th=0.995 -an -f null - 2>&1 | grep -c "black_start")
if [ "$BLACKS" -eq 0 ]; then
    echo "PASS: No black frames detected"
else
    echo "WARN: $BLACKS black frame segments detected"
    ffmpeg -i "$FILE" -vf blackdetect=d=0.3:pic_th=0.995 -an -f null - 2>&1 | grep black
fi

echo ""
echo "=== First Frame ==="
ffmpeg -y -i "$FILE" -vframes 1 -q:v 2 /tmp/qa_frame0.jpg 2>/dev/null
echo "First frame saved to /tmp/qa_frame0.jpg - please visually inspect"

echo ""
echo "=== Summary ==="
BITRATE=$(ffprobe -v quiet -show_entries format=bit_rate -of csv=p=0 "$FILE")
if [ "$BITRATE" -lt 1000000 ]; then
    echo "FAIL: Bitrate too low (${BITRATE}bps < 1Mbps)"
else
    echo "PASS: Bitrate OK (${BITRATE}bps)"
fi
```

### 使用方法

```bash
chmod +x qa_check.sh
./qa_check.sh final.mp4
```

---

## 完整制作流程 (按顺序)

```
1. 写剧本 → 确定每幕内容和配音文本
2. MiniMax TTS → 生成配音 → silencedetect确定时间点
3. 即梦/截图 → 准备关键帧素材 → 目视确认质量
4. Remotion编排 → 代码写好 → still --frame=0 验证首帧
5. Remotion渲染 → --video-bitrate "4M"
6. ffmpeg混音 → 配音+BGM → volumedetect验证
7. ffmpeg字幕 → 硬烧srt
8. 质检 → qa_check.sh → 首帧目视
9. 上传 → OSS → 验证URL → 多设备测试
```

**任何一步不通过都不能进入下一步。没有"先凑合后面再改"。**

---

## 常用时间计算

```python
# 中文配音时长估算
chars = 200  # 中文字数
speed = 0.95  # MiniMax speed参数
duration_seconds = chars / (4.0 * speed)  # ≈52.6秒

# 帧数计算
fps = 30
frames = int(duration_seconds * fps) + 30  # +30帧buffer = 1608帧

# 视频时长验证
expected_duration = frames / fps  # = 53.6秒
```

---

## 工具链版本 (截至2026-05)

| 工具 | 版本 | 用途 |
|------|------|------|
| ffmpeg | 6.x | 音视频处理 |
| Remotion | 4.x | 程序化视频 |
| dreamina CLI | latest | 即梦AI生图/生视频 |
| mmx CLI | latest | MiniMax TTS/Music |
| Node.js | 20+ | Remotion运行环境 |
| Python | 3.11+ | 脚本 |
