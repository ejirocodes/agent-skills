# Vercel AI SDK Integration Reference

## Table of Contents

- [Installation](#installation)
- [Tool Definition](#tool-definition)
- [generateText Patterns](#generatetext-patterns)
- [Streaming Patterns](#streaming-patterns)
- [Next.js Integration](#nextjs-integration)

---

## Installation

```bash
npm install ai @ai-sdk/openai @agentic/exa
# or
pnpm add ai @ai-sdk/openai @agentic/exa
```

---

## Tool Definition

### Using @agentic/exa

```typescript
import { exa } from "@agentic/exa";

// Pre-built tool definitions
const searchTool = exa.searchAndContents;
const findSimilarTool = exa.findSimilarAndContents;
```

### Custom Tool Definition

```typescript
import { tool } from "ai";
import Exa from "exa-js";
import { z } from "zod";

const exaClient = new Exa(process.env.EXA_API_KEY);

const searchWeb = tool({
  description: "Search the web for information",
  parameters: z.object({
    query: z.string().describe("The search query"),
    numResults: z.number().default(5).describe("Number of results"),
  }),
  execute: async ({ query, numResults }) => {
    const results = await exaClient.searchAndContents(query, {
      numResults,
      text: true,
      highlights: true,
    });
    return results.results.map((r) => ({
      title: r.title,
      url: r.url,
      content: r.highlights?.join("\n") || r.text?.slice(0, 500),
    }));
  },
});
```

---

## generateText Patterns

### Basic Text Generation with Search

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import { exa } from "@agentic/exa";

const result = await generateText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  maxSteps: 5,
  prompt: "What are the latest developments in AI agents?",
});

console.log(result.text);
```

### With Custom System Prompt

```typescript
const result = await generateText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  system: `You are a research assistant. When searching, always:
1. Use specific queries
2. Include sources in your response
3. Synthesize information from multiple results`,
  prompt: "Research the current state of WebAssembly adoption",
});
```

### Multiple Tools

```typescript
import { exa } from "@agentic/exa";

const result = await generateText({
  model: openai("gpt-4"),
  tools: {
    search: exa.searchAndContents,
    findSimilar: exa.findSimilarAndContents,
  },
  maxSteps: 10,
  prompt:
    "Find articles about React Server Components and then find similar articles",
});
```

---

## Streaming Patterns

### Basic Streaming

```typescript
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { exa } from "@agentic/exa";

const result = await streamText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  maxSteps: 5,
  prompt: "Summarize the latest TypeScript features",
});

for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

### With Tool Call Handling

```typescript
import { streamText } from "ai";

const result = await streamText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  maxSteps: 5,
  prompt: "Research quantum computing applications",
  onStepFinish: ({ stepType, toolCalls, toolResults }) => {
    if (stepType === "tool-result") {
      console.log("Search completed:", toolResults);
    }
  },
});
```

---

## Next.js Integration

### API Route (App Router)

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { exa } from "@agentic/exa";

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: openai("gpt-4"),
    tools: { search: exa.searchAndContents },
    maxSteps: 5,
    messages,
  });

  return result.toDataStreamResponse();
}
```

### Client Component

```typescript
// components/Chat.tsx
"use client";

import { useChat } from "ai/react";

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: "/api/chat",
    });

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          <strong>{m.role}:</strong> {m.content}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask anything..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>
          Send
        </button>
      </form>
    </div>
  );
}
```

### With Tool Result Display

```typescript
// components/ChatWithTools.tsx
"use client";

import { useChat } from "ai/react";

export function ChatWithTools() {
  const { messages, input, handleInputChange, handleSubmit } = useChat();

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          <strong>{m.role}:</strong>
          {m.content}

          {/* Display tool invocations */}
          {m.toolInvocations?.map((tool, i) => (
            <div key={i} className="tool-result">
              <small>Searched: {tool.args.query}</small>
              {tool.state === "result" && (
                <ul>
                  {tool.result.map((r: any, j: number) => (
                    <li key={j}>
                      <a href={r.url}>{r.title}</a>
                    </li>
                  ))}
                </ul>
              )}
            </div>
          ))}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

### Edge Runtime Support

```typescript
// app/api/search/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { exa } from "@agentic/exa";

export const runtime = "edge";

export async function POST(req: Request) {
  const { query } = await req.json();

  const result = await streamText({
    model: openai("gpt-4"),
    tools: { search: exa.searchAndContents },
    maxSteps: 3,
    prompt: query,
  });

  return result.toDataStreamResponse();
}
```

---

## Error Handling

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import { exa } from "@agentic/exa";

try {
  const result = await generateText({
    model: openai("gpt-4"),
    tools: { search: exa.searchAndContents },
    prompt: "Search for...",
  });
} catch (error) {
  if (error.name === "AI_ToolExecutionError") {
    console.error("Search failed:", error.message);
    // Handle Exa API errors
  } else {
    throw error;
  }
}
```

---

## Best Practices

### Limit Max Steps

```typescript
// Prevent infinite tool loops
const result = await generateText({
  model: openai("gpt-4"),
  tools: { search: exa.searchAndContents },
  maxSteps: 5, // Reasonable limit
  prompt: "...",
});
```

### Cache Search Results

```typescript
import { unstable_cache } from "next/cache";

const cachedSearch = unstable_cache(
  async (query: string) => {
    const exa = new Exa(process.env.EXA_API_KEY);
    return exa.searchAndContents(query, { numResults: 5, text: true });
  },
  ["exa-search"],
  { revalidate: 3600 } // 1 hour
);
```

### Type-Safe Results

```typescript
import { z } from "zod";

const SearchResultSchema = z.object({
  title: z.string(),
  url: z.string(),
  content: z.string(),
});

const searchTool = tool({
  description: "Search the web",
  parameters: z.object({ query: z.string() }),
  execute: async ({ query }) => {
    const results = await exaClient.searchAndContents(query, {
      numResults: 5,
      text: true,
    });
    return results.results.map((r) =>
      SearchResultSchema.parse({
        title: r.title,
        url: r.url,
        content: r.text?.slice(0, 500) || "",
      })
    );
  },
});
```
