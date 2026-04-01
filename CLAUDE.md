# CLAUDE.md — Steering Document for Claude Code

This file guides Claude Code when building and maintaining this project.

## Project Overview

**customer-support-agent-bedrock** — A multi-turn customer support agent with persistent memory and RAG-powered FAQ. Built with Strands Agents SDK, Amazon Bedrock (Claude Sonnet), Amazon Bedrock Knowledge Bases, and AgentCore Memory.

The agent answers customer questions grounded in a knowledge base (product FAQs, policies, troubleshooting guides), maintains full conversation context across turns, and remembers returning customers across sessions.

## Tech Stack

- **Language**: Python 3.11+
- **Agent Framework**: [Strands Agents SDK](https://github.com/strands-agents/sdk-python)
- **LLM**: Amazon Bedrock — `us.anthropic.claude-sonnet-4-20250514-v1:0` via `strands.models.bedrock.BedrockModel`
- **RAG**: Amazon Bedrock Knowledge Bases (`bedrock-agent-runtime` `Retrieve` API)
- **Memory**: Amazon Bedrock AgentCore Memory (STM + LTM)
- **Deployment**: Amazon Bedrock AgentCore Runtime (container-based)
- **No database** — All persistent state lives in AgentCore Memory. No PostgreSQL, no DynamoDB.

## Architecture Principles

1. **Single agent, tool-based** — One `strands.Agent` instance per session with tools for KB retrieval. No sub-agents, no multi-agent orchestration.
2. **RAG-first for customer questions** — Always search the knowledge base before generating answers on product/policy/troubleshooting topics. Ground responses in retrieved context. Cite sources.
3. **Multi-turn memory** — Use AgentCore Memory for both short-term (session) and long-term (cross-session) memory. Returning customers should feel recognized.
4. **Stateless runtime** — No local database. Agent instances are ephemeral. All durable state lives in AgentCore Memory.
5. **Session isolation** — Each `session_id` gets its own Agent instance with separate conversation history.
6. **Lazy initialization** — All heavy setup (model client, memory client, KB client) deferred to first invoke to avoid AgentCore's 30-second startup timeout.

## Project Structure

```
customer-support-agent-bedrock/
├── CLAUDE.md                  # This file — AI coding assistant steering
├── SPEC.md                    # Functional specification
├── README.md                  # Project overview and setup instructions
├── entrypoint.py              # AgentCore Runtime entry point
├── requirements.txt           # Python dependencies
├── pyproject.toml             # Project metadata
├── Dockerfile                 # Container image for AgentCore deploy
├── .bedrock_agentcore.yaml    # AgentCore deployment config
├── src/
│   └── customer_support/
│       ├── __init__.py
│       ├── agent.py           # Agent factory — creates configured Strands Agent
│       ├── config.py          # Environment-based configuration
│       ├── tools/
│       │   ├── __init__.py
│       │   └── kb_retrieval.py    # Bedrock Knowledge Base retrieval tool
│       ├── memory/
│       │   ├── __init__.py
│       │   └── agentcore_memory.py # AgentCore Memory integration (STM + LTM)
│       └── prompts/
│           └── system_prompt.py   # System prompt assembly
├── tests/
│   ├── __init__.py
│   ├── test_agent.py
│   ├── test_tools/
│   │   └── test_kb_retrieval.py
│   └── test_memory/
│       └── test_agentcore_memory.py
├── knowledge-base/
│   ├── README.md              # Instructions to set up Bedrock KB
│   ├── sample-faqs/           # Sample FAQ documents for ingestion
│   └── setup_kb.py            # Script to create/sync Bedrock KB
└── scripts/
    └── local_test.py          # Local testing harness (CLI chat loop)
```

## Key Implementation Patterns

### Entry Point Pattern (from platform-as-agent)

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

_initialized = False

def ensure_initialized():
    global _initialized
    if _initialized:
        return
    # Heavy init here: model client, memory client, etc.
    _initialized = True

@app.entrypoint
def invoke(payload, context):
    ensure_initialized()
    session_id = context.session_id
    agent = agent_pool.get_or_create(session_id)
    result = agent(payload["message"])
    save_to_memory(session_id, payload["message"], result)
    return {"result": str(result)}
```

### Agent Pool Pattern

Use an `OrderedDict`-based LRU cache (max 100 sessions) of per-session `Agent` instances.

```python
from collections import OrderedDict

class AgentPool:
    def __init__(self, max_size: int = 100):
        self._pool: OrderedDict[str, Agent] = OrderedDict()
        self._max_size = max_size

    def get_or_create(self, session_id: str) -> Agent:
        if session_id in self._pool:
            self._pool.move_to_end(session_id)
            return self._pool[session_id]
        if len(self._pool) >= self._max_size:
            self._pool.popitem(last=False)
        agent = create_agent(session_id)
        self._pool[session_id] = agent
        return agent
```

### KB Retrieval Tool

Use `Retrieve` API (not `RetrieveAndGenerate`) so the Strands agent controls response generation.

```python
from strands import tool
import boto3

@tool
def kb_retrieve(query: str, num_results: int = 5) -> dict:
    """Search the customer support knowledge base for relevant information.

    Use this tool when the customer asks about products, policies, returns,
    shipping, billing, troubleshooting, or any topic that may have a
    documented answer.

    Args:
        query: The customer's question or search terms
        num_results: Number of passages to retrieve (default: 5)

    Returns:
        Dict with 'passages' list, each containing 'text' and 'source'
    """
    client = boto3.client("bedrock-agent-runtime", region_name=config.aws_region)
    response = client.retrieve(
        knowledgeBaseId=config.knowledge_base_id,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": num_results
            }
        },
    )
    passages = []
    for result in response.get("retrievalResults", []):
        passages.append({
            "text": result["content"]["text"],
            "source": result.get("location", {}).get("s3Location", {}).get("uri", "unknown"),
            "score": result.get("score", 0),
        })
    return {"passages": passages, "query": query}
```

### Memory Integration

**STM (Short-Term Memory):**
- On session start → load recent STM events for this session from AgentCore Memory
- Inject as conversation context into the agent
- After each turn → save `(user_message, assistant_response)` as STM event
- Strands Agent's built-in message list handles within-invoke context

**LTM (Long-Term Memory):**
- On session start → semantic search LTM for user context using `actor_id`
- Inject as `## Returning Customer Context` in system prompt
- AgentCore Memory auto-extracts key facts from STM events into LTM
- Namespaced by `/actors/{actor_id}/` for user isolation

```python
def load_memory_context(session_id: str, actor_id: str) -> str:
    """Load STM and LTM context for system prompt injection."""
    stm_events = memory_client.get_stm_events(session_id=session_id, limit=20)
    ltm_records = memory_client.search_ltm(actor_id=actor_id, query="customer context")

    context_parts = []
    if ltm_records:
        context_parts.append("## Returning Customer Context")
        for record in ltm_records:
            context_parts.append(f"- {record['content']}")

    if stm_events:
        context_parts.append("## Recent Conversation")
        for event in stm_events:
            context_parts.append(f"{event['role']}: {event['content']}")

    return "\n".join(context_parts)
```

## Coding Conventions

- **Type hints** on all function signatures
- **`logging`** module only — never `print()`
- **ruff** for linting: line-length 100, target py311
- **`@tool`** decorator from `strands` for all agent tools
- **Google-style docstrings** on all public functions
- **Environment variables** for all configuration — zero hardcoded secrets
- **Error handling in tools**: Return helpful error dicts, never raise exceptions to the agent
- **f-strings** preferred over `.format()` or `%`

## Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `MODEL_ID` | Bedrock model ID | No | `us.anthropic.claude-sonnet-4-20250514-v1:0` |
| `KNOWLEDGE_BASE_ID` | Bedrock Knowledge Base ID | **Yes** | — |
| `MEMORY_ID` | AgentCore Memory ID | No | (optional, disables memory if unset) |
| `AWS_REGION` | AWS region | No | `us-west-2` |
| `LOG_LEVEL` | Logging level | No | `INFO` |
| `MAX_SESSIONS` | Max concurrent agent sessions in pool | No | `100` |

## Dependencies (requirements.txt)

```
strands-agents>=0.1.0
strands-agents-tools>=0.1.0
boto3>=1.35.0
bedrock-agentcore>=0.1.0
pydantic>=2.0
```

## Testing

- Unit tests: `pytest tests/`
- Use `unittest.mock` / `moto` for Bedrock and AgentCore mocks
- Integration tests require real AWS credentials
- Local testing: `python scripts/local_test.py` (interactive CLI chat)

## Do NOT

- Do not add a database (PostgreSQL, DynamoDB, SQLite) — memory lives in AgentCore
- Do not create sub-agents or multi-agent orchestration
- Do not use `RetrieveAndGenerate` — use `Retrieve` only
- Do not hardcode Knowledge Base IDs, Memory IDs, or model IDs
- Do not skip error handling in tools
- Do not use `print()` — use `logging`
- Do not store conversation history in local files
- Do not add FastAPI/Flask — AgentCore Runtime provides the HTTP layer
