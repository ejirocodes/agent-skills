# Citations Reference

## Table of Contents

- [Citation Structure](#citation-structure)
- [Extracting Citations](#extracting-citations)
- [Citation Display Patterns](#citation-display-patterns)
- [Verification](#verification)
- [Attribution Best Practices](#attribution-best-practices)

---

## Citation Structure

### From Answer API

```python
response = exa.answer("What is RAG?", text=True)

for citation in response.citations:
    print(f"URL: {citation.url}")
    print(f"Title: {citation.title}")
    print(f"Text: {citation.text}")  # If text=True
    print(f"Highlights: {citation.highlights}")
```

### From Search Results

```python
results = exa.search_and_contents("topic", text=True, highlights=True)

for result in results.results:
    citation = {
        "url": result.url,
        "title": result.title,
        "published_date": result.published_date,
        "author": result.author,
        "text": result.text,
        "highlights": result.highlights
    }
```

---

## Extracting Citations

### From Answer Response

```python
def extract_citations(answer_response):
    """Extract structured citations from answer."""
    return [
        {
            "id": i + 1,
            "url": c.url,
            "title": c.title or "Untitled",
            "snippet": c.highlights[0] if c.highlights else c.text[:200] if c.text else ""
        }
        for i, c in enumerate(answer_response.citations)
    ]

response = exa.answer("What are microservices?", text=True)
citations = extract_citations(response)
```

### From Search Results

```python
def search_with_citations(query, num_results=5):
    """Search and return structured citations."""
    results = exa.search_and_contents(
        query,
        num_results=num_results,
        text=True,
        highlights=True
    )

    return {
        "results": results.results,
        "citations": [
            {
                "id": i + 1,
                "url": r.url,
                "title": r.title,
                "date": r.published_date,
                "snippet": r.highlights[0] if r.highlights else r.text[:200] if r.text else ""
            }
            for i, r in enumerate(results.results)
        ]
    }
```

---

## Citation Display Patterns

### Inline Citations

```python
def format_answer_with_inline_citations(response):
    """Format answer with numbered inline citations."""
    answer = response.answer
    citations = response.citations

    # Add citation numbers to answer (simplified)
    formatted = answer
    for i, citation in enumerate(citations):
        formatted += f" [{i + 1}]"

    # Build references section
    references = "\n\nReferences:\n"
    for i, c in enumerate(citations):
        references += f"[{i + 1}] {c.title} - {c.url}\n"

    return formatted + references
```

### Academic Style

```python
def academic_citation(citation, style="apa"):
    """Format citation in academic style."""
    if style == "apa":
        author = citation.author or "Unknown Author"
        date = citation.published_date[:4] if citation.published_date else "n.d."
        title = citation.title
        url = citation.url
        return f"{author} ({date}). {title}. Retrieved from {url}"

    elif style == "mla":
        author = citation.author or "Unknown Author"
        title = f'"{citation.title}"'
        url = citation.url
        return f'{author}. {title}. Web. <{url}>'
```

### Markdown Format

```python
def citations_to_markdown(citations):
    """Convert citations to markdown format."""
    md = "## Sources\n\n"

    for i, c in enumerate(citations):
        title = c.title or c.url
        url = c.url
        snippet = c.highlights[0] if c.highlights else ""

        md += f"{i + 1}. [{title}]({url})\n"
        if snippet:
            md += f"   > {snippet[:150]}...\n"
        md += "\n"

    return md
```

### HTML Format

```python
def citations_to_html(citations):
    """Convert citations to HTML list."""
    html = '<ol class="citations">\n'

    for c in citations:
        title = c.title or "Untitled"
        url = c.url

        html += f'  <li>\n'
        html += f'    <a href="{url}" target="_blank">{title}</a>\n'
        if c.published_date:
            html += f'    <span class="date">({c.published_date})</span>\n'
        html += f'  </li>\n'

    html += '</ol>'
    return html
```

---

## Verification

### URL Validation

```python
from urllib.parse import urlparse

def validate_citation(citation):
    """Validate citation has required fields."""
    issues = []

    # Check URL
    if not citation.url:
        issues.append("Missing URL")
    else:
        parsed = urlparse(citation.url)
        if not parsed.scheme or not parsed.netloc:
            issues.append("Invalid URL format")

    # Check title
    if not citation.title:
        issues.append("Missing title")

    return {
        "valid": len(issues) == 0,
        "issues": issues
    }

def filter_valid_citations(citations):
    """Return only valid citations."""
    return [c for c in citations if validate_citation(c)["valid"]]
```

### Content Verification

```python
def verify_citation_content(citation, claim):
    """Verify citation supports the claim."""
    if not citation.text and not citation.highlights:
        return {"verified": False, "reason": "No content to verify"}

    content = citation.text or " ".join(citation.highlights)

    # Simple keyword overlap check
    claim_words = set(claim.lower().split())
    content_words = set(content.lower().split())
    overlap = claim_words & content_words

    relevance = len(overlap) / len(claim_words) if claim_words else 0

    return {
        "verified": relevance > 0.3,
        "relevance_score": relevance,
        "matching_terms": list(overlap)
    }
```

### Freshness Check

```python
from datetime import datetime, timedelta

def check_citation_freshness(citation, max_age_days=365):
    """Check if citation is recent enough."""
    if not citation.published_date:
        return {"fresh": None, "reason": "No date available"}

    try:
        pub_date = datetime.fromisoformat(citation.published_date[:10])
        age = datetime.now() - pub_date
        is_fresh = age.days <= max_age_days

        return {
            "fresh": is_fresh,
            "age_days": age.days,
            "published": citation.published_date
        }
    except ValueError:
        return {"fresh": None, "reason": "Invalid date format"}
```

---

## Attribution Best Practices

### Always Include Sources

```python
def generate_response_with_sources(question):
    """Always include sources in response."""
    response = exa.answer(question, text=True)

    output = f"Answer: {response.answer}\n\n"

    if response.citations:
        output += "Sources:\n"
        for c in response.citations:
            output += f"- {c.title}: {c.url}\n"
    else:
        output += "Note: This response could not be verified with sources.\n"

    return output
```

### Distinguish Primary vs Supporting Sources

```python
def categorize_sources(answer_citations, search_results):
    """Categorize sources by relevance."""
    return {
        "primary_sources": [
            {"title": c.title, "url": c.url}
            for c in answer_citations[:3]
        ],
        "supporting_sources": [
            {"title": r.title, "url": r.url}
            for r in search_results
            if r.url not in [c.url for c in answer_citations]
        ]
    }
```

### Handle Missing Citations Gracefully

```python
def safe_citation_display(response):
    """Handle cases with no or few citations."""
    answer = response.answer
    citations = response.citations or []

    if len(citations) == 0:
        return {
            "answer": answer,
            "confidence": "low",
            "warning": "No sources found to verify this information",
            "citations": []
        }
    elif len(citations) < 3:
        return {
            "answer": answer,
            "confidence": "medium",
            "note": "Limited sources available",
            "citations": citations
        }
    else:
        return {
            "answer": answer,
            "confidence": "high",
            "citations": citations
        }
```
