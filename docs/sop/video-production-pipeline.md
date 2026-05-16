# 视频制作全流程 SOP

## 流程概览

```
选题 → 脚本 → AI素材生成 → 语音合成 → 视频合成 → 渲染 → QA → 发布
```

## 1. 选题

- 在 GitHub Issue 中创建选题，标注视频类型（宣传/资讯 or 知识点）
- 明确目标受众、时长、发布平台

## 2. 脚本撰写

- 确定视频结构：开场 → 正文（分段） → 结尾 CTA
- 每段标注：画面描述 + 旁白文字 + 时长预估
- 脚本模板见 templates/

## 3. AI 素材生成（Dreamina）

- 根据脚本中的画面描述生成图片/视频片段
- 使用 text-to-image 或 image-to-video
- 小批量生成，验证质量后再批量扩展
- 注意 credits 余量：生成前 `dreamina user_credit` 检查

## 4. 语音合成（MiniMax TTS）

- 将旁白文字输入 MiniMax TTS 生成音频
- 选择合适的音色和语速
- 输出格式：WAV 或 MP3

## 5. 视频合成（Remotion）

- 使用 Remotion 编排素材：图片/视频 + 音频 + 字幕
- 配置分辨率、帧率、时长
- CJK 字体需显式指定路径，避免 □□□ 乱码

## 6. 渲染（ffmpeg）

- Remotion 输出帧序列 → ffmpeg 编码为最终视频
- 常用参数：`-c:v libx264 -preset medium -crf 23`
- 字幕渲染：`drawtext` 滤镜，注意 fontfile 路径

## 7. 质量检查（QA）

- `ffprobe` 验证输出文件：时长、分辨率、帧率、音轨
- 目视检查：画面完整性、字幕对齐、音画同步
- AI 自评不可靠，必须人工/工具验证

## 8. 发布

- 上传至目标平台（B站、YouTube 等）
- 填写标题、描述、标签
- 在 Issue 中标记完成，附发布链接
