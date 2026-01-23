---
title: New Event Handler Syntax
impact: HIGH
impactDescription: use correct event handler syntax in Svelte 5
type: capability
tags: onclick, on:click, event handler, directive, events
---

# New Event Handler Syntax

**Impact: HIGH** - use correct event handler syntax in Svelte 5

Svelte 5 replaces `on:click` directive syntax with standard HTML attribute syntax `onclick`.

## Symptoms

- `on:click` produces deprecation warnings
- Event modifiers like `|preventDefault` don't work
- Event handlers not being called

## Root Cause

Svelte 5 simplified event handling by using standard DOM event attributes instead of custom directives.

## Fix

### Basic Event Handlers

**Incorrect (Svelte 4 pattern):**

```svelte
<button on:click={handleClick}>Click</button>
<input on:input={handleInput} />
<form on:submit={handleSubmit}>...</form>
```

**Correct (Svelte 5 pattern):**

```svelte
<button onclick={handleClick}>Click</button>
<input oninput={handleInput} />
<form onsubmit={handleSubmit}>...</form>
```

### Event Modifiers Migration

Event modifiers no longer exist. Use wrapper functions:

**Incorrect (Svelte 4 modifiers):**

```svelte
<form on:submit|preventDefault={handleSubmit}>...</form>
<button on:click|stopPropagation={handleClick}>...</button>
<button on:click|once={handleClick}>...</button>
```

**Correct (Svelte 5 - wrapper functions):**

```svelte
<script>
  function handleSubmit(event) {
    event.preventDefault();
    // ... handle form
  }

  function handleClick(event) {
    event.stopPropagation();
    // ... handle click
  }

  let clicked = false;
  function handleOnce(event) {
    if (clicked) return;
    clicked = true;
    // ... handle once
  }
</script>

<form onsubmit={handleSubmit}>...</form>
<button onclick={handleClick}>...</button>
<button onclick={handleOnce}>...</button>
```

### Capture, Passive, and NonPassive

These three modifiers still require special handling via `onclickcapture` etc:

```svelte
<!-- Capture phase -->
<div onclickcapture={handleCapture}>
  <button onclick={handleClick}>Click</button>
</div>

<!-- Passive listener (improves scroll performance) -->
<div ontouchstartpassive={handleTouch}>...</div>

<!-- Non-passive (allows preventDefault on touch events) -->
<div ontouchmovenonpassive={(e) => e.preventDefault()}>...</div>
```

### Inline Handlers

```svelte
<button onclick={() => count++}>
  Count: {count}
</button>

<input oninput={(e) => name = e.target.value} />
```

### Event Handler Shorthand

If your handler function name matches the event:

```svelte
<script>
  function onclick(event) {
    console.log('Clicked!', event);
  }
</script>

<button {onclick}>Click</button>
```

### Spreading Event Handlers

Event handlers can be spread like other attributes:

```svelte
<script>
  let handlers = {
    onclick: () => console.log('clicked'),
    onmouseenter: () => console.log('entered'),
    onmouseleave: () => console.log('left')
  };
</script>

<button {...handlers}>Hover or Click</button>
```

### Multiple Handlers for Same Event

In Svelte 5, you can only have one handler per event. Combine logic:

**Incorrect (Svelte 4 allowed multiple):**

```svelte
<button on:click={handler1} on:click={handler2}>...</button>
```

**Correct (Svelte 5 - combine handlers):**

```svelte
<script>
  function handleClick(event) {
    handler1(event);
    handler2(event);
  }
</script>

<button onclick={handleClick}>...</button>
```

### TypeScript Event Typing

```svelte
<script lang="ts">
  function handleClick(event: MouseEvent) {
    console.log(event.clientX, event.clientY);
  }

  function handleInput(event: Event) {
    const target = event.target as HTMLInputElement;
    console.log(target.value);
  }

  function handleSubmit(event: SubmitEvent) {
    event.preventDefault();
    const formData = new FormData(event.currentTarget as HTMLFormElement);
  }
</script>

<button onclick={handleClick}>Click</button>
<input oninput={handleInput} />
<form onsubmit={handleSubmit}>...</form>
```

### Custom Events on Components

For component events, see the callback-props-pattern rule.

## Reference

- [Svelte 5 Event Handlers](https://svelte.dev/docs/svelte/event-handlers)
- [Svelte 5 Migration Guide - Events](https://svelte.dev/docs/svelte/v5-migration-guide#Event-changes)
