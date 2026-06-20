# Agent Harness Playbook

> A battle-tested `CLAUDE.md` configuration template that turns Claude Code (or any agent harness that supports CLAUDE.md) into a reliable "subtask-decomposing + batch-safe + long-task-supervised" workstation.

---

## 🌐 Languages / 语言

- 🇨🇳 **中文版** → [zh-CN/README.md](zh-CN/README.md)
- 🇺🇸 **English** → [en/README.md](en/README.md)

---

## 📦 What's in this repo

| File (zh-CN)                                              | File (en)                                           | Purpose                                             |
| --------------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------- |
| [zh-CN/README.md](zh-CN/README.md)                           | [en/README.md](en/README.md)                           | Overview & quick start                              |
| [zh-CN/CLAUDE_template.md](zh-CN/CLAUDE_template.md)         | [en/CLAUDE_template.md](en/CLAUDE_template.md)         | **Full sanitized template**, copy & customize |
| [zh-CN/customization-guide.md](zh-CN/customization-guide.md) | [en/customization-guide.md](en/customization-guide.md) | Section-by-section customization guide              |
| [zh-CN/CHECKLIST.md](zh-CN/CHECKLIST.md)                     | [en/CHECKLIST.md](en/CHECKLIST.md)                     | Pre-flight self-check                               |

---

## 🎯 What problems does this solve

Three classes of "AI agent gone wrong" with IRON-RULE-grade countermeasures:

### 1. Batch file ops → data loss

**Real-world incident**: Asked AI to batch-rename sequencing files. AI reported "done" confidently after 3 failures, but actually lost one pair of files and fabricated the count.

**Countermeasure**: `BATCH FILE OPERATION SAFETY PROTOCOL` — forced dry-run, one-by-one execution, byte-count conservation check, two-phase relay rename.

### 2. Subagent spam → token waste

**Real-world incident**: AI spawned subagents for any trivial question, never told you which model it used, token bill exploded.

**Countermeasure**: `SUBAGENT PROTOCOL` — explicit "must spawn / never spawn" boundaries, forced 4-candidate model selection, two-layer nesting forced to cheapest model.

### 3. Long task interruption → unrecoverable

**Real-world incident**: 30-min multi-source literature search crashed at min 20, all progress lost.

**Countermeasure**: `LONG-TASK FOREMAN PROTOCOL` — periodic foreman polling via `deferredResultId`, progress persisted to disk, resumable from `progress.json`.

---

## 🚀 Quick Start (3 steps)

### Step 1 — Copy the template

```bash
# macOS / Linux
cp en/CLAUDE_template.md ~/.claude/CLAUDE.md

# Windows
copy en\CLAUDE_template.md %USERPROFILE%\.claude\CLAUDE.md
```

### Step 2 — Replace placeholders

Search & replace these placeholders in `CLAUDE_template.md` (**mandatory**):

| Placeholder          | Meaning                                  | Example               |
| -------------------- | ---------------------------------------- | --------------------- |
| `<USERNAME>`       | Your OS username                         | `alice`             |
| `<WORKSPACE_PATH>` | Your workspace path                      | `D:\projects\mylab` |
| `<PROJECT_TAG>`    | Project codename (referenced in lessons) | `LAB_A`             |
| `<DATA_DIR>`       | Your data directory name                 | `sequencing_output` |
| `<SAMPLE_ID>`      | Your sample naming prefix                | `S001`              |
| `<CHEAPEST_MODEL>` | Cheapest model for nested subagents      | `MiniMax-M3`        |

### Step 3 — Keep or drop modules

The template contains 3 independent modules marked with `<!-- BEGIN: xxx -->` / `<!-- END: xxx -->`:

| Module            | Necessity               | Use case                               |
| ----------------- | ----------------------- | -------------------------------------- |
| Batch File Safety | 🔴 Strongly recommended | Any agent touching production data     |
| Subagent Protocol | 🟡 Recommended          | Claude Code / Cursor / Copilot etc.    |
| Long-Task Foreman | 🟢 Optional             | Frequent >5min research/scraping tasks |

---

## 🔧 Works with GCMP / one-api / litellm / etc.

Template defaults to [GCMP](https://github.com/) (a public LLM routing plugin that exposes GLM / DeepSeek / MiniMax in Claude Code).

**Using a different router?**

1. Replace the 4 candidate models in `Subagent Protocol §2` with your actual models
2. Replace the "verify model routing" section with your router's usage query

---

## 📋 Before going live

Run through [en/CHECKLIST.md](en/CHECKLIST.md). Key items:

- [ ] All `<PLACEHOLDER>` replaced
- [ ] Dry-run tested batch safety protocol at least once
- [ ] Subagent model list matches your actual available models
- [ ] `.task_cache/` and `.research_cache/` added to `.gitignore`

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

## 📝 Feedback & contributions

This template is born from real production incidents. PRs welcome — especially:

- Domain-specific IRON RULEs (clinical data, bioinformatics, finance, etc.)
- Translations to other languages
- Adaptations to other harnesses (Aider, Cursor, Continue.dev, etc.)

---

**Version**: v1.0 · 2026-06
**License**: [MIT](LICENSE)
