# Deep Agents 学习笔记

把一门 16 章的 Deep Agents 课程，从"跑通 notebook"推进到"组装成一个可部署、可测试、有安全边界的 Agent"的完整记录。

本仓库只包含**学习笔记、工程实践记录与排错手册**，不包含课程原始讲义、模型密钥或课程作业源码。

---

## 这个仓库解决什么问题

大多数 Agent 教程停在"能跑一个 demo"。真正把能力拼起来时会遇到四个断层：

1. **上下文断层**：文件、记忆、技能分别存在哪，什么时候该读、什么时候该写。
2. **控制流断层**：主 Agent、同步子 Agent、异步子 Agent 怎么分工，上下文隔离后材料怎么传。
3. **安全断层**：哪些动作必须停下来问人，哪些路径根本不许写。
4. **质量断层**：怎么证明它做对了，而不是"看起来像对了"。

前四份笔记分别对应这四个断层，第五份是对课程本身的复盘。

## 目录

| 文档 | 内容 |
| --- | --- |
| [notes/01-learnings.md](notes/01-learnings.md) | 学习心得：八条最重要的认知，以及被证伪的直觉 |
| [notes/02-implementation.md](notes/02-implementation.md) | 工程实践：能力分层、文件结构、测试矩阵、复现步骤 |
| [notes/03-troubleshooting.md](notes/03-troubleshooting.md) | 排错手册：症状 → 定位 → 修复，含可复现命令 |
| [notes/04-case-review-turbopuffer.md](notes/04-case-review-turbopuffer.md) | 案例评析：Cursor 用 Turbopuffer 降本 20 倍，哪些可信、哪些是营销 |
| [notes/05-course-feedback.md](notes/05-course-feedback.md) | 课程反馈：5 个优点、7 条改进建议、3 条给后续学员的建议 |
| [snippets/](snippets/) | 关键代码节选：后端路由、权限与审批、脚本化测试 |

## 环境基线

笔记中的所有结论都基于这组锁定版本，换版本前请先核对 API：

```text
OS              Windows 10.0.26200
Python          3.12
deepagents      0.7.8
agentseek-api   0.2.3
langchain       >=1.0
```

课程部分章节基于 deepagents 0.7.10 / 0.7.15，因此 `CodeInterpreterMiddleware` 等能力在 0.7.8 中不存在。笔记里所有"降级为可选开关"的处理都来自这个差异，不是设计偏好。

## 三条核心结论

1. **上下文工程不是写 prompt**，而是设计"文件系统 + 记忆 + 技能"的存储与加载策略。
2. **安全边界必须落在工具层**。prompt 里的"不要做 X"不是机制，`deny` 和 `interrupt` 才是。
3. **没有脚本化测试，改 Agent 就是盲改**。脚本化模型、真实图、真实文件系统，才能在无密钥的情况下验证机制。

## 一个最小验证

在配套工程里执行（不需要任何 API key）：

```bash
uv run --frozen pytest tests -q
```

预期 `11 passed`。这组测试覆盖：Todo 状态、API 流式事件、模型配置、后端路由、记忆幂等、技能注入、权限拒绝、审批中断。

## 版权与免责

- 课程讲义、notebook 与作业内容的版权归原作者，本仓库不转载。
- 文中引用的厂商数据（如 Turbopuffer 的降本比例）均标注来源与证据等级，不代表独立验证结论。
- 本仓库未附开源许可证；如需公开采用，请自行选择并添加 `LICENSE`。

## 上传到 GitHub

```bash
cd /d F:\deepagents-in-action\study-notes
git init
git add .
git commit -m "docs: add deep agents study notes"
git branch -M main
git remote add origin https://github.com/<your-name>/<repo-name>.git
git push -u origin main
```

`.gitignore` 已排除虚拟环境、密钥、构建产物与课程原始材料。
