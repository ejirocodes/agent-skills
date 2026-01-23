---
title: Creating Generic Typed Components
impact: MEDIUM
impactDescription: create reusable generic components in Svelte 5
type: capability
tags: generics, generic component, TypeScript, type parameter
---

# Creating Generic Typed Components

**Impact: MEDIUM** - create reusable generic components in Svelte 5

Svelte 5 introduces the `generics` attribute for creating type-safe reusable components like `List<T>`.

## Symptoms

- Can't create components that work with multiple types
- Type information lost when passing items to components
- Unclear how to use generics in Svelte

## Root Cause

Svelte 5 uses a special `generics` attribute in the script tag to declare type parameters for components.

## Fix

### Basic Generic Component

```svelte
<!-- List.svelte -->
<script lang="ts" generics="T">
  interface Props {
    items: T[];
    children: import('svelte').Snippet<[item: T, index: number]>;
  }

  let { items, children }: Props = $props();
</script>

<ul>
  {#each items as item, index}
    <li>{@render children(item, index)}</li>
  {/each}
</ul>

<!-- Usage -->
<script lang="ts">
  import List from './List.svelte';

  interface User {
    id: number;
    name: string;
  }

  let users: User[] = $state([
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' }
  ]);
</script>

<List items={users}>
  {#snippet children(user, index)}
    <span>{index + 1}. {user.name}</span> <!-- Fully typed! -->
  {/snippet}
</List>
```

### Multiple Type Parameters

```svelte
<!-- KeyValueList.svelte -->
<script lang="ts" generics="K, V">
  interface Props {
    entries: [K, V][];
    renderKey: import('svelte').Snippet<[key: K]>;
    renderValue: import('svelte').Snippet<[value: V]>;
  }

  let { entries, renderKey, renderValue }: Props = $props();
</script>

<dl>
  {#each entries as [key, value]}
    <dt>{@render renderKey(key)}</dt>
    <dd>{@render renderValue(value)}</dd>
  {/each}
</dl>
```

### Constrained Generics

```svelte
<!-- Selectable.svelte -->
<script lang="ts" generics="T extends { id: string | number }">
  interface Props {
    items: T[];
    selected?: T;
    onselect?: (item: T) => void;
    children: import('svelte').Snippet<[item: T, isSelected: boolean]>;
  }

  let { items, selected, onselect, children }: Props = $props();
</script>

{#each items as item}
  <div
    class:selected={selected?.id === item.id}
    onclick={() => onselect?.(item)}
  >
    {@render children(item, selected?.id === item.id)}
  </div>
{/each}
```

### Generic with Default Type

```svelte
<!-- DataGrid.svelte -->
<script lang="ts" generics="T = Record<string, unknown>">
  interface Props {
    data: T[];
    columns: (keyof T)[];
    children?: import('svelte').Snippet<[row: T, column: keyof T]>;
  }

  let { data, columns, children }: Props = $props();
</script>

<table>
  <thead>
    <tr>
      {#each columns as col}
        <th>{String(col)}</th>
      {/each}
    </tr>
  </thead>
  <tbody>
    {#each data as row}
      <tr>
        {#each columns as col}
          <td>
            {#if children}
              {@render children(row, col)}
            {:else}
              {row[col]}
            {/if}
          </td>
        {/each}
      </tr>
    {/each}
  </tbody>
</table>
```

### Generic Select Component

```svelte
<!-- Select.svelte -->
<script lang="ts" generics="T">
  interface Props {
    options: T[];
    value?: T;
    getLabel: (option: T) => string;
    getValue: (option: T) => string;
    onchange?: (selected: T | undefined) => void;
    placeholder?: string;
  }

  let {
    options,
    value = $bindable(),
    getLabel,
    getValue,
    onchange,
    placeholder = 'Select...'
  }: Props = $props();

  function handleChange(e: Event) {
    const selectedValue = (e.target as HTMLSelectElement).value;
    const selected = options.find(o => getValue(o) === selectedValue);
    value = selected;
    onchange?.(selected);
  }
</script>

<select onchange={handleChange}>
  <option value="">{placeholder}</option>
  {#each options as option}
    <option
      value={getValue(option)}
      selected={value && getValue(value) === getValue(option)}
    >
      {getLabel(option)}
    </option>
  {/each}
</select>

<!-- Usage -->
<script lang="ts">
  import Select from './Select.svelte';

  interface Country {
    code: string;
    name: string;
  }

  let countries: Country[] = [
    { code: 'US', name: 'United States' },
    { code: 'UK', name: 'United Kingdom' }
  ];

  let selectedCountry: Country | undefined = $state();
</script>

<Select
  options={countries}
  bind:value={selectedCountry}
  getLabel={(c) => c.name}
  getValue={(c) => c.code}
/>
```

### Generic Async Component

```svelte
<!-- AsyncData.svelte -->
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';

  interface Props {
    promise: Promise<T>;
    loading?: Snippet;
    error?: Snippet<[error: Error]>;
    children: Snippet<[data: T]>;
  }

  let { promise, loading, error, children }: Props = $props();
</script>

{#await promise}
  {#if loading}
    {@render loading()}
  {:else}
    <p>Loading...</p>
  {/if}
{:then data}
  {@render children(data)}
{:catch err}
  {#if error}
    {@render error(err)}
  {:else}
    <p>Error: {err.message}</p>
  {/if}
{/await}
```

## Reference

- [Svelte TypeScript Documentation](https://svelte.dev/docs/svelte/typescript)
- [TypeScript Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
