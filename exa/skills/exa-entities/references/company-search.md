# Company Search Reference

## Table of Contents

- [Basic Company Search](#basic-company-search)
- [Filtering Companies](#filtering-companies)
- [Company Research Patterns](#company-research-patterns)
- [Competitive Intelligence](#competitive-intelligence)
- [Lead Generation](#lead-generation)

---

## Basic Company Search

### Using Company Category

```python
from exa_py import Exa

exa = Exa()

# Search for companies by description
results = exa.search_and_contents(
    "AI startups building developer tools",
    category="company",
    num_results=20,
    text=True
)

for company in results.results:
    print(f"Company: {company.title}")
    print(f"URL: {company.url}")
    print(f"Description: {company.text[:300]}...")
    print()
```

### TypeScript

```typescript
import Exa from "exa-js";

const exa = new Exa();

const results = await exa.searchAndContents(
  "AI startups building developer tools",
  {
    category: "company",
    numResults: 20,
    text: true,
  }
);
```

---

## Filtering Companies

### By Industry and Location

```python
# Healthcare AI in San Francisco
results = exa.search_and_contents(
    "healthcare AI startup San Francisco",
    category="company",
    num_results=20,
    text=True
)

# Fintech in London
results = exa.search_and_contents(
    "fintech company London UK",
    category="company",
    num_results=20,
    text=True
)
```

### By Funding Stage

```python
# Series A companies
results = exa.search_and_contents(
    "series A startup AI infrastructure",
    category="company",
    num_results=20,
    text=True,
    start_published_date="2023-01-01"  # Recent funding
)

# Seed stage
results = exa.search_and_contents(
    "seed funded startup machine learning",
    category="company",
    num_results=20,
    text=True
)
```

### By Company Size

```python
# Enterprise companies
results = exa.search_and_contents(
    "enterprise software company Fortune 500",
    category="company",
    num_results=20,
    text=True
)

# Small startups
results = exa.search_and_contents(
    "early stage startup founded 2023",
    category="company",
    num_results=20,
    text=True
)
```

### Combining Domain Filters

```python
# Only include specific company sites
results = exa.search_and_contents(
    "AI company",
    category="company",
    num_results=20,
    exclude_domains=["linkedin.com", "crunchbase.com", "glassdoor.com"],
    text=True
)
```

---

## Company Research Patterns

### Company Profile Extraction

```python
def get_company_profile(company_name):
    """Research a specific company."""
    results = exa.search_and_contents(
        f"{company_name} company about us",
        category="company",
        num_results=5,
        text=True,
        summary=True
    )

    if results.results:
        company = results.results[0]
        return {
            "name": company_name,
            "url": company.url,
            "summary": company.summary,
            "full_text": company.text
        }
    return None
```

### Industry Overview

```python
def research_industry(industry, limit=50):
    """Get overview of companies in an industry."""
    results = exa.search_and_contents(
        f"{industry} companies startups",
        category="company",
        num_results=limit,
        summary=True
    )

    return {
        "industry": industry,
        "company_count": len(results.results),
        "companies": [
            {
                "name": r.title,
                "url": r.url,
                "summary": r.summary
            }
            for r in results.results
        ]
    }
```

### Technology Stack Research

```python
def find_companies_using_tech(technology):
    """Find companies using specific technology."""
    results = exa.search_and_contents(
        f"company using {technology} tech stack",
        category="company",
        num_results=30,
        text=True,
        include_text=[technology]  # Ensure tech is mentioned
    )

    return results.results
```

---

## Competitive Intelligence

### Find Competitors

```python
def find_competitors(company_url, num_competitors=10):
    """Find similar companies (competitors)."""
    similar = exa.find_similar_and_contents(
        company_url,
        num_results=num_competitors,
        category="company",
        exclude_source_domain=True,
        text=True,
        summary=True
    )

    return [
        {
            "name": r.title,
            "url": r.url,
            "summary": r.summary
        }
        for r in similar.results
    ]
```

### Market Landscape

```python
def map_market_landscape(market_description):
    """Map companies in a market segment."""

    # Search for different company types
    segments = {
        "leaders": f"{market_description} market leader enterprise",
        "challengers": f"{market_description} growing startup series B C",
        "emerging": f"{market_description} new startup seed series A"
    }

    landscape = {}
    for segment, query in segments.items():
        results = exa.search_and_contents(
            query,
            category="company",
            num_results=15,
            summary=True
        )
        landscape[segment] = [
            {"name": r.title, "url": r.url, "summary": r.summary}
            for r in results.results
        ]

    return landscape
```

### Track Company News

```python
def get_company_news(company_name, days=30):
    """Get recent news about a company."""
    from datetime import datetime, timedelta

    start_date = (datetime.now() - timedelta(days=days)).strftime("%Y-%m-%d")

    results = exa.search_and_contents(
        f"{company_name} news announcement",
        category="news",
        num_results=20,
        start_published_date=start_date,
        text=True,
        highlights=True
    )

    return [
        {
            "title": r.title,
            "url": r.url,
            "date": r.published_date,
            "highlights": r.highlights
        }
        for r in results.results
    ]
```

---

## Lead Generation

### Build Prospect List

```python
def build_prospect_list(criteria, limit=100):
    """Build a list of company prospects."""
    query = " ".join([
        criteria.get("industry", ""),
        criteria.get("location", ""),
        criteria.get("size", ""),
        criteria.get("technology", ""),
        "company"
    ])

    results = exa.search_and_contents(
        query,
        category="company",
        num_results=limit,
        text=True,
        summary=True
    )

    prospects = []
    for r in results.results:
        prospects.append({
            "company_name": r.title,
            "website": r.url,
            "summary": r.summary,
            "description": r.text[:500] if r.text else None
        })

    return prospects

# Example usage
prospects = build_prospect_list({
    "industry": "fintech",
    "location": "New York",
    "size": "startup",
    "technology": "AI"
})
```

### Enrich Company Data

```python
def enrich_company(company_url):
    """Get additional data for a company."""
    contents = exa.get_contents(
        urls=[company_url],
        text=True,
        summary=True
    )

    if contents.results:
        company = contents.results[0]

        # Search for additional info
        news = exa.search_and_contents(
            f"site:{company_url} news",
            num_results=5,
            text=True
        )

        return {
            "url": company_url,
            "title": company.title,
            "summary": company.summary,
            "full_description": company.text,
            "recent_news": [r.title for r in news.results]
        }

    return None
```

### Export to CSV

```python
import csv

def export_companies_to_csv(companies, filename="companies.csv"):
    """Export company list to CSV."""
    if not companies:
        return

    fieldnames = ["company_name", "website", "summary"]

    with open(filename, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        for company in companies:
            writer.writerow({
                "company_name": company.get("company_name", ""),
                "website": company.get("website", ""),
                "summary": company.get("summary", "")[:500]
            })
```
