# Google ADK 架构解析：如何用软件工程思维构建 AI Agent 系统

Agent Development Kit（ADK）是 Google 在 2025 年开源的一个 AI Agent 开发框架，目前在 GitHub 上有超过 18000 个 star，支持 Python、TypeScript、Go 和 Java 四种语言的实现。ADK 的设计目标很明确：让 Agent 开发回归软件工程的范式，而不是停留在 prompt engineering 的阶段。这意味着它需要提供清晰的抽象层级、可组合的模块、确定性的执行流程、以及工业级的状态管理能力。

当前 AI Agent 领域的框架呈现出一种两极分化的态势：一端是 LangChain 这样高度灵活但过于松散的链式调用框架，另一端是各家云厂商封闭的托管服务。ADK 试图在这两者之间找到一个平衡点——既保持代码优先（code-first）的灵活性，又提供足够的结构化约束来支撑复杂的多 Agent 系统。这个定位是否站得住脚，需要从架构层面逐一拆解。

在进入细节之前，先从宏观上理解 ADK 的整体分层结构：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                          │
│              adk web / adk run / adk api_server                 │
├─────────────────────────────────────────────────────────────────┤
│                       Runner (Event Loop)                       │
│         yield/pause/resume cycle ── Event processing            │
├───────────────┬───────────────┬─────────────────────────────────┤
│  Agent Layer  │   Tool Layer  │         Flow Layer              │
│  BaseAgent    │   BaseTool    │     BaseLlmFlow                 │
│  ├ LlmAgent   │  ├ FunctionTool│     ├ SingleFlow               │
│  ├ Sequential │  ├ AgentTool  │     └ AutoFlow                  │
│  ├ Parallel   │  ├ MCPTool    │       (LLM request/response     │
│  ├ Loop       │  └ ...        │        + tool execution loop)   │
│  └ Custom     │               │                                 │
├───────────────┴───────────────┴─────────────────────────────────┤
│                      Model Abstraction                          │
│         BaseLlm ── LLMRegistry ── Gemini / OpenAI / ...        │
├─────────────────────────────────────────────────────────────────┤
│                      Services Layer                             │
│    SessionService  │  ArtifactService  │  MemoryService         │
│    (state + history)  (binary blobs)     (cross-session search) │
├─────────────────────────────────────────────────────────────────┤
│                      Infrastructure                             │
│        InMemory / Database / GCS / Vertex AI Agent Engine       │
└─────────────────────────────────────────────────────────────────┘
```

自底向上，Infrastructure 层提供可插拔的存储后端；Services 层管理状态、产物和记忆的持久化；Model 层抽象了不同 LLM 供应商的差异；Agent/Tool/Flow 三层构成了核心的执行逻辑；Runner 层驱动整个事件循环；Application 层则提供了面向开发者和终端用户的接口。

## 1 Agent 体系：从单一抽象到三层分类

### 1.1 BaseAgent 作为统一基座

ADK 的 Agent 体系建立在 `BaseAgent` 这个基类之上，所有类型的 Agent 都继承自它。`BaseAgent` 继承了 pydantic 的 `BaseModel`，这意味着每个 Agent 实例本质上是一个可序列化、可校验的数据模型——而非传统面向对象设计中那种行为驱动的对象。这是一个值得关注的设计选择：pydantic 的引入使得 Agent 的配置天然具备了类型校验、默认值填充和 JSON 序列化的能力，为后续的声明式配置（Agent Config）打下了基础。

`BaseAgent` 定义了几个关键的接口契约。`run_async` 是 Agent 的统一入口方法，它被 `@final` 装饰器标记为不可覆写，子类只能通过实现 `_run_async_impl` 来注入自己的执行逻辑。这种模板方法模式（Template Method Pattern）的运用使得框架能够在 `run_async` 中固定执行前后的 callback 处理、tracing 追踪等横切关注点，而将核心业务逻辑的变化点留给子类。从 `run_async` 的源码中可以清楚地看到这种结构：

```python
@final
async def run_async(self, parent_context: InvocationContext) -> AsyncGenerator[Event, None]:
    with tracer.start_as_current_span(f'invoke_agent {self.name}') as span:
        ctx = self._create_invocation_context(parent_context)
        if event := await self._handle_before_agent_callback(ctx):
            yield event
            if ctx.end_invocation:
                return
        async with Aclosing(self._run_async_impl(ctx)) as agen:
            async for event in agen:
                yield event
        if event := await self._handle_after_agent_callback(ctx):
            yield event
```

`run_async` 的返回类型是 `AsyncGenerator[Event, None]`，这不是一个偶然的选择——异步生成器是整个运行时事件循环机制的基石，后文会详细讨论。

Agent 之间通过 `sub_agents` 字段建立父子关系，形成一棵 Agent 树。框架在 `model_post_init` 阶段自动为每个子 Agent 设置 `parent_agent` 引用，并强制约束单亲规则：一个 Agent 实例只能被添加为一个父 Agent 的子节点。如果需要复用相同逻辑的 Agent，必须通过 `clone` 方法创建新实例。这个约束从根本上避免了 Agent 树退化为 DAG 或更复杂拓扑结构时带来的状态管理困难——在多 Agent 系统中，一个 Agent 同时存在于两条执行路径中的话，其上下文和生命周期的管理会变得异常棘手。

### 1.2 三种 Agent 类型的划分逻辑

在 `BaseAgent` 之上，ADK 将 Agent 划分为三个类别：LLM Agent、Workflow Agent 和 Custom Agent。这个分类的本质是对 Agent 执行逻辑中确定性程度的划分。

```
                         BaseAgent
                        (pydantic BaseModel)
                             │
              ┌──────────────┼──────────────────┐
              ▼              ▼                   ▼
          LlmAgent      Workflow Agents      Custom Agent
       (non-deterministic)  (deterministic)   (user-defined)
              │              │
              │    ┌─────────┼──────────┐
              │    ▼         ▼          ▼
              │  Sequential  Parallel   Loop
              │  Agent       Agent      Agent
              │
         ┌────┴────┐
         ▼         ▼
    BaseLlmFlow   Tools
    (LLM calls)   (FunctionTool, AgentTool, ...)
```

LLM Agent（即 `LlmAgent`，同时也有一个别名 `Agent`）是以大语言模型作为核心推理引擎的 Agent。它的行为是非确定性的——给定相同的输入，LLM 可能产出不同的响应、选择不同的工具、或将控制权委托给不同的子 Agent。`LlmAgent` 的配置项非常丰富，包括模型选择（`model`）、系统指令（`instruction`）、工具列表（`tools`）、输出键（`output_key`）等，几乎所有与 LLM 交互相关的参数都在这个类上定义。值得注意的是 `model` 字段的继承机制：如果一个 `LlmAgent` 没有显式指定模型，它会沿着 Agent 树向上查找祖先节点的模型配置，如果整棵树都没有指定，则回退到类级别的默认模型。这种继承策略降低了多 Agent 系统中模型配置的冗余度。

Workflow Agent 是确定性的编排器，包括 `SequentialAgent`、`ParallelAgent` 和 `LoopAgent` 三种。它们不调用 LLM，只负责按照预定义的模式（顺序、并行、循环）调度子 Agent 的执行。以 `SequentialAgent` 为例，它的 `_run_async_impl` 实现非常直接：按照 `sub_agents` 列表的顺序，依次调用每个子 Agent 的 `run_async` 方法，并将产出的事件逐一 yield 出去。子 Agent 之间通过共享的 session state 传递数据——前一个 Agent 将结果写入 state 的某个 key，后一个 Agent 从 instruction 的模板变量中读取它。

这种三层分类的实用价值在于，实际构建 Agent 系统时经常需要在"让 LLM 自主决策"和"用代码硬编码流程"之间找到恰当的边界。Workflow Agent 提供了一种中间地带：流程的宏观结构是确定的，但每个步骤内部可以是 LLM 驱动的。比如一个内容审核流水线可以用 `SequentialAgent` 串联"提取关键信息"→"风险评估"→"生成报告"三个 LLM Agent，整体流程不会因为 LLM 的幻觉而跑偏，但每一步的具体推理又能充分利用 LLM 的能力：

```python
extract = LlmAgent(name="Extract", model="gemini-2.5-flash",
                   instruction="提取文本中的关键实体和事件", output_key="entities")
assess  = LlmAgent(name="Assess", model="gemini-2.5-flash",
                   instruction="基于 {entities} 评估内容风险等级", output_key="risk")
report  = LlmAgent(name="Report", model="gemini-2.5-flash",
                   instruction="根据风险等级 {risk} 生成审核报告")

pipeline = SequentialAgent(name="ContentReview", sub_agents=[extract, assess, report])
```

### 1.3 多 Agent 协作的两种范式

ADK 中多 Agent 之间的协作有两种根本不同的范式：LLM 驱动的动态委托（Agent Transfer）和 Workflow Agent 驱动的静态编排。

```
动态委托 (Agent Transfer)              静态编排 (Workflow Agent)
┌──────────────────────┐              ┌──────────────────────┐
│   Coordinator (LLM)  │              │   SequentialAgent    │
│                      │              │                      │
│  "I need weather     │              │   step1 ──► step2    │
│   info, let me       │              │              │       │
│   transfer to..."    │              │              ▼       │
│         │            │              │            step3     │
│    transfer_to_agent │              │  (deterministic      │
│         │            │              │   execution order)   │
│         ▼            │              │                      │
│   WeatherAgent (LLM) │              └──────────────────────┘
│   "The temp is..."   │
│         │            │              ┌──────────────────────┐
│      escalate        │              │   ParallelAgent      │
│         │            │              │                      │
│         ▼            │              │  ┌─────┐  ┌─────┐   │
│   Coordinator (LLM)  │              │  │ A   │  │ B   │   │
│   continues...       │              │  └─────┘  └─────┘   │
└──────────────────────┘              │  (concurrent exec)   │
                                      └──────────────────────┘
```

在动态委托模式下，一个 `LlmAgent` 的子 Agent 列表会被框架自动转换为工具描述（通过 `AgentTool` 包装），注入到该 Agent 的 LLM prompt 中。当 LLM 判断当前任务应该由某个子 Agent 处理时，它会发出一个 `transfer_to_agent` 的 function call，框架捕获这个调用后将执行控制权转移给目标 Agent。控制权转移之后，当前 Agent 的上下文会被暂存，目标 Agent 开始在自己的上下文中运行，直到它显式地通过 `escalate` 或类似机制将控制权交还。在此期间产生的所有事件都会被记录到同一个 session 的 event history 中，保证了对话历史的完整性。

静态编排模式则更加直接。`ParallelAgent` 的实现值得一提：它为每个子 Agent 创建不同的 branch（通过修改 `InvocationContext.branch` 字段），虽然多个子 Agent 共享同一个 session state，但 branch 的隔离使得事件历史在逻辑上是分离的。不过正因为 state 是共享的，并行分支之间如果写入相同的 key 就会产生竞态条件——文档中也明确提醒开发者需要为并行分支使用不同的 state key。

这两种范式可以嵌套组合。一个典型的模式是：顶层使用 `SequentialAgent` 定义流水线，流水线中的某个步骤是一个 `LlmAgent`，该 `LlmAgent` 又有多个子 Agent 可以被动态委托。这种组合方式使得系统既有确定性的骨架，又有灵活性的末梢。

## 2 运行时引擎：事件循环与状态管理

### 2.1 基于协程的事件循环

ADK 的运行时核心是一个事件循环（Event Loop），其运作机制与 Python 中的生成器协程有着高度的相似性。`Runner` 作为编排者，`Agent`（及其关联的工具和回调）作为执行逻辑，两者之间通过 `Event` 对象进行协作式的交互。

```
User Query
    │
    ▼
┌─────────┐     yield Event      ┌──────────────────┐
│  Runner  │◄────────────────────│  Agent Logic      │
│          │                     │  (AsyncGenerator) │
│ 1.receive│                     │                   │
│ 2.commit │─── resume ─────────►│ yield event_1     │
│   state  │                     │ # ── pause ──     │
│ 3.forward│◄────────────────────│                   │
│   to UI  │                     │ # ── resumed ──   │
│ 4.resume │─── resume ─────────►│ yield event_2     │
│          │                     │ # ── pause ──     │
│          │◄────────────────────│                   │
│ ...      │                     │ ...               │
│          │                     │ (generator done)  │
└─────────┘                      └──────────────────┘
```

执行逻辑在运行过程中，每当需要向外部报告结果、请求工具调用或提交状态变更时，就构造一个 `Event` 并通过 `yield` 将其交还给 `Runner`。`Runner` 接收到 `Event` 后，通过 `SessionService` 将事件中携带的副作用（`state_delta`、`artifact_delta`）持久化到 session 中，然后将事件转发给上游（通常是 UI 或 API 层），最后再通知执行逻辑继续运行。执行逻辑恢复执行时，可以确信之前 yield 的状态变更已经被可靠地提交。

这个 yield/pause/resume 的循环和 Python 生成器协程的工作原理如出一辙——Agent 的 `_run_async_impl` 方法就是一个 `AsyncGenerator`，每次 `yield` 就像协程的挂起点，`Runner` 的 `async for` 消费事件就像协程的驱动方。如果读过 CPython 中生成器的实现（`YIELD_VALUE` 指令取出栈顶数据、修改栈帧指针、通过 goto 退出执行循环），就能理解 ADK 这种设计在概念上的渊源——只不过 ADK 将栈帧的角色换成了 `InvocationContext`，将字节码执行换成了 Agent 的业务逻辑。

相比于回调链或消息队列，选择 AsyncGenerator 作为执行模型有两个明显的优势：一是执行逻辑的代码可以写成看似同步的线性流程，不需要拆分成多个回调函数或状态机状态；二是 `yield` 提供了一个天然的同步点，使得状态提交的时序语义变得清晰可预测。

### 2.2 Event：运行时的信息载体

`Event` 是 ADK 运行时中最核心的数据结构之一。每个 Event 都由 `invocation_id`（标识这次用户请求）、`author`（产生事件的 Agent 名称）、`branch`（执行分支标识）、`content`（事件的内容载体）和 `actions`（事件携带的副作用声明）等字段组成。

`content` 字段遵循 Gemini API 的 `types.Content` 结构，可以包含文本、function call、function response 等多种类型的 `Part`。这意味着一个 Event 可以表达"Agent 说了一段话"、"Agent 请求调用某个工具"、"工具返回了结果"等多种语义。

`actions` 字段（`EventActions` 类型）是理解 ADK 状态管理模型的关键。它包括 `state_delta`（一个字典，描述了 session state 的增量变更）、`artifact_delta`（二进制产物的变更记录）、`transfer_to_agent`（控制权转移的目标 Agent）、`escalate`（向上级交还控制权）等字段。状态变更不是通过直接修改 session 对象来实现的，而是被声明在 Event 的 actions 中，由 `Runner` 在事件处理阶段统一提交。这种"声明式副作用"的设计模式常见于事件溯源（Event Sourcing）架构中，它使得每一次状态变更都是可追溯、可回放的。一个典型的状态变更流程如下：

```python
# Agent 内部的执行逻辑
ctx.session.state['status'] = 'processing'
event = Event(
    author=self.name,
    actions=EventActions(state_delta={'status': 'processing'}),
    content=types.Content(parts=[types.Part(text="开始处理")])
)
yield event
# ── 暂停，Runner 提交 state_delta 到 SessionService ──
# ── 恢复执行，此时 state['status'] 已被可靠持久化 ──
current = ctx.session.state['status']  # 'processing'
```

这里存在一个有趣的权衡：ADK 允许所谓的"脏读"（dirty read）。在同一次 invocation 中，一个 callback 修改了 state，后续的 tool 可以立即读到这个修改，即使携带这次修改的 Event 还没有被 `Runner` 正式提交。这提高了同一执行步骤内不同组件之间的协调效率，但也引入了一致性风险——如果 invocation 在提交前异常终止，脏读到的状态变更将会丢失。文档中提醒开发者不要对关键状态转移依赖脏读机制，这是一个务实但略显妥协的处理方式。

### 2.3 Session、State 与 Memory 的三层上下文模型

ADK 对 Agent 运行时的上下文做了清晰的三层划分：

```
┌───────────────────────────────────────────────────────────────┐
│                     Memory (MemoryService)                     │
│          跨 Session 的长期语义记忆，支持搜索和检索               │
│          scope: 跨所有会话                                      │
├───────────────────────────────────────────────────────────────┤
│                     Session (SessionService)                   │
│    ┌─────────────────────┐  ┌──────────────────────────────┐  │
│    │    Event History     │  │         State (dict)         │  │
│    │  [event_0, event_1,  │  │  "app:config"  → global     │  │
│    │   event_2, ...]      │  │  "user:pref"   → per-user   │  │
│    │  (ordered log of     │  │  "cart_items"   → session    │  │
│    │   all interactions)  │  │  "temp:scratch" → invocation │  │
│    └─────────────────────┘  └──────────────────────────────┘  │
│          scope: 单次对话                                       │
├───────────────────────────────────────────────────────────────┤
│                     Invocation                                 │
│          单次用户请求的完整处理过程                               │
│          scope: 单次请求（temp: 前缀的 state 仅在此范围内有效）   │
└───────────────────────────────────────────────────────────────┘
```

Session 代表一次完整的对话线程。它包含对话的事件历史（event history，即所有 Event 的有序列表）、当前的状态数据（state），以及会话级的元数据。每次用户发起请求（invocation）时，`Runner` 会从 `SessionService` 加载对应的 Session，将用户输入作为第一个 Event 追加到历史中，然后启动 Agent 的执行。Session 的生命周期由 `SessionService` 管理，ADK 提供了 `InMemorySessionService`（用于开发测试）和基于数据库/云服务的持久化实现。

State 是存储在 Session 中的键值数据。除了常规的 state 之外，ADK 还引入了以 `app:` 和 `user:` 为前缀的 state key 来分别表示跨会话级别的应用状态和用户状态，以及以 `temp:` 为前缀的临时状态（仅在当前 invocation 内有效）。这种前缀约定（prefix convention）是一种轻量级的命名空间机制，避免了引入更复杂的多层 state store 的设计。

Memory 是跨 Session 的长期记忆，由 `MemoryService` 管理，支持将历史 Session 的内容摄入（ingest）到记忆库中，并通过语义搜索进行检索。Memory 和 Session 的职责边界是：Session 关注"当前对话说了什么"，Memory 关注"过去的对话中有什么值得记住的信息"。

这三层模型覆盖了 Agent 系统中三种典型的信息时效性需求：invocation 级（temp state）、session 级（regular state + event history）、跨 session 级（memory + app/user state）。分层在概念上是清晰的，但在实际使用中开发者需要自行判断信息应该放在哪一层，框架并没有提供更强的约束或指导——这可能是一个后续值得改进的方向。

## 3 工具系统：从函数到生态

### 3.1 工具抽象层级

ADK 的工具系统围绕 `BaseTool` 基类构建。最常用的工具类型是 `FunctionTool`，它将一个普通的 Python 函数包装为 Agent 可调用的工具。框架会通过函数签名的类型标注自动生成工具的 schema 描述，传递给 LLM 用于 function calling。这意味着开发者只需要写一个带类型标注和 docstring 的函数，就能让 Agent 调用它——开发体验上的摩擦非常低。

在 `FunctionTool` 之上，ADK 还提供了 `AgentTool`（将一个 Agent 包装为另一个 Agent 的工具）、`LongRunningFunctionTool`（支持异步长时任务的工具）等变体。`AgentTool` 的设计尤其值得关注：当一个 `LlmAgent` 的 `sub_agents` 被框架处理时，实际上每个子 Agent 都会被隐式地包装成 `AgentTool`，使得父 Agent 的 LLM 可以通过 function call 的方式来触发子 Agent 的执行。这种"Agent 即工具"的统一抽象消除了 Agent 间调用和 Agent 调用工具之间的语义鸿沟。

工具的执行流程中嵌入了丰富的 callback 钩子：`before_tool_callback` 在工具执行前触发，可以拦截或修改工具的输入参数；`after_tool_callback` 在工具返回后触发，可以修改或替换工具的输出。这些钩子的存在使得日志记录、权限检查、输入输出校验等横切逻辑可以被干净地注入，而不需要修改工具本身的实现。

### 3.2 工具确认机制与安全考量

ADK 引入了一套工具确认（Tool Confirmation）机制来实现人在回路（Human-in-the-Loop）的控制。当一个工具被标记为需要确认时，Agent 在请求调用该工具后不会立即执行，而是先向用户发送一个确认请求，等待用户批准后才继续执行。这个机制在涉及外部副作用的场景（发送邮件、修改数据库、执行交易等）中至关重要。

从安全的视角来看，ADK 还面临着更深层的挑战。在 Agent 系统中，LLM 的输出被直接用来驱动工具调用和 Agent 间的控制流转移，这意味着 prompt injection 攻击可以直接影响系统的执行路径。ADK 在文档中专门设立了安全章节来讨论这些问题，但客观地说，目前的防护措施更多还是停留在最佳实践和架构建议的层面，尚未形成一套自动化的安全防线。

## 4 LLM 交互层：Flow 的设计

### 4.1 BaseLlmFlow 与请求/响应管道

`LlmAgent` 并不直接与 LLM API 交互，而是通过 `BaseLlmFlow` 这个中间层来管理 LLM 调用的全过程。Flow 的职责包括：将 session 的 event history 转换为 LLM 可消费的消息列表（content assembly）、构建 LLM 请求（包括系统指令、工具声明等）、调用 LLM API、处理 LLM 的响应（文本输出或 function call）、以及在需要工具调用时执行工具并将结果反馈给 LLM。

这个 Flow 的核心循环可以用以下流程概括：

```
                    ┌──────────────────┐
                    │  Content Assembly │
                    │  (history → msgs) │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │   Call LLM API    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
               ┌────│ Response Type?   │────┐
               │    └──────────────────┘    │
          text │                            │ function_call
               ▼                            ▼
    ┌──────────────────┐         ┌──────────────────┐
    │ yield text Event  │         │  Execute Tool     │
    │ (final response)  │         │  yield FC Event   │
    └──────────────────┘         └────────┬─────────┘
                                          ▼
                                 ┌──────────────────┐
                                 │ yield FR Event    │
                                 │ (function result) │
                                 └────────┬─────────┘
                                          │
                                          └──── loop back ────►
```

整个过程中每一个 LLM 响应和工具调用结果都会被包装为 Event 并 yield 出去，确保 `Runner` 能够追踪和持久化每一步的中间状态。

Flow 层还承担了 streaming 处理的责任。当使用流式 LLM API 时，LLM 的响应会以 token 为单位逐步返回。Flow 将这些部分响应包装为 `partial=True` 的 Event yield 出去，`Runner` 会将这些部分事件直接转发给 UI 以实现流式展示，但不会对它们执行 state 提交操作——只有最终的完整响应才会触发状态的持久化。这个区分保证了 state 的原子性不会被 streaming 打破。

### 4.2 模型抽象与多模型支持

ADK 通过 `BaseLlm` 抽象类和 `LLMRegistry` 来实现模型的可替换性。虽然框架针对 Google 的 Gemini 系列模型做了深度优化（使用 `google-genai` SDK），但开发者可以通过实现 `BaseLlm` 接口来接入任何 LLM。`LLMRegistry` 是一个全局注册表，将模型名称字符串（如 `"gemini-2.5-flash"` 或 `"openai/gpt-4o"`）映射到对应的 `BaseLlm` 实现类。`LlmAgent` 的 `model` 字段接受字符串或 `BaseLlm` 实例，前者会在运行时通过 Registry 解析。

这种注册表模式（Registry Pattern）使得模型的切换可以在不修改 Agent 代码的情况下完成——只需要在初始化阶段注册不同的模型实现即可。但这也意味着模型的能力差异（是否支持 function calling、上下文窗口大小、多模态能力等）需要由开发者自行保证兼容性，框架不会在编译时或初始化时进行能力检查。

## 5 部署与可观测性

ADK 提供了从本地开发到生产部署的多层运行方式。`adk web` 启动一个带有 Web UI 的开发服务器，可以直接在浏览器中与 Agent 交互和调试；`adk run` 提供命令行交互模式；`adk api_server` 暴露 RESTful API 供外部系统集成。在生产环境中，Agent 可以被容器化并部署到 Cloud Run，或通过 Vertex AI Agent Engine 进行托管和扩缩容。

在可观测性方面，ADK 通过 OpenTelemetry 集成提供了分布式追踪（tracing）能力。`BaseAgent.run_async` 的入口处会创建一个 tracing span，使得每个 Agent 的调用都能在追踪系统中形成完整的调用链路。这对于调试多 Agent 系统中的执行路径和性能瓶颈非常重要。

评估（Evaluation）是 ADK 提供的另一个有价值的基础设施。通过 `adk eval` 命令，开发者可以针对预定义的测试用例集评估 Agent 的表现，包括最终响应的质量和逐步执行轨迹的正确性。这使得 Agent 的开发可以像传统软件一样建立持续集成和回归测试的流程。

## 6 设计权衡与待改进之处

在 ADK 的文档和源码中，有几个设计决策值得进一步讨论。

首先是 pydantic 作为 Agent 基座的选择。pydantic 带来了出色的配置校验和序列化能力，但也引入了不可忽视的运行时开销。在高吞吐的 Agent 服务中，每次 Agent 实例化和事件创建时的 pydantic 校验是否会成为性能瓶颈，需要在实际工作负载下进行基准测试。

其次是 session state 的并发安全模型。在 `ParallelAgent` 的场景下，多个子 Agent 共享同一个 state 字典，框架将避免竞态条件的责任完全交给了开发者（通过使用不同的 key）。这种做法在小规模系统中是可行的，但在复杂的多 Agent 系统中，手动管理 key 的命名空间容易出错。一个可能的改进方向是引入 Agent 级别的 state 作用域隔离，或者提供类似 CRDT 的冲突自动解决机制。

第三是 Agent Transfer 的控制流语义。当前的 Agent Transfer 实际上是通过 LLM 的 function call 来触发的，这意味着 Transfer 的发生取决于 LLM 的判断，而非开发者的显式代码。在某些需要精确控制的场景中，这种非确定性可能并不理想。ADK 通过 Workflow Agent 提供了确定性的替代方案，但两种方式之间如何选择、如何组合，对于框架的新用户来说可能需要一定的学习成本。

最后，ADK 作为一个相对年轻的框架（2025 年开源），其生态的广度仍在快速扩展中。目前的集成列表已经覆盖了 Google Cloud 全家桶、多种主流的 MCP 工具、以及 Observability 方案，但在跨框架互操作（如与 LangChain、CrewAI 的 Agent 互调用）方面仍有空间。A2A（Agent-to-Agent）协议的引入是朝这个方向迈出的一步，但距离成为行业标准还有很长的路要走。

整体而言，ADK 的架构设计体现了 Google 在大规模分布式系统方面的工程经验——清晰的抽象层级、基于事件的解耦设计、声明式的副作用管理、以及可插拔的服务后端。它不是一个试图讨好所有人的万能框架，而是一个在"结构化"和"灵活性"之间找到了自己位置的工程导向产品。对于需要构建生产级 Agent 系统的团队来说，ADK 提供了一个值得认真考量的起点。
