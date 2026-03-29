# WeChat Publish Stage — Design Spec

**Date:** 2026-03-29
**Author:** Eli + Claude
**Status:** Implemented

---

## 1. Motivation

The Deliver stage produces HTML in the clipboard and opens a browser preview. For WeChat, the user then manually pastes into the editor and uploads images one by one. This gap between "HTML ready" and "article published" is the exact use case the Post-Deliver extension slot was designed for.

`wechat-cli` already automates the full publish flow: pre-flight check, image upload, draft creation, and publishing. This spec documents how it integrates into the pipeline.

---

## 2. Architecture Fit

### 2.1 Position

The Publish stage occupies the **Post-Deliver extension slot**. It runs after Deliver and is the final pipeline stage.

```
write -> illustrate -> format -> deliver -> [publish]
```

Brackets = optional. The stage is conditional on `--publish` + `--platform wechat`.

### 2.2 Invocation Pattern

The three existing sub-skills (great-writer, brilliant-visualizer, typeset) are Claude Code Skills invoked via the `Skill` tool. `wechat-cli` is a bash CLI invoked via `Bash` commands.

This is a different invocation mechanism but the **coupling contract is identical**:

| | Sub-skills (Skill tool) | wechat-cli (Bash tool) |
|---|---|---|
| What orchestrator knows | Skill name, input type, output type | CLI commands, input type, output type |
| What orchestrator doesn't know | Internal phases, engines, modes | API endpoints, JSON encoding, token cache |
| Dependency check | File existence (`SKILL.md`) | `command -v wechat-cli` |

### 2.3 Coupling Contract

**Knows:**
- Tool name: `wechat-cli`
- Commands used: `check`, `upload-images`, `upload-thumb`, `draft`, `publish`, `status`
- Input: HTML file path + metadata flags (title, author, digest, thumb)
- Output: media_id (from `draft`), publish_id (from `publish`)

**Does not know:**
- WeChat API endpoint URLs
- JSON payload construction
- Token caching strategy (2h TTL, 5min buffer)
- Error code → message mapping logic
- sed/awk JSON parsing internals

---

## 3. Safety Design

### 3.1 Opt-in

Publishing requires explicit `--publish` flag. Without it, the pipeline ends at Deliver as before. No behavior change for existing users.

### 3.2 Platform Gating

The Publish stage only activates for `--platform wechat`. Other platforms skip Publish silently.

### 3.3 Mandatory Confirmation Gate

Publishing is irreversible (article goes live to followers). The confirmation gate fires even in `--auto` mode. This is the one exception to auto-mode's "skip all gates" rule.

The gate shows: title, author, digest, cover, HTML path, image count. User can edit metadata before confirming.

### 3.4 Graceful Degradation

| Condition | Behavior |
|-----------|----------|
| wechat-cli not installed | Skip publish, end at Deliver |
| wechat-cli not configured | Skip publish, suggest `wechat-cli init` |
| `--publish` without `--platform wechat` | Skip publish, note platform limitation |
| API error during draft/publish | Show translated error, offer retry |

---

## 4. Metadata Extraction

When `--title` and `--digest` are not provided:

- **Title**: Extract from first `<h1>` in HTML. Fallback: first `# ` heading from Markdown source. Fallback: ask user.
- **Digest**: Extract first `<p>` text content, truncated to 120 characters. Fallback: empty.
- **Author**: Empty if not provided (WeChat allows this).

---

## 5. Publish Sub-steps

1. Extract/validate metadata
2. `wechat-cli check` — pre-flight validation
3. `wechat-cli upload-images` — batch upload local images, replace URLs in-place
4. `wechat-cli upload-thumb` — upload cover image (if provided)
5. **Confirmation gate** (mandatory)
6. `wechat-cli draft` — create article draft → media_id
7. `wechat-cli publish` — submit for publishing → publish_id

---

## 6. Future Extensions

The same Post-Deliver slot can host additional platform CLIs:

```
# Future
deliver -> [publish-wechat] -> [publish-zhihu] -> [publish-juejin]
```

Each publisher would:
- Have its own platform gate (`--platform X`)
- Have its own CLI tool
- Share the same coupling contract pattern
- Not interfere with other publishers

The wechat-cli integration establishes this pattern.
