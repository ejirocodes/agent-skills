# Search Modes Reference

## Table of Contents

- [Auto Search (Default)](#auto-search-default)
- [Neural Search](#neural-search)
- [Keyword Search](#keyword-search)
- [Fast Search](#fast-search)
- [Mode Comparison](#mode-comparison)

---

## Auto Search (Default)

Auto mode intelligently selects between neural and keyword search based on query analysis.

### Usage

```python
# Python - auto is default
results = exa.search_and_contents(
    "best practices for React state management",
    type="auto",  # or omit, auto is default
    num_results=10
)
```

```typescript
// TypeScript - auto is default
const results = await exa.searchAndContents(
  "best practices for React state management",
  { type: "auto", numResults: 10 }
);
```

### Characteristics

- Automatically detects query intent
- Switches between neural and keyword based on query structure
- Best for general-purpose applications
- **Note**: Scores are not returned in auto mode

---

## Neural Search

Neural search uses semantic understanding to find conceptually relevant results.

### When to Use

- Natural language questions ("What is...", "How do I...")
- Topic exploration and research
- Finding related concepts
- Queries where exact terms may vary

### Usage

```python
results = exa.search_and_contents(
    "explain the benefits of microservices architecture",
    type="neural",
    num_results=10
)

# Neural search returns relevance scores
for result in results.results:
    print(f"Score: {result.score} - {result.title}")
```

```typescript
const results = await exa.searchAndContents(
  "explain the benefits of microservices architecture",
  { type: "neural", numResults: 10 }
);

results.results.forEach((r) => console.log(`Score: ${r.score} - ${r.title}`));
```

### Best Practices

- Use for open-ended, exploratory queries
- Leverage scores to rank results by relevance
- Combine with `summary=True` for RAG applications

---

## Keyword Search

Keyword search finds exact term matches, similar to traditional search engines.

### When to Use

- Specific product or company names
- Error codes and technical identifiers
- Exact phrase matching
- Proper nouns and named entities

### Usage

```python
results = exa.search_and_contents(
    "ECONNREFUSED Node.js",
    type="keyword",
    num_results=10
)
```

```typescript
const results = await exa.searchAndContents("ECONNREFUSED Node.js", {
  type: "keyword",
  numResults: 10,
});
```

### Best Practices

- Use for specific technical queries
- Ideal when you know exact terms to match
- Combine with domain filters for targeted searches

---

## Fast Search

Fast search prioritizes speed over comprehensiveness.

### Usage

```python
# Use the fast search endpoint
results = exa.search(
    "breaking news technology",
    type="auto",
    num_results=5
)
# Note: search() is faster than search_and_contents()
```

### When to Use

- Real-time applications requiring low latency
- When you only need URLs, not content
- High-volume search applications

---

## Mode Comparison

| Feature | Auto | Neural | Keyword |
|---------|------|--------|---------|
| Semantic understanding | Yes (when neural selected) | Yes | No |
| Exact matching | Yes (when keyword selected) | Limited | Yes |
| Returns scores | No | Yes | No |
| Best for questions | Yes | Yes | No |
| Best for exact terms | Yes | No | Yes |
| Default mode | Yes | No | No |

### Query Examples by Mode

| Query | Recommended Mode | Reason |
|-------|-----------------|--------|
| "What is retrieval augmented generation" | neural/auto | Conceptual question |
| "OpenAI GPT-4 API pricing" | keyword/auto | Specific product terms |
| "best practices authentication" | neural/auto | Exploratory topic |
| "CORS error localhost:3000" | keyword | Technical error code |
| "Tesla Q4 2024 earnings" | keyword/auto | Specific entity + timeframe |
