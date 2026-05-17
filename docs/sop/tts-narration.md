# 语音合成 SOP

## 工具

- [MiniMax TTS](https://www.minimax.chat/) — speech-2.8-hd 模型
- API: `https://api.minimaxi.com/v1/t2a_v2`

## 流程

### 1. 文本准备

- 从脚本中提取旁白文字
- 添加标点控制停顿节奏
- 长句拆短，每句 < 30 字为宜
- **每个场景单独生成一段 TTS**（便于后续对齐）

### 2. 音色选择

| voice_id | 风格 | 适用场景 |
|----------|------|---------|
| vincent_wenxing_v1 | 闻星克隆声（偏年轻男声） | 机智流日报/产品视频 |
| presenter_male | 播音员男声（中性正式） | 品牌中性版本 |

### 3. 合成代码

```python
import requests, os

payload = {
    'model': 'speech-2.8-hd',
    'text': '旁白文本',
    'voice_setting': {
        'voice_id': 'vincent_wenxing_v1',
        'speed': 1.0,  # 0.95-1.05 范围
    },
    'audio_setting': {
        'sample_rate': 32000,
        'format': 'wav',  # ⚠️ 必须用 wav，mp3 格式有解码问题
    }
}

r = requests.post('https://api.minimaxi.com/v1/t2a_v2',
    headers={'Authorization': f'Bearer {api_key}', 'Content-Type': 'application/json'},
    json=payload, timeout=60)

resp = r.json()
if resp.get('data') and resp['data'].get('audio'):
    # ⚠️ 关键：audio 是 hex 编码，不是 base64！
    audio = bytes.fromhex(resp['data']['audio'])
    with open('output.wav', 'wb') as f:
        f.write(audio)
    # 转 mp3
    os.system('ffmpeg -y -i output.wav -c:a libmp3lame -ar 44100 -q:a 2 output.mp3')
```

### 4. 拼接（多段 TTS + gap）

```bash
# 生成静音间隔
ffmpeg -y -f lavfi -i anullsrc=r=44100:cl=stereo -t 0.8 -q:a 2 gap.mp3

# 拼接
printf 'file s00.mp3\nfile gap.mp3\nfile s01.mp3\nfile gap.mp3\nfile s02.mp3\n' > concat.txt
ffmpeg -y -f concat -safe 0 -i concat.txt -c:a libmp3lame -ar 44100 -q:a 2 narration.mp3
```

### 5. Silencedetect（找段落边界）

```bash
ffmpeg -i narration.mp3 -af silencedetect=noise=-30dB:d=0.7 -f null -
# >1s 的静默 = 段落边界
# 用于精确设置 Remotion Sequence 的 from/durationInFrames
```

### 6. 注意事项

- ⚠️ audio 字段是 **hex 编码**（`bytes.fromhex()`），不是 base64
- ⚠️ 输出格式必须用 **wav**（mp3 有解码问题）
- MiniMax API key 分两种：sk-cp（Token Plan，推荐）和 sk-api（标准）
- 语速 speed: 产品视频用 0.98-1.0，日报用 1.0-1.05
- TTS 输出偏轻，合并时需要 `volume=2.0` 增益
- Fallback：`python3 -m edge_tts --voice zh-CN-YunyangNeural --text "..." --write-media output.mp3`
