# TanStack Query

TanStack Query, formerly React Query, is a server state library. It is not a general client state store like Zustand, Redux, Recoil, or Jotai. Its job is to fetch, cache, synchronize, refetch, and update data that lives on a server.

Official source anchors to read later:

- https://tanstack.com/query/latest/docs/framework/react/overview
- https://tanstack.com/query/latest/docs/framework/react/quick-start
- https://tanstack.com/query/latest/docs/framework/react/guides/queries
- https://tanstack.com/query/latest/docs/framework/react/guides/mutations
- https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation

## Mental Model

TanStack Query treats server data as cached asynchronous state.

- A query is a declarative subscription to async data.
- A query key uniquely identifies cached data.
- A query function fetches the data.
- The query cache stores data, status, errors, timestamps, and observers.
- Components subscribe through hooks like `useQuery`.
- Mutations create, update, delete, or otherwise change server data.
- Invalidation marks matching queries stale and usually refetches active ones.

The key distinction: client state is owned by the browser; server state is owned remotely and can become stale without your app knowing.

## Internal Architecture: How It Works

The main pieces are:

- `QueryClient`: owns the query cache, mutation cache, defaults, and imperative cache APIs.
- `QueryClientProvider`: provides the client to React.
- Query cache: stores records by query key hash.
- Query observers: connect React components to cache entries.
- Mutation cache: tracks mutation executions and lifecycle state.
- Managers: focus, online, notify, and timeout managers coordinate refetching and batching.

The query flow is:

1. A component calls `useQuery({ queryKey, queryFn })`.
2. TanStack Query hashes the query key and checks the cache.
3. If cached data exists and is fresh, it returns that data.
4. If data is missing or stale, it runs the query function.
5. The result is stored in the cache.
6. All observers of the same query key receive the same result.
7. Later, invalidation, refocus, reconnect, polling, or remounting can trigger background refetches.

TanStack Query is not a normalized cache. It does not infer entity relationships from response schemas. Instead, it uses query keys, targeted invalidation, background refetching, and manual cache updates when needed.

## Setup

Install the React package:

```bash
npm install @tanstack/react-query
```

Optional Devtools:

```bash
npm install @tanstack/react-query-devtools
```

Create the client at module scope so it is not recreated on every render:

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      retry: 2,
      refetchOnWindowFocus: true,
    },
  },
});

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <TodosPage />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

Explanation:

- `staleTime` controls how long fetched data is considered fresh.
- `retry` controls failed query retries.
- `refetchOnWindowFocus` lets stale active queries refetch when the tab regains focus.
- Devtools show query keys, status, cache data, observers, and refetch behavior.

## Example Project Types

These examples use a small todo API.

```ts
export type Todo = {
  id: string;
  title: string;
  completed: boolean;
};

export type CreateTodoInput = {
  title: string;
};

export type UpdateTodoInput = {
  id: string;
  title?: string;
  completed?: boolean;
};
```

## API Client Layer

Keep fetch details out of components.

```ts
const API_URL = "/api";

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const response = await fetch(`${API_URL}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...init?.headers,
    },
  });

  if (!response.ok) {
    const message = await response.text();
    throw new Error(message || `Request failed with ${response.status}`);
  }

  return response.json() as Promise<T>;
}

export const todoApi = {
  listTodos: () => request<Todo[]>("/todos"),
  getTodo: (id: string) => request<Todo>(`/todos/${id}`),
  createTodo: (input: CreateTodoInput) =>
    request<Todo>("/todos", {
      method: "POST",
      body: JSON.stringify(input),
    }),
  updateTodo: ({ id, ...patch }: UpdateTodoInput) =>
    request<Todo>(`/todos/${id}`, {
      method: "PATCH",
      body: JSON.stringify(patch),
    }),
  deleteTodo: (id: string) =>
    request<{ success: boolean }>(`/todos/${id}`, {
      method: "DELETE",
    }),
};
```

Why this matters:

- Components stay focused on rendering.
- Query functions become small and testable.
- Error handling is consistent.
- You can later swap `fetch` for another HTTP client without rewriting hooks.

## Query Keys Pattern

Query keys are the architecture. Treat them like public API.

```ts
export const todoKeys = {
  all: ["todos"] as const,
  lists: () => [...todoKeys.all, "list"] as const,
  list: (filter: TodoFilter) => [...todoKeys.lists(), filter] as const,
  details: () => [...todoKeys.all, "detail"] as const,
  detail: (id: string) => [...todoKeys.details(), id] as const,
};

export type TodoFilter = {
  status?: "all" | "active" | "completed";
  search?: string;
};
```

Explanation:

- `['todos']` can invalidate everything todo-related.
- `['todos', 'list']` targets list queries.
- `['todos', 'detail', id]` targets one detail query.
- Objects can be part of query keys, but keep them serializable and stable in shape.

## Basic Query

```tsx
import { useQuery } from "@tanstack/react-query";

export function TodosList() {
  const todosQuery = useQuery({
    queryKey: todoKeys.lists(),
    queryFn: todoApi.listTodos,
  });

  if (todosQuery.isPending) {
    return <p>Loading todos...</p>;
  }

  if (todosQuery.isError) {
    return <p>Error: {todosQuery.error.message}</p>;
  }

  return (
    <ul>
      {todosQuery.data.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
}
```

Explanation:

- `isPending` means no successful data is available yet.
- `isError` means the query function threw.
- After pending and error checks, TypeScript can treat `data` as available.
- If another component asks for the same key, the request is shared and the cache result is reused.

## Query With Parameters

```tsx
function TodoDetails({ id }: { id: string }) {
  const todoQuery = useQuery({
    queryKey: todoKeys.detail(id),
    queryFn: () => todoApi.getTodo(id),
    enabled: Boolean(id),
  });

  if (todoQuery.isPending) return <p>Loading todo...</p>;
  if (todoQuery.isError) return <p>{todoQuery.error.message}</p>;

  return <h2>{todoQuery.data.title}</h2>;
}
```

Explanation:

- Include every variable used by the query function in the query key.
- `enabled` prevents a query from running until required inputs exist.
- A different `id` creates a different cache entry.

## Status vs Fetch Status

```tsx
function TodosStatus() {
  const {
    data = [],
    status,
    fetchStatus,
    isFetching,
  } = useQuery({
    queryKey: todoKeys.lists(),
    queryFn: todoApi.listTodos,
  });

  return (
    <section>
      <p>Data status: {status}</p>
      <p>Fetch status: {fetchStatus}</p>
      {isFetching ? <small>Refreshing...</small> : null}
      <p>Total: {data.length}</p>
    </section>
  );
}
```

Explanation:

- `status` describes whether data exists: `pending`, `error`, or `success`.
- `fetchStatus` describes whether the query function is running: `fetching`, `paused`, or `idle`.
- A successful query can still be `isFetching` during a background refetch.

## Mutation With Invalidation

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

export function AddTodoForm() {
  const queryClient = useQueryClient();

  const createTodo = useMutation({
    mutationFn: todoApi.createTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
    },
  });

  function onSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);
    const title = String(formData.get("title") ?? "").trim();

    if (title) {
      createTodo.mutate({ title });
      event.currentTarget.reset();
    }
  }

  return (
    <form onSubmit={onSubmit}>
      <input name="title" placeholder="Todo title" />
      <button disabled={createTodo.isPending}>
        {createTodo.isPending ? "Adding..." : "Add Todo"}
      </button>
      {createTodo.isError ? <p>{createTodo.error.message}</p> : null}
    </form>
  );
}
```

Explanation:

- Mutations do not run automatically; `mutate` triggers them.
- `onSuccess` invalidates todo lists, causing active list queries to refetch.
- This favors correctness over manually patching every affected cache entry.

## Mutation With `mutateAsync`

Use `mutateAsync` when the caller needs `try/catch` control.

```tsx
function CreateTodoButton() {
  const createTodo = useMutation({ mutationFn: todoApi.createTodo });

  async function handleClick() {
    try {
      const created = await createTodo.mutateAsync({ title: "Read docs" });
      console.log("Created todo", created.id);
    } catch (error) {
      console.error("Failed to create todo", error);
    }
  }

  return <button onClick={handleClick}>Create</button>;
}
```

## Optimistic Update

Optimistic updates make the UI update before the server responds. Always plan rollback.

```tsx
export function ToggleTodoButton({ todo }: { todo: Todo }) {
  const queryClient = useQueryClient();

  const updateTodo = useMutation({
    mutationFn: todoApi.updateTodo,
    onMutate: async (variables) => {
      await queryClient.cancelQueries({ queryKey: todoKeys.lists() });
      await queryClient.cancelQueries({
        queryKey: todoKeys.detail(variables.id),
      });

      const previousList = queryClient.getQueryData<Todo[]>(todoKeys.lists());
      const previousDetail = queryClient.getQueryData<Todo>(
        todoKeys.detail(variables.id),
      );

      queryClient.setQueryData<Todo[]>(todoKeys.lists(), (old = []) =>
        old.map((item) =>
          item.id === variables.id ? { ...item, ...variables } : item,
        ),
      );

      queryClient.setQueryData<Todo>(todoKeys.detail(variables.id), (old) =>
        old ? { ...old, ...variables } : old,
      );

      return { previousList, previousDetail };
    },
    onError: (_error, variables, context) => {
      queryClient.setQueryData(todoKeys.lists(), context?.previousList);
      queryClient.setQueryData(
        todoKeys.detail(variables.id),
        context?.previousDetail,
      );
    },
    onSettled: (_data, _error, variables) => {
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
      queryClient.invalidateQueries({
        queryKey: todoKeys.detail(variables.id),
      });
    },
  });

  return (
    <button
      disabled={updateTodo.isPending}
      onClick={() =>
        updateTodo.mutate({ id: todo.id, completed: !todo.completed })
      }
    >
      {todo.completed ? "Mark active" : "Mark done"}
    </button>
  );
}
```

Explanation:

- `cancelQueries` avoids an in-flight refetch overwriting your optimistic result.
- `getQueryData` snapshots old cache data.
- `setQueryData` applies the optimistic change.
- `onError` rolls back.
- `onSettled` refetches so the cache matches the server.

## Select Data

Use `select` to transform cached data for a component.

```tsx
function CompletedTodoCount() {
  const completedCountQuery = useQuery({
    queryKey: todoKeys.lists(),
    queryFn: todoApi.listTodos,
    select: (todos) => todos.filter((todo) => todo.completed).length,
  });

  if (completedCountQuery.isPending) return null;
  if (completedCountQuery.isError) return null;

  return <span>Completed: {completedCountQuery.data}</span>;
}
```

Explanation:

- The cache still stores the original todos array.
- This component receives only the selected value.
- Use `select` for view-specific derived data.

## Paginated Query

```tsx
type Page<T> = {
  items: T[];
  nextPage?: number;
};

async function listTodosPage(page: number): Promise<Page<Todo>> {
  return request<Page<Todo>>(`/todos?page=${page}`);
}

function PaginatedTodos() {
  const [page, setPage] = React.useState(1);

  const todosPage = useQuery({
    queryKey: ["todos", "page", page],
    queryFn: () => listTodosPage(page),
    placeholderData: (previousData) => previousData,
  });

  return (
    <section>
      {todosPage.data?.items.map((todo) => (
        <p key={todo.id}>{todo.title}</p>
      ))}

      <button
        disabled={page === 1}
        onClick={() => setPage((value) => value - 1)}
      >
        Previous
      </button>
      <button
        disabled={!todosPage.data?.nextPage}
        onClick={() => setPage((value) => value + 1)}
      >
        Next
      </button>

      {todosPage.isFetching ? <small>Loading page...</small> : null}
    </section>
  );
}
```

Explanation:

- Each page is a separate cache entry.
- `placeholderData` keeps previous data visible while the next page loads.
- `isFetching` lets you show background loading without blanking the UI.

## Infinite Query

```tsx
import { useInfiniteQuery } from "@tanstack/react-query";

type CursorPage<T> = {
  items: T[];
  nextCursor?: string;
};

async function listTodosByCursor(cursor?: string): Promise<CursorPage<Todo>> {
  const search = cursor ? `?cursor=${cursor}` : "";
  return request<CursorPage<Todo>>(`/todos${search}`);
}

function InfiniteTodos() {
  const todos = useInfiniteQuery({
    queryKey: ["todos", "infinite"],
    queryFn: ({ pageParam }) => listTodosByCursor(pageParam),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  return (
    <section>
      {todos.data?.pages
        .flatMap((page) => page.items)
        .map((todo) => (
          <p key={todo.id}>{todo.title}</p>
        ))}

      <button
        disabled={!todos.hasNextPage || todos.isFetchingNextPage}
        onClick={() => todos.fetchNextPage()}
      >
        {todos.isFetchingNextPage ? "Loading more..." : "Load more"}
      </button>
    </section>
  );
}
```

Explanation:

- Infinite queries store pages together under one query key.
- `pageParam` is separate from the query key.
- `getNextPageParam` tells TanStack Query how to request the next page.

## Prefetching

```tsx
function TodoLink({ id }: { id: string }) {
  const queryClient = useQueryClient();

  return (
    <a
      href={`/todos/${id}`}
      onMouseEnter={() => {
        queryClient.prefetchQuery({
          queryKey: todoKeys.detail(id),
          queryFn: () => todoApi.getTodo(id),
          staleTime: 60_000,
        });
      }}
    >
      Open todo
    </a>
  );
}
```

Explanation:

- Prefetching warms the cache before navigation.
- If the user opens the detail page soon after, it can render from cache immediately.

## Important APIs

| API                     | Purpose                                                              |
| ----------------------- | -------------------------------------------------------------------- |
| `QueryClient`           | Owns query cache, mutation cache, defaults, and cache methods.       |
| `QueryClientProvider`   | Provides a client to React hooks.                                    |
| `useQuery`              | Fetches and subscribes to one query.                                 |
| `useMutation`           | Runs create/update/delete operations and lifecycle callbacks.        |
| `useQueryClient`        | Gives access to cache APIs such as invalidation and `setQueryData`.  |
| `useInfiniteQuery`      | Handles multi-page cached data.                                      |
| `invalidateQueries`     | Marks matching queries stale and refetches active observers.         |
| `setQueryData`          | Manually updates one cache entry.                                    |
| `getQueryData`          | Reads one cache entry synchronously.                                 |
| `prefetchQuery`         | Fetches data before a component needs it.                            |
| `cancelQueries`         | Cancels in-flight matching queries, often before optimistic updates. |
| `dehydrate` / `hydrate` | Supports SSR and persisted cache workflows.                          |

## How It Differs From RTK Query

TanStack Query:

- Is framework/store independent.
- Does not require Redux.
- Uses query keys and query client APIs directly.
- Encourages targeted invalidation and manual cache updates.
- Has a broad ecosystem across React, Vue, Solid, Svelte, Angular, and more.

RTK Query:

- Is built into Redux Toolkit.
- Stores cache state inside Redux.
- Defines endpoints centrally with `createApi`.
- Uses tags for automated invalidation.
- Generates typed React hooks from endpoint names.

Choose TanStack Query if you do not already want Redux, or if you want a dedicated server-state tool that stays separate from client state management.

## Tips To Work With TanStack Query

- Put every query function variable in the query key.
- Use query key factories for consistency.
- Tune `staleTime` before disabling refetch behavior.
- Prefer invalidation for server truth; use `setQueryData` when you know the exact local patch.
- Use `isPending` for first-load UI and `isFetching` for background refresh UI.
- Keep query functions pure: they should fetch and return data, not update React state.
- Do not create `QueryClient` inside a frequently re-rendering component.
- Use Devtools early; cache behavior becomes much easier to see.

## Patterns To Remember

### Server State Is Not Client State

Do not copy query data into Zustand, Redux, or component state just to render it. Let TanStack Query own server data.

### Query Key Factory

Keep query keys centralized so invalidation stays predictable.

### Invalidate After Mutation

After a mutation, invalidate the smallest query group that is definitely stale.

### Optimistic Update With Rollback

Snapshot before local patching, roll back on error, refetch on settle.

## What Can Break The Architecture

- Missing variables in query keys causes wrong cache reuse.
- Recreating `QueryClient` on render wipes cache and causes repeated fetching.
- Copying server data into separate global stores creates stale duplicate state.
- Overusing `setQueryData` for complex relationship updates recreates normalized-cache problems manually.
- Setting `staleTime: Infinity` everywhere can hide server changes.
- Invalidating overly broad keys can create unnecessary network traffic.
- Returning unstable objects from `select` can reduce render optimization benefits.
- Ignoring errors in query functions prevents hooks from entering error states correctly.

## Quick Decision Checklist

Choose TanStack Query when the main problem is server data: fetching, caching, background refetching, pagination, optimistic updates, retries, and stale data.

Use Zustand, Jotai, or Redux Toolkit alongside it for client-only state such as UI preferences, form drafts, selected tabs, local workflows, and app state that is not owned by the server.
