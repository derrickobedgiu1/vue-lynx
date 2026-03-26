# Vue Lynx Testing

`vue-lynx-testing-library` provides `render`, `fireEvent`, and `waitForUpdate` for testing Vue Lynx components under a simulated dual-thread environment.

## Table of Contents

1. [Setup](#setup)
2. [Writing tests](#writing-tests)
3. [Firing events](#firing-events)
4. [Testing reactivity](#testing-reactivity)
5. [More patterns](#more-patterns)
6. [API reference](#api-reference)

## Setup

Projects created with `create-vue-lynx` come pre-configured — just run `pnpm test`.

### Adding to an existing project

Install dependencies:

```bash
pnpm add -D vitest jsdom @lynx-js/testing-environment @testing-library/dom @testing-library/jest-dom vue-lynx-testing-library
```

Configure vitest with the required aliases (vue-lynx's sub-packages resolve to different bundles):

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import path from 'node:path'

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: [path.resolve(__dirname, 'test/setup.ts')],
    include: ['test/**/*.test.ts'],
    alias: [
      { find: 'vue-lynx/entry-background', replacement: path.resolve(__dirname, 'node_modules/vue-lynx/runtime/dist/entry-background.js') },
      { find: 'vue-lynx/main-thread', replacement: path.resolve(__dirname, 'node_modules/vue-lynx/main-thread/dist/entry-main.js') },
      { find: 'vue-lynx/internal/ops', replacement: path.resolve(__dirname, 'node_modules/vue-lynx/internal/dist/ops.js') },
      { find: /^vue-lynx$/, replacement: path.resolve(__dirname, 'node_modules/vue-lynx/runtime/dist/index.js') },
    ],
  },
})
```

The setup file initializes Lynx's dual-thread simulation using JSDOM, wiring both the main thread and background thread globals so the ops pipeline works end-to-end in tests:

```ts
// test/setup.ts
import { JSDOM } from 'jsdom'
import { LynxTestingEnv } from '@lynx-js/testing-environment'

const jsdom = new JSDOM('<!DOCTYPE html><html><body></body></html>')
const lynxTestingEnv = new LynxTestingEnv(jsdom)
;(globalThis as any).lynxTestingEnv = lynxTestingEnv

// Wire Main Thread globals
lynxTestingEnv.switchToMainThread()
if (typeof (globalThis as any).registerWorkletInternal === 'undefined') {
  ;(globalThis as any).registerWorkletInternal = () => {}
}
await import('vue-lynx/main-thread')

const mainThreadFns = {
  renderPage: (globalThis as any).renderPage,
  vuePatchUpdate: (globalThis as any).vuePatchUpdate,
  processData: (globalThis as any).processData,
  updatePage: (globalThis as any).updatePage,
  updateGlobalProps: (globalThis as any).updateGlobalProps,
}
Object.assign(lynxTestingEnv.mainThread.globalThis as any, mainThreadFns)

// Wire Background Thread globals
lynxTestingEnv.switchToBackgroundThread()
await import('vue-lynx/entry-background')

const publishEventFn = (globalThis as any).publishEvent
;(lynxTestingEnv.backgroundThread.globalThis as any).publishEvent = publishEventFn

// Re-wire after env resets between tests
;(globalThis as any).onSwitchedToMainThread = () => { Object.assign(globalThis, mainThreadFns) }
;(globalThis as any).onSwitchedToBackgroundThread = () => {
  if ((globalThis as any).lynxCoreInject?.tt) {
    ;(globalThis as any).lynxCoreInject.tt.publishEvent = publishEventFn
  }
  ;(globalThis as any).publishEvent = publishEventFn
}
```

## Writing tests

`render()` returns a `container` (the JSDOM root with rendered Lynx elements) and `@testing-library/dom` query helpers:

```ts
import { expect, it } from 'vitest'
import { h, defineComponent } from 'vue-lynx'
import { render } from 'vue-lynx-testing-library'

it('renders a component', () => {
  const Comp = defineComponent({
    render() {
      return h('view', { id: 'wrapper' }, [h('text', null, 'Hello Vue Lynx')])
    },
  })

  const { container, getByText } = render(Comp)
  expect(container.querySelector('#wrapper')).not.toBeNull()
  expect(getByText('Hello Vue Lynx')).not.toBeNull()
})
```

Components with props can be rerendered:

```ts
const { container, rerender } = render(Greeting, { message: 'hi' })
expect(container.querySelector('text')!.textContent).toBe('hi')

const result = rerender(Greeting, { message: 'hey' })
expect(result.container.querySelector('text')!.textContent).toBe('hey')
```

When passing slots with `h()`, wrap children in a function:

```ts
h(Button, { onClick }, () => [h('text', null, 'Click me')])
```

## Firing events

`fireEvent` dispatches Lynx events through the PAPI event system:

```ts
import { render, fireEvent } from 'vue-lynx-testing-library'

fireEvent.tap(container.querySelector('view')!)
```

The event type is determined by the handler property on the element:

| Binding in `h()` | How to fire | Propagation |
|---|---|---|
| `bindtap: handler` | `fireEvent.tap(el)` | Bubbles |
| `catchtap: handler` | `fireEvent.tap(el, { eventType: 'catchEvent' })` | Stops |
| `capture-bindtap: handler` | `fireEvent.tap(el, { eventType: 'capture-bind' })` | Capture + bubbles |

Available helpers: `fireEvent.tap`, `.longtap`, `.longpress`, `.touchstart`, `.touchmove`, `.touchend`, `.touchcancel`, `.scroll`, `.scrollend`, `.focus`, `.blur`, `.layoutchange`, `.transitionend`, `.animationend`.

## Testing reactivity

After changing reactive state, use `waitForUpdate()` — it waits for both Vue's scheduler flush and the ops pipeline to apply changes to JSDOM:

```ts
import { h, defineComponent, ref } from 'vue-lynx'
import { render, waitForUpdate } from 'vue-lynx-testing-library'

it('updates on ref change', async () => {
  const count = ref(0)
  const Comp = defineComponent({
    setup: () => () => h('text', null, `Count: ${count.value}`),
  })

  const { container } = render(Comp)
  expect(container.querySelector('text')!.textContent).toBe('Count: 0')

  count.value = 42
  await waitForUpdate()
  expect(container.querySelector('text')!.textContent).toBe('Count: 42')
})
```

## More patterns

### Lists

```ts
const items = ref([0, 1, 2])
const Comp = defineComponent({
  setup: () => () =>
    h('list', null, items.value.map((i) =>
      h('list-item', { key: i, 'item-key': i }, [h('text', null, `${i}`)])
    )),
})

const { container } = render(Comp)
items.value = [0, 1, 2, 3]
await waitForUpdate()
expect(container.querySelector('list')).not.toBeNull()
```

### Main Thread Script components

MTS worklets are transformed at build time. In tests, pass the worklet context object directly:

```ts
h('view', {
  'main-thread-bindtap': { _wkltId: 1, _closure: {} },
}, [h('text', null, 'MTS component')])
```

### Template refs

Vue Lynx sets `vue-ref-{id}` attributes on rendered elements. `ShadowElement` provides `invoke`, `setNativeProps`, and `animate` methods.

## API reference

| Export | Purpose |
|---|---|
| `render(Component, props?)` | Renders component, returns `{ container, getByText, rerender, ... }` |
| `fireEvent.tap(el, opts?)` | Fires a tap event |
| `fireEvent(el, event)` | Fires a custom Event |
| `waitForUpdate()` | Waits for Vue scheduler flush + cross-thread ops pipeline |
| `cleanup()` | Unmounts rendered components (auto-called between tests) |
