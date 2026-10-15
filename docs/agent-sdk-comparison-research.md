# Agent SDK comparison research

Checked on October 15, 2026. The published article is
`content/posts/ai-agents/comparing-agent-sdks.md`.

## Scope and evidence

- Interpret the requested "Cloud Agent SDK" as Claude Agent SDK, based on the named framework family.
- Compare the OpenAI API client separately from the OpenAI Agents SDK.
- Include LangGraph as a workflow persistence reference.
- Use current official documentation and inspected repository files. No performance benchmark or paid API execution was performed.
- The TypeScript example targets the inspected Vercel interface (`isStepCount`); it is configuration, not a standalone tested application.

## Inspected source snapshots

| Repository | Commit | Inspected files |
| --- | --- | --- |
| openai/openai-python | `95dbee9ee406587053a0a21296eb82418073f2aa` | [`src/openai/resources/responses/responses.py`](https://github.com/openai/openai-python/blob/95dbee9ee406587053a0a21296eb82418073f2aa/src/openai/resources/responses/responses.py) |
| openai/openai-agents-python | `17d994bd3045dae71acda676a4ef4762ed8ecd94` | [`src/agents/run.py`](https://github.com/openai/openai-agents-python/blob/17d994bd3045dae71acda676a4ef4762ed8ecd94/src/agents/run.py) |
| anthropics/claude-agent-sdk-python | `c10b52c674e5c709c7263544b38e8a5dad10e01d` | [`src/claude_agent_sdk/query.py`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/query.py), [`src/claude_agent_sdk/_internal/client.py`](https://github.com/anthropics/claude-agent-sdk-python/blob/c10b52c674e5c709c7263544b38e8a5dad10e01d/src/claude_agent_sdk/_internal/client.py) |
| google/adk-python | `377306e43c5914de3f64f72526173b0732b0c7d8` | [`src/google/adk/runners.py`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/runners.py), [`src/google/adk/agents/sequential_agent.py`](https://github.com/google/adk-python/blob/377306e43c5914de3f64f72526173b0732b0c7d8/src/google/adk/agents/sequential_agent.py) |
| vercel/ai | `a56e32017d800a9e2f4fc0ccc9a09e63911a9420` | [`packages/ai/src/agent/tool-loop-agent.ts`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/agent/tool-loop-agent.ts), [`packages/ai/src/generate-text/stop-condition.ts`](https://github.com/vercel/ai/blob/a56e32017d800a9e2f4fc0ccc9a09e63911a9420/packages/ai/src/generate-text/stop-condition.ts) |
| langchain-ai/langgraph | `5eb60aef6f3837a1fb632a8abc6cf8e8da81785b` | [`libs/langgraph/langgraph/graph/state.py`](https://github.com/langchain-ai/langgraph/blob/5eb60aef6f3837a1fb632a8abc6cf8e8da81785b/libs/langgraph/langgraph/graph/state.py) |

## Primary documentation

- [OpenAI runtime comparison](https://developers.openai.com/api/docs/guides/agents)
- [Claude integration comparison](https://code.claude.com/docs/en/agent-sdk/overview)
- [function-calling guide](https://developers.openai.com/api/docs/guides/function-calling)
- [SDK overview](https://developers.openai.com/api/docs/guides/agents/sdk)
- [orchestration guide](https://developers.openai.com/api/docs/guides/agents/orchestration)
- [overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [hosting guidance](https://code.claude.com/docs/en/agent-sdk/hosting)
- [runtime documentation](https://adk.dev/runtime/)
- [workflow-agent documentation](https://adk.dev/agents/workflow-agents/)
- [loop-control documentation](https://ai-sdk.dev/docs/agents/loop-control)
- [chatbot documentation](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot)
- [`WorkflowAgent`](https://ai-sdk.dev/docs/agents/workflow-agent)
- [persistence documentation](https://docs.langchain.com/oss/python/langgraph/persistence)
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
