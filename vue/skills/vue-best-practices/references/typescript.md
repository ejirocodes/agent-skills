# TypeScript Patterns

## Table of Contents

- [Extract Component Props](#extract-component-props)
- [JSDoc for Script Setup](#jsdoc-for-script-setup)
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

### Enable Reactive Props Destructure

This is enabled by default in Vue 3.5+. For older versions:

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
