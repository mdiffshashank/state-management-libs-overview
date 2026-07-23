# React State Management Guide

This site is a learning map for popular React state management and server-state choices: Zustand, Recoil, Redux, Redux Toolkit, Jotai, TanStack Query, and RTK Query. Each guide explains how the library works internally, how to set it up, which APIs matter, when to choose it, and which patterns keep the architecture healthy.

Use this home page as the starting point. Pick a library below, read its mental model first, then move through setup, examples, APIs, comparisons, tips, and architecture pitfalls.

## Library Guides

| Library           | Guide                                               | Best For                                                       | Mental Model                                                |
| ----------------- | --------------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| 🐻 Zustand        | [Open Zustand Guide](docs/zustand.md)               | Simple shared client state with minimal boilerplate            | External store plus hook selectors                          |
| ⚛️ Recoil         | [Open Recoil Guide](docs/recoil.md)                 | Learning or maintaining atom graph state                       | Atoms and selectors in a dependency graph                   |
| 🔁 Redux          | [Open Redux Guide](docs/redux.md)                   | Understanding classic Redux internals                          | Single store, actions, dispatch, pure reducers              |
| 🧰 Redux Toolkit  | [Open Redux Toolkit Guide](docs/redux-toolkit.md)   | Production Redux apps with modern defaults                     | Redux architecture with slices, Immer, and built-in tooling |
| ⚙️ Jotai          | [Open Jotai Guide](docs/jotai.md)                   | Composable atomic state with fine-grained subscriptions        | Small atoms composed into derived state                     |
| 🔎 TanStack Query | [Open TanStack Query Guide](docs/tanstack-query.md) | Server-state fetching, caching, synchronization, and mutations | Query client, query keys, cache observers, and invalidation |
| 🧰 RTK Query      | [Open RTK Query Guide](docs/rtk-query.md)           | Server-state caching inside Redux Toolkit                      | API slice, generated hooks, Redux cache, and tags           |

## What You Will Learn

- How each library stores state and notifies React components.
- How updates flow through stores, atoms, selectors, reducers, or middleware.
- How to install and use each library with practical React examples.
- Which APIs are important enough to remember.
- How each library differs from the others.
- When a library is a good fit, and when it is unnecessary.
- Patterns that scale well as an app grows.
- Mistakes that can break the architecture or cause performance problems.

## Quick Choosing Guide

Choose **Zustand** when you want a small external store with direct actions and low ceremony.

Choose **Redux Toolkit** when you need Redux's predictability, DevTools workflow, middleware, strong team conventions, or RTK Query.

Choose **Redux** when you want to understand the lower-level architecture behind Redux Toolkit or maintain older Redux code.

Choose **Jotai** when your state is naturally built from many small atoms and derived dependencies.

Choose **Recoil** when you are learning atom graph architecture or maintaining an existing Recoil app, while keeping its ecosystem and maintenance caveats in mind.

Choose **TanStack Query** when your main problem is server state: fetching, caching, retries, pagination, background refetching, and optimistic updates without adopting Redux.

Choose **RTK Query** when you already use Redux Toolkit and want server-state caching, generated hooks, tag invalidation, and Redux DevTools visibility in the same architecture.

## Recommended Reading Order

1. [Redux](docs/redux.md) to understand the classic single-store architecture.
2. [Redux Toolkit](docs/redux-toolkit.md) to learn the modern recommended Redux workflow.
3. [Zustand](docs/zustand.md) to compare Redux with a smaller store-based approach.
4. [Jotai](docs/jotai.md) to learn atomic state composition.
5. [Recoil](docs/recoil.md) to compare another atom graph model and understand its tradeoffs.
6. [TanStack Query](docs/tanstack-query.md) to learn server-state caching without Redux.
7. [RTK Query](docs/rtk-query.md) to learn server-state caching inside Redux Toolkit.

## Official Documentation

- [Zustand documentation](https://zustand.docs.pmnd.rs/)
- [Recoil documentation](https://recoiljs.org/docs/introduction/getting-started)
- [Redux documentation](https://redux.js.org/)
- [Redux Toolkit documentation](https://redux-toolkit.js.org/)
- [Jotai documentation](https://jotai.org/docs)
- [TanStack Query documentation](https://tanstack.com/query/latest/docs/framework/react/overview)
- [RTK Query documentation](https://redux-toolkit.js.org/rtk-query/overview)
