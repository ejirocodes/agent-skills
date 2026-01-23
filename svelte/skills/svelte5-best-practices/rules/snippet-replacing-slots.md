---
title: Replacing Slots with Snippets
impact: HIGH
impactDescription: use Svelte 5 snippets for component composition
type: capability
tags: snippet, slot, children, composition, render
---

# Replacing Slots with Snippets

**Impact: HIGH** - use Svelte 5 snippets for component composition

Svelte 5 replaces `<slot>` elements with snippets - a more powerful and type-safe composition primitive.

## Symptoms

- `<slot>` produces deprecation warnings
- Named slots don't work as expected
- Slot props pattern unclear in Svelte 5

## Root Cause

Svelte 5 introduced snippets as a replacement for slots. Snippets are more explicit, support TypeScript better, and can be passed as props.

## Fix

### Default Content (children)

**Incorrect (Svelte 4 pattern):**

```svelte
<!-- Card.svelte -->
<div class="card">
  <slot />
</div>

<!-- Usage -->
<Card>
  <p>Content here</p>
</Card>
```

**Correct (Svelte 5 pattern):**

```svelte
<!-- Card.svelte -->
<script>
  let { children } = $props();
</script>

<div class="card">
  {@render children?.()}
</div>

<!-- Usage (same as before) -->
<Card>
  <p>Content here</p>
</Card>
```

### Named Slots to Named Snippets

**Incorrect (Svelte 4 named slots):**

```svelte
<!-- Layout.svelte -->
<header><slot name="header" /></header>
<main><slot /></main>
<footer><slot name="footer" /></footer>

<!-- Usage -->
<Layout>
  <h1 slot="header">Title</h1>
  <p>Content</p>
  <span slot="footer">Footer</span>
</Layout>
```

**Correct (Svelte 5 snippets):**

```svelte
<!-- Layout.svelte -->
<script>
  let { header, children, footer } = $props();
</script>

<header>{@render header?.()}</header>
<main>{@render children?.()}</main>
<footer>{@render footer?.()}</footer>

<!-- Usage -->
<Layout>
  {#snippet header()}
    <h1>Title</h1>
  {/snippet}

  <p>Content</p>

  {#snippet footer()}
    <span>Footer</span>
  {/snippet}
</Layout>
```

### Slot Props to Snippet Parameters

**Incorrect (Svelte 4 slot props):**

```svelte
<!-- List.svelte -->
<ul>
  {#each items as item}
    <li><slot {item} /></li>
  {/each}
</ul>

<!-- Usage -->
<List {items} let:item>
  <span>{item.name}</span>
</List>
```

**Correct (Svelte 5 snippet parameters):**

```svelte
<!-- List.svelte -->
<script>
  let { items, children } = $props();
</script>

<ul>
  {#each items as item}
    <li>{@render children?.(item)}</li>
  {/each}
</ul>

<!-- Usage -->
<List {items}>
  {#snippet children(item)}
    <span>{item.name}</span>
  {/snippet}
</List>
```

### Slot Fallback to Snippet Fallback

**Incorrect (Svelte 4 fallback):**

```svelte
<slot>
  <p>Default content</p>
</slot>
```

**Correct (Svelte 5 fallback):**

```svelte
<script>
  let { children } = $props();
</script>

{#if children}
  {@render children()}
{:else}
  <p>Default content</p>
{/if}
```

### Defining Snippets Within Components

Snippets can be defined and used within the same component:

```svelte
<script>
  let items = $state([
    { id: 1, name: 'Item 1', type: 'normal' },
    { id: 2, name: 'Item 2', type: 'featured' }
  ]);
</script>

{#snippet normalItem(item)}
  <li>{item.name}</li>
{/snippet}

{#snippet featuredItem(item)}
  <li class="featured">*** {item.name} ***</li>
{/snippet}

<ul>
  {#each items as item}
    {#if item.type === 'featured'}
      {@render featuredItem(item)}
    {:else}
      {@render normalItem(item)}
    {/if}
  {/each}
</ul>
```

### TypeScript Typing

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Item {
    id: number;
    name: string;
  }

  interface Props {
    header?: Snippet;
    children: Snippet<[item: Item]>;
    footer?: Snippet;
  }

  let { header, children, footer }: Props = $props();
</script>
```

## Reference

- [Svelte 5 Snippets Documentation](https://svelte.dev/docs/svelte/snippet)
- [Migration: Slots to Snippets](https://svelte.dev/docs/svelte/v5-migration-guide#Snippets-instead-of-slots)
