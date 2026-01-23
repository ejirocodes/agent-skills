# Testing Patterns

## Table of Contents

- [Pinia Store Mocking](#pinia-store-mocking)
- [Testing Setup Stores](#testing-setup-stores)
- [Vue Router Typed Params](#vue-router-typed-params)
- [Automatic Route Typing with Volar Plugin](#automatic-route-typing-with-volar-plugin)

---

## Pinia Store Mocking

`createTestingPinia` creates a Pinia instance designed for unit tests that automatically mocks stores.

> **Important (@pinia/testing 1.0+):** The `createSpy` option is **REQUIRED** when `globals: true` is not set in Vitest config. Omitting it throws: "You must configure the `createSpy` option."

### Symptoms

- "injection Symbol(pinia) not found" error
- "You must configure the `createSpy` option" error
- Actions not properly mocked
- Store state not reset between tests

### Basic Setup

```typescript
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import { vi } from 'vitest'
import MyComponent from './MyComponent.vue'
import { useCounterStore } from '@/stores/counter'

test('component uses store', async () => {
  const wrapper = mount(MyComponent, {
    global: {
      plugins: [
        createTestingPinia({
          createSpy: vi.fn,  // REQUIRED without globals: true
          initialState: {
            counter: { count: 10 }  // Set initial state
          }
        })
      ]
    }
  })

  // Get store instance AFTER mounting
  const store = useCounterStore()

  // Actions are automatically stubbed
  await wrapper.find('button').trigger('click')
  expect(store.increment).toHaveBeenCalled()
})
```

### With `globals: true` in Vitest Config

If you have `globals: true` in your Vitest config, `createSpy` is auto-detected:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true  // Enables auto-detection of vi.fn
  }
})

// Test file - createSpy not needed
const pinia = createTestingPinia({
  initialState: { counter: { count: 5 } }
})
```

### Custom Action Behavior

```typescript
test('component handles async action', async () => {
  const wrapper = mount(MyComponent, {
    global: {
      plugins: [
        createTestingPinia({
          createSpy: vi.fn,
          stubActions: false  // Don't stub, use real actions
        })
      ]
    }
  })

  const store = useCounterStore()

  // Override specific action with mock implementation
  store.fetchData = vi.fn().mockResolvedValue({ items: [] })

  await wrapper.find('.load-button').trigger('click')
  expect(store.fetchData).toHaveBeenCalled()
})
```

### Using `vi.spyOn` for More Control

```typescript
test('action with custom mock', async () => {
  const wrapper = mount(MyComponent, {
    global: {
      plugins: [createTestingPinia({ createSpy: vi.fn })]
    }
  })

  const store = useCounterStore()

  // More control with spyOn
  const spy = vi.spyOn(store, 'increment')
  spy.mockImplementation(() => {
    store.count += 10  // Custom increment
  })

  await wrapper.find('button').trigger('click')
  expect(store.count).toBe(10)
})
```

### Reset Between Tests

```typescript
describe('Store Tests', () => {
  let pinia: ReturnType<typeof createTestingPinia>

  beforeEach(() => {
    pinia = createTestingPinia({
      createSpy: vi.fn
    })
  })

  afterEach(() => {
    vi.clearAllMocks()
  })

  test('test 1', () => { /* fresh pinia instance */ })
  test('test 2', () => { /* fresh pinia instance */ })
})
```

### Including Pinia Plugins

```typescript
import { myPiniaPlugin } from '@/plugins/pinia'

const pinia = createTestingPinia({
  createSpy: vi.fn,
  plugins: [myPiniaPlugin]  // Pass plugins here, not with .use()
})
```

**Reference:** [Pinia Testing Guide](https://pinia.vuejs.org/cookbook/testing.html)

---

## Testing Setup Stores

Setup stores (function-based) have special testing considerations because they don't have an `options.actions` object.

### Direct Store Testing

```typescript
import { setActivePinia, createPinia } from 'pinia'
import { useCounterStore } from '@/stores/counter'

describe('Counter Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  test('increments count', () => {
    const store = useCounterStore()
    expect(store.count).toBe(0)

    store.increment()
    expect(store.count).toBe(1)
  })

  test('computed values', () => {
    const store = useCounterStore()
    store.count = 5
    expect(store.doubleCount).toBe(10)
  })
})
```

### Setup Store Definition

```typescript
// stores/counter.ts
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  async function fetchCount() {
    const response = await fetch('/api/count')
    count.value = await response.json()
  }

  return { count, doubleCount, increment, fetchCount }
})
```

### Component Testing with Setup Store

```typescript
test('setup store in component', async () => {
  const pinia = createTestingPinia({
    createSpy: vi.fn,
    initialState: {
      counter: { count: 5 }
    }
  })

  const wrapper = mount(MyComponent, {
    global: { plugins: [pinia] }
  })

  const store = useCounterStore()
  expect(store.count).toBe(5)
  expect(store.doubleCount).toBe(10)

  // Action is stubbed by default
  await wrapper.find('button').trigger('click')
  expect(store.increment).toHaveBeenCalled()
})
```

### Testing $subscribe

```typescript
test('store subscription', async () => {
  setActivePinia(createPinia())
  const store = useCounterStore()

  const callback = vi.fn()
  store.$subscribe(callback)

  store.increment()

  expect(callback).toHaveBeenCalledWith(
    expect.objectContaining({ storeId: 'counter' }),
    expect.objectContaining({ count: 1 })
  )
})
```

**Reference:** [Pinia Testing Guide](https://pinia.vuejs.org/cookbook/testing.html)

---

## Vue Router Typed Params

With `unplugin-vue-router`, `route.params` becomes a union of ALL page param types. TypeScript cannot narrow this properly without help.

### Symptoms

- "Property 'id' does not exist on type 'RouteParams'"
- `route.params.id` shows as `string | undefined` everywhere
- Union type of all route params instead of specific route
- Type narrowing with `if (route.name === 'users-id')` doesn't work

### Root Cause

`unplugin-vue-router` generates a union type of all possible route params. TypeScript's control flow analysis can't narrow this union based on route name checks.

### Fix

**Option 1: Pass route path to useRoute (recommended)**

```typescript
// pages/users/[id].vue
import { useRoute } from 'vue-router/auto'

// Specify the route path for proper typing
const route = useRoute('/users/[id]')

// Now properly typed as { id: string }
console.log(route.params.id)  // string, not string | undefined
```

**Option 2: Type assertion with specific route**

```typescript
import { useRoute } from 'vue-router'
import type { RouteLocationNormalized } from 'vue-router/auto-routes'

const route = useRoute() as RouteLocationNormalized<'/users/[id]'>
route.params.id  // Properly typed
```

**Option 3: Define route-specific param type**

```typescript
// In your page component
interface UserRouteParams {
  id: string
}

const route = useRoute()
const { id } = route.params as UserRouteParams
```

### Route Path Format

The route path matches the file path pattern:

| File Path | Route Path |
|-----------|------------|
| `pages/users/[id].vue` | `/users/[id]` |
| `pages/posts/[slug]/comments.vue` | `/posts/[slug]/comments` |
| `pages/[...path].vue` | `/[...path]` |

### Required tsconfig Settings

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"
  }
}
```

**Reference:** [unplugin-vue-router TypeScript docs](https://uvr.esm.is/guide/typescript)

---

## Automatic Route Typing with Volar Plugin

The `sfc-typed-router` Volar plugin automatically types `useRoute()` and `$route` in page components, eliminating the need for manual path specification.

### Setup

Add to your `tsconfig.json`:

```json
{
  "compilerOptions": {
    "rootDir": "."
  },
  "vueCompilerOptions": {
    "plugins": ["unplugin-vue-router/volar/sfc-typed-router"]
  }
}
```

### How It Works

With the plugin enabled, `useRoute()` in a page component automatically uses the correct route type:

```vue
<!-- pages/users/[id].vue -->
<script setup lang="ts">
// No path needed - automatically typed based on file location
const route = useRoute()

// route.params.id is properly typed as string
console.log(route.params.id)
</script>
```

### Benefits

- No need to write `useRoute('/users/[id]')` in every page
- `$route` in templates is also typed correctly
- Works with both Composition API and Options API
- Autocomplete for route params

### Limitations

- Only works in page components (files under `pages/`)
- Non-page components still need explicit typing
- Requires Volar and correct tsconfig setup

**Reference:** [unplugin-vue-router Volar Plugin](https://uvr.esm.is/guide/typescript.html#auto-typed-useRoute)
