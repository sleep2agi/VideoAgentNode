# 质量检查 SOP

## 原则

> AI 自评分不可靠。必须用工具 + 人工验证。

## 检查流程

### 1. 技术验证（ffprobe）

```bash
ffprobe -v error -show_entries format=duration,size,bit_rate -show_entries stream=codec_name,width,height,r_frame_rate,channels -of json output.mp4
```

检查项：
- [ ] 时长符合预期（±2s）
- [ ] 分辨率正确（1920x1080）
- [ ] 帧率正确（30fps）
- [ ] 音轨存在且编码正确
- [ ] 文件大小合理（1min ≈ 40-60MB）

### 2. 目视检查

- [ ] 画面完整，无黑帧/花屏
- [ ] 字幕清晰可读，无乱码（□□□）
- [ ] 音画同步，旁白与画面节奏匹配
- [ ] 转场自然，无跳帧
- [ ] 开场/结尾完整

### 3. 内容检查

- [ ] 信息准确，无事实错误
- [ ] 品牌元素正确（logo、配色）
- [ ] 无版权问题的素材

### 4. 平台适配

- [ ] B站：≤ 8GB，推荐 H.264
- [ ] YouTube：≤ 128GB，推荐 H.264/H.265
- [ ] 竖屏平台：确认 1080x1920 比例

## 不通过处理

1. 记录具体问题
2. 定位出错环节（生成/合成/渲染）
3. 回到对应 SOP 修复
4. 重新渲染后再次 QA
