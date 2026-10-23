# 排错手册：症状 → 定位 → 修复

> 只收录本次真实遇到、且能复现的问题。每条都给出判断依据，避免"重启试试"。

---

## 快速判断：是 Agent 的问题，还是平台的问题

| 现象 | 大概率来源 |
| --- | --- |
| 聊天无响应、没有 todo、没有工具卡片 | Agent / 模型配置 |
| 能聊天，但某个端点持续 4xx | 平台 / 客户端 |
| 启动即退出，日志里有 traceback | 你的 import 或依赖 |
| 启动成功，但线程列表刷不出 | 平台接口兼容性 |
| 单个能力不生效，其余正常 | middleware 参数没传 |

一个原则：**先看这个请求有没有走到你的图**。没走到，就不是 Agent 代码的问题。

---

## 1. Windows 跨盘符 `cd` 不生效

**症状**

```text
C:\Users\super>cd F:\deepagents-in-action\research_deepagent
C:\Users\super>uv run --frozen pytest tests -q
no tests ran in 0.01s
ERROR: file or directory not found: tests
```

注意提示符仍是 `C:\Users\super>`。

**原因**：`cmd` 的 `cd` 只切目录，不切盘符。

**修复**

```cmd
cd /d F:\deepagents-in-action\research_deepagent
```

或

```cmd
F:
cd \deepagents-in-action\research_deepagent
```

**判断依据**：提示符路径没变，就说明目录没切过去。

---

## 2. `/threads/search` 持续返回 422

**症状**

启动后日志不断刷：

```text
POST /threads/search HTTP/1.1" 422 Unprocessable Entity
GET  /deploy        HTTP/1.1" 404 Not Found
```

但聊天、流式输出正常。

**定位**

用 JSON 文件发请求，避免命令行引号干扰：

```bash
printf '%s' '{"metadata":{}}' > a.json
printf '%s' '{"values":{}}'   > b.json

curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  http://127.0.0.1:2024/threads/search \
  -H "Content-Type: application/json" --data-binary @a.json

curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  http://127.0.0.1:2024/threads/search \
  -H "Content-Type: application/json" --data-binary @b.json
```

实测结果（agentseek-api 0.2.3）：

| 请求体 | 状态 |
| --- | --- |
| `{"metadata":{}}` | 200 |
| `{"metadata":{},"values":null}` | 200 |
| `{"values":{}}` | 422 |
| `{"metadata":null,"values":{}}` | 422 |

错误详情：

```json
{"detail":"'values' filter is not supported yet"}
```

**原因**：LangGraph Studio 的线程列表默认带一个空的 `values` 过滤器（语义是"不过滤"），而当前 agentseek-api 版本尚未实现 `values` 过滤，任何 `values` 键都会 422。

**修复**

- 不影响功能：关掉打开着 Studio 的标签页即可停止轮询。
- 想根治：升级 agentseek-api（本次锁定 0.2.3，不建议在课程中动锁）。
- 不要改 Agent 代码，它没有参与这个请求。

**附带噪音**

- `GET /deploy 404`：Studio 的能力探测，正常。
- `[sample_graphs] skipped ... No module named 'graphs'`：内置示例图加载失败，与本项目无关。

---

## 3. 线程一直显示 busy

**症状**：界面转圈，或线程列表里的旧线程状态永远是 `busy`。

**定位**

```bash
curl -s -X POST http://127.0.0.1:2024/threads/search \
  -H "Content-Type: application/json" \
  --data-binary @empty.json
```

看返回里的 `status` 和 `updated_at`。如果 `updated_at` 停在几小时前，就是僵尸运行。

**修复**：新建 thread，不要续跑旧线程。旧线程的 checkpoint 可能停在半完成的工具调用上。

---

## 4. `CodeInterpreterMiddleware` 不存在

**症状**

```text
ImportError: cannot import name 'CodeInterpreterMiddleware'
```

或开启开关后只看到告警。

**定位**

```bash
uv run python -c "import importlib.util as u; print(u.find_spec('deepagents.middleware.code_interpreter'))"
```

返回 `None` 说明当前版本没有这个模块。

**原因**：讲义基于 deepagents 0.7.10 / 0.7.15，工程锁定 0.7.8。

**修复**：保持"可选开关 + 告警"，不要硬 import。需要落地时升级并加 quickjs extra：

```bash
uv add "deepagents[quickjs]"
```

---

## 5. MCP 工具加载不出来

**症状**：开启 `DEEPAGENTS_ENABLE_MCP=true` 后，模型看不到 MCP 工具。

**定位**

```bash
uv run python -c "import importlib.util as u; print(u.find_spec('langchain_mcp_adapters'))"
```

**原因**：`langchain-mcp-adapters` 不在锁定依赖里，属于可选安装。

**修复**

```bash
uv add langchain-mcp-adapters
```

然后单独验证服务能跑：

```bash
uv run python -m research_deepagent.mcp_server
```

注意：MCP 工具名默认会带前缀（如 `math_add`），提示词里不要写死未加前缀的名字。

---

## 6. PowerShell 里 curl 的 JSON 被吞引号

**症状**：明明是合法 JSON，服务器却一直 422。

**原因**：PowerShell 5.1 向原生程序传参时会重写引号，`{"a":1}` 可能变成 `{a:1}`。

**修复**：把请求体写进文件，用 `--data-binary @file.json`，或改用 `Invoke-RestMethod -Body`。

**教训**：排查 422 时，先确认"发出去的 body 到底是什么"。否则会把客户端的引号问题误判成服务端 bug。

---

## 7. 改了能力但"没生效"

**症状**：加了 skills / memory，但系统提示里看不到。

**定位**：用脚本化模型记录提示词：

```python
assert "report-writing" in model.prompts[0]
assert "Research Agent Memory" in model.prompts[0]
```

**常见原因**

| 原因 | 检查点 |
| --- | --- |
| 没传 `skills=` / `memory=` | `create_deep_agent` 只在显式传参时加载对应 middleware |
| 路径不匹配 | 后端路由前缀与 sources 前缀必须一致 |
| 文件不存在 | 种子逻辑是否执行、路径大小写是否正确 |
| 权限拦住了读取 | `deny` 规则是否误伤 `read` |

---

## 8. 权限看起来生效，实际没生效

**症状**：模型说"我不能修改这个文件"，但文件真的被改了。

**原因**：把约束写在了 prompt 里，而不是工具层。

**验证方法**：让脚本化模型直接调用 `write_file`，断言工具返回：

```text
Error: permission denied for write on /skills/x/SKILL.md
```

**修复**：加 `FilesystemPermission(..., mode="deny")`。注意 `deny` 只拦截工具层，直接调用后端 API 仍可写，所以真实部署还应配合只读挂载或独立进程隔离。

---

## 9. 测试很慢

**现象**：11 个测试跑 80–170 秒。

**原因**：每个测试都会构建完整图（加载技能、读取记忆、组装 middleware），API 测试还要起子进程。

**建议**

- 只改一个能力时，跑对应文件而不是全量。
- 不要为了提速把图换成 mock，否则测不到 middleware 顺序问题。
- 冷启动第一次会明显更慢（文件缓存、依赖导入），属正常。

---

## 一页速查

```text
tests 找不到            → cd /d
端点 422                → 先用文件发 JSON，确认 body
values filter 报错      → 平台限制，关 Studio
线程一直 busy           → 新建 thread
ImportError 某 middleware → 查版本，降级为开关
MCP 工具缺失            → uv add langchain-mcp-adapters
技能/记忆没进 prompt     → 检查 sources 与路由前缀
权限"生效"但文件被改     → 约束没落到工具层
```
