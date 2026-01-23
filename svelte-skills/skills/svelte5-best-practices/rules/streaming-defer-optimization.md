---
title: Stream Non-Critical Data with Promises
impact: MEDIUM
impactDescription: improve perceived performance with SvelteKit streaming
type: efficiency
tags: streaming, defer, promise, SSR, SvelteKit, performance
---

# Stream Non-Critical Data with Promises

**Impact: MEDIUM** - improve perceived performance with SvelteKit streaming

Return promises from load functions to stream non-essential data after initial page render.

## Symptoms

- Page blocked on slow API calls
- User sees blank screen waiting for all data
- Fast critical data delayed by slow analytics/recommendations
- Poor perceived performance despite fast core APIs

## Root Cause

Awaiting all data before returning blocks the entire page. Promises in return values stream in after initial HTML.

## How Streaming Works

1. SvelteKit sends initial HTML with awaited data
2. Promises in return value stream in as they resolve
3. Components use `{#await}` to show loading states
4. User sees content progressively

## Pattern 1: Basic Streaming

**WRONG (blocks on slow data):**

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  const user = await fetch('/api/user').then(r => r.json()); // 100ms
  const analytics = await fetch('/api/analytics').then(r => r.json()); // 2000ms

  return { user, analytics }; // Page blocked for 2.1 seconds
};
```

**CORRECT (stream slow data):**

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // Await critical data
  const user = await fetch('/api/user').then(r => r.json());

  // Don't await - return promise
  const analytics = fetch('/api/analytics').then(r => r.json());

  return {
    user, // Available in 100ms
    analytics // Streams when ready (2000ms)
  };
};
```

## Pattern 2: Component Handling

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<!-- Renders immediately -->
<header>
  <h1>Welcome, {data.user.name}</h1>
</header>

<!-- Streams in when ready -->
<aside>
  {#await data.analytics}
    <div class="skeleton">Loading analytics...</div>
  {:then analytics}
    <div class="analytics">
      <p>Page views: {analytics.views}</p>
      <p>Visitors: {analytics.visitors}</p>
    </div>
  {:catch error}
    <div class="error">
      Failed to load analytics
    </div>
  {/await}
</aside>
```

## Pattern 3: Multiple Streamed Data

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  // Critical - await
  const product = await fetch('/api/product/123').then(r => r.json());

  // Non-critical - stream all
  return {
    product,
    reviews: fetch('/api/product/123/reviews').then(r => r.json()),
    recommendations: fetch('/api/recommendations').then(r => r.json()),
    relatedProducts: fetch('/api/products/related').then(r => r.json())
  };
};
```

```svelte
<main>
  <h1>{data.product.name}</h1>
  <p>{data.product.description}</p>
</main>

<section>
  {#await data.reviews}
    <ReviewsSkeleton />
  {:then reviews}
    <ReviewsList {reviews} />
  {/await}
</section>

<section>
  {#await data.recommendations}
    <RecommendationsSkeleton />
  {:then recommendations}
    <RecommendationsGrid {recommendations} />
  {/await}
</section>
```

## Pattern 4: Nested Promises

```ts
// +page.server.ts
export const load = async ({ fetch }) => {
  return {
    user: await fetch('/api/user').then(r => r.json()),

    // Nested object with streamed data
    dashboard: {
      charts: fetch('/api/charts').then(r => r.json()),
      tables: fetch('/api/tables').then(r => r.json())
    },

    // Array of promises
    feeds: [
      fetch('/api/feed/1').then(r => r.json()),
      fetch('/api/feed/2').then(r => r.json())
    ]
  };
};
```

## Pattern 5: Conditional Streaming

```ts
// +page.server.ts
export const load = async ({ fetch, locals }) => {
  const user = await fetch('/api/user').then(r => r.json());

  return {
    user,
    // Only stream if user is premium
    premiumFeatures: user.isPremium
      ? fetch('/api/premium-features').then(r => r.json())
      : null
  };
};
```

## Pattern 6: Error Boundaries

```svelte
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

{#await data.riskyData}
  <LoadingSpinner />
{:then result}
  <DataDisplay {result} />
{:catch error}
  <!-- Graceful degradation -->
  <FallbackContent />
  <details>
    <summary>Error details</summary>
    <pre>{error.message}</pre>
  </details>
{/await}
```

## Pattern 7: Loading States Component

```svelte
<!-- AsyncSection.svelte -->
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';

  interface Props {
    promise: Promise<T>;
    loading?: Snippet;
    error?: Snippet<[Error]>;
    children: Snippet<[T]>;
  }

  let { promise, loading, error, children }: Props = $props();
</script>

{#await promise}
  {#if loading}
    {@render loading()}
  {:else}
    <div class="loading">Loading...</div>
  {/if}
{:then data}
  {@render children(data)}
{:catch err}
  {#if error}
    {@render error(err)}
  {:else}
    <div class="error">Something went wrong</div>
  {/if}
{/await}

<!-- Usage -->
<AsyncSection promise={data.analytics}>
  {#snippet loading()}
    <AnalyticsSkeleton />
  {/snippet}

  {#snippet children(analytics)}
    <AnalyticsDisplay {analytics} />
  {/snippet}
</AsyncSection>
```

## Important Notes

### Streaming Requires JavaScript

Streamed data needs JavaScript on the client. Without JS, users wait for all promises to resolve.

### Top-Level Promises Are Awaited

```ts
// WRONG - Top-level promise is auto-awaited
export const load = async ({ fetch }) => {
  return fetch('/api/data').then(r => r.json()); // Not streamed!
};

// CORRECT - Nested promise streams
export const load = async ({ fetch }) => {
  return {
    data: fetch('/api/data').then(r => r.json()) // Streamed!
  };
};
```

### Universal vs Server Load

Streaming works with both, but server load (`+page.server.ts`) is more common for database queries.

## When to Stream

| Data Type | Stream? | Reason |
|-----------|---------|--------|
| User info | No | Critical for layout |
| Main content | No | Users came for this |
| Analytics | Yes | Not user-facing |
| Recommendations | Yes | Supplementary |
| Comments | Maybe | Important but can load later |
| Related items | Yes | Supplementary |

## Reference

- [SvelteKit Streaming](https://svelte.dev/docs/kit/load#Streaming-with-promises)
- [SvelteKit Performance](https://svelte.dev/docs/kit/performance)
