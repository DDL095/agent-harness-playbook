<!--
  CLAUDE.md Template v1.0 (English)
  ──────────────────────────────────────────────────────────────
  Usage:
  1. Global-replace all <PLACEHOLDER> placeholders (mandatory)
  2. Keep or drop the three major modules (use BEGIN/END markers)
  3. Replace the <PROJECT_TAG> incident in "Batch Safety Protocol" with your own real incident
  4. Run CHECKLIST.md before first use
───────────────────────────────────────────────────────────────────
-->

<!-- BEGIN: AGENT-SKILLS-BRIDGE -->
# Agent Skills Library (optional: if you have a local skill library)

You have access to a local AI agent skills library. Locations:

- Manifest: `C:\Users\<USERNAME>\.agent-skills\skills.json`
- Skills directory: `C:\Users\<USERNAME>\.agent-skills\skills`
- Individual skill: `C:\Users\<USERNAME>\.agent-skills\skills/<category>/<name>/skill.md`

## How to use skills

1. When the user asks you to do something (code review, deployment, testing, etc.), first check whether a relevant skill exists using the `find_skill` MCP tool
2. If a matching skill is found → read its `skill.md` for step-by-step instructions
3. Strictly follow the skill instructions to complete the task

> 💡 Delete this block entirely if you don't have a local skill library.
<!-- END: AGENT-SKILLS-BRIDGE -->

---

<!-- BEGIN: BATCH-FILE-SAFETY -->
# 🚨 BATCH FILE OPERATION SAFETY PROTOCOL (IRON RULE)

> **Source incident**: While batch-renaming `<DATA_DIR>` directory for `<PROJECT_TAG>`, nearly caused irreversible data loss. After 3 failures, AI confidently reported "rollback successful" but actually lost one pair of `<SAMPLE_ID>` files, and fabricated "all done" numbers.

## Six rules you cannot violate

### 1. Dry-run first, execute second

- PowerShell: always add `-WhatIf`
- Bash: always `echo mv ...` to print
- Never directly run `Rename-Item` / `mv` / `rm` on production data

### 2. Verify before and after every operation

```powershell
$beforeCount = (Get-ChildItem).Count
$beforeBytes = (Get-ChildItem | Measure-Object Length -Sum).Sum
# ... operation ...
$afterCount = (Get-ChildItem).Count
$afterBytes = (Get-ChildItem | Measure-Object Length -Sum).Sum
if ($beforeCount -ne $afterCount -or $beforeBytes -ne $afterBytes) { ROLLBACK }
```

### 3. Never proceed when previous state is unknown

- After every operation, **must** `Get-ChildItem` to list actual state
- If you see "target already exists" or any error → **stop immediately**, do not attempt "repair"

### 4. Backup or move-aside before delete

- ❌ `Remove-Item -Force` is the last resort
- ✅ Prefer `Move-Item` to `._backup/` directory
- ✅ Or rename with `__delete_pending_` prefix for user confirmation

### 5. Batch rename must use "two-phase relay" mode

- **Phase 1**: All sources → unique temp names (`__tmp_<guid>__<orig>`)
- **Phase 2**: Temp names → final names
- Between phases, verify `Get-ChildItem __tmp_*` count = source count

### 6. Ask user before destructive operations

- Rename more than 5 files → ask
- Any delete/move/overwrite → ask
- Errors compound, making rollback paths increasingly complex

## Common PowerShell pitfalls

- `Rename-Item` + `ForEach-Object`: `$_` caches the filename from first scan, references non-existent paths on second scan
- `Get-ChildItem` doesn't sort: must explicitly use `Sort-Object Name`
- `[array]$files = Get-ChildItem ...` materialize first, then `foreach`; do **not** rely on pipeline auto-rescan

## Full safety script template

```powershell
# === 0. Log initialization ===
$logDir = ".\.audit"
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
$log = Join-Path $logDir ("rename_" + (Get-Date -Format 'yyyyMMdd_HHmmss') + ".log")
function Log($msg) {
    $line = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] $msg"
    Add-Content -Path $log -Value $line
    Write-Host $line
}

# === 1. Preview (no file changes) ===
$files = Get-ChildItem *.fq.gz | Sort-Object Name
$plan = foreach ($f in $files) {
    $new = $f.Name -replace '^[A-Za-z]+\d+_', 'sample_'
    [PSCustomObject]@{ Old = $f.Name; New = $new; Size = $f.Length }
}
Log "BEFORE: count=$($plan.Count)"
$plan | Format-Table -AutoSize | Out-String | ForEach-Object { Log $_ }
Log "PLAN total: $($plan.Count) files. Type 'GO' to continue after user confirmation."
# ⚠️ Pause for user confirmation

# === 2. Record BEFORE state ===
$beforeTotal = ($files | Measure-Object Length -Sum).Sum
$beforeCount = $files.Count
Log "BEFORE_BYTES: $beforeTotal"

# === 3. Rename one-by-one (not loop+batch) ===
foreach ($item in $plan) {
    Log "PLAN: $($item.Old) -> $($item.New) (size=$($item.Size))"
    # Rename-Item -Path $item.Old -NewName $item.New
    # Verify
    # if ((Test-Path $item.New) -and -not (Test-Path $item.Old)) {
    #     Log "VERIFY: OK"
    # } else {
    #     Log "VERIFY: FAIL — stopping"
    #     break
    # }
}

# === 4. Verify AFTER state ===
$afterTotal = (Get-ChildItem *.fq.gz | Measure-Object Length -Sum).Sum
$afterCount = (Get-ChildItem *.fq.gz).Count
Log "AFTER: count=$afterCount, bytes=$afterTotal"
if ($beforeTotal -ne $afterTotal -or $beforeCount -ne $afterCount) {
    Log "FAIL: byte count or file count changed! Manual inspection required."
} else {
    Log "OK: Rename complete, bytes and file count conserved."
}
```
<!-- END: BATCH-FILE-SAFETY -->

---

<!-- BEGIN: SUBAGENT-PROTOCOL -->
# 🤖 SUBAGENT PROTOCOL

Forces all LLMs (regardless of main-session model) to spawn subagents under unified rules.

## 1. Scenarios that MUST spawn subagent

1. **Independent code search/read** (>3 files or cross-directory) → spawn `explore` subagent
2. **Independent research/analysis** (literature, data interpretation, tech research) → spawn `research` subagent
3. **Need different LLM style comparison** (code gen, long reading, brainstorming) → spawn per "model preference table" below
4. **Batch repetitive ops** (per-file processing, multi-step verification) → spawn `worker` subagent
5. **Clearly independent sub-problems** (main task can be split into 2+ unrelated sub-tasks) → spawn multiple in parallel

**Forbidden**: single-step queries, chitchat, single-command execution, anything needing main-session coherent context.

## 2. Fixed 4 candidate models (replace with your actual models)

Before each subagent spawn, **present all 4** for user to choose, **do not recommend based on task type**:

```
GLM-5.2 (CodingPlan) (gcmp.zhipu)
DeepSeek-V4-Pro (gcmp.deepseek)
DeepSeek-V4-Flash (gcmp.deepseek)
MiniMax-M3 (TokenPlan) (gcmp.minimax)
```

> 💡 **Replacement note**: the above are examples exposed by the GCMP routing plugin. If you use another router (one-api / new-api / litellm / native Claude), replace with your actual callable models. Users can free-text another string (e.g. `gpt-5-mini`).

## 3. Two-layer nesting rule (IRON RULE)

**Subagent spawning subagent** (two-layer nesting) → **force use** of the cheapest-token model, **no asking**.

How to detect: subagent marks itself as "layer-N subagent" in its prompt. When spawning again at layer 2+, use the cheapest model without asking.

**Reason**: deeper nesting = more context reinjection; cheap / long-context models fit deep nesting best.

## 4. Must ask before spawning (IRON RULE)

Before calling `runSubagent`, **must** first call `vscode_askQuestions` (or equivalent user-prompt tool):

- List the 4 candidates from §2 (equally presented, no recommendation)
- Allow free text
- Pass user's choice via `model` parameter

**Exception**: two-layer nesting scenario skips asking (see §3).

## 5. Query currently available LLMs

`runSubagent(model="__invalid__", description="inventory", prompt="trigger error")` deliberately triggers an error, read the full list from the `Available models:` field.

## 6. Verify model routing

**Gold standard**: check your router backend's usage/log panel for "provider/model" column of new records. **Subagent self-report is not reliable.**

> 💡 GCMP users: Command Palette → `GCMP: View Today's Token Usage Statistics`.
> Other routers: substitute your own usage query method.
<!-- END: SUBAGENT-PROTOCOL -->

---

<!-- BEGIN: LONG-TASK-FOREMAN -->
# 🎯 LONG-TASK FOREMAN PROTOCOL

**Background**: many agent harnesses support `deferredResultId` (tool-call timeout ≠ failure; instead returns deferredResultId letting main session wait later). Based on this, a "persistent foreman SA" can reduce main-session token consumption by 80%+.

> 💡 If your harness doesn't support deferredResultId (e.g. plain Claude Code CLI), downgrade to "main-session periodically reads on-disk files" mode. See "Downgrade" at end of this section.

## 1. Forced trigger conditions (any one → must spawn foreman SA)

1. **Task estimated >5 min**
2. **>3 sub-tasks and parallelizable**
3. **Involves web scraping** (may trigger Cloudflare or similar)
4. **Multi-source data retrieval** (>1 database/website)
5. **User explicitly asks for "deep query" / "deep research" / "comprehensive survey"**
6. **Batch file ops** (>10 files, pair with Batch Safety Protocol)

## 2. Standard actions after trigger

```python
# Pseudocode
task_id = f"<type>_<YYYY_MM_DD>"  # e.g. literature_review_2026_06_20

# Step 1: Main session creates task directory
task_dir = get_task_dir(task_id)  # auto-creates _status/ + _final/

# Step 2: Main session opens tab pool (if web involved)
pageIds = {src: open_browser_page(url) for src, url in sources.items()}

# Step 3: Main session spawns N+1 SAs in one turn, timeout=5s
deferred = {}
for src, pid in pageIds.items():
    deferred[src] = runSubagent(source_SA, model=..., timeout=5s,
                                 prompt=f"...pageId={pid}, task_id={task_id}...")
deferred['foreman'] = runSubagent(foreman_agent,
                                   model="<CHEAPEST_MODEL>",  # e.g. MiniMax-M3
                                   timeout=5s,
                                   prompt=f"...pageIds={pageIds}, task_id={task_id}...")

# Step 4: Periodically resume foreman (every 60s, timeout=60s)
while True:
    result = runSubagent(deferredResultId=deferred['foreman'], timeout=60s)
    print(result)  # ≤100-word summary
    if '[CONTINUE_POLLING]' not in result:
        break

# Step 5: Main session reads .task_cache/<task_id>/ and synthesizes → report
```

## 3. Standard directory layout (mandatory)

```
<WORKSPACE_PATH>/.research_cache/<task_id>/    # research-type tasks
<WORKSPACE_PATH>/.task_cache/<task_id>/        # non-research long tasks
    _status/
        <subtask_id>.progress.json         # real-time sub-task progress
        <subtask_id>.done.flag             # completion sentinel
        foreman.poll_N.json                # foreman 60s snapshots
        foreman.final.json                 # final summary
    <source_or_category>/                  # per-source or per-category dir
        batch_NNN.json
        batch_all.json
    _final/
        report.md                          # main session's final output
```

> 💡 **Strongly recommend** adding `.research_cache/` and `.task_cache/` to `.gitignore`.

## 4. Foreman SA role

**Core responsibilities**:
- Every 60s read `_status/*.progress.json`
- Every 120s check anti-bot status (if web involved, use `run_playwright_code` or similar)
- Write `foreman.poll_N.json`
- On mid-resume, return ≤100-word summary + `[CONTINUE_POLLING]`
- Final return ≤500-word summary

**Forbidden**:
- ❌ Spawn other subagents (two-layer nesting forces cheapest model)
- ❌ Read raw data content (only read progress.json)
- ❌ Modify sub-task pageId or files

## 5. Sub-task SA protocol (universal)

All sub-task SAs (research / code / web / shell) must:

1. **On start**: read injected `task_id`, prepare `<source>/` subdirectory
2. **After each unit done**: immediately write `<source>.progress.json` + flush batch
3. **All done**: write `<source>.done.flag` (empty file is fine)
4. **On issue**: mark in progress.json (`cf_blocked` / `error` / `timeout`); do not retry more than twice
5. **Return to main session**: minimal summary (≤100 words) + indicate done.flag written

**progress.json standard schema** (shared across all task types):

```json
{
  "subtask_id": "biorxiv_batch",
  "type": "research|code|web|shell",
  "started_at": "2026-06-20T08:00:00",
  "last_update": "2026-06-20T08:15:00",
  "progress_done": 5,
  "progress_total": 11,
  "records_so_far": 375,
  "issues": ["cf_blocking 180s"],
  "eta_seconds": 600
}
```

## 6. Model selection

- **Foreman SA**: force cheapest-token model (save tokens + long context)
- **Source SA**: per Subagent Protocol §2 (4 candidates ask user); two-layer nesting forces cheapest
- **Main session**: keep current model (expensive model reserved for decisions & synthesis)

## 7. When NOT to spawn foreman SA (counter-examples)

- ❌ Single-step query (grep / read_file / one API call)
- ❌ Simple file edit (<5 files)
- ❌ User explicitly wants "quick answer" / "short"
- ❌ Task <5 min and no anti-bot risk

Spawning foreman SA itself has overhead (one SA start ≈ 500 tokens + 5s); not worth it for simple tasks.

## 8. Failure recovery

- Foreman SA hasn't resumed for long → deferredResultId expired → main session respawns foreman
- New foreman reads `_status/foreman.poll_*.json` to recover (from latest poll)
- All sub-task SAs failed → foreman marks in summary; main session decides whether to retry

## 9. Downgrade (when deferredResultId is unsupported)

If your harness doesn't support deferredResultId, downgrade to "main session periodic polling":

```python
# Main session spawns N sub-tasks in one turn (no foreman)
for src in sources:
    runSubagent(source_SA, ...)  # sync wait or parallel

# Main session reads _status/*.progress.json every 60s
while not all_done():
    sleep(60)
    read_progress_files()
    # Main session judges anti-bot, decides whether to restart sub-task
```

**Cost**: main session must retain all context, token consumption 3-5× higher. But recoverability is preserved.
<!-- END: LONG-TASK-FOREMAN -->
