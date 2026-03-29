---
name: articulate
description: >
  Content creation pipeline orchestrator for any AI agent. Chains great-writer ->
  brilliant-visualizer -> typeset -> deliver -> publish into a complete write-to-publish flow.
  Optional WeChat publishing via wechat-cli (--publish). Loose coupling: depends only on
  skill names and I/O contracts, not sub-skill internals. Works with Claude Code, Codex,
  Gemini CLI, and any agent that supports skill loading.
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
write (great-writer) -> illustrate (brilliant-visualizer) -> format (typeset) -> deliver (built-in) -> [publish (wechat-cli)]
```

Brackets indicate optional/conditional stages. The Publish stage requires `--publish` flag AND `--platform wechat`.

**Loose coupling contract:** This skill knows sub-skill names and their I/O types (file paths). It does NOT read or depend on any sub-skill's internal files, modes, phases, engines, or themes. For external CLI tools (like wechat-cli), the same contract applies: the orchestrator knows command names and I/O, not internal implementation.

---

## Step 0: Dependency Check

Before executing, verify that the required sub-skills are available. The skill directory varies by agent platform:

| Agent | Typical skill directory |
|-------|----------------------|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` |
| Other | Check agent documentation |

Run this bash block (adjust `SKILL_DIR` to match your agent):

```bash
SKILL_DIR="${SKILL_DIR:-$HOME/.claude/skills}"
_MISSING=""
_AVAILABLE=""
for _skill in great-writer brilliant-visualizer typeset; do
  if [ -f "$SKILL_DIR/$_skill/SKILL.md" ]; then
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

Then check for external CLI tools (used by optional pipeline stages):

```bash
# External tool check (for publish stage)
if command -v wechat-cli >/dev/null 2>&1; then
  echo "OK: wechat-cli"
  _AVAILABLE="$_AVAILABLE wechat-cli"
  if [ -f "$HOME/.config/wechat-cli/config" ]; then
    echo "  config: ready"
  else
    echo "  config: MISSING (run: wechat-cli init)"
  fi
else
  echo "MISSING: wechat-cli (optional — needed for --publish)"
  _MISSING="$_MISSING wechat-cli"
fi
```

**Interpret the results:**

- If `MISSING` contains `typeset`: STOP. Tell the user:
  "typeset skill is required but not installed. Install it to your agent's skill directory before using /articulate."
  Do not proceed.

- If `MISSING` contains `great-writer` or `brilliant-visualizer`: warn the user which skills are missing and that the corresponding pipeline stages will be skipped. Offer to install them. Continue with available stages.

- If all three are present: proceed normally.

- If `MISSING` contains `wechat-cli` AND `--publish` was requested: warn: "wechat-cli not installed. Publish stage will be skipped. Pipeline ends at Deliver." Continue without the Publish stage.

- If `wechat-cli` is present but config is missing: warn: "wechat-cli found but not configured. Run `wechat-cli init` and fill in your AppID/AppSecret before using --publish."

---

## Step 1: Parse Input and Detect Entry Point

Parse the user's `/articulate` invocation to determine:
1. **What to write about** (topic text or existing file path)
2. **Where to start** (which pipeline stage)
3. **How to run** (gated or auto mode)
4. **Target platform and theme** (for typeset stage)

### Command syntax

```
/articulate <topic_or_file> [--from write|visualize|typeset|deliver|publish] [--auto] [--platform wechat|zhihu|juejin|medium|linkedin|x|blog] [--theme default|elegant|tech|minimal|vibrant] [--publish] [--title T] [--author A] [--digest D] [--thumb F] [--check-deps]
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
| `PUBLISH` | Presence of `--publish` | false |
| `TITLE` | `--title` flag value | (auto-extract from HTML/Markdown) |
| `AUTHOR` | `--author` flag value | (empty) |
| `DIGEST` | `--digest` flag value | (auto-extract from first paragraph) |
| `THUMB_FILE` | `--thumb` flag value | (none) |

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

**Rule 5: `--from publish` specified**
- Input must be an existing `.html` file. Set `HTML_PATH = TOPIC_OR_FILE`.
- If input is not a valid `.html` file, show error: "Publish stage requires an HTML file path as input."
- Implies `--publish` (no need to set both).

After detection, announce: "Starting pipeline from [stage name]. Remaining stages: [list]."

Also check: if the detected start stage requires a sub-skill that is missing (from Step 0), skip forward to the next available stage. Warn the user.

---

## Step 2: Execute Pipeline

Execute stages in order from the detected entry point. Each stage follows the same pattern:
1. Check if the sub-skill is available (from Step 0 results)
2. Invoke the sub-skill using your agent's skill invocation mechanism
3. Collect the output artifact (file path)
4. Run the stage gate (unless `AUTO_MODE` is true)
5. Pass the artifact to the next stage

---

### Stage: Write

**Skill:** `great-writer`
**Input:** Topic text from `TOPIC_OR_FILE` + writing type (inferred from topic or platform context)
**Output:** Markdown file path

**Invocation:** Invoke the `great-writer` skill with the user's topic as argument. Pass the topic text directly. Example argument:

```
"写一篇关于 AI Agent 安全的技术博客"
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

**Invocation:** Invoke the `brilliant-visualizer` skill with the markdown file path as argument:

```
"{MD_PATH}"
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

**Invocation:** Invoke the `typeset` skill with the markdown path and platform/theme parameters as argument:

```
"{ILLUSTRATED_MD_PATH} --platform {PLATFORM} --theme {THEME}"
```

Let typeset handle its own rendering, platform-specific constraints, and theme application. Do not interfere with its internal logic.

**After completion:** typeset will produce an HTML file. Note the file path as `HTML_PATH`.

No gate after Typeset. Proceed directly to Deliver.

---

### Stage: Deliver (Built-in)

This stage is handled by the orchestrator directly. No sub-skill invocation.

**Input:** HTML file path (`HTML_PATH`)

**Actions:**

1. Copy rendered rich text to clipboard (NOT raw HTML source):

On macOS, use `NSPasteboard` to set clipboard content as `public.html` type. This allows rich text editors (WeChat, Zhihu, etc.) to receive rendered content instead of raw source code.

```bash
/usr/bin/python3 -c "
import AppKit
with open('{HTML_PATH}', 'r') as f:
    html = f.read()
pb = AppKit.NSPasteboard.generalPasteboard()
pb.clearContents()
pb.setData_forType_(html.encode('utf-8'), 'public.html')
print('Rich text copied to clipboard')
"
```

**Why not `cat | pbcopy`?** `pbcopy` copies as plain text (`public.utf8-plain-text`). When pasted into a rich text editor, it shows raw HTML tags instead of rendered content. Using `public.html` clipboard type tells the editor to interpret and render the HTML.

**Fallback** (if Python/AppKit unavailable): Open the HTML in a browser, then tell the user to Cmd+A → Cmd+C from the browser preview.

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

  Rich text copied to clipboard — paste directly into editor.
  Preview opened in browser.

  Next: {platform_hint}
```

Adjust the "Next" hint per platform:
- wechat (without `--publish`): "Cmd+V to paste into WeChat Official Account editor. Or re-run with --publish to publish directly via API."
- wechat (with `--publish`): "Continuing to publish stage..."
- zhihu: "Cmd+V to paste into Zhihu article editor."
- juejin: "Cmd+V to paste into Juejin editor."
- medium: "Cmd+V to paste into Medium editor."
- linkedin: "Cmd+V to paste into LinkedIn article editor."
- x: "Cmd+V to paste into X Articles editor."
- blog: "Deploy the HTML file to your blog."

---

### Stage: Publish (Post-Deliver)

**Tool:** `wechat-cli` (bash CLI, invoked via Bash tool — not via Skill tool)
**Input:** HTML file path (`HTML_PATH`) + metadata (`TITLE`, `AUTHOR`, `DIGEST`, `THUMB_FILE`)
**Output:** media_id, publish_id (printed to console, not file artifacts)

**Skip conditions (check in order):**
- `PUBLISH` is false (no `--publish` flag): skip entirely. Pipeline ends at Deliver.
- `PLATFORM` is not `wechat`: skip. Print: "Publish stage currently supports wechat only. Skipping."
- `wechat-cli` is not installed (from Step 0): skip. Print: "wechat-cli not found. Install it to enable direct WeChat publishing."
- `wechat-cli` config is missing: skip. Print: "wechat-cli not configured. Run `wechat-cli init` first."

If all skip conditions pass, proceed with publishing.

**Mid-pipeline entry:** If `--from publish`, set `HTML_PATH = TOPIC_OR_FILE`. Validate it is an existing `.html` file.

**Sub-step 1: Extract metadata (if not provided via flags)**

- `TITLE`: If `--title` not given, extract from the first `<h1>` tag in the HTML. If no `<h1>`, extract from the Markdown source's first `# ` heading (using `ILLUSTRATED_MD_PATH` if available). If neither found, ask the user: "Article title for WeChat draft?"
- `DIGEST`: If `--digest` not given, extract the text content of the first `<p>` tag in the HTML, truncated to 120 characters. If not found, leave empty.
- `AUTHOR`: If `--author` not given, leave empty (WeChat allows this).

**Sub-step 2: Pre-publish check**

```bash
wechat-cli check "{HTML_PATH}"
```

- If check returns errors (exit code 1): show the output. Ask: "Fix issues and retry / Skip publish / Abort?"
- If check returns warnings only (exit code 0): show warnings and continue.

**Sub-step 3: Upload local images to WeChat CDN**

```bash
wechat-cli upload-images "{HTML_PATH}"
```

This modifies `HTML_PATH` in-place, replacing local image `src` attributes with WeChat CDN URLs. After this step, the HTML is WeChat-ready.

**Sub-step 4: Upload cover image (if provided)**

If `THUMB_FILE` is set:
```bash
THUMB_ID=$(wechat-cli upload-thumb "{THUMB_FILE}")
```

If `THUMB_FILE` is not set, `THUMB_ID` is empty. The draft will have no cover image.

**Confirmation gate (MANDATORY — not skipped by `--auto`):**

Publishing sends content to WeChat's API and is irreversible. This gate always fires.

```
Ready to publish to WeChat Official Account:

  Title:   {TITLE}
  Author:  {AUTHOR or "(empty)"}
  Digest:  {DIGEST or "(auto-generated)"} (first 120 chars)
  Cover:   {THUMB_FILE or "none"}
  HTML:    {HTML_PATH}
  Images:  {count} uploaded to CDN

  Publish now? (Y: publish / n: cancel / edit: edit title/author/digest)
```

- **Y**: proceed to create draft and publish.
- **n**: print "Publish cancelled. HTML is ready at {HTML_PATH}." and stop.
- **edit**: allow user to change title, author, or digest, then re-show the gate.

**Sub-step 5: Create draft**

```bash
MEDIA_ID=$(wechat-cli draft --title "{TITLE}" --html "{HTML_PATH}" --thumb "{THUMB_ID}" --author "{AUTHOR}" --digest "{DIGEST}")
```

Note the `MEDIA_ID` output. If this fails, show the WeChat API error (wechat-cli translates error codes to Chinese) and ask: "Retry / Abort?" The draft can be retried safely.

**Sub-step 6: Publish draft**

```bash
PUBLISH_ID=$(wechat-cli publish "{MEDIA_ID}")
```

If this fails, show the error and the `MEDIA_ID` so the user can retry manually: `wechat-cli publish {MEDIA_ID}`.

**After completion:** Print publish summary:

```
Published to WeChat!

  Title:      {TITLE}
  Media ID:   {MEDIA_ID}
  Publish ID: {PUBLISH_ID}
  Status:     Submitted (WeChat review may take a few minutes)

  Check status: wechat-cli status {PUBLISH_ID}
  Article will be visible to followers after approval.
```

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
| Invalid `--from` value | Show valid options: write, visualize, typeset, deliver, publish. Ask user to correct. |
| Invalid `--platform` value | Show valid options: wechat, zhihu, juejin, medium, linkedin, x, blog. Ask user to correct. |
| `wechat-cli check` fails (errors) | Show check output. Ask: "Fix and retry / Skip publish / Abort?" |
| `wechat-cli upload-images` fails | Show error. Ask: "Retry / Skip publish / Abort?" |
| `wechat-cli draft` fails | Show WeChat API error (wechat-cli translates error codes). Ask: "Retry / Abort?" |
| `wechat-cli publish` fails | Show error + media_id for manual retry. Ask: "Retry / Abort?" |
| `--publish` but platform is not wechat | Warn: "Publish stage only supports wechat. Skipping." End at Deliver. |
| `--publish` but wechat-cli missing or unconfigured | Warn and skip. End at Deliver. |

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
Status: **Active.** `wechat-cli` registered for WeChat platform (`--publish` + `--platform wechat`).

Current pipeline:
```
write -> illustrate -> format -> deliver -> [publish (wechat-cli)]
```

Future: additional platform CLIs (e.g., zhihu-cli) can be registered to the same slot with their own platform gates.

### How to extend

To register a new tool to an extension slot:
1. Add a stage entry in the pipeline section above with the appropriate position.
2. Add a dependency check in Step 0 (use `command -v` for CLI tools, file check for skills).
3. Add skip conditions at the top of the new stage.
4. Existing stages do not need modification.

The Publish stage (wechat-cli) demonstrates this pattern: an external CLI tool registered to the Post-Deliver slot, with platform-gated activation and a mandatory confirmation gate.
