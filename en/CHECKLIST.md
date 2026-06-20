# CLAUDE.md Pre-flight Checklist (English)

> After configuring CLAUDE.md, check each item. **Any unchecked item should be fixed before going live.**

---

## 🔍 Step 1: Placeholder replacement (mandatory)

All `<PLACEHOLDER>` in the template must be replaced with your actual values.

- [ ] `<USERNAME>` replaced (e.g. `alice`)
- [ ] `<WORKSPACE_PATH>` replaced (e.g. `D:\projects\mylab`)
- [ ] `<PROJECT_TAG>` replaced (e.g. `LAB_A`)
- [ ] `<DATA_DIR>` replaced (e.g. `sequencing_output`)
- [ ] `<SAMPLE_ID>` replaced (e.g. `S001`)
- [ ] `<CHEAPEST_MODEL>` replaced (e.g. `MiniMax-M3 (TokenPlan) (gcmp.minimax)`)
- [ ] Full-text search for `<` shows no remaining placeholders

```bash
# macOS / Linux
grep -n '<[A-Z_]*>' ~/.claude/CLAUDE.md

# Windows PowerShell
Select-String -Path $env:USERPROFILE\.claude\CLAUDE.md -Pattern '<[A-Z_]+>'
```

---

## 🔍 Step 2: Module trimming (as needed)

Template contains 4 modules; confirm which to keep:

- [ ] **AGENT-SKILLS-BRIDGE**: do you have a local skill library? If not, delete
- [ ] **BATCH-FILE-SAFETY**: do you handle production data? Strongly recommend keep
- [ ] **SUBAGENT-PROTOCOL**: do you use a subagent-capable harness? If not, delete
- [ ] **LONG-TASK-FOREMAN**: do you run long tasks? If not, delete

Each module is wrapped with `<!-- BEGIN: xxx -->` and `<!-- END: xxx -->` for easy whole-block deletion.

---

## 🔍 Step 3: Model list verification (if keeping SUBAGENT-PROTOCOL)

- [ ] 4 candidate models are **actually and reliably callable** (don't write "theoretically works")
- [ ] The 4 have division of labor (not all same tier)
- [ ] `<CHEAPEST_MODEL>` is the **cheapest / longest-context** of the 4
- [ ] Model routing verification method (GCMP command / one-api log / etc.) is documented

**Test method**: have AI spawn a subagent. Check:
1. Did it list 4 candidates?
2. Did it ask which you choose?
3. Did it actually use your choice (verify via router usage)?

---

## 🔍 Step 4: Incident authenticity (if keeping BATCH-FILE-SAFETY)

The template's "source incident" is an **example**; strongly recommend replacing with your own real incident:

- [ ] Replaced with real incident (specific time, specific dir, specific loss)
- [ ] If no real incident, write at least a **potential risk you worry about**
- [ ] Incident **must not contain personal privacy** (collaborator names, unpublished project codenames, sensitive sample IDs)

**Why important**: real incidents are 10× more enforceable than abstract rules. AI internalizes rules only when seeing concrete consequences.

---

## 🔍 Step 5: Path verification

- [ ] All path separators correct (Windows `\` / Unix `/`)
- [ ] Username path uses placeholder (`C:\Users\<USERNAME>\`) or your actual username
- [ ] Workspace path points to a **real existing** directory
- [ ] Log path's parent (e.g. `.audit/`) is writable

---

## 🔍 Step 6: `.gitignore` (if keeping LONG-TASK-FOREMAN)

Exclude task cache directories to avoid committing large intermediate state:

```gitignore
# CLAUDE.md long-task cache
.task_cache/
.research_cache/
.audit/
```

- [ ] Added above entries to `.gitignore`
- [ ] Confirmed these dirs are **not** already tracked (if yes, `git rm --cached` first)

---

## 🧪 Step 7: Stress tests (strongly recommended)

Per "stress tests" in [customization-guide.md](customization-guide.md):

### Stress test 1: Batch safety
- [ ] Have AI process 20 test files
- [ ] Observe: dry-run first? Stops on failure? Doesn't self-repair?

### Stress test 2: Subagent
- [ ] Have AI handle an independent small research
- [ ] Observe: spawns subagent? Asks model first? Uses one of 4 candidates?

### Stress test 3: Long-task foreman (if applicable)
- [ ] Open a >5min task
- [ ] Observe: spawns foreman? Writes progress.json? Resumable mid-way?

---

## 🔒 Step 8: Privacy check (if sharing)

If you plan to share CLAUDE.md with others:

- [ ] **Personal / collaborator name initials** removed (e.g. `ZYH` → `<PROJECT_TAG>`)
- [ ] **Unpublished research topics** removed (e.g. "aging mice" → `<RESEARCH_TOPIC>`)
- [ ] **Specific sample IDs** removed (e.g. `Adu116` → `<SAMPLE_ID>`)
- [ ] **OS username** uses placeholder (`C:\Users\Administrator\` → `C:\Users\<USERNAME>\`)
- [ ] **Proprietary internal tool names** sanitized (unless confirmed public)
- [ ] **Personal OneDrive / BaiduYunDrive paths** replaced with `<WORKSPACE_PATH>`
- [ ] **API keys / tokens / passwords** (CLAUDE.md shouldn't contain any, but double-check)
- [ ] **Company internal IPs / domains** removed

**Final test**: send the file to a colleague who doesn't know you. Can they infer your real identity, research topic, or collaborators? If yes, keep sanitizing.

---

## ✅ All passed?

Congratulations, your CLAUDE.md is ready. Recommendations:

1. **Version control**: put CLAUDE.md in git (after sanitizing) to track changes
2. **Monthly review**: run stress tests monthly to verify protocols still work
3. **Incident-driven updates**: after each AI mistake, add the lesson to CLAUDE.md

---

## 🆘 Troubleshooting

| Symptom | Possible cause | Solution |
|------|------|------|
| AI completely ignores CLAUDE.md | Wrong path / filename (case-sensitive) | Confirm at `~/.claude/CLAUDE.md` (Unix) or `%USERPROFILE%\.claude\CLAUDE.md` (Win) |
| Some protocols work, some don't | Failing section too long / weak wording | Trim to <50 lines / use stronger ❌ and IRON RULE tags |
| AI spawns subagent but doesn't ask model | Wrong prompt tool name | Replace `vscode_askQuestions` with your harness's actual tool |
| AI uses wrong model | Router misconfigured / model name typo | Verify model routing (see Subagent Protocol §6) |
| Long task can't resume | deferredResultId unsupported / progress.json malformed | Enable downgrade / validate JSON format |
