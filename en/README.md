# CLAUDE.md Workflow Template · Share Package (English)

> 🌐 **Language**: [🇨🇳 中文](../zh-CN/README.md) | 🇺🇸 English (current) | [🏠 Back to root](../README.md)

> A battle-tested `CLAUDE.md` configuration template that turns Claude Code (or any agent harness supporting CLAUDE.md) into a reliable "subtask-decomposing + batch-safe + long-task-supervised" workstation.

---

## 📦 What's in this package

| File | Purpose |
|------|---------|
| [README.md](README.md) | This file — overview & quick start |
| [CLAUDE_template.md](CLAUDE_template.md) | **Full sanitized template**, copy & customize |
| [customization-guide.md](customization-guide.md) | Section-by-section customization guide, explains why each module is written this way |
| [CHECKLIST.md](CHECKLIST.md) | Pre-flight self-check |

> 🇨🇳 Chinese version: [../zh-CN/README.md](../zh-CN/README.md)

---

## 🎯 What problems does this solve

Three real-world "AI agent gone wrong" scenarios, each with IRON-RULE-grade countermeasures:

### Problem 1: Batch file ops → data loss
**Incident**: Asked AI to batch-rename 36 sequencing files. AI confidently reported "done" after 3 failures, but actually lost 1 pair of files and fabricated "all 36 succeeded" numbers.

**Countermeasure**: `BATCH FILE OPERATION SAFETY PROTOCOL` — forced dry-run, one-by-one execution, byte-count conservation, two-phase relay rename.

### Problem 2: Subagent spam → token waste
**Incident**: AI spawned subagents for any trivial question, never told you which model it used, token bill exploded, model routing unverifiable.

**Countermeasure**: `SUBAGENT PROTOCOL` — explicit "must spawn / never spawn" boundaries, forced 4-candidate model selection, two-layer nesting forced to cheapest model.

### Problem 3: Long task interruption → unrecoverable
**Incident**: 30-min multi-source literature search crashed at min 20 due to context overflow, all progress lost.

**Countermeasure**: `LONG-TASK FOREMAN PROTOCOL` — periodic foreman polling via `deferredResultId`, progress persisted to disk, resumable from `progress.json`.

---

## 🚀 Quick Start (3 steps)

### Step 1 — Copy the template

```bash
# macOS / Linux
cp CLAUDE_template.md ~/.claude/CLAUDE.md

# Windows
copy CLAUDE_template.md %USERPROFILE%\.claude\CLAUDE.md
```

### Step 2 — Global placeholder replacement

Open `CLAUDE_template.md`, search and replace the following (**mandatory**):

| Placeholder | Meaning | Example |
|------|------|------|
| `<USERNAME>` | Your OS username | `alice` |
| `<WORKSPACE_PATH>` | Your main workspace absolute path | `D:\projects\mylab` |
| `<PROJECT_TAG>` | Project codename (for incident references) | `LAB_A` or `PROJECT_X` |
| `<DATA_DIR>` | Your data directory name | `sequencing_output` |
| `<SAMPLE_ID>` | Your sample naming prefix | `S001` or `PT01` |
| `<CHEAPEST_MODEL>` | Cheapest model for nested subagents | `MiniMax-M3` |

### Step 3 — Keep or drop modules

The template contains 3 independent modules marked with `<!-- BEGIN: xxx -->` and `<!-- END: xxx -->`:

| Module | Necessity | Use case |
|------|------|------|
| Batch File Safety | 🔴 Strongly recommended | Any agent touching production data |
| Subagent Protocol | 🟡 Recommended | Claude Code / Cursor / Copilot etc. |
| Long-Task Foreman | 🟢 Optional | Frequent >5min research/scraping tasks |

---

## 🔧 Works with GCMP routing plugin

The template defaults to [GCMP](https://github.com/)-style public LLM routing plugins (which expose GLM / DeepSeek / MiniMax etc. inside Claude Code).

**If you don't use GCMP**:
1. Delete "§2 Fixed 4 candidate models" from `SUBAGENT PROTOCOL`
2. Replace with your actual available models
3. Delete "§6 Verify model routing" GCMP-specific command

**If you use other routers** (one-api / new-api / litellm):
- Replace the 4 candidates with your router's actual models
- Replace "Verify model routing" with your router's usage query interface

---

## 📋 Before going live

Run through [CHECKLIST.md](CHECKLIST.md). Key items:

- [ ] All `<PLACEHOLDER>` replaced
- [ ] Dry-run tested batch safety protocol at least once
- [ ] Subagent model list matches your actual available models
- [ ] `.task_cache/` added to `.gitignore`

---

## 💡 Design philosophy

1. **IRON RULE trumps all** — strongest wording for data-safety rules
2. **Numbers must be verifiable** — no "should be", "about"; every number tested with a real command
3. **Failure must halt** — AI cannot "self-repair" errors; must stop and ask user
4. **Save tokens at the main session** — expensive model decides, cheap model labors

### When NOT to use this config

- Pure chitchat / one-shot Q&A (overhead > benefit)
- Sandbox / test env (no production data → batch safety optional)
- Single-LLM workflow (no subagents → subagent protocol N/A)

---

## 📝 Feedback & improvements

This template is born from real production incidents. Common improvement directions:

- Add your domain's **domain-specific IRON RULEs** (clinical data de-identification, bioinformatics pipelines, financial data audit)
- Replace `<PROJECT_TAG>` incident with your own real incidents (**real incidents are 10× more effective than abstract rules**)
- Adjust subagent model list to your actual subscriptions

---

**Version**: v1.0 · 2026-06  
**License**: MIT
