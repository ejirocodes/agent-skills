# Search Filters Reference

## Table of Contents

- [Domain Filtering](#domain-filtering)
- [Date Filtering](#date-filtering)
- [Text Filtering](#text-filtering)
- [Category Filtering](#category-filtering)
- [Geolocation Filtering](#geolocation-filtering)
- [Combining Filters](#combining-filters)

---

## Domain Filtering

### Include Specific Domains

Limit results to specific websites.

```python
results = exa.search_and_contents(
    "machine learning tutorials",
    include_domains=["github.com", "medium.com", "dev.to"],
    num_results=10
)
```

```typescript
const results = await exa.searchAndContents("machine learning tutorials", {
  includeDomains: ["github.com", "medium.com", "dev.to"],
  numResults: 10,
});
```

### Exclude Domains

Filter out unwanted sources.

```python
results = exa.search_and_contents(
    "python programming",
    exclude_domains=["pinterest.com", "facebook.com", "twitter.com"],
    num_results=10
)
```

### Subdomain Wildcards

Match all subdomains of a domain.

```python
# Matches docs.github.com, gist.github.com, etc.
results = exa.search_and_contents(
    "GitHub Actions examples",
    include_domains=["*.github.com"],
    num_results=10
)
```

---

## Date Filtering

### Published Date Range

Filter by when content was published.

```python
results = exa.search_and_contents(
    "AI developments",
    start_published_date="2024-01-01",
    end_published_date="2024-12-31",
    num_results=10
)
```

```typescript
const results = await exa.searchAndContents("AI developments", {
  startPublishedDate: "2024-01-01",
  endPublishedDate: "2024-12-31",
  numResults: 10,
});
```

### Crawl Date Range

Filter by when Exa indexed the page (useful for finding recently discovered content).

```python
results = exa.search_and_contents(
    "new JavaScript frameworks",
    start_crawl_date="2024-06-01",
    num_results=10
)
```

### Date Format

- Use ISO 8601 format: `YYYY-MM-DD`
- Timestamps also supported: `2024-01-01T00:00:00Z`

---

## Text Filtering

### Include Text

Results must contain all specified strings.

```python
results = exa.search_and_contents(
    "web development",
    include_text=["React", "TypeScript"],
    num_results=10
)
```

```typescript
const results = await exa.searchAndContents("web development", {
  includeText: ["React", "TypeScript"],
  numResults: 10,
});
```

### Exclude Text

Filter out results containing specified strings.

```python
results = exa.search_and_contents(
    "JavaScript tutorials",
    exclude_text=["advertisement", "sponsored", "affiliate"],
    num_results=10
)
```

### Text Filter Best Practices

- Use for required technical terms
- Helps narrow broad queries
- Case-insensitive matching
- Combine with domain filters for precision

---

## Category Filtering

Pre-defined categories for specialized searches.

### Available Categories

| Category | Description | Use Case |
|----------|-------------|----------|
| `company` | Company websites and profiles | Lead generation, company research |
| `research_paper` | Academic papers and studies | Scientific research |
| `news` | News articles | Current events, media monitoring |
| `pdf` | PDF documents | Reports, whitepapers |
| `github` | GitHub repositories | Code search, open source |
| `tweet` | Twitter/X posts | Social media monitoring |
| `personal_site` | Personal websites and blogs | Individual perspectives |
| `linkedin_profile` | LinkedIn profiles | Recruiting, people search |

### Usage

```python
# Company search
results = exa.search_and_contents(
    "AI startups healthcare",
    category="company",
    num_results=10
)

# Research papers
results = exa.search_and_contents(
    "transformer architecture improvements",
    category="research_paper",
    num_results=10
)

# GitHub repositories
results = exa.search_and_contents(
    "React component library",
    category="github",
    num_results=10
)
```

```typescript
const results = await exa.searchAndContents("AI startups healthcare", {
  category: "company",
  numResults: 10,
});
```

---

## Geolocation Filtering

Filter by user location for localized results.

```python
results = exa.search_and_contents(
    "coffee shops",
    user_location={
        "latitude": 37.7749,
        "longitude": -122.4194,
        "radius_miles": 10
    },
    num_results=10
)
```

### Language Filtering

Exa auto-detects query language and filters results accordingly. This is enabled by default.

```python
# Disable language filtering to get results in all languages
results = exa.search_and_contents(
    "machine learning",
    filter_language=False,
    num_results=10
)
```

---

## Combining Filters

### Complex Filter Example

```python
results = exa.search_and_contents(
    "Series A funding announcement",
    type="neural",
    category="news",
    include_domains=["techcrunch.com", "venturebeat.com", "bloomberg.com"],
    exclude_text=["sponsored"],
    start_published_date="2024-01-01",
    num_results=20,
    text=True,
    summary=True
)
```

```typescript
const results = await exa.searchAndContents("Series A funding announcement", {
  type: "neural",
  category: "news",
  includeDomains: ["techcrunch.com", "venturebeat.com", "bloomberg.com"],
  excludeText: ["sponsored"],
  startPublishedDate: "2024-01-01",
  numResults: 20,
  text: true,
  summary: true,
});
```

### Filter Priority Tips

1. Start broad, then narrow with filters
2. Category filters are highly optimized - use when applicable
3. Domain filters reduce noise effectively
4. Date filters essential for time-sensitive content
5. Text filters for required technical terms
