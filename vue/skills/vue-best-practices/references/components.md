# Component Patterns

## Table of Contents

- [defineModel Patterns](#definemodel-patterns)
- [Deep Watch with Numeric Depth](#deep-watch-with-numeric-depth)
- [onWatcherCleanup](#onwatchercleanup)
- [useId for SSR](#useid-for-ssr)
- [Deferred Teleport](#deferred-teleport)

---

## defineModel Patterns

Components using `defineModel` may fire the `@update:model-value` event with `undefined` in certain edge cases. TypeScript types don't always reflect this behavior.

> **Version Note:** This issue may be resolved in Vue 3.5+. Testing with Vue 3.5.26 could not reproduce the double emission with `undefined`. Verify the issue exists in your specific scenario before applying workarounds.

### Symptoms

- Parent component receives `undefined` unexpectedly
- Runtime error: "Cannot read property of undefined"
- Type mismatch between expected `T` and received `T | undefined`
- Issue appears when clearing/resetting the model value

### Root Cause

`defineModel` returns `Ref<T | undefined>` by default, even when `T` is non-nullable. The update event can fire with `undefined` when:
- Component unmounts
- Model is explicitly cleared
- Internal state resets

### Fix

**Option 1: Use required option (Vue 3.5+) - Recommended**

```typescript
// Returns Ref<Item> instead of Ref<Item | undefined>
const model = defineModel<Item>({ required: true })
```

**Option 2: Type parent handler to accept undefined**

```vue
<template>
  <MyComponent
    v-model="item"
    @update:model-value="handleUpdate"
  />
</template>

<script setup lang="ts">
// Handle both value and undefined
const handleUpdate = (value: Item | undefined) => {
  if (value !== undefined) {
    item.value = value
  }
}
</script>
```

**Option 3: Use default value in defineModel**

```typescript
const model = defineModel<string>({ default: '' })
```

### Multiple v-models

```vue
<script setup lang="ts">
// Named models
const firstName = defineModel<string>('firstName', { required: true })
const lastName = defineModel<string>('lastName', { required: true })

// Default model
const selected = defineModel<boolean>({ default: false })
</script>
```

### Type Declaration Pattern

```typescript
// In child component
interface Props {
  modelValue: Item
}
const model = defineModel<Item>({ required: true })

// Emits will be typed as (value: Item) not (value: Item | undefined)
```

**Reference:** [vuejs/core#12817](https://github.com/vuejs/core/issues/12817)

---

## Deep Watch with Numeric Depth

Vue 3.5 introduced `deep: number` for watch depth control. This allows watching array mutations without the performance cost of deep traversal.

### Symptoms

- Array mutations not triggering watch callback
- Deep watch causing performance issues on large nested objects

### The Feature

```typescript
// Vue 3.5+ only
watch(items, (newVal) => {
  // Triggered on array mutations (push, pop, splice, etc.)
}, { deep: 1 })
```

| deep value | Behavior |
|------------|----------|
| `true` | Full recursive traversal (original behavior) |
| `false` | Only reference changes |
| `1` | One level deep - array mutations, not nested objects |
| `2` | Two levels deep |
| `n` | N levels deep |

### Fix

**Step 1: Ensure Vue 3.5+**

```bash
npm install vue@^3.5.0
```

**Step 2: Use numeric depth**

```typescript
import { watch, ref } from 'vue'

const items = ref([{ id: 1, data: { nested: 'value' } }])

// Watch array mutations only (push, pop, etc.)
watch(items, (newItems) => {
  console.log('Array mutated')
}, { deep: 1 })

// Won't trigger on: items.value[0].data.nested = 'new'
// Will trigger on: items.value.push(newItem)
```

### Performance Comparison

```typescript
const largeNestedData = ref({ /* deeply nested structure */ })

// SLOW - traverses entire structure
watch(largeNestedData, handler, { deep: true })

// FAST - only watches top-level changes
watch(largeNestedData, handler, { deep: 1 })

// FASTEST - only reference changes
watch(largeNestedData, handler, { deep: false })
```

### Alternative: watchEffect for Selective Tracking

```typescript
// Only tracks properties actually accessed
watchEffect(() => {
  // Only re-runs when items.value.length or first item changes
  console.log(items.value.length, items.value[0]?.id)
})
```

### TypeScript Note

If TypeScript complains about numeric deep, ensure:
1. Vue version is 3.5+
2. TypeScript version is current (types are included with `vue` package)
3. tsconfig targets correct node_modules types

**Reference:** [Vue 3.5 Release Notes](https://blog.vuejs.org/posts/vue-3-5)

---

## onWatcherCleanup

Vue 3.5 introduced `onWatcherCleanup()` for registering cleanup callbacks in watchers. This simplifies managing side effects like aborting requests, clearing timers, or cleaning up subscriptions.

### Basic Usage

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(searchQuery, async (query) => {
  const controller = new AbortController()

  // Register cleanup - runs when watcher re-runs or component unmounts
  onWatcherCleanup(() => {
    controller.abort()
  })

  const response = await fetch(`/api/search?q=${query}`, {
    signal: controller.signal
  })
  results.value = await response.json()
})
```

### In Nested Functions

Unlike the `onCleanup` parameter, `onWatcherCleanup` can be used in nested functions:

```typescript
watch(id, (newId) => {
  startPolling(newId)
})

function startPolling(id: string) {
  const interval = setInterval(() => {
    fetchData(id)
  }, 5000)

  // Works even though it's not directly in the watch callback
  onWatcherCleanup(() => {
    clearInterval(interval)
  })
}
```

### With watchEffect

```typescript
import { watchEffect, onWatcherCleanup } from 'vue'

watchEffect(() => {
  const connection = createWebSocket(url.value)

  onWatcherCleanup(() => {
    connection.close()
  })
})
```

### Comparison with onCleanup Parameter

```typescript
// Old way (still valid)
watch(source, async (value, oldValue, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())
  // ...
})

// New way (Vue 3.5+)
watch(source, async (value) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())
  // ...
})
```

### Suppress Warnings

If called outside an active watcher, it logs a warning. Suppress with second parameter:

```typescript
// Suppress "onWatcherCleanup called outside of watcher" warning
onWatcherCleanup(() => cleanup(), true)
```

**Reference:** [Vue 3.5 Release Notes](https://blog.vuejs.org/posts/vue-3-5)

---

## useId for SSR

Vue 3.5 introduces `useId()` to generate unique, SSR-stable IDs for accessibility attributes and form elements.

### Basic Usage

```vue
<script setup>
import { useId } from 'vue'

const inputId = useId()
</script>

<template>
  <label :for="inputId">Email</label>
  <input :id="inputId" type="email" />
</template>
```

### Why useId?

- **SSR-safe:** IDs are stable across server and client renders, preventing hydration mismatches
- **Unique per component instance:** Multiple instances get different IDs
- **Accessible:** Perfect for `aria-labelledby`, `aria-describedby`, form labels

### Multiple IDs

```vue
<script setup>
import { useId } from 'vue'

const nameId = useId()
const emailId = useId()
const errorId = useId()
</script>

<template>
  <div>
    <label :for="nameId">Name</label>
    <input :id="nameId" :aria-describedby="errorId" />
    <span :id="errorId" class="error">Required</span>
  </div>
</template>
```

### Compared to Random IDs

```typescript
// ❌ Causes hydration mismatch in SSR
const id = `input-${Math.random().toString(36).slice(2)}`

// ✅ Stable across server/client
const id = useId()
```

**Reference:** [Vue 3.5 Release Notes](https://blog.vuejs.org/posts/vue-3-5)

---

## Deferred Teleport

Vue 3.5 introduces the `defer` prop for `<Teleport>` which mounts content after the current render cycle, solving timing issues with dynamic targets.

### Problem

```vue
<!-- ❌ Target doesn't exist yet during first render -->
<Teleport to="#modal-container">
  <Modal />
</Teleport>

<div id="modal-container"></div>
```

### Solution

```vue
<!-- ✅ Defers teleport until after current render cycle -->
<Teleport to="#modal-container" defer>
  <Modal />
</Teleport>

<div id="modal-container"></div>
```

### Use Cases

- Teleporting to elements rendered later in the same component
- Dynamic portals that may not exist immediately
- Nested component structures where target is created by a child

### Important Notes

- `defer` is opt-in for backwards compatibility
- The teleported content renders in the next tick
- Target element must exist by the time deferred teleport executes

**Reference:** [Vue 3.5 Release Notes](https://blog.vuejs.org/posts/vue-3-5)
