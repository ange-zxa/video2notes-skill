---
name: video2notes
description: Download a video from B站/抖音/YouTube, transcribe its audio with Whisper, capture screenshots at key visual moments, and write a structured Chinese note with embedded illustrations. Use this skill whenever the user wants to take notes from a video, "看视频做笔记", "帮我记笔记", or provides a video URL.
platforms: [macOS]
python: ">=3.9"
dependencies: [ffmpeg, curl, yt-dlp, whisper, Pillow, numpy]
---

# Setup (初次使用必须配置)

## 1. 安装依赖

```bash
# macOS
brew install ffmpeg
pip3 install yt-dlp openai-whisper Pillow numpy
```

首次运行时 Whisper 会自动下载 `small` 模型（~466MB），请耐心等待。

## 2. 设置笔记保存目录

在下方 `## Configuration` 区域设置你的笔记根目录和分类规则。**不设置则无法保存笔记**。

---

# Configuration

```yaml
# 笔记保存根目录（必填）
notes_root: ~/笔记文件夹/

# 标题关键词 → 子目录（按顺序匹配，命中第一个即停止）
category_rules:
  - keywords: ["量化"]
    subdir: 量化/
  - keywords: ["人工智能", "AI"]
    subdir: 人工智能个人笔记/
  - keywords: ["论文", "SCI", "写作"]
    subdir: 论文写作/
  # 未命中任何规则 → 运行时询问用户

# 笔记文件命名规则
note_suffix: _笔记.md
```

---

# Purpose

Given a video URL from any supported platform, automatically:
1. Detect the platform and extract video metadata
2. Download audio + video using yt-dlp
3. Transcribe the audio to Chinese text using Whisper (with timestamps)
4. Identify key visual moments from the VTT transcript
5. Extract frames at key moments from the downloaded video using ffmpeg
6. Write a comprehensive Chinese note with screenshots embedded at the right positions
7. Save notes + screenshots to the appropriate directory

> **平台要求**：macOS only。核心工具（ffmpeg、Whisper、yt-dlp）跨平台，但 fallback 机制（AppleScript、screencapture）依赖 macOS。Windows/Linux 用户需自行替换 fallback。

# Pre-flight Checks

Before processing any video, verify required tools are available:

```bash
which ffmpeg ffprobe python3 curl
python3 -c "from PIL import Image; import numpy; import whisper; print('OK')"
```

If any check fails, return to Setup section.

**Important**: Only one video should be processed at a time. The skill uses fixed paths under `/tmp` — concurrent runs will overwrite each other's files and produce corrupted output.

# Supported Platforms

## Step 0: Detect platform from URL

| URL Pattern | Platform | Metadata Source | Download Method |
|-------------|----------|-----------------|-----------------|
| `bilibili.com/video/BV*` | B站 | B站 API | yt-dlp + Chrome cookies |
| `douyin.com/video/*` / `v.douyin.com/*` / `iesdouyin.com/*` | 抖音 | iesdouyin.com share page ROUTER_DATA | curl direct (no auth) |
| `youtube.com/watch?v=*` / `youtu.be/*` | YouTube | yt-dlp | yt-dlp (built-in) |

**抖音短链解析**: `v.douyin.com/XXXXX` 是短链，需先跟随重定向拿到真实 URL：
```bash
curl -sI -o /dev/null -w '%{redirect_url}' "SHORT_LINK"
```
从重定向 URL 中提取 video_id（`/video/数字ID`），再拼接分享页地址。

## Step 1: Get video info (per platform)

### B站
```bash
curl -s -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" -H "Referer: https://www.bilibili.com/" "https://api.bilibili.com/x/web-interface/view?bvid=BV_XXXXX"
```
Extract: `title`, `desc`, `duration`, `cid`, `aid`, publish date, tags.

### 抖音

**IMPORTANT**: Convert any douyin URL to the share page format:
`https://www.iesdouyin.com/share/video/{video_id}/`

This page embeds all video metadata in `window._ROUTER_DATA` — no cookies, no Chrome JS needed.

```bash
# 1. Extract video_id from URL
#    douyin.com/video/7623623257937235234 → video_id = 7623623257937235234
#    v.douyin.com/XXXXX → follow redirect to get real URL

# 2. Fetch share page with mobile UA, extract ROUTER_DATA JSON
python3 -c "
import re, json, subprocess
url = 'https://www.iesdouyin.com/share/video/{video_id}/'
result = subprocess.run(['curl', '-sL',
    '-H', 'User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X) AppleWebKit/605.1.15',
    '-H', 'Referer: https://www.douyin.com/',
    url], capture_output=True, text=True, timeout=15)
html = result.stdout
match = re.search(r'window\._ROUTER_DATA\s*=\s*({.*?})\s*</script>', html, re.DOTALL)
data = json.loads(match.group(1))
item = data['loaderData']['video_(id)/page']['videoInfoRes']['item_list'][0]
print('Title:', item['desc'])
print('Author:', item['author']['nickname'])
print('Duration:', item['video']['duration'], 'ms')
for u in item['video']['play_addr']['url_list']:
    print('Video URL:', u)
"
```

### YouTube
```bash
python3 -m yt_dlp --print title "VIDEO_URL"
```

## Step 2: Download audio + video (per platform)

### B站

**Pre-check before download**: yt-dlp must be able to read Chrome cookies. If any of these is true, the download will fail:
- Chrome is not installed or not running
- User is not logged into B站 in Chrome
- B站 login session has expired

To verify: open `https://www.bilibili.com/` in Chrome and confirm you're logged in.

```bash
python3 -m yt_dlp --cookies-from-browser chrome -f "bestaudio" -o "/tmp/video_audio.%(ext)s" "VIDEO_URL"
python3 -m yt_dlp --cookies-from-browser chrome -f "bestvideo[height<=720]" -o "/tmp/video_video.%(ext)s" "VIDEO_URL"
```
Without cookies, only the first 5-minute segment is available.

### 抖音

**抖音视频是合并文件（音视频一体），一次下载即可。** Use the video URL from Step 1:
```bash
curl -L -o /tmp/video_video.mp4 \
  -H "User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X) AppleWebKit/605.1.15" \
  -H "Referer: https://www.douyin.com/" \
  "VIDEO_URL_FROM_STEP1"
```
Audio extracted from this file in Step 3 (no separate audio download needed).

### YouTube
```bash
python3 -m yt_dlp -f "bestaudio" -o "/tmp/video_audio.%(ext)s" "VIDEO_URL"
python3 -m yt_dlp -f "bestvideo[height<=720]" -o "/tmp/video_video.%(ext)s" "VIDEO_URL"
```

Always verify duration with ffprobe after download.

## Step 3: Extract audio to WAV

Unified command — finds the downloaded audio/video file regardless of extension:

```bash
cd /tmp && ffmpeg -y -i $(ls video_audio.* video_video.* 2>/dev/null | head -1) -vn -acodec pcm_s16le -ar 16000 -ac 1 video_audio.wav
```

This handles all three platforms:
- B站: `video_audio.m4a` (or `.m4s`)
- 抖音: `video_video.mp4` (combined file, extract audio track)
- YouTube: `video_audio.webm` (or other formats)

Verify:
```bash
ffprobe -v quiet -show_entries format=duration -of csv=p=0 video_audio.wav
```

## Step 4: Transcribe with Whisper

Use VTT format (includes timestamps) for screenshot timing:

```bash
cd /tmp && rm -rf /tmp/whisper_out && python3 -m whisper video_audio.wav --model small --language Chinese --output_dir /tmp/whisper_out --output_format vtt txt
```

**Model choice**: `small` (~466MB) recommended for acceptable Chinese accuracy. `tiny` (~72MB) is faster but produces heavy garbling (错字). Use `tiny` only for quick previews; re-run with `small` for the final note.

**Accuracy disclaimer**: Even `small` model produces noticeable errors in Chinese transcription, especially for technical terms, proper names, and code snippets. Always cross-check important data, numbers, and technical claims against the video itself. The transcript is a note-taking aid, not an authoritative reference.

Read both outputs:
- `.txt` for full transcript text
- `.vtt` for timestamped segments (used to determine when to screenshot)

## Step 5: Identify key visual moments from VTT timestamps

**CRITICAL**: Use the VTT file (not line-number estimation) to get precise timestamps. Grep the VTT for visual cue keywords:

**Visual Cue Keywords (Chinese):**
- "看图" "这张图" "这张表" "这个表" "这个图" "如图所示"
- "代码" "带马" "这段代码" "看代码" "运行"
- "对比" "对别" "对比表" "对比图"
- "曲线" "图表" "图线" "分布" "中形区线"
- "三个部分" "左边一块" "中间一块" "右边一块"
- "下张子图" "放一起" "看一下" "看一组对比"
- "清单" "陷阱" "餐馆" "性格" "资金"

```bash
grep -n -B1 -E '看图|这张图|这张表|这个表|这个图|如图所示|代码|这段代码|看代码|运行|对比|对别|对比表|对比图|曲线|图表|图线|分布|三个部分|左边一块|中间一块|右边一块|下张子图|放一起|看一下|看一组对比|清单|陷阱|餐馆|性格|资金' /tmp/whisper_out/*.vtt
```

Each match gives you a precise `MM:SS.mmm` timestamp. Convert to seconds for ffmpeg.

**Prioritize moments that correspond to:**
1. Code demonstrations (highest priority — unique to the video)
2. Charts/graphs described in detail (e.g., "整张图分三个部分")
3. Comparison tables shown on screen (e.g., "看一组对比")
4. Self-assessment checklists / action items (e.g., "自检清单")
5. Key concept overviews (e.g., factor tables, latency comparisons)

**Target: 5-8 screenshots** per video. Extract a frame at each VTT timestamp, then use Python PIL to check frame quality (std > 20 = has real content). Pick the best ones.

## Step 6: Extract screenshots from video

**Use ffmpeg to extract frames directly from the downloaded video file — NOT screencapture from browser.** This produces clean screenshots without window chrome or desktop background.

### 6a: Verify video file is valid

```bash
ffprobe -v quiet -show_entries format=duration -of csv=p=0 /tmp/video_video.mp4
```

### 6b: For each key timestamp, extract a frame

Use ffmpeg with `-vframes 1` and `-update true` to get a single frame:

```bash
ffmpeg -y -ss TIMESTAMP_IN_SECONDS -i /tmp/video_video.mp4 -vframes 1 -update true /tmp/video_screenshot_N.png
```

The `-ss` before `-i` does fast input seeking.

### 6c: Verify frame quality

Use Python PIL to verify the frame has meaningful content (not just a black screen):

```bash
python3 -c "
from PIL import Image
import numpy as np
img = Image.open('/tmp/video_screenshot_N.png')
arr = np.array(img)
print(f'mean={arr.mean():.1f} std={arr.std():.1f}')
# If std > 20, the frame has real content
"
```

### 6d: Screenshot naming convention

Name screenshots with the note prefix + descriptive English suffix:
- `{笔记前缀}_intro.png` — 视频开篇 / 封面
- `{笔记前缀}_overview.png` — 内容总览 / 全景图
- `{笔记前缀}_comparison.png` — 对比表 / 方案对比
- `{笔记前缀}_chart.png` — 图表 / 数据分布
- `{笔记前缀}_code.png` — 代码演示
- `{笔记前缀}_feature_demo.png` — 功能 / 特色展示
- `{笔记前缀}_checklist.png` — 清单 / 自检表
- `{笔记前缀}_conclusion.png` — 总结 / 核心结论

## Step 7: Determine output directory

Match the video title against the rules in `## Configuration`. Check in order, stop at first match. If no rule matches, ask the user which directory to use.

## Step 8: Write notes and copy screenshots

### Copy screenshots
```bash
mkdir -p "<OUTPUT_DIR>/screenshots"
cp /tmp/video_screenshot_*.png "<OUTPUT_DIR>/screenshots/"
```

### Write the markdown note
```
<OUTPUT_DIR>/
├── {笔记前缀}_笔记.md
└── screenshots/
    ├── {笔记前缀}_code.png
    └── ...
```

**Note structure**:
- **Header**: video title, URL, duration, publish date, UP主 name
- **Body**: organized by the video's natural sections, images at their EXACT positions
- **Image syntax**: `![描述](screenshots/filename.png)` + one-line caption below
- **Style**: Chinese, concise, use tables for comparisons, use blockquotes for key quotes

**Principles**:
- Capture the ACTUAL content from the transcript, not generic domain knowledge
- Preserve the UP主's analogies and examples
- If the video has a checklist or action items, include them exactly

## Step 9: Cleanup

```bash
rm -f /tmp/video_audio.* /tmp/video_video.* /tmp/video_screenshot_*.png
rm -rf /tmp/whisper_out
```

# Error Handling & Fallbacks

Each step has a primary method; fallbacks are tried in order when the primary fails.

| Step | Primary | Fallback 1 | Fallback 2 |
|------|---------|------------|------------|
| **B站 下载** | yt-dlp + Chrome cookies | yt-dlp with `-F` to list formats, pick lower quality | — |
| **抖音 获取信息** | curl iesdouyin.com 分享页 ROUTER_DATA | Chrome AppleScript 提取 `document.querySelector('video').src` | — |
| **抖音 下载** | curl 直接下载视频 CDN 链接 | yt-dlp `--cookies-from-browser chrome` | — |
| **YouTube 下载** | yt-dlp (built-in) | yt-dlp with cookies if geo-restricted | — |
| **截图提取** | ffmpeg 从下载的视频文件提帧 | Chrome 打开视频 → seek → `screencapture -R` 区域截图 | — |
| **获取 URL** | 用户直接粘贴链接 | AppleScript 从 Chrome 当前标签页获取 URL | — |

**具体 fallback 命令：**

### 抖音 share page 失败 → Chrome AppleScript
```bash
osascript -e 'tell application "Google Chrome" to open location "https://www.douyin.com/video/VIDEO_ID"'
sleep 5
osascript -e 'tell application "Google Chrome" to execute active tab of front window javascript "document.querySelector(\"video\").src"'
```

**Security risk**: This requires Chrome → View → Developer → "Allow JavaScript from Apple Events" to be enabled. When this setting is on, **any program on your Mac can run arbitrary JavaScript in Chrome** — reading cookies, manipulating pages, or making requests as you. Only enable it temporarily for the fallback, and **disable it immediately after**. To disable: Chrome → View → Developer → uncheck "Allow JavaScript from Apple Events".

### ffmpeg 提帧失败 → screencapture 区域截图
```bash
osascript -e 'tell application "Google Chrome" to get bounds of front window'
# Returns: {left, top, right, bottom}
screencapture -Rx,y,w,h /tmp/video_screenshot_N.png
```

Requires macOS Screen Recording permission (System Settings → Privacy & Security → Screen Recording).

**Privacy risk**: screencapture captures everything in the specified screen region, including overlapping windows, notifications, or desktop content. Before triggering this fallback, minimize or close any windows showing sensitive information (chat apps, email, banking, password manager, etc.) and enable Do Not Disturb to suppress notification popups.

### 用户未粘贴 URL → 从 Chrome 获取
```bash
osascript -e 'tell application "Google Chrome" to get URL of active tab of front window'
```

---

### 其他常见错误
- If ffmpeg frame extraction fails → try placing `-ss` AFTER `-i` (output seeking, slower but works with more codecs)
- If extracted frame is black → the timestamp may land on a scene transition; try ±2 seconds offset
- If Whisper model not downloaded → auto-downloads on first run
- If B站 API returns no subtitles → proceed with audio-only transcription
- If B站 video is premium-only (1080P) → fall back to lower quality
- If Chrome not running → tell user to open Chrome and log into B站 (needed for yt-dlp cookies)

# Legal Disclaimer

**IMPORTANT — READ BEFORE USING**:

Downloading videos from B站, 抖音, or YouTube may violate their respective Terms of Service. This skill is provided for **personal study, research, and note-taking purposes only**. By using this skill, you agree that:

1. You are responsible for complying with the Terms of Service of each platform
2. You are responsible for complying with applicable copyright laws in your jurisdiction
3. You will not redistribute downloaded videos, transcripts, or screenshots publicly
4. You will not use this skill for commercial purposes or content piracy
5. The skill author(s) assume no liability for any misuse or legal consequences

If you are unsure whether your use case is legal, consult a qualified legal professional before using this skill.

# Security & Privacy Notes

- **All processing is local** — audio and video never leave the machine
- **B站**: Chrome cookies extracted by yt-dlp for authentication only; cookies are never written to disk
- **抖音**: No authentication needed — `iesdouyin.com/share/video/` is a public page; no cookies or login required
- **YouTube**: yt-dlp built-in support, no browser cookies needed
- **Whisper**: Runs fully offline after model download; audio processed locally, never uploaded
- **Screenshots**: Extracted locally from downloaded video via ffmpeg (no screencapture, no desktop exposure)
- **Temporary files**: All `/tmp/video_*` and `/tmp/whisper_out` are cleaned up after notes are saved. Note: `/tmp` is world-readable on macOS — on shared machines, other users could theoretically access temp files during processing
- **Notes**: Saved locally to your filesystem only; you control where they go
- **AppleScript / screencapture (仅 fallback)**: 默认不使用。仅当主方法（curl/ffmpeg）失败时触发。详见上方 Fallback 章节的安全与隐私警告

# Usage Examples

**B站**: "帮我记一下这个视频的笔记 https://www.bilibili.com/video/BV1a59oBmEzZ/"
→ Detect platform → download with Chrome cookies → transcribe → screenshot → write notes → save

**抖音**: "把这个抖音视频转成笔记 https://www.douyin.com/video/XXXXX"
→ Detect platform → iesdouyin.com share page (no auth) → curl download → transcribe → screenshot → write notes → save

**YouTube**: "记一下这个 YouTube 视频 https://www.youtube.com/watch?v=XXXXX"
→ Detect platform → download (no auth needed) → transcribe → screenshot → write notes → save

**Chrome 当前页** (fallback): "看下这个视频讲了什么" (Chrome already open to video, no URL pasted)
→ AppleScript 从 Chrome 标签页获取 URL → detect platform → run full workflow
