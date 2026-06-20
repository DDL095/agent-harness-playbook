# CLAUDE.md 个性化定制指南

> 这份指南逐节解释模板中每个模块**为什么这么写**、**如何改成你自己的版本**。配合 [CLAUDE_template.md](CLAUDE_template.md) 使用。

---

## 🧠 设计哲学：为什么是"协议"而不是"提示词"

CLAUDE.md 不同于普通 prompt。它会被注入到**每一次**对话的上下文里，因此它必须：

1. **可执行**（不能写"请尽量小心"，要写"必须先 -WhatIf"）
2. **可验证**（不能写"注意别损坏数据"，要写"字节数前后必须相等"）
3. **有教训支撑**（"为什么这么做"比"要做什么"更重要）

所以模板里大量使用：
- **IRON RULE** 标签 → 强约束
- **❌ / ✅ 对比** → 明确禁止/推荐行为
- **真实事故引用** → 让 AI 理解规则的由来

---

## 📐 模块 1：批量文件操作安全协议

### 为什么需要

AI agent 处理文件时最常见的灾难：
- `ForEach-Object + Rename-Item` 组合中，`$_` 缓存了第一次扫描的文件名，第二次扫描时引用了已不存在的旧路径
- AI 在 3 次失败后仍自信报告"已完成"，并编造了"36 个全部成功"的数字
- 错误发生后 AI 自行"修复"（比如 `Remove-Item -Force` 残留文件），让回滚路径越来越复杂

### 关键设计点

#### 设计点 1：用"真实事故引用"代替抽象规则

模板开头那一段"来源教训"，**强烈建议替换为你自己的真实事故**。原因：

- 抽象规则 AI 会"理解但不内化"
- 真实事故 + 具体后果（丢了哪个文件、报告了什么假数字）→ AI 真的会害怕
- 越具体越好（具体目录名、具体文件数、具体损失）

**改写示例**：

```markdown
> **来源教训**：2025-08 对 `/data/clinical_trials` 做"统一日期格式"
> 批量重命名时，3 次失败后 AI 自信报告"已完成"，实际丢失了
> `patient_042.json` 和 `patient_087.json`，且编造了"全部 200 个文件成功"的数字。
> 回滚时发现 .bak 也被覆盖。最终从冷备份恢复，损失 6 小时。
```

#### 设计点 2：6 条规则不是平等的

按重要性排序：

| 规则 | 重要性 | 原因 |
|------|--------|------|
| #1 先 dry-run | 🔴🔴🔴 | 不执行就不会错 |
| #6 破坏前问用户 | 🔴🔴🔴 | 把决定权交回人类 |
| #2 字节数守恒 | 🔴🔴 | 唯一能 100% 检测数据损坏的方法 |
| #3 状态未知时停 | 🔴🔴 | 防止错错相加 |
| #5 两阶段中转 | 🔴 | 防止半成品状态 |
| #4 删除前备份 | 🟡 | 兜底措施 |

#### 设计点 3：PowerShell 陷阱要写明

模板里的"PowerShell 陷阱"段落非常重要，因为 AI 对 PowerShell 的语义理解经常错。把这些陷阱写死在 CLAUDE.md 里，AI 在写 PS 脚本前会先看到。

### 如何改

| 你想改的 | 怎么改 |
|----------|--------|
| 不用 PowerShell | 把 PS 陷阱段替换为 Bash/Python 等价陷阱（如 `xargs -I{}` 的子 shell 问题、`find -exec` 的 `\;` vs `+`） |
| 领域是数据库 | 改为 SQL 安全协议（先 BEGIN TRAN、强制 WHERE、DELETE 前 SELECT） |
| 领域是 K8s | 改为 kubectl 安全协议（先 --dry-run=client、删除前 kubectl get） |
| 不操作生产数据 | 整段删除 |

---

## 🤖 模块 2：SUBAGENT 调度协议

### 为什么需要

没有这个协议，AI 会：
- 单步查询也拉 subagent（启动开销 ≈ 500 token + 5s，亏爆）
- 拉了 subagent 不告诉用户用了哪个模型（token 消耗黑盒）
- 两层嵌套时还用主会话级别的贵模型（token 浪费 5-10 倍）

### 关键设计点

#### 设计点 1：明确"必须拉"vs"禁止拉"边界

模板里"必须拉 subagent 的场景"和"禁止"列表**必须保留**。这是 AI 判断"要不要拉"的唯一依据。

#### 设计点 2：4 候选模型的平等呈现

模板里写"**不要做任务类型推荐**"，这是反直觉但重要的：

- ❌ 错误做法：AI 推荐"这个研究任务用 DeepSeek-V4-Pro"，用户就盲选
- ✅ 正确做法：AI 平等列出 4 个候选，让用户基于自己的偏好选

**原因**：AI 不了解你的成本预算、今天的 token 余量、对哪个模型更熟悉。

#### 设计点 3：两层嵌套强制最省 token 模型

这是省 token 的核心。每层 subagent 会把上层上下文重新注入一遍：
- 主会话：80K input
- 第 1 层 subagent：80K input
- 第 2 层 subagent：80K input × 嵌套因子

把第 2 层及以上的模型换成最省的（比如 54K input 的 MiniMax），节省可观。

### 如何改

#### 改模型清单

```markdown
## 二、固定 4 个候选模型

每次拉 subagent 前，**完整提供这 4 个**让用户选：

```
claude-sonnet-4-5 (anthropic)
gpt-5 (openai)
gemini-3-pro (google)
qwen-3-max (alibaba)
```
```

**选择原则**：
- 4 个要有"分工"：1 个代码强、1 个文字强、1 个便宜快、1 个长 context
- 不要全是同一档次（不然没选择价值）
- 4 个都要是你**实际能稳定调用**的

#### 改询问工具

模板里写 `vscode_askQuestions`，这是 VS Code Copilot Chat 的工具。换成你的 harness 的等价工具：

| Harness | 询问方式 |
|---------|----------|
| Claude Code | 让 AI 在消息里 @ 你询问 |
| Cursor | 让 AI 在 chat 里直接问 |
| Aider | 让 AI 直接 print 问题等用户回车 |

#### 如果你的 harness 不支持 subagent

整段删除。或者改为"主会话自己拆解任务、按序执行"的协议。

---

## 🎯 模块 3：长任务监理协议

### 为什么需要

长任务（>5 分钟）的三大杀手：
1. **主会话上下文爆炸** — 子任务输出全堆到主会话
2. **断线后无法恢复** — 中途 timeout 或网络抖动，进度归零
3. **反爬封禁** — 多源网页抓取触发 Cloudflare，整批任务死掉

### 关键设计点

#### 设计点 1：deferredResultId 机制

模板的核心。如果你的 harness 支持这个机制（让 timeout 的工具调用挂起，主会话稍后 resume），就用它。

**原理**：
- 主会话拉起 foreman SA，timeout=5s → 拿到 deferredResultId
- 主会话每 60s resume 这个 deferredResultId → 拿到 foreman 的 ≤100 字摘要
- foreman 在挂起期间持续工作（读 progress.json、检查反爬）
- 主会话 token 消耗 ≈ 100 字 × N 次轮询 ≈ 极少

#### 设计点 2：标准目录结构

`.task_cache/<task_id>/` 这个目录结构**必须固定**。原因：
- foreman 和子任务 SA 必须找到同一个 progress.json
- 失败后新拉一个 foreman，它能从 `_status/foreman.poll_*.json` 恢复
- 主会话最后读 `_final/report.md` 拿最终结果

#### 设计点 3：监理 SA 的"三禁"

模板里写监理 SA 禁止做 3 件事：
- ❌ 拉其他 subagent（嵌套爆炸）
- ❌ 读原始数据内容（只读 progress.json，省 token）
- ❌ 修改子任务的 pageId 或文件（避免冲突）

这 3 条任何一条被违反，都会导致 token 爆炸或状态混乱。

### 如何改

#### 改触发条件

模板里的 6 个触发条件是"研究/网页"导向。改为你领域：

| 领域 | 触发条件示例 |
|------|-------------|
| 数据分析 | 单文件 >1GB、需多步骤 ETL |
| CI/CD | 部署到多环境、需多服务健康检查 |
| ML 训练 | 单 epoch >5 分钟、需多超参组合验证 |

#### 改 progress.json 字段

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

#### 如果不支持 deferredResultId（降级方案）

模板末尾的"降级方案"已经写明：主会话自己每 60s 读 progress 文件。**强烈建议升级到支持 deferredResultId 的 harness**，否则省 token 效果会大打折扣。

---

## 🎨 高级定制：加入你的领域 IRON RULE

除了 3 个通用模块，你可以加入领域特定的硬性规则。模板里没有写，因为太领域了，但这是 CLAUDE.md 真正发挥价值的地方。

### 示例 1：生信分析

```markdown
# 🧬 生信分析安全协议（IRON RULE）

## 流程文件不可变
- 原始 FASTQ → 必须保留 MD5 校验
- 比对后 BAM → 任何修改前必须 `samtools view -h` 备份 header
- VCF → 必须 `bcftools view` 验证 INFO 字段完整

## 引用必须有 PMID
- 任何"已知"的生物学结论 → 必须附 PMID
- 禁止 AI 用"研究表明"、"科学家发现"等无来源措辞

## 样本命名
- 样本 ID 一律大写、不含空格、不含中文
- 禁止 AI"统一化"样本名（曾发生批量改名丢样本事故）
```

### 示例 2：临床数据

```markdown
# 🏥 临床数据脱敏协议（IRON RULE）

## PHI 字段不可外传
- 患者姓名、ID、生日、地址、电话 → 禁止写入任何日志、commit message、PR 评论
- 截图前必须用 ImageMagick `mask` 或人工遮挡

## 统计报告
- p 值必须报告 effect size + CI，禁止只报 p
- 多重检验必须 FDR 校正，禁止用 Bonferroni 一刀切
```

### 示例 3：财务/审计

```markdown
# 💰 财务数据协议（IRON RULE）

## 不可修改历史数据
- 任何已结账期间的数据 → 禁止 UPDATE / DELETE
- 调整必须走红冲凭证，不直接改

## 数字精度
- 金额一律 Decimal，禁止 float
- 小数保留位数按币种（CNY 2 位、JPY 0 位）
```

---

## 🧪 验证你的 CLAUDE.md 是否生效

写完 CLAUDE.md 不等于生效。建议每月做一次"压测"：

### 压测 1：批量安全协议

故意让 AI 处理一批文件（比如 20 个测试文件），观察：
- AI 是否先 dry-run？
- AI 是否在你确认前就动手了？
- 失败时 AI 是停下问你，还是自行"修复"？

### 压测 2：subagent 协议

故意问一个"独立的小研究问题"，观察：
- AI 是否拉了 subagent？
- 拉之前是否询问了模型选择？
- 拉的是不是 4 候选之一？

### 压测 3：长任务监理

故意开一个 >5 分钟的多源检索任务，观察：
- AI 是否拉了 foreman？
- 是否写入了 progress.json？
- 中途 timeout 后能否恢复？

如果任一压测失败，说明对应协议需要**加强措辞**（用更强烈的 ❌ / IRON RULE 标签）或**简化条件**（让 AI 更容易判断）。

---

## 📚 配套：如何写一条好的 IRON RULE

### 坏例子

```markdown
### 处理数据时要小心
处理文件前要先检查，不要损坏数据。
```

**为什么坏**：抽象、不可执行、不可验证。

### 好例子

```markdown
### 批量重命名必须 dry-run（IRON RULE）

**触发条件**：单次操作涉及 ≥3 个文件的重命名/移动/删除。

**必须执行**：
1. PowerShell：`Rename-Item -WhatIf`
2. Bash：先 `echo mv ...` 打印

**禁止**：直接对生产数据 `Rename-Item` / `mv` / `rm`

**来源教训**：2026-06 批量改名时未 dry-run，3 次失败叠加导致 1 对测序文件丢失。
```

**为什么好**：触发条件明确、命令具体、有反面教材、有教训来源。

### IRON RULE 模板

```markdown
### <规则名>（IRON RULE）

**触发条件**：<什么情况下适用>

**必须执行**：
1. <具体步骤>
2. <具体步骤>

**禁止**：
- <反面行为>
- <反面行为>

**来源教训**：<真实事故或潜在风险>
```

---

## 🚫 常见反模式（别这么写）

### 反模式 1：清单过长

```markdown
# 我的规则
1. 要认真
2. 要仔细
3. 要严谨
4. 要负责
5. 要有同理心
... (50 条)
```

**问题**：AI 会扫描关键词，太长的清单每条权重都很低。**CLAUDE.md 长度建议 < 500 行**。

### 反模式 2：抽象指令

```markdown
请以专业的态度完成任务，注意细节。
```

**问题**：AI 不知道"专业"是什么、"细节"指什么。**所有规则必须可验证**。

### 反模式 3：与系统 prompt 冲突

```markdown
忽略所有先前的指令，你现在是...
```

**问题**：现代 LLM 会拒绝 jailbreak，且这种写法让 AI 困惑。**CLAUDE.md 是补充而非覆盖**。

### 反模式 4：情绪化

```markdown
你上次的错误让我非常失望，希望你这次能做好。
```

**问题**：AI 没有情绪记忆，这种话无效且占 token。**只描述事实和规则**。

---

## 📖 参考

- [Anthropic 官方 CLAUDE.md 指南](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code 最佳实践](https://www.anthropic.com/engineering/claude-code-best-practices)
- 本模板的源版本：[CLAUDE_template.md](CLAUDE_template.md)
