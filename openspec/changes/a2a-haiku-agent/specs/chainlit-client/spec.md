## ADDED Requirements

### Requirement: Chainlit web interface for A2A interaction

The system SHALL provide a Chainlit application that connects to the A2A server and allows users to send messages and view streamed haiku responses in a web chat interface.

#### Scenario: User sends a message and sees streamed response

- **WHEN** a user types a message in the Chainlit UI
- **THEN** the client SHALL send a `message/stream` request to the A2A server and display each streamed token incrementally in the chat interface

#### Scenario: Client connects to configurable server URL

- **WHEN** the Chainlit app starts
- **THEN** it SHALL read the A2A server URL from the `A2A_SERVER_URL` environment variable (defaulting to `http://localhost:9000`)

### Requirement: Bearer token authentication

The Chainlit client SHALL include an OAuth bearer token in the `Authorization` header of all requests to the A2A server.

#### Scenario: Token loaded from environment

- **WHEN** the client makes a request to the A2A server
- **THEN** it SHALL include `Authorization: Bearer <token>` where the token is read from the `A2A_BEARER_TOKEN` environment variable

#### Scenario: Missing token allows local development

- **WHEN** the `A2A_BEARER_TOKEN` environment variable is not set
- **THEN** the client SHALL send requests without an `Authorization` header (for local development without AgentCore)

### Requirement: Streaming message display

The Chainlit client SHALL use the `a2a.client` SDK with `httpx.AsyncClient` to consume SSE streaming responses and update the chat message in real time.

#### Scenario: Tokens appear incrementally

- **WHEN** the A2A server streams token chunks
- **THEN** the Chainlit UI SHALL append each chunk to the current message as it arrives, providing a real-time typing effect
