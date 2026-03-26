# Main Thread Script (MTS)

MTS lets you run event handlers directly on the main thread, eliminating the cross-thread round trip that causes visible lag during high-frequency interactions like scrolling and dragging.

## Table of Contents

1. [Why MTS exists](#why-mts-exists)
2. [The three pieces](#the-three-pieces)
3. [Variable capture](#variable-capture)
4. [Maintaining state](#maintaining-state)
5. [Cross-thread communication](#cross-thread-communication)
6. [Shared modules](#shared-modules)
7. [Patterns](#patterns)

## Why MTS exists

Without MTS, a scroll event takes two thread crossings: main thread fires the event → background thread runs Vue handler → main thread renders the update. Those crossings introduce unpredictable delay, especially on lower-end devices. MTS keeps the entire cycle on the main thread: event fires → handler runs → render — zero crossings.

## The three pieces

**1. Mark the function as main-thread.** Add `'main thread'` as the first statement:

```ts
const onScroll = (event: { detail?: { scrollTop?: number } }) => {
  'main thread'
  const scrollTop = event.detail?.scrollTop ?? 0
  // runs on main thread — instant response
}
```

**2. Bind with `:main-thread-bindEVENT`.** This tells Lynx to dispatch directly to the main thread handler instead of crossing to the background thread:

```vue
<scroll-view :main-thread-bindscroll="onScroll" />
```

The pattern works with all event types: `:main-thread-bindtap`, `:main-thread-bindtouchstart`, `:main-thread-bindtouchmove`, `:main-thread-bindtouchend`, `:main-thread-bindlayoutchange`, and catch/capture variants.

**3. Access native elements with `useMainThreadRef`.** Since MTS runs on the main thread, you can synchronously read and write native element properties:

```vue
<script setup lang="ts">
import { useMainThreadRef } from 'vue-lynx'

const boxRef = useMainThreadRef(null)

const onScroll = (event: { detail?: { scrollTop?: number } }) => {
  'main thread'
  const scrollTop = event.detail?.scrollTop ?? 0
  const el = (boxRef as unknown as {
    current?: { setStyleProperty?(k: string, v: string): void }
  }).current
  el?.setStyleProperty?.('transform', `translate(0px, ${scrollTop}px)`)
}
</script>

<template>
  <scroll-view :main-thread-bindscroll="onScroll"><!-- content --></scroll-view>
  <view :main-thread-ref="boxRef"><text>Follows scroll</text></view>
</template>
```

Access `.current` only inside main thread functions — it resolves to the native element on that thread.

## Variable capture

Main thread functions can read variables from the surrounding background-thread scope. The values are serialized with `JSON.stringify()` and synced to the main thread on each component re-render.

```ts
const red = 'red'  // captured at render time

const onTap = (event: MainThread.ITouchEvent) => {
  'main thread'
  event.currentTarget.setStyleProperty('background-color', red)  // reads captured value
}
```

**Constraints:**
- Captured values must be JSON-serializable (no functions, no circular refs)
- Updates sync on re-render, not in real time
- You cannot write back to captured variables from within an MTS function — the write won't propagate to the background thread
- MTS functions can call other MTS functions but cannot be nested

## Maintaining state

Since captured variables are read-only, use `MainThreadRef` for persistent state that lives on the main thread between handler calls:

```vue
<script setup lang="ts">
import { useMainThreadRef } from 'vue-lynx'

const countRef = useMainThreadRef(0)

const handleTap = (event: MainThread.ITouchEvent) => {
  'main thread'
  ++countRef.current
  event.currentTarget.setStyleProperty(
    'background-color',
    countRef.current % 2 ? 'blue' : 'green'
  )
}
</script>

<template>
  <view :main-thread-bindtap="handleTap"><text>Tap to toggle</text></view>
</template>
```

## Cross-thread communication

**Background → Main:** Use `runOnMainThread()` to call an MTS function asynchronously and get a promise back:

```ts
import { useMainThreadRef, runOnMainThread } from 'vue-lynx'

const countRef = useMainThreadRef(0)

const addCount = (value: number) => {
  'main thread'
  countRef.current += value
  return countRef.current
}

async function increment() {
  const result = await runOnMainThread(addCount)(1)
  console.log('New count:', result)
}
```

**Main → Background:** Use `runOnBackground()` to call a regular function from within MTS:

```ts
import { ref } from 'vue'
import { runOnBackground } from 'vue-lynx'

const current = ref(0)

const syncToBackground = runOnBackground((value: number) => {
  current.value = value
  return `Updated to ${value}`
})

const onScroll = async (event: { detail: { scrollTop: number } }) => {
  'main thread'
  const index = Math.round(event.detail.scrollTop / 300)
  await syncToBackground(index)
}
```

## Shared modules

By default, MTS functions can only call other MTS functions. To share plain utility code between threads, import with `{ runtime: 'shared' }`:

```ts
// color-utils.ts — no 'main thread' directive needed
export function getNextColor(index: number): string {
  const COLORS = ['#4FC3F7', '#81C784', '#FFB74D', '#E57373']
  return COLORS[index % COLORS.length]!
}
```

```vue
<script setup lang="ts">
import { getNextColor } from './color-utils' with { runtime: 'shared' }

// Works on background thread
const color = computed(() => getNextColor(count.value))

// Also works inside MTS
const onTap = () => {
  'main thread'
  const c = getNextColor(1)
  el?.setStyleProperty?.('background-color', c)
}
</script>
```

**Third-party libraries** can also be imported as shared. The recommended pattern is a wrapper:

```ts
// src/utils/motion.ts
import { animate as _animate } from 'motion-dom' with { runtime: 'shared' }
export function animate(...args) {
  'main thread'
  return _animate(...args)
}
```

**Limitations:**
- Only the directly imported identifier is recognized as shared. Assigning to a new variable (`const alias = func`) loses the shared status because the compiler can't track it.
- Each thread gets its own copy of module-level state. Shared modules solve code reuse, not state sharing.

## Patterns

### Scroll-driven animation

```vue
<script setup lang="ts">
import { useMainThreadRef } from 'vue-lynx'

const indicatorRef = useMainThreadRef(null)

const onScroll = (event: { detail?: { scrollTop?: number } }) => {
  'main thread'
  const y = event.detail?.scrollTop ?? 0
  const el = (indicatorRef as unknown as {
    current?: { setStyleProperty?(k: string, v: string): void }
  }).current
  el?.setStyleProperty?.('transform', `translateY(${y * 0.5}px)`)
  el?.setStyleProperty?.('opacity', `${Math.max(0, 1 - y / 500)}`)
}
</script>

<template>
  <scroll-view scroll-orientation="vertical" :main-thread-bindscroll="onScroll">
    <!-- content -->
  </scroll-view>
  <view :main-thread-ref="indicatorRef" style="position: absolute; top: 0">
    <text>Parallax element</text>
  </view>
</template>
```

### Touch drag

```vue
<script setup lang="ts">
import { useMainThreadRef } from 'vue-lynx'

const dragRef = useMainThreadRef(null)
const startY = useMainThreadRef(0)

const onTouchStart = (event: MainThread.ITouchEvent) => {
  'main thread'
  startY.current = event.touches[0].clientY
}

const onTouchMove = (event: MainThread.ITouchEvent) => {
  'main thread'
  const deltaY = event.touches[0].clientY - startY.current
  const el = (dragRef as unknown as {
    current?: { setStyleProperty?(k: string, v: string): void }
  }).current
  el?.setStyleProperty?.('transform', `translateY(${deltaY}px)`)
}
</script>

<template>
  <view
    :main-thread-ref="dragRef"
    :main-thread-bindtouchstart="onTouchStart"
    :main-thread-bindtouchmove="onTouchMove"
  >
    <text>Drag me</text>
  </view>
</template>
```
