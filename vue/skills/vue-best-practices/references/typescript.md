# TypeScript Patterns

## Table of Contents

- [Extract Component Props](#extract-component-props)
- [Generic Components](#generic-components)
- [useTemplateRef Typing](#usetemplateref-typing)
- [JSDoc for Script Setup](#jsdoc-for-script-setup)
- [Reactive Props Destructure](#reactive-props-destructure)
- [withDefaults Union Types](#withdefaults-union-types)
- [Strict Template Checking](#strict-template-checking)

---

## Extract Component Props

Use `vue-component-type-helpers` to extract types from `.vue` components:

```bash
npm install -D vue-component-type-helpers
```

```typescript
import type { ComponentProps, ComponentEmit, ComponentSlots, ComponentExposed } from 'vue-component-type-helpers'
import MyButton from './MyButton.vue'

type Props = ComponentProps<typeof MyButton>
type Emits = ComponentEmit<typeof MyButton>
type Slots = ComponentSlots<typeof MyButton>
type Exposed = ComponentExposed<typeof MyButton>
```

### Wrapper Component Pattern

```typescript
import type { ComponentProps } from 'vue-component-type-helpers'
import BaseButton from './BaseButton.vue'

type BaseProps = ComponentProps<typeof BaseButton>

interface Props extends BaseProps {
  size: 'sm' | 'md' | 'lg'
}

defineProps<Props>()
```

### Do NOT Use

```typescript
// ❌ Includes Vue internal properties (onUpdate:*, class, style, etc.)
type Props = InstanceType<typeof MyButton>['$props']
```

Vue's built-in `ExtractPropTypes` is for runtime props objects (`props: { foo: String }`), not for `.vue` components.

**Reference:** [vue-component-type-helpers](https://github.com/vuejs/language-tools/tree/master/packages/component-type-helpers)

---

## Generic Components

Create type-safe generic components using the `generic` attribute on `<script>`.

### Basic Generic Component

```vue
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
  selected?: T
}>()

defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <ul>
    <li v-for="item in items" @click="$emit('select', item)">
      <slot :item="item" />
    </li>
  </ul>
</template>
```

### With Constraints

```vue
<script setup lang="ts" generic="T extends { id: string | number }">
defineProps<{
  items: T[]
  selectedId?: T['id']
}>()
</script>
```

### Multiple Type Parameters

```vue
<script setup lang="ts" generic="T, U extends keyof T">
defineProps<{
  data: T
  field: U
}>()
</script>
```

### Using Generic Components

```vue
<template>
  <!-- TypeScript infers T from items prop -->
  <GenericList :items="users" @select="handleUser">
    <template #default="{ item }">
      {{ item.name }} <!-- item is typed as User -->
    </template>
  </GenericList>
</template>
```

**Reference:** [Vue TypeScript with Composition API](https://vuejs.org/guide/typescript/composition-api)

---

## useTemplateRef Typing

Vue 3.5 introduced `useTemplateRef()` for cleaner template ref management with automatic type inference.

### Basic Usage

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// Type is inferred as ShallowRef<HTMLInputElement | null>
const inputRef = useTemplateRef('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

### With Component Refs

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import type { ComponentExposed } from 'vue-component-type-helpers'
import MyForm from './MyForm.vue'

// For generic components, use ComponentExposed
type FormExposed = ComponentExposed<typeof MyForm>
const formRef = useTemplateRef<InstanceType<typeof MyForm>>('form')

function submit() {
  formRef.value?.validate()
}
</script>

<template>
  <MyForm ref="form" />
</template>
```

### In Composables

```typescript
// composables/useChart.ts
import { useTemplateRef, onMounted, onUnmounted } from 'vue'

export function useChart(refName: string) {
  const canvasRef = useTemplateRef<HTMLCanvasElement>(refName)
  let chart: Chart | null = null

  onMounted(() => {
    if (canvasRef.value) {
      chart = new Chart(canvasRef.value, { /* config */ })
    }
  })

  onUnmounted(() => {
    chart?.destroy()
  })

  return { canvasRef, chart }
}
```

> **Note:** With `@vue/language-tools 2.1+`, static template refs' types are automatically inferred. Manual typing is only needed for edge cases.

**Reference:** [Vue Template Refs](https://vuejs.org/guide/essentials/template-refs)

---

## JSDoc for Script Setup

`<script setup>` doesn't have an obvious place to attach JSDoc comments for the component itself. Use a dual-script pattern.

### Problem

```vue
<script setup lang="ts">
/**
 * This comment doesn't appear in IDE hover or docs
 * @component
 */
import { ref } from 'vue'

const count = ref(0)
</script>
```

JSDoc comments inside `<script setup>` don't attach to the component export because there's no explicit export statement.

### Solution

Use both `<script>` and `<script setup>` blocks:

```vue
<script lang="ts">
/**
 * A counter component that displays and increments a value.
 *
 * @example
 * ```vue
 * <Counter :initial="5" @update="handleUpdate" />
 * ```
 *
 * @component
 */
export default {}
</script>

<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{
  /** Starting value for the counter */
  initial?: number
}>()

const emit = defineEmits<{
  /** Emitted when counter value changes */
  update: [value: number]
}>()

const count = ref(props.initial ?? 0)
</script>
```

### What Gets Documented

| Location | Shows In |
|----------|----------|
| `export default {}` JSDoc | Component import hover |
| `defineProps` JSDoc | Prop hover in templates |
| `defineEmits` JSDoc | Event handler hover |

**Reference:** [Vue Language Tools Discussion #5932](https://github.com/vuejs/language-tools/discussions/5932)

---

## Reactive Props Destructure

Vue 3.5 stabilized Reactive Props Destructure, enabled by default. Variables destructured from `defineProps` are automatically reactive.

### Basic Pattern

```vue
<script setup lang="ts">
// Destructured props are reactive - no ref() needed
const { name, count = 0 } = defineProps<{
  name: string
  count?: number
}>()

// Access directly in script
console.log(name, count)
</script>

<template>
  <!-- Access directly in template -->
  <div>{{ name }}: {{ count }}</div>
</template>
```

### Watching Destructured Props

Wrap in a getter when watching or passing to composables:

```typescript
const { count } = defineProps<{ count: number }>()

// ✅ Correct - wrap in getter
watch(() => count, (newVal) => {
  console.log('count changed:', newVal)
})

// ❌ Incorrect - loses reactivity
watch(count, handler) // Won't work as expected
```

### With Default Values

```vue
<script setup lang="ts">
// Defaults work naturally with destructuring
const {
  title = 'Default Title',
  items = [],
  config = { debug: false }
} = defineProps<{
  title?: string
  items?: string[]
  config?: { debug: boolean }
}>()
</script>
```

### Enable for Vue < 3.5

```javascript
// vite.config.js
export default {
  plugins: [
    vue({
      script: {
        propsDestructure: true
      }
    })
  ]
}
```

**Reference:** [Reactive Props Destructure RFC](https://github.com/vuejs/rfcs/discussions/502)

---

## withDefaults Union Types

Using `withDefaults` with union types like `false | string` may produce a Vue runtime warning "Missing required prop" even when a default is provided.

### Symptoms

- Vue warns "Missing required prop" despite default being set
- Warning appears only with union types like `false | string`
- TypeScript types are correct
- Runtime value IS correct (the default is applied)

### Problematic Pattern

```typescript
// This produces a spurious warning (but works at runtime)
interface Props {
  value: false | string  // Union type
}

const props = withDefaults(defineProps<Props>(), {
  value: 'default'  // Runtime value IS correct, but Vue warns about missing prop
})
```

### Fix

**Option 1: Use Reactive Props Destructure (Vue 3.5+) - Recommended**

```vue
<script setup lang="ts">
interface Props {
  value: false | string
}

// Preferred in Vue 3.5+
const { value = 'default' } = defineProps<Props>()
</script>
```

**Option 2: Use runtime declaration**

```vue
<script setup lang="ts">
const props = defineProps({
  value: {
    type: [Boolean, String] as PropType<false | string>,
    default: 'default'
  }
})
</script>
```

**Option 3: Split into separate props**

```typescript
interface Props {
  enabled: boolean
  customValue?: string
}

const props = withDefaults(defineProps<Props>(), {
  enabled: false,
  customValue: 'default'
})
```

**Reference:** [vuejs/core#12897](https://github.com/vuejs/core/issues/12897)

---

## Strict Template Checking

By default, vue-tsc does not report errors for undefined components in templates. Enable `strictTemplates` to catch these issues during type checking.

### Which tsconfig?

Add `vueCompilerOptions` to the tsconfig that includes Vue source files. In projects with multiple tsconfigs (like those created with `create-vue`), this is typically `tsconfig.app.json`, not the root `tsconfig.json` or `tsconfig.node.json`.

**Incorrect (missing strict checking):**

```json
{
  "compilerOptions": {
    "strict": true
  }
  // vueCompilerOptions not configured - undefined components won't error
}
```

**Correct (strict template checking enabled):**

```json
{
  "compilerOptions": {
    "strict": true
  },
  "vueCompilerOptions": {
    "strictTemplates": true
  }
}
```

### Available Options

| Option | Default | Effect |
|--------|---------|--------|
| `strictTemplates` | `false` | Enables all checkUnknown* options below |
| `checkUnknownComponents` | `false` | Error on undefined/unregistered components |
| `checkUnknownProps` | `false` | Error on props not declared in component definition |
| `checkUnknownEvents` | `false` | Error on events not declared via `defineEmits` |
| `checkUnknownDirectives` | `false` | Error on unregistered custom directives |

### Granular Control

If `strictTemplates` is too strict, enable individual checks:

```json
{
  "vueCompilerOptions": {
    "checkUnknownComponents": true,
    "checkUnknownProps": false
  }
}
```

**Reference:** [Vue Compiler Options](https://github.com/vuejs/language-tools/wiki/Vue-Compiler-Options)
