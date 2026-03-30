## Why

This project needs a working A2A LangGraph agent deployed on AWS AgentCore to validate the end-to-end pattern: agent development with LangGraph, A2A protocol serving, streaming responses, CloudFormation-based deployment, and OAuth security with an external EntraID OIDC provider. A haiku-only agent provides a minimal but complete vertical slice — simple enough to focus on infrastructure and protocol, interesting enough to demonstrate streaming.

## What Changes

- New LangGraph `StateGraph` agent that responds exclusively with haiku poems, using Amazon Bedrock Nova Light (`amazon.nova-light-v1:0`)
- A2A server using `a2a-sdk` (`A2AStarletteApplication`, custom `AgentExecutor`) with true token-level streaming via SSE
- Local Chainlit application connected to the A2A server via `a2a.client` SDK, displaying streamed haiku responses
- CloudFormation stack for full AWS deployment: ECR, CodeBuild (ARM64), IAM roles, AgentCore Runtime (A2A protocol on port 9000)
- OAuth 2.0 bearer token security using EntraID OIDC (tenant `5e39efe7-3ad5-4fc4-a68c-7b12fb4fdf0f`) — pre-generated JWT passed as env var for client invocation
- No dependency on StrandAgents framework or AgentCore starter toolkit CLI

## Capabilities

### New Capabilities

- `haiku-graph`: LangGraph StateGraph agent that generates haiku poems via Bedrock Nova Light, with async streaming support
- `a2a-server`: A2A protocol server using a2a-sdk with streaming AgentExecutor, AgentCard discovery, and health endpoint
- `chainlit-client`: Chainlit web UI that connects to the A2A server via a2a-sdk client with streaming message display
- `cfn-deployment`: CloudFormation stack for deploying the complete agent infrastructure to AWS (ECR, IAM, CodeBuild, AgentCore Runtime)
- `oauth-security`: OAuth 2.0 bearer token authentication using EntraID OIDC for securing the AgentCore Runtime endpoint

### Modified Capabilities

_(none — greenfield project)_

## Impact

- **New files**: Agent source (`src/haiku_agent/`), Chainlit client (`client/`), CloudFormation template (`infra/`), Dockerfile, pyproject.toml
- **Dependencies**: langgraph, langchain-aws, a2a-sdk, uvicorn, httpx, chainlit
- **AWS resources**: ECR repository, IAM roles (agent execution, CodeBuild, Lambda custom resource), CodeBuild project, AgentCore Runtime, CloudWatch log group
- **External**: EntraID app registration (existing), Amazon Bedrock Nova Light model access in eu-west-3
