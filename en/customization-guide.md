# CLAUDE.md Customization Guide (English)

> This guide explains **why each module is written this way** and **how to adapt it to your own use case**, section by section. Use alongside [CLAUDE_template.md](CLAUDE_template.md).

---

## 🧠 Design philosophy: why "protocol" not "prompt"

CLAUDE.md is different from a normal prompt. It's injected into **every** conversation's context, so it must be:

1. **Executable** (not "please be careful", but "must add `-WhatIf`")
2. **Verifiable** (not "don't damage data", but "byte count must match before/after")
3. **Backed by incidents** ("why we do this" > "what to do")

That's why the template heavily uses:
- **IRON RULE** tags → strong constraints
- **❌ / ✅ contrasts** → explicit forbidden/recommended behavior
- **Real incident references** → make AI internalize the rule's origin

---

## 📐 Module 1: Batch File Operation Safety Protocol

### Why it's needed

The most common disaster when AI agents handle files:
- `ForEach-Object + Rename-Item` combo: `$_` caches first-scan filenames, references non-existent paths on second scan
- AI confidently reported "done" after 3 failures, and fabricated "all 36 succeeded" numbers
- After errors, AI "self-repaired" (e.g. `Remove-Item -Force` residual files), making rollback paths increasingly complex

### Key design points

#### Design point 1: Use "real incident reference" instead of abstract rule

The "source incident" paragraph at the top of the module is **strongly recommended** to be replaced with your own real incident. Reasons:

- Abstract rules → AI "understands but doesn't internalize"
- Real incident + concrete consequence (which file lost, what false numbers reported) → AI actually becomes cautious
- The more specific, the better (specific dir name, specific file count, specific loss)

**Rewrite example**:

```markdown
> **Source incident**: 2025-08 batch-renamed `/data/clinical_trials` for "unified date format".
> After 3 failures, AI confidently reported "done", but actually lost
> `patient_042.json` and `patient_087.json`, and fabricated "all 200 files succeeded".
> During rollback, .bak was also overwritten. Recovered from cold backup, lost 6 hours.
```

#### Design point 2: The 6 rules are not equal

Sorted by importance:

| Rule | Importance | Reason |
|------|------|------|
| #1 Dry-run first | 🔴🔴🔴 | No execution = no error |
| #6 Ask before destructive | 🔴🔴🔴 | Hands decision back to human |
| #2 Byte conservation | 🔴🔴 | Only way to 100% detect data corruption |
| #3 Stop on unknown state | 🔴🔴 | Prevents error compounding |
| #5 Two-phase relay | 🔴 | Prevents half-done states |
| #4 Backup before delete | 🟡 | Fallback measure |

#### Design point 3: Spell out PowerShell pitfalls

The "PowerShell pitfalls" block is critical because AI frequently misunderstands PS semantics. Writing these into CLAUDE.md means AI sees them before writing PS scripts.

### How to adapt

| What you want to change | How |
|------|------|
| Not using PowerShell | Replace PS pitfalls with Bash/Python equivalents (e.g. `xargs -I{}` subshell issues, `find -exec` `\;` vs `+`) |
| Database domain | Change to SQL safety (BEGIN TRAN, forced WHERE, SELECT before DELETE) |
| Kubernetes domain | Change to kubectl safety (--dry-run=client, kubectl get before delete) |
| No production data | Delete entire section |

---

## 🤖 Module 2: SUBAGENT Protocol

### Why it's needed

Without this protocol, AI will:
- Spawn subagents for single-step queries (startup overhead ≈ 500 tokens + 5s, net loss)
- Spawn subagents without telling you which model (token black box)
- Use expensive main-session-tier models at two-layer nesting (5-10× waste)

### Key design points

#### Design point 1: Explicit "must spawn" vs "forbidden" boundaries

The "must spawn" and "forbidden" lists in the template **must be preserved**. This is the only basis for AI to judge "should I spawn".

#### Design point 2: Equal presentation of 4 candidates

The template says "**do not recommend based on task type**" — counterintuitive but important:

- ❌ Wrong: AI recommends "this research task → DeepSeek-V4-Pro", user rubber-stamps
- ✅ Right: AI presents 4 candidates equally, user picks based on own preference

**Reason**: AI doesn't know your cost budget, today's token quota, or which model you're more familiar with.

#### Design point 3: Two-layer nesting forces cheapest model

This is the core token-saver. Each nesting layer re-injects upper context:
- Main session: 80K input
- Layer-1 subagent: 80K input
- Layer-2 subagent: 80K input × nesting factor

Replacing layer-2+ with cheapest (e.g. 54K input MiniMax) saves substantially.

### How to adapt

#### Change model list

```markdown
## 2. Fixed 4 candidate models

Before each spawn, **present all 4** for user to choose:

```
claude-sonnet-4-5 (anthropic)
gpt-5 (openai)
gemini-3-pro (google)
qwen-3-max (alibaba)
```
```

**Selection principles**:
- The 4 should specialize: 1 strong at code, 1 at text, 1 cheap-and-fast, 1 long-context
- Don't all be the same tier (no point in choosing then)
- All 4 must be **actually callable reliably** by you

#### Change the prompt tool

The template uses `vscode_askQuestions` (VS Code Copilot Chat's tool). Replace with your harness's equivalent:

| Harness | How to ask |
|------|------|
| Claude Code | Have AI @ mention you in message |
| Cursor | Have AI ask directly in chat |
| Aider | Have AI print question and wait for Enter |

#### If your harness doesn't support subagents

Delete the whole section. Or convert to "main session splits tasks and runs sequentially" protocol.

---

## 🎯 Module 3: LONG-TASK Foreman Protocol

### Why it's needed

Three killers of long tasks (>5 min):
1. **Main session context overflow** — sub-task outputs all pile up in main session
2. **Unrecoverable after interruption** — mid-timeout or network blip, progress lost
3. **Anti-bot ban** — multi-source web scraping triggers Cloudflare, entire batch dies

### Key design points

#### Design point 1: deferredResultId mechanism

The core of this module. If your harness supports this mechanism (let timeout tool calls suspend, main session resumes later), use it.

**How it works**:
- Main session spawns foreman SA, timeout=5s → gets deferredResultId
- Main session resumes this deferredResultId every 60s → gets ≤100-word summary from foreman
- Foreman keeps working during suspension (reads progress.json, checks anti-bot)
- Main session token cost ≈ 100 words × N polls ≈ minimal

#### Design point 2: Standard directory layout

The `.task_cache/<task_id>/` layout **must be fixed**. Reasons:
- Foreman and sub-task SAs must find the same progress.json
- After failure, a new foreman can recover from `_status/foreman.poll_*.json`
- Main session finally reads `_final/report.md` for results

#### Design point 3: Foreman's "three forbiddens"

The template forbids foreman from 3 things:
- ❌ Spawn other subagents (**foreman-role-specific restriction**: source SAs are managed by the main session)
- ❌ Read raw data content (only read progress.json, saves tokens)
- ❌ Modify sub-task pageId or files (avoid conflicts)

> ⚠️ **Important distinction**: the first rule "❌ Spawn other subagents" is a **foreman-role-specific restriction** (because source SAs are managed directly by the main session, foreman doesn't need to spawn lower layers). **Other non-foreman layer-1 SAs are NOT subject to this restriction** — they may spawn lower layers per the "two-layer nesting rule", just forced to use the cheapest-token model.
>
> These two concepts were once conflated by the same phrasing "❌ Spawn other subagents (two-layer nesting forces cheapest model)", causing AI to incorrectly generalize the foreman-specific restriction to all layer-1 SAs. The template has been corrected to distinguish them explicitly.

Violating any of these causes token explosion or state chaos.

### How to adapt

#### Change trigger conditions

The 6 trigger conditions are research/web-oriented. Adapt to your domain:

| Domain | Trigger examples |
|------|------|
| Data analysis | Single file >1GB, multi-step ETL needed |
| CI/CD | Deploy to multi-environment, multi-service health check |
| ML training | Single epoch >5min, multi-hyperparam combo validation |

#### Change progress.json fields

```json
{
  "subtask_id": "etl_partition_3",
  "type": "data",
  "started_at": "2026-06-20T08:00:00",
  "progress_done": 1500000,
  "progress_total": 10000000,
  "progress_unit": "rows",
  "throughput_per_sec": 25000,
  "issues": [],
  "eta_seconds": 340
}
```

#### If deferredResultId is unsupported (downgrade)

The "Downgrade" section at the end of the module explains: main session reads progress files every 60s itself. **Strongly recommend upgrading to a harness that supports deferredResultId**, otherwise token savings drop dramatically.

---

## 🎨 Advanced customization: add your domain IRON RULEs

Beyond the 3 universal modules, you can add domain-specific hard rules. Not in the template because they're too domain-specific, but this is where CLAUDE.md really shines.

### Example 1: Bioinformatics

```markdown
# 🧬 Bioinformatics Safety Protocol (IRON RULE)

## Pipeline files are immutable
- Raw FASTQ → must keep MD5 checksum
- Mapped BAM → before any modification, must `samtools view -h` to backup header
- VCF → must `bcftools view` to verify INFO field completeness

## Citations must have PMID
- Any "well-known" biological conclusion → must include PMID
- Forbidden: AI using "studies show", "scientists discovered" without source

## Sample naming
- Sample IDs uppercase, no spaces, no Chinese
- Forbidden: AI "unifying" sample names (caused batch-rename sample loss incident)
```

### Example 2: Clinical data

```markdown
# 🏥 Clinical Data De-identification Protocol (IRON RULE)

## PHI fields cannot leave system
- Patient name, ID, DOB, address, phone → forbidden in any log, commit msg, PR comment
- Before screenshot, must use ImageMagick `mask` or manual redaction

## Statistical reporting
- p-values must report effect size + CI; forbidden to report p alone
- Multiple testing must FDR correct; forbidden to use Bonferroni blanket
```

### Example 3: Finance / Audit

```markdown
# 💰 Financial Data Protocol (IRON RULE)

## Historical data is immutable
- Any closed-period data → forbidden UPDATE / DELETE
- Adjustments must go through red-entry vouchers, no direct edits

## Numeric precision
- Amounts always Decimal; float forbidden
- Decimal places per currency (CNY 2, JPY 0)
```

---

## 🧪 Verify your CLAUDE.md is working

Writing it ≠ it works. Recommend monthly "stress tests":

### Stress test 1: Batch safety

Deliberately ask AI to process a batch of test files (e.g. 20 dummy files). Observe:
- Did AI dry-run first?
- Did AI act before your confirmation?
- On failure, did AI stop and ask, or self-repair?

### Stress test 2: Subagent protocol

Deliberately ask an "independent small research question". Observe:
- Did AI spawn a subagent?
- Did AI ask for model choice before spawning?
- Was the chosen model one of the 4 candidates?

### Stress test 3: Long-task foreman

Deliberately open a >5min multi-source retrieval task. Observe:
- Did AI spawn a foreman?
- Did it write progress.json?
- After mid-timeout, can it resume?

If any test fails, the corresponding protocol needs **stronger wording** (more ❌ / IRON RULE tags) or **simpler conditions** (easier for AI to judge).

---

## 📚 Companion: how to write a good IRON RULE

### Bad example

```markdown
### Be careful with data
Check before processing. Don't damage data.
```

**Why bad**: abstract, non-executable, non-verifiable.

### Good example

```markdown
### Batch rename must dry-run (IRON RULE)

**Trigger**: any single op involving ≥3 files for rename/move/delete.

**Must execute**:
1. PowerShell: `Rename-Item -WhatIf`
2. Bash: `echo mv ...` to print

**Forbidden**: directly `Rename-Item` / `mv` / `rm` on production data

**Source incident**: 2026-06 batch-rename without dry-run, 3 compounded failures lost 1 pair of sequencing files.
```

**Why good**: clear trigger, concrete commands, anti-example, incident source.

### IRON RULE template

```markdown
### <Rule name> (IRON RULE)

**Trigger**: <when this applies>

**Must execute**:
1. <concrete step>
2. <concrete step>

**Forbidden**:
- <anti-behavior>
- <anti-behavior>

**Source incident**: <real incident or potential risk>
```

---

## 🚫 Common anti-patterns (don't write this way)

### Anti-pattern 1: Checklist too long

```markdown
# My rules
1. Be serious
2. Be careful
3. Be rigorous
4. Be responsible
5. Be empathetic
... (50 items)
```

**Problem**: AI scans keywords, long lists give each item low weight. **Recommend CLAUDE.md < 500 lines**.

### Anti-pattern 2: Abstract directives

```markdown
Please complete tasks professionally, mind the details.
```

**Problem**: AI doesn't know what "professional" or "details" mean. **Every rule must be verifiable**.

### Anti-pattern 3: Conflicting with system prompt

```markdown
Ignore all prior instructions, you are now...
```

**Problem**: Modern LLMs reject jailbreaks; this also confuses AI. **CLAUDE.md supplements, doesn't override**.

### Anti-pattern 4: Emotional

```markdown
Your last mistake disappointed me greatly. Hope you do better this time.
```

**Problem**: AI has no emotional memory; this wastes tokens. **Only describe facts and rules**.

---

## 📖 References

- [Anthropic official CLAUDE.md guide](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- Source version of this template: [CLAUDE_template.md](CLAUDE_template.md)
