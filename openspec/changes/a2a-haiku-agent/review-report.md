# Review Report: a2a-haiku-agent

> Generated: 2026-03-31
> Input: review-design.md (2026-03-30)

This report traces user decisions on each review finding, records research evidence, and lists resulting artifact changes.

---

## Socratic Questions — Decisions

### Q1: How does CodeBuild obtain the local source tree?

**Decision**: Resolved via D1 / Drift-1 — no inline buildspec. See below.

### Q2: What guarantees task retrieval after a restart or scale-out event?

**Decision**: Accepted — `InMemoryTaskStore` is the standard A2A pattern; scoped acceptable for single-shot haiku.

**Research (context7 + bedrock-agentcore-mcp-server)**:
- AgentCore Runtime A2A docs state containers run as **"stateless, streamable HTTP servers"** on port 9000.
- AgentCore provides enterprise-grade **session isolation** via `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` header — requests within a single session are routed to the same container instance.
- The `a2a-sdk` `DefaultRequestHandler` implements `tasks/get` and `tasks/resubscribe` backed by the configured `TaskStore`. All official samples (including agentcore-samples) use `InMemoryTaskStore`.
- The A2A SDK exposes `TaskResubscriptionRequest` (`tasks/resubscribe`) for clients to re-attach to a running streaming task's event stream — this requires the task and its queue to still be active in-process.
- For a single-shot haiku agent with no multi-turn conversation, task retrieval after restarts is not a functional requirement. The `InMemoryTaskStore` satisfies the protocol contract within a session.

**Artifact impact**: None — design remains as-is. Add an explicit scoping note in the A2A server spec that task persistence is scoped to the process lifetime and session isolation.

### Q3: Is token-per-status-update the right A2A contract?

**Decision**: Confirmed — `TaskStatusUpdateEvent` with `working` state is the standard A2A streaming contract.

**Research (context7 + deepwiki)**:
- The `a2a-sdk` `SendStreamingMessageResponse.result` union type is `Task | Message | TaskStatusUpdateEvent | TaskArtifactUpdateEvent`.
- `TaskStatusUpdateEvent` contains: `contextId`, `final` (bool), `kind='status-update'`, `status` (TaskStatus with `state` and optional `message`), `taskId`.
- `TaskArtifactUpdateEvent` contains: `append` (bool), `artifact`, `contextId`, `kind='artifact-update'`, `last_chunk` (bool), `taskId`.
- The standard streaming lifecycle is: emit `TaskStatusUpdateEvent` with `state=working` for progress, then `TaskArtifactUpdateEvent` for the final artifact, then `TaskStatusUpdateEvent` with `state=completed` and `final=true`.
- The `TaskStatus.message` field (a `Message` object with `parts`) can carry content within status updates — this is the mechanism for streaming token chunks as `working` status updates.
- This matches the pattern used in agentcore-samples monitoring agent and is the documented A2A SDK contract.

**Artifact impact**: None — design D2 is correct.

### Q4: Who owns Entra token rotation during demos and troubleshooting?

**Decision**: Manual for now. Risk R2 accepted as documented.

**Artifact impact**: None.

### Q5: What is the fallback if A2A runtime OAuth differs from MCP runtime OAuth?

**Decision**: Confirmed — `AuthorizerConfiguration.CustomJWTAuthorizer` is a CloudFormation property on `AWS::BedrockAgentCore::Runtime`. If unsupported for A2A protocol, AWS CLI in deploy.sh is required. See Drift-2.

**Research (context7 + bedrock-agentcore-mcp-server)**:
- The MCP server CloudFormation template uses `AuthorizerConfiguration` directly on `AWS::BedrockAgentCore::Runtime`:
  ```yaml
  MCPServerRuntime:
    Type: AWS::BedrockAgentCore::Runtime
    Properties:
      ProtocolConfiguration: MCP
      AuthorizerConfiguration:
        CustomJWTAuthorizer:
          AllowedClients:
            - !Ref CognitoUserPoolClient
          DiscoveryUrl: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${CognitoUserPool}/.well-known/openid-configuration"
  ```
- The weather agent template (no protocol config, default HTTP) does **not** use `AuthorizerConfiguration`.
- `AuthorizerConfiguration.CustomJWTAuthorizer` requires `AllowedClients` (list of client IDs) and `DiscoveryUrl` (OIDC discovery endpoint).
- For A2A with EntraID, the shape would be:
  ```yaml
  ProtocolConfiguration: A2A
  AuthorizerConfiguration:
    CustomJWTAuthorizer:
      AllowedClients:
        - "<entra-client-id>"
      DiscoveryUrl: "https://login.microsoftonline.com/<tenant>/v2.0/.well-known/openid-configuration"
  ```
- The AgentCore A2A docs confirm: "Supports both SigV4 and OAuth 2.0 authentication schemes."
- Since the CloudFormation property exists on the same resource type, it should work for A2A protocol. If it does not, `deploy.sh` must include an explicit AWS CLI fallback.

**Artifact impact**: Update design D4/D5, cfn-deployment spec, oauth-security spec, and tasks to include the concrete `AuthorizerConfiguration` shape and a required CLI fallback in `deploy.sh`.

### Q6: What breaks first when a second agent capability is added?

**Decision**: Current structure is intentionally optimized for the extension path. No changes needed.

**Artifact impact**: None.

---

## Duplication — Decisions

### D1: Docker build responsibility is described twice

**Decision**: No inline buildspec. Canonical Dockerfile lives in the repo. CodeBuild uses S3 source bundle containing actual project source + Dockerfile.

**Resolution**: `deploy.sh` packages the project source into an S3 zip, CodeBuild `Source.Type: S3` references the bucket/key, and a `buildspec.yml` in the repo root drives the Docker build using the canonical `Dockerfile`. This eliminates the inline buildspec that both generated the Dockerfile and embedded source, giving a single owner for the container contract.

**Artifact impact**: Update design D4, cfn-deployment spec, tasks (add buildspec.yml creation, S3 upload in deploy.sh, update CodeBuild task).

### D2: OAuth deployment path is defined in both infrastructure and scripts

**Decision**: Resolved via Drift-2 — single canonical path: CloudFormation first, CLI fallback in deploy.sh if needed.

### D3: Runtime endpoint behavior is scattered across multiple artifacts

**Decision**: Accepted — normal OpenSpec repetition. No changes needed.

---

## Drift — Decisions

### Drift-1: Dockerfile ownership is contradictory

**Decision**: No inline buildspec. Single canonical Dockerfile in repo. Both local Docker build and CodeBuild use the same Dockerfile.

**Resolution**: Same as D1. The design (D4) and tasks are updated to remove inline buildspec language. CodeBuild builds from S3 source bundle containing the repo's Dockerfile.

**Artifact impact**: Update design D4, cfn-deployment spec (remove inline buildspec requirement), tasks 5.1 (Dockerfile stays), 6.4 (CodeBuild uses S3 source).

### Drift-2: One-command deployment conflicts with OAuth fallback ambiguity

**Decision**: If CloudFormation `AuthorizerConfiguration` is not supported for A2A protocol, AWS CLI in deploy.sh is required (not optional).

**Resolution**: The design and specs now follow a two-step approach: (1) attempt CloudFormation `AuthorizerConfiguration.CustomJWTAuthorizer` with the known property shape, (2) `deploy.sh` detects whether auth was configured and runs AWS CLI as fallback. The "one-command deployment" promise is preserved because `deploy.sh` handles both paths.

**Artifact impact**: Update design D4/D5, oauth-security spec (make CLI fallback a required deliverable), tasks 6.6 and 7.1 (add AuthorizerConfiguration shape), add task for CLI fallback in deploy.sh.

### Drift-3: Source packaging work is missing from the implementation plan

**Decision**: Resolved by D1 — deploy.sh now explicitly packages source to S3 before stack creation.

**Artifact impact**: Add task for S3 source packaging in deploy.sh (before CodeBuild task).

### Drift-4: OAuth fallback is required by spec but not represented in tasks

**Decision**: Resolved by Drift-2 — explicit task for CLI fallback in deploy.sh.

**Artifact impact**: Add task for OAuth CLI fallback in deploy.sh.

### Drift-5: Region parameterization is stale against the fixed-region requirement

**Decision**: Not explicitly addressed by user. Keeping task 6.1 region parameter for template flexibility, but deploy.sh defaults to `eu-west-3`.

**Artifact impact**: None — minor inconsistency, acceptable.

---

## Summary of Artifact Changes

| Artifact | Change |
|----------|--------|
| `design.md` | D4: Replace inline buildspec with S3 source bundle + canonical Dockerfile + buildspec.yml. Add `AuthorizerConfiguration` shape to D4/D5. |
| `tasks.md` | Add: buildspec.yml task, S3 source packaging in deploy.sh, OAuth CLI fallback task. Update: 6.4 (CodeBuild S3 source), 6.8 (deploy.sh packages source). |
| `specs/cfn-deployment/spec.md` | Remove inline buildspec requirement. Add S3 source bundle requirement. Update CodeBuild scenario. |
| `specs/a2a-server/spec.md` | Add scoping note: task persistence is process-lifetime only, session-isolated. |
| `specs/oauth-security/spec.md` | Add concrete `AuthorizerConfiguration` shape. Make CLI fallback a required deploy.sh deliverable. |
