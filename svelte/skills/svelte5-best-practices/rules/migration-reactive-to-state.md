---
title: Migrating from $: Reactive Statements
impact: HIGH
impactDescription: migrate Svelte 4 reactive statements to Svelte 5 runes
type: capability
tags: migration, reactive statement, $:, $derived, $effect
---

# Migrating from $: Reactive Statements

**Impact: HIGH** - migrate Svelte 4 reactive statements to Svelte 5 runes

Svelte 5 replaces `$:` reactive statements with `$derived` (for values) and `$effect` (for side effects).

## Symptoms

- `$:` produces deprecation warnings
- Unclear which rune to use for different patterns
- Migration doesn't preserve reactivity

## Root Cause

Svelte 5 separates derived values from side effects for clearer code and better performance.

## Migration Rules

### Rule 1: Computed Values → $derived

**Svelte 4:**

```svelte
<script>
  let count = 0;
  $: doubled = count * 2;
  $: quadrupled = doubled * 2;
</script>
```

**Svelte 5:**

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  let quadrupled = $derived(doubled * 2);
</script>
```

### Rule 2: Complex Computed Values → $derived.by

**Svelte 4:**

```svelte
<script>
  let items = [];
  let filter = 'all';

  $: filteredItems = {
    if (filter === 'all') return items;
    if (filter === 'active') return items.filter(i => !i.done);
    return items.filter(i => i.done);
  };
</script>
```

**Svelte 5:**

```svelte
<script>
  let items = $state([]);
  let filter = $state('all');

  let filteredItems = $derived.by(() => {
    if (filter === 'all') return items;
    if (filter === 'active') return items.filter(i => !i.done);
    return items.filter(i => i.done);
  });
</script>
```

### Rule 3: Side Effects → $effect

**Svelte 4:**

```svelte
<script>
  let count = 0;
  $: console.log('Count changed:', count);
  $: document.title = `Count: ${count}`;
</script>
```

**Svelte 5:**

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    console.log('Count changed:', count);
  });

  $effect(() => {
    document.title = `Count: ${count}`;
  });
</script>
```

### Rule 4: Conditional Side Effects → $effect with if

**Svelte 4:**

```svelte
<script>
  let value;
  $: if (value > 100) {
    alert('Value too high!');
  }
</script>
```

**Svelte 5:**

```svelte
<script>
  let value = $state(0);

  $effect(() => {
    if (value > 100) {
      alert('Value too high!');
    }
  });
</script>
```

### Rule 5: Reactive Declarations → $state

**Svelte 4:**

```svelte
<script>
  let count = 0; // Implicitly reactive in components
</script>
```

**Svelte 5:**

```svelte
<script>
  let count = $state(0); // Explicitly reactive
</script>
```

### Rule 6: Props → $props

**Svelte 4:**

```svelte
<script>
  export let name;
  export let count = 0;
  $: greeting = `Hello, ${name}!`;
</script>
```

**Svelte 5:**

```svelte
<script>
  let { name, count = 0 } = $props();
  let greeting = $derived(`Hello, ${name}!`);
</script>
```

### Rule 7: Multiple Dependencies → $derived.by

**Svelte 4:**

```svelte
<script>
  let firstName = '';
  let lastName = '';
  let age = 0;

  $: fullInfo = `${firstName} ${lastName}, age ${age}`;
</script>
```

**Svelte 5:**

```svelte
<script>
  let firstName = $state('');
  let lastName = $state('');
  let age = $state(0);

  let fullInfo = $derived(`${firstName} ${lastName}, age ${age}`);
</script>
```

### Rule 8: Effect with Cleanup

**Svelte 4:**

```svelte
<script>
  import { onDestroy } from 'svelte';

  let count = 0;
  let interval;

  $: {
    clearInterval(interval);
    interval = setInterval(() => console.log(count), 1000);
  }

  onDestroy(() => clearInterval(interval));
</script>
```

**Svelte 5:**

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    const interval = setInterval(() => console.log(count), 1000);
    return () => clearInterval(interval); // Cleanup built-in
  });
</script>
```

## Migration Cheat Sheet

| Svelte 4 Pattern | Svelte 5 Replacement |
|------------------|----------------------|
| `let x = 0` (in component) | `let x = $state(0)` |
| `export let prop` | `let { prop } = $props()` |
| `$: derived = expr` | `let derived = $derived(expr)` |
| `$: { complex }` (value) | `let x = $derived.by(() => { ... })` |
| `$: console.log(x)` | `$effect(() => console.log(x))` |
| `$: if (x) { ... }` | `$effect(() => { if (x) { ... } })` |
| `$: document.title = x` | `$effect(() => { document.title = x })` |

## Automated Migration

Use the Svelte CLI to automate migration:

```bash
npx sv migrate svelte-5
```

Or in VS Code, use the "Migrate Component to Svelte 5 Syntax" command.

## Reference

- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
- [Svelte 5 $derived](https://svelte.dev/docs/svelte/$derived)
- [Svelte 5 $effect](https://svelte.dev/docs/svelte/$effect)
