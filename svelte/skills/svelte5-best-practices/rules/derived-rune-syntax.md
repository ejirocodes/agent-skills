---
title: Computing Derived Values with $derived
impact: HIGH
impactDescription: compute reactive derived values in Svelte 5
type: capability
tags: $derived, computed, reactive, expression, runes
---

# Computing Derived Values with $derived

**Impact: HIGH** - compute reactive derived values in Svelte 5

Svelte 5 replaces `$:` reactive statements with the `$derived` rune for computed values.

## Symptoms

- Using `$: derivedValue = expression` produces deprecation warnings
- Derived values don't update when dependencies change
- Unclear which rune to use for computed values

## Root Cause

Svelte 5 introduced `$derived` as a replacement for `$:` reactive statements when computing derived values. The `$derived` rune provides more explicit dependency tracking.

## Fix

**Incorrect (Svelte 4 pattern):**

```svelte
<script>
  let count = 0;
  $: doubled = count * 2; // Deprecated in Svelte 5
  $: quadrupled = doubled * 2;
</script>
```

**Correct (Svelte 5 pattern):**

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  let quadrupled = $derived(doubled * 2);
</script>
```

## Complex Derivations with $derived.by

For multi-line derivations or complex logic, use `$derived.by()`:

```svelte
<script>
  let items = $state([1, 2, 3, 4, 5]);
  let filter = $state('even');

  let filteredItems = $derived.by(() => {
    if (filter === 'even') {
      return items.filter(n => n % 2 === 0);
    }
    return items.filter(n => n % 2 !== 0);
  });
</script>
```

## Deriving from Props

```svelte
<script>
  let { firstName, lastName } = $props();

  let fullName = $derived(`${firstName} ${lastName}`);
</script>

<p>Hello, {fullName}!</p>
```

## $derived vs $effect

Use `$derived` for computing values, `$effect` for side effects:

```svelte
<script>
  let count = $state(0);

  // CORRECT: Use $derived for computed values
  let doubled = $derived(count * 2);

  // INCORRECT: Don't use $effect to set derived values
  // let doubled;
  // $effect(() => {
  //   doubled = count * 2; // Anti-pattern!
  // });
</script>
```

## Derived Objects

```svelte
<script>
  let user = $state({ firstName: 'John', lastName: 'Doe' });

  let userInfo = $derived({
    fullName: `${user.firstName} ${user.lastName}`,
    initials: `${user.firstName[0]}${user.lastName[0]}`
  });
</script>

<p>{userInfo.fullName} ({userInfo.initials})</p>
```

## Reference

- [Svelte 5 $derived Documentation](https://svelte.dev/docs/svelte/$derived)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
