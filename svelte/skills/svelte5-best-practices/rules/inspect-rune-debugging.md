---
title: Debugging with $inspect Rune
impact: MEDIUM
impactDescription: debug reactive state changes in Svelte 5
type: capability
tags: $inspect, debugging, console.log, reactive, development
---

# Debugging with $inspect Rune

**Impact: MEDIUM** - debug reactive state changes in Svelte 5

The `$inspect` rune provides reactive debugging, logging values when they change.

## Symptoms

- `console.log` only shows initial value, not updates
- Hard to track when reactive values change
- Debugging reactive state is cumbersome

## Root Cause

Regular `console.log` doesn't subscribe to reactive state. `$inspect` automatically re-runs when dependencies change.

## Fix

### Basic Usage

**Incorrect (doesn't track changes):**

```svelte
<script>
  let count = $state(0);
  console.log('Count:', count); // Only logs once at initialization
</script>
```

**Correct (tracks all changes):**

```svelte
<script>
  let count = $state(0);
  $inspect(count); // Logs every time count changes
</script>
```

### Multiple Values

```svelte
<script>
  let name = $state('Alice');
  let age = $state(30);

  // Inspect multiple values
  $inspect(name, age);
  // Output: init ["Alice", 30]
  // On change: update ["Bob", 30]
</script>
```

### Custom Logging with $inspect.with

```svelte
<script>
  let user = $state({ name: 'Alice', age: 30 });

  $inspect.with((type, ...values) => {
    if (type === 'init') {
      console.log('Initial value:', values);
    } else {
      console.log('Updated to:', values);
      console.trace(); // Show call stack
    }
  }, user);
</script>
```

### Debugging Derived Values

```svelte
<script>
  let items = $state([1, 2, 3, 4, 5]);
  let filter = $state('even');

  let filtered = $derived(items.filter(n =>
    filter === 'even' ? n % 2 === 0 : n % 2 !== 0
  ));

  // Inspect the derived value
  $inspect(filtered);
</script>
```

### Conditional Debugging

```svelte
<script>
  let count = $state(0);

  // Only inspect in development
  if (import.meta.env.DEV) {
    $inspect(count);
  }
</script>
```

### Debugging with Breakpoints

```svelte
<script>
  let data = $state({ x: 0, y: 0 });

  $inspect.with((type, value) => {
    if (type === 'update' && value.x > 100) {
      debugger; // Break when x exceeds 100
    }
  }, data);
</script>
```

### Inspecting Props

```svelte
<script>
  let { items, selected } = $props();

  // Debug props as they change
  $inspect('Props:', { items, selected });
</script>
```

### Inspecting Effect Dependencies

```svelte
<script>
  let count = $state(0);
  let multiplier = $state(2);

  // See what triggers the effect
  $effect(() => {
    $inspect('Effect deps:', count, multiplier);
    // ... effect logic
  });
</script>
```

### Object Deep Inspection

```svelte
<script>
  let state = $state({
    user: { name: 'Alice', preferences: { theme: 'dark' } },
    items: []
  });

  // $inspect tracks deep changes
  $inspect(state);

  // This change will be logged:
  state.user.preferences.theme = 'light';
</script>
```

### Formatted Output

```svelte
<script>
  let metrics = $state({ fps: 60, memory: 100 });

  $inspect.with((type, value) => {
    console.log(`[${type.toUpperCase()}] FPS: ${value.fps}, Memory: ${value.memory}MB`);
  }, metrics);
</script>
```

## Production Warning

`$inspect` is development-only and will be stripped in production builds. No need to remove it manually.

```svelte
<script>
  let count = $state(0);

  // Safe to leave in code - removed in production
  $inspect(count);
</script>
```

## vs console.log in $effect

**$effect approach (works but verbose):**

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    console.log('Count:', count);
  });
</script>
```

**$inspect approach (cleaner):**

```svelte
<script>
  let count = $state(0);
  $inspect('Count:', count);
</script>
```

## Reference

- [Svelte 5 $inspect Documentation](https://svelte.dev/docs/svelte/$inspect)
