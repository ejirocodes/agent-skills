---
title: TypeScript Props Typing with $props
impact: HIGH
impactDescription: correctly type component props in Svelte 5
type: capability
tags: TypeScript, Props, interface, $props, typing
---

# TypeScript Props Typing with $props

**Impact: HIGH** - correctly type component props in Svelte 5

Svelte 5's `$props()` rune requires specific patterns for TypeScript typing.

## Symptoms

- Type errors with $props destructuring
- Props not getting proper autocomplete
- "Type 'X' is not assignable to type 'Y'" errors
- Children and snippets not typed correctly

## Root Cause

The `$props()` rune returns a typed object. TypeScript needs proper interface definitions and typing patterns.

## Fix

### Basic Props Typing

**Inline typing:**

```svelte
<script lang="ts">
  let { name, count = 0 }: { name: string; count?: number } = $props();
</script>
```

**Interface typing (recommended for complex props):**

```svelte
<script lang="ts">
  interface Props {
    name: string;
    count?: number;
    disabled?: boolean;
  }

  let { name, count = 0, disabled = false }: Props = $props();
</script>
```

### Required vs Optional Props

```svelte
<script lang="ts">
  interface Props {
    id: string;           // Required - no default
    title: string;        // Required
    count?: number;       // Optional - needs default in destructuring
    visible?: boolean;    // Optional
  }

  let {
    id,
    title,
    count = 0,
    visible = true
  }: Props = $props();
</script>
```

### Children and Snippets Typing

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    children: Snippet;                    // Required children
    header?: Snippet;                     // Optional snippet
    row: Snippet<[data: RowData]>;       // Snippet with parameter
    cell?: Snippet<[value: string, index: number]>;  // Multiple params
  }

  interface RowData {
    id: number;
    name: string;
  }

  let { children, header, row, cell }: Props = $props();
</script>

{@render header?.()}
{@render children()}
```

### Callback Props Typing

```svelte
<script lang="ts">
  interface Props {
    value: string;
    onchange?: (value: string) => void;
    onsubmit?: (data: FormData) => Promise<void>;
    onclick?: (event: MouseEvent) => void;
  }

  let { value, onchange, onsubmit, onclick }: Props = $props();
</script>
```

### Rest Props with HTML Attributes

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements';

  interface Props extends HTMLButtonAttributes {
    variant?: 'primary' | 'secondary';
    loading?: boolean;
  }

  let { variant = 'primary', loading = false, ...rest }: Props = $props();
</script>

<button class={variant} disabled={loading} {...rest}>
  {#if loading}Loading...{:else}{@render children?.()}{/if}
</button>
```

### Input Element Props

```svelte
<script lang="ts">
  import type { HTMLInputAttributes } from 'svelte/elements';

  interface Props extends Omit<HTMLInputAttributes, 'value'> {
    value?: string;
    label: string;
    error?: string;
  }

  let { value = $bindable(''), label, error, ...rest }: Props = $props();
</script>

<label>
  {label}
  <input bind:value {...rest} />
  {#if error}<span class="error">{error}</span>{/if}
</label>
```

### Generic Props

```svelte
<script lang="ts" generics="T">
  interface Props<T> {
    items: T[];
    selected?: T;
    onselect?: (item: T) => void;
    children: Snippet<[item: T]>;
  }

  let { items, selected, onselect, children }: Props<T> = $props();
</script>

{#each items as item}
  <div onclick={() => onselect?.(item)}>
    {@render children(item)}
  </div>
{/each}
```

### Union Type Props

```svelte
<script lang="ts">
  type ButtonVariant = 'primary' | 'secondary' | 'danger';
  type ButtonSize = 'sm' | 'md' | 'lg';

  interface Props {
    variant?: ButtonVariant;
    size?: ButtonSize;
    children: Snippet;
  }

  let { variant = 'primary', size = 'md', children }: Props = $props();
</script>

<button class="{variant} {size}">
  {@render children()}
</button>
```

### Discriminated Union Props

```svelte
<script lang="ts">
  type Props =
    | { type: 'link'; href: string; children: Snippet }
    | { type: 'button'; onclick: () => void; children: Snippet };

  let props: Props = $props();
</script>

{#if props.type === 'link'}
  <a href={props.href}>{@render props.children()}</a>
{:else}
  <button onclick={props.onclick}>{@render props.children()}</button>
{/if}
```

## Reference

- [Svelte TypeScript Documentation](https://svelte.dev/docs/svelte/typescript)
- [Svelte 5 $props Documentation](https://svelte.dev/docs/svelte/$props)
