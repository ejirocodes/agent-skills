---
name: exa-research
description: "Exa.ai deep research and answer generation with citations. Use when building research automation, implementing Answer API for Q&A with sources, creating research reports, or using deep search with summaries. Triggers on: Exa Answer, answer endpoint, exa.answer, deep search, research API, Exa Research, async research, research report, citation extraction, summarization with sources, fact verification, streaming answers, research tasks."
license: MIT
metadata:
  author: ejirocodes
  version: '1.0.0'
---

# Exa Research & Answer API

## Quick Reference

| Topic | When to Use | Reference |
|-------|-------------|-----------|
| **Answer API** | Q&A with citations, grounded responses | [answer-api.md](references/answer-api.md) |
| **Deep Search** | Smart query expansion, high-quality summaries | [deep-search.md](references/deep-search.md) |
| **Citations** | Source attribution, verification | [citations.md](references/citations.md) |

## Essential Patterns

### Answer API (Python)

```python
from exa_py import Exa

exa = Exa()

response = exa.answer(
    "What are the key features of Python 3.12?",
    text=True
)

print(response.answer)
for citation in response.citations:
    print(f"Source: {citation.url}")
```

### Streaming Answers

```python
stream = exa.answer(
    "Explain the benefits of microservices architecture",
    stream=True
)

for chunk in stream:
    print(chunk.text, end="", flush=True)

# Access citations after streaming
print("\nSources:", stream.citations)
```

### Deep Search with Summaries

```python
results = exa.search_and_contents(
    "latest developments in quantum computing",
    type="neural",
    num_results=10,
    summary=True,
    use_autoprompt=True  # Smart query expansion
)

for result in results.results:
    print(f"{result.title}")
    print(f"Summary: {result.summary}")
```

## When to Use

| Feature | Use Case | Output |
|---------|----------|--------|
| **Answer API** | Direct Q&A needing citations | Answer + source URLs |
| **Deep Search** | Query expansion + summaries | Enhanced search results |
| **Exa Research** | Long-form async reports | Structured JSON/Markdown |

## Common Mistakes

1. **Not using streaming for long answers** - Use `stream=True` for better UX on complex questions
2. **Ignoring citations** - Always include `response.citations` for verifiable responses
3. **Missing `text=True`** - Answer API needs content access; include `text=True`
4. **Over-complex queries** - Answer API works best with clear, focused questions
5. **Not validating citations** - Check `citation.url` exists before displaying to users
