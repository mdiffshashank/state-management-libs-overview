# Jotai

Jotai is an atomic state management library for React. Its core idea is to compose state from small atoms. Primitive atoms hold source values, derived atoms compute values from other atoms, and components subscribe only to the atoms they use.

Official source anchors to read later:

- https://jotai.org/docs
- https://jotai.org/docs/core/atom
- https://jotai.org/docs/core/use-atom
- https://jotai.org/docs/guides/core-internals

## Mental Model

Jotai feels like `useState` made composable and shareable.

- An atom config describes a piece of state.
- The atom config itself does not hold the value.
- The Provider store holds atom values.
- Components use atoms through hooks.
- Derived atoms read other atoms with `get`.
- Writable atoms update atoms with `set`.

Jotai is bottom-up: you create tiny units and compose them into larger behavior.

## Internal Architecture: How It Works

Jotai separates atom definitions from atom values.

An atom created with `atom(...)` is an immutable config object. The actual value lives in a store. A Provider supplies a store to a React subtree. If you do not add a Provider, Jotai uses a default store.

The graph works like this:

- Primitive atoms define source values.
- Read-only derived atoms compute values from `get(otherAtom)`.
- Read-write atoms expose both a computed value and write behavior.
- Write-only atoms are command-like atoms used for actions.
- Jotai tracks dependencies read during atom evaluation.
- When a dependency changes, affected derived atoms and subscribers update.

The update flow is:

1. A component calls the setter from `useAtom` or `useSetAtom`.
2. Jotai invokes the atom's write function.
3. The write function calls `set` on one or more atoms.
4. The store updates atom values.
5. Dependent atoms are invalidated.
6. Components subscribed to changed atoms re-render.

Referential equality matters. If you create an atom inside render without `useMemo` or `useRef`, a new atom config is created every render and can cause loops or state resets.

## Setup

```bash
npm install jotai
```

Create atoms:

```tsx
import { atom } from "jotai";

export const countAtom = atom(0);
export const doubledCountAtom = atom((get) => get(countAtom) * 2);
```

Use them in React:

```tsx
import { useAtom, useAtomValue } from "jotai";
import { countAtom, doubledCountAtom } from "./state";

export function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledCountAtom);

  return (
    <button onClick={() => setCount((value) => value + 1)}>
      Count: {count}, doubled: {doubled}
    </button>
  );
}
```

Provider is optional for a single global default store, but useful for scoped state:

```tsx
import { Provider } from "jotai";

export function App() {
  return (
    <Provider>
      <Counter />
    </Provider>
  );
}
```

## Coding Examples

### Primitive, Read-Only, Write-Only, And Read-Write Atoms

```tsx
import { atom } from "jotai";

export const priceAtom = atom(100);

export const taxAtom = atom(0.18);

export const totalAtom = atom((get) => {
  const price = get(priceAtom);
  const tax = get(taxAtom);
  return price + price * tax;
});

export const discountAtom = atom(null, (get, set, discount: number) => {
  set(priceAtom, get(priceAtom) - discount);
});

export const editableTotalAtom = atom(
  (get) => get(totalAtom),
  (get, set, newTotal: number) => {
    const tax = get(taxAtom);
    set(priceAtom, newTotal / (1 + tax));
  },
);
```

### Use The Most Specific Hook

```tsx
import { useAtomValue, useSetAtom } from "jotai";
import { discountAtom, totalAtom } from "./state";

export function CheckoutSummary() {
  const total = useAtomValue(totalAtom);
  const applyDiscount = useSetAtom(discountAtom);

  return (
    <section>
      <p>Total: {total}</p>
      <button onClick={() => applyDiscount(10)}>Apply discount</button>
    </section>
  );
}
```

### Async Atom

```tsx
import { Suspense } from "react";
import { atom, useAtom, useAtomValue } from "jotai";

const userIdAtom = atom("1");

const userAtom = atom(async (get, { signal }) => {
  const userId = get(userIdAtom);
  const response = await fetch(`/api/users/${userId}`, { signal });
  return response.json();
});

function UserName() {
  const user = useAtomValue(userAtom);
  return <p>{user.name}</p>;
}

function UserControls() {
  const [userId, setUserId] = useAtom(userIdAtom);
  return (
    <button onClick={() => setUserId(String(Number(userId) + 1))}>Next</button>
  );
}

export function UserPanel() {
  return (
    <>
      <UserControls />
      <Suspense fallback={<p>Loading...</p>}>
        <UserName />
      </Suspense>
    </>
  );
}
```

### Persistence With Utility Atom

```tsx
import { atomWithStorage } from "jotai/utils";

export const themeAtom = atomWithStorage<"light" | "dark">("theme", "light");
```

### Atom Family

```tsx
import { atomFamily } from "jotai/utils";
import { atom, useAtom } from "jotai";

const todoAtomFamily = atomFamily((id: string) =>
  atom({ id, text: "", completed: false }),
);

export function TodoItem({ id }: { id: string }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(id));

  return (
    <input
      value={todo.text}
      onChange={(event) => setTodo({ ...todo, text: event.target.value })}
    />
  );
}
```

## Important APIs

| API               | Purpose                                                           |
| ----------------- | ----------------------------------------------------------------- |
| `atom`            | Creates primitive, derived, writable, or write-only atom configs. |
| `useAtom`         | Reads and writes an atom.                                         |
| `useAtomValue`    | Reads an atom without exposing a setter.                          |
| `useSetAtom`      | Gets only the write function, avoiding read subscription.         |
| `Provider`        | Provides a scoped atom store.                                     |
| `createStore`     | Creates a Jotai store outside React.                              |
| `getDefaultStore` | Accesses the default store outside React.                         |
| `atomWithStorage` | Persists atom state to storage.                                   |
| `atomFamily`      | Creates parameterized atom instances.                             |
| `selectAtom`      | Subscribes to a selected part of an atom value.                   |
| `splitAtom`       | Splits array atom items into item atoms.                          |
| `useHydrateAtoms` | Initializes atom values for SSR or scoped providers.              |

## How It Differs From Other Libraries

Compared with Recoil:

- Both are atom based.
- Jotai atoms do not require global string keys.
- Jotai atom configs are plain definitions; values live in stores.
- Jotai generally has a smaller core and more utility/extension packages.

Compared with Zustand:

- Jotai is atom-centric; Zustand is store-centric.
- Jotai shines when state is naturally composed from many small dependencies.
- Zustand can feel simpler when one feature store with actions is enough.

Compared with Redux Toolkit:

- Jotai does not use a central event log, reducers, or middleware pipeline.
- Redux Toolkit gives stronger conventions and DevTools action tracing.
- Jotai gives finer-grained atom composition with less ceremony.

## When To Use Jotai

Use Jotai when:

- State can be modeled as small composable atoms.
- Derived state relationships are important.
- You want fine-grained subscriptions without Redux structure.
- You need scoped state per subtree with Provider.
- You like building state from primitive pieces instead of large stores.

Be cautious when:

- Your team wants all state transitions represented as explicit actions.
- You need a central audit log of changes.
- Your atom graph may become hard to discover without naming and file organization discipline.

## Tips To Work With Jotai

- Define atoms outside render unless you intentionally create them dynamically with a stable memoized reference.
- Use `useAtomValue` for read-only components.
- Use `useSetAtom` for write-only components.
- Keep primitive atoms small and compose derived atoms.
- Use write-only atoms as commands when multiple atom updates belong together.
- Use Provider when you need scoped state instances.
- Add `debugLabel` to important atoms for debugging.
- Wrap async atom readers with Suspense and error boundaries.
- Use utilities like `atomWithStorage`, `splitAtom`, and `atomFamily` instead of hand-rolling common patterns.

## Patterns To Remember

### Derived Atom Pattern

Store source data once and derive the rest.

```ts
const todosAtom = atom<Todo[]>([]);
const completedTodosAtom = atom((get) =>
  get(todosAtom).filter((todo) => todo.completed),
);
```

### Write-Only Action Atom

Centralize multi-atom writes.

```ts
const firstNameAtom = atom("");
const lastNameAtom = atom("");

const resetNameAtom = atom(null, (_get, set) => {
  set(firstNameAtom, "");
  set(lastNameAtom, "");
});
```

### Scoped Provider Pattern

Use Provider to create isolated state for reusable widgets, tests, or per-page instances.

```tsx
<Provider>
  <ReusableEditor />
</Provider>
```

### Stable Dynamic Atom Pattern

If an atom must depend on props, memoize it.

```tsx
import { atom, useAtom } from "jotai";
import { useMemo } from "react";

function Rating({ initialValue }: { initialValue: number }) {
  const ratingAtom = useMemo(() => atom(initialValue), [initialValue]);
  const [rating, setRating] = useAtom(ratingAtom);

  return (
    <button onClick={() => setRating((value) => value + 1)}>{rating}</button>
  );
}
```

## What Can Break The Architecture

- Creating atoms in render without stable memoization can cause infinite loops or reset state.
- Duplicating derived values across primitive atoms creates synchronization bugs.
- Creating too many atoms without file organization makes the graph hard to understand.
- Using `useAtom` everywhere can subscribe components that only need to write.
- Large object atoms can cause broad updates if many components read different fields from the same object.
- Async atoms without Suspense or error handling create fragile loading behavior.
- Forgetting Provider scope can accidentally share state globally when isolated instances were intended.
- Atom families with unbounded keys can grow memory usage if not managed.

## Quick Decision Checklist

Choose Jotai when you want small composable atoms, derived dependencies, and fine-grained subscriptions.

Choose Zustand when a direct store with actions is simpler.

Choose Redux Toolkit when you need explicit actions, middleware, event history, and large-app conventions.

Choose Recoil mainly when maintaining existing Recoil code or studying atom graph architecture.
