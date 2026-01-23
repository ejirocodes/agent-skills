---
title: Callback Props Instead of Event Dispatching
impact: HIGH
impactDescription: use callback props for component events in Svelte 5
type: capability
tags: callback, createEventDispatcher, events, props, dispatch, custom events
---

# Callback Props Instead of Event Dispatching

**Impact: HIGH** - use callback props for component events in Svelte 5

Svelte 5 deprecates `createEventDispatcher` in favor of callback props for component-to-parent communication.

## Symptoms

- `createEventDispatcher` produces deprecation warnings
- Custom events from child components not working
- Unclear how to emit events from components

## Root Cause

Svelte 5 favors explicit callback props over the dispatcher pattern. This provides better TypeScript support and clearer component contracts.

## Fix

### Basic Event Pattern

**Incorrect (Svelte 4 dispatcher):**

```svelte
<!-- Button.svelte -->
<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  function handleClick() {
    dispatch('click', { timestamp: Date.now() });
  }
</script>

<button on:click={handleClick}>Click</button>

<!-- Parent.svelte -->
<Button on:click={(e) => console.log(e.detail)} />
```

**Correct (Svelte 5 callback props):**

```svelte
<!-- Button.svelte -->
<script>
  let { onclick } = $props();
</script>

<button onclick={() => onclick?.({ timestamp: Date.now() })}>
  Click
</button>

<!-- Parent.svelte -->
<Button onclick={(data) => console.log(data)} />
```

### Multiple Callbacks

```svelte
<!-- Dialog.svelte -->
<script>
  let { onconfirm, oncancel, onclose } = $props();
</script>

<dialog>
  <button onclick={() => onconfirm?.()}>Confirm</button>
  <button onclick={() => oncancel?.()}>Cancel</button>
  <button onclick={() => onclose?.()}>X</button>
</dialog>

<!-- Parent.svelte -->
<Dialog
  onconfirm={() => save()}
  oncancel={() => reset()}
  onclose={() => visible = false}
/>
```

### Typed Callbacks with TypeScript

```svelte
<!-- SearchInput.svelte -->
<script lang="ts">
  interface Props {
    value?: string;
    onsearch?: (query: string) => void;
    onchange?: (value: string) => void;
    onclear?: () => void;
  }

  let { value = '', onsearch, onchange, onclear }: Props = $props();

  function handleInput(e: Event) {
    const newValue = (e.target as HTMLInputElement).value;
    value = newValue;
    onchange?.(newValue);
  }

  function handleSubmit() {
    onsearch?.(value);
  }

  function handleClear() {
    value = '';
    onclear?.();
  }
</script>

<form onsubmit={(e) => { e.preventDefault(); handleSubmit(); }}>
  <input {value} oninput={handleInput} />
  <button type="button" onclick={handleClear}>Clear</button>
  <button type="submit">Search</button>
</form>
```

### Callback with Return Value

```svelte
<!-- ConfirmDialog.svelte -->
<script lang="ts">
  interface Props {
    message: string;
    onconfirm: () => boolean | Promise<boolean>; // Can return whether to close
  }

  let { message, onconfirm }: Props = $props();
  let loading = $state(false);

  async function handleConfirm() {
    loading = true;
    const shouldClose = await onconfirm();
    loading = false;
    // Parent controls whether dialog closes
  }
</script>
```

### Event Data Pattern

```svelte
<!-- DataTable.svelte -->
<script lang="ts">
  interface Row {
    id: number;
    name: string;
  }

  interface Props {
    rows: Row[];
    onrowclick?: (row: Row, index: number) => void;
    onrowselect?: (selectedRows: Row[]) => void;
    onsort?: (column: string, direction: 'asc' | 'desc') => void;
  }

  let { rows, onrowclick, onrowselect, onsort }: Props = $props();
</script>

<table>
  {#each rows as row, index}
    <tr onclick={() => onrowclick?.(row, index)}>
      <td>{row.name}</td>
    </tr>
  {/each}
</table>
```

### Forwarding Native Events

```svelte
<!-- CustomButton.svelte -->
<script lang="ts">
  import type { MouseEventHandler } from 'svelte/elements';

  interface Props {
    onclick?: MouseEventHandler<HTMLButtonElement>;
    children: import('svelte').Snippet;
  }

  let { onclick, children, ...rest }: Props = $props();
</script>

<button {onclick} {...rest}>
  {@render children()}
</button>
```

### Migration Helper

When migrating many dispatch calls:

```svelte
<script lang="ts">
  // Create a typed callback pattern
  interface Events {
    save: { data: FormData };
    cancel: void;
    error: { message: string };
  }

  interface Props {
    onsave?: (detail: Events['save']) => void;
    oncancel?: () => void;
    onerror?: (detail: Events['error']) => void;
  }

  let { onsave, oncancel, onerror }: Props = $props();

  // Use like dispatch was used
  function save(data: FormData) {
    onsave?.({ data });
  }

  function cancel() {
    oncancel?.();
  }

  function error(message: string) {
    onerror?.({ message });
  }
</script>
```

## Reference

- [Svelte 5 Component Events](https://svelte.dev/docs/svelte/legacy-createEventDispatcher)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
