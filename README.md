# video2notes

自动将视频转为结构化中文笔记：下载视频 → Whisper 语音转文字 → ffmpeg 智能截图 → 生成 Markdown 笔记。

## 支持平台

| 平台 | 状态 | 方式 |
|------|------|------|
| B站 (bilibili) | 需要 Chrome 已登录 | yt-dlp + cookies |
| 抖音 (douyin) | 无需登录 | 公开分享页直接下载 |
| YouTube | 无需登录 | yt-dlp 内置支持 |

## 系统要求

- **macOS only**（Windows/Linux 需自行替换 fallback 机制）
- Python >= 3.9

## 安装

```bash
# 安装系统依赖
brew install ffmpeg

# 安装 Python 依赖
pip3 install yt-dlp openai-whisper Pillow numpy
```

首次运行 Whisper 时会自动下载 `small` 模型（约 466MB）。

## 配置

将 `SKILL.md` 放入 Claude Code 的 skills 目录，然后修改其中的 `## Configuration` 区块：

```yaml
notes_root: ~/你的笔记文件夹/
category_rules:
  - keywords: ["量化"]
    subdir: 量化/
  - keywords: ["人工智能", "AI"]
    subdir: 人工智能个人笔记/
  # ... 按需添加
```

## 使用方式

直接粘贴视频链接即可：

```
帮我记一下这个视频的笔记 https://www.bilibili.com/video/BV1xx/
```

```
把这个抖音视频转成笔记 https://www.douyin.com/video/xxxxx
```

Skill 会自动：
1. 识别平台，获取视频信息
2. 下载音视频
3. Whisper 语音转文字（带时间戳）
4. 根据转文字幕关键词定位关键画面
5. 提帧截图并验证质量
6. 生成结构化中文笔记，截图嵌入对应位置

## 输出结构

```
你的笔记文件夹/分类子目录/
├── 视频标题_笔记.md
└── screenshots/
    ├── 视频标题_intro.png
    ├── 视频标题_overview.png
    └── ...
```

## 隐私与安全

- **全部本地处理**：音视频不上传任何服务器
- **截图来自 ffmpeg 提帧**：不截取桌面，不会泄漏隐私
- **临时文件自动清理**：处理完后 `/tmp` 下的文件会删除
- 详细说明见 SKILL.md 末尾的 Security & Privacy Notes

## 法律声明

本工具仅供**个人学习、研究、做笔记**使用。下载视频可能违反各平台服务条款，使用者需自行承担合规责任。禁止用于商业用途或公开分发下载内容。

详见 SKILL.md 中的 Legal Disclaimer 章节。

## License

MIT
