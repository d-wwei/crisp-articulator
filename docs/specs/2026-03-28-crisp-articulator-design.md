# Crisp Articulator — Design Spec

**Date:** 2026-03-28
**Author:** Eli + Claude
**Status:** Draft

---

## 1. Overview

Crisp Articulator is a content creation pipeline orchestrator for Claude Code. It chains three existing skills into a complete write-illustrate-format-deliver flow:

```
great-writer → brilliant-visualizer → typeset → deliver
```

**Core principle: loose coupling.** The orchestrator depends only on skill names and I/O contracts (file paths), never on sub-skill internals. Any sub-skill can be refactored, add new modes, or change its internal pipeline without affecting the orchestrator.

**Command:** `/articulate`

---

## 2. Architecture

### 2.1 Pipeline Descriptor

The orchestrator defines a stage table. Each stage declares: name, target skill, input type, output type, and whether it can be skipped.

| Stage | Skill | Input | Output | Skippable | Hard Dep |
|-------|-------|-------|--------|-----------|----------|
| Write | `great-writer` | Topic text + writing type | Markdown file path | Yes | No |
| Visualize | `brilliant-visualizer` | Markdown file path | Markdown file path (with images) | Yes | No |
| Typeset | `typeset` | Markdown path + `--platform` + `--theme` | HTML file path | No | **Yes** |
| Deliver | (built-in) | HTML file path | Clipboard + browser preview | No | N/A |

### 2.2 Coupling Contract

**What the orchestrator knows about each sub-skill:**
- Skill name (for `Skill` tool invocation)
- Expected input format (topic text or file path)
- Expected output format (file path)
- Invocation parameters (e.g., `--platform wechat` for typeset)

**What the orchestrator does NOT know (and must not depend on):**
- great-writer's 5 internal phases (Research, Draft, Review, Humanize, Finalize)
- great-writer's 8 writing modes
- brilliant-visualizer's engine routing and fallback chains
- brilliant-visualizer's engine implementation files (`engines/*.md`)
- typeset's theme rendering logic, CJK spacing rules, or platform-specific HTML constraints

### 2.3 Data Flow

```
User Input (topic or .md path)
       │
  ┌────▼──────────┐
  │ Input Parser   │  Parse args, detect entry point
  │ & Router       │  Collect --platform, --theme, --auto, --from
  └────┬──────────┘
       │
  ┌────▼──────────┐
  │ Stage: Write   │  → Skill("great-writer", topic + type)
  │                │  ← Returns: markdown file path
  │  [gate]        │  Show summary, ask to continue (skip if --auto)
  └────┬──────────┘
       │
  ┌────▼──────────┐
  │ Stage:         │  → Skill("brilliant-visualizer", md path)
  │ Visualize      │  ← Returns: markdown file path (with images)
  │  [gate]        │  Show summary, ask to continue (skip if --auto)
  └────┬──────────┘
       │
  ┌────▼──────────┐
  │ Stage: Typeset │  → Skill("typeset", md path + --platform + --theme)
  │                │  ← Returns: HTML file path
  └────┬──────────┘
       │
  ┌────▼──────────┐
  │ Stage: Deliver │  pbcopy HTML content
  │  (built-in)    │  open HTML in browser
  │                │  Print summary
  └───────────────┘
```

---

## 3. Entry Point Detection

When the user runs `/articulate [args]`, the orchestrator determines the starting stage:

### 3.1 Automatic Detection

```
Input is a .md file path?
  ├── File contains image refs (![...](images/...)) AND images exist on disk?
  │   → Start at: Typeset (article already illustrated)
  │
  └── Plain Markdown, no images or images missing?
      → Start at: Visualize (article needs illustration)

Input is a .html file path?
  → Start at: Deliver (just preview/copy)

Input is plain text (not a file path)?
  → Start at: Write (start from scratch)
```

### 3.2 Explicit Override

`--from <stage>` overrides automatic detection:

```bash
/articulate article.md --from typeset    # Skip visualization, go straight to typeset
/articulate article.md --from write      # Rewrite from scratch using article as reference
```

### 3.3 Skipping Stages

When starting from a middle stage, all prior stages are skipped. The orchestrator does not re-run earlier stages unless the user explicitly requests it via `--from`.

---

## 4. Interaction Modes

### 4.1 Stage-Gated Mode (Default)

After each stage completes, the orchestrator pauses and shows a summary:

```
Write complete:
  File: ~/articles/ai-agent-tech-blog.md
  Words: ~2,500
  Type: Tech Article

Continue to illustration? [Y/n/edit]
```

User options at each gate:
- **Y (default):** Proceed to next stage
- **n:** Stop here (output current artifact)
- **edit:** User manually edits the artifact, then resumes

### 4.2 Auto Mode (`--auto`)

All gates are skipped. Sub-skills that have their own internal confirmations (e.g., brilliant-visualizer's illustration plan confirmation) also use defaults.

```bash
/articulate --auto "写一篇关于 AI Agent 安全的技术博客" --platform wechat
```

---

## 5. Command Reference

```bash
# Full pipeline from topic
/articulate <topic>
/articulate "写一篇关于 AI Agent 的技术博客"

# From existing markdown (auto-detects entry point)
/articulate article.md
/articulate ~/drafts/my-post.md

# Explicit entry point
/articulate article.md --from typeset
/articulate article.md --from visualize

# Full auto mode
/articulate --auto <topic>

# Platform and theme
/articulate <topic> --platform wechat
/articulate <topic> --platform wechat --theme tech
/articulate <topic> --platform zhihu --theme elegant

# Dependency check only
/articulate --check-deps

# Supported platforms (passed through to typeset):
#   wechat, zhihu, juejin, medium, linkedin, x, blog

# Supported themes (passed through to typeset):
#   default, elegant, tech, minimal, vibrant
```

---

## 6. Dependency Management

### 6.1 Dependency Declaration

Three sub-skills are declared as dependencies:

| Skill | Path | Required Level |
|-------|------|---------------|
| great-writer | `~/.claude/skills/great-writer/` | Soft (degraded without) |
| brilliant-visualizer | `~/.claude/skills/brilliant-visualizer/` | Soft (degraded without) |
| typeset | `~/.claude/skills/typeset/` | **Hard (cannot run without)** |

### 6.2 First-Run Check

On first invocation, the orchestrator checks for all three sub-skills:

```bash
for skill_dir in great-writer brilliant-visualizer typeset; do
  if [ ! -f ~/.claude/skills/$skill_dir/SKILL.md ]; then
    echo "MISSING: $skill_dir"
  else
    echo "OK: $skill_dir"
  fi
done
```

### 6.3 Missing Dependency Handling

```
All present → Normal execution

Some missing → Show list, ask user:
  "The following sub-skills are not installed: [list]
   Install them now? (Y/n)"

  User confirms → Install each missing skill
  User declines →
    Missing Write skill    → Can only start from Visualize stage
    Missing Visualize skill → Skip illustration stage
    Missing Typeset skill  → ERROR: Cannot run. Typeset is required.
```

### 6.4 Installation Source

Each sub-skill's install source will be declared in the dependency table. The install mechanism depends on how Eli distributes these skills (e.g., git clone, curl download, or a future skill registry). The orchestrator will run the appropriate install command for each missing skill.

**Decision needed before implementation:** Define the install source URL or command for each sub-skill. This can be added to the SKILL.md dependency section once the distribution method is decided.

---

## 7. Deliver Stage (Built-in)

The final stage is handled by the orchestrator itself, not a sub-skill.

### 7.1 Actions

1. **Copy HTML to clipboard:**
   ```bash
   cat output.html | pbcopy
   ```

2. **Open browser preview:**
   ```bash
   open output.html
   ```

3. **Print summary:**
   ```
   Done!
     Article: ~/articles/ai-agent-tech-blog.md
     Images:  5 (3 mermaid, 1 AI-generated, 1 stock photo)
     Output:  ~/articles/ai-agent-tech-blog.html
     Platform: WeChat (wechat)
     Theme:   tech
     Words:   ~2,500

     HTML copied to clipboard. Paste into WeChat editor.
     Preview opened in browser.
   ```

---

## 8. Extension Slots (v2+)

### 8.1 Upstream: Topic Scout

Position: `pre-write`

Potential capabilities:
- Trending topic detection
- Competitor content analysis
- User feedback aggregation
- SEO gap analysis

v1 action: No skill registered. The `pre-write` position exists in the stage table but is empty.

### 8.2 Downstream: Platform Publisher

Position: `post-deliver`

Potential capabilities:
- Direct publish to WeChat via API
- Cross-post to multiple platforms
- Schedule publishing
- Track engagement metrics

v1 action: No skill registered. The `post-deliver` position exists in the stage table but is empty.

### 8.3 Adding Extensions

Future skills can be registered to extension slots by adding a stage entry to the pipeline descriptor:

```yaml
# Example: adding a topic scout in v2
- name: scout
  skill: topic-scout
  position: pre-write
  input: content strategy / keywords
  output: topic brief (text)
  skippable: true
```

No changes needed to existing stages or sub-skills.

---

## 9. Error Handling

| Scenario | Behavior |
|----------|----------|
| Sub-skill invocation fails | Show error, ask user: retry / skip stage / abort |
| Sub-skill produces no output file | Show error, abort pipeline |
| User cancels at gate | Output current artifact, print partial summary |
| `--auto` mode + sub-skill error | Abort pipeline, show error summary |
| Missing optional dependency at runtime | Skip that stage, warn user |
| Missing `typeset` dependency | Abort immediately with install instructions |

---

## 10. File Structure

```
~/.claude/skills/crisp-articulator/
  SKILL.md              # Main skill file: routing, pipeline, gates, deliver
  README.md             # Usage documentation
  docs/
    specs/
      2026-03-28-crisp-articulator-design.md   # This file
```

No sub-directories for engines, modes, or templates. The orchestrator's complexity is in flow control, not content generation.

---

## 11. Non-Goals (Explicit)

- **No content generation:** The orchestrator does not write, illustrate, or format. Sub-skills do that.
- **No sub-skill internal knowledge:** The orchestrator never reads or references sub-skill internal files (modes/*.md, engines/*.md, core/*.md, themes/*.md).
- **No platform API integration in v1:** No direct publishing to any platform.
- **No topic suggestion in v1:** No upstream content strategy features.
- **No caching or state persistence:** Each invocation is stateless. No session continuity between runs.

---

## 12. Success Criteria

1. User can run `/articulate "topic" --platform wechat` and get a WeChat-ready HTML in clipboard
2. Each sub-skill can be independently updated without changing the orchestrator
3. User can start from any stage with an existing artifact
4. Adding a new pipeline stage requires only adding a stage entry, not modifying existing logic
5. Missing optional skills degrade gracefully instead of failing
