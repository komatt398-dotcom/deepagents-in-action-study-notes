# 节选：权限与人工审批

## 声明式权限

```python
from deepagents.middleware.filesystem import FilesystemPermission

PERMISSIONS = [
    FilesystemPermission(operations=["write"], paths=["/skills/**"], mode="deny"),
    FilesystemPermission(operations=["write"], paths=["/policies/**"], mode="deny"),
    FilesystemPermission(operations=["write"], paths=["/secrets/**"], mode="interrupt"),
]
```

三条规则分别对应三种意图：

| 规则 | 意图 | 用户体验 |
| --- | --- | --- |
| `deny` 写技能目录 | 参考材料不允许被"学坏" | 工具报错，流程继续 |
| `deny` 写规范目录 | 同上 | 工具报错，流程继续 |
| `interrupt` 写密钥目录 | 高风险，需要人确认 | 暂停，等待批准 / 编辑 / 拒绝 |

## 审批工具

```python
INTERRUPT_ON = {
    "publish_report": True,
    "send_email": True,
}
```

`publish_report` 真正写盘；`send_email` 是 dry-run，只落审计文件，不联网。两者都需要审批，区别在副作用。

## 权限必须落在工具层

无效做法：

```text
system prompt: 请不要修改 /skills/ 目录
```

有效做法：

```python
FilesystemPermission(operations=["write"], paths=["/skills/**"], mode="deny")
```

区别在于：前者是模型的自我约束，后者是运行时拦截。测试里让模型故意去写只读目录，断言工具返回：

```text
Error: permission denied for write on /skills/x/SKILL.md
```

## 注意边界

`FilesystemPermission` 拦截的是**文件工具**。直接调用 `backend.write()` 仍会写入。因此生产环境还应配合：

- 只读挂载或独立进程隔离；
- 敏感目录不放进任何可路由后端；
- 审计每一次 `interrupt` 的批准结果。
