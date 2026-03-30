# Crisp Articulator

[中文文档](README_ZH.md)

One command turns a topic into a published article. Crisp Articulator orchestrates the full content pipeline -- writing, illustration, typesetting, delivery, and multi-platform publishing -- across independent sub-skills. Supports 6 platforms: WeChat, Medium, LinkedIn, Xiaohongshu, Douyin, X/Twitter. Works with Claude Code, Codex, Gemini CLI, or any agent that supports skill loading.

## Quick Start

```bash
# Install: symlink into your agent's skill directory
ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator

# Full pipeline: topic → write → illustrate → typeset → deliver
/articulate "Write a technical blog about AI Agent security" --platform wechat

# Full pipeline + publish directly to WeChat Official Account
/articulate "AI Agent 安全" --platform wechat --publish --thumb cover.png

# Publish to Medium as draft
/articulate "AI Agent Security" --platform medium --publish --tags "AI,Security" --status draft

# Publish to multiple platforms at once
/articulate "AI Agent 安全" --platform wechat,medium,x --publish

# Start from an existing draft
/articulate my-article.md --platform zhihu

# Skip to formatting
/articulate my-article.md --from typeset --platform medium --theme elegant

# Full auto mode (publish gate still requires confirmation)
/articulate --auto "AI Agent 入门指南" --platform wechat --publish

# Publish an existing HTML file directly
/articulate article.html --from publish --title "我的文章"

# Publish to Xiaohongshu with images
/articulate article.html --from publish --platform xhs --tags "AI,tech"

# Publish to LinkedIn
/articulate article.html --from publish --platform linkedin
```

## Pipeline

```
topic ──→ Write ──→ Visualize ──→ Typeset ──→ Deliver ──→ [Publish]
            │          │             │           │             │
       great-writer  brilliant-   typeset    clipboard    platform
                     visualizer              + preview     CLIs
```

| Stage | Skill / Tool | Input | Output |
|-------|-------------|-------|--------|
| Write | [great-writer](https://github.com/d-wwei/great-writer) | Topic text | Markdown file |
| Visualize | [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | Markdown file | Markdown with images |
| Typeset | [typeset](https://github.com/d-wwei/excellent-typesetter) | Markdown + platform | Platform-specific HTML |
| Deliver | (built-in) | HTML file | Clipboard + browser preview |
| Publish | Platform CLIs (see below) | HTML + metadata | Published content |

### Supported Publish Platforms

| Platform | CLI Tool | Auth | Content Format |
|----------|---------|------|---------------|
| WeChat | [wechat-cli](https://github.com/d-wwei/wechat-cli) | AppID + AppSecret | HTML (WeChat-optimized) |
| Medium | [medium-cli](https://github.com/d-wwei/medium-cli) | Integration Token | HTML or Markdown |
| LinkedIn | [linkedin-cli](https://github.com/d-wwei/linkedin-cli) | OAuth 2.0 | Plain text + images |
| Xiaohongshu | xhs-cli (social-cli) | QR code login | Text + images |
| Douyin | douyin-cli (social-cli) | QR code login | Video + description |
| X/Twitter | x-cli (social-cli) | Browser login | Short text + images |

Multi-platform: `--platform wechat,medium,x` publishes to each platform in sequence.

Each stage has a review checkpoint. Pass `--auto` to skip them all -- except Publish, which always confirms before sending (publishing is irreversible).

## Smart Entry Point

The orchestrator detects what you have and starts at the right stage:

| Input | Starts at |
|-------|----------|
| Topic text | Write (full pipeline) |
| `.md` file without images | Visualize |
| `.md` file with images on disk | Typeset |
| `.html` file | Deliver |

Override with `--from write|visualize|typeset|deliver|publish`.

## Parameters

| Flag | Values | Default |
|------|--------|---------|
| `--platform` | wechat, medium, linkedin, xhs, douyin, x, zhihu, juejin, blog (comma-separated for multi) | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver, publish | auto-detect |
| `--auto` | *(flag)* | off |
| `--publish` | *(flag)* enable publish stage | off |
| `--title` | Article title | auto-extract from `<h1>` |
| `--author` | Author name | empty |
| `--digest` | Summary (max 120 chars) | auto-extract from first paragraph |
| `--thumb` | Cover image file path (WeChat) | none |
| `--tags` | Comma-separated tags (Medium, XHS) | none |
| `--status` | draft, public, unlisted (Medium) | draft |
| `--video` | Video file path (Douyin, required) | none |
| `--cover` | Cover image for video (Douyin) | none |
| `--check-deps` | *(flag)* check dependencies and exit | - |

## Dependencies

### Sub-skills (for pipeline stages)

| Skill | Required | Without it |
|-------|----------|-----------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | **Yes** | Cannot run |
| [great-writer](https://github.com/d-wwei/great-writer) | No | Write stage skipped |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | No | Visualize stage skipped |

### Platform CLIs (for publish stage, all optional)

| CLI | Platform | Install |
|-----|----------|---------|
| [wechat-cli](https://github.com/d-wwei/wechat-cli) | WeChat | `~/.local/bin/wechat-cli` + `wechat-cli init` |
| [medium-cli](https://github.com/d-wwei/medium-cli) | Medium | `~/.local/bin/medium-cli` + `medium-cli init` |
| [linkedin-cli](https://github.com/d-wwei/linkedin-cli) | LinkedIn | `~/.local/bin/linkedin-cli` + `linkedin-cli init && linkedin-cli login` |
| xhs-cli | Xiaohongshu | `social-cli/bin/xhs-cli` + `xhs-cli login` |
| douyin-cli | Douyin | `social-cli/bin/douyin-cli` + `douyin-cli login` |
| x-cli | X/Twitter | `social-cli/bin/x-cli` + `x-cli login` |

### Install

```bash
# Required sub-skill
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# Optional sub-skills
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer

# API-based CLIs (for --publish)
ln -s /path/to/wechat-cli ~/.local/bin/wechat-cli && wechat-cli init
ln -s /path/to/medium-cli ~/.local/bin/medium-cli && medium-cli init
ln -s /path/to/linkedin-cli ~/.local/bin/linkedin-cli && linkedin-cli init && linkedin-cli login

# Browser-based CLIs (social-cli monorepo)
cd /path/to/social-cli && npm install && npm run build
# Then login to each platform as needed:
xhs-cli login
douyin-cli login
x-cli login
```

Run `/articulate --check-deps` to verify everything is in place.

## Architecture

### Loose Coupling

The orchestrator knows exactly three things per sub-skill or tool:

1. **Name** — skill name or CLI command
2. **Input** — topic text or file path
3. **Output** — file path or ID

Nothing else. No internal phases, no engine configs, no API endpoints, no theme logic. Sub-skills are invoked via the agent's Skill tool; CLI tools (wechat-cli) via Bash. Different mechanism, same contract. Any component can be refactored or replaced independently.

### Extension Slots

| Slot | Position | Status | Purpose |
|------|----------|--------|---------|
| Pre-Write | Before Write | Planned | Topic selection, content strategy |
| Post-Deliver | After Deliver | **Active** | Multi-platform publishing |

The Publish stage occupies the Post-Deliver slot with 6 registered platform CLIs. Each platform has its own content adaptation rules, confirmation gate, and error handling. New platforms can be added by implementing a CLI following the same contract (`{platform}-cli publish [--options]`).

## License

MIT
