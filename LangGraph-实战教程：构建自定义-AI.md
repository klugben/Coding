---
created: 2025-09-21T20:58:16 (UTC +08:00)
tags: [后端,架构中文技术社区,前端开发社区,前端技术交流,前端框架教程,JavaScript 学习资源,CSS 技巧与最佳实践,HTML5 最新动态,前端工程师职业发展,开源前端项目,前端技术趋势]
source: https://juejin.cn/post/7499730881831419904
author: 西树同学
---

# LangGraph 实战教程：构建自定义 AI

> ## Excerpt
> LangGraph 是 LangChain 生态系统的一部分，专门用于构建基于 LLM（大型语言模型）的复杂工作流和 Agent 系统。它采用有向图结构来定义工作流程，使开发者能够创建动态、可控且可扩

---
## 什么是 LangGraph

LangGraph 是 LangChain 生态系统的一部分，专门用于构建基于 LLM（大型语言模型）的复杂工作流和 Agent 系统。它采用有向图结构来定义工作流程，使开发者能够创建动态、可控且可扩展的 AI 应用程序。

简单来说，LangGraph 是一个框架，允许你使用图结构来定义 LLM 应用程序的不同组件如何交互，从而实现复杂的、多步骤的 AI 工作流程。

## 为什么选择 LangGraph

在众多 LLM 编排框架中，LangGraph 具有以下优势：

1.  **结构化工作流**：相比于单一的链式调用，LangGraph 允许创建具有分支、循环和条件逻辑的复杂工作流。
    
2.  **状态管理**：提供强大的状态管理机制，使应用可以维护和更新上下文信息。
    
3.  **可视化与监控**：与 LangSmith 集成，提供强大的可视化和监控功能。
    
4.  **可扩展性**：易于集成自定义组件和第三方服务。
    
5.  **类型安全**：支持 TypeScript 式的类型注解，减少运行时错误。
    

## 环境准备与安装

要开始使用 LangGraph，首先需要安装必要的包：

```bash
pip install langchain langgraph langchain-community
```

如果你想使用可视化和监控功能，还需要安装 LangSmith：

```bash
pip install langsmith
```

并设置必要的环境变量：

```python
import os

# LangSmith 配置
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_langsmith_api_key"  # 替换为你的 API Key
os.environ["LANGCHAIN_PROJECT"] = "your_project_name"  # 项目名称
```

## 基础概念

在深入使用 LangGraph 前，先了解一些基本概念：

### 图（Graph）

在 LangGraph 中，图是整个工作流的容器，它包含了节点和边的集合。图定义了工作流的整体结构和执行路径。

想象一个制作披萨的流程图：整个图就像是从面团准备到出炉销售的完整过程，它定义了所有步骤如何连接和执行。在 LangGraph 中，我们使用 `StateGraph` 类来创建这样的工作流容器。

```python
# 创建一个状态图
from langgraph.graph import StateGraph

pizza_workflow = StateGraph(Dict[str, Any])  # 创建基于字典的状态图
# 接下来我们会向这个图中添加节点和边
```

### 节点（Node）

节点是工作流中的基本处理单元，可以是函数、LLM 调用或任何自定义逻辑。每个节点接收输入状态，执行某些操作，然后返回更新后的状态。

继续我们的披萨制作例子，节点就像是流程中的各个步骤：准备面团、添加酱料、撒上奶酪、烘烤等。每个节点都有特定的输入和输出：

```python
def prepare_dough(state: Dict[str, Any]) -> Dict[str, Any]:
    """准备面团节点"""
    # 输入状态可能包含面粉类型、水量等信息
    print(f"正在准备 {state.get('flour_type', '普通')} 面粉的面团")
    
    # 处理后更新状态
    state["dough_ready"] = True
    state["dough_size"] = state.get("size", "中号")
    
    return state  # 返回更新后的状态

# 在图中添加这个节点
pizza_workflow.add_node("prepare_dough", prepare_dough)
```

节点函数既可以是简单的纯 Python 函数，也可以是调用 LLM 的函数，甚至可以是访问外部 API 或数据库的函数。重要的是：它们接收状态，处理状态，然后返回更新后的状态。

### 边（Edge）

边定义了节点之间的连接关系，表示数据和控制流的传递方向。在 LangGraph 中，可以创建条件边和默认边。

在披萨制作过程中，边就是告诉我们"先做什么，后做什么"的连接线。例如：先准备面团，然后加酱料，接着撒奶酪，最后烘烤。LangGraph 允许我们用简单的方式定义这些连接：

```python
# 标准边：定义固定的执行路径
pizza_workflow.add_edge("prepare_dough", "add_sauce")  # 准备面团后添加酱料
pizza_workflow.add_edge("add_sauce", "add_cheese")     # 添加酱料后撒奶酪

# 条件边：根据状态决定下一步
def check_cheese_preference(state: Dict[str, Any]) -> str:
    """检查奶酪偏好，决定下一步"""
    if state.get("extra_cheese", False):
        return "add_extra_cheese"  # 如果客户要求额外奶酪，走这条路径
    return "regular_baking"        # 否则直接烘烤

# 添加条件边，根据函数返回值决定下一步
pizza_workflow.add_conditional_edges(
    "add_cheese",                 # 从这个节点出发
    check_cheese_preference,      # 使用这个函数判断
    {
        "add_extra_cheese": "extra_cheese_node",  # 返回值与下一个节点的映射
        "regular_baking": "baking_node"
    }
)
```

这种方式实现了工作流中的动态决策，使得流程可以根据状态内容自动选择不同路径。

### 状态（State）

状态是在工作流执行过程中传递的数据结构，它在节点之间流动并被修改。状态可以是简单的字典或复杂的自定义类。

回到披萨制作的例子，状态就像是随着披萨制作流转的一张订单表，上面记录了所有信息：披萨大小、面团类型、酱料多少、奶酪种类等。每个工序(节点)都会查看和更新这张表：

```python
# 初始状态：客户订单信息
initial_state = {
    "order_id": "12345",
    "size": "大号",
    "flour_type": "全麦",
    "sauce": "番茄",
    "extra_cheese": True,
    "toppings": ["蘑菇", "橄榄", "青椒"]
}

# 编译工作流
compiled_workflow = pizza_workflow.compile()

# 执行工作流，传入初始状态
final_state = compiled_workflow.invoke(initial_state)

# 最终状态会包含整个过程中添加的所有信息
print(final_state)
# 可能输出: {'order_id': '12345', 'size': '大号', ..., 'dough_ready': True, 
#           'sauce_added': True, 'cheese_type': '马苏里拉', 'baking_time': '12分钟',
#           'pizza_ready': True, 'quality_check': 'passed'}
```

在 LangGraph 中，您可以使用简单的字典作为状态，但更推荐使用 TypedDict 来定义带类型提示的状态结构，这样可以获得更好的代码提示和类型检查：

```python
from typing import TypedDict, List, Optional

class PizzaState(TypedDict):
    order_id: str
    size: str
    flour_type: str
    sauce: str
    extra_cheese: bool
    toppings: List[str]
    dough_ready: Optional[bool]  # 后续步骤中添加的字段
    # 其他字段...

# 使用类型化状态创建图
pizza_workflow = StateGraph(PizzaState)
```

这种方式让代码更加健壮，并且在IDE中提供更好的代码补全体验。

## 构建你的第一个 LangGraph 流程

让我们从简单的例子开始，逐步掌握 LangGraph 的使用。

### Hello World 示例

这个简单示例创建了一个两节点的图，演示了基本的状态传递：

```python
from typing import Dict, Any
from langgraph.graph import StateGraph

# 1. 定义节点函数
def hello_node(state: Dict[str, Any]) -> Dict[str, Any]:
    """第一个节点：添加 Hello 到状态"""
    state["message"] = "Hello"
    return state

def world_node(state: Dict[str, Any]) -> Dict[str, Any]:
    """第二个节点：添加 World 到状态"""
    state["message"] += " World!"
    return state

# 2. 创建图
builder = StateGraph(Dict[str, Any])

# 3. 添加节点
builder.add_node("hello", hello_node)
builder.add_node("world", world_node)

# 4. 设置边（连接节点）
builder.add_edge("hello", "world")
builder.set_entry_point("hello")

# 5. 编译图
graph = builder.compile()

# 6. 执行图
result = graph.invoke({})
print(result)  # 输出: {'message': 'Hello World!'}
```

### 结构化输出示例

进一步利用 TypedDict 提高类型安全性：

```python
from typing import TypedDict, List
from langgraph.graph import StateGraph

# 定义状态类型
class ProcessState(TypedDict):
    input_text: str
    keywords: List[str]
    summary: str

# 节点函数
def extract_keywords(state: ProcessState) -> ProcessState:
    """从输入文本中提取关键词"""
    # 简单示例，实际中可能使用 LLM 或其他算法
    words = state["input_text"].split()
    state["keywords"] = [w for w in words if len(w) > 5]  # 简单处理：提取长度>5的词
    return state

def generate_summary(state: ProcessState) -> ProcessState:
    """生成摘要"""
    # 简单示例
    state["summary"] = f"这是一篇关于 {', '.join(state['keywords'])} 的文章"
    return state

# 创建图
builder = StateGraph(ProcessState)
builder.add_node("extract_keywords", extract_keywords)
builder.add_node("generate_summary", generate_summary)
builder.add_edge("extract_keywords", "generate_summary")
builder.set_entry_point("extract_keywords")

# 编译并执行
graph = builder.compile()
result = graph.invoke({
    "input_text": "人工智能正在改变我们的生活和工作方式"
})
print(result)
```

## 实战案例：构建教育内容生成系统

现在，让我们构建一个更复杂的实际应用：自动化教育内容生成系统。该系统将：

1.  生成课程章节结构
2.  为每个章节创建学习对象
3.  使用 Ollama 生成详细内容
4.  保存为 Markdown 文件

### 系统设计

系统的工作流程包含以下步骤：

1.  **章节生成**：创建课程的章节列表，包括章节名称、简介和学习目标
2.  **学习对象生成**：为每个章节创建学习对象（概念理解、实践应用、知识评估）
3.  **内容增强**：使用 Ollama 生成详细的教育内容
4.  **Markdown 写入**：将生成的内容保存为 Markdown 文件

工作流图如下：

```css
[章节生成] → [学习对象生成] → [内容增强] → [Markdown写入]
```

### 完整代码与解析

以下是实现这个系统的完整代码（精简版）：

```python
import os
import time
import requests
from typing import Dict, List, Any, TypedDict
from langgraph.graph import StateGraph

# Ollama 配置
OLLAMA_API_BASE = "http://localhost:11434/api"
DEFAULT_MODEL = "llama3"

# LangSmith 配置
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_langsmith_api_key"
os.environ["LANGCHAIN_PROJECT"] = "课程创建-LangGraph"

# Ollama API 调用函数
def generate_with_ollama(prompt, model=DEFAULT_MODEL, temperature=0.7, max_tokens=2000):
    """使用 Ollama API 生成内容"""
    try:
        response = requests.post(
            f"{OLLAMA_API_BASE}/generate",
            json={
                "model": model,
                "prompt": prompt,
                "temperature": temperature,
                "stream": False
            }
        )
        
        if response.status_code == 200:
            result = response.json()
            return result.get("response", "")
        else:
            print(f"Ollama API 返回错误状态码: {response.status_code}")
            return f"API调用失败，状态码: {response.status_code}"
    except Exception as e:
        print(f"调用 Ollama API 出错: {e}")
        return f"内容生成失败，请检查 Ollama 服务。错误: {e}"

# 课程结构定义
course_structure = {
    "课程名称": "Python数据分析与可视化",
    "章节": [
        {
            "章节名称": "Python数据分析基础",
            "简介": "介绍Python在数据分析中的基本应用。",
            "学习目标": ["理解数据分析的基本概念", "掌握Python数据处理基础语法"]
        },
        {
            "章节名称": "数据可视化入门",
            "简介": "学习常用数据可视化方法与工具。",
            "学习目标": ["了解数据可视化的意义", "掌握常见可视化库的使用"]
        }
    ]
}

# 提示词模板
prompt_templates = {
    "概念理解": """
    你是一位教育专家，正在为《{course_name}》课程的《{chapter_name}》章节创建概念理解学习材料。
    
    章节介绍: {chapter_intro}
    学习目标: {chapter_goals}
    
    请创建一份全面的概念理解材料，包含核心概念定义、背景知识、相关概念联系等。
    输出格式使用 Markdown，确保标题层级正确，内容应该全部使用中文。
    """,
    
    # 其他模板省略...
}

# 1. 章节生成节点
def chapter_generator(state: Dict[str, Any]) -> Dict[str, Any]:
    """生成课程章节"""
    # 简化版：直接使用预定义的章节
    chapters = course_structure["章节"]
    return {"章节列表": chapters}

# 2. 学习对象生成节点
def learning_object_generator(state: Dict[str, Any]) -> Dict[str, Any]:
    """为每个章节生成学习对象"""
    chapters = state["章节列表"]
    learning_objects = []
    
    for chapter in chapters:
        chapter_name = chapter["章节名称"]
        
        # 创建基础学习对象
        chapter_objects = {
            "章节名称": chapter_name,
            "学习对象": [
                {"类型": "概念理解", "内容": f"本节介绍{chapter_name}的核心概念，帮助学习者建立基础认知。"},
                {"类型": "实践应用", "内容": f"通过实际案例演示{chapter_name}的应用方法，提升实战能力。"},
                {"类型": "知识评估", "内容": f"包含与{chapter_name}相关的自测题，帮助学习者检验学习效果。"}
            ]
        }
        
        learning_objects.append(chapter_objects)
    
    return {"章节学习对象": learning_objects}

# 3. 内容增强节点
def content_enhancer(state: Dict[str, Any]) -> Dict[str, Any]:
    """使用 Ollama 生成详细内容"""
    print("开始内容增强...")
    
    chapter_list = state["章节列表"]
    learning_objects_list = state.get("章节学习对象", [])
    
    # 创建映射以便后续查找
    learning_objects_map = {}
    for lo in learning_objects_list:
        chapter_name = lo.get("章节名称", "")
        if chapter_name:
            learning_objects_map[chapter_name] = lo.get("学习对象", [])
    
    # 存储增强后的章节内容
    enhanced_chapters = []
    
    # 处理每个章节
    for chapter in chapter_list:
        try:
            # 提取章节信息
            chapter_name = chapter["章节名称"]
            chapter_intro = chapter.get("简介", "无介绍")
            
            # 处理学习目标
            chapter_goals = chapter.get("学习目标", [])
            if isinstance(chapter_goals, list):
                chapter_goals = "\n".join([f"- {goal}" for goal in chapter_goals])
            
            # 创建增强章节对象
            enhanced_chapter = {
                "章节名称": chapter_name,
                "章节介绍": chapter_intro,
                "学习目标": chapter_goals,
                "学习对象": []
            }
            
            # 获取学习对象类型
            learning_objects = []
            if chapter_name in learning_objects_map:
                chapter_objects = learning_objects_map[chapter_name]
                learning_objects = [obj["类型"] for obj in chapter_objects]
            
            # 如果没有定义学习对象，使用默认列表
            if not learning_objects:
                learning_objects = ["概念理解", "实践应用", "知识评估"]
            
            # 为每个学习对象生成内容
            for obj_type in learning_objects:
                # 创建提示词
                prompt = prompt_templates.get(obj_type, "").format(
                    course_name=course_structure["课程名称"],
                    chapter_name=chapter_name,
                    chapter_intro=chapter_intro,
                    chapter_goals=chapter_goals
                )
                
                # 使用 Ollama 生成内容
                enhanced_content = generate_with_ollama(prompt)
                
                # 添加到章节对象
                enhanced_chapter["学习对象"].append({
                    "类型": obj_type,
                    "内容": enhanced_content
                })
            
            # 添加到增强章节列表
            enhanced_chapters.append(enhanced_chapter)
            
        except Exception as e:
            print(f"处理章节时发生错误: {e}")
    
    # 更新状态
    updated_state = state.copy()
    updated_state["章节增强内容"] = enhanced_chapters
    
    return updated_state

# 4. Markdown 写入节点
def markdown_writer(state: Dict[str, Any]) -> Dict[str, Any]:
    """保存章节与学习对象到本地 Markdown 文件"""
    course_name = course_structure["课程名称"]
    enhanced_chapters = state.get("章节增强内容", [])
    
    # 创建输出目录
    output_dir = f"course_langgraph"
    os.makedirs(output_dir, exist_ok=True)
    
    # 创建索引文件
    index_path = os.path.join(output_dir, "课程索引.md")
    with open(index_path, "w", encoding="utf-8") as f:
        f.write(f"# {course_name}\n\n")
        f.write("## 课程章节\n\n")
        
        for chapter in enhanced_chapters:
            chapter_name = chapter["章节名称"]
            chapter_intro = chapter["章节介绍"]
            chapter_goals = chapter["学习目标"]
            
            # 写入章节信息
            f.write(f"### {chapter_name}\n\n")
            f.write(f"{chapter_intro}\n\n")
            f.write("**学习目标**:\n")
            if isinstance(chapter_goals, list):
                for goal in chapter_goals:
                    f.write(f"- {goal}\n")
            else:
                f.write(f"{chapter_goals}\n")
            
            # 创建章节目录
            chapter_dir = os.path.join(output_dir, chapter_name)
            os.makedirs(chapter_dir, exist_ok=True)
            
            # 添加学习对象链接
            f.write("\n**学习内容**:\n")
            
            # 写入每个学习对象
            for obj in chapter["学习对象"]:
                obj_type = obj["类型"]
                obj_content = obj["内容"]
                
                # 写入文件链接
                obj_filename = f"{obj_type}.md"
                obj_path = os.path.join(chapter_dir, obj_filename) 
                f.write(f"- [{obj_type}](./{chapter_name}/{obj_filename})\n")
                
                # 创建学习对象文件
                with open(obj_path, "w", encoding="utf-8") as obj_file:
                    obj_file.write(f"# {chapter_name} - {obj_type}\n\n")
                    obj_file.write(obj_content)
            
            f.write("\n")
    
    return state

# 创建工作流图
builder = StateGraph(Dict[str, Any])

# 添加节点
builder.add_node("章节生成", chapter_generator)
builder.add_node("学习对象生成", learning_object_generator)
builder.add_node("内容增强", content_enhancer)
builder.add_node("Markdown写入", markdown_writer)

# 设置起点
builder.set_entry_point("章节生成")

# 添加边（连接节点）
builder.add_edge("章节生成", "学习对象生成")
builder.add_edge("学习对象生成", "内容增强")
builder.add_edge("内容增强", "Markdown写入")

# 编译图
workflow = builder.compile(debug=True)

# 执行工作流
if __name__ == "__main__":
    # 初始状态为空
    state = {}
    
    # 添加运行标识
    run_id = f"run_{int(time.time())}"
    
    print(f"开始执行课程生成流程，运行ID: {run_id}")
    print(f"可在 LangSmith 查看详细执行记录: https://smith.langchain.com/projects/课程创建-LangGraph")
    
    # 执行工作流
    result = workflow.invoke(state)
    
    print(f"执行完成! 课程内容已生成。")
```

代码解析：

1.  **初始化配置**：设置 Ollama API 和 LangSmith 追踪。
    
2.  **定义工作流节点**：
    
    -   `chapter_generator`：生成基础章节结构
    -   `learning_object_generator`：为每个章节创建学习对象
    -   `content_enhancer`：调用 Ollama API 生成详细内容
    -   `markdown_writer`：将内容写入 Markdown 文件
3.  **构建工作流图**：
    
    -   创建 `StateGraph` 实例
    -   添加节点和边
    -   设置入口点
    -   编译图
4.  **执行工作流**：使用空状态调用工作流，并观察结果
    

## 进阶技巧

### 条件分支与循环

LangGraph 支持条件分支和循环，使工作流更灵活：

```python
from typing import Dict, Any, Literal
from langgraph.graph import StateGraph

# 输出可能的值
OutputValues = Literal["需要更多信息", "完成"]

# 定义节点
def process_data(state: Dict[str, Any]) -> Dict[str, Any]:
    # 处理数据...
    return state

def check_completeness(state: Dict[str, Any]) -> OutputValues:
    # 检查数据是否完整
    if len(state.get("data", [])) < 5:
        return "需要更多信息"
    return "完成"

def gather_more_info(state: Dict[str, Any]) -> Dict[str, Any]:
    # 收集更多信息...
    state["data"] = state.get("data", []) + ["新信息"]
    return state

def finalize(state: Dict[str, Any]) -> Dict[str, Any]:
    # 完成处理...
    state["status"] = "完成"
    return state

# 创建工作流
builder = StateGraph(Dict[str, Any])

# 添加节点
builder.add_node("process", process_data)
builder.add_node("check", check_completeness)
builder.add_node("gather", gather_more_info)
builder.add_node("finalize", finalize)

# 设置条件边
builder.set_entry_point("process")
builder.add_edge("process", "check")
builder.add_conditional_edges(
    "check",
    {
        "需要更多信息": "gather",
        "完成": "finalize"
    }
)
# 创建循环
builder.add_edge("gather", "process")

# 编译并执行
graph = builder.compile()
result = graph.invoke({"data": ["初始数据"]})
print(result)
```

### 流程可视化

使用 LangSmith 可视化工作流：

```python
# 设置 LangSmith
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_api_key"
os.environ["LANGCHAIN_PROJECT"] = "langgraph-demo"

# 在编译图时启用调试
workflow = builder.compile(debug=True)
```

### 使用 LangSmith 追踪

LangSmith 提供了强大的追踪和监控功能：

```python
import time

# 执行工作流时添加记录
run_id = f"run_{int(time.time())}"
print(f"开始执行，运行ID: {run_id}")

# 执行工作流
result = workflow.invoke(state)
```

访问 [smith.langchain.com/](https://link.juejin.cn/?target=https%3A%2F%2Fsmith.langchain.com%2F "https://smith.langchain.com/") 查看执行记录。

## 性能优化与最佳实践

1.  **并行执行**：当节点之间没有依赖关系时，考虑并行执行。
    
2.  **缓存结果**：对于计算密集型或网络请求节点，缓存结果可提高性能。
    
3.  **错误处理**：每个节点都应有适当的错误处理机制。
    
4.  **状态隔离**：不同节点应操作状态的不同部分，避免冲突。
    
5.  **使用类型注解**：利用 TypedDict 增强类型安全性。
    
6.  **模块化设计**：将复杂逻辑分解为多个小节点，便于维护和复用。
    

## 常见问题解答

**Q: LangGraph 和普通的 LangChain 链有什么区别？**

A: LangGraph 提供了更灵活的流程控制，支持条件分支、循环和复杂状态管理，而普通链只能线性执行。

**Q: 如何调试 LangGraph 工作流？**

A: 使用 `debug=True` 编译图，结合 LangSmith 进行可视化调试。也可以在每个节点中添加打印语句。

**Q: LangGraph 是否支持异步执行？**

A: 是的，LangGraph 支持异步执行。只需定义异步节点函数和使用 `ainvoke` 而不是 `invoke`。

```python
async def async_node(state):
    # 异步操作
    return state

# 异步执行
result = await graph.ainvoke(state)
```

**Q: 如何在节点之间共享大量数据？**

A: 考虑使用外部存储（如数据库）存储大型数据，在状态中只保留引用或ID。

## 总结与下一步

关注：AGI01，祝你在 AI 工作流构建之旅中取得成功！

LangGraph 是构建复杂 AI 工作流的强大工具，它提供了灵活的流程控制、强大的状态管理和良好的集成能力。本教程介绍了基础概念、简单示例和一个实际应用案例，希望能帮助你开始使用 LangGraph 构建自己的 AI 应用。

接下来，你可以：

1.  尝试构建自己的 LangGraph 工作流
2.  探索更复杂的分支和循环逻辑
3.  集成其他 AI 模型和外部服务
4.  研究 LangGraph 的高级功能和优化策略

## 参考资源

-   [LangGraph 官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fpython.langchain.com%2Fdocs%2Flanggraph "https://python.langchain.com/docs/langgraph")
-   [LangSmith 文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.smith.langchain.com%2F "https://docs.smith.langchain.com/")
-   [LangChain 社区](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Flangchain-ai%2Flangchain "https://github.com/langchain-ai/langchain")
