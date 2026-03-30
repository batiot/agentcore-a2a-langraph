# GitHub Copilot Instructions

## Project Overview

This project experiments with deploying an **A2A (Agent-to-Agent) LangGraph agent** on **AWS AgentCore** (Amazon Bedrock AgentCore). The goal is to explore multi-agent orchestration patterns using LangGraph's graph-based agent framework, deployed and managed via AWS AgentCore services.
A chainlit application (plug with a2a client) is included for testing and demonstration purposes, showcasing how to interact with the A2A agent via a web interface.

## Tech Stack

- **Language**: Python 3.12+
- **Package manager**: [`uv`](https://docs.astral.sh/uv/) — use `uv` for all dependency management and script execution
- **Agent framework**: [LangGraph](https://langchain-ai.github.io/langgraph/) for building stateful, graph-based agent workflows
- **A2A protocol**: Google's [Agent-to-Agent (A2A) protocol](https://google.github.io/A2A/) for inter-agent communication
- **Cloud platform**: [AWS AgentCore](https://aws.amazon.com/bedrock/agentcore/) (Amazon Bedrock AgentCore)
  - AgentCore Runtime — for deploying and serving agents
  - AgentCore Memory — for agent memory and state persistence
- **LLM provider**: Amazon Bedrock (nova light models preferred)

## Project Conventions

### Package & Dependency Management
- Always use `uv` — never `pip`, `pip install`, or `python -m pip`
- Use `uv add <package>` to add dependencies to `pyproject.toml`
- Use `uv run <script>` to execute scripts
- Use `uv sync` to install dependencies from lockfile
- The project uses `pyproject.toml` (not `requirements.txt` or `setup.py`)
- Pin dependency versions in `pyproject.toml` for reproducibility

### Code Style
- Follow PEP 8 and use `ruff` for linting/formatting (via `uv run ruff`)
- Use type hints throughout
- Prefer `async`/`await` for I/O-bound operations (LangGraph and AgentCore are async-native)
- Keep agent nodes and tools in separate modules

### LangGraph Patterns
- Define agent graphs using `StateGraph` with typed state schemas (`TypedDict` or Pydantic)
- Prefer `checkpointer` integration for stateful conversations

### AWS / AgentCore Patterns
- Store AWS credentials/config via environment variables or AWS profiles — never hardcode
- Respect IAM least-privilege principles — define minimal IAM policies
- Use `eu-west-3` as the default region unless otherwise specified

### A2A Protocol
- Handle streaming responses

## Directory Structure (to be defined)


## Development Workflow

1. Use `uv sync` to set up the environment after cloning
2. Use `uv run pytest` for tests
3. Use `uv run ruff check .` for linting
4. Use `uv run ruff format .` for formatting
5. Changes are tracked via OpenSpec in `openspec/changes/`

## Key References

- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)
- [AWS AgentCore documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- [A2A Protocol specification](https://google.github.io/A2A/)
- [uv documentation](https://docs.astral.sh/uv/)
