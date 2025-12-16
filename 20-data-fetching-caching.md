# Chapter 20: Data Fetching & Caching

## Introduction

Junior developers use useEffect and useState for every API call, creating inconsistent loading states and stale data. Senior developers leverage powerful data fetching libraries like React Query or SWR, implementing sophisticated caching strategies that dramatically improve performance and user experience.

## Learning Objectives

- Master React Query for server state management
- Implement effective caching strategies
- Handle optimistic updates properly
- Build infinite scroll with pagination
- Prefetch data for better UX
- Manage cache invalidation correctly
- Handle background refetching
- Implement offline-first approaches

## 20.1 React Query Fundamentals

### Setup and Configuration

```typescript
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <App />
    <ReactQueryDevtools initialIsOpen={false} />
  </QueryClientProvider>
);
```

### Basic Queries

```typescript
// api/users.ts
import apiClient from '@/services/apiClient';

export interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

export const userApi = {
  getAll: async (): Promise<User[]> => {
    const { data } = await apiClient.get('/users');
    return data;
  },

  getById: async (id: number): Promise<User> => {
    const { data } = await apiClient.get(`/users/${id}`);
    return data;
  },

  create: async (user: Omit<User, 'id'>): Promise<User> => {
    const { data } = await apiClient.post('/users', user);
    return data;
  },

  update: async (id: number, user: Partial<User>): Promise<User> => {
    const { data } = await apiClient.patch(`/users/${id}`, user);
    return data;
  },

  delete: async (id: number): Promise<void> => {
    await apiClient.delete(`/users/${id}`);
  },
};

// hooks/useUsers.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '@/api/users';

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: userApi.getAll,
  });
}

export function useUser(id: number) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => userApi.getById(id),
    enabled: id > 0, // Only fetch if ID is valid
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: userApi.create,
    onSuccess: (newUser) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users'] });

      // Or optimistically add to cache
      queryClient.setQueryData<User[]>(['users'], (old) => {
        return old ? [...old, newUser] : [newUser];
      });
    },
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<User> }) =>
      userApi.update(id, data),
    onSuccess: (updatedUser) => {
      // Update specific user in cache
      queryClient.setQueryData(['users', updatedUser.id], updatedUser);

      // Update user in list
      queryClient.setQueryData<User[]>(['users'], (old) => {
        return old?.map((user) =>
          user.id === updatedUser.id ? updatedUser : user
        );
      });
    },
  });
}

export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: userApi.delete,
    onSuccess: (_, deletedId) => {
      // Remove from cache
      queryClient.setQueryData<User[]>(['users'], (old) => {
        return old?.filter((user) => user.id !== deletedId);
      });
    },
  });
}

// Usage in component
function UserList() {
  const { data: users, isLoading, error } = useUsers();
  const deleteUser = useDeleteUser();

  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;

  return (
    <div>
      {users?.map((user) => (
        <div key={user.id}>
          <span>{user.name}</span>
          <button
            onClick={() => deleteUser.mutate(user.id)}
            disabled={deleteUser.isPending}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

## 20.2 Advanced Query Patterns

### Dependent Queries

```typescript
// Fetch user, then their posts
function UserPosts({ userId }: { userId: number }) {
  // First query
  const {
    data: user,
    isLoading: userLoading,
  } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => userApi.getById(userId),
  });

  // Dependent query - only runs after user is fetched
  const {
    data: posts,
    isLoading: postsLoading,
  } = useQuery({
    queryKey: ['posts', { userId }],
    queryFn: () => postApi.getByUserId(userId),
    enabled: !!user, // Only fetch when user exists
  });

  if (userLoading) return <Spinner />;
  if (postsLoading) return <Spinner />;

  return (
    <div>
      <h2>{user?.name}'s Posts</h2>
      {posts?.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}
```

### Parallel Queries

```typescript
// Fetch multiple resources simultaneously
function Dashboard() {
  const users = useQuery({
    queryKey: ['users'],
    queryFn: userApi.getAll,
  });

  const posts = useQuery({
    queryKey: ['posts'],
    queryFn: postApi.getAll,
  });

  const stats = useQuery({
    queryKey: ['stats'],
    queryFn: statsApi.get,
  });

  // Use useQueries for dynamic parallel queries
  const userIds = [1, 2, 3, 4, 5];
  const userQueries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['users', id],
      queryFn: () => userApi.getById(id),
    })),
  });

  const isLoading = users.isLoading || posts.isLoading || stats.isLoading;
  const isError = users.isError || posts.isError || stats.isError;

  if (isLoading) return <Spinner />;
  if (isError) return <Error />;

  return (
    <div>
      <UserStats users={users.data} />
      <RecentPosts posts={posts.data} />
      <Analytics stats={stats.data} />
    </div>
  );
}
```

### Infinite Queries

```typescript
// api/posts.ts
export const postApi = {
  getPage: async ({ pageParam = 1 }): Promise<{
    posts: Post[];
    nextPage: number | null;
    total: number;
  }> => {
    const { data } = await apiClient.get('/posts', {
      params: {
        page: pageParam,
        limit: 20,
      },
    });
    return data;
  },
};

// hooks/usePosts.ts
export function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) => postApi.getPage({ pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextPage,
    getPreviousPageParam: (firstPage) => firstPage.prevPage,
  });
}

// components/InfinitePostList.tsx
function InfinitePostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfinitePosts();

  const { ref, inView } = useInView();

  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <Spinner />;

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.posts.map((post) => (
            <PostCard key={post.id} post={post} />
          ))}
        </div>
      ))}

      {/* Infinite scroll trigger */}
      <div ref={ref}>
        {isFetchingNextPage ? <Spinner /> : hasNextPage ? 'Load More' : 'No more posts'}
      </div>
    </div>
  );
}
```

## 20.3 Optimistic Updates

### Simple Optimistic Update

```typescript
export function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<Post> }) =>
      postApi.update(id, data),

    // Optimistic update
    onMutate: async ({ id, data }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['posts', id] });

      // Snapshot previous value
      const previousPost = queryClient.getQueryData<Post>(['posts', id]);

      // Optimistically update
      queryClient.setQueryData<Post>(['posts', id], (old) => ({
        ...old!,
        ...data,
      }));

      // Return snapshot for rollback
      return { previousPost };
    },

    // Rollback on error
    onError: (err, { id }, context) => {
      queryClient.setQueryData(['posts', id], context?.previousPost);
    },

    // Refetch on success
    onSettled: (data, error, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['posts', id] });
    },
  });
}
```

### Complex Optimistic Update with List

```typescript
export function useToggleLike() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: number) => postApi.toggleLike(postId),

    onMutate: async (postId) => {
      // Cancel queries
      await queryClient.cancelQueries({ queryKey: ['posts'] });

      // Snapshot
      const previousPosts = queryClient.getQueryData<Post[]>(['posts']);

      // Optimistic update
      queryClient.setQueryData<Post[]>(['posts'], (old) =>
        old?.map((post) =>
          post.id === postId
            ? {
                ...post,
                liked: !post.liked,
                likes: post.liked ? post.likes - 1 : post.likes + 1,
              }
            : post
        )
      );

      return { previousPosts };
    },

    onError: (err, postId, context) => {
      queryClient.setQueryData(['posts'], context?.previousPosts);

      // Show error notification
      toast.error('Failed to like post');
    },

    onSuccess: (data, postId) => {
      // Update with server response
      queryClient.setQueryData<Post[]>(['posts'], (old) =>
        old?.map((post) => (post.id === postId ? data : post))
      );
    },
  });
}
```

## 20.4 Prefetching Data

### Hover Prefetch

```typescript
// hooks/usePrefetchUser.ts
export function usePrefetchUser() {
  const queryClient = useQueryClient();

  return useCallback(
    (userId: number) => {
      queryClient.prefetchQuery({
        queryKey: ['users', userId],
        queryFn: () => userApi.getById(userId),
        staleTime: 5 * 60 * 1000, // 5 minutes
      });
    },
    [queryClient]
  );
}

// components/UserLink.tsx
function UserLink({ userId, name }: { userId: number; name: string }) {
  const prefetchUser = usePrefetchUser();

  return (
    <Link
      to={`/users/${userId}`}
      onMouseEnter={() => prefetchUser(userId)}
    >
      {name}
    </Link>
  );
}
```

### Route-Based Prefetching

```typescript
// routes/loaders.ts
import { queryClient } from '@/lib/queryClient';

export const userLoader = async ({ params }: { params: { id: string } }) => {
  const userId = parseInt(params.id);

  // Prefetch user data before route renders
  await queryClient.prefetchQuery({
    queryKey: ['users', userId],
    queryFn: () => userApi.getById(userId),
  });

  // Prefetch related data
  await queryClient.prefetchQuery({
    queryKey: ['posts', { userId }],
    queryFn: () => postApi.getByUserId(userId),
  });

  return null;
};

// router.tsx
const router = createBrowserRouter([
  {
    path: '/users/:id',
    element: <UserProfile />,
    loader: userLoader,
  },
]);
```

### Prefetch on Scroll

```typescript
function PostList() {
  const { data: posts } = usePosts();
  const queryClient = useQueryClient();

  const prefetchPost = useCallback(
    (postId: number) => {
      queryClient.prefetchQuery({
        queryKey: ['posts', postId],
        queryFn: () => postApi.getById(postId),
      });
    },
    [queryClient]
  );

  return (
    <div>
      {posts?.map((post, index) => {
        // Prefetch next 3 posts
        if (index < posts.length - 3) {
          const nextPost = posts[index + 3];
          prefetchPost(nextPost.id);
        }

        return <PostCard key={post.id} post={post} />;
      })}
    </div>
  );
}
```

## 20.5 Cache Management

### Manual Cache Updates

```typescript
// Update cache after mutation
const createPost = useMutation({
  mutationFn: postApi.create,
  onSuccess: (newPost) => {
    queryClient.setQueryData<Post[]>(['posts'], (old) => {
      return old ? [newPost, ...old] : [newPost];
    });
  },
});

// Remove from cache
const deletePost = useMutation({
  mutationFn: postApi.delete,
  onSuccess: (_, deletedId) => {
    queryClient.removeQueries({ queryKey: ['posts', deletedId] });
  },
});
```

### Smart Cache Invalidation

```typescript
// Invalidate specific queries
queryClient.invalidateQueries({
  queryKey: ['posts'],
  exact: true, // Only invalidate exact match
});

// Invalidate all posts queries
queryClient.invalidateQueries({
  queryKey: ['posts'],
});

// Invalidate with predicate
queryClient.invalidateQueries({
  predicate: (query) =>
    query.queryKey[0] === 'posts' &&
    query.state.data?.userId === currentUserId,
});

// Refetch active queries
queryClient.refetchQueries({
  queryKey: ['posts'],
  type: 'active',
});
```

### Cache Persistence

```typescript
// Install: npm install @tanstack/react-query-persist-client

import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      cacheTime: 1000 * 60 * 60 * 24, // 24 hours
      staleTime: 1000 * 60 * 5, // 5 minutes
    },
  },
});

const persister = createSyncStoragePersister({
  storage: window.localStorage,
  key: 'react-query-cache',
});

function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{ persister }}
    >
      <AppRoutes />
    </PersistQueryClientProvider>
  );
}
```

## 20.6 Background Refetching

### Window Focus Refetch

```typescript
// Automatic refetch on window focus (enabled by default)
const { data } = useQuery({
  queryKey: ['notifications'],
  queryFn: notificationApi.getAll,
  refetchOnWindowFocus: true,
  refetchInterval: false, // Don't poll while visible
});

// Custom refetch logic
const { data, refetch } = useQuery({
  queryKey: ['stats'],
  queryFn: statsApi.get,
  refetchOnWindowFocus: false,
});

useEffect(() => {
  const handleFocus = () => {
    const lastFetch = queryClient.getQueryState(['stats'])?.dataUpdatedAt;
    const now = Date.now();

    // Only refetch if data is older than 5 minutes
    if (lastFetch && now - lastFetch > 5 * 60 * 1000) {
      refetch();
    }
  };

  window.addEventListener('focus', handleFocus);
  return () => window.removeEventListener('focus', handleFocus);
}, [refetch, queryClient]);
```

### Polling / Interval Refetch

```typescript
// Poll every 30 seconds
const { data } = useQuery({
  queryKey: ['liveData'],
  queryFn: api.getLiveData,
  refetchInterval: 30000, // 30 seconds
});

// Conditional polling
const { data } = useQuery({
  queryKey: ['order', orderId],
  queryFn: () => orderApi.getById(orderId),
  refetchInterval: (data) => {
    // Stop polling when order is completed
    return data?.status === 'completed' ? false : 5000;
  },
});

// Poll only when window is focused
const { data } = useQuery({
  queryKey: ['prices'],
  queryFn: priceApi.get,
  refetchInterval: 10000,
  refetchIntervalInBackground: false, // Don't poll when tab is hidden
});
```

## 20.7 Error Handling and Retries

### Custom Retry Logic

```typescript
const { data, error, failureCount } = useQuery({
  queryKey: ['posts'],
  queryFn: postApi.getAll,
  retry: (failureCount, error) => {
    // Don't retry on 404
    if (error.response?.status === 404) {
      return false;
    }

    // Retry up to 3 times
    return failureCount < 3;
  },
  retryDelay: (attemptIndex) => {
    // Exponential backoff: 1s, 2s, 4s
    return Math.min(1000 * 2 ** attemptIndex, 30000);
  },
});
```

### Global Error Handling

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      onError: (error: any) => {
        // Log to monitoring service
        ErrorTracker.captureError(error);

        // Show user-friendly message
        if (error.response?.status === 401) {
          toast.error('Session expired. Please login again.');
        } else if (error.response?.status >= 500) {
          toast.error('Server error. Please try again later.');
        }
      },
    },
    mutations: {
      onError: (error: any) => {
        ErrorTracker.captureError(error);
        toast.error('Something went wrong. Please try again.');
      },
    },
  },
});
```

### Error Boundaries with Queries

```typescript
// Use error boundaries with suspense
function PostsPage() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<Spinner />}>
        <Posts />
      </Suspense>
    </ErrorBoundary>
  );
}

function Posts() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: postApi.getAll,
    suspense: true, // Enable suspense mode
  });

  return <PostList posts={data} />;
}
```

## 20.8 SWR as Alternative

### Basic SWR Usage

```typescript
// Install: npm install swr

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then((res) => res.json());

function Profile() {
  const { data, error, isLoading, mutate } = useSWR('/api/user', fetcher);

  if (isLoading) return <Spinner />;
  if (error) return <Error />;

  return (
    <div>
      <h1>{data.name}</h1>
      <button onClick={() => mutate()}>Refresh</button>
    </div>
  );
}

// With TypeScript
function useUser(id: number) {
  const { data, error, isLoading } = useSWR<User>(
    `/api/users/${id}`,
    fetcher,
    {
      revalidateOnFocus: true,
      dedupingInterval: 5000,
    }
  );

  return {
    user: data,
    isLoading,
    isError: error,
  };
}
```

### SWR Mutations

```typescript
import useSWRMutation from 'swr/mutation';

async function updateUser(url: string, { arg }: { arg: Partial<User> }) {
  return fetch(url, {
    method: 'PATCH',
    body: JSON.stringify(arg),
  }).then((res) => res.json());
}

function EditProfile() {
  const { trigger, isMutating } = useSWRMutation('/api/user', updateUser);

  const handleSave = async () => {
    try {
      const result = await trigger({ name: 'John Doe' });
      toast.success('Profile updated');
    } catch (error) {
      toast.error('Failed to update profile');
    }
  };

  return (
    <button onClick={handleSave} disabled={isMutating}>
      Save
    </button>
  );
}
```

## Real-World Scenario: Building a Social Feed

### The Challenge

Build a performant social media feed:
- Infinite scroll with pagination
- Optimistic likes/comments
- Real-time updates
- Offline support
- Prefetching

### Senior Approach

```typescript
// 1. Infinite scroll feed
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam = 1 }) => feedApi.getPage(pageParam),
  initialPageParam: 1,
  getNextPageParam: (lastPage) => lastPage.nextPage,
  staleTime: 30000, // Fresh for 30 seconds
});

// 2. Optimistic likes
const likeMutation = useMutation({
  mutationFn: postApi.like,
  onMutate: async (postId) => {
    await queryClient.cancelQueries({ queryKey: ['feed'] });
    const previous = queryClient.getQueryData(['feed']);

    queryClient.setQueryData(['feed'], (old) =>
      updateLikeInPages(old, postId)
    );

    return { previous };
  },
  onError: (err, vars, context) => {
    queryClient.setQueryData(['feed'], context.previous);
  },
});

// 3. Prefetch on hover
const prefetchPost = (postId: number) => {
  queryClient.prefetchQuery({
    queryKey: ['posts', postId],
    queryFn: () => postApi.getById(postId),
  });
};

// 4. Cache persistence for offline
const persister = createSyncStoragePersister({
  storage: window.localStorage,
});

// 5. Background sync
const { data } = useQuery({
  queryKey: ['feed'],
  queryFn: feedApi.get,
  refetchInterval: 60000, // Refresh every minute
  refetchOnWindowFocus: true,
});
```

## Chapter Exercise: Build Data Layer

Create a complete data fetching layer:

**Requirements:**
1. Set up React Query with proper config
2. Implement CRUD operations with caching
3. Add optimistic updates for mutations
4. Build infinite scroll pagination
5. Implement prefetching strategy
6. Add cache persistence
7. Handle errors properly
8. Set up background refetching

**Deliverables:**
- Complete query hooks for all resources
- Optimistic updates working
- Prefetching on navigation
- Cache persists across sessions

## Review Checklist

- [ ] React Query configured properly
- [ ] Query hooks organized by resource
- [ ] Optimistic updates implemented
- [ ] Cache invalidation strategy
- [ ] Prefetching on hover/navigation
- [ ] Error handling in place
- [ ] Retry logic configured
- [ ] Background refetching enabled
- [ ] Cache persistence (optional)
- [ ] DevTools configured

## Key Takeaways

1. **Use React Query or SWR** - Don't manage server state manually
2. **Cache aggressively** - Instant UI updates
3. **Optimistic updates improve UX** - Don't wait for server
4. **Prefetch strategically** - Hover, navigation, scroll
5. **Invalidate carefully** - Only refetch what changed
6. **Background refetch keeps data fresh** - Window focus, polling
7. **Persist cache for offline** - Better mobile experience

## Further Reading

- React Query documentation
- SWR documentation
- Caching best practices
- Offline-first architecture
- GraphQL with React Query

## Next Chapter

[Chapter 21: Building Scalable Forms](./21-building-scalable-forms.md)
