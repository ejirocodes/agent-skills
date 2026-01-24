# LlamaIndex Integration Reference

## Table of Contents

- [Installation](#installation)
- [ExaReader](#exareader)
- [Query Engine Patterns](#query-engine-patterns)
- [Index Integration](#index-integration)
- [Advanced Patterns](#advanced-patterns)

---

## Installation

```bash
pip install llama-index-readers-web
```

---

## ExaReader

The primary way to load web content into LlamaIndex.

### Basic Usage

```python
from llama_index.readers.web import ExaReader

reader = ExaReader(api_key="your-key")  # or set EXA_API_KEY env var

documents = reader.load_data(
    query="machine learning best practices",
    num_results=10
)

for doc in documents:
    print(f"Source: {doc.metadata['url']}")
    print(f"Content: {doc.text[:200]}...")
```

### With Search Options

```python
documents = reader.load_data(
    query="React Server Components tutorial",
    num_results=10,
    include_domains=["react.dev", "nextjs.org"],
    start_published_date="2024-01-01",
    text_length_limit=2000
)
```

### With Highlights

```python
documents = reader.load_data(
    query="Python async patterns",
    num_results=10,
    highlights=True,
    num_sentences=3
)
# Returns documents with highlight snippets
```

### Configuration Options

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | str | Search query |
| `num_results` | int | Number of results (default: 10) |
| `include_domains` | list | Limit to these domains |
| `exclude_domains` | list | Exclude these domains |
| `start_published_date` | str | Start date (YYYY-MM-DD) |
| `end_published_date` | str | End date |
| `highlights` | bool | Return highlights |
| `num_sentences` | int | Sentences per highlight |
| `text_length_limit` | int | Max characters |

---

## Query Engine Patterns

### Basic Query Engine

```python
from llama_index.readers.web import ExaReader
from llama_index.core import VectorStoreIndex
from llama_index.llms.openai import OpenAI

# Load documents from Exa
reader = ExaReader(api_key="your-key")
documents = reader.load_data(
    query="GraphQL best practices",
    num_results=10
)

# Create index and query engine
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(llm=OpenAI(model="gpt-4"))

# Query
response = query_engine.query("What are the key GraphQL patterns?")
print(response)
```

### Query Engine with Citations

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.response_synthesizers import ResponseMode

documents = reader.load_data(query="kubernetes deployment", num_results=10)
index = VectorStoreIndex.from_documents(documents)

query_engine = index.as_query_engine(
    response_mode=ResponseMode.COMPACT,
    llm=OpenAI(model="gpt-4")
)

response = query_engine.query("How do I deploy to Kubernetes?")

# Access source nodes
for node in response.source_nodes:
    print(f"Source: {node.metadata['url']}")
    print(f"Score: {node.score}")
```

### Chat Engine

```python
from llama_index.core.chat_engine import CondenseQuestionChatEngine

documents = reader.load_data(query="LangChain vs LlamaIndex", num_results=10)
index = VectorStoreIndex.from_documents(documents)

chat_engine = CondenseQuestionChatEngine.from_defaults(
    query_engine=index.as_query_engine(),
    llm=OpenAI(model="gpt-4")
)

response = chat_engine.chat("What are the differences?")
response = chat_engine.chat("Which is better for RAG?")
```

---

## Index Integration

### Combining with Existing Index

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.chroma import ChromaVectorStore
import chromadb

# Existing Chroma collection
chroma_client = chromadb.Client()
collection = chroma_client.get_or_create_collection("my_collection")
vector_store = ChromaVectorStore(chroma_collection=collection)

# Load new documents from Exa
reader = ExaReader(api_key="your-key")
new_docs = reader.load_data(query="latest AI news", num_results=5)

# Add to existing index
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_documents(
    new_docs,
    storage_context=storage_context
)
```

### Refresh with Real-Time Data

```python
def refresh_index_with_exa(index, query, num_results=5):
    """Add fresh web content to an existing index."""
    reader = ExaReader(api_key="your-key")
    fresh_docs = reader.load_data(
        query=query,
        num_results=num_results,
        start_published_date="2024-01-01"  # Recent only
    )

    # Insert into existing index
    for doc in fresh_docs:
        index.insert(doc)

    return index
```

---

## Advanced Patterns

### Parallel Loading

```python
import asyncio
from llama_index.readers.web import ExaReader

async def load_multiple_queries(queries):
    reader = ExaReader(api_key="your-key")
    tasks = [
        asyncio.to_thread(
            reader.load_data,
            query=q,
            num_results=5
        )
        for q in queries
    ]
    results = await asyncio.gather(*tasks)
    # Flatten documents
    return [doc for docs in results for doc in docs]

queries = ["Python async", "Python typing", "Python testing"]
all_docs = asyncio.run(load_multiple_queries(queries))
```

### Custom Node Parser

```python
from llama_index.core.node_parser import SentenceSplitter
from llama_index.readers.web import ExaReader

reader = ExaReader(api_key="your-key")
documents = reader.load_data(query="deep learning", num_results=10)

# Custom chunking for web content
parser = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=50
)

nodes = parser.get_nodes_from_documents(documents)

# Create index from nodes
index = VectorStoreIndex(nodes)
```

### Sub-Question Query Engine

```python
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool

reader = ExaReader(api_key="your-key")

# Create specialized indexes
ml_docs = reader.load_data(query="machine learning frameworks", num_results=10)
web_docs = reader.load_data(query="web development frameworks", num_results=10)

ml_index = VectorStoreIndex.from_documents(ml_docs)
web_index = VectorStoreIndex.from_documents(web_docs)

# Create tools
ml_tool = QueryEngineTool.from_defaults(
    query_engine=ml_index.as_query_engine(),
    name="ml_search",
    description="Search for machine learning information"
)

web_tool = QueryEngineTool.from_defaults(
    query_engine=web_index.as_query_engine(),
    name="web_search",
    description="Search for web development information"
)

# Sub-question engine
query_engine = SubQuestionQueryEngine.from_defaults(
    query_engine_tools=[ml_tool, web_tool],
    llm=OpenAI(model="gpt-4")
)

response = query_engine.query(
    "Compare ML frameworks with web frameworks in terms of learning curve"
)
```

### Metadata Filtering

```python
from llama_index.core.vector_stores import MetadataFilters, FilterCondition

documents = reader.load_data(query="tech news", num_results=20)
index = VectorStoreIndex.from_documents(documents)

# Query with metadata filter
filters = MetadataFilters.from_dicts([
    {"key": "domain", "value": "techcrunch.com"}
])

query_engine = index.as_query_engine(filters=filters)
response = query_engine.query("What's the latest from TechCrunch?")
```
