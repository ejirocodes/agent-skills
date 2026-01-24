# MCP & Tool Calling Reference

## Table of Contents

- [Claude MCP Server](#claude-mcp-server)
- [OpenAI Tool Calling](#openai-tool-calling)
- [OpenAI Compatibility Mode](#openai-compatibility-mode)
- [Anthropic Tool Use](#anthropic-tool-use)
- [CrewAI Integration](#crewai-integration)

---

## Claude MCP Server

Exa provides an MCP (Model Context Protocol) server for Claude Desktop and Claude Code.

### Installation

```bash
npm install -g @anthropic/mcp-exa
# or
npx @anthropic/mcp-exa
```

### Claude Desktop Configuration

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "exa": {
      "command": "npx",
      "args": ["@anthropic/mcp-exa"],
      "env": {
        "EXA_API_KEY": "your-api-key"
      }
    }
  }
}
```

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `exa_search` | Search the web with neural/keyword modes |
| `exa_search_and_contents` | Search and retrieve page contents |
| `exa_find_similar` | Find pages similar to a URL |
| `exa_get_contents` | Get contents for known URLs |

### Claude Code Usage

Once configured, Claude can use Exa directly:

```
User: Search for the latest React 19 features

Claude: I'll search for that using Exa.
[Uses exa_search_and_contents tool]
Based on the search results, React 19 includes...
```

---

## OpenAI Tool Calling

### Function Definition

```python
import openai
from exa_py import Exa

exa = Exa()

tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Search the web for current information",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The search query"
                    },
                    "num_results": {
                        "type": "integer",
                        "description": "Number of results (default: 5)"
                    }
                },
                "required": ["query"]
            }
        }
    }
]

def handle_tool_call(tool_call):
    if tool_call.function.name == "web_search":
        args = json.loads(tool_call.function.arguments)
        results = exa.search_and_contents(
            args["query"],
            num_results=args.get("num_results", 5),
            text=True,
            highlights=True
        )
        return json.dumps([{
            "title": r.title,
            "url": r.url,
            "content": r.highlights or [r.text[:500]]
        } for r in results.results])
```

### Complete Example

```python
import openai
import json
from exa_py import Exa

client = openai.OpenAI()
exa = Exa()

def chat_with_search(user_message):
    messages = [{"role": "user", "content": user_message}]

    response = client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        tools=tools
    )

    # Handle tool calls
    while response.choices[0].message.tool_calls:
        tool_calls = response.choices[0].message.tool_calls
        messages.append(response.choices[0].message)

        for tool_call in tool_calls:
            result = handle_tool_call(tool_call)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })

        response = client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            tools=tools
        )

    return response.choices[0].message.content

answer = chat_with_search("What are the latest AI developments?")
```

---

## OpenAI Compatibility Mode

Exa provides an OpenAI-compatible endpoint for drop-in replacement.

### Basic Usage

```python
from openai import OpenAI

# Point to Exa's OpenAI-compatible endpoint
client = OpenAI(
    base_url="https://api.exa.ai/v1",
    api_key="your-exa-api-key"
)

response = client.chat.completions.create(
    model="exa",
    messages=[
        {"role": "user", "content": "What are the latest trends in AI?"}
    ]
)

print(response.choices[0].message.content)
```

### With Streaming

```python
stream = client.chat.completions.create(
    model="exa",
    messages=[
        {"role": "user", "content": "Summarize recent AI news"}
    ],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### TypeScript

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.exa.ai/v1",
  apiKey: process.env.EXA_API_KEY,
});

const response = await client.chat.completions.create({
  model: "exa",
  messages: [{ role: "user", content: "What's happening in tech?" }],
});
```

---

## Anthropic Tool Use

### Tool Definition

```python
import anthropic
from exa_py import Exa

client = anthropic.Anthropic()
exa = Exa()

tools = [
    {
        "name": "web_search",
        "description": "Search the web for information using Exa",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                },
                "num_results": {
                    "type": "integer",
                    "description": "Number of results",
                    "default": 5
                }
            },
            "required": ["query"]
        }
    }
]
```

### Complete Flow

```python
def chat_with_claude_and_exa(user_message):
    messages = [{"role": "user", "content": user_message}]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )

    # Handle tool use
    while response.stop_reason == "tool_use":
        tool_use = next(
            block for block in response.content
            if block.type == "tool_use"
        )

        # Execute Exa search
        results = exa.search_and_contents(
            tool_use.input["query"],
            num_results=tool_use.input.get("num_results", 5),
            text=True,
            highlights=True
        )

        tool_result = [{
            "title": r.title,
            "url": r.url,
            "content": r.highlights or [r.text[:500]]
        } for r in results.results]

        # Continue conversation
        messages = [
            {"role": "user", "content": user_message},
            {"role": "assistant", "content": response.content},
            {
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": json.dumps(tool_result)
                }]
            }
        ]

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

    return response.content[0].text
```

---

## CrewAI Integration

### Installation

```bash
pip install crewai crewai-tools
```

### Exa Tool

```python
from crewai import Agent, Task, Crew
from crewai_tools import ExaSearchTool

# Create Exa search tool
search_tool = ExaSearchTool(
    api_key="your-exa-key",
    n_results=5
)

# Create agent with Exa
researcher = Agent(
    role="Research Analyst",
    goal="Research and summarize topics thoroughly",
    backstory="Expert researcher with web search capabilities",
    tools=[search_tool],
    verbose=True
)

# Create task
research_task = Task(
    description="Research the latest developments in AI agents",
    agent=researcher,
    expected_output="A comprehensive summary with sources"
)

# Run crew
crew = Crew(
    agents=[researcher],
    tasks=[research_task]
)

result = crew.kickoff()
```

### Multi-Agent with Exa

```python
from crewai import Agent, Task, Crew, Process

search_tool = ExaSearchTool(api_key="your-key")

# Research agent
researcher = Agent(
    role="Researcher",
    goal="Find relevant information",
    tools=[search_tool]
)

# Writer agent
writer = Agent(
    role="Writer",
    goal="Create well-written content from research"
)

# Tasks
research = Task(
    description="Research {topic}",
    agent=researcher
)

write = Task(
    description="Write article based on research",
    agent=writer,
    context=[research]
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research, write],
    process=Process.sequential
)

result = crew.kickoff(inputs={"topic": "AI trends 2024"})
```

---

## Google ADK Integration

```python
from google.adk import Agent
from exa_py import Exa

exa = Exa()

def exa_search(query: str, num_results: int = 5) -> str:
    """Search the web using Exa."""
    results = exa.search_and_contents(
        query,
        num_results=num_results,
        text=True,
        highlights=True
    )
    return "\n\n".join([
        f"Title: {r.title}\nURL: {r.url}\nContent: {r.highlights or r.text[:500]}"
        for r in results.results
    ])

agent = Agent(
    tools=[exa_search],
    model="gemini-1.5-pro"
)

response = agent.run("What are the latest AI developments?")
```
