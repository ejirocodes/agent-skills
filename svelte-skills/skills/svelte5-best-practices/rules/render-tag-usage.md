---
title: Rendering Snippets with {@render}
impact: HIGH
impactDescription: correctly render snippets and children in Svelte 5
type: capability
tags: @render, snippet, children, optional rendering
---

# Rendering Snippets with {@render}

**Impact: HIGH** - correctly render snippets and children in Svelte 5

Snippets must be rendered using the `{@render}` tag. This is different from Svelte 4's automatic slot rendering.

## Symptoms

- Snippets defined but not displayed
- Errors when rendering optional children
- "snippet is not a function" errors
- Content not appearing in component

## Root Cause

In Svelte 5, snippets are functions that must be explicitly called with `{@render}`. Optional snippets need null-safe calling.

## Fix

### Basic Snippet Rendering

**Incorrect (trying to use snippet directly):**

```svelte
<script>
  let { children } = $props();
</script>

<div>
  {children} <!-- Won't work - snippets are functions -->
</div>
```

**Correct (using {@render}):**

```svelte
<script>
  let { children } = $props();
</script>

<div>
  {@render children()}
</div>
```

### Optional Snippets

Always use optional chaining for snippets that might not be provided:

**Incorrect (crashes if children not provided):**

```svelte
<script>
  let { children } = $props();
</script>

{@render children()} <!-- Error if children is undefined -->
```

**Correct (safe optional rendering):**

```svelte
<script>
  let { children } = $props();
</script>

{@render children?.()}
```

### Conditional Rendering

```svelte
<script>
  let { header, children, footer } = $props();
</script>

{#if header}
  <header>{@render header()}</header>
{/if}

<main>{@render children?.()}</main>

{#if footer}
  <footer>{@render footer()}</footer>
{/if}
```

### Rendering with Arguments

Pass arguments to snippets:

```svelte
<script>
  let { items, itemTemplate } = $props();
</script>

<ul>
  {#each items as item, index}
    <li>{@render itemTemplate(item, index)}</li>
  {/each}
</ul>

<!-- Usage -->
<List {items}>
  {#snippet itemTemplate(item, index)}
    <span>{index + 1}. {item.name}</span>
  {/snippet}
</List>
```

### Fallback Content

Provide fallback when snippet is not defined:

```svelte
<script>
  let { children } = $props();
</script>

{#if children}
  {@render children()}
{:else}
  <p>No content provided</p>
{/if}
```

### Rendering Multiple Times

Snippets can be rendered multiple times:

```svelte
<script>
  let { icon } = $props();
</script>

<button>
  {@render icon?.()} <!-- Before text -->
  <span>Click me</span>
  {@render icon?.()} <!-- After text -->
</button>
```

### Rendering Dynamic Snippets

Store snippets in variables and render conditionally:

```svelte
<script>
  let { normalView, editView, isEditing } = $props();

  let currentView = $derived(isEditing ? editView : normalView);
</script>

{@render currentView?.()}
```

### TypeScript with @render

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    children: Snippet;
    header?: Snippet;
    row: Snippet<[data: { id: number; name: string }]>;
  }

  let { children, header, row }: Props = $props();
</script>

{@render header?.()}
{@render children()}
{@render row({ id: 1, name: 'Test' })}
```

### Passing Snippets to Child Components

Snippets received as props can be passed down:

```svelte
<!-- Wrapper.svelte -->
<script>
  import Inner from './Inner.svelte';
  let { itemRenderer } = $props();
</script>

<Inner {itemRenderer} />

<!-- Inner.svelte -->
<script>
  let { itemRenderer } = $props();
</script>

{#each items as item}
  {@render itemRenderer?.(item)}
{/each}
```

## Reference

- [Svelte 5 Snippets Documentation](https://svelte.dev/docs/svelte/snippet)
- [Svelte 5 {@render} Tag](https://svelte.dev/docs/svelte/@render)
