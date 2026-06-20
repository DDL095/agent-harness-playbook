# CLAUDE.md 工作流配置模板 · 分享包

> 🌐 **语言 / Language**: 🇨🇳 中文（当前）| [🇺🇸 English](../en/README.md) | [🏠 返回总入口](../README.md)

> 一套经过实战打磨的 `CLAUDE.md` 配置模板，帮助你把 Claude Code（或任意支持 CLAUDE.md 的 agent harness）调教成可靠的"子任务拆分 + 批量安全 + 长任务监理"工作机。

---

## 📦 这个包里有什么

| 文件 | 用途 |
|------|------|
| [README.md](README.md) | 本文件，总览与使用指引 |
| [CLAUDE_template.md](CLAUDE_template.md) | **完整脱敏模板**，可直接复制改用 |
| [customization-guide.md](customization-guide.md) | 逐节定制指南，解释每个模块为什么这么写、如何改 |
| [CHECKLIST.md](CHECKLIST.md) | 上线前的自检清单 |

---

## 🎯 这套配置解决什么问题

实战中遇到的 3 类"AI agent 失控"场景，本配置都给出了 IRON RULE 级别的硬性约束：

### 问题 1：批量文件操作 → 数据损坏
**典型事故**：让 AI 批量重命名 36 个测序数据文件，AI 自信地报告"已完成"，实际丢了 1 对文件，且编造了"36 个全部成功"的数字。

**对策**：`BATCH FILE OPERATION SAFETY PROTOCOL` — 强制 dry-run、逐个执行、字节数守恒验证、两阶段中转重命名。

### 问题 2：subagent 乱拉、token 浪费
**典型事故**：AI 遇到任何小问题都拉 subagent，或者拉了 subagent 不告诉用户用了哪个模型，导致 token 消耗失控、模型路由不可验证。

**对策**：`SUBAGENT 调度协议` — 明确"必须拉 / 禁止拉"的边界、强制 4 候选模型询问、两层嵌套强制用最省 token 的模型。

### 问题 3：长任务中途断线、无法恢复
**典型事故**：让 AI 跑 30 分钟的多源文献检索，跑到 20 分钟时上下文窗口爆了，前面所有进度归零。

**对策**：`长任务监理协议` — 用 deferredResultId 机制让主会话周期性轮询监理 SA，所有子任务进度落盘，断线后可从 progress.json 恢复。

---

## 🚀 快速开始（3 步）

### Step 1：复制模板

```bash
cp CLAUDE_template.md ~/.claude/CLAUDE.md   # macOS / Linux
copy CLAUDE_template.md %USERPROFILE%\.claude\CLAUDE.md   # Windows
```

### Step 2：全局替换占位符

打开 `CLAUDE_template.md`，搜索并替换以下占位符（**这一步必做**，否则配置无法生效）：

| 占位符 | 含义 | 示例 |
|--------|------|------|
| `<USERNAME>` | 你的系统用户名 | `alice` |
| `<WORKSPACE_PATH>` | 你的主工作区绝对路径 | `D:\projects\mylab` |
| `<PROJECT_TAG>` | 项目代号（用于教训引用） | `LAB_A` 或 `PROJECT_X` |
| `<DATA_DIR>` | 你的数据目录名 | `sequencing_output` |
| `<SAMPLE_ID>` | 你的样本命名前缀 | `S001` 或 `PT01` |

### Step 3：按需保留/删除模块

模板里包含 3 个相对独立的模块，可按需裁剪：

| 模块 | 必要性 | 适用场景 |
|------|--------|----------|
| **批量文件安全协议** | 🔴 强烈推荐 | 任何会接触生产数据的 agent |
| **subagent 调度协议** | 🟡 推荐 | 使用 Claude Code / Cursor / Copilot 等支持 subagent 的工具 |
| **长任务监理协议** | 🟢 可选 | 经常跑 >5 分钟的研究/检索任务 |

每个模块顶部都有 `<!-- BEGIN: xxx -->` 和 `<!-- END: xxx -->` 标记，方便整段删除。

---

## 🔧 与 GCMP 路由插件配合

模板默认使用 [GCMP](https://github.com/) 这类公开的 LLM 路由插件（支持在 Claude Code 中切换 GLM、DeepSeek、MiniMax 等国产模型）。

**如果你不用 GCMP**：
1. 删除 `SUBAGENT 调度协议` 的"第二节 固定 4 个候选模型"
2. 改为你的实际可用模型清单
3. 删除"第六节 验证模型路由"中关于 GCMP 命令面板的描述

**如果你用其他路由方案**（如 one-api、new-api、litellm）：
- 把 4 候选模型替换为你的路由后端实际暴露的模型
- "验证模型路由"一节改为查询你的路由后端的 usage 接口

---

## 📋 上线前必做

跑一遍 [CHECKLIST.md](CHECKLIST.md)，确认配置正确。关键点：

- [ ] 所有 `<PLACEHOLDER>` 已替换
- [ ] 至少跑过一次 dry-run 测试批量安全协议
- [ ] subagent 模型清单与你的实际可用模型一致
- [ ] 长任务监理协议中的 `.task_cache/` 目录已加入 `.gitignore`

---

## 💡 使用心得

### 这套配置的"哲学"

1. **IRON RULE 高于一切** — 涉及数据安全的规则用最强硬的措辞，宁可错杀不可放过
2. **数字必须可验证** — 禁止 AI 用"应该"、"大概"等措辞，所有数字必须用命令实测
3. **失败必须立即停止** — 不允许 AI"自行修复"错误，必须停下来问用户
4. **省 token 的关键在主会话** — 把决策留给主会话（贵模型），把重复劳动留给 subagent（便宜模型）

### 何时**不要**用这套配置

- **纯对话场景**（闲聊、单次问答）—— 协议开销大于收益
- **沙盒/测试环境** —— 不涉及生产数据，批量安全协议可省
- **单 LLM 工作流** —— 没有 subagent 概念，调度协议不适用

---

## 📝 反馈与改进

本模板源于真实使用教训，欢迎基于你的实际场景修订。常见改进方向：

- 加入你所在领域的**领域特定 IRON RULE**（如临床数据脱敏、生信分析流程、财务数据审计）
- 把 `<PROJECT_TAG>` 教训替换成你自己踩过的坑（**真实教训比抽象规则有效 10 倍**）
- 调整 subagent 模型清单为你的实际订阅

---

**版本**：v1.0 · 2026-06  
**许可证**：随意使用、修改、分享
