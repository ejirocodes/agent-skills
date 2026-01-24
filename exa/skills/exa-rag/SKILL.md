---
name: exa-rag
description: "Build RAG pipelines with Exa.ai for real-time web retrieval. Use when building retrieval-augmented generation, integrating Exa with LangChain, LlamaIndex, Vercel AI SDK, or implementing AI agents with web search capabilities. Triggers on: RAG pipeline, retrieval augmented generation, Exa LangChain, Exa LlamaIndex, ExaSearchRetriever, ExaSearchResults, Exa MCP, Exa tool calling, Claude tool use, AI agent web search, grounded generation, citation generation, fact checking, hallucination detection, OpenAI compatibility, chat completions."
license: MIT
metadata:
  author: ejirocodes
  version: '1.0.0'
---

# Exa RAG Integration

## Quick Reference

| Topic | When to Use | Reference |
|-------|-------------|-----------|
| **LangChain** | Building RAG chains with LangChain | [langchain.md](references/langchain.md) |
| **LlamaIndex** | Using Exa as a LlamaIndex data source | [llamaindex.md](references/llamaindex.md) |
| **Vercel AI SDK** | Adding web search to Next.js AI apps | [vercel-ai.md](references/vercel-ai.md) |
| **MCP & Tools** | Claude MCP server, OpenAI tools, function calling | [mcp-tools.md](references/mcp-tools.md) |

## Essential Patterns

### LangChain Retriever

```python
from langchain_exa import ExaSearchRetriever

retriever = ExaSearchRetriever(
    exa_api_key="your-key",
    k=5,
    highlights=True
)

docs = retriever.invoke("latest AI research papers")
```

### LlamaIndex Reader

```python
from llama_index.readers.web import ExaReader

reader = ExaReader(api_key="your-key")
documents = reader.load_data(
    query="machine learning best practices",
    num_results=10
)
```

### Vercel AI SDK Tool

```typescript
import { exa } from "@agentic/exa";
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";

const result = await generateText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  prompt: "Search for the latest TypeScript features",
});
```

### OpenAI-Compatible Endpoint

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.exa.ai/v1",
    api_key="your-exa-key"
)

response = client.chat.completions.create(
    model="exa",
    messages=[{"role": "user", "content": "What are the latest AI trends?"}]
)
```

## Integration Selection

| Framework | Best For | Key Feature |
|-----------|----------|-------------|
| **LangChain** | Complex chains, agents | ExaSearchRetriever, tool integration |
| **LlamaIndex** | Document indexing, Q&A | ExaReader, query engines |
| **Vercel AI SDK** | Next.js apps, streaming | Tool definitions, edge-ready |
| **OpenAI Compat** | Drop-in replacement | Minimal code changes |
| **Claude MCP** | Claude Desktop, Claude Code | Native tool calling |

## Common Mistakes

1. **Not using highlights for RAG** - Full text wastes context; use `highlights=True` for relevant snippets
2. **Missing source attribution** - Always include `result.url` in citations for grounded responses
3. **Ignoring summaries** - `summary=True` provides concise context without full page overhead
4. **Over-fetching results** - Start with 3-5 results; more isn't always better for RAG quality
5. **Not filtering domains** - Use `include_domains` to limit to authoritative sources
6. **Skipping date filters** - For current events, always add `start_published_date` to avoid stale info
7. **Forgetting async patterns** - Use async retrievers in production for better throughput
