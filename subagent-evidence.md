# 子 Agent 调用与边界证据

> 自动生成，非手工编写。原始数据见 [subagent-evidence.json](subagent-evidence.json)。

## 证据来源（重要）

- 模型：`ScriptedModel`，**没有调用真实 LLM**。
- 图：真实 `create_deep_agent` 图。
- 子 Agent：真实 `task()` 子图执行，包含真实的 middleware 与工具调用。
- 边界：真实 `FilesystemPermission` 与 `HumanInTheLoop` 中断。

因此本证据证明的是**机制**，不是"某次真实模型回答的质量"。

## 1. 主 Agent 委派

| 项目 | 值 |
| --- | --- |
| thread | `evidence-subagent` |
| 用户消息 | Plan the research and delegate one fact check. |
| 子 Agent 类型 | `research-agent` |
| 委派描述 | Verify one LangGraph 1.0 fact and return it with a source URL. |
| 主 Agent 工具序列 | write_todos → task → write_file |

## 2. 子 Agent 回传

回传长度：**179 字符**

```markdown
## Key Findings

LangGraph 1.0 adds durable execution and first-class interrupts [1].

### Sources
[1] LangGraph overview: https://docs.langchain.com/oss/python/langgraph/overview
```

## 3. 上下文边界证据

| 检查 | 结果 |
| --- | --- |
| 子 Agent 使用独立 system prompt | True |
| 子 Agent 提示里出现委派描述 | True |
| 子 Agent 提示里出现主线程用户消息 | False |

最后一项为 `False` 才说明上下文隔离生效：主线程的原始对话没有泄漏进子 Agent。

## 4. 权限边界证据

模型尝试写入只读技能目录，工具返回：

```text
Error: permission denied for write on /skills/rogue/SKILL.md
```

## 5. 人工审批证据

调用 `publish_report` 后，图在工具执行前暂停：

```json
{
  "markdown": "# Draft",
  "filename": "draft.md"
}
```

允许的决定：approve, edit, reject, respond

审批前的报告文件是否存在：`False`（应为 `False`，证明暂停发生在副作用之前）。

## 6. 环境

```json
{
  "python": "3.12.13",
  "deepagents": "0.7.8",
  "langchain": "1.4.3",
  "platform": "win32"
}
```
