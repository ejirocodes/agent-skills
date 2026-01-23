---
name: svelte5-best-practices
description: Svelte 5 runes, snippets, SvelteKit patterns, and modern best practices for TypeScript and component development. This skill should be used when writing, reviewing, or refactoring Svelte 5 components and SvelteKit applications to ensure correct usage of runes ($state, $derived, $effect, $props, $bindable), snippets ({#snippet}, {@render}), event handling patterns, and SvelteKit data loading. Triggers on tasks involving Svelte components, state management, props handling, slots migration, Svelte 4 to Svelte 5 migration, or SvelteKit load functions and form actions.
license: MIT
metadata:
  author: ejirocodes
  version: "1.0.0"
---

## Capability Rules

| Rule | Keywords | Description |
|------|----------|-------------|
| [state-rune-declaration](rules/state-rune-declaration.md) | $state, reactive, let, declaration, runes | Declare reactive state with $state rune |
| [derived-rune-syntax](rules/derived-rune-syntax.md) | $derived, computed, reactive, expression | Compute derived values reactively |
| [effect-rune-cleanup](rules/effect-rune-cleanup.md) | $effect, cleanup, side effect, lifecycle | Handle side effects with proper cleanup |
| [props-rune-destructuring](rules/props-rune-destructuring.md) | $props, component props, export let | Receive component props with $props |
| [bindable-props-pattern](rules/bindable-props-pattern.md) | $bindable, two-way binding, bind: | Enable two-way binding on props |
| [snippet-replacing-slots](rules/snippet-replacing-slots.md) | snippet, slot, children, composition | Replace slots with snippets |
| [render-tag-usage](rules/render-tag-usage.md) | @render, snippet, children, optional | Render snippets and children |
| [event-handler-syntax](rules/event-handler-syntax.md) | onclick, on:click, event handler | Use new event handler syntax |
| [callback-props-pattern](rules/callback-props-pattern.md) | callback, createEventDispatcher, events | Replace event dispatching with callbacks |
| [context-api-timing](rules/context-api-timing.md) | setContext, getContext, timing, lifecycle | Correctly use context API |
| [typescript-props-typing](rules/typescript-props-typing.md) | TypeScript, Props, interface, $props | Type component props correctly |
| [typescript-generic-components](rules/typescript-generic-components.md) | generics, generic component, TypeScript | Create generic typed components |
| [migration-reactive-to-state](rules/migration-reactive-to-state.md) | migration, $:, reactive statement | Migrate from Svelte 4 reactive statements |
| [migration-store-to-rune](rules/migration-store-to-rune.md) | migration, store, writable, $store | Migrate from Svelte stores to runes |
| [inspect-rune-debugging](rules/inspect-rune-debugging.md) | $inspect, debugging, console.log | Debug reactive state with $inspect |
| [universal-vs-server-load](rules/universal-vs-server-load.md) | +page.js, +page.server.ts, load, secrets | Choose correct load function type |
| [sveltekit-page-props-typing](rules/sveltekit-page-props-typing.md) | PageProps, PageData, $props, SvelteKit | Type page data with $props |
| [form-actions-error-handling](rules/form-actions-error-handling.md) | fail, form actions, validation, error | Handle form action errors correctly |
| [ssr-state-isolation](rules/ssr-state-isolation.md) | SSR, state leak, shared state, server | Prevent cross-request state leaks |

## Efficiency Rules

| Rule | Keywords | Description |
|------|----------|-------------|
| [universal-reactivity](rules/universal-reactivity.md) | .svelte.js, .svelte.ts, shared state | Use runes outside components |
| [component-testing-vitest](rules/component-testing-vitest.md) | vitest, testing, browser mode | Test Svelte 5 components |
| [avoid-over-reactivity](rules/avoid-over-reactivity.md) | performance, $effect, $derived | Avoid unnecessary reactivity |
| [avoid-load-waterfalls](rules/avoid-load-waterfalls.md) | waterfall, parallel, load, performance | Prevent sequential API calls |
| [streaming-defer-optimization](rules/streaming-defer-optimization.md) | streaming, defer, promise, SSR | Stream non-critical data |

## Reference

- [Svelte 5 Documentation](https://svelte.dev/docs/svelte)
- [Svelte 5 Runes](https://svelte.dev/docs/svelte/$state)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
- [SvelteKit Documentation](https://svelte.dev/docs/kit)
- [SvelteKit Load Functions](https://svelte.dev/docs/kit/load)
- [SvelteKit Form Actions](https://svelte.dev/docs/kit/form-actions)
