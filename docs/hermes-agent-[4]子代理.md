# Hermes Agent Subagent 实现技术白皮书

Subagent（子代理）通过 `delegate_task` 工具实现：父 Agent 在自己的工具循环中调用 `delegate_task`，运行时在**主线程构建子 `AIAgent` 实例**，再用 `ThreadPoolExecutor` 并行执行（默认上限 3 个），每个子代理拥有**独立的对话历史、独立的终端会话、隔离且受限的工具集**。父代理只看到一次工具调用 + 最终摘要，**子代理的中间工具结果与推理永远不进入父上下文**——这是 subagent 机制对上下文的核心杠杆作用。配合**深度限制（最多 1 层）、阻塞工具集（`delegate_task / clarify / memory / send_message / execute_code`）、心跳活性传播、中断级联、凭证池租约**等机制，保证了多代理协作的安全性、可观测性与成本可控。

---

## 一、架构总览

```
父 AIAgent (depth=0)
   │
   │  调用 delegate_task(goal=..., tasks=[...])
   ▼
delegate_task()                        # tools/delegate_tool.py
   ├─ 深度检查：depth < MAX_DEPTH=2
   ├─ 加载 delegation 配置（model/provider/base_url/...）
   ├─ 解析子代理凭证（继承父 / 配置覆盖）
   ├─ 校验 tasks 数量 ≤ max_concurrent_children (默认 3)
   │
   ├─ 主线程构建所有 child AIAgent           ← 线程安全屏障
   │     for t in tasks: _build_child_agent(t)
   │
   ├─ 单任务  → _run_single_child() 直接执行
   │  多任务  → ThreadPoolExecutor 并行 + 进度刷新
   │
   └─ 收集 results[]，回写 JSON 给父代理
```

**关键文件**：`tools/delegate_tool.py`（单文件 ~1140 行，承载全部 subagent 逻辑）

---

## 二、子代理生命周期

### 2.1 构建阶段（`_build_child_agent`）

| 步骤 | 关键动作 |
|---|---|
| **工具集解析** | ① 若 `toolsets` 显式传入，与父工具集**取交集**（子代理永不超权父）<br>② 否则继承父 `enabled_toolsets`<br>③ 调用 `_strip_blocked_tools()` 移除 `delegation / clarify / memory / code_execution` |
| **System Prompt 构建** | `_build_child_system_prompt(goal, context, workspace_path)` —— 焦点式提示，强调"无对话历史、需返回简洁摘要" |
| **工作区注入** | `_resolve_workspace_hint()` 从 `TERMINAL_CWD` / 父 `terminal_cwd` 等候选中**只取已存在的绝对目录**，避免幻觉性的 `/workspace/...` 路径 |
| **凭证解析** | 优先级：`delegation.base_url` > `delegation.provider` > 继承父；支持把子代理路由到完全不同的 provider:model（如父 Nous Portal、子 OpenRouter） |
| **进度回调** | `_build_child_progress_callback()` —— CLI 走 spinner 树形输出，Gateway 走批量 flush（每 5 条 tool 名一次），`_thinking` 事件仅 CLI 显示 |
| **关键参数** | `quiet_mode=True`、`skip_context_files=True`、`skip_memory=True`、`clarify_callback=None`、`iteration_budget=None`（独立预算） |
| **递归保护** | `child._delegate_depth = parent._delegate_depth + 1` |
| **中断挂载** | 把 `child` 加入 `parent._active_children` 列表（持锁），用于父中断时级联传递 |

### 2.2 执行阶段（`_run_single_child`）

```
_run_single_child(task_index, goal, child, parent_agent)
   │
   ├─ 启动心跳线程 (_HEARTBEAT_INTERVAL=30s)
   │     每 30s 调 parent._touch_activity() 描述子代理当前 tool/迭代
   │     ↑ 防止 Gateway 因父侧 _last_activity_ts 不动而误杀会话
   │
   ├─ （可选）credential_pool.acquire_lease()
   │     租用一份凭证 → child._swap_credential(entry) 热替换
   │
   ├─ child.run_conversation(user_message=goal)
   │     ↑ 完整复用父类的 ReAct 主循环
   │
   ├─ 解析结果：interrupted / completed / failed
   │
   ├─ 重建 tool_trace（按 tool_call_id 配对，处理并行调用）
   │     遍历 messages → 对 assistant.tool_calls 与 tool 消息匹配
   │
   └─ finally:
        ├─ 停止心跳线程
        ├─ 释放凭证租约
        ├─ 还原 model_tools._last_resolved_tool_names
        ├─ 从 parent._active_children 摘除
        └─ child.close() 释放 terminal/browser/process 资源
```

### 2.3 单任务 vs 批量

| 模式 | 触发 | 执行路径 |
|---|---|---|
| **单任务** | 提供 `goal` | `_run_single_child()` 直接调，**无线程池开销** |
| **批量** | 提供 `tasks=[{...}, ...]` | `ThreadPoolExecutor(max_workers=max_concurrent_children)` 并发；`as_completed` 改写为 `wait(timeout=0.5, FIRST_COMPLETED)` 轮询，以便能响应父中断 |

---

## 三、上下文隔离机制

这是 subagent 对父代理上下文的核心价值。

| 隔离维度 | 实现 |
|---|---|
| **对话历史** | 子代理是全新 `AIAgent` 实例，`messages=[]`，**完全看不到父对话** |
| **System Prompt** | 通过 `ephemeral_system_prompt=child_prompt` 注入焦点提示，绕过父的 8 层装配 |
| **记忆 / 项目上下文** | `skip_memory=True`、`skip_context_files=True` —— 不加载 MEMORY.md / AGENTS.md 等 |
| **工具结果** | 子代理的 tool_call/tool_result 只存在于其自身 `messages`，父只看到 `delegate_task` 一次返回 |
| **终端会话** | 每个子代理有独立 `task_id` → 独立的 terminal sandbox、独立 cwd、独立 file ops 缓存 |
| **Token 计费** | 子代理拥有独立的 `session_prompt_tokens / session_completion_tokens`，结算后回写给父结果中的 `tokens.input/output` |
| **迭代预算** | `iteration_budget=None` 强制新建独立预算，子代理消耗不计入父配额 |

**返回到父上下文的内容**仅有结构化 JSON：

```json
{
  "results": [{
    "task_index": 0,
    "status": "completed",
    "summary": "<子代理最终自然语言回复>",
    "api_calls": 7,
    "duration_seconds": 23.4,
    "model": "claude-opus-4.6",
    "exit_reason": "completed",
    "tokens": {"input": 12345, "output": 678},
    "tool_trace": [{"tool": "read_file", "args_bytes": 87, "result_bytes": 4096, "status": "ok"}]
  }],
  "total_duration_seconds": 23.4
}
```

---

## 四、安全与约束

### 4.1 阻塞工具集

`DELEGATE_BLOCKED_TOOLS = frozenset(["delegate_task", "clarify", "memory", "send_message", "execute_code"])`

| 工具 | 禁用原因 |
|---|---|
| `delegate_task` | 防止递归喷射子代理 |
| `clarify` | 子代理不与用户交互（无终端连接） |
| `memory` | 防止并发写入污染共享 `MEMORY.md` |
| `send_message` | 阻断跨平台副作用（Telegram/Discord 等） |
| `execute_code` | 推理型子代理应分步思考，不应自写脚本黑盒处理 |

### 4.2 深度限制

- `MAX_DEPTH = 2` —— 父(0) → 子(1) → 拒绝(2)
- 实际效果：**只允许一层委托**，杜绝指数级展开

### 4.3 工具集权限收敛

```python
# 父没有的工具，子也不能拥有
child_toolsets = _strip_blocked_tools([t for t in toolsets if t in parent_toolsets])
```

子代理的工具集是 `(用户传入 toolsets ∩ 父工具集) − 阻塞集`，不存在权限提升路径。

### 4.4 并发上限

- `delegation.max_concurrent_children`（默认 **3**），可被 `DELEGATION_MAX_CONCURRENT_CHILDREN` 环境变量覆盖
- 超过上限直接返回 `tool_error`，**不做截断、不做排队**——交还给模型决策

### 4.5 工作区路径防御

`_resolve_workspace_hint()` 仅当候选目录**实际存在且为绝对路径**时注入；并在 prompt 中明确警告："Never assume a repository lives at /workspace/... unless explicitly given"，防止子代理猜路径。

---

## 五、可观测性与活性

### 5.1 心跳机制

子代理执行期间，独立线程每 30s 执行：

```python
parent._touch_activity(f"delegate_task: subagent running {tool} (iter X/Y)")
```

**问题**：父代理的工具调用循环在 `delegate_task` 处一直阻塞，`_last_activity_ts` 不更新 → Gateway 的不活跃超时会错杀整个会话。
**解决**：心跳线程从 `child.get_activity_summary()` 拉取实时进度，反向"心跳"父代理。

### 5.2 进度回调

| 显示通道 | 行为 |
|---|---|
| **CLI** (`_delegate_spinner`) | 通过 `spinner.print_above()` 输出树形进度：`[1] ├─ 🔧 read_file "src/foo.py"`、`[1] ├─ 💭 "thinking..."` |
| **Gateway** (`tool_progress_callback`) | 批量收集 5 条 tool 名 → flush 一次 `🔀 [1] read_file, file_grep, terminal, ...`；过滤 `_thinking` 事件以减少噪声 |

### 5.3 任务完成通告

每个子任务完成时在 spinner 上方打印 `✓ [1/3] 任务标签 (12.3s)`，并更新 spinner 文本 `🔀 2 tasks remaining`，给用户清晰的并发进度感。

### 5.4 中断级联

```python
# 父端：捕获用户 Ctrl+C → 设置 _interrupt_requested=True
#       并对 _active_children 中每个 child 也设置标志
# delegate_task 主循环：每轮 wait(0.5s) 检查 _interrupt_requested
if parent._interrupt_requested:
    # 收割已完成的 future，未完成的标记为 "interrupted"，不再无限等待
```

**关键设计**：原 `as_completed()` 会一直阻塞到全部完成，使中断无效；改用 `wait(timeout=0.5, FIRST_COMPLETED)` 轮询是为了**让父代理可以"放弃等待"**。

---

## 六、凭证与并发模型

### 6.1 凭证池租约（`_resolve_child_credential_pool`）

```
若子 provider == 父 provider:
    → 共享父的 credential_pool（cooldown / rotation 同步）
否则尝试为子 provider 加载独立 pool:
    → 加载成功 → 子代理在并发中独立轮换
    → 失败    → 子代理使用继承的固定凭证
```

每个子代理执行前调 `pool.acquire_lease()`，结束在 `finally` 中 `release_lease()`。这让 3 个并发子代理可以**各自占用一份凭证**，避免单 key 限速雪崩。

### 6.2 跨 Provider 路由

`_resolve_delegation_credentials()` 三档配置：

| 配置 | 行为 |
|---|---|
| `delegation.base_url` 直配 | 嗅探 URL 自动判断 provider/api_mode（Anthropic / Codex / 通用 OpenAI 兼容） |
| `delegation.provider` 命名 | 走 `resolve_runtime_provider()` —— 与 CLI/Gateway 启动相同的凭证解析路径 |
| 都未配置 | 完全继承父代理（model/provider/base_url/api_key/api_mode） |

典型场景：**父 Claude Opus（强推理）→ 子路由到 OpenRouter 上的廉价快模型**，平衡成本与质量。

### 6.3 ACP 子代理

`acp_command` / `acp_args` 支持把子代理切换为 **ACP 子进程**（如 `claude --acp --stdio`）。这意味着：

- **从任意父代理（CLI / Telegram / Discord）都可生成 Claude Code 子进程**
- 父进程可以是 Hermes 自身，子进程可以是异构 agent

支持**每任务独立 ACP 命令**：`tasks: [{goal, acp_command, acp_args}, ...]`。

---

## 七、关键陷阱与处理

### 7.1 `model_tools._last_resolved_tool_names` 进程级污染

**问题**：`_build_child_agent()` 调用 `AIAgent()` 会触发 `get_tool_definitions()`，后者覆盖 `_last_resolved_tool_names` 全局变量。子代理构建完成后，父代理继续运行 `execute_code` 等依赖该全局的工具就会读到错误值。

**解决**（双重保险）：

```python
# 构建前快照
_parent_tool_names = list(_model_tools._last_resolved_tool_names)
try:
    for t in task_list:
        child = _build_child_agent(...)
        child._delegate_saved_tool_names = _parent_tool_names  # 挂在 child 上
finally:
    _model_tools._last_resolved_tool_names = _parent_tool_names  # 权威还原 #1

# 子代理执行结束（finally）
saved_tool_names = getattr(child, "_delegate_saved_tool_names", None)
if isinstance(saved_tool_names, list):
    model_tools._last_resolved_tool_names = list(saved_tool_names)  # 权威还原 #2
```

### 7.2 资源不能跨越委托边界

`finally` 中显式调用 `child.close()` —— 关闭 terminal sandbox / browser daemon / 后台进程 / httpx client，避免 subprocess 泄漏。

### 7.3 并行 `tool_call_id` 配对

构建 `tool_trace` 时按 `tool_call_id` 而非顺序配对 assistant.tool_calls 与 tool messages —— 因为现代 API 支持单个 assistant 消息内的并行工具调用，按顺序匹配会错位。

### 7.4 ThreadPoolExecutor + Ctrl+C 死等

弃用 `as_completed()`，改用循环 `wait(timeout=0.5, FIRST_COMPLETED)` —— 否则一旦某 child 卡住，父代理无法响应中断信号。

---

## 八、与父代理的交互回路

子代理结束后，父端额外触发：

```python
parent._memory_manager.on_delegation(
    task=goal,
    result=summary,
    child_session_id=child.session_id,
)
```

让记忆系统能感知"本次委托做了什么"，可被外部记忆 Provider（Honcho / Mem0 等）持久化为代理协作的事件流。

---

## 九、关键工程权衡

| 设计决策 | 取舍考量 |
|---|---|
| **子代理只回传摘要** | 上下文隔离的核心收益，避免父侧爆炸性增长 |
| **深度限制为 1 层** | 杜绝递归扇出导致的成本失控；多层任务由父分阶段委托 |
| **批量上限默认 3** | 平衡吞吐与单 provider 限速，可配置上调 |
| **主线程构建 + 子线程执行** | `AIAgent` 构造涉及全局可变状态，主线程构建是线程安全屏障 |
| **独立 `iteration_budget`** | 子代理不消耗父预算，单一任务可跑满 50 轮 |
| **心跳反向 touch 父活性** | 必须有，否则 Gateway 会以"父无活动"为由 kill 会话 |
| **`as_completed` → `wait(timeout)` 轮询** | 唯一能让父在批量中断时优雅退出的方式 |
| **凭证池 lease/release** | 让 N 个并发子代理跨 N 个 key 真正并行，单 key 限速不阻塞集群 |
| **阻塞 `execute_code`** | 强制子代理"逐步推理 + 工具"，避免子代理把任务黑盒成一段脚本 |
| **阻塞 `memory`** | 共享 `MEMORY.md` 没有并发安全协议，禁写避免数据损坏 |
| **不缓存子代理对话** | 子代理多为一次性任务，缓存收益低于资源占用 |
| **跨 provider 路由能力** | 让"指挥型父 + 工兵型子"成为可能（强模型规划 + 廉价模型执行） |

---

## 十、典型调用流图

```
用户 → 父Agent: "审查 src/auth/ 与 src/api/ 两个模块的安全问题"
父Agent → delegate_task(tasks=[
   {goal:"审查 src/auth/", toolsets:["file","terminal"]},
   {goal:"审查 src/api/",  toolsets:["file","terminal"]},
])
                 │
                 ├─ 主线程：构建 child_0、child_1（各自独立 prompt + 终端）
                 ├─ 启动 2 个心跳线程
                 ├─ ThreadPoolExecutor(2):
                 │     child_0.run_conversation("审查 src/auth/")  ─┐
                 │     child_1.run_conversation("审查 src/api/")   ─┤  并行
                 │                                                ─┘
                 ├─ 用户 Ctrl+C? → 否，继续 wait(0.5s) 轮询
                 ├─ ✓ child_0 完成（12.3s, 7 tool calls）→ spinner 输出完成行
                 ├─ ✓ child_1 完成（18.5s, 9 tool calls）
                 ├─ memory_manager.on_delegation(...)（×2）
                 └─ 返回 JSON {results:[{summary:"..."},{summary:"..."}], total:18.5s}
父Agent: 收到 JSON → 综合两份摘要 → 用户最终答复
```

整个流程中，**父上下文增量仅为 1 次 tool_call + 1 次 tool_result（含 2 段子代理摘要）**，而子代理内部各自跑了 7~9 轮工具调用——这就是 subagent 在上下文层面的核心杠杆作用。
