# Agent SDK comparison research

Checked on October 15, 2026. The published article is
`content/posts/ai-agents/comparing-agent-sdks.md`.

## Scope and evidence

- Interpret the requested "Cloud Agent SDK" as Claude Agent SDK, based on the named framework family.
- Scope the comparison to agent-development SDKs and frameworks. Exclude general model API client libraries.
- Include Codex SDK beside Claude Agent SDK as an integration for an existing coding harness.
- Add Pydantic AI, Microsoft Agent Framework and CrewAI to cover typed application contracts, agent/workflow integration and role/task orchestration.
- Explain LangChain high-level agents and LangGraph workflow control as related layers.
- Compare core execution separately from optional durable-runtime integrations. Schema validity, semantic validation and provider substitution are explicit evaluation concerns.
- Use current official documentation and inspected repository files. No performance benchmark or paid API execution was performed.
- The TypeScript example targets the inspected Vercel interface (`isStepCount`); it is configuration, not a standalone tested application.

## Inspected source snapshots

| Repository | Commit | Inspected files |
| --- | --- | --- |
| openai/openai-agents-python | `17d994bd3045dae71acda676a4ef4762ed8ecd94` | [`src/agents/run.py`](https://github.com/openai/openai-agents-python/blob/17d994bd3045dae71acda676a4ef4762ed8ecd94/src/agents/run.py) |
| anthropics/claude-agent-sdk-python | `c10b52c674e5c709c7263544b38e8a5dad10e01d` | [`src/claude_agent_sdk/query.py`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/query.py), [`src/claude_agent_sdk/_internal/client.py`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/_internal/client.py) |
| google/adk-python | `377306e43c5914de3f64f72526173b0732b0c7d8` | [`src/google/adk/runners.py`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/runners.py), [`src/google/adk/agents/sequential_agent.py`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/agents/sequential_agent.py) |
| vercel/ai | `a56e32017d800a9e2f4fc0ccc9a09e63911a9420` | [`packages/ai/src/agent/tool-loop-agent.ts`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/agent/tool-loop-agent.ts), [`packages/ai/src/generate-text/stop-condition.ts`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/generate-text/stop-condition.ts) |
| langchain-ai/langgraph | `5eb60aef6f3837a1fb632a8abc6cf8e8da81785b` | [`libs/langgraph/langgraph/graph/state.py`](https://github.com/langchain-ai/langgraph/blob/5eb60aef6f3837a1fb632a8abc6cf8e8da81785b/libs/langgraph/langgraph/graph/state.py) |

| pydantic/pydantic-ai | `9ac853bb84ffed7ba0edccb026c5b4f3228c1de7` | [`pydantic_ai_slim/pydantic_ai/agent/__init__.py`](https://github.com/pydantic/pydantic-ai/blob/9ac853bb84ffed7ba0edccb026c5b4f3228c1de7/pydantic_ai_slim/pydantic_ai/agent/__init__.py), [`pydantic_ai_slim/pydantic_ai/durable_exec/__init__.py`](https://github.com/pydantic/pydantic-ai/blob/9ac853bb84ffed7ba0edccb026c5b4f3228c1de7/pydantic_ai_slim/pydantic_ai/durable_exec/__init__.py) |
| microsoft/agent-framework | `0cc867d966daa2891c191204b1c9b7a4ff0e7619` | [`python/packages/core/agent_framework/_agents.py`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_agents.py), [`python/packages/core/agent_framework/_workflows/_workflow.py`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_workflows/_workflow.py), [`python/packages/core/agent_framework/_workflows/_workflow_builder.py`](https://github.com/microsoft/agent-framework/blob/0cc867d966daa2891c191204b1c9b7a4ff0e7619/python/packages/core/agent_framework/_workflows/_workflow_builder.py) |
| crewAIInc/crewAI | `aa5f2acd9b48782b6b6a756bcef24cc54ac0530b` | [`lib/crewai/src/crewai/crew.py`](https://github.com/crewAIInc/crewAI/blob/aa5f2acd9b48782b6b6a756bcef24cc54ac0530b/lib/crewai/src/crewai/crew.py), [`lib/crewai/src/crewai/flow/flow.py`](https://github.com/crewAIInc/crewAI/blob/aa5f2acd9b48782b6b6a756bcef24cc54ac0530b/lib/crewai/src/crewai/flow/flow.py) |
| langchain-ai/langchain | `84ba23c414eca6bf6f6ebd1394a3006378113540` | [`libs/langchain_v1/langchain/agents/factory.py`](https://github.com/langchain-ai/langchain/blob/84ba23c414eca6bf6f6ebd1394a3006378113540/libs/langchain_v1/langchain/agents/factory.py) |
| openai/codex | `ac9db7d8b556c7c34a73793aed34b1f980a4fe39` | [`sdk/typescript/src/codex.ts`](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/codex.ts), [`sdk/typescript/src/thread.ts`](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/thread.ts), [`sdk/typescript/src/exec.ts`](https://github.com/openai/codex/blob/ac9db7d8b556c7c34a73793aed34b1f980a4fe39/sdk/typescript/src/exec.ts) |

## Primary documentation

- [SDK overview](https://developers.openai.com/api/docs/guides/agents/sdk)
- [orchestration guide](https://developers.openai.com/api/docs/guides/agents/orchestration)
- [overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [hosting guidance](https://code.claude.com/docs/en/agent-sdk/hosting)
- [SDK documentation](https://learn.chatgpt.com/docs/codex-sdk)
- [Codex app-server](https://developers.openai.com/blog/codex-as-a-platform)
- [runtime documentation](https://adk.dev/runtime/)
- [workflow-agent documentation](https://adk.dev/agents/workflow-agents/)
- [loop-control documentation](https://ai-sdk.dev/docs/agents/loop-control)
- [chatbot documentation](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot)
- [`WorkflowAgent`](https://ai-sdk.dev/docs/agents/workflow-agent)
- [agent documentation](https://docs.langchain.com/oss/python/langchain/agents)
- [persistence documentation](https://docs.langchain.com/oss/python/langgraph/persistence)
- [dependency guide](https://pydantic.dev/docs/ai/core-concepts/dependencies/)
- [output guide](https://pydantic.dev/docs/ai/core-concepts/output/)
- [durable-execution guide](https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/)
- [overview](https://learn.microsoft.com/en-us/agent-framework/overview/)
- [workflow capabilities](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Durable Extension](https://learn.microsoft.com/en-us/agent-framework/hosting/azure-functions)
- [crew documentation](https://docs.crewai.com/en/concepts/crews)
- [Flow documentation](https://docs.crewai.com/en/concepts/flows)
- [results and state guide](https://developers.openai.com/api/docs/guides/agents/results)
- [session guide](https://code.claude.com/docs/en/agent-sdk/sessions)
- [state guide](https://adk.dev/sessions/state/)
- [WorkflowAgent documentation](https://ai-sdk.dev/docs/agents/workflow-agent)
- [interrupt documentation](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [guardrails and human review](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals)
- [permission system](https://code.claude.com/docs/en/agent-sdk/permissions)
- [agent-building guide](https://ai-sdk.dev/docs/agents/building-agents)
- [SDK tracing and observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability)

## Content consolidation

- Remove all three AI article pairs and index registrations from Knowledge Base.
- Migrate the English local hybrid retrieval and Physical AI articles to Blog as Markdown; retain their original publication dates and mark the migration date with `lastmod`.
- Do not duplicate the general AI-agent concepts primer. Existing reliability and framework architecture articles already cover much of it; the new comparison develops control-loop, state, tool and evaluation trade-offs.
- Remove Knowledge Base AI module configuration and obsolete AI samples in design demos.
- Correct consistent hashing from September 9, 2026 to September 9, 2021; keep the existing slug and alias.

## Validation

- Knowledge Base: run its existing unittest suite and rebuild `dist`.
- Blog: production Hugo build, new-page/TOC/link checks, publication date verification, and local preview.
- Build outputs remain ignored. Publish source changes through the repository’s normal GitHub Actions deployment.
