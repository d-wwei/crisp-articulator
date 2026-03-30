# Crisp Articulator

[中文文档](README_ZH.md)

One command turns a topic into a published article. Crisp Articulator orchestrates the full content pipeline -- writing, illustration, typesetting, delivery, and publishing -- across independent sub-skills. Publishing is handled by the `/publish` skill ([superb-publisher](../superb-publisher/)) for supported platforms and setup. Works with Claude Code, Codex, Gemini CLI, or any agent that supports skill loading.

## Quick Start

```bash
# Install: symlink into your agent's skill directory
ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator

# Full pipeline: topic -> write -> illustrate -> typeset -> deliver
/articulate "Write a technical blog about AI Agent security" --platform wechat

# Full pipeline + publish via /publish skill
/articulate "AI Agent 安全" --platform wechat --publish

# Publish to multiple platforms at once
/articulate "AI Agent 安全" --platform wechat,medium,x --publish

# Start from an existing draft
/articulate my-article.md --platform zhihu

# Skip to formatting
/articulate my-article.md --from typeset --platform medium --theme elegant

# Full auto mode (publish gate still requires confirmation)
/articulate --auto "AI Agent 入门指南" --platform wechat --publish

# Publish an existing HTML file directly
/articulate article.html --from publish --platform wechat
```

## Pipeline

```
topic --> Write --> Visualize --> Typeset --> Deliver --> [/publish]
            |          |             |           |            |
       great-writer  brilliant-   typeset    clipboard   superb-
                     visualizer              + preview   publisher
```

| Stage | Skill / Tool | Input | Output |
|-------|-------------|-------|--------|
| Write | [great-writer](https://github.com/d-wwei/great-writer) | Topic text | Markdown file |
| Visualize | [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | Markdown file | Markdown with images |
| Typeset | [typeset](https://github.com/d-wwei/excellent-typesetter) | Markdown + platform | Platform-specific HTML |
| Deliver | (built-in) | HTML file | Clipboard + browser preview |
| Publish | [superb-publisher](../superb-publisher/) (/publish skill) | HTML + metadata | Published content |

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
| `--thumb` | Cover image file path | none |
| `--publish-args` | Extra arguments passed through to /publish | none |
| `--check-deps` | *(flag)* check dependencies and exit | - |

## Dependencies

### Sub-skills (for pipeline stages)

| Skill | Required | Without it |
|-------|----------|-----------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | **Yes** | Cannot run |
| [great-writer](https://github.com/d-wwei/great-writer) | No | Write stage skipped |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | No | Visualize stage skipped |
| [superb-publisher](../superb-publisher/) | No | Publish stage unavailable (pipeline ends at Deliver) |

### Install

```bash
# Required sub-skill
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# Optional sub-skills
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer
ln -s /path/to/superb-publisher ~/.claude/skills/superb-publisher
```

Run `/articulate --check-deps` to verify everything is in place.

## Architecture

### Loose Coupling

The orchestrator knows exactly three things per sub-skill:

1. **Name** -- skill name
2. **Input** -- topic text or file path
3. **Output** -- file path or ID

Nothing else. No internal phases, no engine configs, no API endpoints, no theme logic. All sub-skills are invoked via the agent's Skill tool. Any component can be refactored or replaced independently.

### Extension Slots

| Slot | Position | Status | Purpose |
|------|----------|--------|---------|
| Pre-Write | Before Write | Planned | Topic selection, content strategy |
| Post-Deliver | After Deliver | **Active** | Multi-platform publishing (via /publish skill) |

The Publish stage delegates to the `/publish` skill (superb-publisher). To add new platforms, extend superb-publisher -- the orchestrator does not need modification.

## License

MIT
