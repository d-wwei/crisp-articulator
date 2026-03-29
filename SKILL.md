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

# Crisp Articulator -- Content Creation Pipeline

Orchestrator for the content creation flow. Chains sub-skills, does not do content work itself.

```
write (great-writer) -> illustrate (brilliant-visualizer) -> format (typeset) -> deliver (built-in)
```

**Loose coupling contract:** This skill knows sub-skill names and their I/O types (file paths). It does NOT read or depend on any sub-skill's internal files, modes, phases, engines, or themes.

---

## Step 0: Dependency Check

Run this bash block before doing anything else:

```bash
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
```

**Interpret the results:**

- If `MISSING` contains `typeset`: STOP. Tell the user:
  "typeset skill is required but not installed. Install it at ~/.claude/skills/typeset/ before using /articulate."
  Do not proceed.

- If `MISSING` contains `great-writer` or `brilliant-visualizer`: warn the user which skills are missing and that the corresponding pipeline stages will be skipped. Offer to install them. Continue with available stages.

- If all three are present: proceed normally.

---

## Step 1: Parse Input and Detect Entry Point

Parse the user's `/articulate` invocation to determine:
1. **What to write about** (topic text or existing file path)
2. **Where to start** (which pipeline stage)
3. **How to run** (gated or auto mode)
4. **Target platform and theme** (for typeset stage)

### Command syntax

```
/articulate <topic_or_file> [--from write|visualize|typeset|deliver] [--auto] [--platform wechat|zhihu|juejin|medium|linkedin|x|blog] [--theme default|elegant|tech|minimal|vibrant] [--check-deps]
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

---

## Step 2: Execute Pipeline

Execute stages in order from the detected entry point. Each stage follows the same pattern:
1. Check if the sub-skill is available (from Step 0 results)
2. Invoke the sub-skill via the Skill tool
3. Collect the output artifact (file path)
4. Run the stage gate (unless `AUTO_MODE` is true)
5. Pass the artifact to the next stage

---

### Stage: Write

**Skill:** `great-writer`
**Input:** Topic text from `TOPIC_OR_FILE` + writing type (inferred from topic or platform context)
**Output:** Markdown file path

**Invocation:** Use the Skill tool to invoke `great-writer`. Pass the topic text as the `args` parameter. Example:

```
Skill(skill="great-writer", args="写一篇关于 AI Agent 安全的技术博客")
```

Let great-writer handle its own internal pipeline. Do not interfere with or reference its internal phases.

**After completion:** great-writer will produce a Markdown file. Note the file path as `MD_PATH` for the next stage.

**Gate (if not AUTO_MODE):**
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

---

### Stage: Visualize

**Skill:** `brilliant-visualizer`
**Input:** Markdown file path (`MD_PATH` from Write stage or from user input)
**Output:** Markdown file path (same file, now with image references)

**Skip condition:** If `brilliant-visualizer` is not installed (from Step 0), skip this stage. Warn the user: "brilliant-visualizer not installed, skipping illustration stage." Set `ILLUSTRATED_MD_PATH` = `MD_PATH` and continue to the next stage.

**Mid-pipeline entry:** If the pipeline starts at this stage via `--from visualize` or auto-detection, set `MD_PATH` = `TOPIC_OR_FILE` (the user-provided file path).

**Invocation:** Use the Skill tool to invoke `brilliant-visualizer` with the markdown file path:

```
Skill(skill="brilliant-visualizer", args="{MD_PATH}")
```

Let brilliant-visualizer handle its own analysis, plan confirmation, and generation. Do not interfere with its engine selection or internal workflow.

**After completion:** The Markdown file now contains image references. The file path may be the same as the input. Note the (possibly updated) path as `ILLUSTRATED_MD_PATH`.

**Gate (if not AUTO_MODE):**
Show the user:
```
Illustration stage complete.
  File: {ILLUSTRATED_MD_PATH}
  Images added: {count} (check the images/ directory)

Continue to typesetting? (Y: continue / n: stop here / edit: pause for manual edits)
```

Same gate behavior as Write stage.

---

### Stage: Typeset

**Skill:** `typeset`
**Input:** Markdown file path (`ILLUSTRATED_MD_PATH`) + `--platform {PLATFORM}` + `--theme {THEME}`
**Output:** HTML file path

**Mid-pipeline entry:** If the pipeline starts at this stage via `--from typeset` or auto-detection, set `ILLUSTRATED_MD_PATH` = `TOPIC_OR_FILE` (the user-provided file path). If `TOPIC_OR_FILE` is not a valid file path, show an error: "Typeset stage requires a Markdown file path as input."

**Invocation:** Use the Skill tool to invoke `typeset` with the markdown path and platform/theme parameters:

```
Skill(skill="typeset", args="{ILLUSTRATED_MD_PATH} --platform {PLATFORM} --theme {THEME}")
```

Let typeset handle its own rendering, platform-specific constraints, and theme application. Do not interfere with its internal logic.

**After completion:** typeset will produce an HTML file. Note the file path as `HTML_PATH`.

No gate after Typeset. Proceed directly to Deliver.

---

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

  Source:    {TOPIC_OR_FILE}
  Article:  {ILLUSTRATED_MD_PATH}
  Output:   {HTML_PATH}
  Platform: {PLATFORM}
  Theme:    {THEME}
  Words:    ~{word_count}
  Images:   {image_count}

  HTML copied to clipboard.
  Preview opened in browser.

  Next: {platform_hint}
```

Adjust the "Next" hint per platform:
- wechat: "Paste into WeChat Official Account editor."
- zhihu: "Paste into Zhihu article editor."
- juejin: "Paste into Juejin editor."
- medium: "Paste into Medium editor."
- linkedin: "Paste into LinkedIn article editor."
- x: "Paste into X Articles editor."
- blog: "Deploy the HTML file to your blog."

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Sub-skill invocation fails or produces an error | Show the error. Ask user: "Retry this stage / Skip to next stage / Abort pipeline?" |
| Sub-skill completes but no output file found | Show error: "Expected output file not found after {stage} stage." Abort pipeline. |
| User says "n" at a stage gate | Output the current artifact path. Print partial summary. Stop. |
| `AUTO_MODE` + any error | Abort the entire pipeline. Print error summary with the last successful stage and its artifact. |
| Missing optional sub-skill at runtime | Skip that stage. Warn: "{skill} not installed, skipping {stage} stage." Continue to next. |
| Missing `typeset` at runtime | Abort: "typeset is required. Install it at ~/.claude/skills/typeset/ first." |
| Invalid `--from` value | Show valid options: write, visualize, typeset, deliver. Ask user to correct. |
| Invalid `--platform` value | Show valid options: wechat, zhihu, juejin, medium, linkedin, x, blog. Ask user to correct. |

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
