---
title: Prevent Cross-Request State Leaks in SSR
impact: HIGH
impactDescription: prevent data leaks between concurrent SSR requests
type: capability
tags: SSR, state leak, shared state, server, SvelteKit
---

# Prevent Cross-Request State Leaks in SSR

**Impact: HIGH** - prevent data leaks between concurrent SSR requests

Shared server state persists across requests, potentially leaking data between users.

## Symptoms

- User A sees User B's data
- Authentication state persists incorrectly
- Cached data shown to wrong users
- Random data inconsistencies in production

## Root Cause

Module-level variables on the server persist across all requests. In SSR, multiple concurrent requests share the same server process.

## Dangerous Patterns

### Danger 1: Module-Level State

**WRONG:**

```ts
// +page.server.ts
let currentUser = null; // SHARED ACROSS ALL REQUESTS!

export const load = async ({ locals }) => {
  currentUser = locals.user; // User B's request overwrites User A's
  return { user: currentUser };
};
```

**CORRECT:**

```ts
// +page.server.ts
export const load = async ({ locals }) => {
  // Each request gets its own locals
  return { user: locals.user };
};
```

### Danger 2: Global Stores

**WRONG:**

```ts
// stores.svelte.ts
export const user = $state<User | null>(null); // Server-side singleton!

// +layout.server.ts
export const load = async ({ locals }) => {
  user = locals.user; // DANGER: Affects all requests
  return {};
};
```

**CORRECT:**

```ts
// +layout.server.ts
export const load = async ({ locals }) => {
  // Return data, don't set global state
  return { user: locals.user };
};

// +layout.svelte
<script lang="ts">
  import type { LayoutProps } from './$types';
  let { data, children }: LayoutProps = $props();

  // Client-side only store is safe
</script>
```

### Danger 3: Cached API Responses

**WRONG:**

```ts
// api.ts
let cachedData: Data | null = null;

export async function fetchData() {
  if (cachedData) return cachedData; // Returns stale data to all users
  cachedData = await fetch('/api/data').then(r => r.json());
  return cachedData;
}
```

**CORRECT:**

```ts
// +page.server.ts
export const load = async ({ fetch, depends }) => {
  depends('app:data');

  // Each request fetches fresh data
  const data = await fetch('/api/data').then(r => r.json());
  return { data };
};
```

### Danger 4: Mutable Service Classes

**WRONG:**

```ts
// services/auth.ts
class AuthService {
  private user: User | null = null; // Shared!

  setUser(user: User) {
    this.user = user;
  }

  getUser() {
    return this.user;
  }
}

export const auth = new AuthService(); // Singleton - DANGER
```

**CORRECT:**

```ts
// hooks.server.ts
export const handle = async ({ event, resolve }) => {
  // Set user per-request in locals
  event.locals.user = await getUserFromSession(event.cookies);
  return resolve(event);
};

// +page.server.ts
export const load = async ({ locals }) => {
  return { user: locals.user }; // Safe per-request access
};
```

## Safe Patterns

### Pattern 1: Use locals

```ts
// hooks.server.ts
export const handle = async ({ event, resolve }) => {
  // Set request-specific data
  event.locals.user = await authenticate(event);
  event.locals.requestId = crypto.randomUUID();
  event.locals.startTime = Date.now();

  return resolve(event);
};

// +page.server.ts
export const load = async ({ locals }) => {
  // Access request-specific data safely
  console.log(`Request ${locals.requestId} for user ${locals.user?.id}`);
  return { user: locals.user };
};
```

### Pattern 2: Return Data from Load

```ts
// +layout.server.ts
export const load = async ({ locals, cookies }) => {
  return {
    user: locals.user,
    theme: cookies.get('theme') ?? 'light',
    locale: locals.locale
  };
};

// Access in any child via $page.data or props
```

### Pattern 3: Context for Component Trees

```svelte
<!-- +layout.svelte -->
<script lang="ts">
  import { setContext } from 'svelte';
  import type { LayoutProps } from './$types';

  let { data, children }: LayoutProps = $props();

  // Set context from SSR data - safe per-request
  setContext('user', {
    get current() { return data.user; }
  });
</script>

{@render children()}
```

### Pattern 4: Client-Only State

```ts
// stores.svelte.ts
import { browser } from '$app/environment';

// Only create state on client
function createClientStore() {
  if (!browser) {
    return { value: null };
  }

  const state = $state({ value: null });
  return state;
}

export const clientState = createClientStore();
```

### Pattern 5: Request-Scoped Services

```ts
// +page.server.ts
export const load = async ({ locals, fetch }) => {
  // Create service per-request
  const api = createApiClient({
    fetch,
    token: locals.user?.token
  });

  const data = await api.getData();
  return { data };
};

function createApiClient({ fetch, token }: { fetch: typeof globalThis.fetch; token?: string }) {
  return {
    async getData() {
      const res = await fetch('/api/data', {
        headers: token ? { Authorization: `Bearer ${token}` } : {}
      });
      return res.json();
    }
  };
}
```

## Checklist

- [ ] No module-level `let` variables that store user data
- [ ] No global `$state` that gets set during SSR
- [ ] No singleton service classes with mutable state
- [ ] All user-specific data flows through `locals`
- [ ] All page data comes from load function returns
- [ ] Context is set from load data, not global state

## Reference

- [SvelteKit State Management](https://svelte.dev/docs/kit/state-management)
- [SvelteKit Hooks](https://svelte.dev/docs/kit/hooks)
