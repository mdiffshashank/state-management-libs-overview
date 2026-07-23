# Zustand

Zustand is a small state management library for React built around a simple external store and hook-based subscriptions. The mental model is: create a store outside the component tree, expose state and actions from that store, and let components subscribe to exactly the pieces they need.

Official source anchors to read later:

- https://github.com/pmndrs/zustand
- https://zustand.docs.pmnd.rs/

## Mental Model

Think of Zustand as a tiny Flux-like store without Redux ceremony.

- The store lives outside React components.
- The store contains state and actions together.
- Components read from the store through a hook.
- Selectors decide which slice of state a component subscribes to.
- Calling `set` updates the store and notifies subscribers.

Unlike Context, a component does not re-render just because a provider value changed. It re-renders when the selected value from the store changes.

## Internal Architecture: How It Works

At the core, Zustand has a vanilla external store with a small API:

- `getState()` reads the current state.
- `setState()` updates state.
- `subscribe()` registers listeners.
- `getInitialState()` reads the initial state.

The React `create` API wraps that store in a hook. When a component calls `useStore(selector)`, Zustand subscribes the component to the store and compares the selector result after updates.

The update flow is:

1. An action calls `set` or `setState`.
2. Zustand computes the next state.
3. Zustand merges partial objects by default.
4. Subscribers are notified synchronously.
5. Each component selector runs again.
6. Components whose selected value changed re-render.

By default, selector results are compared with strict equality using `Object.is`-style reference checks. If you return a new object or array from a selector every time, the component can re-render even when the underlying values are effectively the same. Use separate selectors, `useShallow`, or an equality function when selecting multiple values.

Zustand does not require a provider for the common case. It can also be used without React through `zustand/vanilla` and connected back to React with `useStore`.

## Setup

```bash
npm install zustand
```

Create a store:

```tsx
import { create } from "zustand";

type CounterStore = {
  count: number;
  increment: () => void;
  reset: () => void;
};

export const useCounterStore = create<CounterStore>()((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  reset: () => set({ count: 0 }),
}));
```

Use it in a component:

```tsx
import { useCounterStore } from "./counterStore";

export function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);

  return <button onClick={increment}>Count: {count}</button>;
}
```

## Coding Examples

### Select Multiple Values Safely

Use separate selectors when possible:

```tsx
const count = useCounterStore((state) => state.count);
const increment = useCounterStore((state) => state.increment);
```

If you want an object or array result, use shallow comparison:

```tsx
import { useShallow } from "zustand/react/shallow";

const { count, increment } = useCounterStore(
  useShallow((state) => ({
    count: state.count,
    increment: state.increment,
  })),
);
```

### Async Actions

Zustand does not need special thunk middleware. An action can be async and call `set` when data is ready.

```tsx
type User = { id: string; name: string };

type UserStore = {
  users: User[];
  status: "idle" | "loading" | "success" | "error";
  fetchUsers: () => Promise<void>;
};

export const useUserStore = create<UserStore>()((set) => ({
  users: [],
  status: "idle",
  fetchUsers: async () => {
    set({ status: "loading" });

    try {
      const response = await fetch("/api/users");
      const users = await response.json();
      set({ users, status: "success" });
    } catch {
      set({ status: "error" });
    }
  },
}));
```

### Persistence Middleware

```tsx
import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";

type ThemeStore = {
  theme: "light" | "dark";
  setTheme: (theme: "light" | "dark") => void;
};

export const useThemeStore = create<ThemeStore>()(
  persist(
    (set) => ({
      theme: "light",
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: "theme-storage",
      storage: createJSONStorage(() => localStorage),
    },
  ),
);
```

### Devtools Middleware

```tsx
import { create } from "zustand";
import { devtools } from "zustand/middleware";

type CartStore = {
  items: string[];
  addItem: (id: string) => void;
};

export const useCartStore = create<CartStore>()(
  devtools(
    (set) => ({
      items: [],
      addItem: (id) =>
        set((state) => ({ items: [...state.items, id] }), undefined, {
          type: "cart/addItem",
          id,
        }),
    }),
    { name: "CartStore" },
  ),
);
```

### Vanilla Store Without React

```ts
import { createStore } from "zustand/vanilla";

type AuthStore = {
  token: string | null;
  setToken: (token: string | null) => void;
};

export const authStore = createStore<AuthStore>()((set) => ({
  token: null,
  setToken: (token) => set({ token }),
}));

authStore.subscribe((state) => {
  console.log("auth changed", state.token);
});
```

## Important APIs

| API                     | Purpose                                                                      |
| ----------------------- | ---------------------------------------------------------------------------- |
| `create`                | Creates a React hook bound to a Zustand store.                               |
| `createStore`           | Creates a vanilla store without React.                                       |
| `useStore`              | Connects a vanilla store to React with a selector.                           |
| `set` / `setState`      | Updates store state. Partial objects are merged by default.                  |
| `get` / `getState`      | Reads current store state without subscribing.                               |
| `subscribe`             | Listens to store changes outside React or for transient updates.             |
| `useShallow`            | Prevents re-renders when a selector returns a shallow-equal object or array. |
| `persist`               | Persists selected store state to storage.                                    |
| `devtools`              | Connects store updates to Redux DevTools.                                    |
| `immer`                 | Lets you write nested immutable updates with Immer syntax.                   |
| `subscribeWithSelector` | Adds selector-aware subscriptions outside React.                             |

## How It Differs From Other Libraries

Compared with Redux:

- Zustand has less ceremony: no required actions, reducers, or dispatch.
- State and actions usually live together in the same store.
- It does not enforce a single global store, although you can use one.
- It is less opinionated, which is faster for small apps but gives teams fewer guardrails.

Compared with Redux Toolkit:

- Redux Toolkit is still Redux and gives structured reducers, middleware, DevTools, serializable checks, and large-app conventions.
- Zustand is lighter and more direct, but it does not automatically impose Redux's event log architecture.

Compared with Recoil and Jotai:

- Zustand is store-centric; Recoil and Jotai are atom-centric.
- Zustand selectors subscribe to slices of one or more stores.
- Atom libraries model state as a dependency graph of small units.

Compared with React Context:

- Zustand avoids provider re-render pressure in the common case.
- Context is built into React and works well for dependency injection or low-frequency values.
- Zustand is better for frequently changing shared state.

## When To Use Zustand

Use Zustand when:

- You want shared client state with minimal boilerplate.
- You prefer direct actions over reducer/action-type ceremony.
- State is mostly local to the frontend and not dominated by server cache concerns.
- You need good render performance without many Context providers.
- A small or medium app needs global state but not heavy Redux structure.

Be careful choosing Zustand when:

- Your team needs strict event history, replayable actions, and standardized reducer patterns.
- You have complex server caching requirements; consider RTK Query, TanStack Query, or framework data APIs.
- You need strong architectural constraints across a very large team.

## Tips To Work With Zustand

- Select the smallest state slice a component needs.
- Avoid `const state = useStore()` in components unless the component truly needs every field.
- Keep actions in the store so state transitions are easy to find.
- Use TypeScript store types early; retrofitting store types later can be annoying.
- Use `persist` only for state that should survive reloads.
- Use `partialize` with persistence when only part of the store should be saved.
- Use `devtools` names and action labels for debugging.
- Use vanilla stores when you need dependency injection, per-request stores, or stores outside React.

## Patterns To Remember

### Slice Pattern

For larger stores, split creation into feature slices and combine them.

```ts
import { create } from "zustand";

type BearSlice = {
  bears: number;
  addBear: () => void;
};

type FishSlice = {
  fish: number;
  addFish: () => void;
};

type Store = BearSlice & FishSlice;

const createBearSlice = (set: any): BearSlice => ({
  bears: 0,
  addBear: () => set((state: Store) => ({ bears: state.bears + 1 })),
});

const createFishSlice = (set: any): FishSlice => ({
  fish: 0,
  addFish: () => set((state: Store) => ({ fish: state.fish + 1 })),
});

export const useStore = create<Store>()((...args) => ({
  ...createBearSlice(...args),
  ...createFishSlice(...args),
}));
```

### Action-Based Updates

Prefer named actions over setting state from everywhere.

```ts
const useProfileStore = create<{
  name: string;
  rename: (name: string) => void;
}>()((set) => ({
  name: "",
  rename: (name) => set({ name }),
}));
```

This keeps write logic centralized.

### Derived Values

For cheap derived values, compute them in selectors:

```tsx
const completedCount = useTodoStore(
  (state) => state.todos.filter((todo) => todo.done).length,
);
```

For expensive derived values, memoize at the component level or store normalized data so selectors stay cheap.

## What Can Break The Architecture

- Subscribing to the whole store in many components causes broad re-renders.
- Returning new object or array selectors without shallow comparison causes unnecessary re-renders.
- Mutating nested state directly without Immer can keep the same references and hide changes.
- Using `set({}, true)` replaces the whole store and can delete actions.
- Scattering `setState` calls across the app makes state transitions hard to audit.
- Persisting volatile UI state can restore stale or invalid UI after reload.
- Reading or mutating shared module stores in React Server Components can leak data across users in server environments.
- Creating stores dynamically without stable references can reset state unexpectedly.

## Quick Decision Checklist

Choose Zustand if you want a pragmatic, low-boilerplate external store with hook subscriptions.

Choose Redux Toolkit instead if you need a highly standardized architecture, middleware pipeline, serializable event log, or large-team conventions.

Choose Jotai or Recoil if your state naturally forms many small reactive atoms and derived dependencies.

Choose React Context if you mostly need dependency injection or rarely changing values like theme objects, feature flags, or service instances.
