---
title: Two-Way Binding with $bindable
impact: HIGH
impactDescription: enable two-way binding on component props in Svelte 5
type: capability
tags: $bindable, bind, two-way, props, v-model
---

# Two-Way Binding with $bindable

**Impact: HIGH** - enable two-way binding on component props in Svelte 5

In Svelte 5, props must be explicitly marked with `$bindable()` to support two-way binding via `bind:`.

## Symptoms

- `bind:prop` fails with "prop is not bindable" error
- Two-way binding worked in Svelte 4 but not in Svelte 5
- Parent component can't bind to child's prop

## Root Cause

Svelte 5 requires explicit opt-in for bindable props using `$bindable()`. This provides clearer component contracts and prevents accidental mutations.

## Fix

**Incorrect (missing $bindable):**

```svelte
<!-- Input.svelte -->
<script>
  let { value } = $props();
</script>

<input {value} oninput={(e) => value = e.target.value} />

<!-- Parent.svelte - THIS WILL FAIL -->
<script>
  import Input from './Input.svelte';
  let name = $state('');
</script>

<Input bind:value={name} /> <!-- Error: value is not bindable -->
```

**Correct (with $bindable):**

```svelte
<!-- Input.svelte -->
<script>
  let { value = $bindable() } = $props();
</script>

<input bind:value />

<!-- Parent.svelte -->
<script>
  import Input from './Input.svelte';
  let name = $state('');
</script>

<Input bind:value={name} /> <!-- Works! -->
```

## Bindable with Default Value

```svelte
<script>
  let { value = $bindable('default') } = $props();
</script>
```

## Multiple Bindable Props

```svelte
<script>
  let {
    value = $bindable(''),
    checked = $bindable(false),
    selected = $bindable(null)
  } = $props();
</script>
```

## TypeScript with Bindable

```svelte
<script lang="ts">
  interface Props {
    value: string;
    disabled?: boolean;
  }

  let { value = $bindable(''), disabled = false }: Props = $props();
</script>
```

## Custom Components with Bindable

```svelte
<!-- ColorPicker.svelte -->
<script lang="ts">
  let { color = $bindable('#000000') }: { color: string } = $props();
</script>

<input type="color" bind:value={color} />

<!-- Parent.svelte -->
<script>
  import ColorPicker from './ColorPicker.svelte';
  let selectedColor = $state('#ff0000');
</script>

<ColorPicker bind:color={selectedColor} />
<p>Selected: {selectedColor}</p>
```

## Bindable with Callback (Controlled Pattern)

For more control, combine bindable with callback:

```svelte
<!-- Slider.svelte -->
<script lang="ts">
  interface Props {
    value: number;
    onChange?: (value: number) => void;
  }

  let { value = $bindable(0), onChange }: Props = $props();

  function handleInput(e: Event) {
    const newValue = (e.target as HTMLInputElement).valueAsNumber;
    value = newValue;
    onChange?.(newValue);
  }
</script>

<input type="range" {value} oninput={handleInput} />
```

## When NOT to Use $bindable

Prefer one-way data flow when possible:

```svelte
<!-- Prefer callback props over bindable for most cases -->
<script>
  let { value, onValueChange } = $props();
</script>

<input {value} oninput={(e) => onValueChange?.(e.target.value)} />
```

Use `$bindable` primarily for:
- Form inputs that naturally need two-way binding
- Components that wrap native form elements
- Cases where binding significantly simplifies the API

## Reference

- [Svelte 5 $bindable Documentation](https://svelte.dev/docs/svelte/$bindable)
- [Svelte 5 Component Props](https://svelte.dev/docs/svelte/$props)
