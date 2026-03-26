# vue-lynx

An agent skill for building native [Lynx](https://vue.lynxjs.org) applications with Vue 3 using the [vue-lynx](https://www.npmjs.com/package/vue-lynx) custom renderer.

## Install

```bash
npx skills add derrickobedgiu1/vue-lynx
```

## What it teaches your AI agent

This skill gives Claude (or any compatible agent) the knowledge to correctly build Vue Lynx apps, covering the key differences from standard Vue web development:

- **Lynx-native elements** — `<view>`, `<text>`, `<image>`, `<scroll-view>`, `<list>` instead of HTML
- **Lynx event system** — `:bindtap`, `:catchtap`, `:capture-bindtap` instead of `@click`
- **Dual-thread architecture** — Vue runs on the background thread, native rendering on the main thread, with ops/events flowing between them
- **Main Thread Script (MTS)** — Eliminating cross-thread latency for smooth animations and gesture handling using `'main thread'` functions, `useMainThreadRef`, and `runOnMainThread`/`runOnBackground`
- **Ecosystem integration** — Vue Router (with `createMemoryHistory`), Pinia, TanStack Vue Query, and Tailwind CSS (with `@lynx-js/tailwind-preset`)
- **Testing** — Setting up `vue-lynx-testing-library` with vitest and the dual-thread test environment
- **Project setup** — `lynx.config.ts`, `pluginVueLynx` options, entry points, type declarations

## Structure

```
vue-lynx/
├── SKILL.md                          # Core skill (always loaded)
└── references/
    ├── main-thread-script.md         # MTS performance patterns
    ├── ecosystem.md                  # Router, Pinia, Vue Query, Tailwind
    └── testing.md                    # Test setup and patterns
```

The main `SKILL.md` covers everything needed for typical Vue Lynx development. The reference files are loaded on-demand when the task requires deeper guidance on a specific topic.

## Links

- [Vue Lynx documentation](https://vue.lynxjs.org)
- [Lynx](https://lynxjs.org)
- [vue-lynx on npm](https://www.npmjs.com/package/vue-lynx)
- [vue-lynx source](https://github.com/user/vue-lynx)

## License

MIT
