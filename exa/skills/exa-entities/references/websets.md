# Websets API Reference

## Table of Contents

- [Overview](#overview)
- [Creating Websets](#creating-websets)
- [Managing Websets](#managing-websets)
- [Monitoring](#monitoring)
- [Enrichment](#enrichment)
- [Export and Integration](#export-and-integration)

---

## Overview

Websets enable web data collection, verification, and processing at scale:

- **Search-based collection**: Automatically gather data matching your criteria
- **Monitoring**: Get notified when new matches appear
- **Enrichment**: Enhance collected data with additional information
- **Webhooks**: Integrate with your systems in real-time

---

## Creating Websets

### Basic Webset

```python
from exa_py import Exa

exa = Exa()

# Create a webset for company collection
webset = exa.websets.create(
    name="AI Startups 2024",
    description="AI companies founded in 2024",
    search_query="AI startup founded 2024",
    category="company",
    max_results=500
)

print(f"Webset ID: {webset.id}")
print(f"Status: {webset.status}")
```

### With Advanced Filters

```python
webset = exa.websets.create(
    name="Healthcare AI Series A",
    description="Healthcare AI companies with Series A funding",
    search_query="healthcare AI startup series A funding",
    category="company",
    include_domains=None,  # All domains
    exclude_domains=["linkedin.com", "crunchbase.com"],
    start_published_date="2023-01-01",
    max_results=200
)
```

### People Webset

```python
webset = exa.websets.create(
    name="ML Engineers SF",
    description="Machine learning engineers in San Francisco",
    search_query="machine learning engineer San Francisco",
    category="linkedin_profile",
    max_results=1000
)
```

---

## Managing Websets

### List Websets

```python
# Get all websets
websets = exa.websets.list()

for ws in websets:
    print(f"Name: {ws.name}")
    print(f"ID: {ws.id}")
    print(f"Status: {ws.status}")
    print(f"Items: {ws.item_count}")
    print()
```

### Get Webset Details

```python
webset = exa.websets.get(webset_id="ws_abc123")

print(f"Name: {webset.name}")
print(f"Status: {webset.status}")
print(f"Created: {webset.created_at}")
print(f"Item count: {webset.item_count}")
```

### Get Webset Items

```python
# Get items from a webset
items = exa.websets.get_items(
    webset_id="ws_abc123",
    limit=100,
    offset=0
)

for item in items:
    print(f"URL: {item.url}")
    print(f"Title: {item.title}")
    print(f"Added: {item.added_at}")
```

### Update Webset

```python
updated = exa.websets.update(
    webset_id="ws_abc123",
    name="Updated Name",
    description="Updated description"
)
```

### Delete Webset

```python
exa.websets.delete(webset_id="ws_abc123")
```

---

## Monitoring

Set up monitors to track new matches over time.

### Create Monitor

```python
# Daily monitoring for new results
monitor = exa.websets.add_monitor(
    webset_id="ws_abc123",
    schedule="daily",  # or "hourly", "weekly"
    enabled=True
)

print(f"Monitor ID: {monitor.id}")
print(f"Schedule: {monitor.schedule}")
```

### With Webhook Notification

```python
monitor = exa.websets.add_monitor(
    webset_id="ws_abc123",
    schedule="hourly",
    webhook_url="https://your-app.com/webhooks/exa",
    webhook_secret="your-secret-key"
)
```

### Manage Monitors

```python
# List monitors for a webset
monitors = exa.websets.list_monitors(webset_id="ws_abc123")

# Pause monitor
exa.websets.update_monitor(
    monitor_id="mon_xyz",
    enabled=False
)

# Delete monitor
exa.websets.delete_monitor(monitor_id="mon_xyz")
```

### Webhook Payload Structure

```json
{
  "event": "new_items",
  "webset_id": "ws_abc123",
  "webset_name": "AI Startups 2024",
  "new_items_count": 5,
  "items": [
    {
      "url": "https://example.com",
      "title": "New AI Startup",
      "added_at": "2024-01-15T10:30:00Z"
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Verify Webhook Signature

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    """Verify Exa webhook signature."""
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)

# In your webhook handler
@app.post("/webhooks/exa")
def handle_webhook(request):
    signature = request.headers.get("X-Exa-Signature")
    payload = request.body

    if not verify_webhook(payload, signature, WEBHOOK_SECRET):
        return {"error": "Invalid signature"}, 401

    data = json.loads(payload)
    # Process new items
    for item in data["items"]:
        process_new_item(item)

    return {"status": "ok"}
```

---

## Enrichment

Enhance webset items with additional data.

### Basic Enrichment

```python
# Enrich items with full content
enriched = exa.websets.enrich(
    webset_id="ws_abc123",
    enrichments=["text", "summary"]
)

# Get enriched items
items = exa.websets.get_items(webset_id="ws_abc123")

for item in items:
    print(f"URL: {item.url}")
    print(f"Summary: {item.summary}")
    print(f"Full text: {item.text[:500]}")
```

### Custom Enrichment Fields

```python
enriched = exa.websets.enrich(
    webset_id="ws_abc123",
    enrichments={
        "text": True,
        "summary": True,
        "highlights": {"num_sentences": 3}
    }
)
```

---

## Export and Integration

### Export to CSV

```python
# Export webset to CSV
export = exa.websets.export(
    webset_id="ws_abc123",
    format="csv"
)

# Download the export
with open("companies.csv", "wb") as f:
    f.write(export.content)
```

### Export to JSON

```python
export = exa.websets.export(
    webset_id="ws_abc123",
    format="json"
)

data = json.loads(export.content)
```

### Import URLs

```python
# Import existing URLs into a webset
exa.websets.import_urls(
    webset_id="ws_abc123",
    urls=[
        "https://company1.com",
        "https://company2.com",
        "https://company3.com"
    ]
)
```

### Sync with Database

```python
def sync_webset_to_database(webset_id, db):
    """Sync webset items to a database."""
    items = exa.websets.get_items(webset_id=webset_id)

    for item in items:
        db.upsert("companies", {
            "url": item.url,
            "title": item.title,
            "summary": item.summary,
            "added_at": item.added_at,
            "source": "exa_webset"
        })
```

---

## Common Patterns

### Lead Generation Pipeline

```python
def create_lead_pipeline(industry, target_count=500):
    """Create a complete lead generation pipeline."""

    # 1. Create webset for companies
    webset = exa.websets.create(
        name=f"{industry} Companies",
        search_query=f"{industry} startup company",
        category="company",
        max_results=target_count
    )

    # 2. Set up daily monitoring
    monitor = exa.websets.add_monitor(
        webset_id=webset.id,
        schedule="daily",
        webhook_url="https://your-crm.com/webhooks/new-leads"
    )

    # 3. Enrich with content
    exa.websets.enrich(
        webset_id=webset.id,
        enrichments=["text", "summary"]
    )

    return {
        "webset_id": webset.id,
        "monitor_id": monitor.id,
        "status": "pipeline_created"
    }
```

### Competitive Monitoring

```python
def monitor_competitors(competitor_names):
    """Set up monitoring for competitor news."""

    for name in competitor_names:
        webset = exa.websets.create(
            name=f"{name} News",
            search_query=f"{name} news announcement",
            category="news",
            max_results=100
        )

        exa.websets.add_monitor(
            webset_id=webset.id,
            schedule="hourly",
            webhook_url="https://your-app.com/webhooks/competitor-news"
        )
```

### Talent Pool Management

```python
def create_talent_pool(role, skills, location):
    """Create and maintain a talent pool."""

    query = f"{role} {' '.join(skills)} {location}"

    webset = exa.websets.create(
        name=f"Talent Pool: {role}",
        search_query=query,
        category="linkedin_profile",
        max_results=1000
    )

    # Monitor for new profiles weekly
    exa.websets.add_monitor(
        webset_id=webset.id,
        schedule="weekly"
    )

    # Enrich with profile details
    exa.websets.enrich(
        webset_id=webset.id,
        enrichments=["text", "summary"]
    )

    return webset
```
