---
title: Testing Svelte 5 Components with Vitest
impact: HIGH
impactDescription: correctly set up component testing for Svelte 5
type: efficiency
tags: vitest, testing, browser mode, component, @testing-library/svelte
---

# Testing Svelte 5 Components with Vitest

**Impact: HIGH** - correctly set up component testing for Svelte 5

Svelte 5 components with runes require proper Vitest configuration for testing.

## Symptoms

- "$state is not defined" errors in tests
- Runes don't work in test environment
- Components not rendering correctly in tests
- Reactivity not working during tests

## Root Cause

Svelte 5 runes need the Svelte compiler. Tests must either use browser mode or proper JSDOM configuration with Svelte preprocessing.

## Setup

### Option 1: Vitest Browser Mode (Recommended)

Browser mode runs tests in a real browser, ensuring full compatibility.

**Installation:**

```bash
npm install -D vitest @vitest/browser playwright
npx playwright install
```

**vitest.config.ts:**

```ts
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte()],
  test: {
    browser: {
      enabled: true,
      provider: 'playwright',
      name: 'chromium'
    }
  }
});
```

### Option 2: JSDOM with @testing-library/svelte

**Installation:**

```bash
npm install -D vitest @testing-library/svelte jsdom @sveltejs/vite-plugin-svelte
```

**vitest.config.ts:**

```ts
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte({ hot: !process.env.VITEST })],
  test: {
    environment: 'jsdom',
    include: ['src/**/*.{test,spec}.{js,ts}'],
    globals: true
  }
});
```

## Writing Tests

### Basic Component Test

```ts
// Counter.test.ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('increments count when clicked', async () => {
  render(Counter);

  const button = screen.getByRole('button');
  expect(button).toHaveTextContent('Count: 0');

  await fireEvent.click(button);
  expect(button).toHaveTextContent('Count: 1');
});
```

### Testing Props

```ts
// Greeting.test.ts
import { render, screen } from '@testing-library/svelte';
import { expect, test } from 'vitest';
import Greeting from './Greeting.svelte';

test('renders with custom name', () => {
  render(Greeting, { props: { name: 'Alice' } });

  expect(screen.getByText('Hello, Alice!')).toBeInTheDocument();
});

test('uses default name when not provided', () => {
  render(Greeting, { props: { name: 'World' } });

  expect(screen.getByText('Hello, World!')).toBeInTheDocument();
});
```

### Testing Reactive State

```ts
// Form.test.ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import { expect, test } from 'vitest';
import Form from './Form.svelte';

test('updates input value reactively', async () => {
  render(Form);

  const input = screen.getByRole('textbox');
  await fireEvent.input(input, { target: { value: 'test' } });

  expect(input).toHaveValue('test');
  expect(screen.getByTestId('preview')).toHaveTextContent('test');
});
```

### Testing Events/Callbacks

```ts
// Button.test.ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import { expect, test, vi } from 'vitest';
import Button from './Button.svelte';

test('calls onclick callback when clicked', async () => {
  const handleClick = vi.fn();

  render(Button, {
    props: {
      onclick: handleClick,
      children: createRawSnippet(() => ({
        render: () => `<span>Click me</span>`
      }))
    }
  });

  await fireEvent.click(screen.getByRole('button'));

  expect(handleClick).toHaveBeenCalledTimes(1);
});
```

### Testing Async Components

```ts
// AsyncData.test.ts
import { render, screen, waitFor } from '@testing-library/svelte';
import { expect, test, vi } from 'vitest';
import UserProfile from './UserProfile.svelte';

test('loads and displays user data', async () => {
  global.fetch = vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ name: 'Alice', email: 'alice@example.com' })
  });

  render(UserProfile, { props: { userId: '123' } });

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });
});
```

### Testing with Context

```ts
// ThemedComponent.test.ts
import { render, screen } from '@testing-library/svelte';
import { expect, test } from 'vitest';
import ThemeProvider from './ThemeProvider.svelte';
import ThemedButton from './ThemedButton.svelte';

test('uses theme from context', () => {
  render(ThemeProvider, {
    props: {
      theme: 'dark',
      children: createRawSnippet(() => ({
        render: () => `<ThemedButton>Click</ThemedButton>`
      }))
    }
  });

  expect(screen.getByRole('button')).toHaveClass('dark');
});
```

### Testing Snippets

```ts
// Card.test.ts
import { render, screen } from '@testing-library/svelte';
import { createRawSnippet } from 'svelte';
import { expect, test } from 'vitest';
import Card from './Card.svelte';

test('renders children snippet', () => {
  const children = createRawSnippet(() => ({
    render: () => `<p>Card content</p>`
  }));

  render(Card, { props: { children } });

  expect(screen.getByText('Card content')).toBeInTheDocument();
});
```

## Testing Utilities

### Custom Render

```ts
// test-utils.ts
import { render } from '@testing-library/svelte';
import type { ComponentProps, SvelteComponent } from 'svelte';

export function renderWithProviders<T extends SvelteComponent>(
  Component: new (...args: any[]) => T,
  props?: Partial<ComponentProps<T>>
) {
  return render(Component, {
    props: props as ComponentProps<T>
  });
}
```

### Testing Helpers

```ts
// testing.ts
import { tick } from 'svelte';

export async function waitForReactivity() {
  await tick();
}
```

## Reference

- [Svelte Testing Documentation](https://svelte.dev/docs/svelte/testing)
- [Testing Library Svelte](https://testing-library.com/docs/svelte-testing-library/intro/)
- [Vitest Documentation](https://vitest.dev/)
