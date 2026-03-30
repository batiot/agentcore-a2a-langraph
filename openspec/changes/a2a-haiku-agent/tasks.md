## 1. Project Setup

- [ ] 1.1 Initialize `pyproject.toml` with `uv init` — configure project metadata, Python 3.12+, ruff (line-length 100)
- [ ] 1.2 Add core dependencies: `langgraph`, `langchain-aws`, `a2a-sdk`, `uvicorn`, `httpx`
- [ ] 1.3 Add dev dependencies: `pytest`, `pytest-asyncio`, `ruff`
- [ ] 1.4 Create `src/haiku_agent/__init__.py` and directory structure (`graph.py`, `executor.py`, `agent_card.py`, `main.py`)
- [ ] 1.5 Create `client/` directory with `chainlit_app.py` placeholder
- [ ] 1.6 Create `infra/` directory with placeholder files (`template.yaml`, `deploy.sh`, `destroy.sh`)

## 2. LangGraph Haiku Agent

- [ ] 2.1 Define typed state schema (`HaikuState` TypedDict with `messages: Annotated[list[BaseMessage], add_messages]`) in `graph.py`
- [ ] 2.2 Implement `generate_haiku` node: create `ChatBedrockConverse` with `amazon.nova-light-v1:0`, system prompt enforcing haiku-only output, invoke with state messages
- [ ] 2.3 Build `StateGraph` with single `generate_haiku` node, compile to `haiku_graph`
- [ ] 2.4 Test graph locally: invoke with a topic, verify haiku output; test `astream_events` with `on_chat_model_stream` filter, verify token-level streaming

## 3. A2A Server

- [ ] 3.1 Define `AgentCard` in `agent_card.py` with name, description, `streaming: true`, content types `text/plain` — read `url` from `AGENTCORE_RUNTIME_URL` env var (default `http://localhost:9000/`)
- [ ] 3.2 Implement `HaikuAgentExecutor(AgentExecutor)` in `executor.py` with `execute()` method: create `TaskUpdater`, set working status, stream tokens from `haiku_graph.astream_events()`, emit status updates per chunk, `add_artifact()` with complete text, `complete()`
- [ ] 3.3 Implement `cancel()` method in `HaikuAgentExecutor` (set task to canceled)
- [ ] 3.4 Implement non-streaming path in `execute()`: detect non-streaming request, invoke graph synchronously, add artifact, complete
- [ ] 3.5 Wire up `main.py`: create `InMemoryTaskStore`, `DefaultRequestHandler`, `A2AStarletteApplication`, add `/ping` health endpoint (GET returning `{"status": "healthy"}`), run with `uvicorn` on `0.0.0.0:9000`
- [ ] 3.6 Test A2A server locally: start server, fetch `/.well-known/agent-card.json`, send `message/send` and `message/stream` requests, verify haiku responses and streaming

## 4. Chainlit Client

- [ ] 4.1 Add `chainlit` dependency to `pyproject.toml`
- [ ] 4.2 Implement `chainlit_app.py`: read `A2A_SERVER_URL` (default `http://localhost:9000`) and `A2A_BEARER_TOKEN` env vars
- [ ] 4.3 Implement message handler: on user message, send `message/stream` request via `a2a.client` SDK with `httpx.AsyncClient`, include bearer token if set
- [ ] 4.4 Implement streaming display: consume SSE response, append each token chunk to Chainlit message in real time
- [ ] 4.5 Test Chainlit client end-to-end: start A2A server locally, launch Chainlit, send message, verify streamed haiku display

## 5. Dockerfile

- [ ] 5.1 Create `Dockerfile`: Python 3.12 slim base, install `uv`, copy source, `uv sync`, expose port 9000, entrypoint `uv run python -m haiku_agent.main`
- [ ] 5.2 Test Docker build and run locally: build image, run container, verify agent card and haiku generation on port 9000

## 6. CloudFormation Deployment

- [ ] 6.1 Create `infra/template.yaml` with parameters: stack name, region, ECR repo name
- [ ] 6.2 Add ECR repository resource
- [ ] 6.3 Add IAM roles: agent execution role (Bedrock invoke + CloudWatch), CodeBuild role (ECR + CloudWatch), Lambda custom resource role
- [ ] 6.4 Add CodeBuild project: ARM64 (`aws/codebuild/amazonlinux2-aarch64-standard:3.0`), inline buildspec that builds and pushes Docker image to ECR
- [ ] 6.5 Add Lambda custom resource to trigger CodeBuild on stack creation
- [ ] 6.6 Add `AWS::BedrockAgentCore::Runtime` resource: A2A protocol, port 9000, root path `/`
- [ ] 6.7 Add stack outputs: AgentCore Runtime endpoint URL, ECR repository URI
- [ ] 6.8 Implement `infra/deploy.sh`: create stack, wait for completion, print endpoint URL
- [ ] 6.9 Implement `infra/destroy.sh`: delete ECR images, delete stack, wait for deletion
- [ ] 6.10 Test deployment: deploy stack to eu-west-3, verify agent reachable via AgentCore endpoint

## 7. OAuth Security

- [ ] 7.1 Add OIDC authorizer configuration to CloudFormation template (or document post-deploy CLI step if CFn property unavailable)
- [ ] 7.2 Document JWT token generation steps in README (curl to EntraID token endpoint with client credentials)
- [ ] 7.3 Test authenticated access: deploy with OAuth enabled, send request with valid JWT, verify 200; send without token, verify 401

## 8. Documentation & Cleanup

- [ ] 8.1 Create README.md with: project overview, prerequisites, local development steps, deployment instructions, token generation, architecture diagram
- [ ] 8.2 Add `.env.example` with all environment variables (`AWS_REGION`, `A2A_SERVER_URL`, `A2A_BEARER_TOKEN`)
- [ ] 8.3 Run `ruff check .` and `ruff format .` — fix any issues
