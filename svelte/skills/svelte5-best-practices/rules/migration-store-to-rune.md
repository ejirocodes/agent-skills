---
title: Migrating from Stores to Runes
impact: HIGH
impactDescription: migrate Svelte stores to runes for state management
type: capability
tags: migration, store, writable, readable, $store, runes
---

# Migrating from Stores to Runes

**Impact: HIGH** - migrate Svelte stores to runes for state management

Svelte 5 runes can replace most store use cases with simpler, more direct reactive state.

## Symptoms

- Using stores unnecessarily in Svelte 5
- Mixing store and rune patterns
- Unclear when to use stores vs runes

## Root Cause

Svelte 5's universal reactivity (`$state` in `.svelte.js/.svelte.ts` files) eliminates most needs for the store API.

## Migration Patterns

### Pattern 1: Local Component State

**Svelte 4 (store):**

```svelte
<script>
  import { writable } from 'svelte/store';

  const count = writable(0);

  function increment() {
    count.update(n => n + 1);
  }
</script>

<button on:click={increment}>
  Count: {$count}
</button>
```

**Svelte 5 (rune):**

```svelte
<script>
  let count = $state(0);

  function increment() {
    count++;
  }
</script>

<button onclick={increment}>
  Count: {count}
</button>
```

### Pattern 2: Shared State Across Components

**Svelte 4 (store in module):**

```ts
// stores.ts
import { writable } from 'svelte/store';

export const user = writable(null);
export const theme = writable('light');
```

```svelte
<!-- Component.svelte -->
<script>
  import { user, theme } from './stores';
</script>

<p>Theme: {$theme}</p>
```

**Svelte 5 (rune in .svelte.ts):**

```ts
// state.svelte.ts
export const user = $state<User | null>(null);
export const theme = $state({ current: 'light' as 'light' | 'dark' });

export function setTheme(newTheme: 'light' | 'dark') {
  theme.current = newTheme;
}
```

```svelte
<!-- Component.svelte -->
<script>
  import { theme, setTheme } from './state.svelte';
</script>

<p>Theme: {theme.current}</p>
<button onclick={() => setTheme('dark')}>Dark Mode</button>
```

### Pattern 3: Derived Store → $derived

**Svelte 4:**

```ts
// stores.ts
import { writable, derived } from 'svelte/store';

export const items = writable([]);
export const completedCount = derived(items, $items =>
  $items.filter(i => i.done).length
);
```

**Svelte 5:**

```ts
// state.svelte.ts
export const items = $state<Item[]>([]);

// Derived must be in component or use getter
export function getCompletedCount() {
  return items.filter(i => i.done).length;
}
```

```svelte
<script>
  import { items } from './state.svelte';

  // Create derived in component
  let completedCount = $derived(items.filter(i => i.done).length);
</script>
```

### Pattern 4: Custom Store → Reactive Class

**Svelte 4:**

```ts
// stores.ts
import { writable } from 'svelte/store';

function createCounter() {
  const { subscribe, set, update } = writable(0);

  return {
    subscribe,
    increment: () => update(n => n + 1),
    decrement: () => update(n => n - 1),
    reset: () => set(0)
  };
}

export const counter = createCounter();
```

**Svelte 5:**

```ts
// counter.svelte.ts
class Counter {
  value = $state(0);

  increment() {
    this.value++;
  }

  decrement() {
    this.value--;
  }

  reset() {
    this.value = 0;
  }
}

export const counter = new Counter();
```

```svelte
<script>
  import { counter } from './counter.svelte';
</script>

<p>{counter.value}</p>
<button onclick={() => counter.increment()}>+</button>
```

### Pattern 5: Readable Store → $state + getter

**Svelte 4:**

```ts
import { readable } from 'svelte/store';

export const time = readable(new Date(), set => {
  const interval = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(interval);
});
```

**Svelte 5:**

```ts
// time.svelte.ts
let currentTime = $state(new Date());

if (typeof window !== 'undefined') {
  setInterval(() => {
    currentTime = new Date();
  }, 1000);
}

export const time = {
  get current() {
    return currentTime;
  }
};
```

### Pattern 6: Store with Async

**Svelte 4:**

```ts
import { writable } from 'svelte/store';

export const data = writable(null);
export const loading = writable(false);
export const error = writable(null);

export async function fetchData() {
  loading.set(true);
  try {
    const res = await fetch('/api/data');
    data.set(await res.json());
  } catch (e) {
    error.set(e);
  } finally {
    loading.set(false);
  }
}
```

**Svelte 5:**

```ts
// api.svelte.ts
export const state = $state({
  data: null as Data | null,
  loading: false,
  error: null as Error | null
});

export async function fetchData() {
  state.loading = true;
  state.error = null;

  try {
    const res = await fetch('/api/data');
    state.data = await res.json();
  } catch (e) {
    state.error = e as Error;
  } finally {
    state.loading = false;
  }
}
```

## When to Still Use Stores

Stores are still useful for:

1. **Library interop** - Libraries that expect Svelte stores
2. **Legacy code** - Gradual migration
3. **Observable patterns** - When you need the subscribe API
4. **Server-side rendering** - Stores have built-in SSR handling

## Migration Cheat Sheet

| Store Pattern | Rune Replacement |
|---------------|------------------|
| `writable(value)` | `$state(value)` |
| `$store` (auto-subscribe) | Direct access to `$state` value |
| `store.set(x)` | `state = x` |
| `store.update(fn)` | Direct mutation or reassignment |
| `derived(store, fn)` | `$derived(fn())` in component |
| `readable(value, start)` | `$state` + `$effect` for setup |

## Reference

- [Svelte 5 Universal Reactivity](https://svelte.dev/docs/svelte/$state#Universal-reactivity)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
- [SvelteKit State Management](https://svelte.dev/docs/kit/state-management)
