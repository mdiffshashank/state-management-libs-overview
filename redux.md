# Redux

Redux is a predictable state container based on a single store, plain action objects, and pure reducer functions. It teaches a strict architecture: state is read-only, updates happen by dispatching actions, and reducers calculate the next state.

Official source anchors to read later:

- https://redux.js.org/tutorials/fundamentals/part-2-concepts-data-flow
- https://redux.js.org/api/store
- https://redux.js.org/usage/structuring-reducers/normalizing-state-shape

Modern note: the Redux team recommends Redux Toolkit for real applications. Learn classic Redux to understand the architecture, then use Redux Toolkit for day-to-day Redux code.

## Mental Model

Redux is event-driven state management.

- The store contains the state tree.
- Actions describe what happened.
- Dispatch sends an action to the store.
- Reducers receive `(state, action)` and return the next state.
- Subscribers are notified after the state changes.
- UI reads state and dispatches actions.

The core loop is one-way data flow:

1. State renders UI.
2. User or system event happens.
3. Code dispatches an action.
4. Reducers calculate new state.
5. Store notifies subscribers.
6. UI re-renders from new state.

## Internal Architecture: How It Works

A Redux store is a small object with methods such as `getState`, `dispatch`, `subscribe`, and `replaceReducer`.

The store owns one root reducer. During setup, the store calls the root reducer to get initial state. During updates, `dispatch(action)` calls the root reducer again with the current state and action. The returned value becomes the next state.

Reducers must be pure:

- They calculate output from input.
- They do not mutate existing state.
- They do not perform side effects.
- They do not dispatch actions.
- They do not call random/time/network APIs.

Middleware sits between dispatching an action and the reducer receiving it. Middleware can log actions, handle async thunks, report errors, or transform dispatch behavior.

Enhancers wrap the store itself. Middleware is commonly installed through the `applyMiddleware` enhancer.

## Setup

For learning classic Redux:

```bash
npm install redux react-redux
```

For real apps, prefer Redux Toolkit:

```bash
npm install @reduxjs/toolkit react-redux
```

Classic store example:

```tsx
import { createStore } from "redux";

type CounterState = {
  value: number;
};

type CounterAction =
  | { type: "counter/incremented" }
  | { type: "counter/decremented" };

const initialState: CounterState = { value: 0 };

function counterReducer(
  state = initialState,
  action: CounterAction,
): CounterState {
  switch (action.type) {
    case "counter/incremented":
      return { ...state, value: state.value + 1 };
    case "counter/decremented":
      return { ...state, value: state.value - 1 };
    default:
      return state;
  }
}

export const store = createStore(counterReducer);
```

React setup:

```tsx
import { Provider } from "react-redux";
import { store } from "./store";

export function App() {
  return (
    <Provider store={store}>
      <Counter />
    </Provider>
  );
}
```

## Coding Examples

### Dispatch And Select State

```tsx
import { useDispatch, useSelector } from "react-redux";

type RootState = ReturnType<typeof store.getState>;

export function Counter() {
  const count = useSelector((state: RootState) => state.value);
  const dispatch = useDispatch();

  return (
    <section>
      <button onClick={() => dispatch({ type: "counter/decremented" })}>
        -
      </button>
      <span>{count}</span>
      <button onClick={() => dispatch({ type: "counter/incremented" })}>
        +
      </button>
    </section>
  );
}
```

### Combine Reducers

```ts
import { combineReducers, createStore } from "redux";

function todosReducer(state = [], action: any) {
  switch (action.type) {
    case "todos/todoAdded":
      return [...state, action.payload];
    default:
      return state;
  }
}

function filterReducer(state = "all", action: any) {
  switch (action.type) {
    case "filter/changed":
      return action.payload;
    default:
      return state;
  }
}

const rootReducer = combineReducers({
  todos: todosReducer,
  filter: filterReducer,
});

export const store = createStore(rootReducer);
```

### Middleware For Logging

```ts
import { applyMiddleware, createStore } from "redux";

const logger = (storeApi: any) => (next: any) => (action: any) => {
  console.log("dispatching", action);
  const result = next(action);
  console.log("next state", storeApi.getState());
  return result;
};

export const store = createStore(rootReducer, applyMiddleware(logger));
```

### Manual Async With Thunk Middleware

Classic Redux does not include async support by default. Middleware adds it.

```bash
npm install redux-thunk
```

```ts
import { applyMiddleware, createStore } from "redux";
import { thunk } from "redux-thunk";

export const store = createStore(rootReducer, applyMiddleware(thunk));

export const fetchTodos = () => async (dispatch: any) => {
  dispatch({ type: "todos/loading" });

  try {
    const response = await fetch("/api/todos");
    const todos = await response.json();
    dispatch({ type: "todos/loaded", payload: todos });
  } catch (error) {
    dispatch({ type: "todos/failed", error });
  }
};
```

## Important APIs

| API                  | Purpose                                                                         |
| -------------------- | ------------------------------------------------------------------------------- |
| `createStore`        | Creates a Redux store. Legacy for direct app use; RTK wraps this.               |
| `getState`           | Reads the current state tree.                                                   |
| `dispatch`           | Sends an action to the store. This is the only state update path.               |
| `subscribe`          | Registers a listener after dispatches. Usually React bindings use this for you. |
| `replaceReducer`     | Swaps the reducer, often for code splitting or hot reloading.                   |
| `combineReducers`    | Combines slice reducers into one root reducer.                                  |
| `applyMiddleware`    | Installs middleware into the dispatch pipeline.                                 |
| `compose`            | Composes enhancers/functions.                                                   |
| `bindActionCreators` | Wraps action creators with dispatch. Less common with hooks.                    |
| `Provider`           | React-Redux component that makes the store available.                           |
| `useSelector`        | React-Redux hook for reading selected state.                                    |
| `useDispatch`        | React-Redux hook for dispatching actions.                                       |

## How It Differs From Other Libraries

Compared with Redux Toolkit:

- Classic Redux is the lower-level architecture.
- Redux Toolkit is the recommended way to write Redux today.
- RTK removes much boilerplate and adds good defaults.
- Classic Redux reducers require manual immutable updates unless you add helpers.

Compared with Zustand:

- Redux enforces dispatching actions and pure reducers.
- Zustand lets actions directly call `set`.
- Redux has a stronger event log and middleware architecture.
- Zustand is lighter and less opinionated.

Compared with Recoil and Jotai:

- Redux is centralized and event based.
- Recoil and Jotai are atom graph based.
- Redux makes global transitions explicit; atom libraries make dependencies more granular.

## When To Use Redux

Use classic Redux directly when:

- You are learning Redux internals.
- You maintain legacy Redux code.
- You need to understand how Toolkit abstractions map to store, actions, reducers, and middleware.

For new apps, use Redux Toolkit instead when you choose Redux.

Redux architecture is useful when:

- Many parts of the app need the same state.
- State transitions must be traceable.
- Debugging with action history matters.
- You need middleware for cross-cutting workflows.
- A team benefits from strict conventions.

## Tips To Work With Redux

- Keep reducers pure.
- Never mutate state directly in classic reducers.
- Name action types as `domain/eventName`, such as `todos/todoAdded`.
- Keep state normalized for relational data.
- Store minimal source state; derive computed values with selectors.
- Use React-Redux hooks instead of manual `store.subscribe` in components.
- Prefer Redux Toolkit for production code.
- Do not put every local UI value into Redux; keep local-only state in components.

## Patterns To Remember

### Single Source Of Truth

Each piece of global data should have one canonical location in the state tree. Duplicated copies drift out of sync.

### Reducer Composition

Split reducers by state domain, then combine them into a root reducer.

### Normalized State

Store entities by ID instead of deeply nested arrays.

```ts
type UsersState = {
  ids: string[];
  entities: Record<string, { id: string; name: string }>;
};
```

This makes updates easier and avoids copying large nested structures.

### Selectors

Use selectors as the read API for state shape.

```ts
const selectTodos = (state: RootState) => state.todos;
const selectCompletedTodos = (state: RootState) =>
  selectTodos(state).filter((todo) => todo.completed);
```

## What Can Break The Architecture

- Mutating state in reducers breaks change detection and time-travel assumptions.
- Putting side effects in reducers makes updates unpredictable.
- Dispatching inside reducers is forbidden and breaks the update model.
- Duplicating derived data creates synchronization bugs.
- Deeply nested state makes immutable updates fragile.
- Overusing Redux for local form or modal state creates unnecessary ceremony.
- Using broad selectors can cause unnecessary re-renders.
- Inventing inconsistent action names makes debugging harder.

## Quick Decision Checklist

Choose classic Redux when you are studying or maintaining low-level Redux.

Choose Redux Toolkit for new Redux applications.

Choose Zustand for simpler global client state with less ceremony.

Choose Jotai or Recoil for atom graph state where fine-grained dependencies matter more than event logs.
