# SPEC.md — Functional Specification

## Customer Support Agent — Multi-Turn with Memory & RAG

### 1. Purpose

An AI-powered customer support agent that:
- Answers customer questions using a RAG-powered knowledge base (Amazon Bedrock Knowledge Bases)
- Maintains multi-turn conversation context within a session (STM)
- Remembers returning customers across sessions with long-term memory (LTM)
- Runs on Amazon Bedrock AgentCore Runtime

This is a **foundation customer support agent** — focused on RAG + Memory + Conversation. It can be extended with ticketing, escalation, and other tools later.

### 2. User Personas

| Persona | Description |
|---------|-------------|
| **Customer** | End user seeking support via chat. May be new or returning. |
| **Support Admin** | Manages FAQ/policy content in the knowledge base. Not a direct user of the agent. |

### 3. Core Capabilities

#### 3.1 Knowledge Base Retrieval (RAG)

- The agent searches Amazon Bedrock Knowledge Bases for every customer question about products, policies, billing, shipping, troubleshooting
- Retrieved passages ground the response — the agent cites sources when answering from KB
- If the KB has relevant content → respond with grounded answer + source attribution
- If no relevant content found → respond honestly, offer to help another way
- The agent never fabricates facts that should come from the KB

**Knowledge Base Content Types:**
- Product FAQs
- Troubleshooting guides
- Policy documents (returns, shipping, billing)
- How-to articles
- Pricing and plan information

**Knowledge Base Details:**
- Backend: Amazon Bedrock Knowledge Bases
- Embedding model: Amazon Titan Embeddings V2
- Vector store: Managed by Bedrock KB (OpenSearch Serverless)
- Data source: S3 bucket with documents (Markdown, PDF, HTML, TXT)
- Retrieval method: `Retrieve` API (retrieval-only, agent controls generation)
- Default result count: 5 passages

#### 3.2 Multi-Turn Conversation Memory

The memory system has two layers:

**Short-Term Memory (STM) — Within a Session:**
- Full conversation history maintained by the Strands Agent's message list
- Additionally persisted to AgentCore Memory as STM events
- Enables natural follow-ups: "What about the other option?" / "Can you explain that differently?"
- Loaded on session resume to restore context
- Retention: 30 days

**Long-Term Memory (LTM) — Across Sessions:**
- Cross-session customer context stored in AgentCore Memory LTM
- AgentCore automatically extracts key facts from STM events and consolidates into LTM
- Loaded at session start via semantic search on `actor_id`
- Injected into system prompt to personalize the experience
- Retention: Indefinite

**What LTM Remembers (auto-extracted by AgentCore):**
- Customer preferences and product interests
- Past issues and their resolutions
- Key facts the customer has shared (plan type, account details mentioned)
- Topics discussed in prior sessions

**Example Multi-Turn + Memory Flow:**
```
# Session 1
Customer: "I just bought the Pro plan but I can't access advanced features"
Agent: [searches KB → retrieves Pro plan feature activation guide]
       "After purchasing Pro, features activate within 24 hours. Here's how
        to verify: [steps from KB]. Let me know if it's been longer than that."

Customer: "It's been 3 days actually"
Agent: [builds on session context, acknowledges the timeline is unusual]
       "3 days is definitely outside the normal window. Based on our
        troubleshooting guide, try clearing your cache and logging out/in.
        If that doesn't work, this may need manual activation."

# Session 2 (days later)
Customer: "Hi, I'm back about that same issue"
Agent: [loads LTM: Pro plan customer, had feature activation issue 3 days after purchase]
       "Welcome back! I remember you were having trouble with Pro plan
        features not activating. Were you able to resolve it with the
        steps we discussed, or is it still an issue?"
```

#### 3.3 Conversational Behavior

The agent should:
- Be helpful, empathetic, and professional
- Search the KB proactively for any customer question that might have a documented answer
- Acknowledge when it's using general knowledge vs. KB-sourced information
- Ask clarifying questions when the issue is ambiguous
- Reference previous turns naturally ("As I mentioned earlier...")
- Use LTM to personalize without being intrusive (don't dump everything it remembers)
- Handle graceful degradation: if KB is unavailable, still converse; if memory is unavailable, still work
- Always ask if there's anything else the customer needs before closing

### 4. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Amazon Bedrock AgentCore                  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              AgentCore Runtime                     │  │
│  │                                                   │  │
│  │   entrypoint.py                                   │  │
│  │     └── AgentPool (LRU, max 100 sessions)         │  │
│  │           └── Strands Agent (per session)          │  │
│  │                 ├── Tools: [kb_retrieve]           │  │
│  │                 ├── Model: Bedrock Claude Sonnet   │  │
│  │                 └── Memory: AgentCore STM+LTM      │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌──────────────┐
    │  Bedrock   │ │  Bedrock   │ │  AgentCore   │
    │ Knowledge  │ │   LLM      │ │   Memory     │
    │   Base     │ │  (Claude)  │ │  (STM+LTM)   │
    └────────────┘ └────────────┘ └──────────────┘
```

### 5. Agent Configuration

**Model:**
- `us.anthropic.claude-sonnet-4-20250514-v1:0` via `strands.models.bedrock.BedrockModel`

**Tools:**
| Tool | Purpose | When to Use |
|------|---------|-------------|
| `kb_retrieve` | Search Bedrock Knowledge Base | When customer asks about products, policies, troubleshooting, billing, shipping, or any topic that may have a documented answer |

**System Prompt Structure:**
```
[Core Identity]
You are a customer support agent for [Company]. Your goal is to resolve
customer issues quickly and accurately using the knowledge base.

[Behavior Rules]
1. ALWAYS search the knowledge base first for product/policy/troubleshooting questions
2. Cite sources when answering from KB content
3. Clearly distinguish KB-grounded answers from general knowledge
4. Never fabricate information — if you don't know, say so honestly
5. Be empathetic and professional
6. Use memory context naturally — don't recite it back verbatim
7. Ask clarifying questions when the issue is ambiguous
8. Always ask if there's anything else before closing

[Memory Context — dynamically injected]
## Returning Customer Context
- [LTM records loaded at session start]

## Recent Conversation
- [STM events if resuming a session]
```

### 6. API Contract

**Invoke Request:**
```json
{
  "message": "I can't find the return policy for electronics",
  "actor_id": "customer-12345"
}
```

**Invoke Response:**
```json
{
  "result": "Based on our knowledge base, the return policy for electronics is..."
}
```

**Context (provided by AgentCore Runtime):**
- `context.session_id` — unique session identifier
- `context.actor_id` — customer identifier for memory namespacing

### 7. Conversation Session Lifecycle

```
1. REQUEST ARRIVES
   └── Extract session_id, actor_id, message

2. ENSURE INITIALIZED (lazy, first-call only)
   └── Create Bedrock model client, memory client, KB client

3. GET OR CREATE AGENT
   └── AgentPool.get_or_create(session_id)
   └── If new session:
       a. Load LTM for actor_id (semantic search)
       b. Load STM for session_id (if resuming)
       c. Assemble system prompt with memory context
       d. Create Strands Agent instance

4. INVOKE AGENT
   └── agent(message) → result
   └── Agent may call kb_retrieve tool during reasoning

5. SAVE TO MEMORY
   └── Save (user_message, assistant_response) as STM event
   └── AgentCore handles LTM extraction automatically

6. RETURN RESPONSE
```

### 8. Error Handling

| Error | Behavior |
|-------|----------|
| KB retrieval fails (timeout, service error) | Agent acknowledges, answers from general knowledge if possible, logs warning |
| Memory load fails | Agent operates without memory context, logs warning |
| Memory save fails | Response still returned, log warning (non-blocking) |
| LLM timeout | Return graceful timeout message |
| Invalid input (empty message) | Return helpful error message |

Key principle: **Never block the response due to a non-critical failure.** KB and memory are enhancements — the agent should degrade gracefully.

### 9. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Response latency (p95) | < 5 seconds |
| Concurrent sessions | 100+ (AgentPool limit) |
| STM retention | 30 days |
| LTM retention | Indefinite |
| Memory data isolation | Per actor_id namespace |
| Startup time | < 30 seconds (lazy init) |

### 10. Knowledge Base Setup

The repo includes a `knowledge-base/` directory with:
- `README.md` — Step-by-step instructions to create a Bedrock Knowledge Base
- `sample-faqs/` — Sample customer support FAQ documents (markdown)
- `setup_kb.py` — Python script to create KB, attach S3 data source, and trigger sync

**Steps:**
1. Create S3 bucket → upload FAQ/policy documents
2. Run `setup_kb.py` → creates Bedrock KB with Titan Embeddings V2
3. Set `KNOWLEDGE_BASE_ID` environment variable
4. Agent automatically uses it via `kb_retrieve` tool

**Sample FAQ Topics to Include:**
- Return and refund policy
- Shipping timelines and tracking
- Billing and subscription management
- Account access and password reset
- Product feature guides
- Common troubleshooting steps

### 11. Local Development & Testing

**Local CLI chat:**
```bash
export KNOWLEDGE_BASE_ID=your-kb-id
export AWS_REGION=us-west-2
python scripts/local_test.py
```

The local test script should:
- Create a single Agent instance (no AgentPool needed locally)
- Run an interactive `while True` chat loop with `input()`
- Support `/quit` to exit, `/clear` to reset conversation
- Optionally load/save memory if `MEMORY_ID` is set
- Work without memory (for quick testing without AgentCore)

### 12. File-by-File Build Order

Recommended sequence for Claude Code to build:

1. `src/customer_support/__init__.py` — Package init
2. `src/customer_support/config.py` — Environment config with defaults and validation
3. `src/customer_support/tools/__init__.py` + `src/customer_support/tools/kb_retrieval.py` — KB retrieval tool
4. `src/customer_support/memory/__init__.py` + `src/customer_support/memory/agentcore_memory.py` — Memory integration (load STM, load LTM, save turn)
5. `src/customer_support/prompts/system_prompt.py` — System prompt assembly with memory injection
6. `src/customer_support/agent.py` — Agent factory (ties model + tools + prompt together)
7. `entrypoint.py` — AgentCore Runtime entry point with AgentPool and lazy init
8. `scripts/local_test.py` — Local CLI testing harness
9. `requirements.txt` + `pyproject.toml` — Dependencies and project metadata
10. `Dockerfile` + `.bedrock_agentcore.yaml` — Deployment config
11. `knowledge-base/README.md` + `knowledge-base/setup_kb.py` + `knowledge-base/sample-faqs/` — KB setup and sample data
12. `tests/` — Unit tests with mocks for Bedrock and AgentCore

### 13. Future Extensions (Not in v1)

- **Ticket management** — Create/track support tickets (PostgreSQL)
- **Escalation to human** — Detect when AI can't resolve, hand off
- **Streaming responses** via AgentCore streaming protocol
- **Bedrock Guardrails** for content safety
- **Multi-language** — detect language, respond in kind
- **Proactive follow-up** — check in on unresolved issues
- **Analytics pipeline** — export conversation data for insights
- **Evaluation framework** — automated testing of response quality
