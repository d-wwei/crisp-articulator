---
name: articulate
description: >
  Content creation pipeline orchestrator for any AI agent. Chains great-writer ->
  brilliant-visualizer -> typeset -> deliver -> publish into a complete write-to-publish flow.
  Supports publishing to 6 platforms: WeChat, Medium, LinkedIn, Xiaohongshu, Douyin, X/Twitter.
  Multi-platform simultaneous publishing via comma-separated --platform values.
  Loose coupling: depends only on skill names and I/O contracts, not sub-skill internals.
  Works with Claude Code, Codex, Gemini CLI, and any agent that supports skill loading.
triggers:
  # Chinese
  - 一键发文
  - 写文章并排版
  - 从头写一篇
  - 写完整文章
  - 全流程写作
  - 内容流水线
  - 多平台发布
  - 跨平台发文
  # English
  - articulate
  - content pipeline
  - write and publish
  - full article
  - end to end article
  - cross post
  - multi platform publish
---

# Crisp Articulator -- Content Creation Pipeline

Orchestrator for the content creation flow. Chains sub-skills, does not do content work itself.

```
write (great-writer) -> illustrate (brilliant-visualizer) -> format (typeset) -> deliver (built-in) -> [publish (platform CLIs)]
```

Brackets indicate optional/conditional stages. The Publish stage requires `--publish` flag. Supports 6 platforms: WeChat, Medium, LinkedIn, Xiaohongshu (XHS), Douyin, X/Twitter. Multi-platform publishing supported via comma-separated values: `--platform wechat,medium,x`.

**Loose coupling contract:** This skill knows sub-skill names and their I/O types (file paths). It does NOT read or depend on any sub-skill's internal files, modes, phases, engines, or themes. For external CLI tools (wechat-cli, medium-cli, linkedin-cli, xhs-cli, douyin-cli, x-cli), the same contract applies: the orchestrator knows command names and I/O, not internal implementation.

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

Then check for external CLI tools (used by the publish stage):

```bash
# External tool check (for publish stage)
# Platform CLI registry: command name -> config check
_PLATFORM_CLIS="wechat-cli medium-cli linkedin-cli xhs-cli douyin-cli x-cli"
_PUBLISH_AVAILABLE=""
_PUBLISH_MISSING=""

for _cli in $_PLATFORM_CLIS; do
  if command -v $_cli >/dev/null 2>&1; then
    echo "OK: $_cli"
    _PUBLISH_AVAILABLE="$_PUBLISH_AVAILABLE $_cli"
    _AVAILABLE="$_AVAILABLE $_cli"
    # Check config for CLIs that require it
    case $_cli in
      wechat-cli)
        [ -f "$HOME/.config/wechat-cli/config" ] && echo "  config: ready" || echo "  config: MISSING (run: wechat-cli init)" ;;
      medium-cli)
        [ -f "$HOME/.config/medium-cli/config" ] && echo "  config: ready" || echo "  config: MISSING (run: medium-cli init)" ;;
      linkedin-cli)
        [ -f "$HOME/.config/linkedin-cli/config" ] && echo "  config: ready" || echo "  config: MISSING (run: linkedin-cli init && linkedin-cli login)" ;;
      xhs-cli|douyin-cli|x-cli)
        [ -d "$HOME/.config/social-cli" ] && echo "  config: ready (cookie-based)" || echo "  config: MISSING (run: $_cli login)" ;;
    esac
  else
    echo "MISSING: $_cli (optional — needed for --publish --platform ...)"
    _PUBLISH_MISSING="$_PUBLISH_MISSING $_cli"
    _MISSING="$_MISSING $_cli"
  fi
done
echo "PUBLISH_AVAILABLE:$_PUBLISH_AVAILABLE"
echo "PUBLISH_MISSING:$_PUBLISH_MISSING"
```

**Interpret the results:**

- If `MISSING` contains `typeset`: STOP. Tell the user:
  "typeset skill is required but not installed. Install it to your agent's skill directory before using /articulate."
  Do not proceed.

- If `MISSING` contains `great-writer` or `brilliant-visualizer`: warn the user which skills are missing and that the corresponding pipeline stages will be skipped. Offer to install them. Continue with available stages.

- If all three are present: proceed normally.

- For each platform requested via `--platform`, check if the corresponding CLI is available:

| Platform | Required CLI | Config location |
|----------|-------------|-----------------|
| wechat | wechat-cli | `~/.config/wechat-cli/config` |
| medium | medium-cli | `~/.config/medium-cli/config` |
| linkedin | linkedin-cli | `~/.config/linkedin-cli/config` |
| xhs | xhs-cli | `~/.config/social-cli/xhs/cookies.json` |
| douyin | douyin-cli | `~/.config/social-cli/douyin/cookies.json` |
| x | x-cli | `~/.config/social-cli/x/cookies.json` |

- If a requested platform's CLI is missing AND `--publish` was requested: warn: "{cli} not installed. Publishing to {platform} will be skipped." Continue with other platforms (if multi-platform) or end at Deliver (if single platform).

- If a CLI is present but config/auth is missing: warn: "{cli} found but not configured. Run `{cli} init` (or `{cli} login` for browser-based CLIs) before using --publish --platform {platform}."

---

## Step 1: Parse Input and Detect Entry Point

Parse the user's `/articulate` invocation to determine:
1. **What to write about** (topic text or existing file path)
2. **Where to start** (which pipeline stage)
3. **How to run** (gated or auto mode)
4. **Target platform and theme** (for typeset stage)

### Command syntax

```
/articulate <topic_or_file> [--from write|visualize|typeset|deliver|publish] [--auto] [--platform wechat|medium|linkedin|xhs|douyin|x|zhihu|juejin|blog] [--theme default|elegant|tech|minimal|vibrant] [--publish] [--title T] [--author A] [--digest D] [--thumb F] [--tags "a,b,c"] [--status draft|public] [--video F] [--cover F] [--check-deps]
```

Multi-platform: `--platform wechat,medium,x` (comma-separated, publishes to each in sequence).

### Argument extraction

From the user's input, extract these parameters:

| Parameter | Source | Default |
|-----------|--------|---------|
| `TOPIC_OR_FILE` | First positional argument | (required) |
| `FROM_STAGE` | `--from` flag value | (auto-detect) |
| `AUTO_MODE` | Presence of `--auto` | false |
| `PLATFORM` | `--platform` flag value (can be comma-separated) | wechat |
| `THEME` | `--theme` flag value | default |
| `PUBLISH` | Presence of `--publish` | false |
| `TITLE` | `--title` flag value | (auto-extract from HTML/Markdown) |
| `AUTHOR` | `--author` flag value | (empty) |
| `DIGEST` | `--digest` flag value | (auto-extract from first paragraph) |
| `THUMB_FILE` | `--thumb` flag value | (none) |
| `TAGS` | `--tags` flag value (comma-separated) | (none) |
| `STATUS` | `--status` flag value | draft (for medium) |
| `VIDEO_FILE` | `--video` flag value | (none, required for douyin) |
| `COVER_FILE` | `--cover` flag value | (none) |

If `--platform` is not specified, ask the user: "Target platform? (wechat / medium / linkedin / xhs / douyin / x / zhihu / juejin / blog, default: wechat)"

If `--platform` contains commas, split into `PLATFORM_LIST` array. Each platform will be processed sequentially in the Publish stage.

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
- medium (without `--publish`): "Cmd+V to paste into Medium editor. Or re-run with --publish to publish directly via API."
- medium (with `--publish`): "Continuing to publish stage..."
- linkedin (without `--publish`): "Cmd+V to paste into LinkedIn article editor. Or re-run with --publish to publish directly via API."
- linkedin (with `--publish`): "Continuing to publish stage..."
- xhs (without `--publish`): "Manually post to Xiaohongshu. Or re-run with --publish to publish via browser automation."
- xhs (with `--publish`): "Continuing to publish stage..."
- douyin (without `--publish`): "Manually upload to Douyin. Or re-run with --publish to publish via browser automation."
- douyin (with `--publish`): "Continuing to publish stage..."
- x (without `--publish`): "Cmd+V to paste into X. Or re-run with --publish to publish via browser automation."
- x (with `--publish`): "Continuing to publish stage..."
- zhihu: "Cmd+V to paste into Zhihu article editor."
- juejin: "Cmd+V to paste into Juejin editor."
- blog: "Deploy the HTML file to your blog."
- Multi-platform (with `--publish`): "Continuing to publish to {N} platforms..."

---

### Stage: Publish (Post-Deliver) -- Multi-Platform Router

**Tools:** Platform-specific CLIs (bash CLIs, invoked via Bash tool -- not via Skill tool)
**Input:** HTML file path (`HTML_PATH`) + Markdown path (`ILLUSTRATED_MD_PATH`, if available) + metadata (`TITLE`, `AUTHOR`, `DIGEST`, etc.)
**Output:** Platform-specific IDs (printed to console, not file artifacts)

**Supported platforms and their CLIs:**

| Platform | CLI | Install location | Auth method |
|----------|-----|-----------------|-------------|
| wechat | wechat-cli | `~/.local/bin/wechat-cli` | AppID + AppSecret config |
| medium | medium-cli | `~/.local/bin/medium-cli` | Integration Token config |
| linkedin | linkedin-cli | `~/.local/bin/linkedin-cli` | OAuth 2.0 browser login |
| xhs | xhs-cli | `social-cli/bin/xhs-cli` | QR code browser login |
| douyin | douyin-cli | `social-cli/bin/douyin-cli` | QR code browser login |
| x | x-cli | `social-cli/bin/x-cli` | Browser login |

**Skip conditions (check in order):**
- `PUBLISH` is false (no `--publish` flag): skip entirely. Pipeline ends at Deliver.
- For each platform in `PLATFORM_LIST`: if the corresponding CLI is not installed (from Step 0), warn and skip that platform. If ALL platforms are skipped, print: "No publish CLI available for the requested platform(s). Pipeline ends at Deliver."
- If a CLI is present but config/auth is missing: warn and skip that platform.

**Mid-pipeline entry:** If `--from publish`, set `HTML_PATH = TOPIC_OR_FILE`. Validate it is an existing `.html` file.

**Multi-platform execution:** If `PLATFORM_LIST` has multiple entries, process each platform sequentially. Each platform gets its own confirmation gate. A failure on one platform does not block the others.

```
Publishing to {N} platforms: {platform_list}
Processing platform 1/{N}: {platform_name}...
```

**Sub-step 0: Extract metadata (shared across all platforms)**

- `TITLE`: If `--title` not given, extract from the first `<h1>` tag in the HTML. If no `<h1>`, extract from the Markdown source's first `# ` heading (using `ILLUSTRATED_MD_PATH` if available). If neither found, ask the user: "Article title?"
- `DIGEST`: If `--digest` not given, extract the text content of the first `<p>` tag in the HTML, truncated to 120 characters. If not found, leave empty.
- `AUTHOR`: If `--author` not given, leave empty.

**Sub-step 1: Content Adaptation**

Different platforms require different content formats. Before publishing to each platform, adapt the content:

| Platform | Input format | Adaptation rules |
|----------|-------------|-----------------|
| wechat | HTML | Use `HTML_PATH` directly. typeset output is WeChat-optimized. |
| medium | HTML or Markdown | Use `HTML_PATH` directly. Medium API accepts HTML. |
| linkedin | Plain text + optional images | Extract a short summary (max ~1300 chars) from the article. Use the article's first 2-3 paragraphs as text content. Extract image paths from the Markdown for `--images`. |
| xhs | Text + images | Title: max 20 Chinese characters (truncate if needed). Body: max 1000 Chinese characters, plain text extracted from HTML/Markdown. Images: extracted from `images/` directory. |
| douyin | Video + description | Requires `VIDEO_FILE` (from `--video` flag). Text becomes the description (max 500 chars). Cover from `--cover` or first image. If no video file, skip with warning. |
| x | Short text + optional images | Max 280 characters for single tweet. For long articles, generate a summary tweet with a link or create a thread. Images: max 4, extracted from `images/` directory. |

**Content extraction helpers (used by adaptation):**

To extract plain text from HTML:
```bash
# Strip HTML tags, collapse whitespace
PLAIN_TEXT=$(/usr/bin/python3 -c "
import re, sys
with open('{HTML_PATH}', 'r') as f:
    html = f.read()
text = re.sub(r'<[^>]+>', '', html)
text = re.sub(r'\s+', ' ', text).strip()
print(text[:{MAX_CHARS}])
")
```

To extract images from the Markdown source:
```bash
IMAGES=$(grep -oP '!\[.*?\]\(\K[^)]+' "{ILLUSTRATED_MD_PATH}" 2>/dev/null | head -4)
```

---

#### Platform: WeChat (`--platform wechat`)

**CLI:** `wechat-cli`

**Pre-publish check:**
```bash
wechat-cli check "{HTML_PATH}"
```
- Exit code 1 (errors): show output, ask: "Fix issues and retry / Skip wechat / Abort?"
- Exit code 0 (warnings only): show warnings, continue.

**Upload local images to WeChat CDN:**
```bash
wechat-cli upload-images "{HTML_PATH}"
```
This modifies `HTML_PATH` in-place, replacing local image `src` with WeChat CDN URLs.

**Upload cover image (if provided):**
```bash
THUMB_ID=$(wechat-cli upload-thumb "{THUMB_FILE}")
```
If `THUMB_FILE` is not set, `THUMB_ID` is empty.

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
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

**Create draft + Publish:**
```bash
MEDIA_ID=$(wechat-cli draft --title "{TITLE}" --html "{HTML_PATH}" --thumb "{THUMB_ID}" --author "{AUTHOR}" --digest "{DIGEST}")
PUBLISH_ID=$(wechat-cli publish "{MEDIA_ID}")
```

**Success output:**
```
Published to WeChat!
  Title:      {TITLE}
  Media ID:   {MEDIA_ID}
  Publish ID: {PUBLISH_ID}
  Status:     Submitted (review may take a few minutes)
  Check:      wechat-cli status {PUBLISH_ID}
```

**Error handling:**
- `draft` fails: show WeChat API error (wechat-cli translates error codes to Chinese). Ask: "Retry / Abort?"
- `publish` fails: show error + `MEDIA_ID` for manual retry: `wechat-cli publish {MEDIA_ID}`.

---

#### Platform: Medium (`--platform medium`)

**CLI:** `medium-cli`

**Content preparation:** Use `HTML_PATH` directly. Medium's API accepts HTML and handles rendering. If images use local paths, they must be hosted externally first (Medium re-hosts from URLs).

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
```
Ready to publish to Medium:

  Title:   {TITLE}
  Tags:    {TAGS or "none"}
  Status:  {STATUS or "draft"}
  Format:  HTML
  File:    {HTML_PATH}

  Publish now? (Y: publish / n: cancel / edit: edit title/tags/status)
```

**Publish:**
```bash
ARTICLE_URL=$(medium-cli publish --title "{TITLE}" --html "{HTML_PATH}" --tags "{TAGS}" --status "{STATUS}")
```

`STATUS` defaults to `draft` if not specified via `--status`. Valid values: `draft`, `public`, `unlisted`.

**Success output:**
```
Published to Medium!
  Title:  {TITLE}
  URL:    {ARTICLE_URL}
  Status: {STATUS}
  Tags:   {TAGS}
```

**Error handling:**
- HTTP 401: "Medium token invalid or expired. Run `medium-cli init` to update your token."
- HTTP 429: "Rate limited by Medium. Wait a few minutes and retry."
- Other errors: show the error. Ask: "Retry / Skip medium / Abort?"

---

#### Platform: LinkedIn (`--platform linkedin`)

**CLI:** `linkedin-cli`

**Content adaptation:** LinkedIn posts are plain text with optional images. Extract a concise summary from the article:

1. Extract the first 2-3 paragraphs of plain text from HTML (max ~1300 characters).
2. Optionally prefix with the article title as a bold-style header.
3. Collect image paths from the Markdown source (max 4 images).

```bash
# Extract plain text summary
POST_TEXT=$(/usr/bin/python3 -c "
import re
with open('{HTML_PATH}', 'r') as f:
    html = f.read()
text = re.sub(r'<[^>]+>', '', html)
text = re.sub(r'\s+', ' ', text).strip()
print(text[:1300])
")
```

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
```
Ready to publish to LinkedIn:

  Text:    {POST_TEXT[:100]}...
  Images:  {image_list or "none"}
  Length:  {char_count} chars

  Publish now? (Y: publish / n: cancel / edit: edit text)
```

**Publish (text post):**
```bash
POST_ID=$(linkedin-cli publish --text "{POST_TEXT}")
```

**Publish (with images):**
```bash
POST_ID=$(linkedin-cli publish --text "{POST_TEXT}" --images {IMAGE_1} {IMAGE_2})
```

**Publish (article share, if an external URL is available):**
```bash
POST_ID=$(linkedin-cli publish --article --title "{TITLE}" --text "{POST_TEXT}" --url "{ARTICLE_URL}" --thumbnail {COVER_IMAGE})
```

Use the article share format when the content has been published to another platform first (e.g., Medium URL available from a prior publish step in multi-platform mode).

**Success output:**
```
Published to LinkedIn!
  Post ID:  {POST_ID}
  Type:     {text post | article share}
  Images:   {count}
```

**Error handling:**
- HTTP 401: "LinkedIn token expired. Run `linkedin-cli login` to re-authorize."
- HTTP 403: "Missing permissions. Check LinkedIn Developer Portal for w_member_social scope."
- Other errors: show the error. Ask: "Retry / Skip linkedin / Abort?"

---

#### Platform: Xiaohongshu (`--platform xhs`)

**CLI:** `xhs-cli` (from social-cli monorepo)

**Content adaptation:** Xiaohongshu requires a short title, body text, and images.

1. Title: max 20 Chinese characters. If `TITLE` exceeds 20 chars, truncate and append "...".
2. Body: max 1000 Chinese characters. Extract plain text from HTML, strip all formatting.
3. Images: required (at least 1). Collect from `images/` directory or Markdown image references. Max 9 images.
4. Tags: optional, from `--tags` flag.

```bash
# Prepare XHS content
XHS_TITLE=$(echo "{TITLE}" | cut -c1-20)
XHS_CONTENT=$(/usr/bin/python3 -c "
import re
with open('{HTML_PATH}', 'r') as f:
    html = f.read()
text = re.sub(r'<[^>]+>', '', html)
text = re.sub(r'\s+', ' ', text).strip()
print(text[:1000])
")
```

**Pre-publish check:**
```bash
xhs-cli check || xhs-cli login
```
If not logged in, `xhs-cli login` opens a browser for QR code scanning.

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
```
Ready to publish to Xiaohongshu:

  Title:   {XHS_TITLE}
  Content: {XHS_CONTENT[:80]}...
  Images:  {image_list}
  Tags:    {TAGS or "none"}

  Publish now? (Y: publish / n: cancel / edit: edit title/content)
```

**Publish:**
```bash
xhs-cli publish --title "{XHS_TITLE}" --content "{XHS_CONTENT}" --images {IMG_1} {IMG_2} ... --tags "{TAGS}"
```

Or with HTML input (xhs-cli extracts text internally):
```bash
xhs-cli publish --title "{XHS_TITLE}" --html "{HTML_PATH}" --images {IMG_1} {IMG_2} ...
```

**Success output:**
```
Published to Xiaohongshu!
  Title:   {XHS_TITLE}
  Images:  {count}
  Tags:    {TAGS}
```

**Error handling:**
- Login required: "Not logged in to Xiaohongshu. Running `xhs-cli login`..." (opens browser for QR code).
- Upload failure: show error. Ask: "Retry / Skip xhs / Abort?"

---

#### Platform: Douyin (`--platform douyin`)

**CLI:** `douyin-cli` (from social-cli monorepo)

**Content adaptation:** Douyin is a video-first platform. Publishing requires a video file.

1. Video: from `--video` flag (`VIDEO_FILE`). **Required.** If not provided, skip with warning: "Douyin requires a video file. Use `--video clip.mp4` to specify one. Skipping douyin."
2. Cover: from `--cover` flag (`COVER_FILE`), or first image from the article.
3. Title/Description: max 500 characters. Use `TITLE` as the video title and `DIGEST` or first paragraph as description.
4. Tags: optional, from `--tags` flag.

**Pre-publish check:**
```bash
douyin-cli check || douyin-cli login
```

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
```
Ready to publish to Douyin:

  Title:   {TITLE}
  Video:   {VIDEO_FILE}
  Cover:   {COVER_FILE or "auto"}
  Tags:    {TAGS or "none"}
  Desc:    {DESCRIPTION[:80]}...

  Publish now? (Y: publish / n: cancel / edit: edit title/description)
```

**Publish:**
```bash
douyin-cli publish --title "{TITLE}" --video "{VIDEO_FILE}" --cover "{COVER_FILE}" --tags "{TAGS}"
```

**Success output:**
```
Published to Douyin!
  Title:  {TITLE}
  Video:  {VIDEO_FILE}
```

**Error handling:**
- No video file: skip platform with warning (not an error for the pipeline).
- Login required: "Not logged in to Douyin. Running `douyin-cli login`..." (opens browser for QR code).
- Upload failure: show error. Ask: "Retry / Skip douyin / Abort?"

---

#### Platform: X/Twitter (`--platform x`)

**CLI:** `x-cli` (from social-cli monorepo)

**Content adaptation:** X has a 280-character limit per tweet. Two modes:

**Mode A: Single tweet (default)**
Generate a concise summary of the article (max 280 chars) + optional images (max 4).
```bash
TWEET_TEXT=$(/usr/bin/python3 -c "
import re
with open('{HTML_PATH}', 'r') as f:
    html = f.read()
text = re.sub(r'<[^>]+>', '', html)
text = re.sub(r'\s+', ' ', text).strip()
# Take first 250 chars + leave room for hashtags
print(text[:250])
")
```

**Mode B: Thread (for long articles)**
If the article is > 1000 words, suggest creating a thread. Split content into tweet-sized chunks (max 280 chars each), with numbering (1/N, 2/N...).

The orchestrator defaults to Mode A. If the user requests a thread (`--thread` or explicitly asks), use Mode B.

**Pre-publish check:**
```bash
x-cli check || x-cli login
```

**Confirmation gate (MANDATORY -- not skipped by `--auto`):**
```
Ready to publish to X:

  Text:    {TWEET_TEXT[:100]}...
  Images:  {image_list or "none"}
  Mode:    {single tweet | thread (N tweets)}
  Length:  {char_count} chars

  Publish now? (Y: publish / n: cancel / edit: edit text)
```

**Publish (single tweet):**
```bash
x-cli publish --text "{TWEET_TEXT}" --images {IMG_1} {IMG_2}
```

**Publish (thread):**
```bash
# Each tweet in sequence
x-cli publish --text "{TWEET_1}" --images {IMG_1}
x-cli publish --text "{TWEET_2}" --reply-to {PREV_TWEET_ID}
...
```

**Success output:**
```
Published to X!
  Text:    {TWEET_TEXT[:60]}...
  Images:  {count}
  Mode:    {single tweet | thread}
```

**Error handling:**
- Login required: "Not logged in to X. Running `x-cli login`..." (opens browser).
- Rate limit: "Rate limited by X. Wait and retry."
- Other errors: show the error. Ask: "Retry / Skip x / Abort?"

---

#### Multi-Platform Summary

After all platforms in `PLATFORM_LIST` have been processed, print a consolidated summary:

```
Multi-platform publish complete!

  Platform   Status     Details
  ─────────  ─────────  ──────────────────────
  wechat     Success    Publish ID: {PUBLISH_ID}
  medium     Success    URL: {ARTICLE_URL}
  x          Skipped    x-cli not installed

  Total: {success_count}/{total_count} platforms published.
  Failed/Skipped: {list or "none"}
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
| Invalid `--platform` value | Show valid options: wechat, medium, linkedin, xhs, douyin, x, zhihu, juejin, blog. Ask user to correct. |
| `--publish` but CLI missing for requested platform | Warn: "{cli} not installed. Skipping {platform}." Continue to next platform (multi) or end at Deliver (single). |
| `--publish` but CLI unconfigured for platform | Warn: "{cli} not configured. Run `{cli} init` or `{cli} login`." Skip that platform. |
| Platform CLI command fails | Show error. Ask: "Retry / Skip {platform} / Abort?" In multi-platform mode, failure on one platform does not block others. |
| `--platform douyin` without `--video` | Warn: "Douyin requires a video file (--video). Skipping douyin." |
| Multi-platform: some succeed, some fail | Print consolidated summary showing each platform's status. Do not abort successful publishes due to later failures. |
| Browser-based CLI login required (xhs/douyin/x) | Attempt auto-login via `{cli} login`. If login fails, warn and skip that platform. |

---

## Extension Slots (v2+)

The pipeline has two reserved extension points.

### Pre-Write Slot (upstream)

Position: Before the Write stage.
Purpose: Topic selection, content strategy, material gathering.
Status: Empty. No skill registered.

When a skill is registered here in a future version, the pipeline becomes:
```
[topic-scout] -> write -> illustrate -> format -> deliver -> [publish]
```

### Post-Deliver Slot (downstream)

Position: After the Deliver stage.
Purpose: Direct platform publishing, cross-posting, scheduling.
Status: **Active.** 6 platform CLIs registered with platform-gated activation.

Current pipeline:
```
write -> illustrate -> format -> deliver -> [publish (platform router)]
```

Registered platform CLIs:

| Platform | CLI | Type | Status |
|----------|-----|------|--------|
| WeChat | wechat-cli | API-based (shell + curl) | Active |
| Medium | medium-cli | API-based (shell + curl) | Active |
| LinkedIn | linkedin-cli | API-based (shell + curl + OAuth) | Active |
| Xiaohongshu | xhs-cli | Browser automation (Puppeteer) | Active |
| Douyin | douyin-cli | Browser automation (Puppeteer) | Active |
| X/Twitter | x-cli | Browser automation (Puppeteer) | Active |

### How to extend

To register a new platform to the publish stage:
1. Create a CLI following the same contract: `{platform}-cli publish [--options]` with structured output.
2. Add a dependency check in Step 0 (use `command -v` for CLI tools).
3. Add a platform entry in the publish stage's platform router with:
   - Content adaptation rules
   - CLI command pattern
   - Confirmation gate
   - Success/error output format
4. Add the platform to the `--platform` valid values list.
5. Existing stages and other platform handlers do not need modification.
