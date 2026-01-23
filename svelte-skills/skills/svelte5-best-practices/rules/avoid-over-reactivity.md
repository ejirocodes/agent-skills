---
title: Avoiding Over-Reactivity Patterns
impact: MEDIUM
impactDescription: prevent performance issues from excessive reactivity
type: efficiency
tags: performance, over-reactivity, $effect, $derived, optimization
---

# Avoiding Over-Reactivity Patterns

**Impact: MEDIUM** - prevent performance issues from excessive reactivity

Common mistakes with runes can cause unnecessary re-renders, infinite loops, or degraded performance.

## Symptoms

- Performance issues with reactive updates
- Infinite loops in effects
- Unnecessary component re-renders
- Memory leaks from accumulated effects

## Anti-Patterns and Fixes

### Anti-Pattern 1: Using $effect to Set Derived Values

**Incorrect:**

```svelte
<script>
  let count = $state(0);
  let doubled = $state(0);

  // WRONG: Using effect to derive values
  $effect(() => {
    doubled = count * 2;
  });
</script>
```

**Correct:**

```svelte
<script>
  let count = $state(0);

  // Use $derived for computed values
  let doubled = $derived(count * 2);
</script>
```

### Anti-Pattern 2: Circular Dependencies

**Incorrect:**

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // WRONG: Circular - effect changes its own dependency
  $effect(() => {
    if (a > 10) {
      b = 0;
    }
    if (b > 10) {
      a = 0; // This triggers the effect again!
    }
  });
</script>
```

**Correct:**

```svelte
<script>
  let a = $state(1);
  let b = $state(2);

  // Separate effects with clear responsibilities
  $effect(() => {
    if (a > 10) {
      b = 0;
    }
  });

  // Or use event handlers instead
  function incrementA() {
    a++;
    if (a > 10) b = 0;
  }
</script>
```

### Anti-Pattern 3: Effect Inside Loops

**Incorrect:**

```svelte
<script>
  let items = $state([1, 2, 3]);

  // WRONG: Creates new effect on every render
  {#each items as item}
    {$effect(() => console.log(item))}
  {/each}
</script>
```

**Correct:**

```svelte
<script>
  let items = $state([1, 2, 3]);

  // Single effect that watches the array
  $effect(() => {
    items.forEach(item => console.log(item));
  });
</script>
```

### Anti-Pattern 4: Not Using untrack

**Incorrect:**

```svelte
<script>
  let count = $state(0);
  let log = $state([]);

  // WRONG: Effect re-runs when log changes
  $effect(() => {
    log.push(`Count is ${count}`); // log change triggers re-run
  });
</script>
```

**Correct:**

```svelte
<script>
  import { untrack } from 'svelte';

  let count = $state(0);
  let log = $state([]);

  $effect(() => {
    // Only react to count, not log
    untrack(() => {
      log.push(`Count is ${count}`);
    });
  });
</script>
```

### Anti-Pattern 5: Heavy Computations in $derived

**Incorrect:**

```svelte
<script>
  let items = $state([/* thousands of items */]);
  let filter = $state('');

  // WRONG: Expensive computation runs on every filter change
  let filtered = $derived(
    items.filter(item =>
      JSON.stringify(item).toLowerCase().includes(filter.toLowerCase())
    )
  );
</script>
```

**Correct:**

```svelte
<script>
  let items = $state([/* thousands of items */]);
  let filter = $state('');
  let debouncedFilter = $state('');

  // Debounce the filter
  $effect(() => {
    const timeout = setTimeout(() => {
      debouncedFilter = filter;
    }, 300);
    return () => clearTimeout(timeout);
  });

  // Use debounced value for expensive computation
  let filtered = $derived(
    items.filter(item =>
      item.name.toLowerCase().includes(debouncedFilter.toLowerCase())
    )
  );
</script>
```

### Anti-Pattern 6: Creating Objects in $derived

**Incorrect:**

```svelte
<script>
  let x = $state(0);
  let y = $state(0);

  // WRONG: Creates new object reference every time
  let position = $derived({ x, y });

  // This effect runs even when x and y haven't changed
  $effect(() => {
    console.log(position); // New object every read
  });
</script>
```

**Correct:**

```svelte
<script>
  let x = $state(0);
  let y = $state(0);

  // State object that mutates
  let position = $state({ x: 0, y: 0 });

  $effect(() => {
    position.x = x;
    position.y = y;
  });
</script>
```

### Anti-Pattern 7: Effect for DOM Manipulation

**Incorrect:**

```svelte
<script>
  let visible = $state(false);
  let element;

  // WRONG: Manual DOM manipulation
  $effect(() => {
    if (element) {
      element.style.display = visible ? 'block' : 'none';
    }
  });
</script>

<div bind:this={element}>Content</div>
```

**Correct:**

```svelte
<script>
  let visible = $state(false);
</script>

<!-- Use Svelte's built-in reactivity -->
{#if visible}
  <div>Content</div>
{/if}

<!-- Or CSS class -->
<div class:hidden={!visible}>Content</div>
```

### Anti-Pattern 8: Too Many Fine-Grained States

**Incorrect:**

```svelte
<script>
  // WRONG: Excessive granularity
  let firstName = $state('');
  let lastName = $state('');
  let email = $state('');
  let phone = $state('');
  let address = $state('');
  let city = $state('');
  let zip = $state('');
</script>
```

**Correct:**

```svelte
<script>
  // Better: Group related state
  let user = $state({
    firstName: '',
    lastName: '',
    email: '',
    phone: ''
  });

  let address = $state({
    street: '',
    city: '',
    zip: ''
  });
</script>
```

## Performance Tips

1. **Use $derived over $effect** for computed values
2. **Debounce** expensive reactive computations
3. **Use untrack** to prevent unnecessary dependencies
4. **Group related state** into objects
5. **Avoid creating objects** in $derived expressions
6. **Let Svelte handle DOM** updates

## Reference

- [Svelte 5 $effect Documentation](https://svelte.dev/docs/svelte/$effect)
- [Svelte 5 $derived Documentation](https://svelte.dev/docs/svelte/$derived)
