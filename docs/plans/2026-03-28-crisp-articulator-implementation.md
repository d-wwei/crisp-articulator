# Crisp Articulator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code skill that orchestrates great-writer, brilliant-visualizer, and typeset into a loose-coupled content creation pipeline with stage gates and auto-detection.

**Architecture:** Single SKILL.md file containing: YAML frontmatter with triggers/dependencies, a bash dependency checker, prompt instructions for input parsing, pipeline stage definitions with I/O contracts, stage gate logic, built-in deliver stage, and extension slot placeholders.

**Tech Stack:** Claude Code skill (Markdown prompt), Bash (dependency check + deliver actions)

---

## File Structure

| File | Purpose |
|------|---------|
| Create: `~/.claude/skills/crisp-articulator/SKILL.md` | Main skill file: frontmatter, dependency check, input parsing, pipeline stages, gates, deliver, extensions |
| Create: `~/.claude/skills/crisp-articulator/README.md` | Usage documentation with command examples |

This is a prompt-based skill, not traditional code. The "implementation" is writing precise instructions that Claude Code follows. Each task produces a complete, verifiable section of the SKILL.md.

---

### Task 1: SKILL.md Frontmatter + Dependency Check

**Files:**
- Create: `~/.claude/skills/crisp-articulator/SKILL.md`

- [ ] **Step 1: Write YAML frontmatter**

Write the opening of SKILL.md with the full frontmatter block:

```markdown
---
name: crisp-articulator
description: >
  Content creation pipeline orchestrator. Chains great-writer -> brilliant-visualizer ->
  typeset into a complete write-illustrate-format-deliver flow. Loose coupling: depends
  only on skill names and I/O contracts, not sub-skill internals.
triggers:
  # Chinese
  - 一键发文
  - 写文章并排版
  - 从头写一篇
  - 写完整文章
  - 全流程写作
  - 内容流水线
  # English
  - articulate
  - content pipeline
  - write and publish
  - full article
  - end to end article
---
```

- [ ] **Step 2: Write the skill title and overview section**

Immediately after the frontmatter, add:

```markdown
# Crisp Articulator -- Content Creation Pipeline

Orchestrator for the content creation flow. Chains sub-skills, does not do content work itself.

```
write (great-writer) -> illustrate (brilliant-visualizer) -> format (typeset) -> deliver (built-in)
```

**Loose coupling contract:** This skill knows sub-skill names and their I/O types (file paths). It does NOT read or depend on any sub-skill's internal files, modes, phases, engines, or themes.

---
```

- [ ] **Step 3: Write the dependency check section**

Add the dependency check that runs on every invocation:

```markdown
## Step 0: Dependency Check

Run this bash block before doing anything else:

\```bash
_MISSING=""
_AVAILABLE=""
for _skill in great-writer brilliant-visualizer typeset; do
  if [ -f ~/.claude/skills/$_skill/SKILL.md ]; then
    _AVAILABLE="$_AVAILABLE $_skill"
    echo "OK: $_skill"
  else
    _MISSING="$_MISSING $_skill"
    echo "MISSING: $_skill"
  fi
done
echo "AVAILABLE:$_AVAILABLE"
echo "MISSING:$_MISSING"
\```

**Interpret the results:**

- If `MISSING` contains `typeset`: STOP. Tell the user:
  "typeset skill is required but not installed. Install it at ~/.claude/skills/typeset/ before using /articulate."
  Do not proceed.

- If `MISSING` contains `great-writer` or `brilliant-visualizer`: warn the user which skills are missing and that the corresponding pipeline stages will be skipped. Offer to install them. Continue with available stages.

- If all three are present: proceed normally.
```

- [ ] **Step 4: Verify frontmatter + dependency check**

Read back the file to verify:
1. YAML frontmatter parses correctly (name, description, triggers all present)
2. Bash dependency check references correct paths (`~/.claude/skills/{name}/SKILL.md`)
3. typeset is identified as the hard dependency
4. great-writer and brilliant-visualizer are soft dependencies

- [ ] **Step 5: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add SKILL.md
git commit -m "feat(crisp-articulator): add frontmatter and dependency check"
```

Note: `~/.claude/skills/` is typically not a git repo. All commit steps in this plan are optional. If there is no git repo, skip them. If the user has a git repo set up for their skills, run the commits.

---

### Task 2: Input Parsing and Entry Point Detection

**Files:**
- Modify: `~/.claude/skills/crisp-articulator/SKILL.md`

- [ ] **Step 1: Write the input parsing section**

Append to SKILL.md after the dependency check:

```markdown
## Step 1: Parse Input and Detect Entry Point

Parse the user's `/articulate` invocation to determine:
1. **What to write about** (topic text or existing file path)
2. **Where to start** (which pipeline stage)
3. **How to run** (gated or auto mode)
4. **Target platform and theme** (for typeset stage)

### Command syntax

```
/articulate <topic_or_file> [--from write|visualize|typeset|deliver] [--auto] [--platform PLATFORM] [--theme THEME]
```

### Argument extraction

From the user's input, extract these parameters:

| Parameter | Source | Default |
|-----------|--------|---------|
| `TOPIC_OR_FILE` | First positional argument | (required) |
| `FROM_STAGE` | `--from` flag value | (auto-detect) |
| `AUTO_MODE` | Presence of `--auto` | false |
| `PLATFORM` | `--platform` flag value | wechat |
| `THEME` | `--theme` flag value | default |

If `--platform` is not specified, ask the user: "Target platform? (wechat / zhihu / juejin / medium / linkedin / x / blog, default: wechat)"

If `--check-deps` is present, run only the dependency check from Step 0 and stop.
```

- [ ] **Step 2: Write the entry point detection logic**

Append:

```markdown
### Entry point detection

If `--from` is specified, use that stage directly. Otherwise, auto-detect:

**Rule 1: Input is a `.md` file path (file exists on disk)**
- Read the file. Check if it contains image references matching `![...](images/...)` or `![...](./images/...)`.
- If image references exist AND at least one referenced image file exists on disk:
  -> Start at **Typeset** (article already has illustrations)
- If no image references, or referenced images don't exist:
  -> Start at **Visualize** (article needs illustrations)

**Rule 2: Input is a `.html` file path (file exists on disk)**
- Start at **Deliver** (just preview and copy)

**Rule 3: Input is plain text (not an existing file path)**
- Start at **Write** (create article from scratch)

**Rule 4: Input is a `.md` file path but file doesn't exist**
- Treat as topic text. Start at **Write**.

After detection, announce: "Starting pipeline from [stage name]. Remaining stages: [list]."

Also check: if the detected start stage requires a sub-skill that is missing (from Step 0), skip forward to the next available stage. Warn the user.
```

- [ ] **Step 3: Verify input parsing section**

Read back the SKILL.md and verify:
1. All five parameters are documented (TOPIC_OR_FILE, FROM_STAGE, AUTO_MODE, PLATFORM, THEME)
2. Auto-detection covers all four rules (.md with images, .md without images, .html, plain text)
3. Fallback behavior for missing files is defined (Rule 4)
4. Missing sub-skill skip-forward is mentioned

- [ ] **Step 4: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add SKILL.md
git commit -m "feat(crisp-articulator): add input parsing and entry point detection"
```

---

### Task 3: Pipeline Stage Definitions

**Files:**
- Modify: `~/.claude/skills/crisp-articulator/SKILL.md`

- [ ] **Step 1: Write the pipeline execution header and Write stage**

Append to SKILL.md:

```markdown
## Step 2: Execute Pipeline

Execute stages in order from the detected entry point. Each stage follows the same pattern:
1. Check if the sub-skill is available (from Step 0 results)
2. Invoke the sub-skill via the Skill tool
3. Collect the output artifact (file path)
4. Run the stage gate (unless `--auto`)
5. Pass the artifact to the next stage

---

### Stage: Write

**Skill:** `great-writer`
**Input:** Topic text from `TOPIC_OR_FILE` + writing type (inferred from topic or platform context)
**Output:** Markdown file path

**Invocation:** Use the Skill tool to invoke `great-writer` with the user's topic. Pass the topic text directly as the skill argument. Example:

```
Skill("great-writer", "写一篇关于 AI Agent 安全的技术博客")
```

Let great-writer handle its own internal pipeline (research, draft, review, humanize, finalize). Do not interfere with or reference its internal phases.

**After completion:** great-writer will produce a Markdown file. Note the file path as `MD_PATH` for the next stage.

**Gate (if not --auto):**
Show the user:
```
Write stage complete.
  File: {MD_PATH}
  Words: ~{word_count}

Continue to illustration? (Y: continue / n: stop here / edit: pause for manual edits)
```

If user says "n", output the current artifact path and stop the pipeline.
If user says "edit", tell the user to edit the file and say "done" when ready, then continue.
If user says "Y" or presses enter, continue to the next stage.
```

- [ ] **Step 2: Write the Visualize stage**

Append:

```markdown
### Stage: Visualize

**Skill:** `brilliant-visualizer`
**Input:** Markdown file path (`MD_PATH` from Write stage or from user input)
**Output:** Markdown file path (same file, now with image references)

**Skip condition:** If `brilliant-visualizer` is not installed (from Step 0), skip this stage. Warn the user: "brilliant-visualizer not installed, skipping illustration stage."

**Invocation:** Use the Skill tool to invoke `brilliant-visualizer` with the markdown file path:

```
Skill("brilliant-visualizer", "{MD_PATH}")
```

Let brilliant-visualizer handle its own analysis, plan confirmation, and generation. Do not interfere with its engine selection or internal workflow.

**After completion:** The Markdown file now contains image references. The file path may be the same as the input. Note the (possibly updated) path as `ILLUSTRATED_MD_PATH`.

**Gate (if not --auto):**
Show the user:
```
Illustration stage complete.
  File: {ILLUSTRATED_MD_PATH}
  Images added: {count} (check the images/ directory)

Continue to typesetting? (Y: continue / n: stop here / edit: pause for manual edits)
```

Same gate behavior as Write stage.
```

- [ ] **Step 3: Write the Typeset stage**

Append:

```markdown
### Stage: Typeset

**Skill:** `typeset`
**Input:** Markdown file path (`ILLUSTRATED_MD_PATH`) + `--platform {PLATFORM}` + `--theme {THEME}`
**Output:** HTML file path

**Invocation:** Use the Skill tool to invoke `typeset` with the markdown path and platform/theme parameters:

```
Skill("typeset", "{ILLUSTRATED_MD_PATH} --platform {PLATFORM} --theme {THEME}")
```

Let typeset handle its own rendering, platform-specific constraints, and theme application. Do not interfere with its CJK handling, link conversion, or HTML generation.

**After completion:** typeset will produce an HTML file. Note the file path as `HTML_PATH`.

No gate after Typeset. Proceed directly to Deliver.
```

- [ ] **Step 4: Verify all three stages**

Read back the SKILL.md and verify:
1. Each stage has: Skill name, Input, Output, Invocation example, Gate behavior
2. Write stage passes topic text, not a file path
3. Visualize stage has a skip condition for missing dependency
4. Typeset stage passes --platform and --theme parameters
5. No stage references any sub-skill internal detail (phases, engines, modes, themes)
6. Variable names are consistent: `MD_PATH` -> `ILLUSTRATED_MD_PATH` -> `HTML_PATH`

- [ ] **Step 5: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add SKILL.md
git commit -m "feat(crisp-articulator): add write, visualize, and typeset pipeline stages"
```

---

### Task 4: Deliver Stage + Summary

**Files:**
- Modify: `~/.claude/skills/crisp-articulator/SKILL.md`

- [ ] **Step 1: Write the Deliver stage**

Append to SKILL.md:

```markdown
### Stage: Deliver (Built-in)

This stage is handled by the orchestrator directly. No sub-skill invocation.

**Input:** HTML file path (`HTML_PATH`)

**Actions:**

1. Copy HTML content to clipboard:
```bash
cat "{HTML_PATH}" | pbcopy
```

2. Open HTML in default browser for preview:
```bash
open "{HTML_PATH}"
```

3. Print pipeline summary:

```
Pipeline complete!

  Source:    {MD_PATH or TOPIC}
  Article:  {ILLUSTRATED_MD_PATH}
  Output:   {HTML_PATH}
  Platform: {PLATFORM}
  Theme:    {THEME}
  Words:    ~{word_count}
  Images:   {image_count}

  HTML copied to clipboard.
  Preview opened in browser.

  Next: Paste into {PLATFORM} editor.
```

Adjust the "Next" hint per platform:
- wechat: "Paste into WeChat Official Account editor."
- zhihu: "Paste into Zhihu article editor."
- juejin: "Paste into Juejin editor."
- medium: "Paste into Medium editor."
- linkedin: "Paste into LinkedIn article editor."
- x: "Paste into X Articles editor."
- blog: "Deploy the HTML file to your blog."
```

- [ ] **Step 2: Verify deliver stage**

Read back and verify:
1. pbcopy command references the correct variable `HTML_PATH`
2. open command references the correct variable `HTML_PATH`
3. Summary includes all key metrics (source, article, output, platform, theme, words, images)
4. Platform-specific hints cover all 7 platforms

- [ ] **Step 3: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add SKILL.md
git commit -m "feat(crisp-articulator): add built-in deliver stage with clipboard and preview"
```

---

### Task 5: Error Handling + Extension Slots

**Files:**
- Modify: `~/.claude/skills/crisp-articulator/SKILL.md`

- [ ] **Step 1: Write the error handling section**

Append to SKILL.md:

```markdown
---

## Error Handling

| Scenario | Action |
|----------|--------|
| Sub-skill invocation fails or produces an error | Show the error. Ask user: "Retry this stage / Skip to next stage / Abort pipeline?" |
| Sub-skill completes but no output file found | Show error: "Expected output file not found after {stage} stage." Abort pipeline. |
| User says "n" at a stage gate | Output the current artifact path. Print partial summary. Stop. |
| `--auto` mode + any error | Abort the entire pipeline. Print error summary with the last successful stage and its artifact. |
| Missing optional sub-skill at runtime | Skip that stage. Warn: "{skill} not installed, skipping {stage} stage." Continue to next. |
| Missing `typeset` at runtime | Abort: "typeset is required. Install it at ~/.claude/skills/typeset/ first." |
| Invalid `--from` value | Show valid options: write, visualize, typeset, deliver. Ask user to correct. |
| Invalid `--platform` value | Show valid options: wechat, zhihu, juejin, medium, linkedin, x, blog. Ask user to correct. |
```

- [ ] **Step 2: Write the extension slots section**

Append:

```markdown
---

## Extension Slots (v2+)

The pipeline has two reserved extension points. They are not active in v1 but documented here for future use.

### Pre-Write Slot (upstream)

Position: Before the Write stage.
Purpose: Topic selection, content strategy, material gathering.
v1 status: Empty. No skill registered.

When a skill is registered here in a future version, the pipeline becomes:
```
[topic-scout] -> write -> illustrate -> format -> deliver
```

### Post-Deliver Slot (downstream)

Position: After the Deliver stage.
Purpose: Direct platform publishing, cross-posting, scheduling.
v1 status: Empty. No skill registered.

When a skill is registered here in a future version, the pipeline becomes:
```
write -> illustrate -> format -> deliver -> [platform-publisher]
```

### How to extend (for future reference)

To register a skill to an extension slot, add a new stage entry in the pipeline section above with the appropriate position. Existing stages do not need modification.
```

- [ ] **Step 3: Verify error handling and extension slots**

Read back and verify:
1. Error table covers all 8 scenarios from the spec
2. Extension slots match spec sections 8.1 and 8.2
3. No placeholder or TBD text remains
4. Extension documentation does not create any actual dependency

- [ ] **Step 4: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add SKILL.md
git commit -m "feat(crisp-articulator): add error handling and extension slot documentation"
```

---

### Task 6: Write README.md

**Files:**
- Create: `~/.claude/skills/crisp-articulator/README.md`

- [ ] **Step 1: Write README.md with usage examples**

```markdown
# Crisp Articulator

Content creation pipeline orchestrator for Claude Code. Chains three skills into a complete write-illustrate-format-deliver flow.

## Quick Start

```bash
# Full pipeline: write + illustrate + typeset + deliver
/articulate "写一篇关于 AI Agent 安全的技术博客" --platform wechat

# From existing markdown (auto-detects what's needed)
/articulate my-article.md --platform zhihu

# Skip to typesetting only
/articulate my-article.md --from typeset --platform wechat --theme tech

# Full auto mode (no confirmation prompts)
/articulate --auto "AI Agent 入门指南" --platform wechat
```

## Pipeline Stages

| Stage | Skill | What It Does |
|-------|-------|-------------|
| Write | great-writer | Creates the article from a topic |
| Visualize | brilliant-visualizer | Adds diagrams and images |
| Typeset | typeset | Formats for target platform |
| Deliver | (built-in) | Copies to clipboard + browser preview |

## Parameters

| Flag | Values | Default |
|------|--------|---------|
| `--platform` | wechat, zhihu, juejin, medium, linkedin, x, blog | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver | auto-detect |
| `--auto` | (no value) | off |
| `--check-deps` | (no value) | - |

## Dependencies

| Skill | Required | Without It |
|-------|----------|-----------|
| typeset | Yes | Cannot run |
| great-writer | No | Cannot start from Write stage |
| brilliant-visualizer | No | Skips illustration stage |

## Sub-Skill Independence

This orchestrator depends only on skill names and file path I/O. Sub-skills can be updated independently without any changes here.
```

- [ ] **Step 2: Verify README.md**

Read back and verify:
1. Quick start examples match the command syntax from the spec
2. Pipeline stages table matches the spec
3. Parameters table covers all flags
4. Dependencies table correctly identifies typeset as hard dependency

- [ ] **Step 3: Commit**

```bash
cd ~/.claude/skills/crisp-articulator
git add README.md
git commit -m "docs(crisp-articulator): add README with usage examples"
```

---

### Task 7: End-to-End Verification

**Files:**
- Read: `~/.claude/skills/crisp-articulator/SKILL.md`
- Read: `~/.claude/skills/crisp-articulator/README.md`

- [ ] **Step 1: Verify SKILL.md completeness against spec**

Read the full SKILL.md and check each spec section is covered:

| Spec Section | SKILL.md Section | Status |
|-------------|-----------------|--------|
| 2.1 Pipeline Descriptor | Stage definitions | Check |
| 2.2 Coupling Contract | Overview + no internal refs | Check |
| 3.1 Auto Detection | Entry point detection rules | Check |
| 3.2 Explicit Override | --from flag | Check |
| 4.1 Stage-Gated Mode | Gate blocks after each stage | Check |
| 4.2 Auto Mode | --auto flag behavior | Check |
| 5 Command Reference | Input parsing section | Check |
| 6 Dependency Management | Step 0 + missing handling | Check |
| 7 Deliver Stage | Built-in deliver section | Check |
| 8 Extension Slots | Extension slots section | Check |
| 9 Error Handling | Error handling table | Check |
| 11 Non-Goals | Coupling contract (no internals) | Check |

- [ ] **Step 2: Verify no sub-skill internal references**

Search the SKILL.md for any of these strings that should NOT appear:
- `modes/` (great-writer internal)
- `engines/` (brilliant-visualizer internal)
- `themes/` (typeset internal)
- `core/` (great-writer internal)
- `Research`, `Draft`, `Review`, `Humanize`, `Finalize` (great-writer phases)
- `mermaid.md`, `architecture.md`, `ai-image.md` (visualizer engine files)

None of these should appear in the orchestrator's instructions (they may appear in the extension slots documentation as examples of what NOT to reference, which is fine).

- [ ] **Step 3: Verify variable name consistency**

Check that variable names are used consistently throughout:
- `TOPIC_OR_FILE` - user input
- `FROM_STAGE` - entry point
- `AUTO_MODE` - auto flag
- `PLATFORM` - target platform
- `THEME` - target theme
- `MD_PATH` - output of Write stage
- `ILLUSTRATED_MD_PATH` - output of Visualize stage
- `HTML_PATH` - output of Typeset stage

- [ ] **Step 4: Final commit (if any fixes were made)**

```bash
cd ~/.claude/skills/crisp-articulator
git add -A
git commit -m "fix(crisp-articulator): verification fixes"
```

- [ ] **Step 5: Print completion summary**

```
Crisp Articulator implementation complete.

Files created:
  ~/.claude/skills/crisp-articulator/SKILL.md    (main skill)
  ~/.claude/skills/crisp-articulator/README.md   (documentation)

Spec coverage: All 12 sections verified.
Sub-skill coupling: Zero internal references confirmed.

To test: Run /articulate --check-deps to verify sub-skill detection.
```
