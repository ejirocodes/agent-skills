# Answer API Reference

## Table of Contents

- [Basic Usage](#basic-usage)
- [Streaming Responses](#streaming-responses)
- [Configuration Options](#configuration-options)
- [Response Structure](#response-structure)
- [Best Practices](#best-practices)

---

## Basic Usage

The Answer API generates LLM responses with citations from web search results.

### Python

```python
from exa_py import Exa

exa = Exa()  # Uses EXA_API_KEY env var

response = exa.answer(
    "What are the key differences between React and Vue?",
    text=True
)

print("Answer:", response.answer)
print("\nSources:")
for citation in response.citations:
    print(f"- {citation.title}: {citation.url}")
```

### TypeScript

```typescript
import Exa from "exa-js";

const exa = new Exa();

const response = await exa.answer(
  "What are the key differences between React and Vue?",
  { text: true }
);

console.log("Answer:", response.answer);
console.log("Sources:", response.citations.map((c) => c.url));
```

### With Search Filters

```python
response = exa.answer(
    "What's new in TypeScript 5.4?",
    text=True,
    include_domains=["devblogs.microsoft.com", "typescriptlang.org"],
    start_published_date="2024-01-01"
)
```

---

## Streaming Responses

For better UX on complex questions, use streaming.

### Python Streaming

```python
stream = exa.answer(
    "Explain the transformer architecture in detail",
    text=True,
    stream=True
)

# Stream the answer text
for chunk in stream:
    if chunk.text:
        print(chunk.text, end="", flush=True)

# Access citations after streaming completes
print("\n\nSources:")
for citation in stream.citations:
    print(f"- {citation.url}")
```

### TypeScript Streaming

```typescript
const stream = await exa.answer(
  "Explain the transformer architecture in detail",
  { text: true, stream: true }
);

for await (const chunk of stream) {
  if (chunk.text) {
    process.stdout.write(chunk.text);
  }
}

// Citations available after stream ends
console.log("\nSources:", stream.citations);
```

### Streaming with Progress

```python
def answer_with_progress(question):
    stream = exa.answer(question, text=True, stream=True)

    full_answer = ""
    for chunk in stream:
        if chunk.text:
            full_answer += chunk.text
            print(chunk.text, end="", flush=True)

    return {
        "answer": full_answer,
        "citations": stream.citations
    }
```

---

## Configuration Options

### Available Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | str | The question to answer |
| `text` | bool | Include full text from sources |
| `stream` | bool | Enable streaming response |
| `include_domains` | list | Limit sources to these domains |
| `exclude_domains` | list | Exclude these domains |
| `start_published_date` | str | Minimum publish date (YYYY-MM-DD) |
| `end_published_date` | str | Maximum publish date |
| `num_results` | int | Number of sources to consider |

### Example with All Options

```python
response = exa.answer(
    "What are the latest best practices for API security?",
    text=True,
    stream=False,
    include_domains=["owasp.org", "auth0.com", "cloudflare.com"],
    start_published_date="2024-01-01",
    num_results=10
)
```

---

## Response Structure

### Non-Streaming Response

```python
response = exa.answer("question", text=True)

# Answer text
response.answer  # str: The generated answer

# Citations
response.citations  # list[Citation]

# Each citation
for citation in response.citations:
    citation.url        # str: Source URL
    citation.title      # str: Page title
    citation.text       # str: Relevant text (if text=True)
    citation.highlights # list[str]: Key snippets
```

### Streaming Response

```python
stream = exa.answer("question", text=True, stream=True)

# Iterate chunks
for chunk in stream:
    chunk.text  # str: Partial answer text

# After iteration
stream.citations  # list[Citation]: Available after stream ends
stream.answer     # str: Complete answer
```

---

## Best Practices

### Ask Focused Questions

```python
# Good: Specific, focused question
response = exa.answer(
    "What are the three main benefits of using TypeScript over JavaScript?",
    text=True
)

# Avoid: Too broad
response = exa.answer(
    "Tell me everything about TypeScript",
    text=True
)
```

### Filter for Authoritative Sources

```python
response = exa.answer(
    "What are React Server Components?",
    text=True,
    include_domains=["react.dev", "nextjs.org", "vercel.com"]
)
```

### Handle Missing Citations

```python
response = exa.answer("question", text=True)

if response.citations:
    print("Answer:", response.answer)
    print("Sources:")
    for c in response.citations:
        print(f"- {c.url}")
else:
    print("Answer (unverified):", response.answer)
    print("Warning: No citations available")
```

### Combine with Follow-up Search

```python
def research_topic(question):
    # Get initial answer
    answer = exa.answer(question, text=True)

    # If more detail needed, search for related content
    if len(answer.citations) < 3:
        additional = exa.search_and_contents(
            question,
            num_results=5,
            text=True,
            exclude_domains=[c.url for c in answer.citations]
        )
        return {
            "answer": answer.answer,
            "primary_sources": answer.citations,
            "additional_sources": additional.results
        }

    return {"answer": answer.answer, "sources": answer.citations}
```

### Error Handling

```python
from exa_py.exceptions import ExaError

try:
    response = exa.answer(
        "What is the latest version of Node.js?",
        text=True
    )
except ExaError as e:
    print(f"Answer API error: {e}")
    # Fall back to regular search
    results = exa.search_and_contents(
        "latest Node.js version",
        num_results=5,
        text=True
    )
```
