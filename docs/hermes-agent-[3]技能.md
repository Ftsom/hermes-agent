# Hermes Agent 技能系统技术白皮书

> 版本：2.0 | 日期：2026-04-24

---

## 一、定位与核心概念

技能（Skill）是 Hermes Agent 的**程序性记忆**。它以文件形式封装"如何完成某类任务"的方法论，并在每次对话中主动提供给 Agent 参考。

与其他知识层次的区别：

| 层次 | 形式 | 描述的是 |
|------|------|---------|
| 记忆（Memory） | MEMORY.md | 用户偏好、环境事实等声明性知识 |
| **技能（Skill）** | **SKILL.md** | **可复用的操作流程、专项工作流** |
| 工具（Tool） | Python 代码 | 可调用的原子能力 |

---

## 二、技能的结构

### 文件布局

每个技能是 `~/.hermes/skills/<category>/<skill-name>/` 下的一个目录：

```
skill-name/
├── SKILL.md          # 必需，技能主体
├── references/       # 参考文档
├── templates/        # 输出模板
├── scripts/          # 辅助脚本
└── assets/           # 其他资产
```

### SKILL.md Frontmatter

```yaml
---
name: skill-name                 # 必需，^[a-z0-9][a-z0-9._-]*$，≤64 字符
description: 一句话描述           # 必需，≤1024 字符，用于索引展示
version: 1.0.0
platforms: [macos, linux]        # 限制运行平台，省略则全平台可用

required_environment_variables:  # 运行所需的 Secrets（存入 ~/.hermes/.env）
  - name: MY_API_KEY
    prompt: "请输入 API Key"
    help: "https://example.com"
    optional: false              # false 时缺失则阻断加载

required_credential_files:       # 运行所需的凭据文件
  - path: google_token.json
    description: "Google OAuth2 令牌"

metadata:
  hermes:
    tags: [llm, fine-tuning]
    related_skills: [peft, lora]
    requires_toolsets: [web]          # 仅当指定 toolset 可用时在索引中显示
    requires_tools: [web_search]      # 仅当指定工具可用时在索引中显示
    fallback_for_toolsets: [browser]  # 当指定 toolset 存在时隐藏（自身是 fallback）
    fallback_for_tools: [browser_navigate]
    config:                           # 非敏感配置（存入 config.yaml）
      - key: plugin.path
        description: "数据目录"
        default: "~/data"
---

# 技能正文
操作步骤、注意事项、验证方法……
```

---

## 三、Prompt 集成

技能通过两个层次影响 Agent 的行为。

### 3.1 系统提示：技能索引（每次对话）

`build_skills_system_prompt()` 在系统提示中注入 `## Skills (mandatory)` 区块，包含两部分：

**① 强制指令**（指导 Agent 何时、如何使用技能）：

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially
relevant to your task, you MUST load it with skill_view(name) and follow its
instructions. Err on the side of loading — it is always better to have context
you don't need than to miss critical steps, pitfalls, or established workflows.
Skills also encode the user's preferred approach, conventions, and quality
standards — load them even for tasks you already know how to do, because the
skill defines how it should be done here.
If a skill has issues, fix it with skill_manage(action='patch').
After difficult/iterative tasks, offer to save as a skill.

<available_skills>
  software-development:
    - systematic-debugging: 系统化调试流程
    - writing-plans: 编码前的规划方法
  mlops:
    - axolotl: LLM fine-tuning with Axolotl，支持 LoRA/QLoRA/DPO/GRPO
</available_skills>

Only proceed without loading a skill if genuinely none are relevant to the task.
```

**② 技能维护引导**（`SKILLS_GUIDANCE` 常量）：

```
After completing a complex task (5+ tool calls), fixing a tricky error, or
discovering a non-trivial workflow, save the approach as a skill with skill_manage.
When using a skill and finding it outdated, patch it immediately — don't wait to be asked.
Skills that aren't maintained become liabilities.
```

**索引的设计逻辑**：
- `<available_skills>` 只展示 name + description，极简，最小化 token 开销
- 完整内容通过 `skill_view()` 按需加载（渐进式披露）
- 指令使用强制语气（MUST、Err on the side of loading），降低漏加载风险

**条件显隐**（`_skill_should_show()`）：

| frontmatter 字段 | 隐藏条件 |
|-----------------|---------|
| `fallback_for_toolsets: [X]` | toolset X **存在**时隐藏 |
| `fallback_for_tools: [X]` | 工具 X **存在**时隐藏 |
| `requires_toolsets: [X]` | toolset X **不存在**时隐藏 |
| `requires_tools: [X]` | 工具 X **不存在**时隐藏 |

### 3.2 用户消息：技能激活消息（按需注入）

Agent 调用 `skill_view()` 或用户使用斜杠命令后，`_build_skill_message()` 将以下内容组装为一条 **user message** 注入对话：

```
[SYSTEM: The user has invoked the "skill-name" skill, indicating they want
you to follow its instructions. The full skill content is loaded below.]

<SKILL.md 完整内容，含 frontmatter>

[Skill config (from ~/.hermes/config.yaml):
  MY_API_KEY = abc123
  OTHER_KEY = (not set)
]

[Skill setup note: Setup needed: missing env $SOME_VAR. Get your key at https://...]

[This skill has supporting files you can load with the skill_view tool:]
- references/api.md
- templates/config.yaml
To view any of these, use: skill_view(name="skill-name", file_path="<path>")
```

**激活注释的两种变体**：

| 触发方式 | 激活注释 |
|---------|---------|
| 斜杠命令 `/skill-name` | `The user has invoked the "X" skill...` |
| 启动时预加载 `--skill X` | `The user launched this session with "X" preloaded. Treat its instructions as active guidance for the duration of this session...` |

注意：消息以 `[SYSTEM: ...]` 标记，但置于 user role——让 Agent 将其视为操作指令，而非背景知识。

---

## 四、存储与发现

### 存储位置

| 位置 | 说明 |
|------|-----|
| `~/.hermes/skills/` | 主目录，唯一写入位置 |
| `skills.external_dirs`（config.yaml） | 附加目录，只读 |
| 捆绑技能（随发行版附带） | 通过 `skills_sync.py` 同步到主目录 |

### 两层缓存

系统提示索引维护两层缓存，避免每次对话重复扫描：

```
L1：进程内 LRU 字典（max 8 条）
    Key: (skills_dir, external_dirs, tools, toolsets, platform)
       ↓ miss
L2：磁盘快照（~/.hermes/.skills_prompt_snapshot.json）
    通过所有 SKILL.md 的 mtime/size 验证有效性
```

任何 CRUD 操作后，`clear_skills_system_prompt_cache()` 同时清除两层。

---

## 五、生命周期管理

所有 CRUD 操作通过 `skill_manage()`（`skill_manager_tool.py`）统一入口：

| Action | 说明 |
|--------|-----|
| `create` | 原子性创建目录 + SKILL.md，写后安全扫描 |
| `edit` | 完整重写 SKILL.md，扫描失败自动回滚 |
| `patch` | 模糊查找替换（容忍空白差异），适合小范围更新 |
| `delete` | 删除目录，自动清理空 category 目录 |
| `write_file` | 添加/覆盖 references/、templates/、scripts/、assets/ 下的文件 |
| `remove_file` | 删除支撑文件 |

所有写操作使用原子写入：先写临时文件，再 `os.replace()`，防止中途崩溃损坏文件。

**Agent 自动创建技能的触发时机**（由系统提示引导）：
- 完成复杂任务（5+ 次工具调用）
- 克服了非显而易见的错误
- 发现了特定工具/API 的专项工作流
- 用户明确要求记录某个流程

---

## 六、Hub 与安全

### 技能来源与信任等级

| 等级 | 来源 | 安全策略 |
|------|-----|---------|
| `builtin` | 随 Hermes 发行版附带 | 永远信任，跳过扫描 |
| `trusted` | `openai/skills`、`anthropics/skills` | 警告级别允许，危险级别阻断 |
| `community` | 其他 GitHub/Hub 来源 | 任何发现均阻断（除非 `--force`） |
| `agent-created` | Agent 自动创建 | 危险发现阻断，警告需用户确认 |

### 安全扫描

`skills_guard.py` 对技能文件做静态注入模式扫描，触发时机：`skill_view()` 加载时、`skill_manage(create/edit)` 写入后。

结构限制：最多 50 个文件，总大小 ≤ 1024 KB，单文件 ≤ 256 KB。

### Hub CLI

```bash
hermes skills browse          # 搜索 Hub
hermes skills install <name>  # 安装技能
hermes skills list            # 列出本地技能
hermes skills tap <repo>      # 添加外部源
hermes skills publish         # 发布技能
```

支持的注册中心：官方 optional-skills、任意 GitHub 仓库、ClaW Hub、Claude Marketplace、Lobe Hub。

---

## 七、配置参考

**`~/.hermes/config.yaml`**：

```yaml
skills:
  disabled: [godmode]          # 全局禁用
  platform_disabled:           # 按平台禁用
    telegram: [heavy-skill]
  external_dirs:               # 附加技能目录（只读）
    - ~/team-skills
  config:                      # 技能非敏感配置
    plugin:
      path: ~/data
```

**关键环境变量**：

| 变量 | 作用 |
|------|-----|
| `HERMES_BUNDLED_SKILLS` | 覆盖捆绑技能路径 |
| `TERMINAL_ENV` | 终端后端类型（local/docker/modal），影响 setup notes |
| `HERMES_PLATFORM` | 按平台过滤技能 |

---

## 八、核心文件

| 文件 | 职责 |
|------|-----|
| `agent/prompt_builder.py` | 系统提示组装、技能索引、条件过滤、两层缓存 |
| `agent/skill_commands.py` | 斜杠命令注册、激活消息构建、会话预加载 |
| `agent/skill_utils.py` | frontmatter 解析、平台匹配、禁用列表、外部目录 |
| `tools/skills_tool.py` | `skills_list`、`skill_view`，env var 管理，安全防护 |
| `tools/skill_manager_tool.py` | `skill_manage` CRUD，原子写入，自动回滚 |
| `tools/skills_guard.py` | 注入模式扫描、信任策略 |
| `tools/skills_hub.py` | Hub 适配器、数据模型、GitHub 认证、lock 文件 |
| `tools/skills_sync.py` | 捆绑技能清单同步 |

---

## 九、工作流总览

```
每次对话
  └─ 系统提示注入
       ├─ ## Skills (mandatory) + <available_skills> 索引
       └─ SKILLS_GUIDANCE 维护引导
            │
            │ Agent 判断任务相关
            ▼
       skill_view(name)
       ├─ 平台 / 禁用 / 注入安全检查
       ├─ 缺失 env var → 交互式采集 → 写入 ~/.hermes/.env
       └─ 返回完整 SKILL.md + linked_files 列表
            │
            ▼ 注入为 user message
       [SYSTEM: activation_note]
       <SKILL.md 内容>
       [Skill config / setup note / supporting files]
            │
            ▼
       Agent 按技能指令执行
            │
     ┌──────┴──────┐
     │             │
  技能过时？      任务复杂（5+）？
     ↓             ↓
  patch         create
     └──────┬──────┘
            ↓
     清除两层缓存，技能持久化
```
