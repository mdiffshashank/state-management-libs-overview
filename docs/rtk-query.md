# RTK Query

RTK Query is the data fetching and caching layer included with Redux Toolkit. It is designed for server state: fetching data, caching responses, deduplicating requests, generating React hooks, invalidating stale data after mutations, and keeping Redux DevTools visibility.

Official source anchors to read later:

- https://redux-toolkit.js.org/rtk-query/overview
- https://redux-toolkit.js.org/rtk-query/api/createApi
- https://redux-toolkit.js.org/rtk-query/usage/queries
- https://redux-toolkit.js.org/rtk-query/usage/mutations
- https://redux-toolkit.js.org/rtk-query/usage/automated-refetching

## Mental Model

RTK Query is an API slice plus middleware.

- `createApi` defines endpoints.
- `fetchBaseQuery` is a small `fetch` wrapper for common HTTP APIs.
- Query endpoints fetch and cache server data.
- Mutation endpoints change server data.
- Tags describe which cached data a query provides.
- Mutations invalidate tags so active queries refetch.
- Generated hooks connect React components to the Redux-backed cache.

If Redux Toolkit manages client state with slices, RTK Query manages server state with API endpoints.

## Internal Architecture: How It Works

`createApi` generates an API slice containing:

- A reducer mounted at `api.reducerPath`.
- Middleware that handles request lifecycles, cache cleanup, polling, invalidation, and subscriptions.
- Endpoint definitions.
- Auto-generated React hooks when using `@reduxjs/toolkit/query/react`.
- Utilities for cache updates, prefetching, resets, and endpoint initiation.

The query flow is:

1. A component calls a generated hook such as `useGetPostsQuery()`.
2. RTK Query serializes the endpoint name and query argument into a `queryCacheKey`.
3. If a cache entry exists, that data is served.
4. If the entry is missing or refetching is required, middleware runs the request.
5. The result is stored in Redux under the API slice reducer.
6. The hook subscribes the component to that cache entry.
7. When the last subscriber unmounts, RTK Query keeps unused data for `keepUnusedDataFor` seconds before garbage collection.

The mutation flow is:

1. A component calls a mutation trigger such as `updatePost(payload)`.
2. RTK Query runs the mutation request.
3. Lifecycle callbacks can run optimistic updates or side effects.
4. On completion, `invalidatesTags` marks related cached query data invalid.
5. Active queries that provided those tags refetch automatically.

## Setup

RTK Query is included in Redux Toolkit.

```bash
npm install @reduxjs/toolkit react-redux
```

Create an API slice:

```ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export type Post = {
  id: number;
  title: string;
  body: string;
};

export const postsApi = createApi({
  reducerPath: "postsApi",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  tagTypes: ["Posts"],
  endpoints: (build) => ({
    getPosts: build.query<Post[], void>({
      query: () => "/posts",
      providesTags: [{ type: "Posts", id: "LIST" }],
    }),
  }),
});

export const { useGetPostsQuery } = postsApi;
```

Add the API reducer and middleware to the store:

```ts
import { configureStore } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query";
import { postsApi } from "./postsApi";

export const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),
});

setupListeners(store.dispatch);

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
      <PostsPage />
    </Provider>
  );
}
```

Explanation:

- The reducer stores cache state in Redux.
- The middleware runs requests and lifecycle behavior.
- `setupListeners` enables refetch-on-focus and refetch-on-reconnect behavior when configured.

## Recommended File Structure

```text
src/
  app/
    store.ts
  services/
    api.ts
    postsApi.ts
  features/
    posts/
      PostsPage.tsx
      PostDetail.tsx
      AddPostForm.tsx
```

For most apps, prefer one API slice per base URL, then split endpoints with `injectEndpoints` if files grow. Multiple API slices can work, but automatic tag invalidation does not cross API slice boundaries, and each API slice adds its own middleware.

## Base API Slice

Create one empty API slice for the base URL.

```ts
// services/api.ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";
import type { RootState } from "../app/store";

export const api = createApi({
  reducerPath: "api",
  baseQuery: fetchBaseQuery({
    baseUrl: "/api",
    prepareHeaders: (headers, { getState }) => {
      const token = (getState() as RootState).auth?.token;

      if (token) {
        headers.set("authorization", `Bearer ${token}`);
      }

      return headers;
    },
  }),
  tagTypes: ["Posts", "Users"],
  endpoints: () => ({}),
});
```

Explanation:

- `prepareHeaders` can read Redux state and attach auth headers.
- `tagTypes` declares the possible cache labels endpoints can provide or invalidate.
- Empty endpoints let feature files inject their own endpoints later.

## Inject Endpoints

```ts
// services/postsApi.ts
import { api } from "./api";

export type Post = {
  id: number;
  title: string;
  body: string;
  userId: number;
};

export type CreatePostInput = {
  title: string;
  body: string;
  userId: number;
};

export type UpdatePostInput = Partial<CreatePostInput> & Pick<Post, "id">;

export const postsApi = api.injectEndpoints({
  endpoints: (build) => ({
    getPosts: build.query<Post[], void>({
      query: () => "/posts",
      providesTags: (result) =>
        result
          ? [
              { type: "Posts" as const, id: "LIST" },
              ...result.map((post) => ({
                type: "Posts" as const,
                id: post.id,
              })),
            ]
          : [{ type: "Posts" as const, id: "LIST" }],
    }),
    getPost: build.query<Post, number>({
      query: (id) => `/posts/${id}`,
      providesTags: (_result, _error, id) => [{ type: "Posts", id }],
    }),
    addPost: build.mutation<Post, CreatePostInput>({
      query: (body) => ({
        url: "/posts",
        method: "POST",
        body,
      }),
      invalidatesTags: [{ type: "Posts", id: "LIST" }],
    }),
    updatePost: build.mutation<Post, UpdatePostInput>({
      query: ({ id, ...body }) => ({
        url: `/posts/${id}`,
        method: "PATCH",
        body,
      }),
      invalidatesTags: (_result, _error, { id }) => [{ type: "Posts", id }],
    }),
    deletePost: build.mutation<{ success: boolean }, number>({
      query: (id) => ({
        url: `/posts/${id}`,
        method: "DELETE",
      }),
      invalidatesTags: (_result, _error, id) => [
        { type: "Posts", id },
        { type: "Posts", id: "LIST" },
      ],
    }),
  }),
});

export const {
  useGetPostsQuery,
  useGetPostQuery,
  useAddPostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

Explanation:

- `getPosts` provides a `LIST` tag and individual post ID tags.
- `addPost` invalidates the `LIST`, because a new post may appear in the list.
- `updatePost` invalidates one post ID, so detail queries refetch and list queries refetch only if they provided that ID.
- `deletePost` invalidates both the item and the list.

## Basic Query Component

```tsx
import { useGetPostsQuery } from "../../services/postsApi";

export function PostsPage() {
  const {
    data: posts = [],
    error,
    isLoading,
    isFetching,
    isError,
    refetch,
  } = useGetPostsQuery();

  if (isLoading) return <p>Loading posts...</p>;
  if (isError) return <p>Failed to load posts: {String(error)}</p>;

  return (
    <section>
      <button onClick={refetch} disabled={isFetching}>
        {isFetching ? "Refreshing..." : "Refresh"}
      </button>

      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </section>
  );
}
```

Explanation:

- `isLoading` means the first request has no cached data yet.
- `isFetching` can be true during later background refetches.
- `data: posts = []` gives the component a safe fallback after successful type narrowing is not needed.

## Query With Options

```tsx
function PostDetail({ id }: { id?: number }) {
  const postQuery = useGetPostQuery(id!, {
    skip: !id,
    pollingInterval: 30_000,
    refetchOnMountOrArgChange: 60,
    refetchOnFocus: true,
    refetchOnReconnect: true,
  });

  if (!id) return <p>Select a post.</p>;
  if (postQuery.isLoading) return <p>Loading post...</p>;
  if (postQuery.isError) return <p>Could not load post.</p>;
  if (!postQuery.data) return <p>Post not found.</p>;

  return <article>{postQuery.data.title}</article>;
}
```

Explanation:

- `skip` prevents a request until the ID exists.
- `pollingInterval` refetches on an interval while subscribed.
- Refetch options can be global in `createApi` or local per hook.
- `setupListeners` is required for focus and reconnect refetch behavior.

## Mutation Component

```tsx
import { useAddPostMutation } from "../../services/postsApi";

export function AddPostForm() {
  const [addPost, addPostResult] = useAddPostMutation();

  async function onSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);

    try {
      await addPost({
        title: String(formData.get("title")),
        body: String(formData.get("body")),
        userId: 1,
      }).unwrap();

      event.currentTarget.reset();
    } catch (error) {
      console.error("Could not add post", error);
    }
  }

  return (
    <form onSubmit={onSubmit}>
      <input name="title" placeholder="Title" />
      <textarea name="body" placeholder="Body" />
      <button disabled={addPostResult.isLoading}>
        {addPostResult.isLoading ? "Saving..." : "Save"}
      </button>
      {addPostResult.isError ? <p>Save failed.</p> : null}
    </form>
  );
}
```

Explanation:

- Mutation hooks return `[trigger, result]`.
- Calling the trigger starts the request.
- `.unwrap()` returns the fulfilled payload or throws the rejected error, which fits `try/catch`.
- The mutation invalidates the list tag from the endpoint definition.

## `selectFromResult`

Use `selectFromResult` when a component should re-render only for a selected part of a query result.

```tsx
const emptyPosts: Post[] = [];

function PostTitle({ id }: { id: number }) {
  const { post } = useGetPostsQuery(undefined, {
    selectFromResult: ({ data }) => ({
      post: (data ?? emptyPosts).find((item) => item.id === id),
    }),
  });

  return <span>{post?.title ?? "Missing post"}</span>;
}
```

Explanation:

- The component subscribes to the list query but selects one post.
- It re-renders when the selected post reference changes.
- Keep fallback arrays/objects stable; creating new ones inside `selectFromResult` can defeat the optimization.

## Manual Cache Update

Use manual updates when you know the exact cache change and do not want to wait for refetch.

```ts
updatePost: build.mutation<Post, UpdatePostInput>({
  query: ({ id, ...body }) => ({
    url: `/posts/${id}`,
    method: "PATCH",
    body,
  }),
  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    const patchList = dispatch(
      api.util.updateQueryData("getPosts", undefined, (draft) => {
        const post = draft.find((item) => item.id === id);

        if (post) {
          Object.assign(post, patch);
        }
      }),
    );

    const patchDetail = dispatch(
      api.util.updateQueryData("getPost", id, (draft) => {
        Object.assign(draft, patch);
      }),
    );

    try {
      await queryFulfilled;
    } catch {
      patchList.undo();
      patchDetail.undo();
    }
  },
  invalidatesTags: (_result, _error, { id }) => [{ type: "Posts", id }],
});
```

Explanation:

- `onQueryStarted` runs when the mutation starts.
- `updateQueryData` patches existing cache entries with Immer syntax.
- The returned patch object has `undo()` for rollback.
- Keeping `invalidatesTags` still refetches after the mutation so the server remains the final truth.

## Transform Response

Use `transformResponse` when the server response shape is not the shape your UI should consume.

```ts
type PostsEnvelope = {
  data: Post[];
  meta: { total: number };
};

getPosts: build.query<Post[], void>({
  query: () => "/posts",
  transformResponse: (response: PostsEnvelope) => response.data,
  providesTags: (result) =>
    result
      ? [
          { type: "Posts" as const, id: "LIST" },
          ...result.map((post) => ({ type: "Posts" as const, id: post.id })),
        ]
      : [{ type: "Posts" as const, id: "LIST" }],
});
```

Explanation:

- The hook receives `Post[]`, not the envelope.
- Cache entries store the transformed result.
- Tags should describe the transformed cached data.

## Custom `queryFn`

Use `queryFn` when the endpoint is not a simple single HTTP request.

```ts
getPostWithAuthor: build.query<{ post: Post; author: User }, number>({
  async queryFn(id, _api, _extraOptions, baseQuery) {
    const postResult = await baseQuery(`/posts/${id}`);

    if (postResult.error) {
      return { error: postResult.error };
    }

    const post = postResult.data as Post;
    const userResult = await baseQuery(`/users/${post.userId}`);

    if (userResult.error) {
      return { error: userResult.error };
    }

    return {
      data: {
        post,
        author: userResult.data as User,
      },
    };
  },
});
```

Explanation:

- `queryFn` returns `{ data }` or `{ error }`.
- It can call `baseQuery` one or more times.
- Use this for custom async workflows, not for every normal request.

## Prefetching

```tsx
import { api } from "../../services/api";

function PostLink({ id, title }: { id: number; title: string }) {
  const prefetchPost = api.usePrefetch("getPost");

  return (
    <a href={`/posts/${id}`} onMouseEnter={() => prefetchPost(id)}>
      {title}
    </a>
  );
}
```

Explanation:

- Prefetching starts loading detail data before navigation.
- If the user opens the detail page, the cache may already have the result.

## Important APIs

| API                        | Purpose                                                                         |
| -------------------------- | ------------------------------------------------------------------------------- |
| `createApi`                | Defines an API slice, endpoints, tags, reducer, middleware, and hooks.          |
| `fetchBaseQuery`           | Lightweight fetch wrapper for common HTTP APIs.                                 |
| `build.query`              | Defines a cached read endpoint.                                                 |
| `build.mutation`           | Defines a write endpoint.                                                       |
| `build.infiniteQuery`      | Defines a multi-page cache endpoint.                                            |
| `tagTypes`                 | Declares cache tag names.                                                       |
| `providesTags`             | Labels cached query data.                                                       |
| `invalidatesTags`          | Marks related cached data stale after mutations.                                |
| `setupListeners`           | Enables refetch-on-focus and refetch-on-reconnect behavior.                     |
| `selectFromResult`         | Selects a small part of a query result for render optimization.                 |
| `api.util.updateQueryData` | Manually patches an existing cache entry.                                       |
| `api.util.invalidateTags`  | Imperatively invalidates tags.                                                  |
| `injectEndpoints`          | Adds endpoints to an existing API slice for code splitting.                     |
| `.unwrap()`                | Converts mutation/query trigger result promises into payload-or-throw behavior. |

## How It Differs From TanStack Query

RTK Query:

- Requires Redux Toolkit store setup.
- Stores cache state in Redux.
- Defines endpoints centrally.
- Generates hooks from endpoint names.
- Uses tag-based automated invalidation.
- Integrates with Redux DevTools and Redux middleware.

TanStack Query:

- Does not require Redux.
- Uses `QueryClient` and query keys directly.
- Gives more direct cache client control.
- Works across multiple frontend frameworks.
- Uses query invalidation APIs instead of tag definitions.

Choose RTK Query if your app already uses Redux Toolkit or if you want server cache behavior integrated with Redux architecture.

## Tips To Work With RTK Query

- Usually create one API slice per base URL.
- Add the API middleware exactly once.
- Use `LIST` tags for collection-level invalidation.
- Use entity ID tags for detail-level invalidation.
- Use `selectFromResult` for extracting one item from a list query.
- Prefer generated hooks in React components.
- Use `.unwrap()` when a component needs imperative success/error handling.
- Keep endpoint definitions close to the domain they represent.
- Avoid putting RTK Query server data into normal Redux slices unless you are intentionally deriving client state.

## Patterns To Remember

### API Slice Pattern

One API slice owns one base URL and shared invalidation graph.

### Tag Invalidation Pattern

Queries provide tags. Mutations invalidate tags. Active queries refetch when invalidated.

### LIST Tag Pattern

Use `{ type: 'Posts', id: 'LIST' }` to target collection queries separately from individual item queries.

### Manual Patch Pattern

Use `onQueryStarted` plus `api.util.updateQueryData` for optimistic updates or immediate cache edits.

## What Can Break The Architecture

- Creating many API slices for the same base URL prevents tag invalidation across endpoints and adds middleware overhead.
- Forgetting to add `api.middleware` prevents query lifecycle behavior from working correctly.
- Forgetting to mount `api.reducer` means cache state has nowhere to live.
- Invalidating a broad root tag can trigger far more requests than necessary.
- Missing `LIST` tags makes collection invalidation awkward.
- Duplicating RTK Query data in normal Redux slices creates stale sources of truth.
- Returning new arrays or objects from `selectFromResult` without stable references can cause avoidable re-renders.
- Using mutations for reads or queries for writes makes cache behavior harder to reason about.

## Quick Decision Checklist

Choose RTK Query when you already use Redux Toolkit and want data fetching, caching, invalidation, generated hooks, and DevTools visibility in the same architecture.

Choose TanStack Query when you want a dedicated server-state library without adopting Redux.

Use normal Redux slices for client state, not for duplicating data RTK Query already caches.
