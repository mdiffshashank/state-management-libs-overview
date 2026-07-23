# React State Management Guide

This site is a learning map for five popular React state management choices: Zustand, Recoil, Redux, Redux Toolkit, and Jotai. Each guide explains how the library works internally, how to set it up, which APIs matter, when to choose it, and which patterns keep the architecture healthy.

Use this home page as the starting point. Pick a library below, read its mental model first, then move through setup, examples, APIs, comparisons, tips, and architecture pitfalls.

## Library Guides

| Library          | Guide                                             | Best For                                                | Mental Model                                                |
| ---------------- | ------------------------------------------------- | ------------------------------------------------------- | ----------------------------------------------------------- |
| 🐻 Zustand       | [Open Zustand Guide](docs/zustand.md)             | Simple shared client state with minimal boilerplate     | External store plus hook selectors                          |
| ⚛️ Recoil        | [Open Recoil Guide](docs/recoil.md)               | Learning or maintaining atom graph state                | Atoms and selectors in a dependency graph                   |
| 🔁 Redux         | [Open Redux Guide](docs/redux.md)                 | Understanding classic Redux internals                   | Single store, actions, dispatch, pure reducers              |
| 🧰 Redux Toolkit | [Open Redux Toolkit Guide](docs/redux-toolkit.md) | Production Redux apps with modern defaults              | Redux architecture with slices, Immer, and built-in tooling |
| ⚙️ Jotai         | [Open Jotai Guide](docs/jotai.md)                 | Composable atomic state with fine-grained subscriptions | Small atoms composed into derived state                     |

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

## Recommended Reading Order

1. [Redux](docs/redux.md) to understand the classic single-store architecture.
2. [Redux Toolkit](docs/redux-toolkit.md) to learn the modern recommended Redux workflow.
3. [Zustand](docs/zustand.md) to compare Redux with a smaller store-based approach.
4. [Jotai](docs/jotai.md) to learn atomic state composition.
5. [Recoil](docs/recoil.md) to compare another atom graph model and understand its tradeoffs.

## Official Documentation

- [Zustand documentation](https://zustand.docs.pmnd.rs/)
- [Recoil documentation](https://recoiljs.org/docs/introduction/getting-started)
- [Redux documentation](https://redux.js.org/)
- [Redux Toolkit documentation](https://redux-toolkit.js.org/)
- [Jotai documentation](https://jotai.org/docs)
