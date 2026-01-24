# Contents Retrieval Reference

## Table of Contents

- [Content Options Overview](#content-options-overview)
- [Text Extraction](#text-extraction)
- [Highlights](#highlights)
- [Summaries](#summaries)
- [Livecrawl Options](#livecrawl-options)
- [Contents Endpoint](#contents-endpoint)

---

## Content Options Overview

When using `search_and_contents()` or `searchAndContents()`, specify what content to retrieve:

| Option | Type | Description |
|--------|------|-------------|
| `text` | bool or object | Full page text content |
| `highlights` | bool or object | Relevant snippets matching query |
| `summary` | bool | AI-generated page summary |

---

## Text Extraction

### Basic Text Retrieval

```python
results = exa.search_and_contents(
    "Python best practices",
    text=True,  # Returns full text
    num_results=5
)

for result in results.results:
    print(result.text)  # Full page content
```

### Limit Text Length

```python
# Limit to 1000 characters per result
results = exa.search_and_contents(
    "Python best practices",
    text={"max_characters": 1000},
    num_results=5
)
```

```typescript
const results = await exa.searchAndContents("Python best practices", {
  text: { maxCharacters: 1000 },
  numResults: 5,
});
```

### Text Format

Exa returns content in **Markdown format** by default, optimized for AI consumption.

```python
# Result text is markdown-formatted
print(result.text)
# Output: "# Page Title\n\nParagraph content...\n\n## Section..."
```

---

## Highlights

Highlights return the most relevant snippets from the page matching your query.

### Basic Highlights

```python
results = exa.search_and_contents(
    "React hooks tutorial",
    highlights=True,
    num_results=5
)

for result in results.results:
    for highlight in result.highlights:
        print(highlight)  # Relevant snippet
```

### Highlight Options

```python
results = exa.search_and_contents(
    "React hooks tutorial",
    highlights={
        "num_sentences": 3,  # Sentences per highlight
        "highlights_per_url": 3  # Max highlights per result
    },
    num_results=5
)
```

```typescript
const results = await exa.searchAndContents("React hooks tutorial", {
  highlights: {
    numSentences: 3,
    highlightsPerUrl: 3,
  },
  numResults: 5,
});
```

### When to Use Highlights

- RAG applications needing relevant context
- Search result previews
- When full text is too large
- Extracting key information quickly

---

## Summaries

AI-generated summaries of page content.

### Usage

```python
results = exa.search_and_contents(
    "machine learning frameworks comparison",
    summary=True,
    num_results=5
)

for result in results.results:
    print(result.summary)  # AI-generated summary
```

```typescript
const results = await exa.searchAndContents(
  "machine learning frameworks comparison",
  { summary: true, numResults: 5 }
);
```

### Best Practices

- Use for RAG when full context not needed
- Combine with highlights for comprehensive context
- More token-efficient than full text

---

## Livecrawl Options

Control how Exa fetches fresh content.

### Livecrawl Modes

| Mode | Description |
|------|-------------|
| `never` | Only use cached content |
| `fallback` | Use cache first, crawl if unavailable (default) |
| `preferred` | Prefer fresh crawl, fall back to cache |
| `always` | Always crawl fresh content |

### Usage

```python
# Always get fresh content
results = exa.search_and_contents(
    "breaking tech news",
    livecrawl="always",
    text=True,
    num_results=5
)

# Prefer fresh but accept cached
results = exa.search_and_contents(
    "company updates",
    livecrawl="preferred",
    text=True,
    num_results=5
)
```

```typescript
const results = await exa.searchAndContents("breaking tech news", {
  livecrawl: "always",
  text: true,
  numResults: 5,
});
```

### When to Use Each Mode

- `never`: Historical research, reproducible results
- `fallback`: General use, balance of speed and freshness
- `preferred`: News, rapidly changing content
- `always`: Real-time monitoring, critical freshness

---

## Contents Endpoint

Retrieve contents for known URLs without searching.

### Usage

```python
# Get contents for specific URLs
contents = exa.get_contents(
    urls=["https://example.com/article1", "https://example.com/article2"],
    text=True,
    summary=True
)

for content in contents.results:
    print(f"{content.title}: {content.summary}")
```

```typescript
const contents = await exa.getContents(
  ["https://example.com/article1", "https://example.com/article2"],
  { text: true, summary: true }
);
```

### Use Cases

- Fetch content for URLs from other sources
- Update cached content
- Batch content retrieval

### Response Status

Contents endpoint returns detailed status instead of HTTP errors:

```python
for content in contents.results:
    if content.text:
        print(content.text)
    else:
        print(f"Status: {content.status_code}")  # e.g., 404, timeout
```

---

## Combining Content Options

### Full Context Retrieval

```python
results = exa.search_and_contents(
    "GraphQL vs REST API comparison",
    text={"max_characters": 5000},
    highlights={"num_sentences": 2, "highlights_per_url": 3},
    summary=True,
    num_results=10
)

for result in results.results:
    print(f"Title: {result.title}")
    print(f"URL: {result.url}")
    print(f"Summary: {result.summary}")
    print("Highlights:")
    for h in result.highlights:
        print(f"  - {h}")
    print(f"Full text: {result.text[:500]}...")
```

### Token-Efficient Retrieval

```python
# For RAG with limited context window
results = exa.search_and_contents(
    "query",
    highlights={"num_sentences": 2, "highlights_per_url": 2},
    summary=True,
    # Skip full text to save tokens
    num_results=5
)
```
