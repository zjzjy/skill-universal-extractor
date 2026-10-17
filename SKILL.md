---
name: universal-extractor
description: "多平台内容提取器。输入B站/微信公众号/小红书的链接，自动识别平台并转录成文字，存入知识库。触发：保存这篇文章、把链接转成文字、整理内容、universal extractor、存档、保存到知识库、/archive"
---

# 多平台内容提取器

根据 URL 自动识别平台，调用对应工具提取内容，统一存储到知识库。

## 平台识别

| URL 特征 | 平台 | 处理方式 |
|----------|------|----------|
| `bilibili.com/video/` | B站 | `scripts/transcribe.py` 的 `transcribe_video()` |
| `mp.weixin.qq.com` | 微信公众号 | MCP `weixin-articles:read_article` |
| `xiaohongshu.com/explore/` | 小红书 | `xiaohongshu-skills` CLI + OCR |

## 工作流

### 1. 识别平台

从 URL 中判断平台类型。

### 2. 提取内容

**B站视频：**
```python
import sys; sys.path.insert(0, r'C:\Users\Hanah\bilibili-auto-transcript')
from scripts.transcribe import transcribe_video
result = transcribe_video(url)
# 结果已按 §3 统一存储规则写进 Obsidian 库，不需额外移动
```

字幕获取优先级（v6，参考 Bilibili-Obsidian-Clipper 的直连 API 方案）：

1. **直连 B 站 API 秒取字幕**：`x/web-interface/view` 拿元数据 → `x/player/wbi/v2`（失败兜底 `x/player/v2`）拿字幕轨 → 下载字幕 JSON；中文轨优先，并按视频时长校验字幕是否匹配
2. **yt-dlp 字幕下载**：直连 API 不可用时回退
3. **Whisper 本地语音转文字**：前两级都拿不到时才使用

⚠️ B 站 AI 字幕需要登录态：`cookies.txt` 中的 SESSDATA 必须有效。若输出提示「该视频字幕需要 B 站登录态」，说明 cookie 已过期，登录 bilibili.com 后重新导出 cookie 覆盖 `C:\Users\Hanah\bilibili-auto-transcript\cookies.txt` 即可秒出字幕。

B站转录使用 bili-env conda 环境。结果按下面的「统一存储」规则直接写进 Obsidian 库。

**微信公众号：**
使用 MCP 工具 `weixin-articles:read_article(url)` 获取文章 Markdown 和图片。将内容和图片保存到知识库。

**小红书：**
```powershell
# 解析 URL 中的 feed_id 和 xsec_token
# 调用 xiaohongshu-skills CLI
cd C:\Users\Hanah\.agents\skills\xiaohongshu-skills
python scripts\cli.py get-feed-detail --feed-id <id> --xsec-token <token> > %TEMP%\xhs_detail.json
# 使用 build_xhs_note_md.py 生成 Markdown（含 OCR）
python C:\Users\Hanah\.agents\skills\xhs-note-to-md\scripts\build_xhs_note_md.py --detail-json %TEMP%\xhs_detail.json --output <output.md> --asset-dir <assets_dir> --mode text-ocr --ocr-engine wx
```

### 3. 统一存储

所有内容写进 **Obsidian 库**（不是 vault 外面的目录）：

```
C:\Users\Hanah\OneDrive\obsidian-v2\03Resources\01Clippings\{YYYY}\{MM}\
```

日期取视频 / 文章的**发布时间**，取不到就用当天。

文件名：`{平台}-{标题}-{YYYYMMDD}.md`

- 平台用 `bilibili` / `weixin` / `xiaohongshu`
- 标题里的 `/ \ : * ? " < > |` 换成 `-`，截到 80 字
- 配图放同级目录 `{文稿名}_assets/`

**每个文件都必须带下面这套 frontmatter**（库的规范，字段名不要另创）：

```markdown
---
type: 长文稿
date: YYYY-MM-DD
平台: bilibili
source: https://...    # 库的规范字段名就是 source，别写「链接」
作者: ""
发布时间: YYYY-MM-DD
关键词: []
加工: 待加工
status: 待确认
一句话: ""          # 留空，由加工步骤填
---

> [!abstract]- AI 导读 · 待确认（YYYY-MM-DD）
> **一句话**：……
> **讲了什么**：3–5 条要点
> **值得深挖**：……
> **关联**：[[相关笔记]]

---

## 正文
（原始转录全文，一个字不改）
```

两个要点：

- `> [!abstract]-` 末尾那个 `-` 让 callout **默认折叠**——时间不够时只看标题和前几行。这是「概括挂在上面」的实现方式，不能省。
- **AI 导读在转录的同时就要生成**，不能留给后面补。正文一个字不改，只在前面加导读。

## 输出要求

- B站：转录文本（已自动分段纠错）+ AI 导读 + 完整 frontmatter；直连字幕不落盘，Whisper/yt-dlp 回退产生的临时文件自动清理
- 微信：Markdown 正文（不保存图片到本地）+ AI 导读 + 完整 frontmatter
- 小红书：正文 + 图片 OCR 内容 + AI 导读 + 完整 frontmatter；OCR 后清理下载的图片和 JSON 临时文件
- 落点一律 `03Resources/01Clippings/{YYYY}/{MM}/`，`加工: 待加工` 不要漏（Home 的 Inbox 面板靠它抓取）
- 多个链接批量处理，每个处理完报告结果

## 清理规则

| 平台 | 清理内容 |
|------|----------|
| B站 | `bilibili_audio*`、`bilibili_subtitle*`、`bilibili_ai_subtitle*`（仅回退路径产生，转录完成后自动清理） |
| 微信 | 无需清理（MCP 直接返回内容，不落盘） |
| 小红书 | `{name}_images/` 目录、`%TEMP%\xhs_note_detail.json`（MD 生成后删除） |

## 依赖

- conda 环境 `bili-env`：yt-dlp, openai-whisper, wx-ocr, requests
- ffmpeg：`~/ffmpeg/ffmpeg-8.1.2-essentials_build/bin/`
- MCP `weixin-articles`：已配置
- XHS Bridge 浏览器插件：已安装
- `xiaohongshu-skills`：已安装
- cookies.txt：B站 Cookie（必须为有效登录态），位于 `~/bilibili-auto-transcript/cookies.txt`
