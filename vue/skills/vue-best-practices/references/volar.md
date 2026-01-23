# Volar Configuration

## Table of Contents

- [Volar 3.0 Breaking Changes](#volar-30-breaking-changes)
- [vueCompilerOptions Overview](#vuecompileroptions-overview)
- [Strict CSS Modules](#strict-css-modules)
- [Fallthrough Attributes](#fallthrough-attributes)
- [Data Attributes Allowlist](#data-attributes-allowlist)
- [Vue Directive Comments](#vue-directive-comments)
- [Code Actions Performance](#code-actions-performance)

---

## Volar 3.0 Breaking Changes

Volar 3.0 (vue-language-server 3.x) introduced breaking changes to the language server protocol. Editors configured for Volar 2.x will break.

### Symptoms

- `vue_ls doesn't work with ts_ls`
- TypeScript features stop working in Vue files
- No autocomplete, type hints, or error highlighting
- Editor shows "Language server initialization failed"

### Fix by Editor

**VSCode**

Update the "Vue - Official" extension to latest version. It manages the language server automatically.

**NeoVim (nvim-lspconfig)**

Option 1: Use vtsls instead of ts_ls

```lua
-- Replace ts_ls/tsserver with vtsls
require('lspconfig').vtsls.setup({})
require('lspconfig').volar.setup({})
```

Option 2: Downgrade vue-language-server

```bash
npm install -g @vue/language-server@2.1.10
```

**JetBrains IDEs**

Update to latest Vue plugin. If issues persist, disable and re-enable the Vue plugin.

### What Changed in 3.0

| Feature | Volar 2.x | Volar 3.0 |
|---------|-----------|-----------|
| TypeScript integration | ts_ls/tsserver | vtsls recommended (Neovim) |
| Hybrid mode | Optional | Default |

**Reference:** [vuejs/language-tools#5598](https://github.com/vuejs/language-tools/issues/5598)

---

## vueCompilerOptions Overview

| Option | Default | Effect |
|--------|---------|--------|
| `strictTemplates` | `false` | Enables all checkUnknown* options |
| `checkUnknownComponents` | `false` | Error on undefined components |
| `checkUnknownProps` | `false` | Error on undeclared props |
| `checkUnknownEvents` | `false` | Error on undeclared events |
| `checkUnknownDirectives` | `false` | Error on unregistered directives |
| `strictCssModules` | `false` | Check CSS module class names |
| `fallthroughAttributes` | `false` | Enable IDE autocomplete for $attrs |
| `dataAttributes` | `[]` | Allow specific data-* patterns |

---

## Strict CSS Modules

When using CSS modules with `<style module>`, Vue doesn't validate class names by default. Enable `strictCssModules` to catch typos.

### Problem

```vue
<script setup lang="ts">
// No error for typo in class name
</script>

<template>
  <div :class="$style.buttn">Click me</div>
</template>

<style module>
.button {
  background: blue;
}
</style>
```

The typo `buttn` instead of `button` silently fails at runtime.

### Solution

```json
// tsconfig.json or tsconfig.app.json
{
  "vueCompilerOptions": {
    "strictCssModules": true
  }
}
```

### What Gets Checked

| Access | With strictCssModules |
|--------|----------------------|
| `$style.validClass` | OK |
| `$style.typo` | Error: Property 'typo' does not exist |
| `$style['dynamic']` | OK (dynamic access not checked) |

**Limitations:** Only checks static property access. Dynamic access (`$style[variable]`) is not validated.

**Reference:** [Vue Language Tools Wiki - Vue Compiler Options](https://github.com/vuejs/language-tools/wiki/Vue-Compiler-Options)

---

## Fallthrough Attributes

When building component libraries with wrapper components, enable `fallthroughAttributes` to get IDE autocomplete for attributes that will be forwarded to child elements.

### What It Does

```vue
<!-- MyButton.vue - wrapper around native button -->
<template>
  <button v-bind="$attrs"><slot /></button>
</template>
```

### Solution

```json
// tsconfig.json or tsconfig.app.json
{
  "vueCompilerOptions": {
    "fallthroughAttributes": true
  }
}
```

When `fallthroughAttributes: true`:
- Vue Language Server analyzes which element receives `$attrs`
- IDE autocomplete suggests valid attributes for the target element

> **Note:** This primarily enables IDE autocomplete for valid fallthrough attributes. It does NOT reject invalid attributes as type errors.

**Reference:** [Vue Language Tools Wiki - Vue Compiler Options](https://github.com/vuejs/language-tools/wiki/Vue-Compiler-Options)

---

## Data Attributes Allowlist

With `strictTemplates` enabled, `data-*` attributes on components cause type errors. Use the `dataAttributes` option to allow specific patterns.

### Problem

```vue
<template>
  <!-- Error: Property 'data-testid' does not exist on type... -->
  <MyComponent data-testid="submit-button" />
</template>
```

### Solution

```json
// tsconfig.json or tsconfig.app.json
{
  "vueCompilerOptions": {
    "strictTemplates": true,
    "dataAttributes": ["data-*"]
  }
}
```

### Specific Patterns

You can be more selective:

```json
{
  "vueCompilerOptions": {
    "dataAttributes": [
      "data-testid",
      "data-cy",
      "data-test-*"
    ]
  }
}
```

### Common Testing Attributes

| Library | Attribute | Pattern |
|---------|-----------|---------|
| Testing Library | `data-testid` | `"data-testid"` |
| Cypress | `data-cy` | `"data-cy"` |
| Playwright | `data-testid` | `"data-testid"` |
| Generic | All data attributes | `"data-*"` |

**Reference:** [Vue Language Tools Wiki - Vue Compiler Options](https://github.com/vuejs/language-tools/wiki/Vue-Compiler-Options)

---

## Vue Directive Comments

Vue Language Tools supports special directive comments to control type checking behavior in templates.

### @vue-ignore

Suppress type errors for the next line:

```vue
<template>
  <!-- @vue-ignore -->
  <Component :prop="valueWithTypeError" />
</template>
```

### @vue-expect-error

Assert that the next line should have a type error (useful for testing):

```vue
<template>
  <!-- @vue-expect-error -->
  <Component :invalid-prop="value" />
</template>
```

### @vue-skip

Skip type checking for an entire block:

```vue
<template>
  <!-- @vue-skip -->
  <div>
    <!-- Everything in here is not type-checked -->
    <LegacyComponent :any="props" :go="here" />
  </div>
</template>
```

### @vue-generic

Declare template-level generic types:

```vue
<template>
  <!-- @vue-generic {T extends string} -->
  <GenericList :items="items as T[]" />
</template>
```

### Use Cases

- Migrating legacy components with incomplete types
- Working with third-party components that have incorrect type definitions
- Temporarily suppressing errors during refactoring
- Testing that certain patterns produce expected type errors

**Reference:** [Vue Language Tools Wiki - Directive Comments](https://github.com/vuejs/language-tools/wiki/Directive-Comments)

---

## Code Actions Performance

In large Vue projects, saving files can take 30-60+ seconds due to VSCode's code actions triggering expensive TypeScript state synchronization.

### Symptoms

- Save operation takes 30+ seconds
- Editor becomes unresponsive during save
- CPU spikes when saving Vue files

### Solution

**Option 1: Disable code actions (fastest)**

```json
// .vscode/settings.json
{
  "vue.codeActions.enabled": false
}
```

**Option 2: Limit code action time**

```json
// .vscode/settings.json
{
  "vue.codeActions.savingTimeLimit": 1000
}
```

**Option 3: Disable specific code actions**

```json
// .vscode/settings.json
{
  "vue.codeActions.enabled": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": "never"
  }
}
```

### Additional Optimizations

```json
// .vscode/settings.json
{
  "vue.codeActions.enabled": false,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {},
  "[vue]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "Vue.volar"
  }
}
```

VSCode 1.81.0+ includes fixes that reduce save time issues.

**Reference:** [Vue Language Tools Discussion #2740](https://github.com/vuejs/language-tools/discussions/2740)
