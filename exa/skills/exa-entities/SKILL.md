---
name: exa-entities
description: "Exa.ai company and people search for lead generation, competitive intelligence, and data enrichment. Use when searching for companies, finding people profiles, building lead gen tools, or implementing Websets for data collection at scale. Triggers on: Exa company search, Exa people search, category company, lead generation, company research, profile search, LinkedIn profiles, Websets API, data enrichment, company lookup, find companies, competitive intelligence, recruiting, talent search, 1B profiles."
license: MIT
metadata:
  author: ejirocodes
  version: '1.0.0'
---

# Exa Entity Search

## Quick Reference

| Topic | When to Use | Reference |
|-------|-------------|-----------|
| **Company Search** | Finding companies, competitive research | [company-search.md](references/company-search.md) |
| **People Search** | Finding profiles, recruiting | [people-search.md](references/people-search.md) |
| **Websets** | Data collection at scale, monitoring | [websets.md](references/websets.md) |

## Essential Patterns

### Company Search

```python
from exa_py import Exa

exa = Exa()

results = exa.search_and_contents(
    "AI startups in healthcare series A funding",
    category="company",
    num_results=20,
    text=True
)

for company in results.results:
    print(f"{company.title}: {company.url}")
```

### People Search

```python
results = exa.search_and_contents(
    "machine learning engineers San Francisco",
    category="linkedin_profile",
    num_results=20,
    text=True
)

for profile in results.results:
    print(f"{profile.title}: {profile.url}")
```

### Websets for Lead Generation

```python
# Create a webset for company collection
webset = exa.websets.create(
    name="AI Healthcare Companies",
    search_query="AI healthcare startups",
    category="company",
    max_results=100
)

# Monitor for new matches
exa.websets.add_monitor(
    webset_id=webset.id,
    schedule="daily"
)
```

## Category Reference

| Category | Use Case | Index Size |
|----------|----------|------------|
| `company` | Company websites, about pages | Millions |
| `linkedin_profile` | Professional profiles | 1B+ profiles |
| `personal_site` | Individual blogs, portfolios | Millions |
| `github` | Repositories, developer profiles | Millions |

## Common Mistakes

1. **Not using category filter** - Always set `category="company"` or `category="linkedin_profile"` for entity search
2. **Expecting structured data** - Exa returns web pages; parse text for structured fields
3. **Over-broad queries** - Add location, industry, or role specifics for better results
4. **Ignoring rate limits** - Batch requests and implement backoff for large-scale collection
5. **Missing domain filters** - Use `include_domains=["linkedin.com"]` for profile-only results
