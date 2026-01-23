---
title: Runes in .svelte.js/.svelte.ts Files
impact: MEDIUM
impactDescription: use runes for shared state outside components
type: efficiency
tags: svelte.js, svelte.ts, shared state, universal, module
---

# Runes in .svelte.js/.svelte.ts Files

**Impact: MEDIUM** - use runes for shared state outside components

Svelte 5 allows runes (`$state`, `$derived`, `$effect`) to work outside `.svelte` components when using `.svelte.js` or `.svelte.ts` file extensions.

## Symptoms

- Creating Svelte stores when runes would be simpler
- "Cannot use runes outside of Svelte components" error
- Unclear how to share reactive state across components

## Root Cause

Runes require the Svelte compiler. Files with `.svelte.js` or `.svelte.ts` extensions are processed by the compiler, enabling rune usage.

## Fix

### Shared Counter State

**Incorrect (using stores):**

```ts
// store.ts
import { writable, derived } from 'svelte/store';

export const count = writable(0);
export const doubled = derived(count, $count => $count * 2);
```

**Correct (using runes in .svelte.ts):**

```ts
// counter.svelte.ts
export const counter = $state({ count: 0 });

export function increment() {
  counter.count++;
}

export function decrement() {
  counter.count--;
}

export function reset() {
  counter.count = 0;
}
```

```svelte
<!-- Counter.svelte -->
<script>
  import { counter, increment, decrement, reset } from './counter.svelte';
</script>

<p>Count: {counter.count}</p>
<button onclick={increment}>+</button>
<button onclick={decrement}>-</button>
<button onclick={reset}>Reset</button>
```

### Object State with Getters

Use getters for derived values in modules:

```ts
// user.svelte.ts
interface User {
  firstName: string;
  lastName: string;
  email: string;
}

const state = $state<User>({
  firstName: '',
  lastName: '',
  email: ''
});

export const user = {
  get firstName() { return state.firstName; },
  set firstName(v: string) { state.firstName = v; },

  get lastName() { return state.lastName; },
  set lastName(v: string) { state.lastName = v; },

  get email() { return state.email; },
  set email(v: string) { state.email = v; },

  // Derived value as getter
  get fullName() {
    return `${state.firstName} ${state.lastName}`;
  },

  get isValid() {
    return state.firstName && state.lastName && state.email.includes('@');
  }
};
```

### Reactive Class Pattern

```ts
// todo.svelte.ts
interface Todo {
  id: number;
  text: string;
  done: boolean;
}

class TodoStore {
  items = $state<Todo[]>([]);
  filter = $state<'all' | 'active' | 'completed'>('all');

  get filtered() {
    switch (this.filter) {
      case 'active': return this.items.filter(t => !t.done);
      case 'completed': return this.items.filter(t => t.done);
      default: return this.items;
    }
  }

  get remaining() {
    return this.items.filter(t => !t.done).length;
  }

  add(text: string) {
    this.items.push({
      id: Date.now(),
      text,
      done: false
    });
  }

  toggle(id: number) {
    const item = this.items.find(t => t.id === id);
    if (item) item.done = !item.done;
  }

  remove(id: number) {
    const index = this.items.findIndex(t => t.id === id);
    if (index !== -1) this.items.splice(index, 1);
  }

  clearCompleted() {
    this.items = this.items.filter(t => !t.done);
  }
}

export const todos = new TodoStore();
```

### Async State Pattern

```ts
// api.svelte.ts
interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

export function createAsyncState<T>(fetcher: () => Promise<T>) {
  const state = $state<FetchState<T>>({
    data: null,
    loading: false,
    error: null
  });

  async function fetch() {
    state.loading = true;
    state.error = null;

    try {
      state.data = await fetcher();
    } catch (e) {
      state.error = e as Error;
    } finally {
      state.loading = false;
    }
  }

  return {
    get data() { return state.data; },
    get loading() { return state.loading; },
    get error() { return state.error; },
    fetch,
    reset() {
      state.data = null;
      state.error = null;
      state.loading = false;
    }
  };
}

// Usage
export const users = createAsyncState(() =>
  fetch('/api/users').then(r => r.json())
);
```

### Theme State Example

```ts
// theme.svelte.ts
type Theme = 'light' | 'dark' | 'system';

const state = $state({ current: 'system' as Theme });

export const theme = {
  get current() { return state.current; },

  set(value: Theme) {
    state.current = value;
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem('theme', value);
    }
  },

  toggle() {
    this.set(state.current === 'light' ? 'dark' : 'light');
  },

  init() {
    if (typeof localStorage !== 'undefined') {
      const saved = localStorage.getItem('theme') as Theme;
      if (saved) state.current = saved;
    }
  }
};
```

## Important Notes

1. **File extension matters**: Must use `.svelte.js` or `.svelte.ts`
2. **No $derived in module scope**: Use getters for derived values
3. **SSR caution**: Avoid initializing browser-only state at module level

## Reference

- [Svelte 5 Universal Reactivity](https://svelte.dev/docs/svelte/$state#Universal-reactivity)
- [SvelteKit State Management](https://svelte.dev/docs/kit/state-management)
