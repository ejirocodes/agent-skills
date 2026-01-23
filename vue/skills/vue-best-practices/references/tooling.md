# Tooling Configuration

## Table of Contents

- [moduleResolution Bundler Migration](#moduleresolution-bundler-migration)
- [HMR Debugging for SSR](#hmr-debugging-for-ssr)
- [Duplicate Plugin Detection](#duplicate-plugin-detection)

---

## moduleResolution Bundler Migration

Recent versions of `@vue/tsconfig` changed `moduleResolution` from `"node"` to `"bundler"`. This can break existing projects.

### Symptoms

- `Cannot find module 'vue'` or other packages
- `Option '--resolveJsonModule' cannot be specified without 'node' module resolution`
- Errors appear after updating `@vue/tsconfig`
- Some third-party packages no longer resolve

### Root Cause

`moduleResolution: "bundler"` requires:
1. TypeScript 5.0+
2. Packages to have proper `exports` field in package.json
3. Different resolution rules than Node.js classic resolution

### Fix

**Option 1: Ensure TypeScript 5.0+ everywhere**

```bash
npm install -D typescript@^5.0.0
```

In monorepos, ALL packages must use TypeScript 5.0+.

**Option 2: Add compatibility workaround**

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolvePackageJsonExports": false
  }
}
```

Setting `resolvePackageJsonExports: false` restores compatibility with packages that don't have proper exports.

**Option 3: Revert to Node resolution**

```json
{
  "compilerOptions": {
    "moduleResolution": "node"
  }
}
```

### Which Packages Break?

Packages break if they:
- Lack `exports` field in package.json
- Have incorrect `exports` configuration
- Rely on Node.js-specific resolution behavior

### Diagnosis

```bash
# Check which resolution is being used
cat tsconfig.json | grep moduleResolution

# Test if a specific module resolves
npx tsc --traceResolution 2>&1 | grep "module-name"
```

**Reference:** [vuejs/tsconfig#8](https://github.com/vuejs/tsconfig/issues/8)

---

## HMR Debugging for SSR

Hot Module Replacement breaks when modifying Vue component `<script setup>` sections in SSR applications.

### Symptoms

- HMR works for `<template>` changes but breaks for `<script setup>`
- "Cannot read property of undefined" after saving
- Full page reload required after script changes
- HMR works in dev:client but not dev:ssr

### Root Cause

SSR mode has a different transformation pipeline. The Vue plugin's HMR boundary detection doesn't handle SSR modules the same way as client modules.

### Fix

**Step 1: Ensure correct SSR plugin configuration**

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  ssr: {
    // Don't externalize these for HMR to work
    noExternal: ['vue', '@vue/runtime-core', '@vue/runtime-dom']
  }
})
```

**Step 2: Configure dev server for SSR HMR**

```typescript
// server.ts
import { createServer } from 'vite'

const vite = await createServer({
  server: { middlewareMode: true },
  appType: 'custom'
})

// Use vite.ssrLoadModule for server-side imports
const { render } = await vite.ssrLoadModule('/src/entry-server.ts')

// Handle HMR
vite.watcher.on('change', async (file) => {
  if (file.endsWith('.vue')) {
    // Invalidate the module
    const mod = vite.moduleGraph.getModuleById(file)
    if (mod) {
      vite.moduleGraph.invalidateModule(mod)
    }
  }
})
```

**Step 3: Add HMR acceptance in entry-server**

```typescript
// entry-server.ts
import { createApp } from './main'

export async function render(url: string) {
  const app = createApp()
  // ... render logic
}

// Accept HMR updates
if (import.meta.hot) {
  import.meta.hot.accept()
}
```

### Framework-Specific Solutions

**Nuxt 3**

HMR should work out of the box. If not:

```bash
rm -rf .nuxt node_modules/.vite
npm install
npm run dev
```

**Vite SSR Template**

Ensure you're using the latest `@vitejs/plugin-vue`:

```bash
npm install @vitejs/plugin-vue@latest
```

### Debugging

Enable verbose HMR logging:

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    hmr: {
      overlay: true
    }
  },
  logLevel: 'info'  // Shows HMR updates
})
```

### Known Limitations

- HMR for `<script>` (not `<script setup>`) may require full reload
- SSR components with external dependencies may not hot-reload
- State is not preserved for SSR components (expected behavior)

**Reference:** [vite-plugin-vue#525](https://github.com/vitejs/vite-plugin-vue/issues/525)

---

## Duplicate Plugin Detection

When using Vite's JavaScript API, if the Vue plugin is loaded in `vite.config.js` and specified again in `inlineConfig`, it gets registered twice, causing cryptic build errors.

### Symptoms

- Build produces unexpected output or fails silently
- "Cannot read property of undefined" during build
- Different build behavior between CLI and JavaScript API
- Vue components render incorrectly after build

### Root Cause

Vite doesn't deduplicate plugins by name when merging configs. The Vue plugin's internal state gets corrupted when registered twice.

### Fix

**Option 1: Use configFile: false with inline plugins**

```typescript
import { build } from 'vite'
import vue from '@vitejs/plugin-vue'

await build({
  configFile: false,  // Don't load vite.config.js
  plugins: [vue()],
  // ... rest of config
})
```

**Option 2: Don't specify plugins in inlineConfig**

```typescript
// vite.config.js already has vue plugin
import { build } from 'vite'

await build({
  // Don't add vue plugin here - it's in vite.config.js
  root: './src',
  build: { outDir: '../dist' }
})
```

**Option 3: Filter out Vue plugin before merging**

```typescript
import { build, loadConfigFromFile } from 'vite'
import vue from '@vitejs/plugin-vue'

const { config } = await loadConfigFromFile({ command: 'build', mode: 'production' })

// Remove existing Vue plugin
const filteredPlugins = config.plugins?.filter(
  p => !p || (Array.isArray(p) ? false : p.name !== 'vite:vue')
) || []

await build({
  ...config,
  plugins: [...filteredPlugins, vue({ /* your options */ })]
})
```

### Detection Script

Add this to debug plugin registration:

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [
    vue(),
    {
      name: 'debug-plugins',
      configResolved(config) {
        const vuePlugins = config.plugins.filter(p => p.name?.includes('vue'))
        if (vuePlugins.length > 1) {
          console.warn('WARNING: Multiple Vue plugins detected:', vuePlugins.map(p => p.name))
        }
      }
    }
  ]
})
```

### Common Scenarios

| Scenario | Solution |
|----------|----------|
| Using `vite.createServer()` | Use `configFile: false` |
| Build script with custom config | Don't duplicate plugins |
| Monorepo with shared config | Check for plugin inheritance |

**Reference:** [Vite Issue #5335](https://github.com/vitejs/vite/issues/5335)
