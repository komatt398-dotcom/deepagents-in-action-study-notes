# 工程实践：把 16 章能力组装成一个 Agent

> 本文记录"怎么装、为什么这样装、怎么验证"。代码是节选，完整实现见配套工程。

---

## 1. 目标与约束

目标：把课程里分散的能力，组装成**一个** `build_graph()` 调用，同时满足四个约束：

1. **向后兼容**：原有调用方只传 `model` 和 `checkpointer`，不能改签名。
2. **可测试**：没有模型密钥也能验证机制。
3. **可降级**：当前版本没有的能力，不能导致 import 失败。
4. **安全默认**：危险动作默认需要审批，只读目录默认不可写。

## 2. 能力分层

| 层面 | 章节 | 实现 | 默认状态 |
| --- | --- | --- | --- |
| 虚拟文件系统 | ch03 | `CompositeBackend`：线程态 + Store + 真实文件 | 开启 |
| 任务规划 | ch04 | `TodoListMiddleware`，`todos` 进入状态流 | 开启 |
| 专业子 Agent | ch05 | `research-agent` | 开启 |
| 只读审计子 Agent | ch05 | `report-reviewer`，写权限全拒 | 开启 |
| 异步子 Agent | ch06 | 远程 Agent Protocol 规格 | 环境变量开启 |
| 技能 | ch07 | `/skills/` 只读路由 + `SKILL.md` | 开启 |
| 长期记忆 | ch08 | `/memories/` + `StoreBackend` | 开启 |
| 人工审批 | ch09 | `interrupt_on` 发布 / 发信 | 开启 |
| 沙箱 | ch10 | `LocalShellBackend`（免 Docker） | 环境变量开启 |
| 权限 | ch11 | `FilesystemPermission` | 开启 |
| MCP | ch12 | FastMCP stdio 服务 + 适配器加载 | 环境变量开启 |
| 评分量表 | ch13 | `RubricMiddleware` + 取证工具 | 环境变量开启 |
| 解释器 / 动态子 Agent | ch15/16 | `CodeInterpreterMiddleware` | 当前版本不可用 |

## 3. 后端路由

核心是"一条路径，三种存储"：

```text
/                    → StateBackend        线程内临时文件（报告草稿、请求）
/memories/           → StoreBackend        跨线程持久记忆
/skills/             → FilesystemBackend   只读技能目录（真实文件）
/policies/           → FilesystemBackend   只读规范目录（真实文件）
```

这样设计的好处：

- `/final_report.md` 跟着线程走，不会污染本地目录。
- `/memories/preferences.md` 跨线程复用，但不会无限增长到状态里。
- `/skills/` 改文件即可生效，不需要重新打包代码；写操作被权限拒绝。

节选见 [snippets/backend-routing.md](../snippets/backend-routing.md)。

## 4. 记忆种子与幂等

记忆文件在构建图时写入，但只在不存在时写：

```python
for key, content in MEMORY_FILES.items():
    if store.get(NAMESPACE, key) is None:
        store.put(NAMESPACE, key, create_file_data(content))
```

这一步很容易写错成"每次启动覆盖"，后果是用户昨天教它的偏好，今天开机就没了。测试里专门有一条断言：先写入 `learned`，再执行种子，内容必须仍是 `learned`。

## 5. 权限与审批

三类规则，按"最小惊讶"排列：

```python
FilesystemPermission(operations=["write"], paths=["/skills/**"],   mode="deny")
FilesystemPermission(operations=["write"], paths=["/policies/**"], mode="deny")
FilesystemPermission(operations=["write"], paths=["/secrets/**"],  mode="interrupt")

interrupt_on = {"publish_report": True, "send_email": True}
```

注意 `deny` 与 `interrupt` 的区别：

- `deny`：工具直接返回错误，流程继续。
- `interrupt`：流程暂停，等人批准、编辑或拒绝。

`publish_report` 是"写本地文件"，`send_email` 是"对外发送"，两者都需要审批，但实现上一个是真写盘、一个是 dry-run 落盘审计。

节选见 [snippets/permissions-and-hitl.md](../snippets/permissions-and-hitl.md)。

## 6. 组装顺序

`create_deep_agent` 会自动拼装大部分 middleware，调用方只需要声明能力：

```python
create_deep_agent(
    model=chat_model,
    middleware=[TodoListMiddleware(), *rubric, *interpreter],
    tools=[tavily_search, think_tool, publish_report, send_email, *mcp_tools],
    system_prompt=INSTRUCTIONS,
    subagents=[research_agent, report_reviewer, *async_subagents],
    skills=SKILL_SOURCES,
    memory=MEMORY_SOURCES,
    permissions=PERMISSIONS,
    backend=build_backend(store),
    interrupt_on=INTERRUPT_ON,
    store=store,
    checkpointer=checkpointer,
)
```

两个容易忽略的点：

1. `skills=` 和 `memory=` 必须显式传，否则对应的 middleware 根本不会加载；不是"文件在那儿就会自动生效"。
2. `subagents` 里混入带 `graph_id` 的字典会被识别为异步子 Agent，带 `runnable` 的会被识别为已编译子图。

## 7. 提示词分层

把能力说明从主工作流里拆出来，单独一段：

```text
# Workspace and Capabilities
## Durable memory
## Skills and policies
## Approval gates
## Sub-agents
```

好处是"什么时候读、什么时候写、什么时候停"各自成块，不会和"研究流程"互相干扰。实践下来，模型对分段规则的遵守率明显高于混在一起的长段落。

## 8. 测试矩阵

| 测试文件 | 验证内容 | 需要密钥 |
| --- | --- | --- |
| `test_planning.py` | Todo 状态在流中从 in_progress 到 completed | 否 |
| `test_capabilities.py` | 后端路由、记忆幂等、技能注入、权限拒绝、审批中断 | 否 |
| `test_model_config.py` | provider / 并行工具调用参数正确落到模型配置 | 否 |
| `test_api_planning.py` | 起真实 API 服务，验证前端消费的 `values.todos` | 否 |

脚本化模型的关键设计：只替换 `_generate`，`bind_tools` 仍记录真实工具列表。

```python
class ScriptedModel(BaseChatModel):
    replies: list[AIMessage]
    seen_tools: list[str] = []
    prompts: list[str] = []

    def _generate(self, messages, stop=None, run_manager=None, **kwargs):
        self.prompts.append("\n".join(str(m.content) for m in messages))
        reply = self.replies[self._cursor]
        self._cursor += 1
        return ChatResult(generations=[ChatGeneration(message=reply)])
```

`prompts` 这个字段是意外收获：可以直接断言"技能和记忆确实进了系统提示"，而不需要调真实模型。

节选见 [snippets/scripted-test.md](../snippets/scripted-test.md)。

## 9. 复现步骤

```bash
cd research_deepagent

# 1. 零密钥验证
uv run --frozen pytest tests -q

# 2. 配置环境
cp .env.example .env
# 填写 OPENAI_API_KEY / TAVILY_API_KEY

# 3. 启动（Windows 注意跨盘符用 cd /d）
uvx agentseek doctor
uvx agentseek dev
```

浏览器打开 `http://127.0.0.1:5174`，后端 `http://127.0.0.1:2024`。

## 10. 已知限制

| 限制 | 影响 | 处理 |
| --- | --- | --- |
| 记忆用 `InMemoryStore` | API 进程重启后记忆清空 | 换持久 `BaseStore`，图接线不变 |
| 0.7.8 无 `CodeInterpreterMiddleware` | ch15/16 无法落地 | 环境变量开启时告警，不做硬 import |
| `langchain-mcp-adapters` 未锁 | MCP 工具默认不可用 | `uv add` 后开开关 |
| Docker 沙箱在 Windows 不可用 | ch10 原方案受阻 | 用 `LocalShellBackend` 替代 |
| `FilesystemBackend` 路由是真实文件 | 依赖权限拦截写入 | 权限 + 只读目录约定，生产环境建议只读挂载 |

## 11. 设计取舍记录

| 取舍 | 选择 | 理由 |
| --- | --- | --- |
| 技能放 Store 还是真实文件 | 真实文件 | 改技能不用重打包；写权限交给权限层 |
| 记忆放状态还是 Store | Store | 跨线程复用是刚需 |
| 审批用 interrupt 还是工具内确认 | interrupt | 暂停点由运行时保证，模型无法绕过 |
| 异步子 Agent 默认开还是关 | 关 | 需要远程服务，默认开会让本地启动变脆 |
| 评分默认开还是关 | 关 | 会多一轮模型调用，影响测试与成本 |

---

## 附：目录结构

```text
research_deepagent/
├── src/research_deepagent/
│   ├── agent.py              # 模型配置 + build_graph 组装
│   ├── capabilities.py       # 后端、记忆、权限、可选能力开关
│   ├── prompts.py            # 工作流 / 能力 / 子 Agent 提示词
│   ├── tools.py              # 搜索、反思、发布、发信
│   ├── evidence.py           # 评分取证工具
│   ├── mcp_server.py         # FastMCP stdio 服务
│   ├── policies/             # 只读规范
│   └── skills/               # 只读技能
└── tests/                    # 脚本化测试
```
