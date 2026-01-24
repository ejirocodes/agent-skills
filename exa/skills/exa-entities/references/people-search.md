# People Search Reference

## Table of Contents

- [Basic People Search](#basic-people-search)
- [LinkedIn Profile Search](#linkedin-profile-search)
- [Filtering Profiles](#filtering-profiles)
- [Recruiting Patterns](#recruiting-patterns)
- [Professional Research](#professional-research)

---

## Basic People Search

Exa indexes over 1 billion professional profiles, including LinkedIn.

### Using LinkedIn Category

```python
from exa_py import Exa

exa = Exa()

results = exa.search_and_contents(
    "software engineer machine learning Python",
    category="linkedin_profile",
    num_results=20,
    text=True
)

for profile in results.results:
    print(f"Name: {profile.title}")
    print(f"Profile: {profile.url}")
    print(f"Bio: {profile.text[:200]}...")
    print()
```

### TypeScript

```typescript
import Exa from "exa-js";

const exa = new Exa();

const results = await exa.searchAndContents(
  "software engineer machine learning Python",
  {
    category: "linkedin_profile",
    numResults: 20,
    text: true,
  }
);
```

---

## LinkedIn Profile Search

### By Role and Skills

```python
# Senior engineers with specific skills
results = exa.search_and_contents(
    "senior software engineer React TypeScript Node.js",
    category="linkedin_profile",
    num_results=30,
    text=True
)
```

### By Company

```python
# Find people at specific companies
results = exa.search_and_contents(
    "engineer at Google machine learning",
    category="linkedin_profile",
    num_results=20,
    text=True
)

# Alumni search
results = exa.search_and_contents(
    "former Google engineer startup founder",
    category="linkedin_profile",
    num_results=20,
    text=True
)
```

### By Location

```python
# Profiles in specific locations
results = exa.search_and_contents(
    "data scientist San Francisco Bay Area",
    category="linkedin_profile",
    num_results=30,
    text=True
)

# Remote-friendly
results = exa.search_and_contents(
    "remote software engineer distributed team",
    category="linkedin_profile",
    num_results=30,
    text=True
)
```

---

## Filtering Profiles

### Domain Filtering

```python
# Only LinkedIn profiles
results = exa.search_and_contents(
    "product manager fintech",
    category="linkedin_profile",
    include_domains=["linkedin.com"],
    num_results=30,
    text=True
)

# Personal sites and portfolios
results = exa.search_and_contents(
    "UX designer portfolio",
    category="personal_site",
    num_results=30,
    text=True
)
```

### By Experience Level

```python
# Senior/leadership
results = exa.search_and_contents(
    "VP Engineering CTO tech startup",
    category="linkedin_profile",
    num_results=20,
    text=True
)

# Mid-level
results = exa.search_and_contents(
    "senior software engineer 5 years experience",
    category="linkedin_profile",
    num_results=30,
    text=True
)

# Entry-level
results = exa.search_and_contents(
    "junior developer bootcamp graduate",
    category="linkedin_profile",
    num_results=30,
    text=True
)
```

### GitHub Developer Search

```python
# Find developers by their GitHub presence
results = exa.search_and_contents(
    "open source contributor Rust systems programming",
    category="github",
    num_results=20,
    text=True
)
```

---

## Recruiting Patterns

### Candidate Search

```python
def search_candidates(role, skills, location=None, limit=50):
    """Search for job candidates."""
    query_parts = [role] + skills
    if location:
        query_parts.append(location)

    query = " ".join(query_parts)

    results = exa.search_and_contents(
        query,
        category="linkedin_profile",
        num_results=limit,
        text=True
    )

    candidates = []
    for r in results.results:
        candidates.append({
            "name": r.title,
            "profile_url": r.url,
            "bio": r.text[:500] if r.text else None
        })

    return candidates

# Example
candidates = search_candidates(
    role="senior backend engineer",
    skills=["Python", "Django", "PostgreSQL"],
    location="New York"
)
```

### Find Similar Profiles

```python
def find_similar_candidates(profile_url, limit=20):
    """Find candidates similar to a known good profile."""
    similar = exa.find_similar_and_contents(
        profile_url,
        num_results=limit,
        category="linkedin_profile",
        exclude_source_domain=False,  # Keep LinkedIn results
        text=True
    )

    return [
        {
            "name": r.title,
            "profile_url": r.url,
            "bio": r.text[:500] if r.text else None
        }
        for r in similar.results
    ]
```

### Build Talent Pipeline

```python
def build_talent_pipeline(job_requirements):
    """Build a pipeline of potential candidates."""
    pipeline = {
        "active_seekers": [],
        "passive_candidates": [],
        "leadership": []
    }

    # Active job seekers
    active = exa.search_and_contents(
        f"{job_requirements['role']} open to opportunities seeking",
        category="linkedin_profile",
        num_results=30,
        text=True
    )
    pipeline["active_seekers"] = [
        {"name": r.title, "url": r.url}
        for r in active.results
    ]

    # Passive candidates at target companies
    if job_requirements.get("target_companies"):
        for company in job_requirements["target_companies"][:3]:
            passive = exa.search_and_contents(
                f"{job_requirements['role']} at {company}",
                category="linkedin_profile",
                num_results=10,
                text=True
            )
            pipeline["passive_candidates"].extend([
                {"name": r.title, "url": r.url, "company": company}
                for r in passive.results
            ])

    # Leadership candidates
    leadership = exa.search_and_contents(
        f"head of {job_requirements['role']} director manager",
        category="linkedin_profile",
        num_results=20,
        text=True
    )
    pipeline["leadership"] = [
        {"name": r.title, "url": r.url}
        for r in leadership.results
    ]

    return pipeline
```

---

## Professional Research

### Expert Finding

```python
def find_experts(domain, num_experts=20):
    """Find experts in a specific domain."""
    # Search for thought leaders
    results = exa.search_and_contents(
        f"{domain} expert thought leader speaker author",
        category="linkedin_profile",
        num_results=num_experts,
        text=True,
        summary=True
    )

    return [
        {
            "name": r.title,
            "profile": r.url,
            "summary": r.summary,
            "expertise": domain
        }
        for r in results.results
    ]
```

### Team Research

```python
def research_company_team(company_name):
    """Research key people at a company."""
    roles = ["CEO", "CTO", "VP Engineering", "Head of Product"]

    team = {}
    for role in roles:
        results = exa.search_and_contents(
            f"{role} at {company_name}",
            category="linkedin_profile",
            num_results=3,
            text=True
        )
        if results.results:
            team[role] = {
                "name": results.results[0].title,
                "profile": results.results[0].url
            }

    return team
```

### Investor/Advisor Search

```python
def find_investors(industry, stage="seed"):
    """Find investors active in an industry."""
    results = exa.search_and_contents(
        f"angel investor VC partner {industry} {stage}",
        category="linkedin_profile",
        num_results=30,
        text=True
    )

    investors = []
    for r in results.results:
        investors.append({
            "name": r.title,
            "profile": r.url,
            "bio": r.text[:300] if r.text else None
        })

    return investors
```

### Conference Speaker Research

```python
def find_speakers(topic, conference_type="tech"):
    """Find potential speakers for events."""
    results = exa.search_and_contents(
        f"{topic} speaker {conference_type} conference keynote",
        category="linkedin_profile",
        num_results=30,
        text=True,
        summary=True
    )

    return [
        {
            "name": r.title,
            "profile": r.url,
            "summary": r.summary
        }
        for r in results.results
    ]
```
