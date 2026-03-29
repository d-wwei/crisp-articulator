# Crisp Articulator

[English](README.md)

一条命令，从主题到发布。Crisp Articulator 串联写作、配图、排版、交付和可选的微信公众号发布，编排独立的子 skill 完成全流程。支持 Claude Code、Codex、Gemini CLI 等所有支持 skill 加载的 agent。

## 快速开始

```bash
# 安装：软链接到你的 agent 的 skill 目录
ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator

# 完整流水线：主题 → 写作 → 配图 → 排版 → 交付
/articulate "写一篇关于 AI Agent 安全的技术博客" --platform wechat

# 完整流水线 + 直接发布到微信公众号
/articulate "AI Agent 安全" --platform wechat --publish --thumb cover.png

# 已有 Markdown 草稿，从配图开始
/articulate my-article.md --platform zhihu

# 直接跳到排版
/articulate my-article.md --from typeset --platform medium --theme elegant

# 全自动模式（发布前仍需确认）
/articulate --auto "AI Agent 入门指南" --platform wechat --publish

# 直接发布现有 HTML 文件
/articulate article.html --from publish --title "我的文章"
```

## 流水线

```
主题 ──→ 写作 ──→ 配图 ──→ 排版 ──→ 交付 ──→ [发布]
          │        │        │        │          │
     great-    brilliant-  typeset  剪贴板   wechat-cli
     writer    visualizer          + 预览    (可选)
```

| 阶段 | 子 Skill / 工具 | 输入 | 输出 |
|------|----------------|------|------|
| 写作 | [great-writer](https://github.com/d-wwei/great-writer) | 主题文本 | Markdown 文件 |
| 配图 | [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | Markdown 文件 | 带图 Markdown |
| 排版 | [typeset](https://github.com/d-wwei/excellent-typesetter) | Markdown + 平台参数 | 平台专用 HTML |
| 交付 | (内置) | HTML 文件 | 剪贴板 + 浏览器预览 |
| 发布 | [wechat-cli](https://github.com/d-wwei/wechat-cli) | HTML + 元数据 | 已发布文章 |

每个阶段之间有检查点，可以审阅、编辑或停止。`--auto` 跳过所有检查点——发布阶段除外，发布前必须确认（不可逆操作）。

## 智能起点检测

编排器根据输入自动判断从哪个阶段开始：

| 输入 | 起点 |
|------|------|
| 主题文本 | 写作（完整流水线） |
| `.md` 文件，无配图 | 配图 |
| `.md` 文件，已有配图 | 排版 |
| `.html` 文件 | 交付 |

手动指定：`--from write|visualize|typeset|deliver|publish`。

## 参数

| 参数 | 可选值 | 默认值 |
|------|-------|-------|
| `--platform` | wechat, zhihu, juejin, medium, linkedin, x, blog | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver, publish | 自动检测 |
| `--auto` | *(标志)* | 关闭 |
| `--publish` | *(标志)* 启用微信公众号发布 | 关闭 |
| `--title` | 微信草稿标题 | 自动从 `<h1>` 提取 |
| `--author` | 作者名 | 空 |
| `--digest` | 摘要（最长 120 字） | 自动从首段提取 |
| `--thumb` | 封面图片路径 | 无 |
| `--check-deps` | *(标志)* 检查依赖后退出 | - |

## 依赖

| Skill / 工具 | 是否必须 | 缺失时 |
|-------------|---------|-------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | **是** | 无法运行 |
| [great-writer](https://github.com/d-wwei/great-writer) | 否 | 跳过写作阶段 |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | 否 | 跳过配图阶段 |
| [wechat-cli](https://github.com/d-wwei/wechat-cli) | 否 | 跳过发布阶段 |

### 安装

```bash
# 必须
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# 可选子 skill
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer

# 可选 CLI 工具（用于 --publish）
ln -s /path/to/wechat-cli ~/.local/bin/wechat-cli
wechat-cli init   # 填入 AppID 和 AppSecret
```

运行 `/articulate --check-deps` 检查安装状态。

## 架构

### 松耦合

编排器对每个子 skill 或工具只知道三样东西：

1. **名称** — skill 名称或 CLI 命令
2. **输入** — 主题文本或文件路径
3. **输出** — 文件路径或 ID

仅此而已。不知道内部阶段、引擎配置、API 端点、主题逻辑。子 skill 通过 agent 的 Skill 工具调用，CLI 工具（wechat-cli）通过 Bash 调用。机制不同，契约相同。任何组件可以独立重构或替换。

### 扩展插槽

| 插槽 | 位置 | 状态 | 用途 |
|------|------|------|------|
| Pre-Write | 写作之前 | 规划中 | 选题、内容策略 |
| Post-Deliver | 交付之后 | **已激活** | 平台发布 |

发布阶段（wechat-cli）占据 Post-Deliver 插槽。未来可注册更多平台 CLI 到同一插槽。

## 许可证

MIT
