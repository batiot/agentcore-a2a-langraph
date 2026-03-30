# Design Review: a2a-haiku-agent

> Generated: 2026-03-30

## Socratic Questions

### Q1: How does CodeBuild obtain the local source tree?
Decision D4 says the stack will create a CodeBuild project with an inline buildspec that "generates the Dockerfile and copies source from the repo." What is the explicit source-of-truth handoff from the developer workstation to CodeBuild at stack creation time, and how is that packaging step represented in the design rather than left implicit?

### Q2: What guarantees task retrieval after a restart or scale-out event?
The design chooses `DefaultRequestHandler` with `InMemoryTaskStore`, while the A2A server spec requires tasks to be retrievable by task ID. If AgentCore restarts the container or routes follow-up task access to another instance, what behavior is expected for task lookup and task status continuity?

### Q3: Is token-per-status-update the right A2A contract?
Decision D2 streams each chunk by emitting repeated `working` status updates and only adds the final haiku as an artifact at completion. Is that pattern required by the `a2a-sdk` contract for streamed content, or is it a convention taken from one sample that could make downstream clients depend on status messages as content transport?

### Q4: Who owns Entra token rotation during demos and troubleshooting?
Decision D5 intentionally uses a pre-generated JWT in `A2A_BEARER_TOKEN`, and risk R2 acknowledges normal Entra token expiry. When the token expires during a demo or smoke test, who refreshes it, how is expiry detected quickly, and what operator experience is expected when authentication failures start surfacing as 401s?

### Q5: What is the fallback if A2A runtime OAuth differs from MCP runtime OAuth?
The design treats `AuthorizerConfiguration` as uncertain for `AWS::BedrockAgentCore::Runtime`, while sample infrastructure in the workspace shows the property for MCP runtimes. What evidence confirms the same shape works for an A2A runtime, and what exact deploy-time branch is taken if the property exists but differs by protocol?

### Q6: What breaks first when a second agent capability is added?
The design argues that a single-node graph and a focused executor can "extend naturally if nodes are added later." Which files would have to change first if the project added a second output mode, a second content type, or a second agent persona, and is the current structure intentionally optimized for that extension path or just minimal for today's demo?

---

## Principle Analysis

### Separation of Concerns / KISS — Major concern
Decision D4 combines infrastructure provisioning, container build orchestration, source packaging assumptions, and potentially OAuth bootstrap logic into one CloudFormation-centric flow: "Single CloudFormation template creating: ECR repo, CodeBuild project ... Lambda function to trigger CodeBuild, and `AWS::BedrockAgentCore::Runtime` resource." That is already several independent responsibilities, and the design still leaves the source delivery mechanism unstated. The risk is not just complexity; it is an incomplete deployment boundary where stack creation can succeed conceptually in the design while still lacking a defined path for the code that CodeBuild is supposed to build.

### Coupling — Major concern
The A2A server design binds runtime behavior to `InMemoryTaskStore`, while the spec requires created tasks to be retrievable by task ID. That couples correctness to a hidden infrastructure assumption: one stable process instance. On a managed runtime, restarts, multiple instances, or proxy routing can break task lookup semantics even if basic streaming works, which means a protocol-level requirement is being satisfied only under a narrow deployment topology.

### SRP — Minor concern
`HaikuAgentExecutor` is planned to handle request mode branching, graph invocation, token streaming, task lifecycle transitions, and cancellation. For a minimal prototype that may be acceptable, but it gives the executor multiple reasons to change: A2A SDK lifecycle changes, streaming policy changes, prompt/graph changes, or artifact formatting changes. The risk is moderate now, but this file becomes the first hotspot once the agent grows beyond a single haiku path.

### OCP — Minor concern
The design hard-codes one content type (`text/plain`), one model, one output artifact shape, and one graph behavior directly into the executor/card pairing. That is consistent with the current scope, but it means adding a second output mode or swapping delivery behavior requires editing core server code rather than extending configuration or composition points. The risk is acceptable for the first slice, but the design should be explicit that extension currently requires source changes.

### DRY — Minor concern
Critical operational values are repeated across proposal, design, tasks, and specs: port `9000`, region `eu-west-3`, runtime URL behavior, and the pre-generated token flow. Some repetition is normal in OpenSpec, but here the same deployment decisions are duplicated in enough places that drift has already appeared around Docker build ownership and OAuth fallback behavior. The risk is not verbosity; it is decision fragmentation.

### YAGNI — Minor concern
The inline CodeBuild plus Lambda custom resource path appears optimized for a one-command bootstrap experience before the basic deployment path has been proven. For a greenfield reference implementation, that may be more automation than the current scope needs, especially when the project explicitly treats CI/CD and production hardening as non-goals. The risk is spending complexity budget on provisioning choreography instead of validating the actual A2A and AgentCore reference path.

---

## Duplication

### D1: Docker build responsibility is described twice
Decision D4 says CodeBuild will use an inline buildspec that "generates the Dockerfile," while the tasks include a separate top-level Dockerfile deliverable and a dedicated local Docker validation step. That duplicates ownership of the container contract between cloud build logic and repository source, increasing the chance that local and deployed images diverge.

### D2: OAuth deployment path is defined in both infrastructure and scripts
The design says OAuth may require CloudFormation configuration, a custom resource, or a post-deploy CLI step. The OAuth spec further says the deploy script shall perform the fallback CLI configuration if the CloudFormation property is unavailable. The same operational concern is therefore spread across design rationale, infrastructure expectations, and shell script behavior without one canonical path.

### D3: Runtime endpoint behavior is scattered across multiple artifacts
Proposal, design, tasks, and the A2A server spec all restate that the runtime URL comes from `AGENTCORE_RUNTIME_URL` with a localhost fallback and that the service listens on port `9000`. This is a small example, but it shows the broader pattern that environment assumptions are repeated across artifacts instead of anchored in one primary contract.

---

## Artifact Drift

### Drift-1: Dockerfile ownership is contradictory
**Source**: `design.md` | **Target**: `tasks.md` | **Type**: Contradictory

The design says: "CodeBuild uses an inline buildspec that generates the Dockerfile and copies source from the repo." The tasks say: "Create `Dockerfile`" and then "Test Docker build and run locally." These are different ownership models. Either the repository has a canonical Dockerfile that both local and cloud builds consume, or the stack generates one dynamically. Keeping both paths creates avoidable divergence in the most deployment-sensitive artifact.

### Drift-2: One-command deployment conflicts with OAuth fallback ambiguity
**Source**: `proposal.md` | **Target**: `design.md` and `specs/oauth-security/spec.md` | **Type**: Contradictory

The proposal promises "One-command CloudFormation deployment and teardown" and describes a "CloudFormation stack for full AWS deployment." The design later says the runtime authorizer "may need a post-deploy AWS CLI step or custom resource to configure the OIDC authorizer," and the OAuth spec explicitly allows a deploy-script fallback when the CloudFormation property is unavailable. That means the deployment story is not actually settled as one command through one mechanism.

### Drift-3: Source packaging work is missing from the implementation plan
**Source**: `design.md` | **Target**: `tasks.md` | **Type**: Missing

Decision D4 assumes CodeBuild can build from repository source during stack creation, but the task plan never adds a step to upload, package, or otherwise provide the local source tree to CodeBuild. The tasks cover ECR, CodeBuild, Lambda triggering, and runtime creation, but not the source ingress that makes the build possible.

### Drift-4: OAuth fallback is required by spec but not represented in tasks
**Source**: `specs/oauth-security/spec.md` | **Target**: `tasks.md` | **Type**: Missing

The OAuth spec says: "WHEN the CloudFormation `AuthorizerConfiguration` property is not supported THEN the deploy script SHALL include a post-deploy AWS CLI step to configure the OIDC authorizer on the Runtime endpoint." The tasks mention adding OIDC config to the template "or document post-deploy CLI step if CFn property unavailable," but there is no explicit task to update `infra/deploy.sh` with that fallback branch.

### Drift-5: Region parameterization is stale against the fixed-region requirement
**Source**: `tasks.md` | **Target**: `proposal.md` and `specs/cfn-deployment/spec.md` | **Type**: Stale

Task 6.1 says the template should take a region parameter, while the proposal and CloudFormation spec both treat `eu-west-3` as the required deployment region. This is not a severe contradiction, but it indicates the task list is carrying a more generic template shape than the rest of the change currently intends to support.

---

## Proposed Alternatives

### Alt-1: Move image build and push into the deploy script
**Addresses**: Separation of Concerns / KISS, Drift-1, Drift-3
**Type**: Simpler
**Description**: Keep CloudFormation responsible only for durable infrastructure, and let `infra/deploy.sh` build and push the image from the local repository before stack creation or stack update. That makes the source handoff explicit, preserves a single canonical Dockerfile, and removes the Lambda-triggered CodeBuild bootstrap path.
**Trade-offs**: The developer machine needs Docker and ECR login configured, and the deployment flow is less cloud-native than a fully managed build service.

### Alt-2: Keep CodeBuild, but make source packaging a first-class artifact
**Addresses**: Separation of Concerns / KISS, Drift-3
**Type**: Cleaner
**Description**: If managed cloud builds are required, define an explicit source bundle path in the design: package the repository into S3 from `deploy.sh`, point CodeBuild at that bundle, and treat the repository Dockerfile as the only container contract. This preserves one-command deployment while removing the implicit "copy source from the repo" assumption.
**Trade-offs**: Adds an S3 artifact step and still leaves more moving parts than a local build-and-push flow.

### Alt-3: Narrow the protocol promise or persist task state outside the container
**Addresses**: Coupling, Q2
**Type**: Cleaner
**Description**: Either explicitly scope the first release to streaming responses without durable task retrieval semantics, or replace `InMemoryTaskStore` with an external store aligned to the deployment model. Both paths are better than quietly depending on single-instance behavior while the spec promises retrievable task state.
**Trade-offs**: Narrowing the promise reduces capability; external persistence adds infrastructure and implementation work.

### Alt-4: Make OAuth setup a single, chosen path
**Addresses**: Drift-2, Drift-4, operational ambiguity
**Type**: Simpler
**Description**: Decide now whether OAuth is configured directly in CloudFormation or always applied in `deploy.sh` after runtime creation, and document only that path in proposal, design, tasks, and spec. Given the current uncertainty, a scripted post-deploy step may be the more honest short-term choice.
**Trade-offs**: You give up some declarative purity if you choose the scripted path, or you accept more implementation risk up front if you force everything through CloudFormation.

### Alt-5: Split executor responsibilities before the second feature arrives
**Addresses**: SRP, OCP
**Type**: Cleaner
**Description**: Keep `HaikuAgentExecutor` thin by extracting graph invocation and stream-to-A2A adaptation into separate helpers, even if the initial implementation remains small. That keeps the A2A lifecycle code stable while prompt/model/output behavior evolves separately.
**Trade-offs**: Slightly more files and indirection in a project whose current behavior is intentionally tiny.

---

## Summary

| Area | Severity | Note |
|------|----------|------|
| Cloud build orchestration boundary | High | Infrastructure flow assumes CodeBuild can access local source without a defined packaging step. |
| Task state persistence | High | `InMemoryTaskStore` satisfies the spec only if the runtime behaves like a single stable process. |
| Docker build ownership drift | Medium | Design and tasks disagree on whether Dockerfile generation happens in the repo or in CodeBuild. |
| OAuth deployment path drift | Medium | Proposal promises one-command deployment, but design and spec still allow a separate fallback path. |
| Missing deploy task for OAuth fallback | Medium | The OAuth spec requires deploy-script behavior that the tasks do not currently represent. |
| Executor responsibility growth | Medium | The executor is positioned to accumulate streaming, lifecycle, and business logic changes in one place. |
| Decision duplication across artifacts | Low | Repeated deployment constants have already started to drift. |
| Region parameter mismatch | Low | Tasks still describe a generic region parameter although the rest of the change is fixed to `eu-west-3`. |