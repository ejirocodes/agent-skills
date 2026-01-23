---
title: Choose Correct Load Function Type
impact: HIGH
impactDescription: prevent security issues and serialization errors in SvelteKit
type: capability
tags: +page.js, +page.server.ts, load, secrets, serialization, SvelteKit
---

# Choose Correct Load Function Type

**Impact: HIGH** - prevent security issues and serialization errors in SvelteKit

SvelteKit has two load function types: universal (`+page.js`) and server-only (`+page.server.ts`). Choosing wrong can expose secrets or cause errors.

## Symptoms

- API keys or secrets exposed in browser
- "Cannot serialize" errors
- Functions not available on client
- Data fetched twice (server + client)

## Root Cause

Universal load functions run on both server and client, exposing their code to browsers. Server load functions only run on the server.

## Rules for Choosing

### Use +page.server.ts When:

1. **Accessing secrets/credentials**
2. **Database connections**
3. **Server-only APIs**
4. **Sensitive business logic**

### Use +page.js When:

1. **Public APIs only**
2. **Need to return non-serializable data** (functions, classes, components)
3. **Client-side caching benefits**

## Security Issues

**WRONG (secrets in universal load):**

```ts
// +page.js - DANGEROUS: runs in browser!
export const load = async ({ fetch }) => {
  const response = await fetch('https://api.stripe.com/charges', {
    headers: {
      'Authorization': `Bearer ${STRIPE_SECRET_KEY}` // EXPOSED TO BROWSER!
    }
  });
  return { charges: await response.json() };
};
```

**CORRECT (secrets in server load):**

```ts
// +page.server.ts - only runs on server
import { STRIPE_SECRET_KEY } from '$env/static/private';

export const load = async ({ fetch }) => {
  const response = await fetch('https://api.stripe.com/charges', {
    headers: {
      'Authorization': `Bearer ${STRIPE_SECRET_KEY}` // Safe!
    }
  });
  return { charges: await response.json() };
};
```

## Serialization Issues

**WRONG (non-serializable in server load):**

```ts
// +page.server.ts
export const load = async () => {
  return {
    // ERROR: Functions can't be serialized
    formatDate: (date: Date) => date.toLocaleDateString(),
    // ERROR: Class instances can't be serialized
    parser: new DOMParser()
  };
};
```

**CORRECT (non-serializable in universal load):**

```ts
// +page.js
export const load = async () => {
  return {
    // OK: Universal load can return functions
    formatDate: (date: Date) => date.toLocaleDateString(),
    // OK: Can return any JavaScript value
    parser: typeof window !== 'undefined' ? new DOMParser() : null
  };
};
```

## Database Access

**WRONG (database in universal load):**

```ts
// +page.js - WRONG
import { db } from '$lib/server/database';

export const load = async () => {
  // This will fail - db is server-only
  const users = await db.query('SELECT * FROM users');
  return { users };
};
```

**CORRECT (database in server load):**

```ts
// +page.server.ts
import { db } from '$lib/server/database';

export const load = async () => {
  const users = await db.query('SELECT * FROM users');
  return { users };
};
```

## Environment Variables

```ts
// +page.server.ts - Private env vars
import { DATABASE_URL, API_SECRET } from '$env/static/private';

export const load = async () => {
  // Safe to use private env vars
};

// +page.js - Only public env vars
import { PUBLIC_API_URL } from '$env/static/public';

export const load = async ({ fetch }) => {
  // Only use PUBLIC_ prefixed env vars
  const data = await fetch(PUBLIC_API_URL);
};
```

## Combining Both Load Types

You can have both for the same route:

```ts
// +page.server.ts - Runs first, server-only data
export const load = async () => {
  const sensitiveData = await getFromDatabase();
  return {
    user: sensitiveData.user,
    permissions: sensitiveData.permissions
  };
};

// +page.js - Runs after, can use server data
export const load = async ({ data }) => {
  // data contains what +page.server.ts returned
  return {
    ...data,
    // Add client-side utilities
    formatters: {
      date: (d: Date) => d.toLocaleDateString()
    }
  };
};
```

## Quick Reference

| Need | Use |
|------|-----|
| Private API keys | `+page.server.ts` |
| Database queries | `+page.server.ts` |
| Return functions | `+page.js` |
| Return class instances | `+page.js` |
| Public API calls | Either (prefer `+page.js` for caching) |
| Server-only imports | `+page.server.ts` |

## Reference

- [SvelteKit Load Functions](https://svelte.dev/docs/kit/load)
- [SvelteKit Server-only Modules](https://svelte.dev/docs/kit/server-only-modules)
