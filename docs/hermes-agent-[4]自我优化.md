# Hermes Agent 自我进化技术白皮书

> 版本：1.2 | 日期：2026-04-20

---

## 概述

Hermes 的自我进化不是一个单独的功能模块，而是一个**贯穿每次会话的闭合反馈循环**，由四个机制共同驱动，各自在不同时机触发：

| 机制 | 触发时机 | 持久化介质 |
|------|---------|-----------|
| 定期自醒 | 每 N 轮对话后 / context 压缩前 / session 过期时（代码强制） | `MEMORY.md` / `USER.md` / `SKILL.md` |
| 技能创建 | 复杂任务完成后 / 用户指令 | `~/.hermes/skills/{name}/SKILL.md` |
| 技能自我优化 | 每次调用技能时即时发现缺陷 | 就地 patch SKILL.md |
| 跨会话搜索与摘要 | 感知到历史相关性时主动触发 | SQLite FTS5 (`state.db`) + LLM 摘要 |

这四个机制共享同一个底层数据层（`state.db` + 文件系统），并通过系统提示词中的行为指令统一驱动，形成一个自增强的认知飞轮。

---

## 一、定期自醒（Periodic Memory Review）

### 1.1 设计意图

定期自醒是**代码层面强制执行**的记忆巩固机制，不依赖 LLM 的指令遵从意愿。Agent 在特定时机被系统强制唤起，审视刚刚发生的对话，自主判断什么值得写入持久记忆，什么可以丢弃。

这模拟了人类大脑在任务间隙对短期记忆进行筛选和固化的过程。

### 1.2 三个触发时机

| 触发点 | 触发条件 | 执行方式 | 工具集 | session 文件 | 代码位置 |
|--------|---------|---------|-------|-------------|---------|
| **轮次 nudge** | 每 N 次工具调用后（默认 10），当轮回复完成后触发 | fork 独立 `AIAgent`，后台线程异步 | 完整工具集 | 生成新文件 | `run_agent.py:8376` |
| **context 压缩前** | 上下文窗口接近上限 | 主 agent 内部直接一次裸 API 调用 | 仅 `memory` | 不生成 | `run_agent.py:7040` |
| **session 过期时** | 空闲 24h 或每日凌晨 4 点，后台每 5 分钟轮询 | fork `tmp_agent`，工具受限 | `memory` + `skills` | 生成新文件 | `gateway/run.py:2046` |

### 1.3 轮次 nudge（最常见路径）

- **执行方式**：fork 独立 `AIAgent`（`quiet_mode=True`，stdout/stderr 重定向至 `/dev/null`），后台线程异步运行，不阻塞用户
- **工具集**：完整工具集（含 `memory` + `skill_manage`）
- **消息来源**：内存中的消息快照（`messages_snapshot`）
- **session 文件**：生成新文件，格式与主对话相同；内容为主对话全量历史（复制）+ review prompt + review agent 回复
- **结果通知**：若有写入，向主会话发简短通知（`"💾 Memory updated"` 等）
- **防重复**：无（每次达到阈值就触发）

**触发计数**：

```python
# run_agent.py:8376 — 每轮入口计数
self._turns_since_memory += 1
if self._turns_since_memory >= self._memory_nudge_interval:  # 默认 10
    _should_review_memory = True
    self._turns_since_memory = 0
```

**Prompt**：

```
Review the conversation above and consider two things:

**Memory**: Has the user revealed things about themselves — their persona,
desires, preferences, or personal details? Has the user expressed expectations
about how you should behave, their work style, or ways they want you to operate?
If so, save using the memory tool.

**Skills**: Was a non-trivial approach used to complete a task that required trial
and error, or changing course due to experiential findings along the way, or did
the user expect or desire a different method or outcome? If a relevant skill
already exists, update it. Otherwise, create a new one if the approach is reusable.

Only act if there's something genuinely worth saving.
If nothing stands out, just say 'Nothing to save.' and stop.
```

**间隔可配置**（`agent_config.yaml`）：
```yaml
memory:
  nudge_interval: 10           # 每 10 轮触发记忆审查
skills:
  creation_nudge_interval: 10  # 每 10 次工具调用后触发技能审查
```

### 1.4 context 压缩前 flush

- **执行方式**：不 fork，主 agent 内部直接通过 `auxiliary_client.call_llm()` 发一次裸 API 调用（优先走辅助模型，更便宜）
- **工具集**：仅 `memory`
- **消息来源**：当前主对话消息列表（原地操作）
- **session 文件**：不生成
- **结果通知**：无
- **防重复**：flush 消息写入后立即剥除，不留痕迹

**Prompt**：

```
[System: The session is being compressed.
Save anything worth remembering — prioritize user preferences,
corrections, and recurring patterns over task-specific details.]
```

### 1.5 session 过期 flush

- **执行方式**：fork `tmp_agent`（`AIAgent` 实例，`skip_memory=True`，`max_iterations=8`），由 `_session_expiry_watcher()` 每 5 分钟轮询触发
- **工具集**：`memory` + `skills`
- **消息来源**：从 SQLite 重新加载的历史记录（`session_store.load_transcript()`）
- **session 文件**：生成新文件
- **结果通知**：无
- **防重复**：flush 完成后 `memory_flushed=True` 写入 `sessions.json`，网关重启后不重复触发

**触发条件**（默认 `mode="both"`，两个条件先到先得）：
- 空闲超过 `idle_minutes`（默认 1440 分钟 = 24h）
- 最后活动时间早于今日 `at_hour` 整点（默认凌晨 4 点）

**Prompt**（额外附上当前 `MEMORY.md` + `USER.md` 内容，防止覆盖已有条目）：

```
[System: This session is about to be automatically reset ...
Review the conversation above and:
1. Save any important facts, preferences, or decisions to memory ...
2. If you discovered a reusable workflow ..., consider saving it as a skill.
3. If nothing is worth saving, that's fine — just skip.
Do NOT respond to the user. Just use the memory and skill_manage tools if needed, then stop.]
```

### 1.6 系统提示中的常驻自醒指令

除代码层面的强制触发外，Agent 每次推理时也会在系统提示中看到两段常驻行为指令（`agent/prompt_builder.py:144`），形成**内嵌于每次推理的反思底色**：

**`MEMORY_GUIDANCE`**（记忆写入指南）：
```
你拥有跨会话的持久记忆。使用 memory 工具保存持久事实：用户偏好、
环境细节、工具怪癖、稳定约定。优先保存能减少未来用户纠正成本的内容——
最有价值的记忆是让用户不必再次提醒你的那种。
如果你发现了做某事的新方法，将其保存为技能。
```

**`SKILLS_GUIDANCE`**（技能反思指南）：
```
完成复杂任务（5+ 次工具调用）、修复棘手错误、发现非平凡工作流后，
使用 skill_manage 将方法保存为技能。
使用技能时发现其过时、不完整或错误，立即用 skill_manage(action='patch') 修复——
不要等待被要求。未被维护的技能会成为负担。
```

---

## 二、技能自动生成（Skill Auto-Generation）

### 2.1 技能的本质

技能是 Agent 的**程序性记忆**——将曾经历过的复杂工作流、踩过的坑、有效的策略，编码为结构化的 Markdown 文件，供未来会话直接复用。

技能的生成有两条路径：**指令驱动**（Agent 在推理中主动判断）和**后台强制审查**（代码定期触发）。两者共同保证技能库持续积累。

### 2.2 存储结构

```
~/.hermes/skills/
└── {skill-name}/
    ├── SKILL.md          ← 主文件：YAML frontmatter + 指令正文
    ├── references/       ← 参考文档
    ├── templates/        ← 模板文件
    ├── scripts/          ← 辅助脚本
    └── assets/           ← 其他资产
```

SKILL.md frontmatter 示例：
```yaml
---
name: deploy-k8s-rollout
description: Kubernetes rolling deployment with health check gates
version: 1.2.0
---
# 指令正文...
```

### 2.3 路径一：指令驱动（主动生成）

系统通过三处重叠的提示指令，在 Agent 的每次推理中埋入技能创建意识：

**① `SKILLS_GUIDANCE`**（系统提示常驻，`agent/prompt_builder.py:164`）：
```
After completing a complex task (5+ tool calls), fixing a tricky error,
or discovering a non-trivial workflow, save the approach as a skill with
skill_manage so you can reuse it next time.
When using a skill and finding it outdated, incomplete, or wrong,
patch it immediately with skill_manage(action='patch') — don't wait to be asked.
Skills that aren't maintained become liabilities.
```

**② `SKILL_MANAGE_SCHEMA` description**（工具说明，`tools/skill_manager_tool.py:681`）：

Agent 看到的 `skill_manage` 工具说明中明确列出触发条件：

| 条件 | 说明 |
|------|------|
| 复杂任务成功 | 5+ 次工具调用 |
| 克服了错误 | 试错后找到解决路径 |
| 用户纠正生效 | 用户指正后的正确方法 |
| 非平凡工作流 | 发现可复用的操作序列 |
| 用户明确要求 | 用户让 Agent 记住某个步骤 |

工具说明还包含：`"After difficult/iterative tasks, offer to save as a skill."`——要求 Agent 在任务结束时主动提议保存。

**③ 技能索引头部**（`agent/prompt_builder.py:790`）：
```
After difficult/iterative tasks, offer to save as a skill.
If a skill you loaded was missing steps, had wrong commands, or needed
pitfalls you discovered, update it before finishing.
```

三处指令形成**递进的行为契约**：系统提示说"你应该"，工具说明说"什么时候该"，索引头部说"任务结束前检查"。

### 2.4 路径二：后台强制审查（被动生成）

后台审查是定期自醒机制的一部分（详见第一章），在不依赖 LLM 主动意愿的情况下强制触发技能回顾。

**触发条件**：`_iters_since_skill`（每次 API 调用递增）达到 `skill_nudge_interval`（默认 10）。

```python
# run_agent.py:8655 — 每次 API 调用（iteration）递增
if self._skill_nudge_interval > 0 and "skill_manage" in self.valid_tool_names:
    self._iters_since_skill += 1

# run_agent.py:11319 — 当轮回复完成后检查
if self._iters_since_skill >= self._skill_nudge_interval:
    _should_review_skills = True
    self._iters_since_skill = 0
```

注意：计数器按**工具调用次数（API iterations）**递增，而非对话轮次——复杂任务（多次工具调用）比简单对答触发更快。调用 `skill_manage` 时计数器重置为 0。

触发后，`_spawn_background_review()` fork 独立 Agent，注入以下 prompt：

```
Review the conversation above and consider saving or updating a skill if appropriate.

Focus on: was a non-trivial approach used to complete a task that required trial
and error, or changing course due to experiential findings along the way, or did
the user expect or desire a different method or outcome?

If a relevant skill already exists, update it with what you learned.
Otherwise, create a new skill if the approach is reusable.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

后台 Agent 拥有完整工具集（含 `skill_manage`），可直接创建或更新技能，完成后向主会话发送简短通知（`"💾 Skill '...' created"`）。

### 2.5 创建流程（`tools/skill_manager_tool.py`）

`skill_manage(action="create")` 执行以下步骤：

```
_create_skill()
├── 名称校验：^[a-z0-9][a-z0-9._-]*$，最长 64 字符
├── YAML frontmatter 校验：必须含 name + description，正文非空，≤100,000 字符
├── 跨目录冲突检测（本地 + 外部 skills_dirs）
├── 原子写入：temp file + os.replace()（崩溃安全）
├── 安全扫描（skills_guard.py）：提示注入、数据渗漏命令检测，失败则回滚
└── 清除系统提示缓存 → 新技能在下一轮立即生效
```

**安全扫描**对 Agent 自创技能与社区安装技能一视同仁，防止恶意内容注入系统提示。

### 2.6 技能在系统提示中的展现

每次会话，`build_skills_system_prompt()`（`agent/prompt_builder.py:583`）构建技能索引，注入系统提示：

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially
relevant to your task, you MUST load it with skill_view(name) and follow its
instructions. Err on the side of loading — it is always better to have context
you don't need than to miss critical steps, pitfalls, or established workflows.
<available_skills>
  category:
    - skill-name: description
  ...
</available_skills>
Only proceed without loading a skill if genuinely none are relevant to the task.
```

**缓存策略**：双层缓存避免重复构建：
1. 进程内 LRU 缓存（缓存键：`(skills_dir, tools, toolsets, platform)`）
2. 磁盘快照 `~/.hermes/.skills_prompt_snapshot.json`（以文件 mtime/size manifest 校验有效性）

新技能创建后立即清除两层缓存，下一轮对话即生效。

---

## 三、技能自我优化（Skill Self-Optimization）

### 3.1 设计理念

技能不是静态文档，而是**活文档**。每次 Agent 执行任务时，都同时扮演两个角色：**执行者**和**质量审查者**。质量审查不是任务结束后的独立步骤，而是内嵌于执行过程本身——Agent 被要求在执行完成前就完成修复，而不是事后再回来。

这个设计的关键假设是：**发现缺陷的最佳时机就是踩到坑的那一刻**，此时上下文最完整，修复最精准。

### 3.2 Agent 如何认识到技能需要优化

**没有独立的评估阶段，没有评分机制，不做事前检查。** 识别完全是被动的、即时的，发生在执行过程中。

**认知过程**：

Agent 加载技能后，SKILL.md 的完整内容就在它的 context 里。之后 Agent 一边执行任务，一边隐式地将"自己实际在做什么"与"技能说要做什么"进行对照。只要出现以下任一情况，偏差就产生了：

| 偏差类型 | 具体表现 |
|---------|---------|
| 技能步骤失败 | 按技能命令执行，返回错误或非预期结果 |
| 需要额外步骤 | 技能没提到，但不做不行（如等待异步操作完成） |
| 踩到未预警的坑 | 技能没有警告，但执行中撞上了已知陷阱 |
| 环境不匹配 | 技能的命令在当前 OS/版本下不适用 |

**关键点**：这个对照不是显式的"检查步骤"，而是 LLM 推理的自然产物。Agent 在 context 里同时持有技能文本和执行结果，当两者出现张力时，模型自然会形成"技能说 X，但我实际做了 Y，原因是 Z"这样的内部判断。

**不触发的情况**：如果技能步骤全部顺利执行、结果符合预期，Agent 不会去"主动检查技能质量"——什么都不会发生。优化是**纯被动的**，只在实际遇到问题时触发。

### 3.3 优化点是如何找到的

Agent 看到的技能是完整的 SKILL.md 原文（通过 `skill_view()` 加载），包括 frontmatter 和正文。没有质量评分、没有过期标记、没有结构化的自评字段——技能是纯粹的指令文档。

优化点的定位完全依赖 Agent 在执行中的**对照比较**：

```
技能说：执行步骤 A → B → C
Agent 实际执行：A → B → 发现 B 返回错误 → 额外诊断 → 修复 → C
                              ↑
                         这个"额外诊断+修复"就是缺失的步骤
```

Agent 被明确要求在任务完成**之前**（not after）完成修复：

> "If a skill you loaded was missing steps, had wrong commands, or needed pitfalls you discovered, **update it before finishing**."

"before finishing"是关键词——patch 是当前任务的最后一步，而不是下次再说。

### 3.4 三层重叠的行为指令

系统通过三处位置反复强化这个行为，形成多层保障：

**① 系统提示 `SKILLS_GUIDANCE`**（每次推理都存在）：
```
When using a skill and finding it outdated, incomplete, or wrong,
patch it immediately with skill_manage(action='patch') — don't wait to be asked.
Skills that aren't maintained become liabilities.
```

**② 技能索引头部**（每次推理都存在，Agent 看到技能列表前先看到这段）：
```
If a skill has issues, fix it with skill_manage(action='patch').
After difficult/iterative tasks, offer to save as a skill.
If a skill you loaded was missing steps, had wrong commands, or needed
pitfalls you discovered, update it before finishing.
```

**③ `skill_manage` 工具说明**（Agent 调用工具时可见）：
```
Update when: instructions stale/wrong, OS-specific failures,
missing steps or pitfalls found during use.
If you used a skill and hit issues not covered by it, patch it immediately.
```

三处指令都没有"等用户提醒"的选项——**立即修复**是唯一被允许的响应。

### 3.5 优化的粒度：patch vs edit

Agent 根据修改范围选择操作方式：

- **`patch`**（首选）：定向替换，用 `old_string` / `new_string` 精准修改某一处。适合补充遗漏步骤、修正命令、添加 pitfall 警告。
- **`edit`**（大规模重构）：完整重写整个 SKILL.md。适合技能结构性过时，需要从头重组。

`patch` 优先的原因：技能的其他部分可能仍然有效，大范围重写有丢失已有知识的风险。

### 3.6 技能文件本身的结构约定

虽然 SKILL.md 没有自评字段，但工具说明给出了"好技能"的结构规范，隐性地告诉 Agent 什么是"缺失"的：

```
Good skills: trigger conditions, numbered steps with exact commands,
pitfalls section, verification steps.
```

当 Agent 发现一个技能没有 `## Pitfalls` 小节但执行中踩了坑，就会识别为"缺失 pitfalls section"并补充。结构规范成为了隐式的质量检查清单。

### 3.7 完整行为模式示例

```
1. Agent 收到任务，扫描技能索引，发现 "deploy-k8s-rollout" 相关
        ↓
2. skill_view("deploy-k8s-rollout") 加载完整 SKILL.md
        ↓
3. 按技能步骤执行：A → B → C
   执行到 B 时：kubectl rollout 命令返回，但没有等待完成就进入下一步
   结果：C 步骤在 rollout 未完成时失败
        ↓
4. Agent 诊断：技能步骤 B 后缺少"等待 rollout 完成"的验证步骤
        ↓
5. 任务完成前，主动调用：
   skill_manage(action="patch", name="deploy-k8s-rollout",
                old_string="3. 执行 kubectl rollout apply...",
                new_string="3. 执行 kubectl rollout apply...\n4. 等待完成：kubectl rollout status deploy/xxx")
        ↓
6. 缓存清除，技能在下一次会话即生效
```

全程无需用户介入，技能质量随每次使用单向递增。

---

## 四、跨会话搜索与摘要（Cross-Session Search & Summarization）

### 4.1 核心问题

LLM 的上下文窗口是有限的，历史会话记录不可能全部塞入当前上下文。`session_search` 解决的问题是：**在需要时，以最低成本从所有历史会话中精准召回相关信息**。

### 4.2 存储层：SQLite + FTS5（`hermes_state.py`）

数据库位于 `~/.hermes/state.db`（schema version 6），启用 WAL 模式支持多写者并发（gateway + CLI + cron 进程共享同一数据库）。

**核心表结构**：

```sql
sessions (
    id, source, user_id, model, system_prompt,
    parent_session_id,   -- 上下文压缩链：子会话指向父会话
    started_at, ended_at,
    input_tokens, output_tokens, cost_usd,
    title
)

messages (
    session_id, role, content,
    tool_calls, tool_name,
    timestamp, reasoning, reasoning_details
)

-- FTS5 全文索引（INSERT/UPDATE/DELETE 触发器自动维护）
messages_fts USING fts5(content, role, tool_name, session_id UNINDEXED)
```

**`parent_session_id` 链**：当上下文被压缩时，新的"子会话"创建并指向父会话，保持历史可追溯。`session_search` 通过 `_resolve_to_parent()` 自动将子会话归并到其根父会话，确保委派（delegation）任务的输出归属原始对话。

**并发安全**：20–150ms 抖动重试（最多 15 次），处理多进程写冲突。

### 4.3 搜索流程（`tools/session_search_tool.py:297`）

```
session_search(query="上周的 k8s 部署问题")
    │
    ├── [无 query] → 直接返回最近会话列表（零 LLM 成本）
    │
    └── [有 query] →
        ├── 1. FTS5 MATCH 查询（sanitized 输入）
        │      → 最多 50 条原始匹配
        │
        ├── 2. 按会话分组，取 top-N 会话（默认 3，最大 5）
        │
        ├── 3. _resolve_to_parent()：子会话溯源到根父会话
        │      排除当前会话谱系（Agent 已有该上下文）
        │
        ├── 4. 获取完整对话记录 → _truncate_around_matches()
        │      以匹配位置为中心，窗口化到 100k 字符
        │
        └── 5. asyncio.gather() 并行摘要
               _summarize_session() × N
               → async_call_llm()（auxiliary 模型，如 Gemini Flash）
               → 摘要提示：关注问题/行动/结论/遗留事项
```

### 4.4 FTS5 查询安全化（`hermes_state.py:938`）

`_sanitize_fts5_query()` 防止恶意或格式错误的查询破坏 FTS5 解析器：

```python
步骤：
1. 提取平衡引号短语，替换为编号占位符
2. 剥离 FTS5 特殊字符 +{}()"^
3. 折叠 ** 重复
4. 移除首尾悬空的布尔运算符 AND/OR/NOT
5. 带点或带连字符的词汇加引号（"chat-send" 防止 tokenizer 拆分）
6. 恢复引号占位符
```

支持的查询语法：
- 关键词：`deploy kubernetes`
- 精确短语：`"rolling update"`
- 布尔：`deploy AND NOT staging`
- 前缀：`deploy*`

### 4.5 并行摘要与辅助模型

摘要调用 `agent/auxiliary_client.py` 的 `async_call_llm()`，这是一个**与主 Agent 解耦的独立 LLM 客户端**，可配置使用更廉价/更快的模型（如 Gemini Flash）专门处理摘要任务，控制成本。

摘要 prompt 的指令：
```
聚焦于：用户提出了什么问题、采取了哪些行动、结果是什么、
关键命令/文件/URL、未解决的遗留问题。
```

### 4.6 系统提示中的主动召回指令

`SESSION_SEARCH_GUIDANCE`（`agent/prompt_builder.py:158`）：
```
当用户引用过去对话中的内容，或你怀疑存在相关的跨会话上下文时，
在要求用户重复之前，先用 session_search 召回。
```

`SESSION_SEARCH_SCHEMA` description 中列出的触发短语：
- "我们之前做过这个"
- "还记得吗"
- "上次"
- "就像我提到的"

---

## 五、整体架构：闭合反馈循环

### 5.1 数据流全景

```
用户消息
    │
    ▼
MemoryManager.prefetch_all()          ← 外部记忆插件：Honcho/Mem0/Hindsight/...
    │（注入当前轮次的记忆上下文）
    ▼
系统提示（每会话构建一次，LRU 缓存）：
  ┌─────────────────────────────────────────┐
  │ DEFAULT_AGENT_IDENTITY                  │
  │ MEMORY_GUIDANCE       ← 记忆写入指令    │
  │ SESSION_SEARCH_GUIDANCE ← 跨会话指令   │
  │ SKILLS_GUIDANCE       ← 技能创建指令   │
  │ 技能索引（所有 SKILL.md frontmatter）   │
  │ MEMORY.md 快照（Agent 个人笔记）        │
  │ USER.md 快照（用户画像）                │
  └─────────────────────────────────────────┘
    │
    ▼
Agent 推理 → 可能调用：
  ├── memory(add/replace/remove)     → MEMORY.md / USER.md
  ├── session_search(query)          → FTS5 → 并行 LLM 摘要 → 返回摘要
  └── skill_manage(create/patch)     → ~/.hermes/skills/{name}/SKILL.md
    │
    ▼
MemoryManager.sync_all()            → 外部记忆插件同步
SessionDB.append_message()          → state.db（FTS5 自动索引）
    │
    ├── 每 N 轮 → _spawn_background_review()    ← 定期自醒（轮次 nudge）
    │              → fork AIAgent → 审查对话 → 写入 MEMORY.md / SKILL.md
    │
    └── 每 5 分钟（后台）→ _session_expiry_watcher()  ← 定期自醒（过期 flush）
                           → 检测过期 session → flush memories
```

### 5.2 四机制的协同关系

```
         ┌────────────────────────────────────────────────┐
         │                   用户会话                      │
         │                                                │
         │  [技能加载] → 任务执行 → [技能 patch]          │
         │       ↑                        │               │
         │       └────── 技能自我优化 ────┘               │
         │                                                │
         │  [session_search] → 历史上下文召回              │
         │                                                │
         │  [memory write] → 持久化用户画像/偏好           │
         │                                                │
         │  每 10 轮 ──→ 定期自醒 ──→ 写入记忆/技能        │
         └──────────────────────────┬─────────────────────┘
                                    │ state.db（FTS5）
                                    ▼
                       ┌────────────────────────┐
                       │   定期自醒（过期 flush）  │
                       │  24h 空闲 / 凌晨 4 点   │
                       │  → MEMORY.md / SKILL.md │
                       └────────────────────────┘
```

### 5.3 记忆的分层架构

Hermes 将记忆分为三个层次，各自承担不同的认知功能：

| 层次 | 载体 | 内容类型 | 生命周期 |
|------|-----|---------|---------|
| **陈述性记忆** | `MEMORY.md` + `USER.md` | 用户偏好、环境配置、工具规律 | 永久（§ 分隔条目，手动管理） |
| **程序性记忆** | `~/.hermes/skills/*/SKILL.md` | 工作流步骤、命令序列、踩坑记录 | 永久（带版本，可 patch） |
| **情节性记忆** | `state.db` (FTS5) | 完整会话记录 | 永久（全文检索，LLM 摘要） |

外部记忆插件（Honcho、Mem0 等）在系统层面扩展了陈述性记忆的能力，支持向量检索、AI 对等自建模等高级功能。


## 附录：关键文件索引

| 组件 | 文件路径 |
|------|---------|
| 定期自醒（轮次 nudge + 后台 fork） | `run_agent.py`（L2340–L2480, L8366–L8383, L11315–L11345）|
| 定期自醒（context 压缩前 flush） | `run_agent.py`（L7040–L7200）|
| 定期自醒（session 过期 flush） | `gateway/run.py`（L743–L860, L2046–L2100）|
| Session 过期检测 | `gateway/session.py`（L582–L618）|
| 行为指令注入 | `agent/prompt_builder.py`（L144–L171, L777–L799）|
| 技能管理工具（创建/patch/edit） | `tools/skill_manager_tool.py`（L681 SKILL_MANAGE_SCHEMA, L_create_skill）|
| 技能后台审查 prompt | `run_agent.py`（L2351–L2373 _SKILL_REVIEW_PROMPT）|
| 技能 nudge 计数器与触发 | `run_agent.py`（L1330, L8655, L11319）|
| 技能索引构建与缓存 | `agent/prompt_builder.py`（L583–L808 build_skills_system_prompt）|
| 技能工具函数（文件解析、平台过滤） | `agent/skill_utils.py` |
| 跨会话搜索工具 | `tools/session_search_tool.py` |
| SQLite + FTS5 存储 | `hermes_state.py` |
| 记忆存储工具 | `tools/memory_tool.py` |
| 记忆管理器 | `agent/memory_manager.py` |
| 记忆提供者 ABC | `agent/memory_provider.py` |
| 辅助 LLM 客户端 | `agent/auxiliary_client.py` |
| 安全扫描 | `tools/skills_guard.py` |
| 模糊匹配引擎 | `tools/fuzzy_match.py` |
| 外部记忆插件 | `plugins/memory/{honcho,mem0,hindsight,...}/` |
