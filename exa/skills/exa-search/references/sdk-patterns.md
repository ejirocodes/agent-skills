# SDK Patterns Reference

## Table of Contents

- [Installation](#installation)
- [Authentication](#authentication)
- [Python SDK (exa_py)](#python-sdk-exa_py)
- [TypeScript SDK (exa-js)](#typescript-sdk-exa-js)
- [Error Handling](#error-handling)
- [Response Objects](#response-objects)

---

## Installation

### Python

```bash
pip install exa_py
```

### TypeScript/JavaScript

```bash
npm install exa-js
# or
yarn add exa-js
# or
pnpm add exa-js
```

---

## Authentication

### Environment Variable (Recommended)

```bash
export EXA_API_KEY="your-api-key"
```

### Python

```python
from exa_py import Exa

# Uses EXA_API_KEY env var automatically
exa = Exa()

# Or pass explicitly
exa = Exa(api_key="your-api-key")
```

### TypeScript

```typescript
import Exa from "exa-js";

// Uses EXA_API_KEY env var automatically
const exa = new Exa();

// Or pass explicitly
const exa = new Exa("your-api-key");
```

---

## Python SDK (exa_py)

### Search Methods

```python
from exa_py import Exa

exa = Exa()

# Search only (returns URLs, no content)
results = exa.search(
    "query",
    type="auto",
    num_results=10
)

# Search with contents (most common)
results = exa.search_and_contents(
    "query",
    type="auto",
    num_results=10,
    text=True,
    highlights=True,
    summary=True
)

# Find similar to a URL
similar = exa.find_similar(
    "https://example.com/article",
    num_results=10
)

# Find similar with contents
similar = exa.find_similar_and_contents(
    "https://example.com/article",
    num_results=10,
    text=True,
    exclude_source_domain=True
)

# Get contents for known URLs
contents = exa.get_contents(
    urls=["https://example.com/page1", "https://example.com/page2"],
    text=True
)
```

### Async Support

```python
import asyncio
from exa_py import Exa

async def search_async():
    exa = Exa()

    # All methods have async versions
    results = await exa.search_and_contents_async(
        "async programming patterns",
        num_results=10,
        text=True
    )
    return results

results = asyncio.run(search_async())
```

### Parameter Naming Convention

Python SDK uses **snake_case**:
- `num_results`
- `include_domains`
- `exclude_domains`
- `start_published_date`
- `include_text`
- `max_characters`

---

## TypeScript SDK (exa-js)

### Search Methods

```typescript
import Exa from "exa-js";

const exa = new Exa();

// Search only
const results = await exa.search("query", {
  type: "auto",
  numResults: 10,
});

// Search with contents
const results = await exa.searchAndContents("query", {
  type: "auto",
  numResults: 10,
  text: true,
  highlights: true,
  summary: true,
});

// Find similar
const similar = await exa.findSimilar("https://example.com/article", {
  numResults: 10,
});

// Find similar with contents
const similar = await exa.findSimilarAndContents(
  "https://example.com/article",
  {
    numResults: 10,
    text: true,
    excludeSourceDomain: true,
  }
);

// Get contents
const contents = await exa.getContents(
  ["https://example.com/page1", "https://example.com/page2"],
  { text: true }
);
```

### Parameter Naming Convention

TypeScript SDK uses **camelCase**:
- `numResults`
- `includeDomains`
- `excludeDomains`
- `startPublishedDate`
- `includeText`
- `maxCharacters`

---

## Error Handling

### Python

```python
from exa_py import Exa
from exa_py.exceptions import ExaError, RateLimitError, AuthenticationError

exa = Exa()

try:
    results = exa.search_and_contents("query", num_results=10)
except AuthenticationError:
    print("Invalid API key")
except RateLimitError:
    print("Rate limit exceeded, retry later")
except ExaError as e:
    print(f"Exa error: {e}")
```

### TypeScript

```typescript
import Exa from "exa-js";

const exa = new Exa();

try {
  const results = await exa.searchAndContents("query", { numResults: 10 });
} catch (error) {
  if (error.status === 401) {
    console.log("Invalid API key");
  } else if (error.status === 429) {
    console.log("Rate limit exceeded");
  } else {
    console.log(`Error: ${error.message}`);
  }
}
```

---

## Response Objects

### Search Result Structure

```python
# Python
result.url         # str: Page URL
result.title       # str: Page title
result.score       # float: Relevance score (neural only)
result.published_date  # str: Publication date
result.author      # str: Author if available
result.text        # str: Full text (if requested)
result.highlights  # list[str]: Relevant snippets (if requested)
result.summary     # str: AI summary (if requested)
```

```typescript
// TypeScript
result.url; // string
result.title; // string
result.score; // number (neural only)
result.publishedDate; // string
result.author; // string
result.text; // string (if requested)
result.highlights; // string[] (if requested)
result.summary; // string (if requested)
```

### Search Response Structure

```python
# Python
response.results           # list[Result]: Search results
response.autoprompt_string # str: (deprecated)
response.resolved_search_type  # str: "neural" or "keyword" (auto mode)
```

```typescript
// TypeScript
response.results; // Result[]
response.autopromptString; // string (deprecated)
response.resolvedSearchType; // string
```

---

## Common Patterns

### Pagination

```python
# Exa doesn't support offset pagination
# Instead, use date filters or increase num_results
results = exa.search_and_contents(
    "query",
    num_results=50,  # Get more results at once
    text=True
)
```

### Batch Processing

```python
# Process multiple queries
queries = ["AI trends", "ML frameworks", "data science tools"]

all_results = []
for query in queries:
    results = exa.search_and_contents(query, num_results=5, text=True)
    all_results.extend(results.results)
```

### Retry Logic

```python
import time
from exa_py import Exa
from exa_py.exceptions import RateLimitError

def search_with_retry(query, max_retries=3):
    exa = Exa()
    for attempt in range(max_retries):
        try:
            return exa.search_and_contents(query, num_results=10, text=True)
        except RateLimitError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```
