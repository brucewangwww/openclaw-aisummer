# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

mini-OpenClaw 是一个为期 10 天的课程项目，学生在此骨架基础上逐步构建一个 Claude Code 式的命令行 AI 智能体。整个项目使用 `# TODO[DayN]` 标记指导学生按天完成各模块。里程碑：v1（Day6）端到端可用 → v3（Day9）可扩展 → 终版（Day10）含安全层。

## 常用命令

```bash
# 环境准备
conda create -n openclaw python=3.11 && conda activate openclaw
pip install -r requirements.txt

# 骨架自检（Day1 即可跑，验证所有模块可导入）
python -m agent.cli --selfcheck

# 运行智能体（需要 DEEPSEEK_API_KEY 环境变量；未配 key 自动回退 FakeBackend）
python -m agent.cli "你的任务描述"

# 查看所有施工点
grep -rn "TODO\[Day" .
```

## 核心架构

### 数据流（ReAct 主循环）

```
用户任务 → AgentLoop.run()
  └─ while 未完成（最多 max_turns=20 轮）:
       backend.chat(messages, tools)  →  assistant 消息
       ├─ 含 tool_calls →  ToolRegistry.get(name).run(**args)  →  结果注入 messages
       └─ 无 tool_calls  →  返回 content 作为最终答案
```

### 模块间接口约定

**Backend → Loop**：`chat(messages, tools) -> dict` 返回归一化格式：
```python
{"role": "assistant", "content": str, "tool_calls": [{"id": str, "name": str, "arguments": dict}]}
```
`DeepSeekBackend`（真实 API）和 `FakeBackend`（离线开发）都遵循此接口，主循环不关心具体后端。

**Tool → Loop**：每个工具是 `Tool` 数据类（name / description / parameters / run 可调用对象）。`run(**arguments) -> str` 返回文本 observation，由主循环以 `role="tool"` 注入消息历史。

**提示词渲染**（`prompt/render.py`，Day3）：核心认知——模型看到的不是消息列表，而是一段拼接好的纯文本。`render_prompt` 用角色标记包裹消息，`parse_tool_calls` 从模型输出中手动提取 `<tool_call>` 标签，不依赖 API 的 function-calling 能力。

### 关键设计模式

- **ToolRegistry**：工具按名称注册的字典，提供 `register()`、`get()`、`schemas()`（供发送给 API）。`build_default_registry()` 是工厂函数，随课程推进逐步取消注释以激活各工具。
- **MCP 集成**（Day8）：MCP 工具以 `mcp__` 前缀注册到同一个 ToolRegistry，对主循环透明——模型可以像调用内置工具一样调用外部 MCP server 的工具。
- **SKILL.md 格式**：YAML frontmatter（name + description）+ Markdown 正文。`skills/loader.py` 扫描 `*/SKILL.md`，生成可注入系统提示词的能力清单。Skill 不同于 Tool——它是一包领域知识和工作流程，而非单次函数调用。
- **上下文管理**（Day7）：`maybe_compact()` 在超 token 预算时将早期消息摘要为 system 备忘；`truncate_observation()` 截断过长的工具结果。

## 环境变量

| 变量 | 用途 | 默认值 |
|------|------|--------|
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥（必须） | — |
| `DEEPSEEK_BASE_URL` | API 地址 | `https://api.deepseek.com` |
| `DEEPSEEK_MODEL` | 模型名称 | `deepseek-chat` |

## 逐日构建节奏

| 天 | 模块 | 关键交付 |
|---|------|---------|
| Day1–2 | `backend/` | API 客户端跑通，FakeBackend 可用 |
| Day3 | `prompt/` | `render_prompt` + `parse_tool_calls` 手动实现 |
| Day5 | `agent/` + `tools/` | ReAct 主循环 + read/write/bash 工具 |
| Day6 | `tools/` | edit/grep/glob → v1 里程碑（端到端可用） |
| Day7 | `agent/` + `tools/` | 上下文管理 + web_fetch/task_list |
| Day8 | `mcp/` | MCP 客户端（stdio + JSON-RPC） |
| Day9 | `skills/` | 技能加载器 + 自定义领域 Skill |
| Day10 | `eval/` | 评测 + 安全层 → 终版 |

课程按 day 打 git tag（`v1`, `v3`, `final`）。各模块自带 README.md 用于记录设计决策。
