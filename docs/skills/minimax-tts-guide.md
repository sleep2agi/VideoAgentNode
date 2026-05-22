# MiniMax TTS 语音合成完整指南

> 基于机智流视频系列配音的实战经验。关键坑: hex编码不是base64, 必须用wav格式。

---

## API 配置

### Endpoint

```
POST https://api.minimaxi.com/v1/t2a_v2
```

### 认证

```bash
# Token Plan key (推荐，不消耗余额)
Authorization: Bearer sk-cp-xxxxxxxxxxxxxxxx

# 普通key (消耗余额)
Authorization: Bearer sk-xxxxxxxxxxxxxxxx
```

**推荐用 Token Plan key (sk-cp- 开头)**，有独立配额，不消耗账户余额。

### Model

```
speech-2.8-hd
```

---

## 关键注意事项 (血泪教训)

### 1. audio字段是 HEX 编码，不是 base64!

```python
# 正确 ✓
audio_bytes = bytes.fromhex(response_json["data"]["audio"])

# 错误 ✗ (会得到损坏的音频文件)
audio_bytes = base64.b64decode(response_json["data"]["audio"])
```

这是MiniMax独有的设计，和其他TTS服务不同。文档里也没写清楚。

### 2. 输出格式必须用 wav

```python
payload = {
    "model": "speech-2.8-hd",
    "text": "...",
    "voice_setting": {...},
    "audio_setting": {
        "format": "wav",      # 必须wav! mp3有解码问题
        "sample_rate": 32000
    }
}
```

mp3格式在某些播放器/ffmpeg处理链路中会出现时长偏差或解码噪音。
wav虽然大但100%兼容，后续用ffmpeg转mp3即可。

### 3. Token Plan key 不消耗余额

sk-cp- 开头的key有独立额度(通常100万字符/月)，用完前不扣余额。
优先使用Token Plan key。

---

## 完整 Python 代码示例

```python
import requests
import subprocess
import os

MINIMAX_API_KEY = os.environ.get("MINIMAX_API_KEY", "sk-cp-xxxxxxxxxx")
MINIMAX_ENDPOINT = "https://api.minimaxi.com/v1/t2a_v2"


def synthesize_speech(
    text: str,
    voice_id: str = "vincent_wenxing_v1",
    speed: float = 1.0,
    output_path: str = "narration.mp3"
) -> str:
    """合成语音并输出mp3文件"""

    payload = {
        "model": "speech-2.8-hd",
        "text": text,
        "voice_setting": {
            "voice_id": voice_id,
            "speed": speed,
            "vol": 1.0,
            "pitch": 0
        },
        "audio_setting": {
            "format": "wav",
            "sample_rate": 32000
        }
    }

    headers = {
        "Authorization": f"Bearer {MINIMAX_API_KEY}",
        "Content-Type": "application/json"
    }

    response = requests.post(MINIMAX_ENDPOINT, json=payload, headers=headers)
    response.raise_for_status()

    result = response.json()

    if result.get("base_resp", {}).get("status_code") != 0:
        error_msg = result.get("base_resp", {}).get("status_msg", "Unknown error")
        raise RuntimeError(f"MiniMax TTS error: {error_msg}")

    # 关键: HEX解码，不是base64!
    audio_hex = result["data"]["audio"]
    audio_bytes = bytes.fromhex(audio_hex)

    # 先写wav
    wav_path = output_path.replace(".mp3", ".wav")
    with open(wav_path, "wb") as f:
        f.write(audio_bytes)

    # wav转mp3 (高质量)
    subprocess.run([
        "ffmpeg", "-y",
        "-i", wav_path,
        "-codec:a", "libmp3lame",
        "-b:a", "192k",
        "-ac", "2",
        output_path
    ], check=True, capture_output=True)

    # 清理wav
    os.remove(wav_path)

    return output_path
```

---

## Voice ID 表

| voice_id | 风格 | 适用场景 | 备注 |
|----------|------|----------|------|
| vincent_wenxing_v1 | 闻星克隆(年轻男声) | 机智流品牌内容 | 克隆的Vincent声音 |
| presenter_male | 播音员(中性正式) | 品牌中性版/对外 | 专业感强 |
| mother_voice_v1 | 母亲克隆 | 特殊用途 | 温暖女声 |
| male-qn-qingse | 青涩男声 | 轻松内容 | MiniMax预置 |
| male-qn-jingying | 精英男声 | 商务内容 | MiniMax预置 |
| female-shaonv | 少女声 | 活泼内容 | MiniMax预置 |
| female-yujie | 御姐声 | 知性内容 | MiniMax预置 |

### 机智流标准配置

```python
# 日常视频用闻星
voice_id = "vincent_wenxing_v1"
speed = 0.95  # 稍慢，稳重感

# 对外/正式用播音员
voice_id = "presenter_male"
speed = 1.0
```

---

## 语速调优

| speed值 | 效果 | 适用 |
|---------|------|------|
| 0.85 | 很慢，像在念稿 | 不推荐 |
| 0.90 | 偏慢，沉稳 | 解说长段落 |
| 0.95 | 略慢，稳重 | **日常首选** |
| 1.00 | 标准 | 通用 |
| 1.05 | 略快 | 快节奏剪辑 |
| 1.10 | 快，像赶时间 | 不推荐 |

**经验**: 0.95是最佳平衡点。太快听不清，太慢拖节奏。

---

## 多段配音拼接

### 场景: 每一幕单独合成，最后拼接

```python
import subprocess

scenes = [
    {"text": "第一幕配音文本...", "file": "s01.mp3"},
    {"text": "第二幕配音文本...", "file": "s02.mp3"},
    {"text": "第三幕配音文本...", "file": "s03.mp3"},
]

# 1. 逐段合成
for scene in scenes:
    synthesize_speech(scene["text"], output_path=f"parts/{scene['file']}")

# 2. 生成0.8s静音间隔
subprocess.run([
    "ffmpeg", "-y",
    "-f", "lavfi", "-i", "anullsrc=r=44100:cl=stereo",
    "-t", "0.8",
    "-q:a", "2",
    "parts/gap.mp3"
], check=True)

# 3. 创建concat列表
with open("parts/concat.txt", "w") as f:
    for i, scene in enumerate(scenes):
        f.write(f"file '{scene['file']}'\n")
        if i < len(scenes) - 1:
            f.write("file 'gap.mp3'\n")

# 4. 拼接
subprocess.run([
    "ffmpeg", "-y",
    "-f", "concat", "-safe", "0",
    "-i", "parts/concat.txt",
    "-c:a", "libmp3lame", "-b:a", "192k",
    "narration_full.mp3"
], check=True)
```

### 间隔时长选择

| 场景 | gap时长 | 理由 |
|------|---------|------|
| 同一幕内句子间 | 0.3-0.5s | 自然语流 |
| 不同幕之间 | 0.8-1.0s | 明显分段感 |
| 重要观点前 | 1.2-1.5s | 制造悬念 |

---

## silencedetect 时间轴对齐

用于配音和视频画面的时间对齐:

```bash
ffmpeg -i narration_full.mp3 -af silencedetect=noise=-30dB:d=0.7 -f null - 2>&1 | grep silence
```

输出示例:
```
[silencedetect] silence_start: 15.234
[silencedetect] silence_end: 16.089 | silence_duration: 0.855
[silencedetect] silence_start: 32.567
[silencedetect] silence_end: 33.401 | silence_duration: 0.834
```

### 解析脚本

```python
import re
import subprocess

def detect_silence_boundaries(audio_path: str, noise=-30, duration=0.7):
    """检测配音中的静音段，用于对齐视频切换点"""
    cmd = [
        "ffmpeg", "-i", audio_path,
        "-af", f"silencedetect=noise={noise}dB:d={duration}",
        "-f", "null", "-"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    stderr = result.stderr

    boundaries = []
    for match in re.finditer(r"silence_end: ([\d.]+)", stderr):
        boundaries.append(float(match.group(1)))

    return boundaries  # 这些时间点就是场景切换点
```

---

## 中英混合处理

MiniMax对中英混合文本的处理不稳定(英文单词可能被按字母读)。

### 策略: 数字和缩写写中文

```python
# 差 (可能读成 "G P T 杠 5")
text = "GPT-5比GPT-4强了10倍"

# 好
text = "GPT五比GPT四强了十倍"

# 或者用全称
text = "GPT五代比上一代强了十倍"
```

### 策略: 长英文段落单独合成

如果有完整英文句子，用英文voice单独合成再拼接。

---

## MiniMax CLI (mmx)

```bash
# 安装
pip install minimax-cli

# 配置
mmx config set api_key sk-cp-xxxxxxxx

# 单句合成
mmx speech tts --text "这是一段测试文本" --voice presenter_male --output test.mp3

# 指定语速
mmx speech tts --text "..." --voice vincent_wenxing_v1 --speed 0.95 --output out.mp3

# 查看可用voices
mmx speech voices
```

---

## 错误处理

### 常见错误码

| status_code | 含义 | 解决 |
|-------------|------|------|
| 1000 | 未知错误 | 重试 |
| 1001 | 超时 | 文本太长，拆分 |
| 1002 | 触发限流 | 等待后重试 |
| 1004 | 鉴权失败 | 检查API key |
| 1008 | 余额不足 | 充值或换Token Plan key |
| 1013 | 文本超限 | 单次最大5000字，拆分 |

### 重试逻辑

```python
import time

def synthesize_with_retry(text, max_retries=3, **kwargs):
    for attempt in range(max_retries):
        try:
            return synthesize_speech(text, **kwargs)
        except RuntimeError as e:
            if "1002" in str(e):  # 限流
                wait = 2 ** attempt * 5
                print(f"Rate limited, waiting {wait}s...")
                time.sleep(wait)
            elif "1013" in str(e):  # 超限
                raise ValueError(f"Text too long ({len(text)} chars), max 5000")
            else:
                raise
    raise RuntimeError(f"Failed after {max_retries} retries")
```

---

## 文本预处理

### 清洁脚本

```python
import re

def clean_text_for_tts(text: str) -> str:
    """清洁文本使TTS效果更好"""
    # 去除markdown格式
    text = re.sub(r'[#*_`~]', '', text)
    # 去除括号中的英文注释
    text = re.sub(r'\([a-zA-Z\s]+\)', '', text)
    # 数字转中文(简单版)
    text = text.replace("1.", "第一，")
    text = text.replace("2.", "第二，")
    text = text.replace("3.", "第三，")
    # 确保句子有停顿
    text = re.sub(r'([。！？])', r'\1 ', text)
    # 去除多余空格
    text = re.sub(r'\s+', ' ', text).strip()
    return text
```

### 时长估算

```python
def estimate_duration(text: str, speed: float = 0.95) -> float:
    """估算配音时长(秒)。中文约4字/秒"""
    char_count = len(re.sub(r'[^\u4e00-\u9fff]', '', text))
    chars_per_second = 4.0 * speed
    return char_count / chars_per_second

# 示例: 200字中文 ÷ (4 × 0.95) ≈ 52.6秒
```

---

## 与视频制作的配合

### 配音 → 视频时长计算

```python
import subprocess
import json

def get_audio_duration(path):
    result = subprocess.run([
        "ffprobe", "-v", "quiet",
        "-print_format", "json",
        "-show_format", path
    ], capture_output=True, text=True)
    info = json.loads(result.stdout)
    return float(info["format"]["duration"])

duration = get_audio_duration("narration.mp3")
fps = 30
frames = int(duration * fps) + 30  # +30帧buffer
print(f"Remotion durationInFrames = {frames}")
```

### 配音+BGM混音(标准参数)

```bash
ffmpeg -y -i video.mp4 -i narration.mp3 -i bgm.mp3 \
  -filter_complex "[1:a]volume=2.0[voice];[2:a]volume=0.06[bgm];[voice][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.7[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k -ac 2 final.mp4
```

音量比: voice=2.0, bgm=0.06 (配音是BGM的33倍音量)
