---
title: Prevent Sequential API Calls in Load Functions
impact: HIGH
impactDescription: avoid performance-killing request waterfalls in SvelteKit
type: efficiency
tags: waterfall, parallel, load, performance, SvelteKit, fetch
---

# Prevent Sequential API Calls in Load Functions

**Impact: HIGH** - avoid performance-killing request waterfalls in SvelteKit

Sequential (waterfall) API calls in load functions multiply latency. Use parallel requests and streaming.

## Symptoms

- Slow page loads despite fast individual APIs
- Load time equals sum of all API calls
- Server waiting between requests
- User sees loading spinner longer than necessary

## Root Cause

`await`ing each request before starting the next creates a waterfall where total time = sum of all request times.

## Anti-Pattern: Waterfall

**WRONG (sequential - 3 seconds total):**

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // Request 1: 1 second
  const user = await fetch('/api/user').then(r => r.json());

  // Request 2: 1 second (waits for request 1)
  const posts = await fetch(`/api/users/${user.id}/posts`).then(r => r.json());

  // Request 3: 1 second (waits for request 2)
  const comments = await fetch('/api/comments').then(r => r.json());

  return { user, posts, comments }; // Total: 3 seconds
};
```

## Pattern 1: Parallel with Promise.all

**CORRECT (parallel - 1 second total):**

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // All requests start simultaneously
  const [user, posts, comments] = await Promise.all([
    fetch('/api/user').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
    fetch('/api/comments').then(r => r.json())
  ]);

  return { user, posts, comments }; // Total: ~1 second (slowest request)
};
```

## Pattern 2: Partial Dependencies

When some data depends on others:

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // First: Get user (needed for other requests)
  const user = await fetch('/api/user').then(r => r.json());

  // Then: Parallel requests that depend on user
  const [posts, followers, settings] = await Promise.all([
    fetch(`/api/users/${user.id}/posts`).then(r => r.json()),
    fetch(`/api/users/${user.id}/followers`).then(r => r.json()),
    fetch(`/api/users/${user.id}/settings`).then(r => r.json())
  ]);

  return { user, posts, followers, settings };
};
```

## Pattern 3: Streaming Non-Critical Data

Return promises for data that can load after initial render:

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // Critical data - await immediately
  const user = await fetch('/api/user').then(r => r.json());

  // Non-critical - return promise for streaming
  const recommendations = fetch('/api/recommendations').then(r => r.json());
  const analytics = fetch('/api/analytics').then(r => r.json());

  return {
    user, // Available immediately
    recommendations, // Streams in
    analytics // Streams in
  };
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<!-- Shows immediately -->
<h1>Welcome, {data.user.name}</h1>

<!-- Streams in when ready -->
{#await data.recommendations}
  <p>Loading recommendations...</p>
{:then recommendations}
  <ul>
    {#each recommendations as item}
      <li>{item.title}</li>
    {/each}
  </ul>
{:catch}
  <p>Failed to load recommendations</p>
{/await}
```

## Pattern 4: Nested Streaming

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  return {
    // Immediate
    user: await fetch('/api/user').then(r => r.json()),

    // Nested streaming object
    dashboard: {
      stats: fetch('/api/stats').then(r => r.json()),
      charts: fetch('/api/charts').then(r => r.json()),
      alerts: fetch('/api/alerts').then(r => r.json())
    }
  };
};
```

## Pattern 5: Combine in Database Query

Best performance - let database do the joining:

**WRONG (N+1 query problem):**

```ts
export const load = async ({ fetch }) => {
  const posts = await fetch('/api/posts').then(r => r.json());

  // N additional requests!
  const postsWithAuthors = await Promise.all(
    posts.map(async post => ({
      ...post,
      author: await fetch(`/api/users/${post.authorId}`).then(r => r.json())
    }))
  );

  return { posts: postsWithAuthors };
};
```

**CORRECT (single query with join):**

```ts
export const load = async ({ fetch }) => {
  // Single request with data joined server-side
  const posts = await fetch('/api/posts?include=author').then(r => r.json());
  return { posts };
};
```

## Pattern 6: Using depends() for Invalidation

```ts
// +page.server.ts
export const load = async ({ fetch, depends }) => {
  depends('app:posts'); // Declare dependency

  const [user, posts] = await Promise.all([
    fetch('/api/user').then(r => r.json()),
    fetch('/api/posts').then(r => r.json())
  ]);

  return { user, posts };
};

// Invalidate from component:
// invalidate('app:posts')
```

## Performance Comparison

| Pattern | 3 APIs × 1s each |
|---------|------------------|
| Sequential | 3 seconds |
| Parallel (Promise.all) | 1 second |
| Streaming (non-critical) | 0s initial, streams rest |

## Checklist

- [ ] No `await` between independent fetch calls
- [ ] Use `Promise.all` for parallel requests
- [ ] Stream non-critical data as promises
- [ ] Combine related data in single API endpoint
- [ ] Check for N+1 query patterns

## Reference

- [SvelteKit Streaming](https://svelte.dev/docs/kit/load#Streaming-with-promises)
- [SvelteKit Performance](https://svelte.dev/docs/kit/performance)
