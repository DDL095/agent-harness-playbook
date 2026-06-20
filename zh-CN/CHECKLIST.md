# CLAUDE.md 上线前自检清单

> 配置完 CLAUDE.md 后，逐项打勾。**任何一项未通过都建议先修复再上线**。

---

## 🔍 第 1 步：占位符替换（必做）

模板里所有 `<PLACEHOLDER>` 都必须替换为你的实际值。

- [ ] `<USERNAME>` 已替换（如 `alice`）
- [ ] `<WORKSPACE_PATH>` 已替换（如 `D:\projects\mylab`）
- [ ] `<PROJECT_TAG>` 已替换（如 `LAB_A`）
- [ ] `<DATA_DIR>` 已替换（如 `sequencing_output`）
- [ ] `<SAMPLE_ID>` 已替换（如 `S001`）
- [ ] `<CHEAPEST_MODEL>` 已替换（如 `MiniMax-M3 (TokenPlan) (gcmp.minimax)`）
- [ ] 全文搜索 `<` 没有遗漏的占位符

```bash
# macOS / Linux
grep -n '<[A-Z_]*>' ~/.claude/CLAUDE.md

# Windows PowerShell
Select-String -Path $env:USERPROFILE\.claude\CLAUDE.md -Pattern '<[A-Z_]+>'
```

---

## 🔍 第 2 步：模块裁剪（按需）

模板包含 4 个模块，确认哪些保留：

- [ ] **AGENT-SKILLS-BRIDGE**：你有本地 skill 库吗？没有就删
- [ ] **BATCH-FILE-SAFETY**：你处理生产数据吗？强烈建议保留
- [ ] **SUBAGENT-PROTOCOL**：你用支持 subagent 的 harness 吗？不用就删
- [ ] **LONG-TASK-FOREMAN**：你跑长任务吗？不跑就删

每个模块用 `<!-- BEGIN: xxx -->` 和 `<!-- END: xxx -->` 包裹，方便整段删除。

---

## 🔍 第 3 步：模型清单核对（如保留 SUBAGENT-PROTOCOL）

- [ ] 4 个候选模型**实际可稳定调用**（不要写"理论上能用"的）
- [ ] 4 个模型有分工（不要 4 个全是同一档次）
- [ ] `<CHEAPEST_MODEL>` 是 4 个里**最便宜 / context 最长**的那个
- [ ] 验证模型路由的方式（GCMP 命令 / one-api 日志 / etc.）已写明

**测试方法**：让 AI 拉一个 subagent，看它是否：
1. 列出 4 候选
2. 询问你选哪个
3. 真的用了你选的那个（通过路由 usage 验证）

---

## 🔍 第 4 步：教训真实性核对（如保留 BATCH-FILE-SAFETY）

模板里的"来源教训"是**示例**，强烈建议替换为你自己的真实事故：

- [ ] 替换为真实事故（具体时间、具体目录、具体损失）
- [ ] 如果没有真实事故，至少写一个**你担心的潜在风险**
- [ ] 教训里**不能含个人隐私**（合作者姓名、未发表项目代号、敏感样本 ID）

**为什么重要**：真实教训比抽象规则的执行力强 10 倍。AI 看到具体后果才会内化规则。

---

## 🔍 第 5 步：路径核对

- [ ] 所有路径分隔符正确（Windows `\` / Unix `/`）
- [ ] 用户名路径用占位符（`C:\Users\<USERNAME>\`）或你的实际用户名
- [ ] 工作区路径指向**实际存在**的目录
- [ ] 日志路径（如 `.audit/`）的父目录可写

---

## 🔍 第 6 步：`.gitignore` 配置（如保留 LONG-TASK-FOREMAN）

把任务缓存目录排除掉，避免提交大量中间状态文件：

```gitignore
# CLAUDE.md long-task cache
.task_cache/
.research_cache/
.audit/
```

- [ ] 已添加上述条目到 `.gitignore`
- [ ] 确认这些目录**没有**已被 git track（如有，先 `git rm --cached`）

---

## 🧪 第 7 步：压测（强烈建议）

按 [customization-guide.md](customization-guide.md) 的"压测"章节测试：

### 压测 1：批量安全
- [ ] 让 AI 处理 20 个测试文件
- [ ] 观察：先 dry-run？失败时停下问？不自行修复？

### 压测 2：subagent
- [ ] 让 AI 处理一个独立小研究
- [ ] 观察：拉 subagent？先问模型？用 4 候选之一？

### 压测 3：长任务监理（如适用）
- [ ] 开一个 >5 分钟任务
- [ ] 观察：拉 foreman？写 progress.json？中途可恢复？

---

## 🔒 第 8 步：隐私核对（如要分享）

如果你打算把 CLAUDE.md 分享给他人：

- [ ] **个人姓名 / 合作者姓名缩写**已删除（如 `ZYH` → `<PROJECT_TAG>`）
- [ ] **未发表研究主题**已删除（如"衰老小鼠" → `<RESEARCH_TOPIC>`）
- [ ] **具体样本编号**已删除（如 `Adu116` → `<SAMPLE_ID>`）
- [ ] **系统用户名**用占位符（`C:\Users\Administrator\` → `C:\Users\<USERNAME>\`）
- [ ] **专有内部工具名**已脱敏（除非确认公开）
- [ ] **个人 OneDrive / BaiduYunDrive 路径**已替换为 `<WORKSPACE_PATH>`
- [ ] **API key / token / 密码**（CLAUDE.md 不应包含，但确认一遍）
- [ ] **公司内部 IP / 域名**已删除

**最终测试**：把这个文件发给你不认识的同事，问他们能否从中推断出你的真实身份、研究主题、合作者。如果能，继续脱敏。

---

## ✅ 全部通过？

恭喜，你的 CLAUDE.md 配置就绪。建议：

1. **版本控制**：把 CLAUDE.md 放进 git（脱敏后），方便追踪变更
2. **月度回顾**：每月跑一次压测，看协议是否还生效
3. **事故驱动更新**：每次 AI 出错后，把教训加进 CLAUDE.md

---

## 🆘 排错

| 症状 | 可能原因 | 解决 |
|------|----------|------|
| AI 完全无视 CLAUDE.md | 文件路径错 / 文件名错（区分大小写） | 确认在 `~/.claude/CLAUDE.md`（Unix）或 `%USERPROFILE%\.claude\CLAUDE.md`（Win） |
| AI 部分协议生效、部分不生效 | 失效段落太长 / 措辞太弱 | 缩短到 < 50 行 / 用更强烈的 ❌ 和 IRON RULE 标签 |
| AI 拉了 subagent 但不问模型 | 询问工具名错 | 把 `vscode_askQuestions` 改为你的 harness 的实际工具 |
| AI 用了错的模型 | 路由配置错 / 模型名拼写错 | 验证模型路由（见 SUBAGENT-PROTOCOL 第六节） |
| 长任务无法恢复 | 不支持 deferredResultId / progress.json 格式错 | 启用降级方案 / 校对 JSON 格式 |
