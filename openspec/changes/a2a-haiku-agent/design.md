## Context

Greenfield project. No existing code — workspace contains only `.github/` and `openspec/`.

The goal is a minimal A2A agent on AWS AgentCore that proves the full vertical: LangGraph agent → A2A protocol → streaming → cloud deployment → OAuth security. The agent's sole capability is generating haiku poems, keeping domain logic trivial so the focus stays on infrastructure and protocol integration.

Key constraints:
- Must use LangGraph (not StrandAgents) and raw `a2a-sdk` (not AgentCore starter toolkit CLI)
- Must stream tokens, not complete responses
- Must deploy via CloudFormation to AgentCore Runtime
- Must secure with EntraID OIDC (not Cognito)
- Region: `eu-west-3`

## Goals / Non-Goals

**Goals:**
- Working LangGraph haiku agent with true token-level streaming over A2A protocol
- Local Chainlit client that displays streamed haiku responses
- One-command CloudFormation deployment and teardown
- OAuth 2.0 bearer token security via EntraID OIDC on the AgentCore endpoint
- Clean, minimal codebase that serves as a reference pattern

**Non-Goals:**
- Multi-turn conversation or memory persistence (single-shot haiku generation)
- Local authentication middleware (pre-generated JWT used directly)
- Production-grade error handling or retry logic
- CI/CD pipeline (manual deployment via CloudFormation)
- Tool calling or multi-agent orchestration

## Decisions

### D1: LangGraph single-node StateGraph with `astream_events`

Use a single-node `StateGraph` with a `generate_haiku` node that calls `ChatBedrockConverse`. Stream tokens via `graph.astream_events(input, version="v2")` filtering for `on_chat_model_stream` events.

**Why over `astream`**: `astream` yields state updates per node, not per token. `astream_events` exposes individual LLM token events, which is required for true streaming through the A2A protocol.

**Why single node**: The agent does one thing — generate a haiku. A multi-node graph adds complexity with no benefit. The pattern extends naturally if nodes are added later.

### D2: Custom AgentExecutor with EventQueue/TaskUpdater

Implement `HaikuAgentExecutor(AgentExecutor)` from `a2a-sdk` with `execute()` and `cancel()` methods. Use `TaskUpdater` to emit `working` status updates with each streamed token, then `add_artifact()` with the complete haiku and `complete()` at the end.

**Why over wrapping a higher-level server class**: The `a2a-sdk` `AgentExecutor` + `DefaultRequestHandler` pattern gives full control over the streaming lifecycle, matching the proven pattern from the agentcore-samples monitoring agent. There is no simpler abstraction available that supports streaming.

### D3: A2AStarletteApplication on port 9000

Use `A2AStarletteApplication` from `a2a-sdk` with an `AgentCard` declaring `streaming: true`. Run via `uvicorn` on port `0.0.0.0:9000` (AgentCore Runtime convention). Agent card served at `/.well-known/agent-card.json`.

The `AgentCard.url` SHALL be set from the `AGENTCORE_RUNTIME_URL` environment variable (falling back to `http://localhost:9000/` for local development). AgentCore Runtime injects this variable at deploy time with the public-facing endpoint URL — the agent card must reflect it so that clients connecting via the runtime proxy resolve the correct base URL.

The Starlette app SHALL expose a `/ping` GET endpoint returning `{"status": "healthy"}`. All AgentCore A2A samples include this endpoint; it serves as a lightweight health check for the runtime.

**Why port 9000**: AgentCore Runtime expects the agent process to listen on port 9000. This is the documented convention across all agentcore-samples.

### D4: CloudFormation with inline CodeBuild

Single CloudFormation template creating: ECR repo, CodeBuild project (ARM64, `aws/codebuild/amazonlinux2-aarch64-standard:3.0`), IAM roles (agent execution, CodeBuild, Lambda custom resource), Lambda function to trigger CodeBuild, and `AWS::BedrockAgentCore::Runtime` resource.

CodeBuild uses an **inline buildspec** that generates the Dockerfile and copies source from the repo. This avoids needing a pre-built Docker image before stack creation.

**Why inline buildspec over external**: Keeps everything in one template file. The haiku agent is small enough that the inline approach works without hitting CloudFormation size limits.

**Why ARM64**: Cost-effective for a lightweight Python agent. No native dependencies that would break on ARM.

### D5: EntraID OIDC with pre-generated JWT

AgentCore Runtime configured with OAuth authorizer pointing to EntraID's OIDC discovery endpoint (`https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration`). The Chainlit client passes a pre-generated JWT bearer token (loaded from `A2A_BEARER_TOKEN` env var) in the `Authorization` header.

**Why pre-generated JWT over MSAL flow**: Simplifies the local client — no MSAL dependency, no device code flow, no token refresh. For an experimental project, a long-lived token (or manually refreshed one) is sufficient.

**Why EntraID over Cognito**: User's existing identity provider. AgentCore supports external OIDC providers via its Identity service.

### D6: Project structure

```
src/
  haiku_agent/
    __init__.py
    graph.py          # LangGraph StateGraph definition
    executor.py       # A2A AgentExecutor implementation
    agent_card.py     # AgentCard configuration
    main.py           # Uvicorn entrypoint
client/
  chainlit_app.py     # Chainlit A2A client
  .chainlit/          # Chainlit config
infra/
  template.yaml       # CloudFormation template
  deploy.sh           # Deploy script (create stack, wait, output endpoint)
  destroy.sh          # Cleanup script (delete stack, ECR images)
Dockerfile
pyproject.toml
```

**Why `src/` layout**: Standard Python packaging convention. Avoids import confusion. Works with `uv` build system.

## Risks / Trade-offs

**[R1] CloudFormation OAuth authorizer configuration undocumented** → The `AWS::BedrockAgentCore::Runtime` resource's `AuthorizerConfiguration` property is not well-documented in CloudFormation. May need a post-deploy AWS CLI step or custom resource to configure the OIDC authorizer. Mitigation: deploy without auth first, add auth as a separate step if CFn property doesn't work.

**[R2] Pre-generated JWT expiration** → EntraID tokens expire (default 1 hour). For demo/testing, the user must manually refresh the token. Mitigation: document the token generation command; keep token refresh as a future enhancement.

**[R3] `a2a-sdk` API instability** → The A2A SDK is at version 0.3.x and APIs may change. Mitigation: pin exact version in `pyproject.toml`; keep the integration surface minimal.

**[R4] Nova Light streaming behavior** → `amazon.nova-light-v1:0` streaming chunk granularity is model-dependent. Tokens may arrive in multi-token chunks rather than single tokens. Mitigation: acceptable for haiku display; each chunk is forwarded as-is through the A2A streaming pipeline.

**[R5] CodeBuild cold start on first deploy** → First deployment triggers a Docker build which may take several minutes. Mitigation: document expected deployment time; CodeBuild caches layers on subsequent deploys.
