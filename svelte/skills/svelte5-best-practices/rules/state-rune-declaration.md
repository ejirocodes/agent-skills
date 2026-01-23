---
title: Declaring Reactive State with $state
impact: HIGH
impactDescription: enables reactive state management in Svelte 5 components
type: capability
tags: $state, reactive, declaration, runes, svelte5, let
---

# Declaring Reactive State with $state

**Impact: HIGH** - enables reactive state management in Svelte 5 components

In Svelte 5, plain `let` declarations are no longer automatically reactive. You must use the `$state()` rune to create reactive state.

## Symptoms

- UI doesn't update when variable changes
- Counter or form values appear stuck
- Coming from Svelte 4, `let` declarations don't trigger reactivity

## Root Cause

Svelte 5 introduced runes as explicit reactivity primitives. Unlike Svelte 4 where `let` in components was automatically reactive, Svelte 5 requires explicit `$state()` calls.

## Fix

**Incorrect (Svelte 4 pattern, won't be reactive in Svelte 5):**

```svelte
<script>
  let count = 0; // NOT reactive in Svelte 5!
</script>

<button onclick={() => count++}>
  Clicks: {count}
</button>
```

**Correct (Svelte 5 pattern):**

```svelte
<script>
  let count = $state(0);
</script>

<button onclick={() => count++}>
  Clicks: {count}
</button>
```

## Object and Array State

Objects and arrays are deeply reactive by default:

```svelte
<script>
  let user = $state({ name: 'Alice', age: 30 });
  let items = $state(['apple', 'banana']);
</script>

<button onclick={() => user.age++}>
  Age: {user.age}
</button>

<button onclick={() => items.push('cherry')}>
  Items: {items.length}
</button>
```

## Class State

Use `$state()` in class fields:

```svelte
<script>
  class Counter {
    count = $state(0);

    increment() {
      this.count++;
    }
  }

  const counter = new Counter();
</script>

<button onclick={() => counter.increment()}>
  {counter.count}
</button>
```

## Raw State (Opt-out of Deep Reactivity)

Use `$state.raw()` when you don't need deep reactivity:

```svelte
<script>
  // Only reassignment triggers updates, not mutations
  let items = $state.raw([1, 2, 3]);

  // This WON'T trigger an update:
  items.push(4);

  // This WILL trigger an update:
  items = [...items, 4];
</script>
```

## Reference

- [Svelte 5 $state Documentation](https://svelte.dev/docs/svelte/$state)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
