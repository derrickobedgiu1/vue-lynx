# Vue Lynx Ecosystem Integration

## Table of Contents

1. [Vue Router](#vue-router)
2. [Pinia](#pinia)
3. [TanStack Vue Query](#tanstack-vue-query)
4. [Tailwind CSS](#tailwind-css)

---

## Vue Router

Vue Router works in Vue Lynx with one key change: use `createMemoryHistory()` instead of `createWebHistory()`. Lynx has no browser History API, no `window.location`, no URL bar — routing state lives entirely in memory.

### Setup

```bash
npm install vue-router
```

```ts
// src/router.ts
import { createRouter, createMemoryHistory } from 'vue-router'
import Home from './views/Home.vue'
import About from './views/About.vue'

const router = createRouter({
  history: createMemoryHistory(),
  routes: [
    { path: '/', name: 'home', component: Home },
    { path: '/about', name: 'about', component: About },
    { path: '/users/:id', name: 'user-detail', component: () => import('./views/UserDetail.vue') },
  ],
})
export default router
```

```ts
// src/index.ts
import { createApp } from 'vue-lynx'
import router from './router'
import App from './App.vue'

const app = createApp(App)
app.use(router)
app.mount()
```

### Navigation

Lynx has no `<a>` element, so `<RouterLink>` can't render its default anchor tag. Two options:

**RouterLink with `custom` prop** — preserves `isActive` reactivity:

```vue
<RouterLink :to="to" custom v-slot="{ navigate, isActive }">
  <text
    :bindtap="navigate"
    :style="{ color: isActive ? '#fff' : '#333', backgroundColor: isActive ? '#1a73e8' : '#e8e8e8' }"
  >
    {{ label }}
  </text>
</RouterLink>
```

**Programmatic navigation:**

```vue
<script setup lang="ts">
import { useRouter } from 'vue-router'
const router = useRouter()
</script>

<template>
  <view v-for="user in users" :key="user.id" :bindtap="() => router.push(`/users/${user.id}`)">
    <text>{{ user.name }}</text>
  </view>
</template>
```

`router.back()` and `router.replace()` work as normal.

---

## Pinia

Pinia works out of the box with no Lynx-specific changes.

### Setup

```bash
npm install pinia
```

```ts
// src/index.ts
import { createApp } from 'vue-lynx'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())
app.mount()
```

### Define a store (setup syntax)

```ts
// src/stores/counter.ts
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)
  function increment() { count.value++ }
  return { count, doubleCount, increment }
})
```

### Use in a component

```vue
<script setup lang="ts">
import { useCounterStore } from './stores/counter'
const counter = useCounterStore()
</script>

<template>
  <view>
    <text>Count: {{ counter.count }}</text>
    <text :bindtap="counter.increment">+1</text>
  </view>
</template>
```

---

## TanStack Vue Query

Lynx provides the Fetch API globally. Vue Query adds caching, background refetching, reactive query keys, and optimistic mutations.

### Setup

```bash
npm install @tanstack/vue-query
```

```ts
// src/index.ts
import { createApp } from 'vue-lynx'
import { VueQueryPlugin } from '@tanstack/vue-query'
import App from './App.vue'

const app = createApp(App)
app.use(VueQueryPlugin)
app.mount()
```

### Basic query

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'

const { data: users, isLoading, isError } = useQuery({
  queryKey: ['users'],
  queryFn: async () => {
    const res = await fetch('https://jsonplaceholder.typicode.com/users')
    if (!res.ok) throw new Error('Failed to fetch')
    return res.json()
  },
})
</script>

<template>
  <view>
    <text v-if="isLoading">Loading...</text>
    <text v-else-if="isError">Error loading data</text>
    <scroll-view v-else scroll-orientation="vertical">
      <view v-for="user in users" :key="user.id">
        <text>{{ user.name }}</text>
      </view>
    </scroll-view>
  </view>
</template>
```

### Reactive query keys

A standout Vue Query feature: query keys can be `ref` or `computed` values. When they change, the query automatically refetches:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue-lynx'
import { useQuery } from '@tanstack/vue-query'

const selectedUserId = ref<number | null>(null)

const { data: posts } = useQuery({
  queryKey: computed(() => ['users', selectedUserId.value, 'posts']),
  queryFn: async () => {
    const res = await fetch(`https://jsonplaceholder.typicode.com/users/${selectedUserId.value}/posts`)
    return res.json()
  },
  enabled: computed(() => selectedUserId.value !== null),
})
</script>
```

### Simple fetch (without Vue Query)

For one-off requests that don't need caching:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue-lynx'

const users = ref([])
const loading = ref(true)

onMounted(async () => {
  try {
    const res = await fetch('https://jsonplaceholder.typicode.com/users')
    users.value = await res.json()
  } finally {
    loading.value = false
  }
})
</script>
```

---

## Tailwind CSS

Tailwind works with Vue Lynx through `@lynx-js/tailwind-preset`, which replaces core plugins with Lynx-compatible equivalents.

### Setup

```bash
pnpm add -D tailwindcss @lynx-js/tailwind-preset rsbuild-plugin-tailwindcss
```

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss'
import preset from '@lynx-js/tailwind-preset'

const config: Config = {
  content: ['./src/**/*.{vue,js,ts}'],
  presets: [preset],
}
export default config
```

```js
// postcss.config.js
export default { plugins: { tailwindcss: {} } }
```

```ts
// lynx.config.ts
import { defineConfig } from '@lynx-js/rspeedy'
import { pluginTailwindCSS } from 'rsbuild-plugin-tailwindcss'
import { pluginVueLynx } from 'vue-lynx/plugin'

export default defineConfig({
  plugins: [
    pluginVueLynx(),
    pluginTailwindCSS({ config: 'tailwind.config.ts', exclude: [/[\\/]node_modules[\\/]/] }),
  ],
})
```

```css
/* src/App.css */
@tailwind base;
@tailwind utilities;
```

Then use utility classes on Lynx elements:

```vue
<view class="p-4 flex flex-col gap-2">
  <text class="text-blue-500 text-lg font-bold">Hello, Tailwind!</text>
</view>
```

### Design tokens with CSS variables

For a shadcn/ui-style token system, define CSS variables and wire them into Tailwind:

```css
:root {
  --color-background: rgba(9, 9, 11, 1);
  --color-primary: rgba(255, 100, 72, 1);
  --color-card: rgba(24, 24, 27, 1);
}
```

```ts
// tailwind.config.ts — extend theme
theme: {
  extend: {
    colors: {
      background: 'var(--color-background)',
      primary: { DEFAULT: 'var(--color-primary)' },
      card: { DEFAULT: 'var(--color-card)' },
    },
  },
},
```

CSS variables require two Lynx engine flags in `lynx.config.ts`:

```ts
pluginVueLynx({
  enableCSSInheritance: true,       // variables cascade parent → children
  enableCSSInlineVariables: true,   // --* properties work in :style bindings
})
```

### Runtime theme switching

Override variables via `:style` on a root element:

```vue
<script setup>
import { ref, computed } from 'vue-lynx'

const themes = {
  dark: { '--color-background': 'rgba(9,9,11,1)', '--color-primary': 'rgba(255,100,72,1)' },
  light: { '--color-background': 'rgba(255,255,255,1)', '--color-primary': 'rgba(234,88,12,1)' },
}
const current = ref('dark')
const themeStyle = computed(() => themes[current.value])
</script>

<template>
  <scroll-view :style="themeStyle" class="bg-background">
    <text class="text-primary">Themed</text>
    <view :bindtap="() => current = current === 'dark' ? 'light' : 'dark'">
      <text>Toggle</text>
    </view>
  </scroll-view>
</template>
```
