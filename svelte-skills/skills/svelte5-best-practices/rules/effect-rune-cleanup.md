---
title: Managing Side Effects with $effect
impact: HIGH
impactDescription: handle side effects with proper cleanup in Svelte 5
type: capability
tags: $effect, cleanup, side-effect, lifecycle, onMount, subscription
---

# Managing Side Effects with $effect

**Impact: HIGH** - handle side effects with proper cleanup in Svelte 5

The `$effect` rune runs code when component mounts and when dependencies change. Effects require cleanup for subscriptions and event listeners.

## Symptoms

- Memory leaks from uncleaned subscriptions
- Event listeners accumulating on DOM elements
- Effects running at unexpected times
- Using `$effect` when `$derived` would be more appropriate

## Root Cause

`$effect` runs after the DOM updates and re-runs when reactive dependencies change. Without proper cleanup, subscriptions and listeners accumulate.

## Fix

### Basic Effect with Cleanup

**Incorrect (no cleanup):**

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    const interval = setInterval(() => count++, 1000);
    // Memory leak! Interval never cleared
  });
</script>
```

**Correct (with cleanup):**

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    const interval = setInterval(() => count++, 1000);

    return () => {
      clearInterval(interval); // Cleanup when effect re-runs or component unmounts
    };
  });
</script>
```

### DOM Event Listeners

```svelte
<script>
  let mouseX = $state(0);
  let mouseY = $state(0);

  $effect(() => {
    function handleMouseMove(event) {
      mouseX = event.clientX;
      mouseY = event.clientY;
    }

    window.addEventListener('mousemove', handleMouseMove);

    return () => {
      window.removeEventListener('mousemove', handleMouseMove);
    };
  });
</script>

<p>Mouse position: {mouseX}, {mouseY}</p>
```

### External Subscriptions

```svelte
<script>
  let { userId } = $props();
  let userData = $state(null);

  $effect(() => {
    // Re-runs when userId changes
    const unsubscribe = database.subscribe(`users/${userId}`, (data) => {
      userData = data;
    });

    return () => {
      unsubscribe(); // Clean up previous subscription
    };
  });
</script>
```

## $effect.pre for Pre-DOM Updates

Use `$effect.pre` when you need to run before DOM updates:

```svelte
<script>
  let div;
  let messages = $state([]);

  $effect.pre(() => {
    // Runs BEFORE DOM updates
    // Useful for scroll position preservation
    if (div) {
      const shouldScroll = div.scrollTop + div.clientHeight >= div.scrollHeight - 20;
      if (shouldScroll) {
        // Will scroll after messages update
        tick().then(() => {
          div.scrollTop = div.scrollHeight;
        });
      }
    }
  });
</script>
```

## When NOT to Use $effect

**Don't use $effect to derive values:**

```svelte
<script>
  let count = $state(0);

  // WRONG: Using effect to set derived value
  let doubled = $state(0);
  $effect(() => {
    doubled = count * 2;
  });

  // CORRECT: Use $derived instead
  let doubledCorrect = $derived(count * 2);
</script>
```

**Don't use $effect for event handlers:**

```svelte
<!-- WRONG -->
<script>
  let button;
  $effect(() => {
    button?.addEventListener('click', handleClick);
    return () => button?.removeEventListener('click', handleClick);
  });
</script>
<button bind:this={button}>Click</button>

<!-- CORRECT -->
<button onclick={handleClick}>Click</button>
```

## Untracked Dependencies

Use `untrack` to read state without creating a dependency:

```svelte
<script>
  import { untrack } from 'svelte';

  let count = $state(0);
  let logCount = $state(0);

  $effect(() => {
    // Only runs when count changes, not logCount
    console.log(count, untrack(() => logCount));
  });
</script>
```

## Reference

- [Svelte 5 $effect Documentation](https://svelte.dev/docs/svelte/$effect)
- [Svelte 5 Lifecycle](https://svelte.dev/docs/svelte/lifecycle-hooks)
