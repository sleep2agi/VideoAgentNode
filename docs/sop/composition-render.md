# 视频合成与渲染 SOP

## 工具

- [Remotion](https://www.remotion.dev/) — React/TypeScript 视频合成
- [ffmpeg](https://ffmpeg.org/) — 编码、拼接、字幕渲染

## Remotion 合成

### 1. 项目结构

```
src/
  compositions/
    SceneOne.tsx
    SceneTwo.tsx
  Root.tsx
  index.ts
```

### 2. 配置

- 分辨率：1920x1080（16:9）
- 帧率：30fps
- 时长：根据音频自动计算

### 3. 常见组件

- `<Img>` — 静态画面展示
- `<Video>` — AI 生成视频片段
- `<Audio>` — 旁白音频
- `<Sequence>` — 时间轴排列

### 4. CJK 字体

**关键：必须显式指定字体路径，否则出现 □□□ 乱码**

```tsx
fontFamily: "Noto Sans SC"
// 确保字体文件在 public/ 目录下
```

## ffmpeg 渲染

### 1. 基础编码

```bash
ffmpeg -i input.mov -c:v libx264 -preset medium -crf 23 -c:a aac output.mp4
```

### 2. 字幕叠加

```bash
ffmpeg -i input.mp4 -vf "drawtext=fontfile=/path/to/font.ttf:text='字幕':fontsize=48:fontcolor=white:x=(w-text_w)/2:y=h-100" output.mp4
```

### 3. 视频拼接

```bash
# concat.txt:
# file 'scene01.mp4'
# file 'scene02.mp4'
ffmpeg -f concat -safe 0 -i concat.txt -c copy output.mp4
```

### 4. 输出规范

- 格式：MP4 (H.264 + AAC)
- 比特率：视频 5-8 Mbps，音频 192 Kbps
- 最终文件命名：`[类型]_[标题]_[日期].mp4`
