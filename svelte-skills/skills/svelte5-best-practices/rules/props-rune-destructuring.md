---
title: Receiving Props with $props Rune
impact: HIGH
impactDescription: receive component props correctly in Svelte 5
type: capability
tags: $props, props, component, destructuring, defaults, export let
---

# Receiving Props with $props Rune

**Impact: HIGH** - receive component props correctly in Svelte 5

Svelte 5 replaces `export let` with the `$props()` rune for declaring component props.

## Symptoms

- `export let prop` produces deprecation warnings
- Props don't have default values
- Rest props pattern unclear
- TypeScript errors with prop types

## Root Cause

Svelte 5 introduced `$props()` as a single rune to handle all component props, replacing the `export let` pattern from Svelte 4.

## Fix

**Incorrect (Svelte 4 pattern):**

```svelte
<script>
  export let name;
  export let count = 0;
  export let disabled = false;
</script>
```

**Correct (Svelte 5 pattern):**

```svelte
<script>
  let { name, count = 0, disabled = false } = $props();
</script>
```

## Required vs Optional Props

```svelte
<script>
  let {
    // Required prop (no default)
    id,
    // Optional props (with defaults)
    title = 'Default Title',
    visible = true
  } = $props();
</script>
```

## Rest Props Pattern

Collect remaining props with rest syntax:

```svelte
<script>
  let { class: className, children, ...restProps } = $props();
</script>

<div class={className} {...restProps}>
  {@render children?.()}
</div>
```

## Renaming Props

Rename props that conflict with reserved words:

```svelte
<script>
  let {
    class: className,  // 'class' is reserved
    for: htmlFor,      // 'for' is reserved
    ...rest
  } = $props();
</script>

<label class={className} for={htmlFor} {...rest}>
  {@render children?.()}
</label>
```

## TypeScript Props

```svelte
<script lang="ts">
  interface Props {
    name: string;
    count?: number;
    disabled?: boolean;
    onClick?: (event: MouseEvent) => void;
  }

  let { name, count = 0, disabled = false, onClick }: Props = $props();
</script>
```

## Inline TypeScript Typing

```svelte
<script lang="ts">
  let {
    name,
    count = 0,
    disabled = false
  }: {
    name: string;
    count?: number;
    disabled?: boolean;
  } = $props();
</script>
```

## Props with Children

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    title: string;
    children: Snippet;
    footer?: Snippet;
  }

  let { title, children, footer }: Props = $props();
</script>

<article>
  <h1>{title}</h1>
  <main>{@render children()}</main>
  {#if footer}
    <footer>{@render footer()}</footer>
  {/if}
</article>
```

## Reactive Props

Props are automatically reactive:

```svelte
<script>
  let { count } = $props();

  // Derived from prop
  let doubled = $derived(count * 2);

  // Effect that runs when prop changes
  $effect(() => {
    console.log('Count changed:', count);
  });
</script>
```

## Reference

- [Svelte 5 $props Documentation](https://svelte.dev/docs/svelte/$props)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
