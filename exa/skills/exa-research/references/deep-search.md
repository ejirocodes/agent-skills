# Deep Search Reference

## Table of Contents

- [Overview](#overview)
- [Smart Query Expansion](#smart-query-expansion)
- [Summaries](#summaries)
- [Deep Search Patterns](#deep-search-patterns)
- [Exa Research API](#exa-research-api)

---

## Overview

Deep Search provides enhanced search capabilities with:
- Smart query expansion (autoprompt)
- High-quality AI summaries
- Better results for complex queries

---

## Smart Query Expansion

Exa can automatically expand your query for better results.

### Autoprompt (Deprecated but Functional)

```python
from exa_py import Exa

exa = Exa()

# Autoprompt expands the query intelligently
results = exa.search_and_contents(
    "best practices for building APIs",
    type="neural",
    use_autoprompt=True,
    num_results=10,
    text=True
)

# Check expanded query
print(f"Expanded query: {results.autoprompt_string}")
```

### Manual Query Enhancement

```python
def enhance_query(base_query, context=None):
    """Enhance query for better search results."""
    if context:
        return f"{base_query} {context}"

    # Add common enhancers based on query type
    if "how to" in base_query.lower():
        return f"{base_query} tutorial guide example"
    elif "best" in base_query.lower():
        return f"{base_query} comparison review 2024"

    return base_query

results = exa.search_and_contents(
    enhance_query("how to implement OAuth"),
    type="neural",
    num_results=10
)
```

---

## Summaries

AI-generated summaries provide concise context without full page text.

### Basic Summary Usage

```python
results = exa.search_and_contents(
    "microservices architecture patterns",
    type="neural",
    num_results=10,
    summary=True  # Get AI-generated summaries
)

for result in results.results:
    print(f"Title: {result.title}")
    print(f"Summary: {result.summary}")
    print(f"URL: {result.url}\n")
```

### Summaries with Highlights

```python
results = exa.search_and_contents(
    "kubernetes deployment strategies",
    type="neural",
    num_results=10,
    summary=True,
    highlights=True
)

for result in results.results:
    print(f"Title: {result.title}")
    print(f"Summary: {result.summary}")
    print("Key points:")
    for h in result.highlights:
        print(f"  - {h}")
```

### Summary vs Text vs Highlights

| Feature | Size | Best For |
|---------|------|----------|
| `summary` | ~100 words | Quick overview, RAG context |
| `highlights` | ~50 words each | Specific relevant snippets |
| `text` | Full page | Complete content analysis |

```python
# Token-efficient: summaries only
results = exa.search_and_contents(query, summary=True, num_results=5)

# Balanced: summaries + highlights
results = exa.search_and_contents(query, summary=True, highlights=True, num_results=5)

# Complete: full text
results = exa.search_and_contents(query, text=True, num_results=5)
```

---

## Deep Search Patterns

### Research Deep Dive

```python
def deep_research(topic, num_results=20):
    """Comprehensive research on a topic."""
    results = exa.search_and_contents(
        topic,
        type="neural",
        num_results=num_results,
        summary=True,
        highlights=True,
        text={"max_characters": 2000}
    )

    research = {
        "topic": topic,
        "sources": [],
        "key_points": []
    }

    for result in results.results:
        research["sources"].append({
            "title": result.title,
            "url": result.url,
            "summary": result.summary,
            "key_points": result.highlights
        })
        research["key_points"].extend(result.highlights or [])

    return research
```

### Multi-Angle Research

```python
def multi_angle_research(topic):
    """Research topic from multiple angles."""
    angles = [
        f"{topic} benefits advantages",
        f"{topic} challenges problems",
        f"{topic} best practices",
        f"{topic} examples case studies"
    ]

    all_results = {}
    for angle in angles:
        results = exa.search_and_contents(
            angle,
            type="neural",
            num_results=5,
            summary=True
        )
        all_results[angle] = results.results

    return all_results
```

### Temporal Research

```python
def research_over_time(topic, years=[2022, 2023, 2024]):
    """Track topic evolution over time."""
    timeline = {}

    for year in years:
        results = exa.search_and_contents(
            topic,
            type="neural",
            num_results=5,
            summary=True,
            start_published_date=f"{year}-01-01",
            end_published_date=f"{year}-12-31"
        )
        timeline[year] = [
            {"title": r.title, "summary": r.summary, "url": r.url}
            for r in results.results
        ]

    return timeline
```

---

## Exa Research API

For long-form async research tasks producing structured reports.

### Basic Research Task

```python
# Start async research task
task = exa.research(
    "Comprehensive analysis of AI agent frameworks in 2024",
    output_format="markdown"
)

# Poll for completion
import time
while task.status != "completed":
    time.sleep(5)
    task = exa.get_research_task(task.id)

# Get results
print(task.result)  # Markdown report with citations
```

### Research with JSON Output

```python
task = exa.research(
    "Compare the top 5 vector databases",
    output_format="json",
    schema={
        "type": "object",
        "properties": {
            "databases": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "pros": {"type": "array"},
                        "cons": {"type": "array"},
                        "use_cases": {"type": "array"}
                    }
                }
            }
        }
    }
)
```

### Research Task Management

```python
# List active tasks
tasks = exa.list_research_tasks()

# Cancel a task
exa.cancel_research_task(task_id)

# Get specific task
task = exa.get_research_task(task_id)
print(f"Status: {task.status}")  # pending, running, completed, failed
print(f"Progress: {task.progress}%")
```

---

## Combining Deep Search with Answer API

```python
def comprehensive_answer(question):
    """Get answer with deep supporting research."""

    # Get direct answer
    answer_response = exa.answer(question, text=True)

    # Get additional context with deep search
    deep_results = exa.search_and_contents(
        question,
        type="neural",
        num_results=10,
        summary=True,
        highlights=True
    )

    return {
        "answer": answer_response.answer,
        "primary_citations": answer_response.citations,
        "supporting_research": [
            {
                "title": r.title,
                "url": r.url,
                "summary": r.summary,
                "key_points": r.highlights
            }
            for r in deep_results.results
        ]
    }
```
