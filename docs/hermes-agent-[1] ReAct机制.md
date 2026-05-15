# Hermes Agent ReAct 机制

## 一、概述

**Hermes Agent** 是由 Nous Research 开发的开源 AI Agent 框架，其核心实现了 **ReAct（Reasoning + Acting）** 范式——即让 LLM 在"推理"与"行动"之间交替循环，直到任务完成。整个系统以 `run_agent.py` 中的 `AIAgent` 类为核心，通过 `run_conversation()` 方法驱动完整的 ReAct 循环。

**ReAct 范式的核心思想**：

```
用户输入 → [Reason] LLM 推理 → 有工具调用？
                                    ├── Yes → [Act] 执行工具 → 结果追加到 messages → 回到 Reason
                                    └── No  → 输出最终响应，结束
```

Hermes 的 ReAct 实现相比朴素循环增加了**四个关键机制**：

| 机制 | 解决的问题 | 位置 |
|------|-----------|------|
| **迭代预算** | 防止无限循环 | `IterationBudget` 线程安全计数器 + Grace Call |
| **上下文压缩** | 防止上下文溢出 | `ContextCompressor` 自动摘要中间轮次 |
| **工具护栏** | 防止重复失败/无进展循环 | `ToolCallGuardrailController` 检测并阻断 |
| **Steer** | 中途引导而不中断 | `/steer` 注入到工具结果，不破坏消息序列 |

---

## 二、ReAct 主循环实现

### 2.1 `run_conversation()` 整体流程

```
用户输入
    │
    ▼
[入循环前 — 每 turn 执行一次]
  1. 恢复主 Provider（fallback 回滚）+ 输入清洗（Surrogate / 注入攻击）
  2. System Prompt 构建/复用
     └── 首 turn 构建并写入 SQLite；续 session 从 DB 读取（保护 prefix cache）
  3. 预检上下文压缩（preflight compression）
     └── 若历史 token 已超阈值，先压缩再进循环，防止首轮 API 报错
  4. Plugin pre_llm_call hook —— 生成 ephemeral 插件上下文（不持久化）
  5. 外部记忆预取（memory_manager.prefetch_all）
     └── 只取一次，循环内复用，避免每轮重复查询
    │
    ▼
[ReAct 主循环]  ←─────────────────────────────────────────────┐
  while iteration_budget.remaining > 0:                        │
                                                               │
    ① 中断检查（跨线程信号，支持用户随时打断）                  │
                                                               │
    ② 构建 api_messages（每轮重新构建，不修改 messages）        │
       ├── 将记忆预取结果 + Plugin 上下文注入当轮 user message  │
       │   ★ 仅注入 api_messages，messages 保持原样            │
       │     → 上下文不污染持久化，重跑时可复现                 │
       └── 拼接 System Prompt（stable prefix → 命中 KV cache）  │
                                                               │
    ③ [Reason] 调用 LLM → 获取 response                       │
                                                               │
    ④ 解析 response                                            │
       ├── 有 tool_calls                                       │
       │     → [Act] 执行工具（顺序 or 并发）                  │
       │     → 工具结果以 role=tool 追加到 messages            │
       │     → post-response 压缩（服务端真实 token 计数）     │
       │         ├─ < 阈值(50%) → 回到循环顶部 ────────────────┘
       │         └─ ≥ 阈值     → 压缩后回到循环顶部 ──────────┘
       │
       └── 无 tool_calls（finish_reason=stop）
             → 最终响应，退出循环
    │
    ▼
[后处理]
  - 持久化 Session（SQLite）+ 保存 Trajectory（JSONL，用于 RL 训练）
  - 后台触发记忆 / 技能审查（background review）
  - 资源清理（子 Agent、terminal、browser）
```

**关键设计：ephemeral 注入**

记忆和插件上下文不写入 `messages`，只在每轮 `api_messages` 里临时拼接。这样：
- `messages` 始终是"干净"的对话历史，可持久化、可复现
- System Prompt 保持稳定，Anthropic prefix cache 跨轮命中

### 2.2 核心循环代码结构

> 源码：`run_agent.py`，行号标注对应实际位置，便于源码定位。

```python
# run_agent.py:8340 — 迭代预算初始化（线程安全计数器）
self.iteration_budget = IterationBudget(self.max_iterations)

# run_agent.py:8599 — ReAct 主循环
while (api_call_count < self.max_iterations
       and self.iteration_budget.remaining > 0) or self._budget_grace_call:

    # run_agent.py:8604 — ① 中断检查（跨线程信号）
    if self._interrupt_requested:
        interrupted = True
        break

    # run_agent.py:8618 — ② 预算消耗（budget_grace_call 为超限后的最后一次机会）
    if self._budget_grace_call:
        self._budget_grace_call = False
    elif not self.iteration_budget.consume():
        break

    # run_agent.py:8664 — ③ 构建 api_messages（ephemeral 注入，不修改 messages）
    api_messages = []
    for idx, msg in enumerate(messages):
        api_msg = msg.copy()
        if idx == current_turn_user_idx:          # 当轮 user message
            # 注入记忆预取结果 + Plugin 上下文（仅在 api_msg，不写回 messages）
            api_msg["content"] += "\n\n" + _ext_prefetch_cache + _plugin_user_context
        api_messages.append(api_msg)
    # 拼接 System Prompt（stable prefix → KV cache 命中）
    api_messages = [{"role": "system", "content": effective_system}] + api_messages

    # run_agent.py:8950 — ④ [Reason] 调用 LLM
    response = self._interruptible_api_call(api_kwargs)

    # run_agent.py:10532 — ⑤ 解析响应
    if assistant_message.tool_calls:
        # run_agent.py:10684-10761 — 去重、限流后执行工具（顺序 or 并发）
        self._execute_tool_calls(assistant_message, messages, effective_task_id, api_call_count)
        # 工具结果以 role=tool 追加到 messages → 回到循环顶部

    else:
        # run_agent.py:10827 — 无工具调用 → 最终响应，退出循环
        final_response = assistant_message.content or ""
        break
```

### 2.3 迭代预算（IterationBudget）

线程安全的迭代计数器，防止无限循环：

```python
class IterationBudget:
    def consume() -> bool    # 消耗一次迭代，返回是否允许
    def refund()             # 退还一次（execute_code 等编程式调用）
    @property remaining      # 剩余次数（线程安全）
```

- 默认 `max_iterations=90`（父 Agent）
- 子 Agent 独立预算（默认 50，通过 `delegation.max_iterations` 配置）
- 预算耗尽时注入一条提示消息，给模型最后一次机会输出文本响应（**Grace Call** 机制）

### 2.4 上下文压缩（Context Compression）

当对话历史接近模型上下文窗口阈值（默认 50%）时，`ContextCompressor` 自动压缩：

```
压缩算法（ContextCompressor）：
1. 预剪枝：将旧工具输出替换为单行摘要（无 LLM 调用，纯规则）
   - [terminal] ran `npm test` -> exit 0, 47 lines output
   - [read_file] read config.py from line 1 (1,200 chars)
   - ...
2. 保护头部：系统提示 + 前 N 条消息（protect_first_n=3）
3. 保护尾部：最近 ~20K tokens 的消息（protect_last_n=20）
4. LLM 摘要：用辅助模型（cheap/fast，如 Gemini Flash）摘要中间轮次
5. 迭代更新：多次压缩时，在前次摘要基础上增量更新
```

**摘要模板**包含结构化字段：
- `## Resolved Questions`：已解决的问题
- `## Active Task`：当前活跃任务（Agent 从此处恢复）
- `## Key Findings`：关键发现
- `## Remaining Work`：剩余工作

**关键约束**：压缩是**唯一允许修改历史上下文**的时机，其他任何操作均不得改变已有消息（保护 Prompt Caching 有效性）。

---

## 三、Reason 阶段：LLM 推理

### 3.1 多 API 模式支持

Hermes Agent 支持四种 LLM API 模式，通过 `api_mode` 字段统一路由：

| API 模式 | 适用场景 | 特点 |
|----------|----------|------|
| `chat_completions` | OpenAI 兼容接口（默认） | 标准 OpenAI 格式，最广泛兼容 |
| `anthropic_messages` | 原生 Anthropic API | 支持 Prompt Caching、Extended Thinking |
| `codex_responses` | OpenAI Responses API | GPT-5.x 系列，支持加密推理链（encrypted reasoning） |
| `bedrock_converse` | AWS Bedrock | boto3 直接调用，支持 Guardrail |

### 3.2 流式推理（Streaming）

所有 API 调用均采用**流式输出**，通过独立线程执行，主线程轮询结果：

```python
# 流式调用在独立线程中执行，主线程轮询
t = threading.Thread(target=_call, daemon=True)
t.start()
while t.is_alive():
    t.join(timeout=0.3)
    # 检测 stale stream（超过阈值无 chunk → 重建连接）
    if time.time() - last_chunk_time["t"] > _stream_stale_timeout:
        # 重建连接，重试
```

**流式回调链**：

| 回调 | 触发时机 | 用途 |
|------|----------|------|
| `stream_delta_callback` | 每个文本 token | CLI 实时渲染、TTS 管道 |
| `reasoning_callback` | 推理 token（Extended Thinking） | 推理过程展示 |
| `tool_gen_callback` | 工具调用名称生成时 | 工具调用预告 |
| `tool_start_callback` | 工具开始执行 | 进度通知 |
| `tool_complete_callback` | 工具执行完成 | 结果通知 |

### 3.3 System Prompt 构建

System Prompt 由 `_build_system_prompt()` 组装，**每个 Session 只构建一次**（缓存在 `_cached_system_prompt`），以保证 Anthropic Prompt Caching 的前缀一致性：

```
System Prompt 组成（按顺序拼接）：
1. DEFAULT_AGENT_IDENTITY    ← 身份定义（Hermes Agent by Nous Research）
2. SOUL.md                   ← 用户自定义人格文件（~/.hermes/SOUL.md）
3. TOOL_USE_ENFORCEMENT      ← 工具使用强制指令（按模型类型动态注入）
4. MEMORY_GUIDANCE           ← 记忆使用指导
5. SESSION_SEARCH_GUIDANCE   ← 跨会话搜索指导
6. SKILLS_GUIDANCE           ← 技能使用指导
7. Memory Block              ← MEMORY.md + USER.md 内容快照
8. External Memory Block     ← 外部记忆插件（Honcho/Mem0 等）
9. Skills Index              ← 可用技能列表（frontmatter 摘要）
10. Context Files            ← AGENTS.md / .hermes.md / HERMES.md
11. 时间戳 + Session ID + 模型信息
12. 平台 Hints               ← WhatsApp/Telegram/Slack 等平台特定指令
```

### 3.4 工具使用强制指令（Tool-Use Enforcement）

针对不同模型家族，注入专项执行纪律提示，防止模型"只说不做"：

| 模型家族 | 注入内容 |
|----------|----------|
| **通用**（GPT/Gemini/Grok 等） | `TOOL_USE_ENFORCEMENT_GUIDANCE`：禁止只描述不行动，必须立即执行 |
| **GPT/Codex 系列** | `OPENAI_MODEL_EXECUTION_GUIDANCE`：工具持久性、前提检查、验证、反幻觉 |
| **Gemini/Gemma 系列** | `GOOGLE_MODEL_OPERATIONAL_GUIDANCE`：绝对路径、并行工具调用、简洁性 |

核心约束原文：
> "当你说你将执行某个动作时，你必须立即在同一响应中进行对应的工具调用。永远不要以未来行动的承诺结束你的轮次——现在就执行它。"

---

## 四、Act 阶段：工具执行

### 4.1 工具注册体系

工具通过 `ToolRegistry` 单例管理，每个工具文件在 import 时自动注册：

```python
# tools/your_tool.py
registry.register(
    name="example_tool",
    toolset="example",
    schema={...},           # OpenAI Function Calling Schema
    handler=lambda args, **kw: ...,
    check_fn=check_requirements,  # 可用性检查（API Key 是否配置等）
    requires_env=["EXAMPLE_API_KEY"],
)
```

**自动发现机制**：`discover_builtin_tools()` 通过 **AST 静态分析**扫描 `tools/*.py`，找到包含顶层 `registry.register()` 调用的文件并动态 import，无需手动维护工具列表。

### 4.2 工具分发链

```
run_agent._execute_tool_calls()
    │
    ├── 是否可并发？_should_parallelize_tool_batch()
    │     ├── 包含交互工具（clarify）→ 顺序执行
    │     ├── 全部只读工具 → 并发执行
    │     └── 文件工具路径无冲突 → 并发执行
    │
    ├── [顺序] _execute_tool_calls_sequential()
    │
    └── [并发] _execute_tool_calls_concurrent()
          └── ThreadPoolExecutor(max_workers=8)
```

**Agent 级工具**（在 `run_agent.py` 内直接处理，不经过 registry dispatch）：

| 工具 | 说明 |
|------|------|
| `todo` | 任务列表管理（`TodoStore`，in-memory） |
| `memory` | 持久记忆读写（`MEMORY.md` / `USER.md`） |
| `session_search` | 历史会话 SQLite FTS5 全文搜索 |
| `delegate_task` | 子 Agent 委派（支持并行批量） |
| `clarify` | 向用户提问（通过 `clarify_callback`） |

**Registry 分发工具**（通过 `handle_function_call()` → `registry.dispatch()`）：

- `terminal`、`read_file`、`write_file`、`patch`、`search_files`
- `web_search`、`web_extract`
- `browser_navigate`、`browser_click`、`browser_snapshot` 等
- `execute_code`（沙箱代码执行）
- MCP 工具（外部 MCP Server 动态注册）

### 4.3 并发工具执行

```python
# 并发安全分类
_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})        # 永不并发（交互工具）
_PARALLEL_SAFE_TOOLS = frozenset({                     # 始终可并发（只读）
    "read_file", "web_search", "web_extract",
    "session_search", "vision_analyze", ...
})
_PATH_SCOPED_TOOLS = frozenset({                       # 路径无冲突时并发
    "read_file", "write_file", "patch"
})
```

并发结果按**原始工具调用顺序**收集后追加到 messages，保证 API 消息序列的正确性。

### 4.4 工具结果大小控制

工具结果过大时，`maybe_persist_tool_result()` 自动将结果写入临时文件，返回文件路径引用，防止上下文窗口被单个工具结果撑爆。`enforce_turn_budget()` 在每轮工具执行后检查总结果大小。

---

## 附录：关键文件索引

| 组件 | 文件路径 | 关键位置 |
|------|---------|---------|
| ReAct 主循环 | `run_agent.py` | L8237 `run_conversation()`, L8599 主循环 |
| 迭代预算 | `run_agent.py` | L175 `IterationBudget` 类 |
| 工具执行（顺序） | `run_agent.py` | `_execute_tool_calls_sequential()` |
| 工具执行（并发） | `run_agent.py` | `_execute_tool_calls_concurrent()` |
| 工具分发 | `model_tools.py` | `handle_function_call()` |
| 工具注册中心 | `tools/registry.py` | `ToolRegistry` 类 |
| 工具自动发现 | `tools/registry.py` | `discover_builtin_tools()` |
| System Prompt 构建 | `agent/prompt_builder.py` | `build_skills_system_prompt()` 等 |
| 上下文压缩 | `agent/context_compressor.py` | `ContextCompressor` 类 |
| 流式 API 调用 | `run_agent.py` | `_interruptible_api_call()` |
