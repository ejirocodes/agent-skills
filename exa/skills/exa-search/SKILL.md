---
name: exa-search
description: "Exa.ai search API integration for neural and keyword web search with content retrieval. Use when implementing web search features, integrating Exa SDK (exa_py, exa-js), or retrieving web content. Triggers on: Exa, exa_py, exa-js, neural search, web search API, search_and_contents, searchAndContents, find_similar, findSimilar, domain filtering, date filtering, text extraction, page summaries, highlights, search auto mode, fast search, search categories, livecrawl, excluding domains, include text, exclude text, EXA_API_KEY."
license: MIT
metadata:
  author: ejirocodes
  version: '1.0.0'
---

# Exa Search Integration

## Quick Reference

| Topic | When to Use | Reference |
|-------|-------------|-----------|
| **Search Modes** | Choosing between auto, neural, and keyword search | [search-modes.md](references/search-modes.md) |
| **Filters** | Domain, date, text, and category filtering | [filters.md](references/filters.md) |
| **Contents** | Text extraction, highlights, summaries, livecrawl | [contents.md](references/contents.md) |
| **SDK Patterns** | Python (exa_py) and TypeScript (exa-js) usage | [sdk-patterns.md](references/sdk-patterns.md) |

## Essential Patterns

### Basic Search (Python)

```python
from exa_py import Exa

exa = Exa(api_key="your-api-key")  # or set EXA_API_KEY env var

results = exa.search_and_contents(
    "latest developments in quantum computing",
    type="auto",
    num_results=10,
    text=True,
    highlights=True
)

for result in results.results:
    print(f"{result.title}: {result.url}")
    print(result.text[:500])
```

### Basic Search (TypeScript)

```typescript
import Exa from "exa-js";

const exa = new Exa(process.env.EXA_API_KEY);

const results = await exa.searchAndContents(
  "latest developments in quantum computing",
  {
    type: "auto",
    numResults: 10,
    text: true,
    highlights: true,
  }
);

results.results.forEach((result) => {
  console.log(`${result.title}: ${result.url}`);
});
```

### Search with Filters

```python
results = exa.search_and_contents(
    "AI startup funding rounds",
    type="neural",
    num_results=10,
    include_domains=["techcrunch.com", "venturebeat.com"],
    start_published_date="2024-01-01",
    text={"max_characters": 2000},
    summary=True
)
```

### Find Similar Links

```python
similar = exa.find_similar_and_contents(
    "https://example.com/interesting-article",
    num_results=10,
    exclude_source_domain=True,
    text=True
)
```

## Search Mode Selection

| Mode | When to Use | Notes |
|------|-------------|-------|
| `auto` | Default for most queries | Exa optimizes between neural/keyword automatically |
| `neural` | Natural language, conceptual queries | Best for "what is...", "how to...", topic exploration |
| `keyword` | Exact matches, technical terms, names | Best for specific product names, error codes, proper nouns |

## Common Mistakes

1. **Using `keyword` for conceptual queries** - Neural search understands intent better; use `auto` or `neural` for natural language questions
2. **Not setting `text=True`** - Search returns URLs only by default; explicitly request content with `text=True`
3. **Ignoring `highlights`** - Use `highlights=True` for relevant snippets without downloading full page text
4. **Missing API key** - Set `EXA_API_KEY` environment variable or pass explicitly to constructor
5. **Over-filtering initially** - Start with broad searches, then add domain/date filters to refine
6. **Not using `summary`** - For RAG applications, `summary=True` provides concise context without full page text
7. **Expecting scores in auto mode** - Relevance scores are only returned with `type="neural"`; auto mode doesn't include them
