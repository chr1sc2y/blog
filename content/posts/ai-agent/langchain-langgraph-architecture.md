---
title: "LangChain 与 LangGraph 架构解析：从链式调用到图驱动的 Agent 编排"
date: 2026-03-03T08:00:00+08:00
draft: false
categories: ["AI Agent"]
---

# LangChain 与 LangGraph 架构解析：从链式调用到图驱动的 Agent 编排

LangChain 是当前 LLM 应用开发领域中使用最广泛的开源框架之一，而 LangGraph 则是 LangChain 团队在 Agent 编排层面的第二次架构尝试——一个受 Google Pregel 和 Apache Beam 启发的、基于有向图的状态化工作流引擎。两者的关系并不是简单的替代，而是分层互补：LangChain 提供模型抽象、工具封装和高层 Agent 接口，LangGraph 则负责底层的执行编排、状态持久化和人在回路控制。理解这两个项目的架构设计，本质上是理解"如何用软件工程的方式构建可靠的 LLM 应用"这个问题在 2024-2026 年间的演进路径。

从宏观上看，LangChain 生态的整体分层结构如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                          │
│         create_agent / Custom Graph / Deep Agents               │
├─────────────────────────────────────────────────────────────────┤
│                      LangChain (High-Level)                     │
│    Agent Abstraction   │  Middleware  │  Structured Output      │
│    (create_agent,      │  (wrap_model │  (with_structured       │
│     tool loop)         │   _call)     │   _output)              │
├────────────────────────┴─────────────┴──────────────────────────┤
│                      LangGraph (Low-Level Orchestration)        │
│    StateGraph  │  Functional API  │  Checkpointer  │  Command   │
│    (nodes,     │  (entrypoint,    │  (persistence, │  (control  │
│     edges,     │   task)          │   threads,     │   flow +   │
│     state)     │                  │   time-travel) │   updates) │
├─────────────────────────────────────────────────────────────────┤
│                      LangChain Core                             │
│    Runnable Protocol  │  Chat Models  │  Messages  │  Tools     │
│    (invoke/stream/    │  (BaseChatModel│  (Human/AI/│  (BaseTool,│
│     batch/transform)  │   + providers)│   Tool/Sys)│   @tool)   │
├─────────────────────────────────────────────────────────────────┤
│                      Integrations                               │
│    OpenAI / Anthropic / Google / ... / MCP / Vector Stores      │
└─────────────────────────────────────────────────────────────────┘
```

自底向上，Integrations 层对接各家模型供应商和外部工具；LangChain Core 定义了 Runnable 协议和核心数据类型；LangGraph 在 Core 之上构建了图执行引擎和持久化基础设施；LangChain 的高层 Agent 抽象则是面向开发者的最终接口。这种分层在项目结构上体现为独立的 Python 包——`langchain-core`、`langgraph`、`langchain` 分别发布和版本管理，通过依赖关系松散耦合。

## 1 LangChain Core：Runnable 协议与组合式抽象

### 1.1 Runnable 作为统一接口

LangChain Core 的设计核心是 `Runnable` 协议。这个协议定义了 LangChain 体系中所有可执行组件的统一接口，包括 `invoke`（同步调用）、`ainvoke`（异步调用）、`batch`（批量处理）、`stream`（流式输出）和 `transform`（流式转换）五个基本操作。无论是聊天模型、Prompt 模板、输出解析器还是自定义函数，只要实现了 `Runnable` 接口，就可以被纳入 LangChain 的组合体系中。

```
                           Runnable (Protocol)
                          invoke / stream / batch
                                   │
               ┌───────────────────┼───────────────────┐
               ▼                   ▼                   ▼
         BaseChatModel      PromptTemplate      OutputParser
         (LLM providers)    (message assembly)  (structured extraction)
               │
    ┌──────────┼──────────────┐
    ▼          ▼              ▼
ChatOpenAI  ChatAnthropic  ChatGoogle
```

这个设计选择的意义在于，它将"可组合性"提升到了框架的第一公民地位。两个 Runnable 可以通过管道操作符 `|` 串联成 `RunnableSequence`，也可以通过 `RunnableParallel` 并行执行。`RunnableSequence` 是 LangChain 中使用频率最高的组合操作符——几乎所有的链式调用（chain）最终都会被编译为 `RunnableSequence` 实例。它的核心语义很直接：前一个 Runnable 的输出作为后一个 Runnable 的输入，依次传递。

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

chain = ChatPromptTemplate.from_template("翻译为英文：{text}") | ChatOpenAI() | StrOutputParser()
result = chain.invoke({"text": "你好世界"})
```

这段代码中，`|` 操作符在底层调用了 `Runnable.__or__` 方法，将三个组件包装为一个 `RunnableSequence`。`RunnableSequence` 继承自 `RunnableSerializable`，后者又继承自 `Runnable` 和 pydantic 的 `BaseModel`——这意味着整条链本身也是一个 Runnable，可以被进一步组合，同时也是可序列化的。这种递归组合的能力使得 LangChain 的表达式语言（LCEL, LangChain Expression Language）能够构建任意复杂的处理管道。

值得注意的是，`RunnableSequence` 的流式处理并非简单地将最后一个组件的输出流式返回。它的 `transform` 方法会将流式输入逐步传递给链中的每一个组件——只要该组件实现了 `transform` 方法。这意味着在理想情况下，用户可以看到 LLM 生成的 token 经过输出解析器后实时地返回，而不需要等待 LLM 完成整个生成过程。但现实中，并非所有组件都能支持真正的流式转换——例如需要完整输入才能工作的 JSON 解析器就会打断流式链条。这是组合式抽象与流式处理之间的一个固有张力。

### 1.2 消息体系与聊天模型

LangChain Core 定义了一套结构化的消息类型体系，包括 `HumanMessage`（用户输入）、`AIMessage`（模型响应）、`SystemMessage`（系统指令）、`ToolMessage`（工具返回）等。这些消息类型继承自 `BaseMessage`，每条消息都包含 `content`（内容）、`role`（角色）和可选的 `additional_kwargs`（额外参数）等字段。`AIMessage` 还包含 `tool_calls` 字段，用于承载模型发出的工具调用请求。

```
BaseMessage
├── HumanMessage      (role: "human")
├── AIMessage         (role: "ai", tool_calls: [...])
├── SystemMessage     (role: "system")
├── ToolMessage       (role: "tool", tool_call_id: "...")
└── RemoveMessage     (用于从历史中删除特定消息)
```

`BaseChatModel` 是所有聊天模型的抽象基类，它定义了 `invoke`、`stream` 等方法的签名，并提供了 `bind_tools` 方法来将工具声明绑定到模型上。各家模型供应商（OpenAI、Anthropic、Google 等）通过继承 `BaseChatModel` 并实现 `_generate` 方法来接入 LangChain 体系。这种适配器模式使得上层代码可以在不修改业务逻辑的情况下切换底层模型——前提是开发者自行保证不同模型在能力上的兼容性（是否支持 function calling、上下文窗口大小、多模态能力等），框架本身并不做能力匹配的校验。

LangChain 还提供了一个便利的 `init_chat_model` 工厂函数，它接受形如 `"openai:gpt-4o"` 或 `"anthropic:claude-sonnet-4-5-20250929"` 的模型标识符字符串，自动解析供应商前缀并实例化对应的模型类。这种注册表式的初始化方式降低了模型切换的代码改动量，在需要动态选择模型的场景中也很实用。

### 1.3 工具抽象

LangChain 的工具系统围绕 `BaseTool` 基类构建，但在实际使用中最常见的方式是通过 `@tool` 装饰器将普通 Python 函数转换为工具。框架会从函数的类型标注和 docstring 中自动提取工具的名称、描述和参数 schema，将其转换为符合 OpenAI function calling 规范的 JSON schema 描述。这意味着开发者只需要写一个带类型标注的函数，就能让 LLM 通过 function calling 来调用它。

```python
from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """搜索数据库并返回匹配结果。"""
    return f"找到 {limit} 条匹配 '{query}' 的结果"
```

框架自动为这个函数生成的 schema 大致如下：

```json
{
  "name": "search_database",
  "description": "搜索数据库并返回匹配结果。",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {"type": "string"},
      "limit": {"type": "integer", "default": 10}
    },
    "required": ["query"]
  }
}
```

这种自动化的 schema 生成极大地降低了工具定义的摩擦。但它也带来一个潜在问题：工具的描述质量直接影响 LLM 选择和调用工具的准确性，而 docstring 往往被开发者视为"文档"而非"Prompt 工程"的一部分。在实际项目中，工具的 docstring 需要被视为 Prompt 的组成部分来精心设计，这一点在框架层面并没有足够的引导。

## 2 LangGraph：图驱动的状态化编排

### 2.1 从链到图的范式转变

LangChain 早期的核心抽象是"链"（Chain）——一种线性的、从输入到输出的处理管道。链式调用在简单场景下直观高效，但当 Agent 系统需要条件分支、循环重试、并行执行和人在回路等复杂控制流时，线性管道就显得力不从心了。LangGraph 的出现正是为了解决这个问题：它将执行流程从链式结构升级为有向图结构，使得任意复杂的控制流都可以被自然地表达。

LangGraph 的底层执行模型受到了 Google Pregel 系统的启发。Pregel 是 Google 在 2010 年发表的一个大规模图处理框架，其核心思想是"以顶点为中心的计算"（vertex-centric computation）：计算以离散的"超步"（super-step）推进，每个超步中所有活跃的顶点并行执行，通过消息传递与其他顶点通信。LangGraph 借鉴了这个模型——图中的节点在收到消息（状态更新）时变为活跃状态并执行其逻辑，执行完成后通过边将更新后的状态传递给下游节点。当所有节点都不再活跃且没有消息在传输中时，图的执行终止。

```
Super-step 0          Super-step 1          Super-step 2
┌──────────┐         ┌──────────┐         ┌──────────┐
│ __start__ │────────►│  node_a  │────────►│  node_b  │────► END
│ (input)   │         │ (active) │         │ (active) │
└──────────┘         └──────────┘         └──────────┘
                     state: {foo: ""}      state: {foo: "a"}
                         ▼                     ▼
                     state: {foo: "a"}     state: {foo: "b"}

                     ┌──────────┐
                     │  node_c  │  (同一个 super-step 内可并行)
                     │ (active) │
                     └──────────┘
```

这个设计使得 LangGraph 天然具备了两个优势：一是并行执行——同一个超步内的多个活跃节点可以并发运行；二是确定性——每个超步的边界提供了一个天然的同步点和状态提交点，使得执行过程可以被精确地追踪和回放。

### 2.2 StateGraph：图的构建与编译

`StateGraph` 是 LangGraph 中最核心的类，用于定义和构建状态化的有向图。它接受一个类型化的状态 schema（通常是 `TypedDict` 或 `dataclass`）作为参数，这个 schema 定义了在整个图的生命周期中共享的状态数据结构。

```python
from typing import Annotated, TypedDict
from operator import add
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    messages: Annotated[list, add]
    current_step: str

builder = StateGraph(State)
builder.add_node("analyze", analyze_node)
builder.add_node("execute", execute_node)
builder.add_edge(START, "analyze")
builder.add_conditional_edges("analyze", route_function)
builder.add_edge("execute", END)

graph = builder.compile(checkpointer=checkpointer)
```

图的构建分为三个阶段：定义状态、添加节点和边、编译。编译阶段（`.compile()`）会对图的结构进行校验（检查是否有孤立节点、是否所有节点都可达等），并将构建器（builder）转换为可执行的 `CompiledStateGraph` 实例。编译时还可以指定运行时配置，如检查点器（checkpointer）、断点（breakpoints）和缓存策略等。

节点（Node）在 LangGraph 中就是普通的 Python 函数——它接收当前状态作为输入，执行某些计算或副作用，然后返回一个状态更新。返回的更新不需要包含完整的状态，只需要包含发生变化的字段即可。这个"部分更新"的语义与 React 中 `setState` 的合并行为类似。

边（Edge）分为两种：静态边始终从一个节点指向另一个固定节点；条件边则通过一个路由函数在运行时决定下一步的去向。条件边是实现 Agent 中"决策"逻辑的关键机制——LLM 的输出可以通过路由函数被映射到不同的处理节点上。

### 2.3 Reducer：状态更新的合并策略

LangGraph 的状态管理中最精妙的设计之一是 Reducer 机制。每个状态字段都可以关联一个独立的 reducer 函数，用于定义当多个节点对同一个字段产生更新时，这些更新应该如何被合并。

默认情况下（没有指定 reducer），新值会直接覆盖旧值。但对于消息列表这样的字段，覆盖语义显然是不合适的——我们希望新消息被追加到列表末尾，而不是替换整个列表。通过使用 Python 的 `Annotated` 类型标注来指定 reducer 函数，开发者可以精确地控制每个字段的更新行为：

```python
from typing import Annotated
from operator import add

class State(TypedDict):
    count: int                              # 默认：覆盖
    messages: Annotated[list, add]          # 使用 operator.add 追加
    metadata: Annotated[dict, merge_dicts]  # 自定义合并策略
```

这种按字段粒度定义合并策略的设计，与事件溯源（Event Sourcing）架构中的"投影"（projection）概念有异曲同工之处——状态不是被直接修改的，而是通过一系列更新（事件）经由 reducer 投影出最终的状态快照。这个设计在并行执行的场景下尤为重要：当多个节点在同一个超步中并行执行并分别产出对同一个字段的更新时，reducer 提供了一种声明式的冲突解决机制，避免了手动处理竞态条件的复杂性。

LangGraph 还提供了一个专门为消息列表优化的 `add_messages` reducer。它不仅支持追加新消息，还能通过消息 ID 识别和更新已有消息——这在人在回路场景中修改历史消息时非常有用。`MessagesState` 是一个预构建的状态类，内置了 `messages` 字段和 `add_messages` reducer，几乎所有涉及对话的 Agent 应用都以它为基础。

### 2.4 Command：控制流与状态更新的统一原语

在早期版本的 LangGraph 中，控制流（通过条件边决定下一步）和状态更新（通过节点返回值修改状态）是两个独立的机制。这在某些场景下会导致不便——比如一个节点在处理完请求后，既需要更新状态，又需要根据处理结果动态地路由到不同的下游节点。

`Command` 原语的引入解决了这个问题。它将状态更新和控制流跳转合并为一个原子操作：

```python
from langgraph.types import Command
from typing import Literal

def router_node(state: State) -> Command[Literal["agent", "human_review"]]:
    if state["risk_level"] > 0.8:
        return Command(
            update={"status": "pending_review"},
            goto="human_review"
        )
    return Command(
        update={"status": "approved"},
        goto="agent"
    )
```

`Command` 接受四个参数：`update`（状态更新）、`goto`（跳转目标）、`resume`（中断后恢复的值）和 `graph`（跨子图导航时指定目标图）。其中 `goto` 的行为与条件边类似，但它可以在节点函数内部根据任意逻辑动态决定，而不需要单独定义一个路由函数。返回类型标注中的 `Command[Literal["agent", "human_review"]]` 不是纯粹的类型约束，它同时也被 LangGraph 用于图的可视化渲染——框架据此知道该节点可能的出边方向。

`Command` 的 `graph=Command.PARENT` 参数使得子图中的节点能够直接导航到父图中的节点，这为多 Agent 系统中的"交接"（handoff）模式提供了原生支持。一个客服场景中的"转接专家"操作，在 LangGraph 中可以被建模为子图通过 `Command` 将控制权交还给父图中的另一个子图。

这里有一个值得讨论的设计权衡：`Command` 的引入使得控制流逻辑可以被分散在各个节点中，而不再集中于条件边的路由函数。这提高了单个节点的自治能力，但也可能导致图的整体控制流变得不那么直观——当阅读图的定义时，静态边和条件边一目了然，而 `Command` 中的 `goto` 则需要深入每个节点的实现才能发现。在复杂系统中，这可能增加理解和维护的成本。

## 3 持久化与持久执行

### 3.1 Checkpointer：基于超步的状态快照

LangGraph 的持久化层建立在"检查点"（checkpoint）的概念之上。当图在编译时指定了一个 checkpointer，框架会在每个超步完成时自动将当前的完整状态保存为一个检查点。这些检查点被组织在"线程"（thread）中——每个线程代表一次完整的执行历程，由一个唯一的 `thread_id` 标识。

```
Thread: "conversation-42"
┌─────────────────────────────────────────────────────────┐
│  Checkpoint 0 (step: -1)                                │
│  input: {"messages": [HumanMessage("你好")]}            │
│  next: (__start__,)                                     │
├─────────────────────────────────────────────────────────┤
│  Checkpoint 1 (step: 0)                                 │
│  state: {"messages": [HumanMessage("你好")]}            │
│  next: (agent,)                                         │
├─────────────────────────────────────────────────────────┤
│  Checkpoint 2 (step: 1)                                 │
│  state: {"messages": [..., AIMessage("你好！")]}        │
│  next: ()  ← 图执行完毕                                 │
└─────────────────────────────────────────────────────────┘
```

LangGraph 提供了多种 checkpointer 实现：`InMemorySaver` 用于开发和测试；`PostgresSaver` 和 `SqliteSaver` 用于生产环境的持久化存储。Checkpointer 的接口设计遵循了可插拔的存储后端模式——上层的执行引擎不关心状态被存储在内存、文件系统还是数据库中，只依赖于 checkpointer 的抽象接口。

检查点的存在使得几个强大的能力成为可能。首先是**时间旅行**（time travel）：通过指定一个历史检查点的 `checkpoint_id`，可以将图的状态回退到任意历史时刻，并从该时刻重新执行。框架会识别出哪些步骤已经在此前执行过（replay），哪些需要重新执行（fork）。其次是**状态检查**：在图执行的任意时刻，外部程序可以通过 `graph.get_state(config)` 获取当前的状态快照，或通过 `graph.get_state_history(config)` 获取完整的检查点历史。

### 3.2 持久执行：从断点到恢复

持久执行（durable execution）是 LangGraph 在工程层面最有价值的能力之一。它的核心思想是：工作流在关键节点保存进度，使得它可以在中断后从上次保存的位置恢复，而不需要重新执行已完成的步骤。这在两个场景中至关重要：一是 Agent 调用外部 API 时可能遭遇超时或故障，需要在修复后从断点继续；二是人在回路的交互中，人类审核者可能需要数小时甚至数天才能完成审批，Agent 不能也不应该在此期间一直占用资源。

LangGraph 要求持久执行的工作流满足两个条件：**确定性**和**幂等性**。确定性意味着给定相同的输入和状态，工作流的执行路径应该是一致的；幂等性意味着重复执行同一个操作不会产生额外的副作用。这两个约束并不是 LangGraph 特有的——它们是所有持久化执行引擎（如 Temporal、Durable Functions 等）的共同要求。

为了处理非确定性操作（如随机数生成）和有副作用的操作（如 API 调用），LangGraph 引入了 `task` 的概念。被 `@task` 装饰的函数会在首次执行时将其结果持久化到检查点中，当工作流从中断处恢复并"回放"到该 task 时，框架会直接从检查点中读取之前的结果，而不是重新执行函数体。这种机制与 Temporal 中的 activity 概念非常相似。

```python
from langgraph.func import task

@task
def call_external_api(url: str):
    """这个函数的结果会被持久化，恢复时不会重复执行。"""
    return requests.get(url).json()

def process_node(state: State):
    result = call_external_api(state["api_url"])
    return {"data": result.result()}
```

LangGraph 还支持三种持久化模式，在性能和数据一致性之间提供不同的权衡：`"sync"` 模式在每个步骤完成后同步写入检查点，提供最高的持久性但性能开销最大；`"async"` 模式异步写入检查点，在性能和持久性之间取得平衡；`"exit"` 模式只在图执行结束时写入检查点，性能最优但无法从中间步骤的故障中恢复。这三种模式的设计反映了一个务实的工程判断：不是所有工作流都需要最高级别的持久性保证，开发者应该根据具体场景选择合适的一致性级别。

### 3.3 中断与人在回路

LangGraph 通过 `interrupt` 函数和 `Command(resume=...)` 实现了原生的人在回路（Human-in-the-Loop）支持。当图的执行到达一个包含 `interrupt()` 调用的节点时，执行会被暂停，当前状态被持久化为检查点，然后控制权被交还给调用者。调用者（通常是 UI 层或 API 层）可以检查当前状态、展示给人类审核者，然后在审核完成后通过 `Command(resume=value)` 恢复执行。

```python
from langgraph.types import interrupt, Command

def human_review_node(state: State):
    decision = interrupt({
        "question": "是否批准这笔交易？",
        "transaction": state["transaction"]
    })
    if decision == "approve":
        return {"status": "approved"}
    return {"status": "rejected"}

# 第一次调用——执行到 interrupt 处暂停
result = graph.invoke(input_data, config)
# result 中包含中断信息

# 人类审核完成后恢复执行
result = graph.invoke(Command(resume="approve"), config)
```

`interrupt` 的返回值就是后续 `Command(resume=...)` 中传入的值。这个设计使得中断点和恢复点之间的数据传递变得非常自然——中断时抛出的问题和恢复时收到的答案形成了一个清晰的请求-响应对。

一个图中可以包含多个 `interrupt` 调用，框架会按照执行顺序逐一处理它们。这使得实现"逐步审批"的工作流成为可能——每一步都可以暂停等待人类确认后再继续。但需要注意的是，恢复执行时图并不是从 `interrupt` 所在的代码行继续的，而是从包含 `interrupt` 的整个节点重新开始执行。这意味着节点中 `interrupt` 之前的代码会被重新运行，因此开发者需要确保这些代码是幂等的，或者将有副作用的操作包装在 `@task` 中。

## 4 LangChain Agent：高层抽象与编排模式

### 4.1 create_agent 工厂

LangChain 在 LangGraph 之上提供了 `create_agent` 工厂函数，作为构建 Agent 的高层入口。它封装了一个典型的 Agent 循环：模型接收消息和工具声明 → 模型生成响应（可能包含工具调用）→ 执行工具 → 将工具结果返回给模型 → 循环直到模型产出最终响应。

```
┌──────────────────────────────────────────────┐
│                 Agent Graph                   │
│                                              │
│   START ──► model_node ──► tools_node        │
│                 ▲               │             │
│                 │               │             │
│                 └───────────────┘             │
│                 (loop until done)             │
│                                              │
│             model_node ──► END               │
│             (no tool calls)                  │
└──────────────────────────────────────────────┘
```

这个循环在底层被实现为一个 LangGraph 的 `StateGraph`，包含两个核心节点（模型节点和工具节点）以及一个条件边（根据模型输出中是否包含 `tool_calls` 来决定是进入工具节点还是结束）。因为底层是 LangGraph，`create_agent` 创建的 Agent 自动继承了持久执行、流式输出和人在回路等能力。

`create_agent` 的设计目标是"10 行代码创建一个生产可用的 Agent"，这确实降低了入门门槛：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[search, calculator],
    system_prompt="你是一个有帮助的助手"
)

result = agent.invoke({"messages": [{"role": "user", "content": "帮我查一下..."}]})
```

但"简单"和"灵活"之间往往存在张力。`create_agent` 通过中间件（middleware）机制来提供扩展能力——开发者可以通过 `@wrap_model_call` 装饰器定义中间件，在模型调用前后注入自定义逻辑，如动态选择模型、过滤工具列表、修改 Prompt 等。这种中间件模式借鉴了 Web 框架（如 Express、Koa）的设计，提供了一个在不修改核心循环的前提下注入横切关注点的扩展机制。

### 4.2 多 Agent 编排

当系统的复杂度超越了单个 Agent 的能力边界时，需要将问题分解为多个专业化的 Agent 协作完成。LangGraph 支持通过子图（subgraph）来实现多 Agent 编排——每个 Agent 可以被定义为一个独立的子图，然后被嵌入到一个父图中作为节点使用。

多 Agent 之间的协作主要有两种模式。第一种是**监督者模式**（supervisor）：一个中心化的监督 Agent 负责接收用户请求、分析任务类型，并将任务分派给合适的专业 Agent 处理。监督者根据子 Agent 的返回结果决定是否需要进一步的处理或是否可以向用户返回最终答案。

```
                    ┌──────────────┐
                    │  Supervisor  │
                    │  (Router)    │
                    └──────┬───────┘
                     ┌─────┼─────┐
                     ▼     ▼     ▼
              ┌──────┐ ┌──────┐ ┌──────┐
              │Agent │ │Agent │ │Agent │
              │  A   │ │  B   │ │  C   │
              └──────┘ └──────┘ └──────┘
```

第二种是**交接模式**（handoff）：Agent 之间通过 `Command(goto=..., graph=Command.PARENT)` 将控制权直接传递给另一个 Agent，不经过中心化的调度。这种模式更加去中心化，适用于 Agent 之间的边界清晰、交接条件明确的场景。

两种模式各有适用场景。监督者模式在任务分解的灵活性上更强，但监督者本身成为了单点瓶颈——它的 Prompt 需要理解所有子 Agent 的能力边界，随着子 Agent 数量的增长，这个 Prompt 会变得越来越复杂。交接模式避免了中心化瓶颈，但要求每个 Agent 自行判断何时应该交接以及交接给谁，这对单个 Agent 的"自我认知"能力提出了更高的要求。

## 5 设计权衡与待改进之处

### 5.1 抽象层级的取舍

LangChain 生态中最常被讨论的问题是其抽象层级的选择。LangChain Core 的 Runnable 协议提供了高度统一的接口，但这种统一性也带来了认知负担——开发者需要理解 `invoke`、`batch`、`stream`、`transform` 等多个方法的语义差异，以及不同 Runnable 组合后这些方法行为的变化。在调试时，一层层的 Runnable 包装使得调用栈变得很深，难以快速定位问题所在。

LangGraph 的 `StateGraph` API 比 Runnable 的组合式 API 更直观——图的拓扑结构用节点和边显式地定义，状态的流转通过 reducer 显式地管理。但 LangGraph 也引入了自己的复杂度：checkpointer、super-step、reducer、Command、interrupt 等概念构成了一个相当陡峭的学习曲线。对于简单场景，这些概念是过度设计；对于复杂场景，它们又是必要的基础设施。框架面临的挑战是如何在"简单场景足够简单"和"复杂场景足够强大"之间找到平衡。

### 5.2 状态管理的边界

LangGraph 的状态模型是全局共享的——图中的所有节点都读写同一个 `State` 对象。虽然可以通过 `PrivateState` 和多 schema 机制实现一定程度的状态隔离，但这本质上还是基于命名约定而非强制隔离。在大型多 Agent 系统中，不同 Agent 可能对同一个状态字段有不同的理解和使用方式，这种隐式的耦合可能导致难以追踪的状态冲突。

与之形成对比的是 Google ADK 的做法：通过 `app:`、`user:`、`temp:` 等前缀约定来划分状态的作用域和生命周期。虽然两者都是"软约束"（框架不会阻止你违反约定），但 ADK 的前缀约定至少在命名层面提供了一种显式的意图声明。LangGraph 在这个方向上还有改进空间——比如引入 Agent 级别的状态命名空间，或者提供类型系统层面的状态访问控制。

### 5.3 确定性与非确定性的边界

Agent 系统的一个根本挑战是如何管理确定性代码和非确定性 LLM 输出之间的边界。LangGraph 的条件边将 LLM 的输出用作路由决策的依据，这意味着图的执行路径在运行时是非确定性的。持久执行机制通过检查点和回放来应对这种非确定性，但它要求开发者自行保证节点的幂等性和确定性——这不是一个容易满足的条件，尤其是当节点中包含复杂的业务逻辑时。

文档中明确指出了需要将非确定性操作包装在 `@task` 中，但框架本身并没有提供编译时或运行时的检查来验证这一约束是否被满足。一个包含未被 `@task` 包装的 API 调用的节点，在正常执行时工作良好，但在从中断恢复时可能会产生重复请求。这种"正确使用才正确"的 API 设计，在实际项目中容易成为隐蔽的 Bug 来源。

### 5.4 生态定位

LangChain 和 LangGraph 在 AI Agent 框架的生态中处于一个独特的位置：它们既不是像 Google ADK 那样由云厂商主导的、与特定基础设施深度绑定的框架，也不是像 AutoGen 或 CrewAI 那样专注于特定 Agent 模式的库。LangChain 的优势在于其广泛的模型和工具集成——几乎所有主流的 LLM 供应商和向量数据库都有对应的 LangChain 集成包。这种"万物可连接"的定位使得 LangChain 成为了 LLM 应用开发的"连接层"。

但这种广度也带来了维护上的挑战。LangChain 的核心仓库经历了多次重大重构（从早期的单体包到后来的 `langchain-core` + `langchain-community` 的拆分，再到现在的 `langchain` + `langgraph` 的分层），每次重构都会带来 API 的不兼容变更。对于生产环境中的用户来说，跟上框架的演进节奏本身就是一项成本。LangGraph 的出现在一定程度上缓解了这个问题——它提供了一个相对稳定的底层执行模型，使得上层 API 的变化不至于影响到核心的执行逻辑。但从长期来看，如何在快速迭代和 API 稳定性之间取得平衡，仍然是 LangChain 团队需要持续关注的课题。
