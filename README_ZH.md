# Crisp Articulator

[English](README.md)

适用于任何 AI Agent 的 skill，串联写作、配图、排版三个环节，一条命令完成从主题到可发布 HTML 的全流程。支持 Claude Code、Codex、Gemini CLI 等所有支持 skill 加载的 agent。

## 做什么用

给它一个主题，它调用 [great-writer](https://github.com/d-wwei/great-writer) 写文章，调用 [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) 配图，调用 [typeset](https://github.com/d-wwei/excellent-typesetter) 排版成目标平台格式。最终 HTML 自动复制到剪贴板，浏览器打开预览。

三个子 skill 各自独立运行。Crisp Articulator 只知道它们的名字和产出的文件路径。子 skill 更新时，编排器不需要任何改动。

## 快速开始

```bash
# 安装：软链接到你的 agent 的 skill 目录
# Claude Code:  ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator
# Codex:        ln -s /path/to/crisp-articulator ~/.agents/skills/crisp-articulator
# Gemini CLI:   ln -s /path/to/crisp-articulator ~/.gemini/skills/crisp-articulator
ln -s /path/to/crisp-articulator <your-agent-skill-dir>/crisp-articulator

# 完整流水线：主题 → 写作 → 配图 → 排版 → 交付
/articulate "写一篇关于 AI Agent 安全的技术博客" --platform wechat

# 已有 Markdown 草稿？从配图开始
/articulate my-article.md --platform zhihu

# 直接跳到排版
/articulate my-article.md --from typeset --platform medium --theme elegant

# 全自动模式，中间不停
/articulate --auto "AI Agent 入门指南" --platform wechat
```

## 流水线

```
主题文本 ──→ 写作 ──→ 配图 ──→ 排版 ──→ 交付
              │        │        │        │
         great-    brilliant-  typeset  剪贴板
         writer    visualizer          + 预览
```

| 阶段 | 子 Skill | 输入 | 输出 |
|------|---------|------|------|
| 写作 | great-writer | 主题文本 | Markdown 文件 |
| 配图 | brilliant-visualizer | Markdown 文件 | 带图 Markdown 文件 |
| 排版 | typeset | Markdown 文件 + 平台参数 | 平台专用 HTML |
| 交付 | (内置) | HTML 文件 | 剪贴板 + 浏览器预览 |

每个阶段之间有检查点，你可以审阅、编辑或停止。加 `--auto` 跳过所有检查点。

## 智能起点检测

不一定从头开始。编排器会根据输入自动判断从哪个阶段启动：

| 你的输入 | 检测到的起点 |
|---------|------------|
| 主题文本 | 写作（完整流水线） |
| `.md` 文件，无配图 | 配图 |
| `.md` 文件，已有配图 | 排版 |
| `.html` 文件 | 交付（仅预览） |

手动指定：`--from write|visualize|typeset|deliver`。

## 参数

| 参数 | 可选值 | 默认值 |
|------|-------|-------|
| `--platform` | wechat, zhihu, juejin, medium, linkedin, x, blog | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver | 自动检测 |
| `--auto` | *(标志)* | 关闭 |
| `--check-deps` | *(标志)* | - |

## 依赖

| Skill | 是否必须 | 缺失时 |
|-------|---------|-------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | 是 | 无法运行 |
| [great-writer](https://github.com/d-wwei/great-writer) | 否 | 跳过写作阶段 |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | 否 | 跳过配图阶段 |

运行 `/articulate --check-deps` 检查安装状态。

### 安装子 skill

```bash
# 必须
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# 可选
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer
```

## 架构：松耦合

编排器对每个子 skill 只依赖三样东西：
1. **Skill 名称**（用于调用）
2. **输入类型**（主题文本或文件路径）
3. **输出类型**（文件路径）

不读取、不引用、不依赖任何子 skill 的内部文件、配置、模式、阶段或引擎。子 skill 可以自由重构、扩展或重写，编排器无需改动。

## 扩展插槽

两个扩展点预留给未来版本：

| 插槽 | 位置 | 用途 |
|------|------|------|
| Pre-Write | 写作阶段之前 | 选题、内容策略 |
| Post-Deliver | 交付阶段之后 | 平台发布、多平台分发 |

v1 中这两个位置为空，仅作为文档化占位。

## 许可证

MIT
