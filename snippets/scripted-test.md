# 节选：只脚本化模型，其余全真

目标：在没有 API key 的情况下，验证"机制"而不是"模型回答得好不好"。

## 脚本化模型

```python
from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.messages import AIMessage
from langchain_core.outputs import ChatGeneration, ChatResult
from pydantic import PrivateAttr


class ScriptedModel(BaseChatModel):
    replies: list[AIMessage]
    seen_tools: list[str] = []
    prompts: list[str] = []
    _cursor: int = PrivateAttr(default=0)

    def bind_tools(self, tools, **kwargs):
        self.seen_tools = [t.name if hasattr(t, "name") else t["name"] for t in tools]
        return self

    def _generate(self, messages, stop=None, run_manager=None, **kwargs):
        self.prompts.append("\n".join(str(m.content) for m in messages))
        reply = self.replies[self._cursor]
        self._cursor += 1
        return ChatResult(generations=[ChatGeneration(message=reply)])
```

三个字段各有用途：

- `seen_tools`：断言工具真的注册进了模型；
- `prompts`：断言技能索引、记忆正文真的进了系统提示；
- `_cursor`：按顺序吐出预设回复，让多轮工具调用可控。

## 权限测试

```python
model = ScriptedModel(replies=[
    call("write_file", {"file_path": "/skills/rogue/SKILL.md", "content": "x"}, "deny"),
    AIMessage(content="Denied."),
])
graph = build_graph(model, checkpointer=InMemorySaver())
result = graph.invoke(
    {"messages": [{"role": "user", "content": "Try to edit a skill."}]},
    {"configurable": {"thread_id": "permissions"}},
)

tool_messages = [m for m in result["messages"] if getattr(m, "type", "") == "tool"]
assert "permission denied" in str(tool_messages[-1].content).lower()
```

## 审批中断测试

```python
model = ScriptedModel(replies=[
    call("publish_report", {"markdown": "# Draft", "filename": "draft.md"}, "publish"),
    AIMessage(content="Published."),
])
graph = build_graph(model, checkpointer=InMemorySaver())
result = graph.invoke(
    {"messages": [{"role": "user", "content": "Publish."}]},
    {"configurable": {"thread_id": "hitl"}},
)

request = result["__interrupt__"][0].value["action_requests"][0]
assert request["name"] == "publish_report"
```

## 为什么不用 mock 图

mock 掉 middleware 会掩盖真正的问题：顺序错了、参数没传、权限没挂上。脚本化模型只替换"最不可控"的那一层，其余保持真实，才能测出集成错误。

## 成本

11 个测试约 80–170 秒，零密钥。冷启动最慢，之后稳定。
