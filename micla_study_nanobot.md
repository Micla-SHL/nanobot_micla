# 🐈 nanobot 学习笔记

> 作者：Micla  
> 仓库：[HKUDS/nanobot](https://github.com/HKUDS/nanobot)  
> 学习日期：2026-04

---

## 一、项目概览

### 一句话简介

nanobot 是一个**极简的个人 AI Agent 框架**——用约 4,000 行 Python 代码实现了完整的 ReAct 循环、持久记忆、多渠道接入和技能扩展体系，定位是"可读、可改、可研究"的轻量版 Claude.ai / OpenClaw。

> ⚡️ 比 OpenClaw 少 99% 的代码，功能覆盖率却极高。

### 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.11+（主）、TypeScript（WhatsApp Bridge）、Shell |
| LLM 接入 | OpenRouter（默认代理）、Anthropic、Azure、VolcEngine 等 |
| CLI | Typer |
| 日志 | Loguru |
| 富文本终端 | Rich |
| 调度 | APScheduler |
| 依赖管理 | pyproject.toml |

### 安装

```bash
# 从 PyPI 安装（稳定版）
pip install nanobot-ai

# 从源码安装（最新特性，推荐开发用）
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
pip install -e .
```

### 快速上手

```bash
nanobot onboard           # 初始化配置 ~/.nanobot/config.json
nanobot agent -m "你好"   # 单次对话
nanobot agent             # 交互模式 REPL
nanobot gateway           # 启动 Telegram/WhatsApp 网关
```

**最简配置（~/.nanobot/config.json）：**

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-xxx"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5"
    }
  }
}
```

> 🇨🇳 国内用户提示：可将 model 改为 `minimax/minimax-m2` 降低费用。

---

## 二、项目结构解析

```
nanobot/                  ← Python 包根目录
├── agent/                ← 🧠 核心：整个 Agent 的大脑
│   ├── loop.py           ←   ReAct 主循环（最重要的文件）
│   ├── context.py        ←   Prompt 组装器
│   ├── memory.py         ←   持久记忆读写与压缩
│   ├── skills.py         ←   技能加载器
│   ├── subagent.py       ←   后台异步子任务
│   └── tools/            ←   内置工具集（shell、文件、搜索、MCP...）
├── skills/               ← 🎯 预置技能包（weather、github、memory...）
├── channels/             ← 📱 消息渠道适配（Telegram、WhatsApp、WeChat...）
├── bus/                  ← 🚌 内部消息总线（pub/sub 解耦）
├── cron/                 ← ⏰ 定时任务调度
├── heartbeat/            ← 💓 主动唤醒（Agent 自发触发）
├── providers/            ← 🤖 LLM Provider 抽象层
├── session/              ← 💬 多会话管理
├── config/               ← ⚙️ 配置解析（Pydantic schema）
├── cli/                  ← 🖥️ 命令行入口（Typer）
├── bridge/               ← WhatsApp Node.js 桥接层
├── workspace/            ← 默认工作区模板
└── case/                 ← 示例/用例
```

---

## 三、核心架构：消息流

```
用户消息
  │
  ▼
[channels] → Telegram / WhatsApp / WeChat / Slack / Discord / CLI...
  │  （统一封装为 InboundMessage）
  ▼
[bus] → 消息总线（异步 pub/sub，解耦渠道与 Agent）
  │
  ▼
[agent/loop.py]  ←  🔁 ReAct 循环核心
  │  ┌─────────────────────────────────────────┐
  │  │ 1. context.py → 组装 System Prompt      │
  │  │    (身份 + 记忆 + 技能列表 + 对话历史)   │
  │  │ 2. providers  → 调用 LLM                │
  │  │ 3. 有 tool_call → 执行工具（并发）       │
  │  │    └→ tools/ (shell/web/file/MCP/spawn) │
  │  │ 4. 无 tool_call → 输出回复，退出循环     │
  │  └─────────────────────────────────────────┘
  │
  ▼
[session] → 存储对话历史
[memory]  → 异步压缩长期记忆（MEMORY.md + HISTORY.md）
  │
  ▼
[channels] → 回复用户
```

---

## 四、核心模块精读：`agent/loop.py`

### 4.1 AgentLoop 类结构

```
AgentLoop
│
├── __init__()               # 初始化：Provider、工具、内存、会话、SubAgent
├── _register_default_tools()  # 注册内置工具（loop.py:115-131）
├── _connect_mcp()            # 懒加载 MCP 工具（loop.py:133-153）
│
├── 【入口层】
│   ├── run()                # 订阅 MessageBus，持续监听（网关模式）
│   └── process_direct()     # 单次处理一条消息（CLI 直接模式）
│
├── 【调度层】
│   └── _process_message()   # 单条消息完整处理流程
│
└── 【核心循环】
    └── _run_agent_loop()    # LLM ↔ 工具的反复迭代（ReAct 心跳）
```

### 4.2 `_process_message()` 主流程

每条用户消息进来时的完整处理顺序：

```
用户消息进来
    │
    ├─ 1. 特殊命令拦截
    │       /new  → 清空当前 session（保留记忆摘要）
    │       /stop → 取消所有 subagent 后台任务
    │       /help → 返回命令列表
    │
    ├─ 2. _set_tool_context()
    │       给 MessageTool / SpawnTool 注入当前 channel 和 chat_id
    │       （loop.py:155-160，每条消息开始前必须调用）
    │
    ├─ 3. _connect_mcp()
    │       首次消息时懒加载 MCP 工具，失败则下次重试
    │
    ├─ 4. memory_consolidator.maybe_consolidate_by_tokens(session)
    │       如果 session token 接近上限，先压缩历史记忆
    │
    ├─ 5. session.get_history() + context.build_messages()
    │       取出对话历史，组装成完整消息列表（含 system prompt）
    │
    ├─ 6. _run_agent_loop()     ← 🔁 核心 ReAct 循环
    │
    ├─ 7. 保存新的对话轮次到 session
    │
    └─ 8. 发送回复 → MessageBus → 路由回用户渠道
```

### 4.3 `_run_agent_loop()` ReAct 核心

```python
# 伪代码
async def _run_agent_loop(messages, ...):
    for iteration in range(max_tool_iterations):  # 默认最多 40 轮
        
        # Step 1: 调用 LLM
        response = await provider.chat(
            messages=messages,
            tools=tool_registry.get_definitions(),  # OpenAI function calling schema
        )
        
        if response.tool_calls:
            # LLM 想用工具 → 执行工具（多个并发）
            tool_results = await _execute_tool_calls(response.tool_calls)
            
            # 追加到消息链，进入下一轮
            messages.append(response)       # assistant turn
            messages.append(tool_results)   # tool turn
            continue
            
        else:
            # LLM 返回纯文本 → 本轮结束
            break
    
    return final_content, messages
```

**退出条件：**
1. ✅ LLM 返回纯文本（无 tool_call）→ 正常结束
2. ⚠️ 迭代次数达到 `max_tool_iterations`（默认 40）→ 强制截断
3. ❌ 发生异常 → 清理后退出

### 4.4 工具执行：`_execute_tool_calls()`

```
LLM 返回 [tool_call_A, tool_call_B, tool_call_C]
    │
    ├─ 参数校验（Tool.validate_params()）
    │       失败 → 返回错误字符串给 LLM，继续下一轮
    │
    ├─ 并发执行：asyncio.gather(exec_A, exec_B, exec_C)
    │
    ├─ 结果截断：保存到 session 最多 500 字符
    │   （_TOOL_RESULT_MAX_CHARS = 500，loop.py:47）
    │   注意：完整结果仍传给当前 LLM，截断只影响历史持久化
    │
    └─ 返回 tool_result 消息追加回 messages
```

### 4.5 内置工具清单

| 工具 | 功能 | 备注 |
|------|------|------|
| `ReadFileTool` | 读文件内容 | 支持 workspace 沙箱限制 |
| `WriteFileTool` | 写/覆盖文件 | |
| `EditFileTool` | 打补丁修改（高效） | 适合大文件局部修改 |
| `ListDirTool` | 列目录结构 | |
| `ExecTool` | 执行 Shell 命令 | 支持 deny_patterns 黑名单 |
| `WebSearchTool` | 网络搜索 | 需 Brave Search API Key |
| `WebFetchTool` | 抓取网页内容 | |
| `MessageTool` | 主动发消息到渠道 | 含防重复发送机制 |
| `SpawnTool` | 派生 SubAgent 后台任务 | 非阻塞 |
| `CronTool` | 添加定时任务 | 仅 gateway 模式 |

**MCP 工具：懒加载。** 第一条消息到来时才连接 MCP Server，避免启动阻塞。

---

## 五、设计亮点

### 5.1 Skills = Markdown 文件

nanobot 没有注册表，技能就是 `workspace/skills/<name>/SKILL.md`，Agent 自己用 `read_file` 工具读取。扩展技能只需写一个 Markdown 文件，**零代码**。

```
workspace/skills/
├── weather/
│   └── SKILL.md    ← 告诉 Agent 如何查天气
├── github/
│   └── SKILL.md    ← 告诉 Agent 如何操作 GitHub
└── my-custom-skill/
    └── SKILL.md    ← 你自己写的技能
```

### 5.2 记忆两级存储

```
HISTORY.md   ← 全量对话日志，grep 友好，按时间戳存储
MEMORY.md    ← LLM 压缩后的语义摘要（精华记忆）
```

当 session token 超限时，`MemoryConsolidator` 自动触发压缩：把旧历史总结进 MEMORY.md，清空 session。

**MEMORY.md 注入方式（context.py）：**
```python
memory = self.memory.get_memory_context()
if memory:
    parts.append(f"# Memory\n\n{memory}")  # 直接塞进 system prompt
```

### 5.3 Bus 解耦

所有渠道统一发消息到内部 MessageBus，AgentLoop 只订阅总线。新增渠道（比如接入企业微信）完全不需要改 Agent 代码。

### 5.4 MessageTool 防重复回复

LLM 可以主动调用 `message` 工具发消息，也可以直接返回文本（AgentLoop 自动发）。防重复机制：`_sent_in_turn` 标志——如果 LLM 已通过工具发消息到**同一目标**，AgentLoop 不再自动发送。跨渠道转发不受影响。

### 5.5 SubAgent 非阻塞后台任务

```
主 Agent 收到耗时任务
    │
    ├─ LLM 调用 spawn(task="...", label="xxx")
    ├─ SubagentManager 创建 asyncio.Task（后台）
    │   └─ 独立 AgentLoop，工具集受限（无 spawn/message）
    ├─ 主 Agent 立刻回复："任务已启动，完成后通知你"
    └─ [后台完成] → 注入 InboundMessage(channel="system") → 通知用户
```

---

## 六、已知 Bug & 学习价值（Issue #2638）

### Bug：session 历史无限增长

来自 [Issue #2638](https://github.com/HKUDS/nanobot/issues/2638)，揭示了 `_process_message` 里的微妙问题：

```python
# loop.py
await self.memory_consolidator.maybe_consolidate_by_tokens(session)
history = session.get_history(max_messages=0)  # Python 坑！
```

`max_messages=0` 的底层实现是 `list[-0:]`，等价于 `list[0:]`——返回全部内容。

更严重的是 `maybe_consolidate_by_tokens` 里：
```python
estimated, source = self.estimate_session_prompt_tokens(session)
if estimated <= 0:
    return  # ← 静默跳过！session 继续增长
```

如果 token 估算异常返回 0，压缩被跳过，session 无限增长，最终 LLM 上下文溢出，Agent 失去响应。

**v0.1.5 已修复。** 这个 Bug 是学习"生产级 Agent 系统如何做健壮性设计"的好案例。

---

## 七、学习路径（推荐阅读顺序）

```
阶段 1：理解项目（30min）
- [ ] README.md              — 项目目标、安装、CLI 命令
- [ ] pyproject.toml         — 依赖一览

阶段 2：跑起来（1h）
- [ ] nanobot onboard        — 跑通 onboard
- [ ] nanobot agent -m "hi"  — 第一次对话
- [ ] ~/.nanobot/config.json — 理解配置结构

阶段 3：读核心逻辑（2-3h）
- [ ] agent/loop.py          — ReAct 循环（最重要）
- [ ] agent/context.py       — Prompt 组装方式
- [ ] agent/memory.py        — 记忆存储与压缩逻辑

阶段 4：理解扩展机制（1h）
- [ ] agent/tools/base.py    — Tool 基类，理解如何自定义工具
- [ ] agent/skills.py        — Skills 加载机制
- [ ] skills/weather/SKILL.md — 一个真实技能的写法

阶段 5：扩展实践（自由）
- [ ] 写一个自定义 Tool
- [ ] 写一个自定义 Skill（SKILL.md）
- [ ] 接入 Telegram
```

---

## 八、CLI 命令速查

| 命令 | 说明 |
|------|------|
| `nanobot onboard` | 初始化配置和工作区 |
| `nanobot agent -m "..."` | 单次对话 |
| `nanobot agent` | 交互模式 |
| `nanobot gateway` | 启动网关（Telegram/WhatsApp 等） |
| `nanobot status` | 查看状态 |
| `nanobot channels login` | 连接 WhatsApp（扫码） |
| `nanobot cron add --name "daily" --message "早报" --cron "0 9 * * *"` | 添加定时任务 |
| `nanobot cron list` | 查看定时任务 |
| `nanobot cron remove <job_id>` | 删除定时任务 |

---

## 九、待研究问题

- [ ] `context.py` 里 System Prompt 的完整拼装流程（身份 + 记忆 + 技能 + Bootstrap）
- [ ] `memory.py` 中 `MemoryConsolidator` 的压缩触发时机和策略
- [ ] MCP 工具的注册和调用流程（`agent/tools/mcp.py`）
- [ ] HeartbeatService 的主动唤醒机制（`heartbeat/service.py`）
- [ ] 如何自定义一个完整的 Channel 插件

---

## 十、参考资源

- 官方仓库：https://github.com/HKUDS/nanobot
- DeepWiki（自动生成的代码文档）：https://deepwiki.com/HKUDS/nanobot
- MarkTechPost 深度教程：https://www.marktechpost.com/2026/03/28/a-coding-guide-to-exploring-nanobots-full-agent-pipeline
- 我的 fork：https://github.com/Micla-SHL/nanobot_micla

---

*本笔记由学习对话整理生成，持续更新中。*
