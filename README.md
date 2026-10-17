# universal-extractor

把 B 站、微信公众号、小红书的链接交给 AI，一次完成「识别平台 → 提取正文 → 生成导读 → 存进 Obsidian 库」的全流程 Skill。

这是为 [Codex](https://openai.com/codex/) / Claude Code 一类本地 Agent 写的技能包：你说一句「保存这篇文章」或 `/archive`，Agent 就会按平台调用对应工具，把内容落到统一的目录结构里。

## 支持平台

| 平台 | URL 特征 | 提取方式 |
| --- | --- | --- |
| B 站 | `bilibili.com/video/` | 直连 API 取字幕 → yt-dlp → Whisper 本地转录（三级回退） |
| 微信公众号 | `mp.weixin.qq.com` | MCP 工具 `weixin-articles:read_article` |
| 小红书 | `xiaohongshu.com/explore/` | `xiaohongshu-skills` CLI + 图片 OCR |

## 产出

每篇内容生成一个带完整 frontmatter 的 Markdown 文件，落点固定为：

```
<KB_ROOT>/03Resources/01Clippings/{YYYY}/{MM}/{平台}-{标题}-{YYYYMMDD}.md
```

文件结构：

```markdown
---
type: 长文稿
date: 2026-10-17
平台: bilibili
source: https://...
作者: ""
发布时间: 2026-10-17
关键词: []
加工: 待加工
status: 待确认
一句话: ""
---

> [!abstract]- AI 导读 · 待确认（2026-10-17）
> **一句话**：……
> **讲了什么**：3–5 条要点
> **值得深挖**：……
> **关联**：[[相关笔记]]

---

## 正文
（原始转录全文，一个字不改）
```

`> [!abstract]-` 末尾的 `-` 让导读默认折叠，扫一眼标题就能决定要不要展开。

## 安装

把仓库克隆到你的 Agent 技能目录。Codex 默认读 `~/.agents/skills`，Claude Code 默认读 `~/.claude/skills`：

```powershell
git clone https://github.com/zjzjy/skill-universal-extractor.git "$HOME\.agents\skills\universal-extractor"
```

装好后在对话里 `@` 这个 Skill，或者说「把链接转成文字」「保存到知识库」即可触发。

## 路径配置

`SKILL.md` 里的路径都写成了占位符，使用前按你的机器替换即可：

| 占位符 | 含义 | 示例 |
| --- | --- | --- |
| `<KB_ROOT>` | Obsidian 库根目录 | `D:\Notes` |
| `<SKILLS_DIR>` | Agent 技能目录 | `~/.agents/skills` |
| `<BILI_DIR>` | B 站转录项目目录 | `~/bilibili-auto-transcript` |
| `<TMP_DIR>` | 临时文件目录 | `~/bilibili-auto-transcript/.tmp` |
| `<FFMPEG_BIN>` | ffmpeg 可执行文件所在目录 | `/usr/bin` |

其中 `<BILI_DIR>` 是唯一需要额外准备的目录：B 站转录脚本和 `cookies.txt` 都放在那里。小红书相关脚本放在 `<SKILLS_DIR>/xiaohongshu-skills` 和 `<SKILLS_DIR>/xhs-note-to-md`。

## 依赖

| 依赖 | 用途 | 备注 |
| --- | --- | --- |
| conda 环境 `bili-env` | B 站转录 | `yt-dlp`、`openai-whisper`、`wx-ocr`、`requests` |
| ffmpeg | 音频处理 | 需在 PATH 中，或填进 `<FFMPEG_BIN>` |
| `weixin-articles` MCP | 微信公众号 | 需在 Agent 中配置 |
| `xiaohongshu-skills` | 小红书 | 配套浏览器插件需处于登录态 |
| B 站 Cookie | 取 AI 字幕 | `cookies.txt` 中的 SESSDATA 必须有效 |

## 隐私

仓库不包含 Cookie、密钥或抓取到的正文，也没有作者本机的绝对路径。请勿把 `cookies.txt` 提交进版本库（已在 `.gitignore` 中排除）。

## License

未指定。若要公开复用，建议补一份 LICENSE（如 MIT）。
