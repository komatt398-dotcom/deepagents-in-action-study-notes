# 节选：三路后端路由

一条路径，三种存储：线程态、跨线程记忆、只读真实文件。

```python
from deepagents.backends import (
    CompositeBackend,
    FilesystemBackend,
    StateBackend,
    StoreBackend,
)


def build_backend(store):
    routes = {
        "/memories/": StoreBackend(
            store=store,
            namespace=lambda _runtime: ("research_deepagent", "memories"),
        ),
        "/skills/": FilesystemBackend(root_dir=SKILLS_ROOT, virtual_mode=True),
        "/policies/": FilesystemBackend(root_dir=POLICIES_ROOT, virtual_mode=True),
    }
    return CompositeBackend(default=StateBackend(), routes=routes)
```

## 路由行为

| 虚拟路径 | 实际后端 | 实际 key |
| --- | --- | --- |
| `/final_report.md` | StateBackend | 线程状态 |
| `/memories/preferences.md` | StoreBackend | namespace 内 `/preferences.md` |
| `/skills/report-writing/SKILL.md` | FilesystemBackend | `<skills>/report-writing/SKILL.md` |

## 为什么用 `virtual_mode=True`

把宿主机绝对路径映射到虚拟根 `/`，同时拒绝 `..` 越界。这样技能目录既能复用真实文件（改完即生效），又不会把整个磁盘暴露给文件工具。

## 记忆种子必须幂等

```python
for key, content in MEMORY_FILES.items():
    if store.get(NAMESPACE, key) is None:
        store.put(NAMESPACE, key, create_file_data(content))
```

先判断再写入。否则每次构建图都会把用户学到的偏好覆盖回初始值。
