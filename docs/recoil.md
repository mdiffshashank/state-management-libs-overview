# Recoil

Recoil is a React state management library based on atoms and selectors. Its core idea is to model shared state as a dependency graph: atoms are writable source state, selectors are derived state, and React components subscribe to the nodes they read.

Official source anchors to read later:

- https://recoiljs.org/docs/introduction/getting-started
- https://recoiljs.org/docs/basic-tutorial/atoms
- https://recoiljs.org/docs/basic-tutorial/selectors

Important practical note: Recoil has not had the same level of active ecosystem momentum as Redux Toolkit, Zustand, or Jotai. Treat it as valuable for learning atom graph architecture, but check current maintenance status and React version compatibility before choosing it for new production work.

## Mental Model

Think of Recoil as React state split into globally addressable atoms.

- An atom is a unit of source state.
- A selector is derived state computed from atoms or other selectors.
- Components subscribe automatically when they read an atom or selector.
- Updating an atom invalidates dependent selectors.
- Recoil re-renders only components that depend on changed graph nodes.

This feels close to `useState`, but the state can be shared anywhere below `RecoilRoot`.

## Internal Architecture: How It Works

Recoil builds a graph of state dependencies.

- Atoms are graph roots that hold writable values.
- Selectors are graph nodes with `get` functions.
- A selector records which atoms/selectors it reads.
- When an atom changes, Recoil invalidates dependent selectors.
- Components using `useRecoilValue`, `useRecoilState`, or related hooks subscribe to graph nodes.

The update flow is:

1. A component writes to an atom with a setter.
2. Recoil stores the new atom value.
3. Selectors depending on that atom are invalidated.
4. Any component reading the changed atom or derived selector is scheduled to re-render.
5. Selectors are recomputed as needed.

Selectors can be synchronous or asynchronous. Async selectors integrate with React Suspense patterns because their `get` function can return a promise.

Each atom and selector must have a stable unique `key`. The key is part of Recoil's identity and is used for debugging, persistence integrations, and internal graph management.

## Setup

```bash
npm install recoil
```

Wrap your app with `RecoilRoot`:

```tsx
import { RecoilRoot } from "recoil";

export function App() {
  return (
    <RecoilRoot>
      <TodoPage />
    </RecoilRoot>
  );
}
```

Create an atom:

```tsx
import { atom } from "recoil";

export const filterState = atom<"all" | "active" | "completed">({
  key: "filterState",
  default: "all",
});
```

Use it in a component:

```tsx
import { useRecoilState } from "recoil";
import { filterState } from "./state";

export function FilterSelect() {
  const [filter, setFilter] = useRecoilState(filterState);

  return (
    <select
      value={filter}
      onChange={(event) => setFilter(event.target.value as typeof filter)}
    >
      <option value="all">All</option>
      <option value="active">Active</option>
      <option value="completed">Completed</option>
    </select>
  );
}
```

## Coding Examples

### Atom And Selector

```tsx
import { atom, selector, useRecoilValue, useSetRecoilState } from "recoil";

type Todo = {
  id: string;
  text: string;
  completed: boolean;
};

export const todosState = atom<Todo[]>({
  key: "todosState",
  default: [],
});

export const completedTodoCountState = selector<number>({
  key: "completedTodoCountState",
  get: ({ get }) => {
    const todos = get(todosState);
    return todos.filter((todo) => todo.completed).length;
  },
});

export function AddTodoButton() {
  const setTodos = useSetRecoilState(todosState);

  return (
    <button
      onClick={() =>
        setTodos((todos) => [
          ...todos,
          { id: crypto.randomUUID(), text: "Learn Recoil", completed: false },
        ])
      }
    >
      Add
    </button>
  );
}

export function TodoSummary() {
  const completedCount = useRecoilValue(completedTodoCountState);
  return <p>Completed: {completedCount}</p>;
}
```

### Writable Selector

A writable selector can centralize derived write logic.

```tsx
import { atom, selector } from "recoil";

export const celsiusState = atom<number>({
  key: "celsiusState",
  default: 0,
});

export const fahrenheitState = selector<number>({
  key: "fahrenheitState",
  get: ({ get }) => get(celsiusState) * 1.8 + 32,
  set: ({ set }, newValue) => {
    if (typeof newValue === "number") {
      set(celsiusState, (newValue - 32) / 1.8);
    }
  },
});
```

### Atom Family

Atom families create parameterized atoms.

```tsx
import { atomFamily, useRecoilState } from "recoil";

export const todoItemState = atomFamily({
  key: "todoItemState",
  default: (id: string) => ({
    id,
    text: "",
    completed: false,
  }),
});

export function TodoItem({ id }: { id: string }) {
  const [todo, setTodo] = useRecoilState(todoItemState(id));

  return (
    <input
      value={todo.text}
      onChange={(event) => setTodo({ ...todo, text: event.target.value })}
    />
  );
}
```

### Async Selector

```tsx
import { atom, selector, useRecoilValue } from "recoil";

const userIdState = atom<string>({
  key: "userIdState",
  default: "1",
});

const userState = selector({
  key: "userState",
  get: async ({ get }) => {
    const userId = get(userIdState);
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  },
});

export function UserName() {
  const user = useRecoilValue(userState);
  return <span>{user.name}</span>;
}
```

Use this with a Suspense boundary around the component.

## Important APIs

| API                   | Purpose                                                        |
| --------------------- | -------------------------------------------------------------- |
| `RecoilRoot`          | Provider that owns the Recoil graph for a React subtree.       |
| `atom`                | Creates a writable source state node.                          |
| `selector`            | Creates derived state, optionally writable.                    |
| `atomFamily`          | Creates parameterized atom instances.                          |
| `selectorFamily`      | Creates parameterized selectors.                               |
| `useRecoilState`      | Reads and writes an atom or writable selector.                 |
| `useRecoilValue`      | Reads an atom or selector.                                     |
| `useSetRecoilState`   | Gets only the setter, avoiding read subscription.              |
| `useResetRecoilState` | Resets state to its default.                                   |
| `useRecoilCallback`   | Reads/writes Recoil state from callbacks with snapshot access. |
| `useRecoilSnapshot`   | Observes a snapshot of the current graph.                      |

## How It Differs From Other Libraries

Compared with Zustand:

- Recoil is atom graph based; Zustand is store based.
- Recoil tracks dependencies between atoms and selectors.
- Zustand is smaller and less provider-driven in common usage.

Compared with Redux and Redux Toolkit:

- Recoil does not use a single reducer pipeline or action log.
- Recoil updates individual graph nodes rather than dispatching actions to a root reducer.
- Redux Toolkit gives stronger conventions, middleware, and better ecosystem momentum.

Compared with Jotai:

- Both are atom based.
- Recoil requires string keys for atoms/selectors.
- Jotai atoms are config objects and generally feel more minimal.
- Jotai has stronger current momentum in the pmndrs ecosystem.

## When To Use Recoil

Use Recoil when:

- You want to understand atom graph state management.
- Your state is naturally split into many independent units.
- Derived state dependencies are more important than centralized events.
- You are maintaining an existing Recoil codebase.

Be cautious using Recoil for new production apps when:

- You need strong current maintenance guarantees.
- You are targeting the newest React ecosystem features.
- Your team already standardizes on Redux Toolkit, Zustand, or Jotai.

## Tips To Work With Recoil

- Keep atom and selector keys unique and stable.
- Use `useSetRecoilState` when a component only writes state.
- Keep atom defaults serializable when you plan to persist or debug state.
- Model derived data as selectors instead of duplicating it in atoms.
- Use atom families for per-entity state, but watch cache growth for unbounded IDs.
- Wrap async selector consumers in Suspense and error boundaries.
- Keep selectors pure; do not mutate external state inside `get`.

## Patterns To Remember

### Source State In Atoms, Derived State In Selectors

Do not store both `todos` and `completedTodoCount` as atoms. Store `todos` and derive the count with a selector.

### Prefer Write-Only Components When Possible

If a button only updates state, use `useSetRecoilState` instead of `useRecoilState`. This avoids subscribing the button to values it does not render.

### Use Families For Dynamic Instances

Use `atomFamily` when each item has its own state instance keyed by an ID. Do not manually generate many unrelated atom definitions for the same pattern.

## What Can Break The Architecture

- Duplicate atom or selector keys can cause runtime errors or state collisions.
- Storing derived data in atoms creates synchronization bugs.
- Writing side effects inside selectors makes recomputation unpredictable.
- Creating unbounded atom families without cleanup strategy can grow memory usage.
- Using async selectors without Suspense or error boundaries can create poor loading/error behavior.
- Treating Recoil like Redux can lead to unnecessary action layers that fight the graph model.
- Choosing Recoil without checking current maintenance status can create upgrade risk.

## Quick Decision Checklist

Choose Recoil if you are learning or maintaining atom graph state with derived dependencies.

Choose Jotai if you want a similar atom mental model with a smaller, more actively used modern API.

Choose Zustand if you want simple store-based state without graph keys.

Choose Redux Toolkit if you need explicit events, reducers, middleware, DevTools, and larger-app conventions.
