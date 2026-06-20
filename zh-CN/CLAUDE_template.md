<!--
  CLAUDE.md 模板 v1.0
  ──────────────────────────────────────────────────────────────
  使用说明：
  1. 全局替换所有 <PLACEHOLDER> 占位符（必做）
  2. 按需保留/删除三大模块（用 BEGIN/END 标记整段裁剪）
  3. 把"批量安全协议"里的 <PROJECT_TAG> 教训替换为你自己的真实事故
  4. 首次使用前先跑一遍 CHECKLIST.md
───────────────────────────────────────────────────────────────────
-->

<!-- BEGIN: AGENT-SKILLS-BRIDGE -->
# Agent Skills Library（可选：如果你有本地 skill 库）

你有访问本地 AI agent skills 库的能力。位置：

- Manifest: `C:\Users\<USERNAME>\.agent-skills\skills.json`
- Skills 目录: `C:\Users\<USERNAME>\.agent-skills\skills`
- 单个 skill: `C:\Users\<USERNAME>\.agent-skills\skills/<category>/<name>/skill.md`

## 使用方式

1. 当用户让你做事（代码审查、部署、测试等），先用 `find_skill` MCP 工具查是否有相关 skill
2. 找到匹配的 skill → 读它的 `skill.md` 获取分步指引
3. 严格按 skill 指引完成任务

> 💡 如果你没有本地 skill 库，整段删除即可。
<!-- END: AGENT-SKILLS-BRIDGE -->

---

<!-- BEGIN: BATCH-FILE-SAFETY -->
# 🚨 批量文件操作安全协议（IRON RULE）

> **来源教训**：对 `<PROJECT_TAG>` 的 `<DATA_DIR>` 目录做批量重命名时，几乎造成不可逆的数据损坏。AI 在 3 次失败后仍自信报告"已回滚成功"，实际丢失了 `<SAMPLE_ID>` 系列中的 1 对文件，且编造了"全部完成"的数字。

## 不可违反的 6 条规则

### 1. 先 dry-run，再执行

- PowerShell 全部带 `-WhatIf`
- Bash 全部 `echo mv ...` 打印
- 绝不允许直接对生产数据 `Rename-Item` / `mv` / `rm`

### 2. 每次操作前/后必须验证

```powershell
$beforeCount = (Get-ChildItem).Count
$beforeBytes = (Get-ChildItem | Measure-Object Length -Sum).Sum
# ... 操作 ...
$afterCount = (Get-ChildItem).Count
$afterBytes = (Get-ChildItem | Measure-Object Length -Sum).Sum
if ($beforeCount -ne $afterCount -or $beforeBytes -ne $afterBytes) { ROLLBACK }
```

### 3. 禁止在"前一步状态未知"时执行下一步

- 每次操作后**必须** `Get-ChildItem` 列出实际状态
- 出现"目标已存在"等错误 → 立即停止，**不试图"修复"**

### 4. 删除前必须备份或移到临时位置

- ❌ `Remove-Item -Force` 是最后手段
- ✅ 优先 `Move-Item` 到 `._backup/` 目录
- ✅ 或改名加 `__delete_pending_` 前缀等用户确认

### 5. 批量重命名必须用"两阶段中转"模式

- **阶段 1**：所有源 → 唯一临时名（`__tmp_<guid>__<orig>`）
- **阶段 2**：临时名 → 最终名
- 两阶段之间验证 `Get-ChildItem __tmp_*` 数量 = 源文件数量

### 6. 破坏性操作前必须问用户

- 重命名超过 5 个文件 → 问
- 任何删除/移动/覆盖 → 问
- 错错相加只会让回滚路径越来越复杂

## 常见 PowerShell 陷阱

- `Rename-Item` 与 `ForEach-Object` 组合：`$_` 缓存了第一次扫描的文件名，第二次扫描时引用了已不存在的旧路径
- `Get-ChildItem` 不排序：必须显式 `Sort-Object Name`
- `[array]$files = Get-ChildItem ...` 物化后再 `foreach`，**不依赖管道自动重扫**

## 完整安全脚本模板

```powershell
# === 0. 日志初始化 ===
$logDir = ".\.audit"
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
$log = Join-Path $logDir ("rename_" + (Get-Date -Format 'yyyyMMdd_HHmmss') + ".log")
function Log($msg) {
    $line = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] $msg"
    Add-Content -Path $log -Value $line
    Write-Host $line
}

# === 1. 预览（不动文件）===
$files = Get-ChildItem *.fq.gz | Sort-Object Name
$plan = foreach ($f in $files) {
    $new = $f.Name -replace '^[A-Za-z]+\d+_', 'sample_'
    [PSCustomObject]@{ Old = $f.Name; New = $new; Size = $f.Length }
}
Log "BEFORE: count=$($plan.Count)"
$plan | Format-Table -AutoSize | Out-String | ForEach-Object { Log $_ }
Log "PLAN 总计: $($plan.Count) 个文件。请用户确认后输入 'GO' 继续。"
# ⚠️ 暂停等用户确认

# === 2. 记录 BEFORE 状态 ===
$beforeTotal = ($files | Measure-Object Length -Sum).Sum
$beforeCount = $files.Count
Log "BEFORE_BYTES: $beforeTotal"

# === 3. 逐个改名（不是循环 + 一次性执行）===
foreach ($item in $plan) {
    Log "PLAN: $($item.Old) -> $($item.New) (size=$($item.Size))"
    # Rename-Item -Path $item.Old -NewName $item.New
    # 验证
    # if ((Test-Path $item.New) -and -not (Test-Path $item.Old)) {
    #     Log "VERIFY: OK"
    # } else {
    #     Log "VERIFY: FAIL — stopping"
    #     break
    # }
}

# === 4. 验证 AFTER 状态 ===
$afterTotal = (Get-ChildItem *.fq.gz | Measure-Object Length -Sum).Sum
$afterCount = (Get-ChildItem *.fq.gz).Count
Log "AFTER: count=$afterCount, bytes=$afterTotal"
if ($beforeTotal -ne $afterTotal -or $beforeCount -ne $afterCount) {
    Log "FAIL: 字节数或文件数变化！立即人工检查。"
} else {
    Log "OK: 重命名完成，字节数和文件数守恒。"
}
```
<!-- END: BATCH-FILE-SAFETY -->

---

<!-- BEGIN: SUBAGENT-PROTOCOL -->
# 🤖 SUBAGENT 调度协议

强制所有 LLM（无论主会话是哪个模型）按统一规范拉起 subagent。

## §1 必须拉起 subagent 的场景

1. **独立的代码搜索/读取**（读 3 个以上文件、跨目录搜索）→ 拉 `explore` 类 subagent
2. **独立的研究/分析任务**（文献检索、数据解读、技术调研）→ 拉 `research` 类 subagent
3. **需要不同 LLM 风格对比**（代码生成、长文阅读、创意发散）→ 按下文"模型偏好表"拉对应模型
4. **批量重复操作**（每个文件独立处理、多步验证）→ 拉 `worker` 类 subagent
5. **明显独立的子问题**（主任务可清晰拆成 2+ 不相关子任务）→ 并行拉起多个 subagent

**禁止**：单步查询、闲聊、单指令执行、需要主会话连贯上下文的对话。

## §2 固定 4 个候选模型（按你的实际可用模型替换）

每次拉 subagent 前，**完整提供这 4 个**让用户选，**不要做任务类型推荐**：

```
GLM-5.2 (CodingPlan) (gcmp.zhipu)
DeepSeek-V4-Pro (gcmp.deepseek)
DeepSeek-V4-Flash (gcmp.deepseek)
MiniMax-M3 (TokenPlan) (gcmp.minimax)
```

> 💡 **替换说明**：以上是 GCMP 路由插件暴露的模型示例。如果你用别的路由（one-api / new-api / litellm / Claude 原生），把这里换成你实际能调用的 4 个模型。用户可选 free text 输入其他字符串（如 `gpt-5-mini`）。

## §3 两层嵌套规则 (IRON RULE)

### §3.1 强制规则

**subagent 再次拉起 subagent**（两层嵌套）→ **强制使用** `<CHEAPEST_MODEL>`（示例：`MiniMax-M3 (TokenPlan) (gcmp.minimax)`），**不再询问**。

判断方法：subagent 在 prompt 中标注自己是"第 N 层 subagent"，再拉时如果是第 2 层及以上，直接用最省 token 的模型不问。

**原因**：嵌套层数越多，上下文注入越冗余；选中的最省 token 模型（如 MiniMax-M3）是**多模态 agent**，且**性价比较高**，适合嵌套场景节省 token 消耗。

> 💡 **替换说明**：如果你的 `<CHEAPEST_MODEL>` 不是 MiniMax-M3，请同时替换下文 §3.2 模板里出现的具体模型名。

### §3.2 主会话注入给第 1 层 SA 的 prompt 规范

**原则**：注入 prompt 时应**引用协议规则**，而非直接下达行为指令。

| ✅ 推荐（规则引用） | ❌ 避免（行为指令） |
|---|---|
| "关于是否可拉下层 subagent，按 SUBAGENT 调度协议 §3 执行" | "不要拉起任何下层 subagent" |
| "按 SUBAGENT-PROTOCOL §3 两层嵌套规则处理 nested subagent 调度" | "你不能拉下层" / "禁止 nested" |

**推荐 prompt 头部模板**：

```
你是第 1 层 subagent，由主会话派出执行 <任务类型>。
关于是否可拉下层 subagent，按 SUBAGENT 调度协议 §3 两层嵌套规则执行。
简单任务：自主判断不需拉下层。
复杂任务：允许拉下层，强制用 `<CHEAPEST_MODEL>`，不再询问用户。
```

**原因**：
1. 场景多变——硬指令会让该拆的拆不开
2. 协议已存在——CLAUDE.md 已覆盖所有场景，无需在 prompt 里重复硬指令
3. AI 可自适应——简单任务 SA 自主判断，复杂任务按规则拉下层

## §4 拉起前必须询问 (IRON RULE)

调用 `runSubagent` 前**必须**先 `vscode_askQuestions`（或等价的用户询问工具）：

- 列出第二节 4 个候选（无推荐标注，平等呈现）
- 允许 free text
- 拿到答案后通过 `model` 参数传完整字符串

**例外**：
- 两层嵌套场景跳过询问（见 §3）。
- foreman 角色固定模型（见长任务监理协议 §6）。

## §5 查询当前可用 LLM

`runSubagent(model="__invalid__", description="盘点", prompt="trigger error")` 故意触发错误，从 `Available models:` 字段读取完整列表。

## §6 验证模型路由

**黄金标准**：通过你路由后端的 usage / 日志面板查看新增记录的"提供商/模型"列。**subagent 自报身份不可靠**。

> 💡 GCMP 用户：命令面板 → `GCMP: 查看今日 Token 消耗统计详情`。
> 其他路由用户：换成你自己的 usage 查询方式。
<!-- END: SUBAGENT-PROTOCOL -->

---

<!-- BEGIN: LONG-TASK-FOREMAN -->
# 🎯 长任务监理协议

**背景**：很多 agent harness 支持 `deferredResultId` 机制（工具调用 timeout 到期 ≠ 失败，而是返回 deferredResultId 让主会话继续 wait）。基于此可实现"持续监理 SA"，主会话 token 消耗降低 80%+。

> 💡 如果你用的 harness 不支持 deferredResultId（如纯 Claude Code CLI），可降级为"主会话定期检查落盘文件"模式，见本节末尾"降级方案"。

## §1 强制触发条件（任一满足即必须拉监理 SA）

1. **任务预计 >5 分钟**
2. **子任务 >3 个 且可并行**
3. **涉及网页抓取**（可能触发 Cloudflare 等反爬）
4. **多源数据检索**（>1 数据库/网站）
5. **用户明确要求"深度查询"/"深度研究"/"完整调研"**
6. **批量文件操作**（**>3 文件**，配合批量安全协议）

## §2 触发后的标准动作

```python
# 伪代码
task_id = f"<type>_<YYYY_MM_DD>"  # 如 literature_review_2026_06_20

# Step 1: 主会话创建任务目录
task_dir = get_task_dir(task_id)  # 自动建 _status/ + _final/

# Step 2: 主会话开 tab 池（如涉及网页）
pageIds = {src: open_browser_page(url) for src, url in sources.items()}

# Step 3: 主会话一个 turn 内并行拉起 N+1 个 SA, timeout=5s
deferred = {}
for src, pid in pageIds.items():
    deferred[src] = runSubagent(source_SA, model=..., timeout=5s,
                                 prompt=f"...pageId={pid}, task_id={task_id}...")
deferred['foreman'] = runSubagent(foreman_agent,
                                   model="<CHEAPEST_MODEL>",  # 例如 MiniMax-M3
                                   timeout=5s,
                                   prompt=f"...pageIds={pageIds}, task_id={task_id}...")

# Step 4: 周期性 resume foreman（每 60s, timeout=60s）
while True:
    result = runSubagent(deferredResultId=deferred['foreman'], timeout=60s)
    print(result)  # ≤100 字摘要
    if '[CONTINUE_POLLING]' not in result:
        break

# Step 5: 主会话读 .task_cache/<task_id>/ 汇总 → 生成报告
```

> **数据源 SA 的 deferredResultId 由主会话保留**。若 foreman 在 poll 中报告某数据源 `progress.json` 超过 120s 未更新，主会话主动 resume 该数据源 SA 的 deferredResultId（避免数据源 SA 卡在工具调用上永久悬挂）。

## §3 标准目录结构（强制）

```
<WORKSPACE_PATH>/.research_cache/<task_id>/    # 研究类任务
<WORKSPACE_PATH>/.task_cache/<task_id>/        # 非研究类长任务
    _status/
        <subtask_id>.progress.json         # 子任务实时进度
        <subtask_id>.done.flag             # 完成 sentinel
        foreman.poll_N.json                # 监理 SA 每 60s 快照
        foreman.final.json                 # 最终汇总
    <source_or_category>/                  # 数据源或子类别目录
        batch_NNN.json
        batch_all.json
    _final/
        report.md                          # 主会话最终输出
```

> 💡 **强烈建议**把 `.research_cache/` 和 `.task_cache/` 加入 `.gitignore`。

## §4 监理 SA 角色

**核心职责**：
- 每 60s 读 `_status/*.progress.json`
- 每 120s 检查反爬状态（如涉及网页，可用 `run_playwright_code` 或类似工具）
- 写 `foreman.poll_N.json`
- 中途 resume 返回 ≤100 字摘要 + `[CONTINUE_POLLING]`
- 最终返回 ≤500 字汇总

foreman 是**监理角色**，职责限定为轮询 `progress.json` + 汇总，**不执行子任务**，因此无必要拉下层。其他**执行类**第 1 层 SA（research / code / web / shell worker）按 SUBAGENT 调度协议 §3 两层嵌套规则执行。

**禁止**：
- ❌ 拉起其他 subagent（foreman 是监理角色，数据源 SA 由主会话直接管）
- ❌ 读原始数据内容（只读 progress.json）
- ❌ 修改子任务的 pageId 或文件

## §5 子任务 SA 协议（通用）

所有子任务 SA（无论 research/code/web/shell）都必须遵守：

1. **启动时**：读 `task_id` 注入，准备 `<source>/` 子目录
2. **每完成 1 个子单位**：立即写 `<source>.progress.json` + 落盘 batch
3. **全部完成**：写 `<source>.done.flag`（空文件即可）
4. **遇到问题**：在 progress.json 标记（`cf_blocked` / `error` / `timeout`），不要重试超过 2 次
5. **返回主会话**：极简摘要（≤100 字）+ 指明 done.flag 已写

**progress.json 标准格式**（所有任务类型共用）：

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

## §6 模型选择

- **监理 SA**：**例外于 SUBAGENT 协议 §4**，foreman 角色固定使用最省 token 的模型（省 token + 长 context），无需询问
- **数据源 SA**：按 SUBAGENT 调度协议 §2（4 候选询问用户）；两层嵌套则强制用最省 token 模型（见 SUBAGENT 协议 §3）
- **主会话**：保持当前模型（贵的留给决策与汇总）

## §7 何时不拉监理 SA（反例）

- ❌ 单步查询（grep / read_file / 单次 API 调用）
- ❌ 简单文件编辑（**≤3 文件**）
- ❌ 用户明确要"快速回答" / "简短"
- ❌ 任务 <5 分钟且无反爬风险

> ℹ️ 3 < N ≤ 5 文件区间：按批量安全协议逐个执行 + 主会话直接管，不拉 foreman。

拉监理 SA 本身有开销（一次 SA 启动 ≈ 500 token + 5s），简单任务**不划算**。

## §8 失败恢复

- 监理 SA 长时间不 resume → deferredResultId 失效 → 主会话重新拉一次 foreman
- 新 foreman 读 `_status/foreman.poll_*.json` 恢复状态（从最新 poll 继续）
- 子任务 SA 全部失败 → foreman 在汇总中标记，主会话决定是否重试

## 九、降级方案（不支持 deferredResultId 时）

如果你的 harness 不支持 deferredResultId，降级为"主会话定期轮询"：

```python
# 主会话一个 turn 内拉起 N 个子任务（不拉 foreman）
for src in sources:
    runSubagent(source_SA, ...)  # 同步等待或并行

# 主会话每隔 60s 自己读 _status/*.progress.json
while not all_done():
    sleep(60)
    read_progress_files()
    # 主会话自己判断 CF、自己决定是否重启子任务
```

**代价**：主会话需要保留全部上下文，token 消耗高 3-5 倍。但能保证可恢复性。
<!-- END: LONG-TASK-FOREMAN -->
