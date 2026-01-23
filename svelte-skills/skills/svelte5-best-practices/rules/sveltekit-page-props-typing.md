---
title: Type Page Data with $props in SvelteKit
impact: MEDIUM
impactDescription: properly type SvelteKit page data in Svelte 5
type: capability
tags: PageProps, PageData, $props, SvelteKit, TypeScript
---

# Type Page Data with $props in SvelteKit

**Impact: MEDIUM** - properly type SvelteKit page data in Svelte 5

Svelte 5's `$props()` requires specific typing patterns for SvelteKit page data and form actions.

## Symptoms

- No autocomplete for `data` or `form` properties
- TypeScript errors with page props
- Unclear how to type load function returns

## Root Cause

SvelteKit generates types in `./$types` that must be used with `$props()` for full type safety.

## Fix

### Basic Page Data Typing

**Incorrect (untyped):**

```svelte
<!-- +page.svelte -->
<script>
  const { data } = $props(); // No types!
</script>

<h1>{data.title}</h1> <!-- No autocomplete -->
```

**Correct (with PageProps):**

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';

  let { data }: PageProps = $props();
</script>

<h1>{data.title}</h1> <!-- Full autocomplete! -->
```

### Page Data with Form Actions

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';

  let { data, form }: PageProps = $props();
</script>

{#if form?.success}
  <p class="success">{form.message}</p>
{/if}

{#if form?.error}
  <p class="error">{form.error}</p>
{/if}

<h1>{data.title}</h1>
```

### Corresponding Load Function

```ts
// +page.server.ts
import type { PageServerLoad, Actions } from './$types';

export const load: PageServerLoad = async () => {
  return {
    title: 'My Page',
    items: await fetchItems()
  };
};

export const actions: Actions = {
  default: async ({ request }) => {
    const formData = await request.formData();
    // ... process form

    return {
      success: true,
      message: 'Saved successfully'
    };
  }
};
```

### Layout Data Typing

```svelte
<!-- +layout.svelte -->
<script lang="ts">
  import type { LayoutProps } from './$types';

  let { data, children }: LayoutProps = $props();
</script>

<nav>
  <a href="/">Home</a>
  {#if data.user}
    <span>Welcome, {data.user.name}</span>
  {/if}
</nav>

{@render children()}
```

### Accessing Parent Layout Data

```svelte
<!-- +page.svelte (nested) -->
<script lang="ts">
  import type { PageProps } from './$types';

  // PageProps includes inherited layout data
  let { data }: PageProps = $props();

  // Access both page and layout data
  const { user } = data; // From layout
  const { posts } = data; // From page
</script>
```

### Custom Props Interface

When you need to extend or customize:

```svelte
<script lang="ts">
  import type { PageData, ActionData } from './$types';

  interface Props {
    data: PageData;
    form: ActionData;
  }

  let { data, form }: Props = $props();
</script>
```

### Typing with Generics

For reusable page components:

```svelte
<!-- PageWrapper.svelte -->
<script lang="ts" generics="T extends Record<string, unknown>">
  import type { Snippet } from 'svelte';

  interface Props {
    data: T;
    children: Snippet<[data: T]>;
  }

  let { data, children }: Props = $props();
</script>

<main>
  {@render children(data)}
</main>
```

### Error Page Typing

```svelte
<!-- +error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<h1>{page.status}: {page.error?.message}</h1>
```

### Route Parameters

```ts
// +page.server.ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params }) => {
  // params.slug is typed based on route: /blog/[slug]
  const post = await getPost(params.slug);
  return { post };
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';

  let { data }: PageProps = $props();
</script>

<article>
  <h1>{data.post.title}</h1>
</article>
```

### Using $page Store (Alternative)

```svelte
<script lang="ts">
  import { page } from '$app/state';

  // Access data reactively
  $effect(() => {
    console.log(page.data);
  });
</script>

<h1>{page.data.title}</h1>
```

## Generated Types Location

Types are auto-generated in `.svelte-kit/types`:

```
.svelte-kit/types/
└── src/routes/
    ├── $types.d.ts           # Root types
    ├── blog/
    │   ├── $types.d.ts       # /blog types
    │   └── [slug]/
    │       └── $types.d.ts   # /blog/[slug] types
```

## Reference

- [SvelteKit Generated Types](https://svelte.dev/docs/kit/types)
- [SvelteKit Load Functions](https://svelte.dev/docs/kit/load)
