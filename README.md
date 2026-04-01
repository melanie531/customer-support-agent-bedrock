# Customer Support Agent (Bedrock)

Multi-turn customer support agent with persistent memory and RAG-powered FAQ, built with Strands Agents SDK, Amazon Bedrock Knowledge Bases, and AgentCore Memory.

## Quick Start

```bash
# Read the steering docs first
cat CLAUDE.md
cat SPEC.md

# Then let Claude Code build it
# "Read CLAUDE.md and SPEC.md, then build the project following the build order in SPEC.md §12"
```

## Architecture

- **Agent Framework**: Strands Agents SDK
- **LLM**: Claude Sonnet on Amazon Bedrock
- **RAG**: Amazon Bedrock Knowledge Bases (`Retrieve` API)
- **Memory**: Amazon Bedrock AgentCore Memory (STM + LTM)
- **Runtime**: Amazon Bedrock AgentCore Runtime (container-based)
