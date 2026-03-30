# Crisp Articulator

[English](README.md)

一条命令，从主题到发布。Crisp Articulator 串联写作、配图、排版、交付和发布，编排独立的子 skill 完成全流程。发布由 `/publish` skill（[superb-publisher](../superb-publisher/)）处理，支持的平台和配置详见该项目。兼容 Claude Code、Codex、Gemini CLI 等所有支持 skill 加载的 agent。

## 快速开始

```bash
# 安装：软链接到你的 agent 的 skill 目录
ln -s /path/to/crisp-articulator ~/.claude/skills/crisp-articulator

# 完整流水线：主题 -> 写作 -> 配图 -> 排版 -> 交付
/articulate "写一篇关于 AI Agent 安全的技术博客" --platform wechat

# 完整流水线 + 通过 /publish skill 发布
/articulate "AI Agent 安全" --platform wechat --publish

# 多平台同时发布
/articulate "AI Agent 安全" --platform wechat,medium,x --publish

# 已有 Markdown 草稿，从配图开始
/articulate my-article.md --platform zhihu

# 直接跳到排版
/articulate my-article.md --from typeset --platform medium --theme elegant

# 全自动模式（发布前仍需确认）
/articulate --auto "AI Agent 入门指南" --platform wechat --publish

# 直接发布现有 HTML 文件
/articulate article.html --from publish --platform wechat
```

## 流水线

```
主题 --> 写作 --> 配图 --> 排版 --> 交付 --> [/publish]
          |        |        |        |          |
     great-    brilliant-  typeset  剪贴板   superb-
     writer    visualizer          + 预览   publisher
```

| 阶段 | 子 Skill / 工具 | 输入 | 输出 |
|------|----------------|------|------|
| 写作 | [great-writer](https://github.com/d-wwei/great-writer) | 主题文本 | Markdown 文件 |
| 配图 | [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | Markdown 文件 | 带图 Markdown |
| 排版 | [typeset](https://github.com/d-wwei/excellent-typesetter) | Markdown + 平台参数 | 平台专用 HTML |
| 交付 | (内置) | HTML 文件 | 剪贴板 + 浏览器预览 |
| 发布 | [superb-publisher](../superb-publisher/)（/publish skill） | HTML + 元数据 | 已发布内容 |

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
| `--platform` | wechat, medium, linkedin, xhs, douyin, x, zhihu, juejin, blog（逗号分隔支持多平台） | wechat |
| `--theme` | default, elegant, tech, minimal, vibrant | default |
| `--from` | write, visualize, typeset, deliver, publish | 自动检测 |
| `--auto` | *(标志)* | 关闭 |
| `--publish` | *(标志)* 启用发布阶段 | 关闭 |
| `--title` | 文章标题 | 自动从 `<h1>` 提取 |
| `--author` | 作者名 | 空 |
| `--digest` | 摘要（最长 120 字） | 自动从首段提取 |
| `--thumb` | 封面图片路径 | 无 |
| `--publish-args` | 传递给 /publish 的额外参数 | 无 |
| `--check-deps` | *(标志)* 检查依赖后退出 | - |

## 依赖

### 子 Skill（流水线阶段）

| Skill | 是否必须 | 缺失时 |
|-------|---------|-------|
| [typeset](https://github.com/d-wwei/excellent-typesetter) | **是** | 无法运行 |
| [great-writer](https://github.com/d-wwei/great-writer) | 否 | 跳过写作阶段 |
| [brilliant-visualizer](https://github.com/d-wwei/brilliant-visualizer) | 否 | 跳过配图阶段 |
| [superb-publisher](../superb-publisher/) | 否 | 发布阶段不可用（流水线在交付阶段结束） |

### 安装

```bash
# 必须的子 skill
ln -s /path/to/excellent-typesetter ~/.claude/skills/typeset

# 可选子 skill
ln -s /path/to/great-writer ~/.claude/skills/great-writer
ln -s /path/to/brilliant-visualizer ~/.claude/skills/brilliant-visualizer
ln -s /path/to/superb-publisher ~/.claude/skills/superb-publisher
```

运行 `/articulate --check-deps` 检查安装状态。

## 架构

### 松耦合

编排器对每个子 skill 只知道三样东西：

1. **名称** -- skill 名称
2. **输入** -- 主题文本或文件路径
3. **输出** -- 文件路径或 ID

仅此而已。不知道内部阶段、引擎配置、API 端点、主题逻辑。所有子 skill 通过 agent 的 Skill 工具调用。任何组件可以独立重构或替换。

### 扩展插槽

| 插槽 | 位置 | 状态 | 用途 |
|------|------|------|------|
| Pre-Write | 写作之前 | 规划中 | 选题、内容策略 |
| Post-Deliver | 交付之后 | **已激活** | 多平台发布（通过 /publish skill） |

发布阶段委托给 `/publish` skill（superb-publisher）。要添加新平台，扩展 superb-publisher 即可——编排器无需修改。

## 许可证

MIT
