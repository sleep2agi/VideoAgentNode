# 即梦 Dreamina 5.0 完整使用指南

> 基于20+轮视频制作迭代的实战经验总结

---

## CLI 安装和登录

### 安装

```bash
pip install dreamina
```

### 登录(无头模式)

```bash
dreamina login --headless
```

会输出一个URL，浏览器打开登录后自动完成认证。Token保存在 `~/.dreamina/credentials.json`。

### 查看积分余额

```bash
dreamina user_credit
```

输出示例:
```
Total credits: 8800
Used credits: 3200
Remaining: 5600
```

积分消耗参考:
- text2image: ~20积分/张
- image2video (seedance2.0): ~110积分/次
- image2video (seedance2.0fast_vip): ~80积分/次
- image2video (3.5pro): ~60积分/次

---

## 文生图 text2image

### 基本命令

```bash
dreamina text2image \
  --prompt="赛博朋克科技感画面，深蓝黑色背景，中央一个发光的AI芯片，周围环绕数据流粒子，霓虹蓝色光效，cinematic dramatic lighting，4K high detail，不要文字不要人物" \
  --ratio=9:16 \
  --model_version=5.0 \
  --resolution_type=4k \
  --poll=60
```

### 参数详解

| 参数 | 值 | 说明 |
|------|------|------|
| --prompt | string | 图片描述，中文最优 |
| --ratio | 比例字符串 | 画面比例 |
| --model_version | 3.0/3.1/4.0/4.1/4.5/4.6/5.0 | 推荐5.0 |
| --resolution_type | 2k/4k | 推荐4k |
| --poll | 秒 | 轮询等待时间 |
| --seed | int | 固定种子复现结果 |
| --num | 1-4 | 生成数量 |

### 支持的画面比例

| 比例 | 用途 |
|------|------|
| 21:9 | 超宽电影感 |
| 16:9 | 横版视频封面/关键帧 |
| 3:2 | 横版海报 |
| 4:3 | 传统横版 |
| 1:1 | 正方形(小红书封面) |
| 3:4 | 竖版海报 |
| 2:3 | 竖版海报(更窄) |
| 9:16 | 竖版短视频关键帧 |

### Prompt写法铁律

1. **语言**: 中文为主，英文辅助(术语/修饰词用英文)
2. **长度**: 300-400字最优，超500字会被截断
3. **必加后缀**: 每个prompt末尾必须加 `不要文字 no text 不要人物 no characters`
4. **结构**: 主体描述 → 环境/背景 → 光影/氛围 → 画质修饰 → 负面排除

### model_version 选择策略

| 版本 | 特点 | 适用场景 |
|------|------|----------|
| 5.0 | 最新最强，细节丰富，光影真实 | 默认首选 |
| 4.6 | 稳定，色彩饱和 | 需要鲜艳色彩时 |
| 4.5 | 写实偏强 | 真实场景 |
| 4.0 | 概念艺术风格 | 抽象/概念图 |
| 3.1 | 老版，质量偏低 | 不推荐 |

---

## 图生视频 image2video

### 基本命令

```bash
dreamina image2video \
  --image ./keyframes/scene01.png \
  --prompt="镜头缓慢推进，粒子从中心向四周扩散，光效脉动闪烁" \
  --model_version seedance2.0fast_vip \
  --duration 10 \
  --poll 120
```

### 模型选择对比

| 模型 | 时长 | 分辨率 | 音频 | 排队 | 推荐度 |
|------|------|--------|------|------|--------|
| seedance2.0 | 5/10s | 720p | 无 | 可能排24h | 质量最好但不可控 |
| seedance2.0fast_vip | 5/10s | 720p | 有(自动) | 即时 | **日常首选** |
| 3.5pro | 5/12s | 720p | 有 | 即时 | 需要12s时用 |
| 3.0pro | 5/10s | 1080p | 无 | 即时 | 需要1080p时用 |

### 视频Prompt写法

视频prompt描述的是**运动**，不是画面内容(画面已经由图片定义):

```
# 好的视频prompt
镜头缓慢推进，画面中的光效粒子缓缓流动，背景星云微微旋转

# 差的视频prompt(把画面内容又写一遍)
赛博朋克城市，霓虹灯闪烁，高楼大厦...  ← 这些图片已经有了
```

常用运动描述:
- 镜头推进/拉远/平移/环绕
- 粒子流动/扩散/聚集
- 光效闪烁/脉动/渐变
- 元素旋转/漂浮/上升

---

## Prompt 模板库

### 科技风(机智流日常)

```
赛博朋克科技感画面，深蓝黑色背景，{主体内容}，发光网络节点连线，
霓虹蓝青色光效，cinematic dramatic lighting，4K high detail，
不要文字 no text 不要人物 no characters
```

### 自然/史诗风

```
dramatic cinematic landscape, golden horizon, {场景}, volumetric god rays,
dark sky with stars, epic scale, photorealistic, no text no characters
```

### 产品展示/网络可视化

```
futuristic dark network visualization, {产品/概念}, glowing nodes connected
by energy lines, holographic UI style, dark background, professional tech demo,
no text no characters
```

### 数据/图表风

```
minimalist data visualization on dark background, {数据主题},
clean geometric shapes, subtle gradient, professional infographic style,
soft ambient lighting, no text no characters
```

### 抽象概念风

```
abstract 3D visualization, {概念}, smooth flowing forms, iridescent surface,
dark void background, studio lighting, octane render quality,
no text no characters
```

---

## 批量生成

### Python脚本批量文生图

```python
import subprocess
import json

scenes = [
    {"name": "scene01", "prompt": "...", "ratio": "9:16"},
    {"name": "scene02", "prompt": "...", "ratio": "9:16"},
]

for scene in scenes:
    cmd = [
        "dreamina", "text2image",
        f"--prompt={scene['prompt']}",
        f"--ratio={scene['ratio']}",
        "--model_version=5.0",
        "--resolution_type=4k",
        "--poll=60",
        f"--output=./keyframes/{scene['name']}.png"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    print(f"{scene['name']}: {result.returncode}")
```

### 批量图生视频

```python
import glob
import subprocess

keyframes = sorted(glob.glob("./keyframes/scene*.png"))

for kf in keyframes:
    name = kf.split("/")[-1].replace(".png", "")
    cmd = [
        "dreamina", "image2video",
        f"--image={kf}",
        "--prompt=镜头缓慢推进，画面元素微微流动",
        "--model_version=seedance2.0fast_vip",
        "--duration=10",
        "--poll=120",
        f"--output=./clips/{name}.mp4"
    ]
    subprocess.run(cmd)
```

---

## 常见坑和解决方案

### 1. 文字 artifacts (出现率: 100%)

AI生成的图**一定**会出现乱码文字/字母，无一例外。

**解决**: 每个prompt末尾必须加 `不要文字 no text`。即使加了仍可能出现，
需要在关键帧阶段目视检查，严重的要重新生成。

### 2. 中英混排效果差

纯中文prompt效果最好。英文只用于:
- 画质修饰词: cinematic, 4K, high detail
- 风格术语: cyberpunk, synthwave
- 负面排除: no text, no characters

**不要做**: 中英文交替描述场景内容

### 3. seedance2.0排队问题

高峰期seedance2.0可能排队24小时以上。

**解决**: 日常生产用 `seedance2.0fast_vip`，质量差异<5%，
但有音频输出且即时返回。只有做showcase/demo时才用seedance2.0。

### 4. 积分消耗速度

一个5幕视频(10张关键帧+5个视频)大约消耗:
- 文生图: 10 x 20 = 200积分
- 图生视频: 5 x 110 = 550积分
- 总计: ~750积分/集

**策略**: 关键帧先用4k确认，视频生成一次到位不要反复重试

### 5. AI感太强 (Vincent原话: "即梦概念图AI感很强")

即梦生成的概念图有明显AI风格(过度光滑、色彩过饱和、细节不自然)。

**解决策略**:
- 产品视频: 优先用真实截图/录屏，不要用AI生成图
- 概念视频: 可以用，但要配合后期处理(降饱和、加噪点)
- 混合: 70%真实素材 + 30%AI生成作为转场/背景

### 6. 图片中出现人物

即使prompt里没提人物，AI有时会自作主张加人。

**解决**: 必须加 `不要人物 no characters no people no humans`

### 7. prompt被截断

超过500字的prompt会被静默截断，后半部分描述丢失。

**解决**: 控制在300-400字，把最重要的描述放前面，
修饰词和排除项放后面(即使被截断影响最小)。

### 8. 生成结果不稳定

同样的prompt每次生成结果差异很大。

**解决**:
- 满意的结果记录seed
- 用 `--seed` 参数复现
- 批量生成4张(`--num=4`)从中挑选

### 9. 视频抖动/闪烁

image2video有时会产生画面抖动或颜色闪烁。

**解决**:
- 视频prompt加 `smooth camera movement, stable`
- 减少运动描述的幅度
- 选择seedance2.0fast_vip(比3.5pro稳定)

### 10. 深色图片生成视频变暗

深色关键帧生成视频后会进一步变暗，细节丢失。

**解决**: 关键帧稍微提亮(不要纯黑背景，用深灰#1a1a1a)

---

## 与其他工具的配合

### 即梦 → Remotion Pipeline

```
1. dreamina text2image → keyframes/*.png (关键帧)
2. 人工确认关键帧质量
3. dreamina image2video → clips/*.mp4 (视频片段)
4. Remotion编排 + 字幕 + 动效
5. ffmpeg混音(配音+BGM)
6. 质检(volumedetect + blackdetect + 目视)
```

### 即梦 → 纯剪辑 Pipeline

```
1. dreamina text2image → backgrounds (背景图)
2. dreamina image2video → b-roll clips (转场素材)
3. 配合真实截图/录屏作为主体内容
4. ffmpeg拼接 + 字幕 + 混音
```

---

## 环境变量

```bash
# ~/.bashrc or ~/.zshrc
export DREAMINA_TOKEN="your_token_here"  # 可选，覆盖credentials文件
export DREAMINA_OUTPUT_DIR="./outputs"    # 默认输出目录
```

---

## 故障排查

| 症状 | 原因 | 解决 |
|------|------|------|
| `401 Unauthorized` | Token过期 | `dreamina login --headless` 重新登录 |
| `insufficient credits` | 积分不够 | 充值或等月初重置 |
| poll超时 | 生成太慢/排队 | 增加--poll值，或换fast模型 |
| 输出全黑 | prompt触发安全审核 | 修改prompt避免敏感词 |
| 文件0字节 | 网络中断 | 重试，检查网络 |
