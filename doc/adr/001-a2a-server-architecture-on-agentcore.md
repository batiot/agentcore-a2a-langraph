# ADR-001: A2A Server Architecture on AgentCore Runtime

**Date:** 2026-03-30
**Status:** Accepted
**Context:** a2a-haiku-agent proposal validation against AgentCore documentation and samples

## Context

We are deploying a LangGraph-based A2A agent on AWS AgentCore Runtime. AgentCore provides two deployment paths:

1. **Starter toolkit CLI** (`agentcore configure` + `agentcore launch`) with `BedrockAgentCoreApp` wrapper — the officially documented and recommended approach.
2. **Raw A2A server** using `a2a-sdk` (`A2AStarletteApplication` + custom `AgentExecutor`) running on uvicorn — the pattern used by multi-agent samples in `agentcore-samples`.

The choice affects streaming capability, deployment method, dependency surface, and control over the A2A protocol lifecycle.

## Decision

**Use the raw A2A server pattern (option 2) with a custom CloudFormation template.**

Specifically:
- `A2AStarletteApplication` + `DefaultRequestHandler` + `InMemoryTaskStore` + custom `HaikuAgentExecutor(AgentExecutor)`
- Uvicorn on `0.0.0.0:9000` at root `/`
- Agent card at `/.well-known/agent-card.json` with URL from `AGENTCORE_RUNTIME_URL` env var
- `/ping` health endpoint
- CloudFormation deployment (not `agentcore launch`)
- No dependency on `bedrock-agentcore` SDK or `BedrockAgentCoreApp`

## Rationale

### Why raw A2A over BedrockAgentCoreApp

The `BedrockAgentCoreApp` + `@app.entrypoint` pattern is **synchronous only**. The official LangGraph example (`langgraph_agent_web_search.py`) uses `graph.invoke()` and returns a complete `{"result": ...}` dict. There is no streaming path — the entrypoint function returns a value, and the framework handles the response.

Our core requirement is **true token-level streaming** via `graph.astream_events()` through the A2A protocol's `message/stream` method. This requires direct control over the `TaskUpdater` lifecycle: emitting `working` status updates per token chunk, then `add_artifact()` + `complete()` at the end. The `BedrockAgentCoreApp` abstraction does not expose this streaming lifecycle.

The raw A2A pattern is proven in production-like samples:
- `agentcore-samples/02-use-cases/A2A-multi-agent-incident-response/monitoring_strands_agent/main.py`
- `agentcore-samples/02-use-cases/A2A-multi-agent-incident-response/web_search_openai_agents/main.py`

Both use the exact `A2AStarletteApplication` → `DefaultRequestHandler` → custom `AgentExecutor` stack.

### Why custom CloudFormation over agentcore CLI

- The `agentcore launch` CLI uses `direct_code_deploy` which assumes `BedrockAgentCoreApp` wrapper code. Our raw A2A server entry point (`uvicorn` running a Starlette app) doesn't conform to this pattern.
- CloudFormation gives us version-controlled, repeatable infrastructure with explicit control over IAM roles, ECR, and CodeBuild.
- The `agentcore` CLI is a convenience layer — it creates the same underlying resources (ECR, CodeBuild, IAM roles, `AWS::BedrockAgentCore::Runtime`). We replicate this directly.

### Why no bedrock-agentcore SDK dependency

The `bedrock-agentcore` package provides `BedrockAgentCoreApp` (unused), identity helpers (not needed for pre-generated JWT), and OpenTelemetry integration (out of scope). Adding it would introduce unused code and couple us to the starter toolkit's assumptions. If observability or identity features are needed later, the dependency can be added incrementally.

## Consequences

### What we gain

- Full control over the A2A streaming lifecycle (token-level SSE)
- Alignment with the proven multi-agent sample patterns
- Infrastructure-as-code via CloudFormation
- Minimal dependency surface (`a2a-sdk`, `uvicorn`, `langgraph`, `langchain-aws`)

### What we give up

- Automatic OpenTelemetry instrumentation from `BedrockAgentCoreApp`
- `agentcore invoke` / `agentcore status` / `agentcore destroy` CLI convenience
- Potential future features in the starter toolkit (auto-scaling hints, memory integration hooks)

### Risks

- **R1: `AWS::BedrockAgentCore::Runtime` CFn resource evolving** — The resource's `AuthorizerConfiguration` property is not well-documented. OAuth setup may require a post-deploy CLI/API step.
- **R2: `a2a-sdk` API instability** — At v0.3.x, breaking changes possible. Mitigated by pinning the exact version.
- **R3: ~~Missing runtime conventions~~** — Resolved: `AGENTCORE_RUNTIME_URL` for agent card URL and `/ping` endpoint are specified in the openspec change artifacts (design D3, a2a-server spec, tasks 3.1/3.5).

## Compliance with AgentCore Runtime Conventions

Items verified against current AgentCore docs and samples:

| Convention | Status | Notes |
|---|---|---|
| Port 9000 | Compliant | Required for A2A protocol |
| Root path `/` | Compliant | A2A mounts at root (vs `/invocations` for HTTP) |
| Agent card at `/.well-known/agent-card.json` | Compliant | Built into `A2AStarletteApplication` |
| `AGENTCORE_RUNTIME_URL` env var for agent card URL | Compliant | Specified in design D3 and a2a-server spec |
| `/ping` health endpoint | Compliant | Specified in design D3 and a2a-server spec |
| `streaming: true` in AgentCard capabilities | Compliant | Required for `message/stream` support |
| JSON-RPC 2.0 protocol | Compliant | Handled by `a2a-sdk` |
| ARM64 container | Compliant | CodeBuild with `amazonlinux-aarch64-standard:3.0` |

## References

- [AgentCore A2A deployment guide](https://aws.github.io/bedrock-agentcore-starter-toolkit/user-guide/runtime/a2a.md)
- [LangGraph on AgentCore example](https://aws.github.io/bedrock-agentcore-starter-toolkit/examples/integrations/agentic-frameworks/langgraph/langgraph-agent-readme.md)
- [AgentCore Identity quickstart](https://aws.github.io/bedrock-agentcore-starter-toolkit/user-guide/identity/quickstart.md)
- `agentcore-samples/02-use-cases/A2A-multi-agent-incident-response/` (raw A2A pattern reference)
