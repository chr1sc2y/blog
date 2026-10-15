---
title: "Comparing Agent SDKs and Frameworks: Who Owns the Loop, State, and Runtime?"
date: 2026-10-15T10:00:00+08:00
lastmod: 2026-10-15
draft: false
categories: ["AI Agents"]
description: "Compare OpenAI Agents, Claude, Codex, Google ADK, Vercel AI SDK, LangChain/LangGraph, Pydantic AI, Microsoft Agent Framework, and CrewAI through execution, state, and recovery."
---

Agent development frameworks increasingly expose similar vocabulary: tools, sessions, memory, handoffs, and streaming. That vocabulary hides substantial architectural differences. Defining tools for an OpenAI agent, embedding Claude Code through the Claude Agent SDK, and compiling a stateful LangGraph workflow place different parts of the system under framework control. Comparing them through a list of supported features misses the decision that will determine most of the integration work: which execution responsibilities the application keeps, and which it delegates.

The useful starting point is a layered architecture:

```text
Product             chat UI, task API, acceptance criteria
                         |
Control flow        model/tool loop, routing, workflow transitions
                         |
State               conversation, business records, run checkpoints
                         |
Execution           application functions, files, shell, external services
                         |
Infrastructure      workers, isolation, storage, identity, telemetry
```

A framework can span several layers. Its name does not establish where the boundary lies.

This comparison covers nine agent-development stacks: OpenAI Agents SDK, Claude Agent SDK, Codex SDK, Google ADK, Vercel AI SDK, LangChain/LangGraph, Pydantic AI, Microsoft Agent Framework, and CrewAI. Each provides agent execution or orchestration abstractions. Their useful differences concern reusable agent loops, developed coding harnesses, typed application contracts, and explicit workflows. Those responsibilities overlap, so the comparison follows execution behavior rather than treating every framework as an interchangeable package.

The scope is architectural behavior documented and inspected on October 15, 2026. Source links point to inspected commits; documentation links can evolve. These are qualitative engineering judgments, not benchmark results or adoption rankings. Language implementations within a project may expose different capabilities.

## 1. Establish the execution boundary

Consider an incident assistant. It reads telemetry, inspects a repository, proposes a patch, requests approval, and applies a change. Every implementation needs model inference, but that is only one step. Someone must schedule tool calls, preserve evidence, decide whether another model turn is necessary, authenticate approvals, and recover after a worker disappears.

The first comparison is therefore about ownership:

| Integration | Primary abstraction | Main execution responsibility | Application responsibility to examine |
| --- | --- | --- | --- |
| OpenAI Agents SDK | Agent and runner | Repeated model/tool execution and agent handoffs | Deployment, tool implementations, storage and approval policy |
| Claude Agent SDK | Programmable Claude Code session | An existing harness with tools and context management | Process hosting, workspace lifecycle and isolation |
| Codex SDK | Codex thread and turn | Programmatic execution of the Codex harness | Local runtime, workspaces and thread lifecycle |
| Google ADK | Agents, runner, events and services | Agent execution with structured workflow components | Service backends, state design and tool boundaries |
| Vercel AI SDK | Model/provider interface and agent loop | TypeScript generation, tool execution and product streaming | Choice of in-memory or workflow execution and persistence |
| LangChain / LangGraph | High-level agent and state graph | Ready-made agent loops or explicit checkpointed transitions | Choice of abstraction, node logic and external-effect semantics |
| Pydantic AI | Typed agent, dependencies and output | Tool execution with typed application interfaces | Semantic validation, storage and durable-engine integration |
| Microsoft Agent Framework | Agents, sessions and workflows | Agent execution and explicit multi-agent coordination | Hosting, checkpoint storage and durable-extension configuration |
| CrewAI | Agents, tasks, crews and flows | Role/task collaboration and event-driven workflow logic | Task boundaries, persistence and validation of delegated work |

The table describes integration surfaces rather than hard capability ceilings. A coding-harness SDK starts with an established file/tool execution model. An agent-loop SDK starts with an application-defined tool contract. A workflow framework gives transitions and state their own explicit representation. For the incident assistant, the relevant decision is whether the product primarily needs repository work, bounded tool use, or a business process that must survive interruption.

I would start a design review by drawing this boundary before discussing model quality. Moving the loop from application code into a framework changes debugging and lifecycle responsibilities even when the underlying model remains identical.

## 2. Follow one tool call through each architecture

### 2.1 OpenAI Agents SDK: delegate the recurring loop

The OpenAI Agents SDK moves recurring agent execution into a runner. An agent definition groups instructions, tools and related behavior; the runner handles repeated execution and handoffs. The inspected [`Runner.run` implementation and its underlying runner](https://github.com/openai/openai-agents-python/blob/17d994bd3045dae71acda676a4ef4762ed8ecd94/src/agents/run.py) make the control flow explicit: final output ends a run, a handoff changes the active agent, and tool calls can lead to another turn. See also the [SDK overview](https://developers.openai.com/api/docs/guides/agents/sdk).

A handoff and a specialist exposed as a tool have different ownership semantics. With a handoff, the active conversational role changes. With a specialist tool, the caller can retain responsibility for assembling the answer. This distinction is more useful than simply counting supported agents: it determines which component must enforce the final response contract. The official [orchestration guide](https://developers.openai.com/api/docs/guides/agents/orchestration) describes these patterns.

My preference would be to use the runner when its execution contract matches the product. Reimplementing a standard tool loop adds maintenance without much differentiation. Conversely, forcing a transaction-oriented workflow into model-directed handoffs can obscure the business states that operations teams need to inspect.

### 2.2 Claude Agent SDK: embed a developed harness

Claude Agent SDK starts from another point in the stack: it exposes the agent loop, tools and context management used by Claude Code. Python and TypeScript applications can operate that harness programmatically, including built-in file and command tools, sessions, hooks and MCP integrations. The [overview](https://code.claude.com/docs/en/agent-sdk/overview) explicitly distinguishes it from direct client-SDK access.

In the inspected Python source, [`query()`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/query.py) delegates to [`InternalClient.process_query()`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/_internal/client.py), which selects a transport and uses the subprocess CLI transport by default. The Python wrapper is therefore not the complete implementation of the underlying agent. Some behavior lives across a process boundary.

For repository maintenance, that is an attractive starting point: file inspection, editing and command execution already have a coherent interaction model. For a narrow customer-service product, those capabilities may introduce more runtime surface than the application needs.

The engineering question becomes whether the product wants this harness's working model. Its host must account for workspaces, subprocesses, cancellation and tool permissions. Those responsibilities belong in deployment design, even if the integration begins with a short `query()` call. Anthropic's [hosting guidance](https://code.claude.com/docs/en/agent-sdk/hosting) addresses the operating environment separately from the library API.

### 2.3 Codex SDK: program a coding agent through threads

Codex SDK also embeds an existing coding harness, making it a direct architectural peer of Claude Agent SDK. The official [SDK documentation](https://learn.chatgpt.com/docs/codex-sdk) describes starting, continuing and resuming local Codex threads. In the inspected TypeScript implementation, [`Codex.startThread()` and `resumeThread()`](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/codex.ts) create thread handles, while [`Thread.run()` and `runStreamed()`](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/thread.ts) drive turns. The [`CodexExec` transport](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/exec.ts) manages interaction with a Codex executable.

For an incident assistant that must inspect a repository and attempt a patch, this provides a concrete execution environment to integrate. It also makes the thread and workspace separate lifecycle resources. Resuming a conversation cannot, by itself, establish that the checked-out branch or generated files still match the earlier turn.

I would compare Codex SDK and Claude Agent SDK on repository behavior, permission boundaries, cancellation, event handling and workspace recovery. For a product that needs detailed interactive lifecycle control, OpenAI separately documents [Codex app-server](https://developers.openai.com/blog/codex-as-a-platform); that integration should be evaluated on its own terms. Neither harness should be scored as though it were simply a provider-neutral business-tool loop.

### 2.4 Google ADK: agents within an event and service architecture

Google ADK gives agent behavior a runtime context composed of runners, events and backing services. The inspected Python [`Runner`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/runners.py) coordinates execution and session-service interactions. This makes session persistence and event processing visible architectural components rather than incidental message-list handling. The [runtime documentation](https://adk.dev/runtime/) explains the separation.

ADK also exposes explicit sequential, parallel and loop workflow agents. In [`SequentialAgent._run_async_impl`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/agents/sequential_agent.py), execution advances through child agents; the sequence is represented by application structure rather than inferred afresh by a model. The [workflow-agent documentation](https://adk.dev/agents/workflow-agents/) distinguishes these template workflows from model-directed behavior.

That separation suits pipelines whose stages are known: retrieve evidence, analyze it, then validate a report. A model can reason inside a stage while the application controls the stage order. Parallel branches still need deliberate data ownership; sharing an invocation does not make concurrent writes into independent fields automatically coherent.

I would evaluate ADK when the service architecture fits the surrounding application, then verify the particular language implementation and persistence backend. A framework's cloud integrations are useful options, but they do not by themselves establish deployment requirements or guarantee provider equivalence.

### 2.5 Vercel AI SDK: connect the loop to a product interface

Vercel AI SDK combines provider abstractions, generation primitives and UI integration. Its [`ToolLoopAgent`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/agent/tool-loop-agent.ts) packages a reusable tool loop around generation and streaming. The inspected stop-condition source exports [`isStepCount`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/generate-text/stop-condition.ts); current examples use that name. Older examples using other spellings should be checked against the installed package.

A minimal configuration shape looks like this:

```typescript
import { ToolLoopAgent, isStepCount } from 'ai';

// model and tools are application-supplied, provider-compatible objects.
const incidentAgent = new ToolLoopAgent({
  model,
  tools,
  instructions: 'Investigate the incident and cite the evidence used.',
  stopWhen: isStepCount(8),
});
```

This snippet illustrates configuration only. It leaves provider credentials, tool schemas, persistence and the HTTP route to the application. The [loop-control documentation](https://ai-sdk.dev/docs/agents/loop-control) explains stopping and per-step customization.

The product-facing advantage is the connection between server-side tool activity and interactive UI. The [chatbot documentation](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot) describes message handling and streaming integration. This can reduce translation work for a TypeScript application whose main complexity is presenting partial text, tool activity and user responses.

It would nevertheless be outdated to characterize the entire ecosystem as request-bound. The separately packaged [`WorkflowAgent`](https://ai-sdk.dev/docs/agents/workflow-agent) runs inside a workflow and adds durable execution and resumable approvals. Choosing that integration changes the runtime contract. It should be evaluated separately from an ordinary in-memory `ToolLoopAgent`.

### 2.6 LangChain and LangGraph: choose the level of workflow control

LangChain and LangGraph occupy related layers. LangChain's [`create_agent`](https://github.com/langchain-ai/langchain/blob/84ba23c414eca6bf6f6ebd1394a3006378113540/libs/langchain_v1/langchain/agents/factory.py) constructs a graph-based agent runtime, giving applications a higher-level entry point for models, tools and middleware. The [agent documentation](https://docs.langchain.com/oss/python/langchain/agents) describes that ready-made loop. An application does not have to design a custom graph merely to use an agent in this ecosystem.

LangGraph becomes the more direct interface when workflow transitions need explicit application structure. Its inspected [`StateGraph` and `compile()`](https://github.com/langchain-ai/langgraph/blob/5eb60aef6f3837a1fb632a8abc6cf8e8da81785b/libs/langgraph/langgraph/graph/state.py) expose state updates, graph construction and checkpointer configuration. The [persistence documentation](https://docs.langchain.com/oss/python/langgraph/persistence) distinguishes checkpointed thread state from stores used across threads.

For an incident assistant, I would start with the higher-level agent when the task is mainly repeated tool use. A custom graph earns its additional state and transition definitions when investigation, review and remediation must remain separately inspectable across failures. Using a graph provides a place to represent those obligations; the application must still define them correctly and make external operations safe to replay.

### 2.7 Pydantic AI: make application contracts explicit

An incident assistant eventually needs to return something the application can act on: an incident identifier, evidence references, a proposed action and a review status. Pydantic AI makes typed dependencies and output definitions central to that integration. Its [`Agent` implementation](https://github.com/pydantic/pydantic-ai/blob/9ac853bb84ffed7ba0edccb026c5b4f3228c1de7/pydantic_ai_slim/pydantic_ai/agent/__init__.py) connects these contracts to agent execution; the official [dependency guide](https://pydantic.dev/docs/ai/core-concepts/dependencies/) explains how tools receive application services through run context.

That separation gives the model a tool interface while keeping a tenant-scoped database client or policy service in application code. It also gives a Python team a concrete seam for dependency substitution during testing. The [output guide](https://pydantic.dev/docs/ai/core-concepts/output/) describes validating returned data against output types.

A valid schema still cannot prove that the cited evidence supports a diagnosis. I would use typed output to reject malformed results, then apply domain checks to verify identifiers, evidence and allowed transitions. This distinguishes interface correctness from task correctness.

Durability remains a separate execution choice. Pydantic AI documents integrations with engines including Temporal and DBOS in its [durable-execution guide](https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/). A typed agent definition alone does not mean a run survives worker replacement. I would evaluate the chosen engine's replay behavior, serialization constraints and deployment cost alongside the agent API.

### 2.8 Microsoft Agent Framework: connect agents and explicit workflows

Microsoft Agent Framework provides another useful comparison for applications that combine autonomous tool use with application-controlled coordination. Its [overview](https://learn.microsoft.com/en-us/agent-framework/overview/) describes agents, session state, middleware and graph-based workflows. Microsoft identifies it as the successor to the agent work in Semantic Kernel and AutoGen; this article evaluates the current framework rather than counting those predecessor APIs as separate contemporary choices.

The inspected Python [`Agent`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_agents.py), [`Workflow`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_workflows/_workflow.py) and [`WorkflowBuilder`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_workflows/_workflow_builder.py) expose both agent execution and workflow composition. The official [workflow capabilities](https://learn.microsoft.com/en-us/agent-framework/workflows/) cover external input, checkpoints and orchestration patterns. For the incident assistant, these provide a place to distinguish an investigative agent from a mandatory review transition.

The hosting boundary deserves a separate decision. Microsoft's [Durable Extension](https://learn.microsoft.com/en-us/agent-framework/hosting/azure-functions) adds durable agent and workflow execution through Durable Task infrastructure, with Azure Functions and self-hosted options. Persistence and recovery therefore depend on the selected integration, storage and workers; they should not be inferred from the core agent definition.

I would evaluate this stack when its application interfaces and deployment options fit the team. The trade-off is integration breadth: verify the exact language, package and hosting path instead of assuming every documented capability is available in every configuration.

### 2.9 CrewAI: separate role collaboration from workflow control

CrewAI expresses collaboration through agents, tasks and crews. Its [crew documentation](https://docs.crewai.com/en/concepts/crews) describes coordinating task execution through a configured process. The inspected [`Crew.kickoff()`](https://github.com/crewAIInc/crewAI/blob/aa5f2acd9b48782b6b6a756bcef24cc54ac0530b/lib/crewai/src/crewai/crew.py) is a concrete entry point into that execution model.

The incident assistant could express investigation and review as separate tasks, each with an expected result and assigned responsibility. This can make specialist responsibilities easy to describe. My concern would be whether each delegation produces additional evidence or merely another model-generated opinion. A reviewer role needs a concrete evidence contract and an independent check to provide meaningful assurance.

CrewAI also exposes Flows for event-driven application logic, branching and state. Its inspected [`Flow`](https://github.com/crewAIInc/crewAI/blob/aa5f2acd9b48782b6b6a756bcef24cc54ac0530b/lib/crewai/src/crewai/flow/flow.py) implementation and [Flow documentation](https://docs.crewai.com/en/concepts/flows) provide a separate surface for controlling how work advances. A flow can coordinate crew execution while application logic determines what follows.

I would inspect flow persistence and resume configuration independently of crew memory. Restoring state does not by itself establish that an interrupted external write cannot happen twice. The appeal is a role/task model connected to explicit flow control; the evaluation burden is to make task outputs, delegation cost and recovery semantics observable.

## 3. Separate conversation memory from execution recovery

All nine stacks can participate in a stateful application. That statement provides almost no guidance until the kind of state is specified.

**Conversation state** contains messages and tool results. **Business state** contains authoritative records such as an approved incident action. **Execution state** records what the runtime has completed and where it can resume. A workspace adds another resource: files may survive even when the worker or conversation does not.

| Approach | State surface to inspect | Recovery question |
| --- | --- | --- |
| OpenAI Agents SDK | Sessions, run results and resumable run state | Which objects are persisted, and who schedules continuation? |
| Claude Agent SDK | Session identity and persisted session data | Can a replacement worker access both the session and its workspace? |
| Codex SDK | Persisted thread history and a separate workspace | Does the restored thread still refer to the intended repository state? |
| Google ADK | Session service, events and state | What survives with the selected backend and resume configuration? |
| Vercel AI SDK | Stored messages; workflow state when using WorkflowAgent | Is only the transcript durable, or is execution durable too? |
| LangChain / LangGraph | Agent state, thread checkpoints and cross-thread stores | What work re-executes when the graph resumes? |
| Pydantic AI | Message history and the selected durable engine | Which model/tool steps are recorded, and can dependencies be reconstructed? |
| Microsoft Agent Framework | Sessions, workflow checkpoints and optional Durable Task integration | Which backing store and hosting model provide recovery? |
| CrewAI | Crew memory, flow state and configured persistence | What state is restored, and which unfinished work can repeat? |

The OpenAI Agents SDK [results and state guide](https://developers.openai.com/api/docs/guides/agents/results) describes result and continuation surfaces. Claude's [session guide](https://code.claude.com/docs/en/agent-sdk/sessions) describes continuing earlier sessions. ADK's [state guide](https://adk.dev/sessions/state/) makes persistence dependent on the selected session service; an in-memory service does not survive a process restart. These are different contracts, even though each is described with the word “session.”

For the incident assistant, I would store the approved action independently of chat history. If approval means “apply patch X to revision Y,” a resumed agent must not reinterpret it as permission to apply a different patch after the branch changes.

Recovery also has an external-effect problem. Suppose a tool updates an incident ticket, then the process crashes before saving its result. Replaying the call may create a duplicate update. A persisted transcript cannot resolve whether the external service committed the first attempt. A workflow checkpoint reduces repeated work but does not automatically provide an atomic transaction across an unrelated service.

The application needs a stable operation identifier, an idempotent endpoint or deduplication record, and a way to reconcile ambiguous outcomes. Vercel's [WorkflowAgent documentation](https://ai-sdk.dev/docs/agents/workflow-agent) explicitly distinguishes completed-step replay from exactly-once effects. LangGraph's [interrupt documentation](https://docs.langchain.com/oss/python/langgraph/interrupts) likewise warns about work that can re-execute around interruption. These are implementation constraints to test, not reasons to reject durable runtimes.

Context compaction solves a different problem again: fitting relevant evidence into the next inference. A summary can preserve the gist while losing an exact revision identifier. Durable authority belongs in typed application records, with the model receiving the subset needed for the current decision.

## 4. Treat tool permissions as a runtime contract

A tool schema describes what arguments look like. It does not establish that the current user may perform the requested action.

For an incident assistant, `restart_service(service_id)` must enforce tenant identity, environment, resource scope and approval status outside model-generated arguments. A well-formed request to restart another customer's production service is still unauthorized.

The frameworks provide different interception points. OpenAI Agents SDK documents [guardrails and human review](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals). Claude Agent SDK exposes a [permission system](https://code.claude.com/docs/en/agent-sdk/permissions) for deciding which tool actions can proceed. Vercel's current [agent-building guide](https://ai-sdk.dev/docs/agents/building-agents) exposes `toolApproval` for the ordinary tool loop; its workflow integration documents a different approval configuration. Those mechanisms need to be connected to application identity and policy.

My preferred approval unit is a concrete operation: tool name, validated arguments, tenant, target revision and expiry. The service should verify those values immediately before execution. A chat message saying “approved” is insufficient unless it is authenticated and bound to the pending operation.

Typed dependencies and structured outputs strengthen the application boundary in a different way. Pydantic AI emphasizes these contracts; other frameworks expose their own schema and validation surfaces. Across stacks, an output that names a valid service can still prescribe the wrong remediation. Validation must check the relationship between evidence, requested action and current business state.

The containment boundary matters as much as the permission callback. A shell-capable agent can invoke a network client through an allowed command. An application function can access credentials even when it never launches a subprocess. Tool admission, operating-system isolation and downstream authorization protect different boundaries.

This distinction changes framework selection. If the application needs repository exploration, a developed filesystem harness may save substantial implementation work. If it only needs three business APIs, deliberately narrow functions can simplify the authority model. The relevant comparison is the amount of executable capability the product actually needs.

## 5. Compare observed behavior on the same workload

Framework features do not establish task success, latency or cost. A useful evaluation must hold enough variables constant to make the comparison interpretable.

For the incident assistant, I would begin with the same evidence corpus, tool contracts, task budget and acceptance criteria. Where model compatibility permits, use the same model. Where a harness is tied to a different model or execution environment, report that difference explicitly and avoid attributing the entire outcome to the SDK.

Provider portability should also be tested as behavior. Two adapters can accept similar tool schemas while differing in supported output modes, parallel calls or native tools. A useful substitution test changes the model and reruns the same acceptance suite; successfully changing a model name is only the first step.

The evaluation should include failure as well as completion:

| Scenario | Evidence to collect |
| --- | --- |
| Routine incident investigation | Correct diagnosis, cited evidence, tool and model-call counts |
| A tool returns a malformed result | Validation outcome and whether the agent invents missing evidence |
| A typed result cites unrelated evidence | Whether domain validation catches a semantically wrong answer |
| A specialist delegates work to another agent | Additional evidence, model-call overhead and final-answer ownership |
| A worker dies after an external update | Duplicate effects, reconciliation and restart behavior |
| Approval arrives after a deployment changes | Whether the stale action is rejected or reauthorized |
| A document includes malicious instructions | Which tool requests reach the authorization boundary |
| The user cancels during a slow command | Worker termination, pending effects and reported task state |

These are proposed experiments, not results obtained for this article.

Telemetry should connect the task, model requests, tool calls, approvals and external operation identifiers. OpenAI documents [SDK tracing and observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability); other runtimes have their own integration surfaces. A useful trace explains the execution without requiring storage of private reasoning. It should also support redaction: incident evidence and tool results may contain credentials or customer data.

Cost comparison should include failed attempts, repeated context, retry behavior and operating overhead. A framework with fewer lines of integration code can still generate more model turns. A more structured workflow can reduce repeated work while increasing implementation and maintenance cost. Both effects need measurement under the intended workload.

## 6. Choose the abstraction that matches the product

My selection would follow the dominant engineering constraint. These are starting points for evaluation, not exclusive capabilities. Several stacks can satisfy the same requirement through different integrations.

| Dominant requirement | Starting point I would evaluate | Main trade-off to verify |
| --- | --- | --- |
| Application-defined tools and agent delegation | OpenAI Agents SDK | Fit between runner semantics and business transitions |
| Repository work with file and command tools | Claude Agent SDK or Codex SDK | Harness assumptions, worker hosting, thread/workspace recovery and isolation |
| Composable agent pipelines with explicit service backends | Google ADK | State ownership, backend configuration and language parity |
| TypeScript product with interactive model/tool streaming | Vercel AI SDK | In-memory loop versus durable workflow integration |
| A ready-made agent loop with ecosystem middleware | LangChain agents | Whether high-level customization is enough before introducing a custom graph |
| Explicit branching, checkpoints and delayed human decisions | LangGraph | Graph complexity and replay-safe external operations |
| Typed Python dependencies and structured application results | Pydantic AI | Schema validity versus semantic correctness; chosen durable engine |
| Agent/workflow composition with Microsoft application integrations | Microsoft Agent Framework | Language/package parity and the hosting/durability boundary |
| Role-based specialist tasks coordinated by an explicit flow | CrewAI | Delegation overhead, task-output quality and persistence semantics |

These choices can coexist. A TypeScript UI can stream events from a separate agent worker. A durable workflow can call a model client for bounded decisions. A repository harness can be one specialized stage inside a broader business process. Combining them is useful when ownership remains explicit; overlapping schedulers and competing session stores make failures harder to explain.

Three trade-offs remain central. More harness behavior reduces integration work while increasing dependence on the harness's execution model. More explicit workflow structure improves inspectability while requiring the team to maintain that structure. Broader provider abstraction can ease model substitution while leaving differences in tools, state and output semantics unresolved.

I would therefore prototype the difficult lifecycle first: interrupt a run, replace its worker, deliver a delayed approval, and inspect the external effects. A framework that makes that sequence understandable is a stronger foundation than one that only makes the first successful response easy to produce.

The open question is how much of the product's execution model should become a dependency. The answer comes from the application's state, authority and recovery requirements. Model access is the entry point; the runtime contract determines what happens afterward.
