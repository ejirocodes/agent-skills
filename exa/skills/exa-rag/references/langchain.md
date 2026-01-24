# LangChain Integration Reference

## Table of Contents

- [Installation](#installation)
- [ExaSearchRetriever](#exasearchretriever)
- [ExaSearchResults Tool](#exasearchresults-tool)
- [RAG Chain Patterns](#rag-chain-patterns)
- [Agent Integration](#agent-integration)

---

## Installation

```bash
pip install langchain-exa
```

---

## ExaSearchRetriever

The primary way to use Exa with LangChain for RAG.

### Basic Usage

```python
from langchain_exa import ExaSearchRetriever

retriever = ExaSearchRetriever(
    exa_api_key="your-key",  # or set EXA_API_KEY env var
    k=5
)

docs = retriever.invoke("latest developments in quantum computing")
for doc in docs:
    print(f"URL: {doc.metadata['url']}")
    print(f"Title: {doc.metadata['title']}")
    print(f"Content: {doc.page_content[:200]}...")
```

### With Highlights

```python
retriever = ExaSearchRetriever(
    exa_api_key="your-key",
    k=5,
    highlights=True,
    num_sentences=3
)

docs = retriever.invoke("React Server Components best practices")
# docs contain highlight snippets instead of full text
```

### With Filters

```python
retriever = ExaSearchRetriever(
    exa_api_key="your-key",
    k=10,
    include_domains=["github.com", "stackoverflow.com"],
    start_published_date="2024-01-01",
    type="neural"
)
```

### Configuration Options

| Parameter | Type | Description |
|-----------|------|-------------|
| `exa_api_key` | str | API key (or use env var) |
| `k` | int | Number of results |
| `type` | str | "auto", "neural", or "keyword" |
| `include_domains` | list | Limit to these domains |
| `exclude_domains` | list | Exclude these domains |
| `start_published_date` | str | Start date filter (YYYY-MM-DD) |
| `end_published_date` | str | End date filter |
| `highlights` | bool | Return highlights instead of full text |
| `num_sentences` | int | Sentences per highlight |
| `text_length_limit` | int | Max characters for text |

---

## ExaSearchResults Tool

For LangChain agents that need web search as a tool.

### Basic Tool

```python
from langchain_exa import ExaSearchResults
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain_openai import ChatOpenAI

# Create the tool
search_tool = ExaSearchResults(
    exa_api_key="your-key",
    max_results=5
)

# Use in an agent
llm = ChatOpenAI(model="gpt-4")
agent = create_openai_functions_agent(llm, [search_tool], prompt)
executor = AgentExecutor(agent=agent, tools=[search_tool])

result = executor.invoke({"input": "What are the latest AI news today?"})
```

### Tool with Custom Configuration

```python
search_tool = ExaSearchResults(
    exa_api_key="your-key",
    max_results=10,
    text_length_limit=1000,
    highlights=True,
    include_domains=["techcrunch.com", "wired.com"]
)
```

---

## RAG Chain Patterns

### Basic RAG Chain

```python
from langchain_exa import ExaSearchRetriever
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

retriever = ExaSearchRetriever(exa_api_key="your-key", k=5, highlights=True)
llm = ChatOpenAI(model="gpt-4")

prompt = ChatPromptTemplate.from_template("""
Answer the question based on the following context. Include source URLs.

Context:
{context}

Question: {question}

Answer:
""")

def format_docs(docs):
    return "\n\n".join(
        f"Source: {doc.metadata['url']}\n{doc.page_content}"
        for doc in docs
    )

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = chain.invoke("What are the key features of Python 3.12?")
```

### RAG with Citations

```python
from langchain_core.pydantic_v1 import BaseModel, Field
from typing import List

class Citation(BaseModel):
    url: str
    title: str
    snippet: str

class AnswerWithCitations(BaseModel):
    answer: str
    citations: List[Citation]

def create_cited_answer(docs, question):
    context = format_docs(docs)
    llm_with_structure = llm.with_structured_output(AnswerWithCitations)

    prompt = f"""
    Based on the context below, answer the question and cite your sources.

    Context:
    {context}

    Question: {question}
    """

    return llm_with_structure.invoke(prompt)
```

### Conversational RAG

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationalRetrievalChain

retriever = ExaSearchRetriever(exa_api_key="your-key", k=5)
memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)

chain = ConversationalRetrievalChain.from_llm(
    llm=ChatOpenAI(model="gpt-4"),
    retriever=retriever,
    memory=memory
)

response = chain.invoke({"question": "What is LangChain?"})
response = chain.invoke({"question": "How does it compare to LlamaIndex?"})
```

---

## Agent Integration

### ReAct Agent with Exa

```python
from langchain_exa import ExaSearchResults
from langchain.agents import AgentExecutor, create_react_agent
from langchain_openai import ChatOpenAI
from langchain import hub

# Get ReAct prompt
prompt = hub.pull("hwchase17/react")

# Create tools
search = ExaSearchResults(exa_api_key="your-key", max_results=5)
tools = [search]

# Create agent
llm = ChatOpenAI(model="gpt-4")
agent = create_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({
    "input": "Research the latest developments in AI agents and summarize the key trends"
})
```

### Multi-Tool Agent

```python
from langchain_exa import ExaSearchResults, ExaFindSimilar

search_tool = ExaSearchResults(
    exa_api_key="your-key",
    name="web_search",
    description="Search the web for current information"
)

similar_tool = ExaFindSimilar(
    exa_api_key="your-key",
    name="find_similar",
    description="Find pages similar to a given URL"
)

tools = [search_tool, similar_tool]
# Use with your preferred agent type
```

### LangGraph Integration

```python
from langgraph.graph import StateGraph, END
from langchain_exa import ExaSearchRetriever

retriever = ExaSearchRetriever(exa_api_key="your-key", k=5)

def search_node(state):
    query = state["query"]
    docs = retriever.invoke(query)
    return {"documents": docs}

def generate_node(state):
    docs = state["documents"]
    # Generate answer using docs
    return {"answer": answer}

# Build graph
workflow = StateGraph(dict)
workflow.add_node("search", search_node)
workflow.add_node("generate", generate_node)
workflow.set_entry_point("search")
workflow.add_edge("search", "generate")
workflow.add_edge("generate", END)

app = workflow.compile()
```
