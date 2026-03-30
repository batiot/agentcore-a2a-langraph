## ADDED Requirements

### Requirement: Haiku generation via LangGraph

The system SHALL expose a LangGraph `StateGraph` with a single `generate_haiku` node that invokes `ChatBedrockConverse` with model `amazon.nova-light-v1:0` to produce a haiku poem in response to any user message.

#### Scenario: User sends a topic and receives a haiku

- **WHEN** the graph is invoked with a user message containing a topic (e.g. "autumn leaves")
- **THEN** the graph SHALL return a response containing a haiku poem (three lines, 5-7-5 syllable structure) related to the topic

#### Scenario: User sends a message without a clear topic

- **WHEN** the graph is invoked with a vague or empty user message
- **THEN** the graph SHALL still return a valid haiku poem, using a default or interpreted theme

### Requirement: System prompt enforces haiku-only output

The `ChatBedrockConverse` model SHALL be configured with a system prompt that instructs it to respond exclusively with haiku poems and nothing else.

#### Scenario: Model output contains only a haiku

- **WHEN** the LLM generates a response
- **THEN** the response SHALL contain only a haiku poem without preamble, explanation, or additional text

### Requirement: Async streaming token generation

The graph SHALL support token-level streaming via `astream_events(input, version="v2")` filtering for `on_chat_model_stream` event kind.

#### Scenario: Tokens are streamed incrementally

- **WHEN** the graph streams a haiku response
- **THEN** each `on_chat_model_stream` event SHALL yield one or more tokens that can be forwarded to the caller before the full response is complete

### Requirement: Typed state schema

The graph state SHALL be defined as a `TypedDict` with a `messages` field typed as `Annotated[list[BaseMessage], add_messages]` following the LangGraph messaging convention.

#### Scenario: State contains message history

- **WHEN** the graph is invoked
- **THEN** the state SHALL contain the user's input message and the model's response message in the `messages` list
