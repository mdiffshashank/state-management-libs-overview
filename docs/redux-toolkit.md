# Redux Toolkit

Redux Toolkit, often called RTK, is the official recommended way to write Redux logic. It keeps Redux's architecture: store, actions, reducers, middleware, and one-way data flow, but replaces most hand-written boilerplate with focused APIs and strong defaults.

Official source anchors to read later:

- https://redux-toolkit.js.org/introduction/getting-started
- https://redux-toolkit.js.org/api/configureStore
- https://redux-toolkit.js.org/api/createSlice
- https://redux-toolkit.js.org/rtk-query/overview

## Mental Model

Redux Toolkit is modern Redux with batteries included.

- `configureStore` creates the Redux store with good defaults.
- `createSlice` creates reducer logic and action creators together.
- Immer lets reducers use mutation-looking syntax safely.
- Middleware is configured by default, including thunk support.
- Redux DevTools are enabled by default in development.
- RTK Query handles server data fetching and caching when you need it.

You still dispatch actions. You still read state with selectors. You still have a single Redux store. RTK just makes the correct path shorter.

## Internal Architecture: How It Works

RTK wraps Redux core APIs.

`configureStore` internally creates a Redux store. When you pass an object of slice reducers, it combines them into a root reducer. It also adds default middleware, development checks, thunk support, and DevTools integration.

`createSlice` generates:

- A slice reducer.
- Action type strings.
- Action creator functions.
- Case reducer wiring.

Inside `createSlice`, reducer functions receive a draft state powered by Immer. You can write code like `state.value += 1`; Immer records the draft mutation and produces an immutable next state.

The update flow is the normal Redux flow:

1. UI dispatches an action creator, such as `dispatch(increment())`.
2. Middleware receives the action first.
3. The root reducer delegates to slice reducers.
4. Slice reducers calculate next state, often through Immer.
5. The store saves next state.
6. React-Redux subscribers check selected values and re-render as needed.

RTK Query adds an API cache slice plus middleware. It stores request status, cached data, subscriptions, invalidation tags, and refetch behavior in Redux.

## Setup

For an existing React app:

```bash
npm install @reduxjs/toolkit react-redux
```

Create a store:

```ts
import { configureStore } from "@reduxjs/toolkit";
import counterReducer from "./counterSlice";

export const store = configureStore({
  reducer: {
    counter: counterReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

Connect React:

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

### Slice

```ts
import { createSlice, type PayloadAction } from "@reduxjs/toolkit";

type CounterState = {
  value: number;
};

const initialState: CounterState = { value: 0 };

const counterSlice = createSlice({
  name: "counter",
  initialState,
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

Use the slice in React:

```tsx
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";
import { decrement, increment } from "./counterSlice";

export function Counter() {
  const count = useSelector((state: RootState) => state.counter.value);
  const dispatch = useDispatch<AppDispatch>();

  return (
    <section>
      <button onClick={() => dispatch(decrement())}>-</button>
      <span>{count}</span>
      <button onClick={() => dispatch(increment())}>+</button>
    </section>
  );
}
```

### Typed Hooks

Create app-specific hooks once.

```ts
import {
  useDispatch,
  useSelector,
  type TypedUseSelectorHook,
} from "react-redux";
import type { AppDispatch, RootState } from "./store";

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

Then use them everywhere:

```tsx
const count = useAppSelector((state) => state.counter.value);
const dispatch = useAppDispatch();
```

### Async Thunk

```ts
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

type User = { id: string; name: string };

export const fetchUsers = createAsyncThunk("users/fetchUsers", async () => {
  const response = await fetch("/api/users");
  return (await response.json()) as User[];
});

const usersSlice = createSlice({
  name: "users",
  initialState: {
    entities: [] as User[],
    status: "idle" as "idle" | "loading" | "success" | "error",
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.status = "loading";
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = "success";
        state.entities = action.payload;
      })
      .addCase(fetchUsers.rejected, (state) => {
        state.status = "error";
      });
  },
});

export default usersSlice.reducer;
```

### Entity Adapter

Entity adapters help normalize CRUD state.

```ts
import { createEntityAdapter, createSlice } from "@reduxjs/toolkit";

type Book = { id: string; title: string };

const booksAdapter = createEntityAdapter<Book>();

const booksSlice = createSlice({
  name: "books",
  initialState: booksAdapter.getInitialState(),
  reducers: {
    bookAdded: booksAdapter.addOne,
    booksReceived: booksAdapter.setAll,
    bookRemoved: booksAdapter.removeOne,
  },
});

export const { bookAdded, booksReceived, bookRemoved } = booksSlice.actions;
export default booksSlice.reducer;
```

### RTK Query

```ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

type Post = { id: string; title: string };

export const postsApi = createApi({
  reducerPath: "postsApi",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  tagTypes: ["Post"],
  endpoints: (builder) => ({
    getPosts: builder.query<Post[], void>({
      query: () => "/posts",
      providesTags: ["Post"],
    }),
    addPost: builder.mutation<Post, Partial<Post>>({
      query: (body) => ({
        url: "/posts",
        method: "POST",
        body,
      }),
      invalidatesTags: ["Post"],
    }),
  }),
});

export const { useGetPostsQuery, useAddPostMutation } = postsApi;
```

Add it to the store:

```ts
export const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),
});
```

## Important APIs

| API                        | Purpose                                                                  |
| -------------------------- | ------------------------------------------------------------------------ |
| `configureStore`           | Creates a Redux store with reducers, middleware, DevTools, and checks.   |
| `createSlice`              | Generates action creators and a reducer from slice case reducers.        |
| `createReducer`            | Creates reducers with builder syntax and Immer.                          |
| `createAction`             | Creates an action creator for a type string.                             |
| `createAsyncThunk`         | Generates pending, fulfilled, and rejected async action lifecycle types. |
| `createEntityAdapter`      | Provides normalized CRUD reducers and selectors.                         |
| `createSelector`           | Memoizes derived selectors, re-exported from Reselect.                   |
| `combineSlices`            | Combines slices and supports lazy-loaded slice injection.                |
| `createListenerMiddleware` | Runs effects in response to actions or state changes.                    |
| `createApi`                | Defines RTK Query API endpoints.                                         |
| `fetchBaseQuery`           | Small fetch wrapper for RTK Query base requests.                         |

## How It Differs From Other Libraries

Compared with classic Redux:

- RTK is Redux with official abstractions and defaults.
- It removes hand-written action types and action creators.
- It uses Immer to simplify immutable updates.
- It includes thunk and development checks by default.

Compared with Zustand:

- RTK has more structure and more boilerplate than Zustand.
- RTK has a standardized event pipeline and middleware model.
- Zustand is faster to set up for simple client state.

Compared with Recoil and Jotai:

- RTK is centralized and action based.
- Atom libraries are dependency graph based.
- RTK Query gives RTK a strong built-in server cache story.

## When To Use Redux Toolkit

Use Redux Toolkit when:

- You want Redux's predictability with modern ergonomics.
- You have complex global state shared across many routes or features.
- State transitions need to be traceable in DevTools.
- You need middleware for async workflows, analytics, persistence, or orchestration.
- Your team benefits from consistent conventions.
- You want RTK Query for server data caching.

Avoid RTK when:

- The app has only simple local state.
- A server-state library already solves most data needs and little client global state remains.
- You want a tiny unopinionated store and do not need Redux architecture.

## Tips To Work With Redux Toolkit

- Use `configureStore`; do not manually wire Redux store setup.
- Create one slice per feature domain, not one slice per component.
- Export action creators from slices.
- Keep reducers focused on state transitions, not side effects.
- Use `createAsyncThunk` for simple request lifecycles.
- Use listener middleware for reactive workflows that do not belong in reducers.
- Use RTK Query for server cache instead of hand-writing loading/error/data slices for every endpoint.
- Use entity adapters for normalized collections.
- Create typed hooks once and use them throughout the app.

## Patterns To Remember

### Feature Slice Pattern

Put related state, reducers, and selectors together by feature.

```ts
export const selectCounterValue = (state: RootState) => state.counter.value;
```

### Builder Callback For Extra Reducers

Use builder syntax for thunks and external actions because it gives better TypeScript inference.

### Source State Plus Selectors

Keep minimal state in slices and derive computed data with selectors.

### API Slice Pattern

With RTK Query, usually define one API slice per base URL, then inject endpoints when features are split.

## What Can Break The Architecture

- Mutating values outside Immer reducers can still break immutability.
- Putting non-serializable values in actions or state can break DevTools, persistence, and checks.
- Doing side effects inside reducers breaks Redux's pure update model.
- Creating slices around UI components instead of business domains fragments the state model.
- Duplicating server cache manually while also using RTK Query creates conflicting sources of truth.
- Adding the same middleware twice, especially RTK Query middleware, can cause bugs and warnings.
- Replacing default middleware/enhancers without understanding them can remove thunk or safety checks.

## Quick Decision Checklist

Choose Redux Toolkit for serious Redux usage, large shared client state, traceable workflows, and RTK Query.

Choose Zustand for simple, direct shared client state.

Choose Jotai or Recoil when atom-level dependency composition is the main design need.

Choose classic Redux only for learning internals or maintaining legacy code.
