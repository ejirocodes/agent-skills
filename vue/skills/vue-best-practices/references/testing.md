# Testing Patterns

## Table of Contents

- [Pinia Store Mocking](#pinia-store-mocking)
- [Vue Router Typed Params](#vue-router-typed-params)

---

## Pinia Store Mocking

Developers struggle to properly mock Pinia stores: `createTestingPinia` requires explicit `createSpy` configuration, and "injection Symbol(pinia) not found" errors occur without proper setup.

> **Important (@pinia/testing 1.0+):** The `createSpy` option is **REQUIRED**, not optional. Omitting it throws an error: "You must configure the `createSpy` option."

### Symptoms

- "injection Symbol(pinia) not found" error
- "You must configure the `createSpy` option" error
- Actions not properly mocked
- Store state not reset between tests

### Fix

**Pattern 1: Basic setup with createTestingPinia**

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
          createSpy: vi.fn,  // REQUIRED in @pinia/testing 1.0+
          initialState: {
            counter: { count: 10 }  // Set initial state
          }
        })
      ]
    }
  })

  // Get the store instance AFTER mounting
  const store = useCounterStore()

  // Actions are automatically stubbed
  await wrapper.find('button').trigger('click')
  expect(store.increment).toHaveBeenCalled()
})
```

**Pattern 2: Customize action behavior**

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

  // Override specific action
  store.fetchData = vi.fn().mockResolvedValue({ items: [] })

  await wrapper.find('.load-button').trigger('click')
  expect(store.fetchData).toHaveBeenCalled()
})
```

**Pattern 3: Testing store directly**

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
})
```

### Setup Store with Vitest

```typescript
// stores/counter.ts - Setup store syntax
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  return { count, doubleCount, increment }
})

// Test file
test('setup store works', async () => {
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
})
```

### Reset Between Tests

```typescript
describe('Store Tests', () => {
  let pinia: Pinia

  beforeEach(() => {
    pinia = createTestingPinia({
      createSpy: vi.fn
    })
  })

  afterEach(() => {
    vi.clearAllMocks()
  })

  test('test 1', () => { /* ... */ })
  test('test 2', () => { /* ... */ })
})
```

**Reference:** [Pinia Testing Guide](https://pinia.vuejs.org/cookbook/testing.html)

---

## Vue Router Typed Params

With `unplugin-vue-router` typed routes, `route.params` becomes a union of ALL page param types. TypeScript cannot narrow this properly.

### Symptoms

- "Property 'id' does not exist on type 'RouteParams'"
- `route.params.id` shows as `string | undefined` everywhere
- Union type of all route params instead of specific route
- Type narrowing with `if (route.name === 'users-id')` doesn't work

### Root Cause

`unplugin-vue-router` generates a union type of all possible route params. TypeScript's control flow analysis can't narrow this union based on route name checks.

### Fix

**Option 1: Pass route name to useRoute (recommended)**

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

### Required tsconfig Setting

Ensure `moduleResolution: "bundler"` for unplugin-vue-router:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"
  }
}
```

### Route Name Format

The route name matches the file path pattern:
- `pages/users/[id].vue` → `/users/[id]`
- `pages/posts/[slug]/comments.vue` → `/posts/[slug]/comments`

**Reference:** [unplugin-vue-router TypeScript docs](https://uvr.esm.is/guide/typescript.html)
