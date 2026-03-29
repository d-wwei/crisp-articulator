# Crisp Articulator

[中文文档](README_ZH.md)

A skill for AI agents that orchestrates the full content creation pipeline: write, illustrate, format, and deliver. One command, four stages, ready-to-publish output. Works with any agent that supports skill loading (Claude Code, Codex, Gemini CLI, etc.).

## What it does

You give it a topic. It calls [great-writer](https://github.com/d-wwei/great-writer) to write the article, [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) to add diagrams and images, and [typeset](https://github.com/d-wwei/excellent-typesetter) to format everything for your target platform. The final HTML lands in your clipboard, and a browser preview opens automatically.

Each sub-skill runs independently. Crisp Articulator only knows their names and what file paths they produce. When a sub-skill gets updated, nothing here needs to change.

## Quick Start

```bash
# Install: symlink into your agent's skill directory
# Claude Code:  ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator
# Codex:        ln -s /path/to/crisp-articulator ~/.agents/skills/crisp-articulator
# Gemini CLI:   ln -s /path/to/crisp-articulator ~/.gemini/skills/crisp-articulator
ln -s /path/to/crisp-articulator <your-agent-skill-dir>/crisp-articulator

# Full pipeline: topic → article → illustrations → formatted HTML
/articulate "Write a technical blog about AI Agent security" --platform wechat

# Already have a markdown draft? Start from illustration
/articulate my-article.md --platform zhihu

# Skip straight to formatting
/articulate my-article.md --from typeset --platform medium --theme elegant

# No confirmations, just run everything
/articulate --auto "AI Agent 入门指南" --platform wechat
```

## Pipeline

```
topic text ──→ Write ──→ Visualize ──→ Typeset ──→ Deliver
               │          │             │            │
          great-writer  brilliant-   typeset     clipboard
                        visualizer               + preview
```

| Stage | Skill | Input | Output |
|-------|-------|-------|--------|
| Write | great-writer | Topic text | Markdown file |
| Visualize | brilliant-visualizer | Markdown file | Markdown file with images |
| Typeset | typeset | Markdown file + platform | Platform-specific HTML |
| Deliver | (built-in) | HTML file | Clipboard + browser preview |

Between each stage, you get a checkpoint to review, edit, or stop. Pass `--auto` to skip all checkpoints.

## Smart Entry Point

You don't always start from scratch. The orchestrator detects what you have and picks the right starting stage:

| You provide | Detected start |
|-------------|---------------|
| Topic text | Write (full pipeline) |
| `.md` file, no images | Visualize |
| `.md` file with images | Typeset |
| `.html` file | Deliver (preview only) |

Override with `--from write|visualize|typeset|deliver`.

## Parameters

| Flag | Options | Default |
|------|---------|---------|
| `--platform` | wechat, zhihu, juejin, medium, linkedin, x, blog | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver | auto-detect |
| `--auto` | *(flag)* | off |
| `--check-deps` | *(flag)* | - |

## Dependencies

| Skill | Required | Without it |
|-------|----------|-----------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | Yes | Cannot run |
| [great-writer](https://github.com/d-wwei/great-writer) | No | Write stage skipped |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | No | Visualize stage skipped |

Run `/articulate --check-deps` to verify all sub-skills are installed.

### Install sub-skills

```bash
# Required
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# Optional
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer
```

## Architecture: Loose Coupling

The orchestrator depends on exactly three things per sub-skill:
1. **Skill name** (to invoke it)
2. **Input type** (topic text or file path)
3. **Output type** (file path)

It does not read, reference, or depend on any sub-skill's internal files, configuration, modes, phases, or engines. Sub-skills can be refactored, extended, or rewritten without touching the orchestrator.

## Extension Slots

Two extension points are reserved for future versions:

| Slot | Position | Purpose |
|------|----------|---------|
| Pre-Write | Before Write stage | Topic selection, content strategy |
| Post-Deliver | After Deliver stage | Platform publishing, cross-posting |

These are documented placeholders. No skill is registered to them in v1.

## License

MIT
