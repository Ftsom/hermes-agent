# Hermes Agent 自我进化技术白皮书

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

定期自醒是**代码层面强制执行**的记忆巩固机制，不依赖 LLM 的指令遵从意愿。Agent 在特定时机被系统强制唤起，审视刚刚发生的对话，自主判断什么值得写入持久记忆或技能库，什么可以丢弃。

这模拟了人类大脑在任务间隙对短期记忆进行筛选和固化的过程。

### 1.2 触发机制：双计数器

系统维护两个独立的计数器，分别追踪记忆和技能的审查时机：

| 计数器 | 递增维度 | 默认阈值 | 重置条件 | 写入目标 | 代码位置 |
|--------|---------|---------|---------|---------|---------|
| `_turns_since_memory` | 每个**用户对话轮次** +1 | 10 轮 | 达到阈值触发后重置；用户主动调用 memory 工具时重置 | `MEMORY.md`、`USER.md` | `run_agent.py:10931` |
| `_iters_since_skill` | 每次**工具调用迭代** +1 | 10 次 | 达到阈值触发后重置；用户主动调用 skill_manage 时重置 | `~/.hermes/skills/{name}/SKILL.md` 及其 `references/`、`templates/`、`scripts/` | `run_agent.py:11211` |

**关键区别**：memory 按对话轮次计数（简单对话也会触发），skill 按工具调用次数计数（复杂任务触发更快，纯对话不触发）。

**间隔可配置**（`config.yaml`）：
```yaml
memory:
  nudge_interval: 10           # 每 10 轮用户对话触发记忆审查
skills:
  creation_nudge_interval: 10  # 每 10 次工具调用迭代触发技能审查
```

### 1.3 触发判定流程

**Memory 判定**（轮次入口，`run_agent.py:10928-10934`）：

```python
# 每个用户轮次开始时计数
self._turns_since_memory += 1
if self._turns_since_memory >= self._memory_nudge_interval:  # 默认 10
    _should_review_memory = True
    self._turns_since_memory = 0
```

前提条件：`_memory_nudge_interval > 0` 且 `"memory" in self.valid_tool_names` 且 `self._memory_store` 存在。

**Skill 判定**（当轮回复完成后，`run_agent.py:14384-14389`）：

```python
# 当轮结束后检查本轮工具调用累计
if (self._skill_nudge_interval > 0
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True
    self._iters_since_skill = 0
```

### 1.4 执行时机与方式

后台回顾在**本轮响应交付之后**才启动（`run_agent.py:14398-14408`），确保不与用户当前任务竞争模型资源：

```python
if final_response and not interrupted and (_should_review_memory or _should_review_skills):
    self._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

**执行细节**（`_spawn_background_review`，`run_agent.py:3643`）：

- **线程隔离**：在独立后台线程中 fork 一个新 `AIAgent` 实例，不阻塞用户
- **工具集限定**：`enabled_toolsets=["memory", "skills"]`，只能操作记忆和技能库，不能执行任何终端命令
- **安全约束**：安装 `auto_deny` 回调，所有危险命令请求自动拒绝（防止 deadlock）
- **静默运行**：`suppress_status_output=True`，stdout/stderr 重定向到 `/dev/null`
- **运行时继承**：继承父 agent 的 model、provider、api_key、credential_pool、memory_store
- **最大迭代**：16 次工具调用
- **消息来源**：当前对话的内存快照（`messages_snapshot`），review prompt 作为 user turn 追加

### 1.5 Review Prompt 三档选择

根据触发条件组合，选择不同的审查指令：

| 条件 | Prompt 常量 | 核心关注点 |
|------|------------|-----------|
| 仅 memory 触发 | `_MEMORY_REVIEW_PROMPT` | 用户身份、偏好、行为期望 |
| 仅 skill 触发 | `_SKILL_REVIEW_PROMPT` | 技能更新（纠正、新技术、修补过时技能） |
| 两者同时触发 | `_COMBINED_REVIEW_PROMPT` | 合并处理：memory + skills |

**Memory Review Prompt**（`run_agent.py:3437`）：
```
Review the conversation above and consider saving to memory if appropriate.

Focus on:
1. Has the user revealed things about themselves — their persona, desires,
   preferences, or personal details worth remembering?
2. Has the user expressed expectations about how you should behave, their work
   style, or ways they want you to operate?

If something stands out, save it using the memory tool.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

**Skill Review Prompt**（`run_agent.py:3448`）：
```
Review the conversation above and update the skill library. Be ACTIVE — most
sessions produce at least one skill update, even if small. A pass that does
nothing is a missed learning opportunity, not a neutral outcome.

Target shape of the library: CLASS-LEVEL skills, each with a rich SKILL.md and
a `references/` directory for session-specific detail. Not a long flat list of
narrow one-session-one-skill entries. This shapes HOW you update, not WHETHER
you update.

Signals to look for (any one of these warrants action):
  • User corrected your style, tone, format, legibility, or verbosity.
    Frustration signals like 'stop doing X', 'this is too verbose', 'don't
    format like this', 'why are you explaining', 'just give me the answer',
    'you always do Y and I hate it', or an explicit 'remember this' are
    FIRST-CLASS skill signals, not just memory signals. Update the relevant
    skill(s) to embed the preference so the next session starts already knowing.
  • User corrected your workflow, approach, or sequence of steps. Encode the
    correction as a pitfall or explicit step in the skill that governs that
    class of task.
  • Non-trivial technique, fix, workaround, debugging path, or tool-usage
    pattern emerged that a future session would benefit from. Capture it.
  • A skill that got loaded or consulted this session turned out to be wrong,
    missing a step, or outdated. Patch it NOW.

Preference order — prefer the earliest action that fits, but do pick one when
a signal above fired:
  1. UPDATE A CURRENTLY-LOADED SKILL. Look back through the conversation for
     skills the user loaded via /skill-name or you read via skill_view. If any
     of them covers the territory of the new learning, PATCH that one first.
  2. UPDATE AN EXISTING UMBRELLA (via skills_list + skill_view). If no loaded
     skill fits but an existing class-level skill does, patch it.
  3. ADD A SUPPORT FILE under an existing umbrella. Skills can be packaged with
     three kinds of support files:
     • `references/<topic>.md` — session-specific detail AND condensed knowledge
       banks: quoted research, API docs, external authoritative excerpts.
     • `templates/<name>.<ext>` — starter files meant to be copied and modified.
     • `scripts/<name>.<ext>` — statically re-runnable actions the skill can
       invoke directly.
  4. CREATE A NEW CLASS-LEVEL UMBRELLA SKILL when no existing skill covers the
     class. The name MUST be at the class level — NOT a specific PR number,
     error string, feature codename, or 'fix-X / debug-Y' session artifact.

User-preference embedding (important): when the user expressed a
style/format/workflow preference, the update belongs in the SKILL.md body, not
just in memory. Memory captures 'who the user is'; skills capture 'how to do
this class of task for this user'.

'Nothing to save.' is a real option but should NOT be the default.
```

**Combined Review Prompt**（`run_agent.py:3524`）：
```
Review the conversation above and update two things:

**Memory**: who the user is. Did the user reveal persona, desires, preferences,
personal details, or expectations about how you should behave? Save facts about
the user and durable preferences with the memory tool.

**Skills**: how to do this class of task. Be ACTIVE — most sessions produce at
least one skill update. A pass that does nothing is a missed learning
opportunity, not a neutral outcome.

Target shape of the skill library: CLASS-LEVEL skills with a rich SKILL.md and
a `references/` directory for session-specific detail. Not a long flat list of
narrow one-session-one-skill entries.

Signals that warrant a skill update (any one is enough):
  • User corrected your style, tone, format, legibility, verbosity, or approach.
    Frustration is a FIRST-CLASS skill signal, not just a memory signal. 'stop
    doing X', 'don't format like this', 'I hate when you Y' — embed the lesson
    in the skill that governs that task so the next session starts fixed.
  • Non-trivial technique, fix, workaround, or debugging path emerged.
  • A skill that was loaded or consulted turned out wrong, missing, or outdated
    — patch it now.

Preference order for skills — pick the earliest that fits:
  1. UPDATE A CURRENTLY-LOADED SKILL. Check what skills were loaded via
     /skill-name or skill_view in the conversation. If one of them covers the
     learning, PATCH it first.
  2. UPDATE AN EXISTING UMBRELLA (skills_list + skill_view to find the right
     one). Patch it.
  3. ADD A SUPPORT FILE under an existing umbrella via skill_manage
     action=write_file. Three kinds: `references/<topic>.md` for session-specific
     detail OR condensed knowledge banks; `templates/<name>.<ext>` for starter
     files; `scripts/<name>.<ext>` for statically re-runnable actions.
  4. CREATE A NEW CLASS-LEVEL UMBRELLA when nothing exists. Name at the class
     level — NOT a PR number, error string, codename, or 'fix-X / debug-Y'
     session artifact.

User-preference embedding: when the user complains about how you handled a task,
update the skill that governs that task — memory alone isn't enough. Memory says
'who the user is'; skills say 'how to do this class of task for this user'. Both
should carry user-preference lessons when relevant.

Act on whichever of the two dimensions has real signal. If genuinely nothing
stands out on either, say 'Nothing to save.' and stop — but don't reach for
that conclusion as a default.
```

### 1.6 结果汇总与通知

回顾完成后，`_summarize_background_review_actions()`（`run_agent.py:3581`）扫描 review agent 的消息：

1. 提取所有成功的工具调用结果（`"success": true`）
2. 根据 message 内容分类：created / updated / added / removed
3. 去重：跳过 `messages_snapshot` 中已有的 tool 消息（通过 `tool_call_id` 或内容匹配）
4. 向用户展示精简摘要（如 "Memory updated"、"Skill 'xxx' created"）

### 1.7 系统提示中的常驻自醒指令

除代码层面的强制触发外，Agent 每次推理时也会在系统提示中看到两段常驻行为指令（`agent/prompt_builder.py`），形成**内嵌于每次推理的反思底色**：

**`MEMORY_GUIDANCE`**（`agent/prompt_builder.py:150`）：
```
You have persistent memory across sessions. Save durable facts using the memory
tool: user preferences, environment details, tool quirks, and stable conventions.
Memory is injected into every turn, so keep it compact and focused on facts that
will still matter later.
Prioritize what reduces future user steering — the most valuable memory is one
that prevents the user from having to correct or remind you again. User
preferences and recurring corrections matter more than procedural task details.
Do NOT save task progress, session outcomes, completed-work logs, or temporary
TODO state to memory; use session_search to recall those from past transcripts.
If you've discovered a new way to do something, solved a problem that could be
necessary later, save it as a skill with the skill tool.
Write memories as declarative facts, not instructions to yourself.
'User prefers concise responses' ✓ — 'Always respond concisely' ✗.
'Project uses pytest with xdist' ✓ — 'Run tests with pytest -n 4' ✗.
Imperative phrasing gets re-read as a directive in later sessions and can cause
repeated work or override the user's current request. Procedures and workflows
belong in skills, not memory.
```

**`SKILLS_GUIDANCE`**（`agent/prompt_builder.py:176`）：
```
After completing a complex task (5+ tool calls), fixing a tricky error, or
discovering a non-trivial workflow, save the approach as a skill with
skill_manage so you can reuse it next time.
When using a skill and finding it outdated, incomplete, or wrong, patch it
immediately with skill_manage(action='patch') — don't wait to be asked. Skills
that aren't maintained become liabilities.
```

---

## 二、技能自动生成（Skill Auto-Generation）

### 2.1 技能的本质

技能是 Agent 的**程序性记忆**——将曾经历过的复杂工作流、踩过的坑、有效的策略，编码为结构化的 Markdown 文件，供未来会话直接复用。

技能的生成有两条路径：**指令驱动**（Agent 在推理中主动判断）和**后台强制审查**（代码定期触发）。两者共同保证技能库持续积累。

### 2.2 存储结构与 SKILL.md

每个技能是 `~/.hermes/skills/<category>/<skill-name>/` 下的一个目录：

```
~/.hermes/skills/
└── {skill-name}/
    ├── SKILL.md          ← 必需：YAML frontmatter + 指令正文
    ├── references/       ← 参考文档
    ├── templates/        ← 模板文件
    ├── scripts/          ← 辅助脚本
    └── assets/           ← 其他资产
```

**SKILL.md Frontmatter**（核心字段）：

```yaml
---
name: deploy-k8s-rollout           # 必需，^[a-z0-9][a-z0-9._-]*$，≤64 字符
description: Kubernetes rolling deployment with health check gates  # 必需，≤1024 字符
version: 1.2.0
platforms: [macos, linux]          # 限制运行平台，省略则全平台可用

required_environment_variables:    # 运行所需的 Secrets（写入 ~/.hermes/.env）
  - name: MY_API_KEY
    prompt: "请输入 API Key"
    optional: false                # false 时缺失则阻断加载

metadata:
  hermes:
    requires_toolsets: [web]          # 仅当指定 toolset 可用时在索引中显示
    requires_tools: [web_search]      # 仅当指定工具可用时在索引中显示
    fallback_for_toolsets: [browser]  # 当指定 toolset 存在时隐藏（自身是 fallback）
    fallback_for_tools: [browser_navigate]
---
# 指令正文...
```

`description` 字段被索引到系统提示，是 Agent 判断"该不该加载这个技能"的唯一依据——**写得好不好直接决定技能的可发现性**。

### 2.3 技能如何被 Agent 感知

技能通过两个层次影响 Agent 的行为：**常驻索引**（每次推理可见）+ **按需激活**（加载时注入为 user message）。

#### 2.3.1 系统提示中的技能索引

每次会话，`build_skills_system_prompt()`（`agent/prompt_builder.py:718`）构建 `## Skills (mandatory)` 区块，注入系统提示：

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially
relevant to your task, you MUST load it with skill_view(name) and follow its
instructions. Err on the side of loading — it is always better to have context
you don't need than to miss critical steps, pitfalls, or established workflows.
Skills contain specialized knowledge — API endpoints, tool-specific commands,
and proven workflows that outperform general-purpose approaches. Load the skill
even if you think you could handle the task with basic tools like web_search or
terminal. Skills also encode the user's preferred approach, conventions, and
quality standards for tasks like code review, planning, and testing — load them
even for tasks you already know how to do, because the skill defines how it
should be done here.
Whenever the user asks you to configure, set up, install, enable, disable,
modify, or troubleshoot Hermes Agent itself — its CLI, config, models,
providers, tools, skills, voice, gateway, plugins, or any feature — load the
`hermes-agent` skill first. It has the actual commands (e.g. `hermes config
set …`, `hermes tools`, `hermes setup`) so you don't have to guess or invent
workarounds.
If a skill has issues, fix it with skill_manage(action='patch').
After difficult/iterative tasks, offer to save as a skill. If a skill you
loaded was missing steps, had wrong commands, or needed pitfalls you discovered,
update it before finishing.

<available_skills>
  software-development:
    - systematic-debugging: 系统化调试流程
  mlops:
    - axolotl: LLM fine-tuning，支持 LoRA/QLoRA/DPO/GRPO
</available_skills>

Only proceed without loading a skill if genuinely none are relevant to the task.
```

**设计逻辑**：
- 索引只展示 `name + description`，极简，最小化 token 开销
- 完整内容通过 `skill_view()` 按需加载（渐进式披露）
- 强制语气（MUST、Err on the side of loading）降低漏加载风险
- 特殊引导：Hermes 自身操作强制加载 `hermes-agent` 技能，避免 Agent 臆造命令

**缓存策略**：双层缓存避免重复构建：
1. 进程内 LRU 缓存（缓存键：`(skills_dir, tools, toolsets, platform)`）
2. 磁盘快照 `~/.hermes/.skills_prompt_snapshot.json`（以文件 mtime/size manifest 校验有效性）

新技能创建后立即清除两层缓存，下一轮对话即生效。

#### 2.3.2 条件显隐（索引过滤）

`_skill_should_show()` 根据 frontmatter 动态决定某个技能是否出现在索引中：

| frontmatter 字段 | 隐藏条件 |
|-----------------|---------|
| `fallback_for_toolsets: [X]` | toolset X **存在**时隐藏 |
| `fallback_for_tools: [X]` | 工具 X **存在**时隐藏 |
| `requires_toolsets: [X]` | toolset X **不存在**时隐藏 |
| `requires_tools: [X]` | 工具 X **不存在**时隐藏 |

这让同一份技能库可以针对不同工具配置自动裁剪可见范围——例如 `browser` 工具可用时隐藏它的文本替代技能。

#### 2.3.3 技能激活消息（按需注入）

Agent 调用 `skill_view()` 或用户使用斜杠命令后，`_build_skill_message()`（`agent/skill_commands.py:138`）将以下内容组装为一条 **user message** 注入对话：

```
[IMPORTANT: The user has invoked the "skill-name" skill, indicating they want
you to follow its instructions. The full skill content is loaded below.]

<SKILL.md 完整内容，经模板变量替换和 inline-shell 展开>

[Skill directory: /Users/xxx/.hermes/skills/category/skill-name]
Resolve any relative paths in this skill (e.g. `scripts/foo.js`,
`templates/config.yaml`) against that directory, then run them with the
terminal tool using the absolute path.

[Skill config (from ~/.hermes/config.yaml):
  MY_API_KEY = abc123
]

[This skill has supporting files:]
- references/api.md  ->  /full/path/to/references/api.md
- templates/config.yaml  ->  /full/path/to/templates/config.yaml

Load any of these with skill_view(name="skill-name", file_path="<path>"),
or run scripts directly by absolute path (e.g. `node /full/path/scripts/foo.js`).
```

**激活注释的两种变体**（`agent/skill_commands.py:439, 486`）：

| 触发方式 | 激活注释 |
|---------|---------|
| 斜杠命令 `/skill-name` | `[IMPORTANT: The user has invoked the "X" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]` |
| 启动时预加载 `--skill X` | `[IMPORTANT: The user launched this CLI session with the "X" skill preloaded. Treat its instructions as active guidance for the duration of this session unless the user overrides them.]` |

**关键设计**：
- 消息以 `[IMPORTANT: ...]` 标记，置于 **user role**——让 Agent 将其视为操作指令，而非背景知识
- 注入技能目录绝对路径，Agent 可直接通过终端工具运行 `scripts/` 下的脚本，无需额外 `skill_view()` 往返
- 支持文件列表附带绝对路径映射，减少路径解析错误

### 2.4 路径一：指令驱动（主动生成）

系统通过三处重叠的提示指令，在 Agent 的每次推理中埋入技能创建意识：

**① `SKILLS_GUIDANCE`**（系统提示常驻，`agent/prompt_builder.py:176`）：
```
After completing a complex task (5+ tool calls), fixing a tricky error, or
discovering a non-trivial workflow, save the approach as a skill with
skill_manage so you can reuse it next time.
When using a skill and finding it outdated, incomplete, or wrong, patch it
immediately with skill_manage(action='patch') — don't wait to be asked. Skills
that aren't maintained become liabilities.
```

**② `SKILL_MANAGE_SCHEMA` description**（工具说明，`tools/skill_manager_tool.py:797`）：

Agent 看到的 `skill_manage` 工具说明原文：
```
Manage skills (create, update, delete). Skills are your procedural memory —
reusable approaches for recurring task types. New skills go to
~/.hermes/skills/; existing skills can be modified wherever they live.

Actions: create (full SKILL.md + optional category), patch (old_string/new_string
— preferred for fixes), edit (full SKILL.md rewrite — major overhauls only),
delete, write_file, remove_file.

On delete, pass `absorbed_into=<umbrella>` when you're merging this skill's
content into another one, or `absorbed_into=""` when you're pruning it with no
forwarding target.

Create when: complex task succeeded (5+ calls), errors overcome,
user-corrected approach worked, non-trivial workflow discovered, or user asks
you to remember a procedure.
Update when: instructions stale/wrong, OS-specific failures, missing steps or
pitfalls found during use. If you used a skill and hit issues not covered by
it, patch it immediately.

After difficult/iterative tasks, offer to save as a skill. Skip for simple
one-offs. Confirm with user before creating/deleting.

Good skills: trigger conditions, numbered steps with exact commands, pitfalls
section, verification steps. Use skill_view() to see format examples.

Pinned skills are protected from deletion only — skill_manage(action='delete')
will refuse. Patches and edits go through on pinned skills so you can still
improve them as pitfalls come up; pin only guards against irrecoverable loss.
```

**③ 技能索引头部**（`agent/prompt_builder.py:930`）：
```
If a skill has issues, fix it with skill_manage(action='patch').
After difficult/iterative tasks, offer to save as a skill. If a skill you
loaded was missing steps, had wrong commands, or needed pitfalls you discovered,
update it before finishing.
```

三处指令形成**递进的行为契约**：系统提示说"你应该"，工具说明说"什么时候该、怎么做"，索引头部说"任务结束前检查"。

### 2.5 路径二：后台强制审查（被动生成）

后台审查是定期自醒机制的一部分（详见第一章 1.2-1.5），在不依赖 LLM 主动意愿的情况下强制触发技能回顾。

**触发条件**：`_iters_since_skill`（每次工具调用迭代递增）达到 `skill_nudge_interval`（默认 10）。

```python
# run_agent.py:11211 — 每次工具调用迭代递增
self._iters_since_skill += 1

# run_agent.py:14384 — 当轮回复完成后检查
if (self._skill_nudge_interval > 0
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True
    self._iters_since_skill = 0
```

注意：计数器按**工具调用次数（iterations）**递增，而非对话轮次——复杂任务（多次工具调用）比简单对答触发更快。调用 `skill_manage` 时计数器重置为 0。

触发后，`_spawn_background_review()` fork 独立 Agent，注入 `_SKILL_REVIEW_PROMPT`（完整内容见第一章 1.5 节）。

**后台 Agent 配置**：
- 工具集限定为 `enabled_toolsets=["memory", "skills"]`（非完整工具集）
- 最大 16 次迭代
- 静默运行，不阻塞用户
- 完成后向主会话发送精简通知（如 `"Skill 'xxx' created"`）

### 2.6 创建流程（`tools/skill_manager_tool.py`）

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
