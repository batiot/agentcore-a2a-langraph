## ADDED Requirements

### Requirement: A2A server with streaming support

The system SHALL run an `A2AStarletteApplication` from the `a2a-sdk` library, configured with `streaming: true` in the `AgentCard`, listening on `0.0.0.0:9000` via `uvicorn`.

#### Scenario: Server starts and serves agent card

- **WHEN** the server process starts
- **THEN** the server SHALL listen on port 9000 and serve the agent card at `/.well-known/agent-card.json` with `streaming` capability set to `true`

#### Scenario: Server accepts A2A JSON-RPC requests

- **WHEN** a client sends a valid A2A JSON-RPC `message/send` or `message/stream` request
- **THEN** the server SHALL route the request to the `HaikuAgentExecutor` for processing

### Requirement: Custom AgentExecutor with streaming lifecycle

The system SHALL implement `HaikuAgentExecutor(AgentExecutor)` with `execute()` and `cancel()` methods. The `execute()` method SHALL use `TaskUpdater` to emit streaming status updates.

#### Scenario: Streaming execution lifecycle

- **WHEN** a message/stream request is received
- **THEN** the executor SHALL set task status to `working`, stream each token chunk as a status update via `TaskUpdater`, call `add_artifact()` with the complete haiku text, and call `complete()` to finalize the task

#### Scenario: Non-streaming execution

- **WHEN** a message/send request is received
- **THEN** the executor SHALL invoke the graph, collect the full response, add it as an artifact, and complete the task

### Requirement: DefaultRequestHandler with InMemoryTaskStore

The system SHALL use `DefaultRequestHandler` from `a2a-sdk` with an `InMemoryTaskStore` for task state management. Task persistence is scoped to the process lifetime — tasks are not durable across container restarts or scale-out events. AgentCore Runtime provides session isolation via the `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` header, routing requests within a session to the same container instance.

#### Scenario: Task state is tracked within a session

- **WHEN** a request creates a task
- **THEN** the task SHALL be stored in the `InMemoryTaskStore` and retrievable by task ID

### Requirement: AgentCard configuration with runtime URL

The `AgentCard` SHALL declare the agent's name, description, URL, supported capabilities (streaming), and supported content types (`text/plain`). The `url` field SHALL be read from the `AGENTCORE_RUNTIME_URL` environment variable, falling back to `http://localhost:9000/` for local development.

#### Scenario: Agent card contains required fields

- **WHEN** a client fetches `/.well-known/agent-card.json`
- **THEN** the response SHALL include `name`, `description`, `url`, `capabilities.streaming: true`, and `defaultInputModes`/`defaultOutputModes` containing `text/plain`

#### Scenario: Agent card URL reflects runtime endpoint

- **WHEN** the `AGENTCORE_RUNTIME_URL` environment variable is set
- **THEN** the agent card `url` field SHALL match the value of `AGENTCORE_RUNTIME_URL`

- **WHEN** the `AGENTCORE_RUNTIME_URL` environment variable is not set
- **THEN** the agent card `url` field SHALL default to `http://localhost:9000/`

### Requirement: Health check endpoint

The server SHALL expose a `/ping` GET endpoint for health checking.

#### Scenario: Ping returns healthy status

- **WHEN** a client sends a GET request to `/ping`
- **THEN** the server SHALL respond with HTTP 200 and JSON body `{"status": "healthy"}`
