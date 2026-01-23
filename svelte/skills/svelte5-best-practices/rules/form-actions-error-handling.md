---
title: Handle Form Action Errors Correctly
impact: HIGH
impactDescription: properly handle validation and errors in SvelteKit form actions
type: capability
tags: fail, form actions, validation, error, SvelteKit
---

# Handle Form Action Errors Correctly

**Impact: HIGH** - properly handle validation and errors in SvelteKit form actions

SvelteKit distinguishes between validation errors (`fail()`) and unexpected errors (`throw error()`).

## Symptoms

- Validation errors treated as 500 server errors
- Form data lost after validation failure
- Wrong error pages shown to users
- `form` prop not updating with errors

## Root Cause

Using `throw error()` for validation resets the page. Use `fail()` to return validation errors while preserving form state.

## Rules

### Rule 1: Use fail() for Validation Errors

**WRONG (validation as thrown error):**

```ts
// +page.server.ts
import { error } from '@sveltejs/kit';

export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email');

    if (!email?.toString().includes('@')) {
      throw error(400, 'Invalid email'); // WRONG: Shows error page
    }
  }
};
```

**CORRECT (validation with fail):**

```ts
// +page.server.ts
import { fail } from '@sveltejs/kit';

export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email')?.toString() ?? '';

    if (!email.includes('@')) {
      return fail(400, {
        error: 'Invalid email address',
        email // Return submitted value for repopulation
      });
    }

    // Success
    return { success: true };
  }
};
```

### Rule 2: Use throw error() for Unexpected Errors

```ts
// +page.server.ts
import { fail, error } from '@sveltejs/kit';

export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();

    // Validation - use fail()
    if (!data.get('title')) {
      return fail(400, { error: 'Title is required' });
    }

    try {
      await database.save(data);
      return { success: true };
    } catch (e) {
      // Unexpected error - use throw
      console.error('Database error:', e);
      throw error(500, 'Unable to save. Please try again later.');
    }
  }
};
```

### Rule 3: Structure Validation Responses

```ts
// +page.server.ts
import { fail } from '@sveltejs/kit';

interface FormErrors {
  email?: string;
  password?: string;
  general?: string;
}

export const actions = {
  register: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email')?.toString() ?? '';
    const password = data.get('password')?.toString() ?? '';

    const errors: FormErrors = {};

    // Collect all errors
    if (!email) {
      errors.email = 'Email is required';
    } else if (!email.includes('@')) {
      errors.email = 'Invalid email format';
    }

    if (!password) {
      errors.password = 'Password is required';
    } else if (password.length < 8) {
      errors.password = 'Password must be at least 8 characters';
    }

    // Return all errors at once
    if (Object.keys(errors).length > 0) {
      return fail(400, {
        errors,
        values: { email } // Don't return password
      });
    }

    // Process registration...
    return { success: true };
  }
};
```

### Rule 4: Handle in Component

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  import { enhance } from '$app/forms';

  let { form }: PageProps = $props();
</script>

<form method="POST" action="?/register" use:enhance>
  <label>
    Email
    <input
      name="email"
      type="email"
      value={form?.values?.email ?? ''}
      aria-invalid={form?.errors?.email ? 'true' : undefined}
    />
    {#if form?.errors?.email}
      <span class="error">{form.errors.email}</span>
    {/if}
  </label>

  <label>
    Password
    <input
      name="password"
      type="password"
      aria-invalid={form?.errors?.password ? 'true' : undefined}
    />
    {#if form?.errors?.password}
      <span class="error">{form.errors.password}</span>
    {/if}
  </label>

  {#if form?.errors?.general}
    <div class="error-banner">{form.errors.general}</div>
  {/if}

  {#if form?.success}
    <div class="success-banner">Registration successful!</div>
  {/if}

  <button type="submit">Register</button>
</form>
```

### Rule 5: Named Actions

```ts
// +page.server.ts
import { fail } from '@sveltejs/kit';

export const actions = {
  login: async ({ request }) => {
    const data = await request.formData();
    // ... validation
    if (!valid) {
      return fail(400, { action: 'login', error: 'Invalid credentials' });
    }
    return { action: 'login', success: true };
  },

  register: async ({ request }) => {
    const data = await request.formData();
    // ... validation
    if (!valid) {
      return fail(400, { action: 'register', error: 'Email taken' });
    }
    return { action: 'register', success: true };
  }
};
```

```svelte
<!-- +page.svelte -->
<form method="POST" action="?/login" use:enhance>
  {#if form?.action === 'login' && form?.error}
    <p class="error">{form.error}</p>
  {/if}
  <!-- login fields -->
</form>

<form method="POST" action="?/register" use:enhance>
  {#if form?.action === 'register' && form?.error}
    <p class="error">{form.error}</p>
  {/if}
  <!-- register fields -->
</form>
```

### Rule 6: Progressive Enhancement

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';

  let loading = $state(false);
</script>

<form
  method="POST"
  use:enhance={() => {
    loading = true;

    return async ({ update }) => {
      loading = false;
      await update(); // Updates form prop
    };
  }}
>
  <button disabled={loading}>
    {loading ? 'Saving...' : 'Save'}
  </button>
</form>
```

## Error Response Summary

| Situation | Function | HTTP Status | Result |
|-----------|----------|-------------|--------|
| Missing field | `fail(400, {...})` | 400 | Form state preserved |
| Invalid format | `fail(400, {...})` | 400 | Form state preserved |
| Duplicate entry | `fail(409, {...})` | 409 | Form state preserved |
| Not found | `throw error(404, ...)` | 404 | Error page |
| Server crash | `throw error(500, ...)` | 500 | Error page |
| Auth required | `throw redirect(303, ...)` | 303 | Redirect |

## Reference

- [SvelteKit Form Actions](https://svelte.dev/docs/kit/form-actions)
- [SvelteKit fail()](https://svelte.dev/docs/kit/@sveltejs-kit#fail)
