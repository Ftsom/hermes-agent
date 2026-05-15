# Hermes Agent 消息网关技术白皮书

## 它解决什么问题？

想象一下：Alice 希望她的 Hermes Agent **同时**——

- 在 Telegram 私聊里回答日常问题
- 在 Slack `#devops` 值班、处理告警
- 在飞书群里参与产品讨论
- 凌晨跑一份 `weather_report` 定时投递到 Telegram
- 再顺带暴露 OpenAI 兼容 HTTP 端点给前端（Open WebUI / LobeChat）

把这些需求拉齐后，问题立刻冒出来：

1. **四个平台、四套 SDK**，难道业务逻辑写四遍？
2. **会话如何归属？** 同一个人在不同平台的对话算一段还是多段？重启后还认得吗？
3. **并发消息会串吗？** Alice 和 Bob 同时发消息，工具里的 `send_message` 怎么知道当前回给谁？
4. **Agent 跑 30 秒用户在看空白怎么办？** 多数平台没有 SSE 推流。
5. **陌生人私聊怎么授权？** 硬写 `.env` 白名单不够灵活。
6. **进程被 systemd 和 `--replace` 同时拉起**，两个实例抢长轮询 token 怎么办？

**Hermes 消息网关（Gateway）就是一次性解决这六类问题的基础设施。** 它是一个长驻的 asyncio 守护进程，向下统一 20+ 消息平台，向上给 AIAgent 提供"你只管写业务逻辑、平台差异我全抹平"的抽象。源码在 `gateway/`，入口是 `gateway/run.py::start_gateway()`。

---

## 一、整体定位：一台"平台路由交换机"

Gateway **不是** Agent 的一部分，而是 **Agent 与外部消息世界之间的一层薄翻译层**。

```
    外部世界                Gateway（平台无关层）              Hermes 内核
 ─────────────────      ──────────────────────────────     ──────────────────
 Telegram   ─┐                                              ┌─ AIAgent #1
 Discord    ─┤          ┌────────────────────────────┐     │
 Slack      ─┼──┐       │ BasePlatformAdapter        │     ├─ AIAgent #2
 飞书       ─┤  │       │  connect / send / edit ... │     │
 WhatsApp   ─┘  ├──────►│                            │────►├─ AIAgent #3
 Signal         │       │ GatewayRunner              │     │
 邮件/SMS       │事件    │  session_store / pairing / │     ├─ AIAgent #N
 OpenAI API ────┘(统一) │  agent_cache / hooks /     │     │
 Webhook                │  delivery_router           │     └─ 共享记忆/技能/工具
                        └────────────────────────────┘
```

一句话概括它的职责：

> **把 N 个平台的 M 个聊天收敛成 N×M 个 Hermes 会话，每个会话绑定一个 AIAgent 实例，消息进来不串、出去不丢、重启后还能续。**

落到代码里由四个抽象承担：

| 职责 | 核心抽象 | 解决的问题 |
|------|----------|-----------|
| 统一接入 | `BasePlatformAdapter` 协议 | 抹平 20+ 平台 SDK 差异 |
| 会话映射 | `SessionSource` → `session_key` | "这条消息属于谁的哪段对话" |
| 并发隔离 | `contextvars.ContextVar` | 并发消息互不串扰 |
| 可靠投递 | `StreamConsumer` + `DeliveryRouter` | 流式回显、多目标投递 |

---

## 二、一条消息的完整旅程

以 Alice 在 Telegram 群 `chat_id=-1001234` 的 `thread_id=42` 里 @bot 发 "帮我查下今天北京天气" 为例：

```
 ① Telegram 推送     ② adapter 翻译    ③ Runner 路由决策
     ↓                    ↓                   ↓
 MessageHandler       MessageEvent        斜杠命令？占用？
                      SessionSource       入队？普通消息？
                              ↓
 ④ 会话定位           ⑤ ContextVar 注入   ⑥ Agent 执行
     session_key       task-local 隔离    缓存/新建实例
                              ↓
 ⑦ 流式 edit 投递                        ⑧ 收尾
     首次 send → 后续 edit 节流 1s         清 ContextVar、持久化、触发 hook
```

### ① 接收：平台 SDK 把消息推进来

各平台接入方式不同：Telegram 长轮询、Discord gateway websocket、Slack Socket Mode、HTTP Webhook 平台用 aiohttp 监听。**每个 adapter 唯一的职责就是把平台原生事件翻译成 `MessageEvent`**，抽取出：谁发的、哪里来的（chat_id/chat_type/thread_id）、说了什么、附带了什么媒体。

### ② 翻译：生成 `SessionSource`

`SessionSource` 是**平台无关的身份描述**：

```python
SessionSource(
    platform=Platform.TELEGRAM,
    chat_id="-1001234", chat_type="group",
    thread_id="42", user_id="789", user_name="Alice",
    ...
)
```

从这步开始，**gateway 后续逻辑再也不需要关心具体是什么平台**——只跟 `SessionSource` 打交道。这就是为什么同一套会话管理、Agent 缓存、命令分发能无缝服务 20+ 平台。

### ③ 路由决策：`GatewayRunner.handle_message()`

进入 `GatewayRunner`（`gateway/run.py`）后要做几个判断：

- **授权检查**：陌生用户启动 DM Pairing 流程（第五章）；已知用户放行。
- **斜杠命令？** `/reset`、`/model`、`/queue` 走专用 handler。命令识别依赖 `hermes_cli/commands.py::COMMAND_REGISTRY`——**全系统唯一的命令真相源**，CLI、gateway、Telegram 菜单、Slack 子命令、autocomplete 全从这里派生。
- **会话正忙？** 若 `_running_agents[key]` 有值，根据 `busy_input_mode` 选择打断（默认）或入队。
- **普通消息**：进入会话定位。

### ④ 会话定位：`SessionSource` → `session_key`

要回答"Alice 这条消息属于哪段历史对话"，且答案必须在重启后依然成立。

`SessionStore._generate_session_key(source)` 生成纯函数式会话键：

```
agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}]
```

Alice 拿到 `agent:main:telegram:group:-1001234:42`。这个键是**确定性、正交（不同平台永不碰撞）、thread 感知**的。拿到 key 后 `get_or_create_session()` 从 `sessions.json + {session_id}.jsonl + SQLite FTS5` 加载或创建。

> **这里的 `thread_id` 是平台的业务概念，与操作系统线程 id 无关。** 各平台对应关系：
>
> | 平台 | `thread_id` 对应什么 |
> |------|---------------------|
> | Telegram | forum group 里的 **topic**（"公告话题"、"闲聊话题"） |
> | Discord | 频道里的 **thread**（讨论串） |
> | Slack | 某条消息下的 **thread**（回帖串） |
> | 私聊 / 普通群 | `None` |
>
> 同一群的不同 topic 视为不同会话，各自独立 transcript 与 Agent 实例。

**会话重置策略**也在这里生效。`SessionResetPolicy` 支持 `daily / idle / both / none` 四种，重置前让 Agent 有机会把重点写入长期记忆，且 `process_registry.has_active_for_session()` 会延后有活跃子进程的会话，避免把正在干活的 Agent 杀了。

### ⑤ ContextVar 注入：并发隔离的核心创新

**这是 gateway 最精彩也最容易被忽视的一层。**

Alice 和 Bob 在不同平台几乎同时发消息，Agent 工具（`send_message`、`sleep_now` 等）必须知道"我现在是在为谁服务"。

#### 为什么 `os.environ` 和 `threading.local()` 都不行

- **`os.environ` 不行**：它是**进程级全局**，Bob 的消息会覆盖 Alice 设置的 chat_id，Alice 的工具读到 Bob 的值——典型串台。这是把会话状态塞进环境变量时的经典陷阱。
- **`threading.local()` 也不行**：gateway 是 **asyncio 单进程**，Alice、Bob、Charlie 的 task 全部跑在**同一个事件循环线程**里。`await` 一让出，task 切换了但线程没变，ThreadLocal 依然共用一份，串台照旧。

#### ContextVar 为何成立

`contextvars.ContextVar` 把隔离单位从"线程"换成了 **Context（在 asyncio 下 ≈ Task）**：asyncio 创建 Task 时自动复制一份父 Context，Task 切换时事件循环自动切换 Context。**每个 task 拥有自己独立副本**，即使共用线程也互不影响。

更关键的是，`loop.run_in_executor()` 会**把当前 Task 的 Context 复制到工作线程**——所以 Agent 虽然在线程池里同步跑工具，读到的依然是"发起它的那条消息"的值。这一步 ThreadLocal 做不到（线程池线程是复用的）。

一句话：**ContextVar 是为 async/await 时代重新设计的 ThreadLocal**；在 asyncio 场景下，只有它能正确跨越 task 与线程池的边界。

#### 三态语义

实现在 `gateway/session_context.py`：

| 状态 | 含义 | `get_session_env` 行为 |
|------|------|---------------------|
| `_UNSET`（从未 set） | 当前在 CLI 或 cron | fallback 到 `os.environ`（兼容老路径） |
| `""`（显式清空） | gateway 正在清理 | 直接返回 `""`，**不**回退到环境变量 |
| 具体值 | 正在处理一条消息 | 返回该值 |

没有这三态，同一份工具代码就无法在 CLI / Gateway / Cron 三种语境下都正确工作。

### ⑥ Agent 执行：缓存 + 线程池

拿到 `session_key` 后查 `_agent_cache`（LRU `OrderedDict`，容量 128，空闲 3600s 驱逐）：

- **命中**：复用已有 `AIAgent`，**关键收益是保留 Prompt KV 缓存**（否则 system prompt 重新拼装，缓存全 miss，成本 × 10）。
- **未命中**：新建 `AIAgent(session_id=..., platform="telegram", callbacks={...})`，callbacks 包含 `stream_delta_callback`、`tool_call_started/completed`、`approval_callback`、`notification_callback`。

Agent 放线程池跑（`loop.run_in_executor(..., agent.run_conversation, user_msg)`），事件循环继续处理其他会话。

### ⑦ 流式 edit 投递：边跑边回显

一次 Agent run 常需 20~60 秒，用户看着一片空白体验很差。多数平台没有 SSE 推流，但普遍支持**"编辑已发送消息"**——gateway 的做法是**利用 edit 做假推流**。

`gateway/stream_consumer.py::GatewayStreamConsumer` 的模式：

1. Agent 产生增量 token 时同步调 `stream_delta_callback(text)`
2. Consumer 入队，异步消费
3. 首次：`adapter.send(累积文本)` 拿到 `message_id`
4. 后续每 `edit_interval=1.0s` 或 `buffer_threshold=40` 字符：`adapter.edit(message_id, 最新累积)`
5. **Fresh Final**：若预览已挂超 60 秒，最终消息**改发新消息而非 edit**，让时间戳贴合完成时刻
6. **Flood control**：连败 3 次禁用 edit，fallback 到整块 fresh send

状态机还会 suppress `<think>...</think>` 等 reasoning 标签（除非 `show_reasoning`）。

### ⑧ 收尾

Agent 返回 → 发最终 edit/fresh → gateway 执行：

- `clear_session_vars(tokens)` 显式清空 ContextVar
- 更新 `sessions.json` 与 `{session_id}.jsonl`
- 若配置 `deliver=telegram:xxx`，经 `DeliveryRouter` 投递额外目标；同时 `mirror_to_session` **反向写入目标会话 transcript**
- 触发 `agent:end` / `session:end` 等 hook
- 从 `_running_agents` 摘除；队列里还有 event 则立即开始下一轮

至此 Alice 看到一条从"正在思考..."慢慢变成"今天北京晴..."的消息，整条链路完成。

---

## 三、撑起这套流转的三大核心抽象

### 3.1 `BasePlatformAdapter`：把 20 个平台抹平成一个协议

**文件**：`gateway/platforms/base.py`

所有适配器继承同一基类，必须实现七个方法：

```python
class BasePlatformAdapter(ABC):
    async def connect(self) -> bool
    async def disconnect(self) -> None
    async def send(self, chat_id, text, **kw) -> SendResult
    async def send_typing(self, chat_id)
    async def send_image(self, chat_id, url, caption) -> SendResult
    async def get_chat_info(self, chat_id) -> dict
    async def edit(self, chat_id, message_id, text) -> SendResult
```

可选方法（`send_document/voice/video/animation` 等）有默认空实现。

**基类同时承担平台横向共性**，adapter 无需重复造轮子：

- **文本长度处理**：UTF-16 代码单元计数（Telegram 要求），surrogate pair 安全截断
- **代理与 SSRF**：HTTP/HTTPS 代理、macOS 系统代理自动检测（`scutil`）、loopback vs 公网检测
- **媒体缓存**：平台字节流写 `~/.hermes/cache/`，返回本地 `file://` URL
- **统一返回**：`SendResult = (success, message_id, error, platform_response)`

**加新平台的最小代价就是一个插件目录 + 一个继承 `BasePlatformAdapter` 的类**，核心代码零改动。

### 3.2 `SessionSource` → `session_key`：平台身份的确定性归一化

这里补一个跨平台同人识别的微妙点。

**Alice 在 Telegram 的 user_id=789、Slack 的 U0A1B2C3、飞书的 open_id——她们是同一个人吗？**

**Gateway 的立场是：不是。** 不同平台会话天然独立，两段独立 transcript、两个独立 Agent 实例。这是故意的——跨平台合并会引起隐私、一致性、竞争问题。

需要跨平台协作时走两个显式通道：

1. **共享长期记忆**：`~/.hermes/MEMORY.md` 进程级、不分平台
2. **主动镜像**：`mirror_to_session()` 把消息写入目标会话 transcript（典型场景：cron 往 Telegram 投递播报，同时镜像到该会话，让 Agent 下次对话能自然提及）

### 3.3 `ContextVar` 三态：并发安全的隐形护城河

第 ⑤ 步已经讲了为什么要用 `ContextVar`（asyncio 场景下 `os.environ` 和 `threading.local()` 都会串台）。这里补一个容易忽视的设计决策——**为什么是三态，不能简化成"有值 / 无值"的二态**？

因为工具代码要同时在三种语境下跑：

| 语境 | 特征 |
|------|------|
| CLI | 进程里根本没 gateway，ContextVar 从未被 set；但用户可能手动 `export` 环境变量 |
| Gateway | 每条消息 set 一次，处理完 clear 掉 |
| Cron | 后台 task 里 per-job set 自己的值 |

三态的分工：

- **`_UNSET`**：当前根本没关心过 → fallback 到 `os.environ`，让 CLI 用户 export 的变量生效
- **`""`**：关心过但已显式清空 → 返回 `""`，**不**穿透到 `os.environ`，防止 task 复用或事件循环调度后读到上次残留
- **具体值**：正常运行态

简化成二态会破坏其中一个语境：只留"有值 / 无值"就区分不出"gateway 清完了"和"CLI 从没关心过"，从而要么让 CLI 环境变量失效、要么让串台 bug 再次出现。

---

## 四、性能与可靠性的关键机制

### 4.1 Agent 实例缓存：保留 Prompt Cache 的必要基建

Anthropic/OpenAI 的 Prompt Caching 对**前缀稳定性**极度敏感——system prompt 一个字符变了整段 KV cache 就失效。而 Hermes 的 system prompt 包含记忆层、技能索引、Platform hint、工具集 schema，每次新建 Agent 都要重拼。

`_agent_cache` 按 `session_key` 缓存实例：

- `OrderedDict` LRU（容量 128）
- 空闲 > 3600s 驱逐
- Agent 附带 `config_sig` 哈希，用户改配置则签名对不上自动重建

### 4.2 edit-based 流式：没有推流协议时的假推流

| 痛点 | 对策 |
|------|------|
| Telegram editMessage 速率限制（~1/s） | `edit_interval=1.0s` + `buffer_threshold=40` 双闸门 |
| Flood control | adaptive backoff，3 次失败禁用 edit fallback |
| 预览挂太久时间线错乱 | Fresh Final：超 60s 改发新消息 |
| reasoning 想藏起来 | 状态机 suppress `<think>...</think>` |
| 工具调用插播活动提示 | `on_segment_break()` + commentary API |

结果：**用户在 Telegram 看到的体验和 ChatGPT 网页打字机效果基本一致，底层只用 `sendMessage` + `editMessageText`。**

### 4.3 双实例互斥：systemd 与 `--replace` 的三方博弈

常见运维场景：systemd 自动拉起、`hermes update` 升级、`hermes gateway restart` 热更、偶发的重复启动。不加保护会"双实例抢长轮询 token，50% 消息石沉大海"，甚至"重启风暴"。

Gateway 叠了三层锁：

1. **PID 文件**（`~/.hermes/gateway.pid`）：`O_CREAT|O_EXCL` 原子获取
2. **运行时锁**（`~/.hermes/gateway.lock`）：POSIX `fcntl` 独占锁，防崩溃残留导致假阳性
3. **Takeover Marker**：`--replace` 先写 marker 再发 SIGTERM，让旧进程分辨"计划内交接"（退出码 0）和"外部意外杀"（退出码 1），systemd 才不会无脑重启

### 4.4 中断的级联

长跑任务常被打断：`/stop`、空闲超时、SSE 断连、`/restart`、SIGUSR1 重载等。Gateway 归纳为六种 `_INTERRUPT_REASON_*`，通过 `agent.interrupt(reason=...)` 注入。Agent 在下一个可中断点抛 `InterruptedError`，finally 清理 terminal/browser/subprocess/credential，返回 `{interrupted: True, reason}`。

父 Agent 被中断时**级联**传播到所有子 Agent，详见 Subagent 白皮书。

### 4.5 自动续作（Auto-Continue）

中断不等于放弃。Gateway 在会话写 `resume_pending` 标记，用户下次发消息时若：

- 最后一条是 tool 结果 **或** 有 `resume_pending`
- **且** 距最后活动 < 1 小时（`HERMES_AUTO_CONTINUE_FRESHNESS` 可配）

则自动续作，超过阈值视为新请求——避免"昨天挂了的任务被今早的新消息误续"。

---

## 五、DM 配对码：不用静态白名单的用户授权

一个 Telegram bot 上线后 username 是公开可发现的，陌生人会主动 `/start`。静态白名单（`TELEGRAM_ALLOWED_USERS=...`）不够灵活。

方案是 **DM Pairing**：

1. 陌生 DM → `PairingStore.generate_code()` 生成 8 位配对码（32 字符去歧义字母表，排除 `0/O/1/I`，约 1.1×10¹² 组合）
2. Bot 回复："您的配对码是 `XK7P2Q9R`，请告知 bot 所有者"
3. 所有者执行 `hermes gateway approve XK7P2Q9R` → 写入 approved 列表
4. 下次 DM 被允许

安全参数（`gateway/pairing.py`）：TTL 1 小时；每用户 10 分钟限 1 次；每平台最多 3 个待审批；5 次失败锁定 1 小时；持久化文件 `chmod 0600`；配对码永不进日志。

**一个容易忽视的细节：锁定检查必须前置于 pending 码查找**，否则锁定期内合法码仍可被 approve，brute-force 保护失效。这是历史上修复过的一个 bug。

静态白名单仍被支持（`PlatformEntry.allowed_users_env`），两者互补。

---

## 六、扩展点

### 6.1 加一个新平台

**推荐插件路径**（零核心改动）：

```
~/.hermes/plugins/my-platform/
├── plugin.yaml         # name / description / events
├── __init__.py         # register(ctx): ctx.register_platform(...)
└── adapter.py          # class MyAdapter(BasePlatformAdapter)
```

在 `config.yaml::gateway` 启用即可，gateway 自动注册 `Platform("my-platform")` 伪成员。

**核心路径**仅为官方预装：需改 16 个集成点（`Platform` 枚举、`_create_adapter`、cron 投递、`send_message` 路由、prompt_builder 的 PLATFORM_HINTS 等），详见 `gateway/platforms/ADDING_A_PLATFORM.md`。

### 6.2 加一个事件钩子

文件系统级 Hook（`gateway/hooks.py`），放目录就生效：

```
~/.hermes/hooks/my-logger/
├── HOOK.yaml           # events: ["agent:end", "command:*"]
└── handler.py          # async def handle(event_type, context): ...
```

支持事件：`gateway:startup` / `session:start|end|reset` / `agent:start|step|end` / `command:*`。

典型用途：Prometheus exporter、敏感关键词拦截、跨会话广播。与 plugin hook 的区别：plugin hook 需写 manifest，gateway hook 丢目录即可，更适合 SRE 快速挂钩。

### 6.3 消息镜像与投递路由

让 Agent 的行动跨平台流动的两个基础工具：

- `DeliveryRouter.route("telegram:-1001234:42", text)`：把结果投递到**非当前会话**的任意聊天，target 格式 `{platform}[:{chat_id}[:{thread_id}]]`
- `mirror_to_session(platform, chat_id, text, source_label="cron")`：把投递的消息**反向写入目标会话 transcript**，让接收侧 Agent 下次运行时能看到外部事件

---

## 附录：关键文件索引

| 文件 | 作用 |
|------|------|
| `gateway/run.py` | 入口、`GatewayRunner` 主控、消息处理、中断/重启/关闭 |
| `gateway/config.py` | `Platform` 枚举、`PlatformConfig`、`SessionResetPolicy` |
| `gateway/session.py` | `SessionSource` / `SessionStore` / 会话键 / 重置策略 |
| `gateway/session_context.py` | **ContextVar 三态任务级会话隔离** |
| `gateway/stream_consumer.py` | 流式 edit 投递、节流、flood control、Fresh Final |
| `gateway/delivery.py` | `DeliveryTarget` 解析与多目标投递路由 |
| `gateway/pairing.py` | DM 配对码生成/审批、rate limit、lockout |
| `gateway/mirror.py` | 跨会话消息镜像 |
| `gateway/hooks.py` | 文件系统级事件钩子 |
| `gateway/platforms/base.py` | `BasePlatformAdapter` + 横向共性（UTF-16/代理/SSRF/缓存） |
| `gateway/platforms/api_server.py` | OpenAI 兼容 HTTP（chat/completions + responses + SSE） |
| `gateway/platforms/ADDING_A_PLATFORM.md` | 新平台接入清单 |
| `gateway/status.py` | PID 文件、运行时锁、Takeover marker |
| `hermes_cli/gateway.py` | CLI 子命令：`hermes gateway run/stop/restart/status/approve` |
| `hermes_cli/commands.py` | `COMMAND_REGISTRY` 全局命令真相源 |
