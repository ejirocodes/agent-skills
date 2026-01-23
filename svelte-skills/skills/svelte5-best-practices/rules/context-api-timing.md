---
title: Context API Timing and Usage
impact: HIGH
impactDescription: correctly use context API in Svelte 5
type: capability
tags: setContext, getContext, timing, initialization, lifecycle
---

# Context API Timing and Usage

**Impact: HIGH** - correctly use context API in Svelte 5

Context functions must be called synchronously during component initialization, not in effects or event handlers.

## Symptoms

- "getContext must be called during component initialization" error
- "setContext must be called during component initialization" error
- Context values undefined
- Context not updating between components

## Root Cause

Svelte's context API (`setContext`, `getContext`) relies on component initialization timing. Calling them asynchronously or in callbacks breaks the association.

## Fix

### Correct Context Usage

**Incorrect (in $effect or callback):**

```svelte
<script>
  import { setContext, getContext } from 'svelte';

  $effect(() => {
    setContext('theme', 'dark'); // ERROR: Not during initialization
  });

  function handleClick() {
    const theme = getContext('theme'); // ERROR: Not during initialization
  }
</script>
```

**Correct (at top level):**

```svelte
<script>
  import { setContext, getContext } from 'svelte';

  // These run during initialization
  setContext('theme', 'dark');
  const theme = getContext('theme');
</script>
```

### Reactive Context Values

Make context reactive by passing `$state` objects:

```svelte
<!-- Provider.svelte -->
<script>
  import { setContext } from 'svelte';

  // Create reactive state
  let theme = $state('light');

  // Pass the reactive object
  setContext('theme', {
    get current() { return theme; },
    toggle() { theme = theme === 'light' ? 'dark' : 'light'; }
  });

  let { children } = $props();
</script>

{@render children()}

<!-- Consumer.svelte -->
<script>
  import { getContext } from 'svelte';

  const themeContext = getContext('theme');
  // themeContext.current is reactive!
</script>

<p>Theme: {themeContext.current}</p>
<button onclick={themeContext.toggle}>Toggle</button>
```

### Type-Safe Context Pattern

```svelte
<!-- context.ts -->
<script context="module" lang="ts">
  import { setContext, getContext } from 'svelte';

  const THEME_KEY = Symbol('theme');

  interface ThemeContext {
    current: 'light' | 'dark';
    toggle: () => void;
  }

  export function setThemeContext(context: ThemeContext) {
    setContext(THEME_KEY, context);
  }

  export function getThemeContext(): ThemeContext {
    const context = getContext<ThemeContext>(THEME_KEY);
    if (!context) {
      throw new Error('Theme context not found. Did you wrap with ThemeProvider?');
    }
    return context;
  }
</script>

<!-- ThemeProvider.svelte -->
<script lang="ts">
  import { setThemeContext } from './context';

  let theme = $state<'light' | 'dark'>('light');

  setThemeContext({
    get current() { return theme; },
    toggle() { theme = theme === 'light' ? 'dark' : 'light'; }
  });

  let { children } = $props();
</script>

{@render children()}

<!-- ThemedButton.svelte -->
<script lang="ts">
  import { getThemeContext } from './context';

  const theme = getThemeContext();
</script>

<button class={theme.current}>
  {@render children()}
</button>
```

### Context with Stores (Alternative)

For truly global state, consider `$state` in a module:

```ts
// store.svelte.ts
export const theme = $state({ current: 'light' as 'light' | 'dark' });

export function toggleTheme() {
  theme.current = theme.current === 'light' ? 'dark' : 'light';
}
```

```svelte
<script>
  import { theme, toggleTheme } from './store.svelte';
</script>

<p>Theme: {theme.current}</p>
<button onclick={toggleTheme}>Toggle</button>
```

### Avoid Generic Keys

**Incorrect (collision-prone):**

```svelte
<script>
  setContext('user', userData); // Could conflict with other libraries
</script>
```

**Correct (use unique keys):**

```svelte
<script>
  // Option 1: Symbol
  const USER_KEY = Symbol('user');
  setContext(USER_KEY, userData);

  // Option 2: Unique string
  setContext('myapp:user', userData);

  // Option 3: Object key
  const userKey = {};
  setContext(userKey, userData);
</script>
```

### hasContext Check

```svelte
<script>
  import { hasContext, getContext } from 'svelte';

  // Check if context exists before using
  const hasTheme = hasContext('theme');
  const theme = hasTheme ? getContext('theme') : { current: 'light' };
</script>
```

### Context Boundaries

Context flows down the component tree:

```svelte
<!-- App.svelte -->
<script>
  import { setContext } from 'svelte';
  setContext('level', 0);
</script>

<Parent />

<!-- Parent.svelte -->
<script>
  import { getContext, setContext } from 'svelte';
  const level = getContext('level'); // 0
  setContext('level', level + 1); // Override for children
</script>

<Child /> <!-- Will see level = 1 -->

<!-- Child.svelte -->
<script>
  import { getContext } from 'svelte';
  const level = getContext('level'); // 1
</script>
```

## Reference

- [Svelte Context API](https://svelte.dev/docs/svelte/context)
- [Svelte setContext](https://svelte.dev/docs/svelte/setContext)
- [Svelte getContext](https://svelte.dev/docs/svelte/getContext)
