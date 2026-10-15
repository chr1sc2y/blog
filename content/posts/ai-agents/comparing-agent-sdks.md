---
title: "Comparing Agent SDKs: Who Owns the Loop, State, and Runtime?"
date: 2026-10-15T10:00:00+08:00
draft: false
categories: ["AI Agents"]
description: "An architectural comparison of the OpenAI client SDK, OpenAI Agents SDK, Claude Agent SDK, Google ADK, Vercel AI SDK, and LangGraph through execution, state, permissions, and recovery."
---

Agent development frameworks increasingly expose similar vocabulary: tools, sessions, memory, handoffs, and streaming. That vocabulary hides substantial architectural differences. Calling a model through the OpenAI client SDK, embedding Claude Code through the Claude Agent SDK, and compiling a stateful LangGraph workflow place different parts of the system under framework control. Comparing them through a list of supported features misses the decision that will determine most of the integration work: which execution responsibilities the application keeps, and which it delegates.

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

This comparison covers the OpenAI client SDK and OpenAI Agents SDK separately, then Claude Agent SDK, Google ADK, and Vercel AI SDK. LangGraph provides an additional reference for explicit workflow state and interruption. The scope is architectural behavior documented and inspected on October 15, 2026. Source links point to inspected commits; documentation links can evolve. These are qualitative engineering judgments, not benchmark results or adoption rankings. Language implementations within a project may expose different capabilities.

## 1. Establish the execution boundary

Consider an incident assistant. It reads telemetry, inspects a repository, proposes a patch, requests approval, and applies a change. Every implementation needs model inference, but that is only one step. Someone must schedule tool calls, preserve evidence, decide whether another model turn is necessary, authenticate approvals, and recover after a worker disappears.

The first comparison is therefore about ownership:

| Integration | Primary abstraction | Main execution responsibility | Application responsibility to examine |
| --- | --- | --- | --- |
| OpenAI client SDK | API request and response | Typed access to model and platform APIs | Custom tool loop and business workflow, unless selecting a managed API |
| OpenAI Agents SDK | Agent and runner | Repeated model/tool execution and agent handoffs | Deployment, tool implementations, storage and approval policy |
| Claude Agent SDK | Programmable Claude Code session | An existing harness with tools and context management | Process hosting, workspace lifecycle and isolation |
| Google ADK | Agents, runner, events and services | Agent execution with structured workflow components | Service backends, state design and tool boundaries |
| Vercel AI SDK | Model/provider interface and agent loop | TypeScript generation, tool execution and product streaming | Choice of in-memory or workflow execution and persistence |
| LangGraph | State graph and checkpointed execution | Explicit transitions and persisted workflow progress | Node logic, checkpointer backend and external-effect semantics |

The table describes default integration surfaces rather than hard capability ceilings. OpenAI's current documentation also distinguishes the hosted Agents API from the application-operated Agents SDK. The same client library can access a managed service without becoming an application-side orchestration framework. Likewise, Anthropic distinguishes its client SDK, Agent SDK, and Managed Agents. Both distinctions matter when reviewing an architecture diagram: a library import alone does not reveal where the agent runs. See the official [OpenAI runtime comparison](https://developers.openai.com/api/docs/guides/agents) and [Claude integration comparison](https://code.claude.com/docs/en/agent-sdk/overview).

I would start a design review by drawing this boundary before discussing model quality. Moving the loop from application code into a framework changes debugging and lifecycle responsibilities even when the underlying model remains identical.

## 2. Follow one tool call through each architecture

### 2.1 OpenAI client SDK: explicit application control

The ordinary OpenAI client SDK exposes API resources. In the inspected Python implementation, [`Responses` and `AsyncResponses`](https://github.com/openai/openai-python/blob/95dbee9ee406587053a0a21296eb82418073f2aa/src/openai/resources/responses/responses.py) implement the Responses resource. That is a useful place to see the distinction between transporting a request and orchestrating application work.

For a custom function tool, the model can produce a call; the application executes the function and returns its result to the API. Hosted tools have different execution ownership. The official [function-calling guide](https://developers.openai.com/api/docs/guides/function-calling) documents this exchange. Describing every Responses request as a single isolated inference would be misleading, because the API also offers hosted tools and conversation-state mechanisms.

An application-owned loop can be understood through the following deliberately incomplete pseudocode:

```text
load authorized conversation and task state
repeat within a task budget:
    request a response with the currently allowed tools
    persist the response and any proposed calls
    if no tool work remains:
        verify the required result and finish
    for each proposed application tool call:
        validate arguments and authorize the exact action
        execute with timeout and idempotency protection
        persist the outcome and return it to the conversation
```

This is a control-flow sketch, not runnable SDK code. It omits streaming, cancellation, provider-specific item formats, concurrency, and approval suspension.

The attraction is direct control. A product with an established workflow engine can use a model as one decision step without introducing a second scheduler. The cost is that the team owns every omitted lifecycle detail. A small successful demonstration does not establish that the resulting loop is operationally complete.

### 2.2 OpenAI Agents SDK: delegate the recurring loop

The OpenAI Agents SDK moves recurring agent execution into a runner. An agent definition groups instructions, tools and related behavior; the runner handles repeated execution and handoffs. The inspected [`Runner.run` implementation and its underlying runner](https://github.com/openai/openai-agents-python/blob/17d994bd3045dae71acda676a4ef4762ed8ecd94/src/agents/run.py) make the control flow explicit: final output ends a run, a handoff changes the active agent, and tool calls can lead to another turn. See also the [SDK overview](https://developers.openai.com/api/docs/guides/agents/sdk).

A handoff and a specialist exposed as a tool have different ownership semantics. With a handoff, the active conversational role changes. With a specialist tool, the caller can retain responsibility for assembling the answer. This distinction is more useful than simply counting supported agents: it determines which component must enforce the final response contract. The official [orchestration guide](https://developers.openai.com/api/docs/guides/agents/orchestration) describes these patterns.

My preference would be to use the runner when its execution contract matches the product. Reimplementing a standard tool loop adds maintenance without much differentiation. Conversely, forcing a transaction-oriented workflow into model-directed handoffs can obscure the business states that operations teams need to inspect.

### 2.3 Claude Agent SDK: embed a developed harness

Claude Agent SDK starts from another point in the stack: it exposes the agent loop, tools and context management used by Claude Code. Python and TypeScript applications can operate that harness programmatically, including built-in file and command tools, sessions, hooks and MCP integrations. The [overview](https://code.claude.com/docs/en/agent-sdk/overview) explicitly distinguishes it from direct client-SDK access.

In the inspected Python source, [`query()`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/query.py) delegates to [`InternalClient.process_query()`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/_internal/client.py), which selects a transport and uses the subprocess CLI transport by default. The Python wrapper is therefore not the complete implementation of the underlying agent. Some behavior lives across a process boundary.

For repository maintenance, that is an attractive starting point: file inspection, editing and command execution already have a coherent interaction model. For a narrow customer-service product, those capabilities may introduce more runtime surface than the application needs.

The engineering question becomes whether the product wants this harness's working model. Its host must account for workspaces, subprocesses, cancellation and tool permissions. Those responsibilities belong in deployment design, even if the integration begins with a short `query()` call. Anthropic's [hosting guidance](https://code.claude.com/docs/en/agent-sdk/hosting) addresses the operating environment separately from the library API.

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

### 2.6 LangGraph: make transitions inspectable

LangGraph provides a useful comparison when the workflow itself is the product's central state machine. Its inspected [`StateGraph` and `compile()`](https://github.com/langchain-ai/langgraph/blob/5eb60aef6f3837a1fb632a8abc6cf8e8da81785b/libs/langgraph/langgraph/graph/state.py) expose state updates, graph construction and checkpointer configuration. The graph structure gives application code a place to represent branching and joins explicitly.

That can be valuable for an incident workflow with mandatory review and remediation stages. It also creates work: the team must define state, reducers and transition conditions. Graph structure earns its complexity when operators need those states to remain meaningful across failures and human intervention. The [persistence documentation](https://docs.langchain.com/oss/python/langgraph/persistence) distinguishes checkpointed thread state from stores used across threads.

## 3. Separate conversation memory from execution recovery

All six approaches can participate in a stateful application. That statement provides almost no guidance until the kind of state is specified.

**Conversation state** contains messages and tool results. **Business state** contains authoritative records such as an approved incident action. **Execution state** records what the runtime has completed and where it can resume. A workspace adds another resource: files may survive even when the worker or conversation does not.

| Approach | State surface to inspect | Recovery question |
| --- | --- | --- |
| OpenAI client SDK | Application records and API conversation mechanisms | Which pending calls and outcomes can the application reconstruct? |
| OpenAI Agents SDK | Sessions, run results and resumable run state | Which objects are persisted, and who schedules continuation? |
| Claude Agent SDK | Session identity and persisted session data | Can a replacement worker access both the session and its workspace? |
| Google ADK | Session service, events and state | What survives with the selected backend and resume configuration? |
| Vercel AI SDK | Stored messages; workflow state when using WorkflowAgent | Is only the transcript durable, or is execution durable too? |
| LangGraph | Thread checkpoints and cross-thread stores | What work re-executes when the graph resumes? |

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

The containment boundary matters as much as the permission callback. A shell-capable agent can invoke a network client through an allowed command. An application function can access credentials even when it never launches a subprocess. Tool admission, operating-system isolation and downstream authorization protect different boundaries.

This distinction changes framework selection. If the application needs repository exploration, a developed filesystem harness may save substantial implementation work. If it only needs three business APIs, deliberately narrow functions can simplify the authority model. The relevant comparison is the amount of executable capability the product actually needs.

## 5. Compare observed behavior on the same workload

Framework features do not establish task success, latency or cost. A useful evaluation must hold enough variables constant to make the comparison interpretable.

For the incident assistant, I would begin with the same evidence corpus, tool contracts, task budget and acceptance criteria. Where model compatibility permits, use the same model. Where a harness is tied to a different model or execution environment, report that difference explicitly and avoid attributing the entire outcome to the SDK.

The evaluation should include failure as well as completion:

| Scenario | Evidence to collect |
| --- | --- |
| Routine incident investigation | Correct diagnosis, cited evidence, tool and model-call counts |
| A tool returns a malformed result | Validation outcome and whether the agent invents missing evidence |
| A worker dies after an external update | Duplicate effects, reconciliation and restart behavior |
| Approval arrives after a deployment changes | Whether the stale action is rejected or reauthorized |
| A document includes malicious instructions | Which tool requests reach the authorization boundary |
| The user cancels during a slow command | Worker termination, pending effects and reported task state |

These are proposed experiments, not results obtained for this article.

Telemetry should connect the task, model requests, tool calls, approvals and external operation identifiers. OpenAI documents [SDK tracing and observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability); other runtimes have their own integration surfaces. A useful trace explains the execution without requiring storage of private reasoning. It should also support redaction: incident evidence and tool results may contain credentials or customer data.

Cost comparison should include failed attempts, repeated context, retry behavior and operating overhead. A framework with fewer lines of integration code can still generate more model turns. A more structured workflow can reduce repeated work while increasing implementation and maintenance cost. Both effects need measurement under the intended workload.

## 6. Choose the abstraction that matches the product

My selection would follow the dominant engineering constraint.

| Dominant requirement | Starting point I would evaluate | Main trade-off to verify |
| --- | --- | --- |
| Existing workflow engine; precise control over model calls | OpenAI client SDK with an explicit application loop | Ownership of retries, continuation and tool scheduling |
| Application-defined tools and agent delegation | OpenAI Agents SDK | Fit between runner semantics and business transitions |
| Repository work with file and command tools | Claude Agent SDK | Harness assumptions, worker hosting and isolation |
| Composable agent pipelines with explicit service backends | Google ADK | State ownership, backend configuration and language parity |
| TypeScript product with interactive model/tool streaming | Vercel AI SDK | In-memory loop versus durable workflow integration |
| Explicit branching, checkpoints and delayed human decisions | LangGraph | Graph complexity and replay-safe external operations |

These choices can coexist. A TypeScript UI can stream events from a separate agent worker. A durable workflow can call a model client for bounded decisions. A repository harness can be one specialized stage inside a broader business process. Combining them is useful when ownership remains explicit; overlapping schedulers and competing session stores make failures harder to explain.

Three trade-offs remain central. More harness behavior reduces integration work while increasing dependence on the harness's execution model. More explicit workflow structure improves inspectability while requiring the team to maintain that structure. Broader provider abstraction can ease model substitution while leaving differences in tools, state and output semantics unresolved.

I would therefore prototype the difficult lifecycle first: interrupt a run, replace its worker, deliver a delayed approval, and inspect the external effects. A framework that makes that sequence understandable is a stronger foundation than one that only makes the first successful response easy to produce.

The open question is how much of the product's execution model should become a dependency. The answer comes from the application's state, authority and recovery requirements. Model access is the entry point; the runtime contract determines what happens afterward.
