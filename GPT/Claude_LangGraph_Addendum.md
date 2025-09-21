
# LangGraph 加强版方法论补遗（可与现有工作流合并）
*版本：2025-09-21 12:35 UTC*

## 1) 为什么引入 LangGraph
- **图式思维**：用节点/边表达流程，天然适合多步骤、可回路的 Agent 工作流；可视化强，调试直观。
- **有状态**：统一的 `State` 在节点间流动；支持持久化与“时光回溯/中断继续”。
- **可控循环与人审**：条件边、interrupt、human-in-the-loop，避免“失控的自我调用”。

## 2) 最小落地 8 步（替换/精简你现有步骤）
1. 在 `contracts/state.py` 定义 `TypedDict`/Pydantic `State`（等同于“契约”最小集）。  
2. 列出节点清单：`load_cfg`、`fetch_market`、`enrich`、`decide`、`report`…（每个节点=一个小模块）。  
3. 节点输入输出一律以 `State` 片段读/写，避免自定义形参爆炸。  
4. 建 `StateGraph` 并加边：顺序边+条件边（失败/重试/降级）。  
5. `graph.compile()`；用 `fixtures/e2e/` 的黄金样例跑 `graph.invoke()` 做端到端。  
6. 接入 `checkpointer`（内存/SQLite）与 `interrupt`，支持人审与恢复。  
7. （可选）接 LangSmith/MLflow 追踪。  
8. 把现有 `smoke/run_min.sh` 改为执行 `graph.invoke` 的脚本。

## 3) 目录建议（与原骨架兼容）
```
docs/arch.md
contracts/state.py              # 统一 State 契约（最关键）
src/graph/nodes/*.py            # 节点
src/graph/build.py              # 组图与编译
tests/{unit,e2e}/
fixtures/{module,e2e}/
smoke/run_min.sh
```

## 4) Claude 提示词（单节点开发）
```md
请按 contracts/state.py 定义的 State，完成 src/graph/nodes/<node>.py。
【上下文】<<<contracts/state.py:1-200>>> 以及节点骨架片段。
【输出】仅 udiff 补丁 + 新文件全文 + pytest 命令。
【约束】只通过读/写 State 键值修改状态，添加必要的错误类型与日志。
```

## 5) Claude 提示词（组图与集成）
```md
目标：在 src/graph/build.py 中创建 StateGraph 并编译。
【上下文】<<<contracts/state.py:1-200>>> <<<src/graph/nodes/*>>> 以及 fixtures/e2e/*。
【输出】仅给 build.py 的补丁；生成/修复 smoke/run_min.sh 以调用 graph.invoke。
【完成定义】graph.invoke 对黄金样例通过比对。
```

## 6) Token 管理（LangGraph 场景）
- **状态即上下文索引**：只把 State schema 与需要的节点片段喂给 Claude。  
- **状态摘要**：把长文本压缩进 `state.summaries`，节点仅读摘要。  
- **子图（subgraph）**：把复杂子流程收敛为一个节点调用，减少上下文爆炸。

## 7) 快速代码骨架（节选）
```python
# contracts/state.py
from typing import TypedDict, List, Optional
class State(TypedDict, total=False):
    symbols: List[str]
    market: dict
    analysis: dict
    report_md: str
    errors: List[str]
```

```python
# src/graph/nodes/fetch_market.py
from typing import Dict
from contracts.state import State
def fetch_market(state: State) -> Dict:
    # TODO: 调用 Tushare/Finnhub，写入 state['market']
    return { "market": {"AAPL": {"close": 123.4}} }
```

```python
# src/graph/build.py
from langgraph.graph import StateGraph, END
from contracts.state import State
from src.graph.nodes.fetch_market import fetch_market

def build_graph():
    g = StateGraph(State)
    g.add_node("fetch_market", fetch_market)
    g.set_entry_point("fetch_market")
    g.add_edge("fetch_market", END)
    return g.compile()
```

> 与原方法论的结合点：将“契约文档”**压缩**为 `State` 类型及少量节点注释；“任务卡”聚焦到**单节点**；冒烟脚本直接调用 `graph.invoke()`。
