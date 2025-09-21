
# LangGraph × AI 编程 Guide · v2（适配 Claude 新手）

> **版本**: v2.0  
> **日期**: 2025-09-21  
> **面向读者**: 0-1 新手到中级开发者（使用 Claude/ChatGPT 等大模型做“模块化 + 可恢复”的智能体开发）  
> **依据素材**: 综合你提供的两份附件：
> - 《LangGraph-实战教程：构建自定义-AI.md》（工作流/节点/检查点等概念与范式）
> - 《AI_Programming_Coach_Guide.md》（面向新手的项目拆解、协作与质量保障）
>
> 本 v2 在此基础上做了：① 术语白话解释 + 双语对照；② 最小可跑示例（Python/TS）；③ “需求→拆解→子模块→集成”的 LangGraph 编排范式；④ 记忆/恢复（checkpointer）落地；⑤ Claude 新手 Prompt/文件约定；⑥ 骨架模板与清单。

---

## 目录
1. [LangGraph 是什么（白话 + 术语表）](#langgraph-是什么白话--术语表)
2. [5 分钟上手：最小可跑示例](#5-分钟上手最小可跑示例)
3. [从零到一：把“AI 编程”装进 LangGraph](#从零到一把ai-编程装进-langgraph)
4. [带记忆与可恢复：Checkpointer 实战](#带记忆与可恢复checkpointer-实战)
5. [多智能体与 Agentic RAG（可选进阶）](#多智能体与-agentic-rag可选进阶)
6. [Claude 新手专区：Prompt 与项目约定](#claude-新手专区prompt-与项目约定)
7. [项目骨架与清单（可复制）](#项目骨架与清单可复制)
8. [常见坑与排错](#常见坑与排错)
9. [附录：Mermaid 流程图 & Prompt 模板](#附录mermaid-流程图--prompt-模板)

---

## LangGraph 是什么（白话 + 术语表）

**一句话**：LangGraph = 用“**有状态的图**”来编排你的 LLM/Agent 工作流。每个能力都是一个**节点**（调用 LLM、调用工具、检索、评审/路由…），节点之间的执行顺序是**边**；框架自带**检查点（Checkpoint）**把每一步状态保存为**线程（Thread）**，从而可以**带记忆、可回放、断点续跑、插入人工复核**。

### 术语速记（中英对照）
- **Graph / 图**：把流程画出来（节点/边）。  
- **Node / 节点**：一步工作，如“问 LLM”“调一个工具”“查库”。
- **Edge / 边**：节点之间的控制流。可以是直连或**条件边**（Conditional Edges）。
- **State / 状态**：流程中的共享变量，例如对话消息、临时结果。常用内置：`MessagesState`。
- **Tool / 工具**：可被 LLM 触发的函数（带参数/返回值），如 `get_price()`。
- **ToolNode**：把一组工具挂成一个节点供 LLM 调用。
- **tools_condition**：判断 LLM 是否产出 tool call，从而决定是否走“工具分支”。
- **Checkpointer / 检查点**：把每一步执行后的 **State** 存盘（SQLite/Postgres/Redis/Mongo 等），形成**可恢复线程**。
- **Thread / 线程**：一次“会话/任务”的状态轨迹，用 `thread_id` 关联。

**为什么选 LangGraph？**  
- **显式控制流**：清楚表达“先干啥/何时分支/何时停止”。  
- **内建状态 + 记忆**：插上 Checkpointer 即得“断点续跑 + 对话记忆”。  
- **多智能体编排**：Supervisor 路由到不同专家 Agent，复杂也能稳住。  
- **双栈（Python/TS）+ 工业化落地**：支持流式、可观察、可回放、可部署。

---

## 5 分钟上手：最小可跑示例

> 下方给出 **Python** 与 **TypeScript** 两套最短路径，均包含：LLM→（判断是否需要工具）→工具→再回到 LLM 的“ReAct 循环”。

### A. Python 版本（工具调用 + 条件边 + 流式）

**安装**：
```bash
pip install -U langgraph langchain-openai langchain-core
# 若使用 Claude：
pip install -U langchain-anthropic
# 选装：Postgres/SQLite 等持久化
pip install -U "psycopg[binary,pool]" langgraph-checkpoint-postgres
```

**代码**：
```python
from typing import Annotated
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool

# ① 定义工具（示例）
@tool
def get_price(ticker: str) -> str:
    "查询股票价格（示例，实际可接 Finnhub/TuShare 等 API）"
    return f"Price of {ticker} is 123.45"

# ② 选择模型（OpenAI 或 Claude 都行）
USE_CLAUDE = False
if USE_CLAUDE:
    from langchain_anthropic import ChatAnthropic
    llm = ChatAnthropic(model="claude-3-5-sonnet-20240620")  # 或 haiku
else:
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(model="gpt-4o-mini")

# ③ 绑定工具，让 LLM 具备“叫函数”的能力
model = llm.bind_tools([get_price])
tools_node = ToolNode([get_price])

# ④ LLM 节点：一次对话步
def call_model(state: MessagesState):
    resp = model.invoke(state["messages"])
    return {"messages": [resp]}

# ⑤ 组图：LLM →（若需要）Tools → LLM；否则结束
builder = StateGraph(MessagesState)
builder.add_node("llm", call_model)
builder.add_node("tools", tools_node)

builder.add_edge(START, "llm")
builder.add_conditional_edges("llm", tools_condition, {"tools": "tools", "end": END})
builder.add_edge("tools", "llm")

graph = builder.compile()

# ⑥ 运行（流式处理每一次状态变化）
it = graph.stream({"messages": [("user", "查一下 AAPL 的价格并一句话总结")]}, stream_mode="values")
for event in it:
    # 这里 event["messages"] 会不断追加：模型回答 / 工具调用 / 工具结果
    pass
```

### B. TypeScript 版本（预制 ReAct Agent）
```ts
// npm i @langchain/langgraph @langchain/openai @langchain/core zod
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { ChatOpenAI } from "@langchain/openai";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getPrice = tool(({ ticker }: { ticker: string }) => {
  return `Price of ${ticker} is 123.45`;
}, {
  name: "get_price",
  description: "查询股票价格（示例）",
  schema: z.object({ ticker: z.string() })
});

const llm = new ChatOpenAI({ model: "gpt-4o-mini" });
const agent = createReactAgent({ llm, tools: [getPrice] });

for await (const ev of agent.stream(
  { messages: [{ role: "user", content: "查下 AAPL 并总结" }] },
  { streamMode: "values" }
)) {
  // 处理增量事件
}
```

> **要点**：
> - `ToolNode + tools_condition` 是 Python 里最稳妥的工具调用范式。
> - `MessagesState` 自动累积对话/中间结果，无需你手动存取。
> - 用 `stream()` 能拿到“工具调用/结果/模型回复”的逐步事件，便于 UI 显示与审计。

---

## 从零到一：把“AI 编程”装进 LangGraph

结合《AI_Programming_Coach_Guide.md》的新手路径，这里给出一条**“需求 → 拆解 → 子模块 → 集成”**的工程化落地：

### 1) 需求固化（Why/What/How 模板）
- **Why（动机/目标）**：这个能力解决什么问题？对用户/业务的价值是什么？
- **What（输入/输出/约束）**：输入结构、成功判据、边界条件、依赖。
- **How（实现思路）**：调用哪些工具/数据源？需要几个步骤？错误兜底？

> **建议**：把每个能力都写成 `contracts/<ability>.md`，沉淀接口与质量门槛。

### 2) 任务分解（从模块到节点）
- 一项能力 → 1~N 个 **节点**：如“问 LLM”“检索”“工具调用”“评审”“路由”。
- 用**条件边**明确**分支与循环**：例如“若模型触发工具→走工具节点→回到 LLM”。

### 3) 数据与工具清单
- 工具原则：**纯函数化**（入参/出参清晰，避免隐式状态），具备**可单测**。
- 连接外部 API（如 Finnhub/TuShare/自研服务）时，约定**输入 schema** 与**错误码**。

### 4) 编排到图（Graph）
- 每个“能力”可做成一张子图（Subgraph）；业务顶层再做**编排图**（Orchestrator）。
- 复用 `ToolNode`；把“检索/校验/汇总”抽成共用节点，利于维护。

### 5) 测试与回放
- **单元测试**：对每个 Tool/Node 写最小样例；
- **集成测试**：对关键线程（`thread_id`）做**回放**，检查关键节点的输入/输出快照；
- **人审**：在 Checkpoint 之间插入“需人工确认”的停靠点。

> **图示（Mermaid）**：
```mermaid
flowchart TD
    A[需求/Why-What-How] --> B[任务分解: 节点/边]
    B --> C[实现工具/检索/LLM节点]
    C --> D[组装 Graph: 条件边/循环]
    D --> E[Checkpointer: 记忆/回放]
    E --> F[测试: 单元/集成/人审]
    F --> G[部署: API/UI/可观测]
```

---

## 带记忆与可恢复：Checkpointer 实战

### 方案选型
- **本地演示**：SQLite（轻量，无运维）。
- **生产首选**：Postgres / Redis / Mongo（并发/可靠性更好）。

### 代码示例（Postgres）
```python
from langgraph.checkpoint.postgres import PostgresSaver

db_uri = "postgresql://user:pass@host:5432/db"
with PostgresSaver.from_conn_string(db_uri) as cp:
    graph = builder.compile(checkpointer=cp)

config = {"configurable": {"thread_id": "user-42"}}
for _ in graph.stream({"messages": [("user", "记住我叫小李")]}, config, stream_mode="values"):
    pass
# 之后同一个 thread_id 再跑，状态与中间步骤都会被恢复
```

### 最佳实践
- **thread_id 策略**：`<app>-<tenant>-<user>-<session>`，既能区分用户，也利于排查。
- **数据最小化**：State 中避免塞大对象（图片/超大文本），以“引用 + 按需加载”为主。
- **审计可观测**：保存每个节点的“入参/出参/耗时/错误”；必要时落到 APM。

---

## 多智能体与 Agentic RAG（可选进阶）

- **多智能体**：顶层 **Supervisor** 读取意图→路由给“检索专家/计算专家/写作专家”等子代理；子代理完成后把结果**汇聚/裁决**。
- **Agentic RAG**：把“重写查询→检索→阅读→自检→回问→重检索→汇总”拆成节点，用条件边控制循环次数与停止准则（如“置信度≥阈值”）。
- **人审插拔**：关键节点后放“人工审批”节点，审批通过→继续；拒绝→回退到指定节点重跑。

> 建议对每个子代理也使用 **Checkpointer**，实现“可回放 + 可比较”。

---

## Claude 新手专区：Prompt 与项目约定

### 1) 建议的项目文件约定
- `claude.md`：**项目级规则**（工作流、代码风格、Review 规则、质量门槛、文件约定）。
- `claude.local.md`：**个人本地覆盖**（键位、终端、私钥；加入 `.gitignore`）。
- `contracts/`：每个能力的 Why/What/How 与 I/O 契约。
- `tasks/`：面向 Claude 的“可执行任务卡”（详述上下文、输入、产出、验收标准）。
- `fixtures/`：最小可运行样例数据（方便 Claude 与本地自测）。
- `smoke/`：冒烟用例，验证“主链路是否 OK”。

### 2) 面向 Claude 的 Prompt 模板（摘自并增强自你的附件）
**任务卡模板（`tasks/<name>.md`）**：
```md
# 任务名：<实现 xxx 能力>

## 背景/目标（Why）
- 业务价值：
- 成功判据：

## 输入/输出（What）
- 输入结构：
- 期望输出：
- 约束 & 边界：

## 实现要点（How）
- 关键步骤：
- 使用工具/数据源：
- 错误兜底与重试：

## 交付与验收
- 交付物：代码/测试/文档
- 验收清单：功能√、健壮性√、日志√、可回放√
```

**对话式 Prompt（与 Claude 协作）**：
- “请先阅读 `contracts/<ability>.md` 与 `tasks/<name>.md`，给出 5 行内的实现计划；若信息不全，列出需要我补齐的字段。”  
- “实现时请按 **节点函数** 粒度输出代码：`call_model()` / `tools_node` / `router()` … 并附上最小运行示例。”  
- “每一步完成后，更新 `smoke/<name>.md` 的冒烟用例，并在必要时给出回放步骤（基于 `thread_id`）。”

### 3) Token 与成本控制
- 把长上下文放到**外部存储**：用 ID/引用在 State 里传递。
- 关键节点前做**压缩/结构化**（如提取三要点/关键字段），减少重复 Token。

---

## 项目骨架与清单（可复制）

```
your_app/
├─ claude.md
├─ claude.local.md           # 本地覆盖（加入 .gitignore）
├─ contracts/                # Why/What/How 契约
├─ tasks/                    # Claude 任务卡
├─ fixtures/                 # 最小样例
├─ smoke/                    # 冒烟用例
├─ app/
│  ├─ graph.py               # LangGraph 主编排（Python）
│  ├─ tools.py               # 工具函数集合
│  ├─ memory.py              # Checkpointer/线程管理
│  └─ server.py              # FastAPI/Streamlit 接口（可选）
└─ tests/
   ├─ test_tools.py
   └─ test_graph.py
```

**落地清单**：
- [ ] 明确 1~3 个首发能力（Why/What/How 写进 `contracts/`）  
- [ ] 把能力拆成节点/边，画草图并固化到 `app/graph.py`  
- [ ] `tools.py` 写好纯函数工具，提供 1~2 个 `fixtures/` 样例  
- [ ] 接入 Checkpointer，约定 `thread_id` 命名规则  
- [ ] 写 1 个 `smoke/` 冒烟用例，跑通端到端  
- [ ] 编写 `tests/` 的最小单测，覆盖工具与关键节点  
- [ ] 在 `claude.md` 中写入工作流、代码风格与验收标准

---

## 常见坑与排错

- **工具不触发**：确认 `llm.bind_tools([...])` 是否绑定正确；工具名/参数与模型输出对齐。  
- **死循环**：条件边必须设置**停止条件**（如“没有 tool call 就 END”“达到轮次上限”）。  
- **状态过大**：State 别塞大对象；用“外部存储 + 引用”替代。  
- **持久化失败**：先用 SQLite 本地演示；生产再切 Postgres/Redis。  
- **审计困难**：务必开启节点日志 & 保存入参/出参快照，方便回放和 A/B。

---

## 附录：Mermaid 流程图 & Prompt 模板

### A. ReAct Agent（工具调用）
```mermaid
flowchart LR
    U[User] --> LLM[LLM 节点]
    LLM -- 需要工具? -->|是| T[ToolNode 调用]
    LLM -- 否 --> END((END))
    T --> LLM
    LLM --> END
```

### B. 多智能体（带 Supervisor）
```mermaid
flowchart TD
    U[User/任务] --> S[Supervisor 路由]
    S --> R[检索专家]
    S --> C[计算/代码专家]
    S --> W[写作/汇总专家]
    R --> S
    C --> S
    W --> O[输出/审计/持久化]
```

---

> **你可以直接把本文件放到项目根目录作为 `guide.md`，并按“项目骨架与清单”开干。需要我把 `app/graph.py` 的最小骨架也生成出来吗？我可以按你的现有仓库命名直接补齐。**
