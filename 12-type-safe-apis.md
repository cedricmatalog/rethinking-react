# Chapter 12: Type-Safe APIs & Data Flow

## Introduction

Junior developers make API calls and hope the data is correct. Senior developers enforce type safety from API to UI, catching errors at compile time.

## Learning Objectives

- Generate types from API schemas
- Implement end-to-end type safety
- Handle API errors type-safely
- Create type-safe form handling
- Use Zod for runtime validation
- Build type-safe data transformations
- Ensure type safety across the stack

## 12.1 API Type Generation

### From OpenAPI/Swagger

```typescript
// Using openapi-typescript
// Generate types from OpenAPI spec
// npx openapi-typescript https://api.example.com/openapi.json -o types/api.ts

// Generated types
import type { paths } from './types/api';

type UserResponse = paths['/users/{id}']['get']['responses']['200']['content']['application/json'];
type CreateUserRequest = paths['/users']['post']['requestBody']['content']['application/json'];

// Type-safe API client
async function getUser(id: number): Promise<UserResponse> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

async function createUser(data: CreateUserRequest): Promise<UserResponse> {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  return response.json();
}
```

### Using tRPC for Full-Stack Type Safety

```typescript
// server/router.ts
import { z } from 'zod';
import { router, publicProcedure } from './trpc';

export const appRouter = router({
  getUser: publicProcedure
    .input(z.object({ id: z.number() }))
    .query(async ({ input }) => {
      const user = await db.user.findUnique({ where: { id: input.id } });
      return user;
    }),

  createUser: publicProcedure
    .input(z.object({
      name: z.string().min(2),
      email: z.string().email(),
      age: z.number().min(18)
    }))
    .mutation(async ({ input }) => {
      const user = await db.user.create({ data: input });
      return user;
    })
});

export type AppRouter = typeof appRouter;

// client/api.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/router';

export const trpc = createTRPCReact<AppRouter>();

// Usage - fully typed!
function UserProfile({ userId }: { userId: number }) {
  const { data: user, isLoading } = trpc.getUser.useQuery({ id: userId });
  const createUserMutation = trpc.createUser.useMutation();

  if (isLoading) return <Spinner />;

  return (
    <div>
      <h1>{user?.name}</h1>
      <button onClick={() => {
        createUserMutation.mutate({
          name: 'John',
          email: 'john@example.com',
          age: 25
        }); // Fully typed and validated!
      }}>
        Create User
      </button>
    </div>
  );
}
```

## 12.2 Runtime Validation with Zod

### Schema Definition

```typescript
import { z } from 'zod';

// Define schema
const UserSchema = z.object({
  id: z.number(),
  name: z.string().min(2).max(50),
  email: z.string().email(),
  age: z.number().min(18).max(120).optional(),
  roles: z.array(z.enum(['admin', 'user', 'guest'])),
  metadata: z.record(z.string(), z.unknown()).optional(),
  createdAt: z.string().datetime()
});

// Infer TypeScript type from schema
type User = z.infer<typeof UserSchema>;

// Validate data
function validateUser(data: unknown): User {
  return UserSchema.parse(data); // Throws if invalid
}

// Safe parse (doesn't throw)
function safeValidateUser(data: unknown) {
  const result = UserSchema.safeParse(data);
  
  if (result.success) {
    return { user: result.data, error: null };
  } else {
    return { user: null, error: result.error };
  }
}

// Usage in API call
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  
  // Validate at runtime
  return UserSchema.parse(data);
}
```

### Form Validation

```typescript
const RegisterFormSchema = z.object({
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be less than 20 characters')
    .regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores'),
  
  email: z.string()
    .email('Invalid email address'),
  
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number'),
  
  confirmPassword: z.string(),
  
  age: z.number()
    .min(18, 'Must be at least 18 years old')
    .or(z.string().transform(Number)),
  
  terms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms' })
  })
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword']
});

type RegisterForm = z.infer<typeof RegisterFormSchema>;

function RegisterForm() {
  const [formData, setFormData] = useState<Partial<RegisterForm>>({});
  const [errors, setErrors] = useState<z.ZodError | null>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    const result = RegisterFormSchema.safeParse(formData);
    
    if (result.success) {
      // Submit valid data
      submitRegistration(result.data);
    } else {
      // Show errors
      setErrors(result.error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="username"
        onChange={e => setFormData(prev => ({ ...prev, username: e.target.value }))}
      />
      {errors?.formErrors.fieldErrors.username && (
        <span>{errors.formErrors.fieldErrors.username[0]}</span>
      )}
      
      {/* More fields... */}
      
      <button type="submit">Register</button>
    </form>
  );
}
```

## 12.3 Type-Safe React Query

### Typed Query Keys

```typescript
// queryKeys.ts
export const queryKeys = {
  users: {
    all: ['users'] as const,
    lists: () => [...queryKeys.users.all, 'list'] as const,
    list: (filters: UserFilters) => [...queryKeys.users.lists(), filters] as const,
    details: () => [...queryKeys.users.all, 'detail'] as const,
    detail: (id: number) => [...queryKeys.users.details(), id] as const
  },
  posts: {
    all: ['posts'] as const,
    detail: (id: number) => [...queryKeys.posts.all, id] as const
  }
} as const;

// Usage
function UserProfile({ userId }: { userId: number }) {
  const { data: user } = useQuery({
    queryKey: queryKeys.users.detail(userId),
    queryFn: () => fetchUser(userId)
  });

  return <div>{user?.name}</div>;
}

// Invalidation
function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.users.lists() });
    }
  });
}
```

### Type-Safe Query Functions

```typescript
// api/users.ts
export const userApi = {
  getAll: async (): Promise<User[]> => {
    const response = await fetch('/api/users');
    const data = await response.json();
    return z.array(UserSchema).parse(data);
  },

  getById: async (id: number): Promise<User> => {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();
    return UserSchema.parse(data);
  },

  create: async (input: CreateUserInput): Promise<User> => {
    const validInput = CreateUserSchema.parse(input);
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(validInput)
    });
    const data = await response.json();
    return UserSchema.parse(data);
  },

  update: async (id: number, input: UpdateUserInput): Promise<User> => {
    const validInput = UpdateUserSchema.parse(input);
    const response = await fetch(`/api/users/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(validInput)
    });
    const data = await response.json();
    return UserSchema.parse(data);
  }
};

// hooks/useUsers.ts
export function useUsers() {
  return useQuery({
    queryKey: queryKeys.users.lists(),
    queryFn: userApi.getAll
  });
}

export function useUser(id: number) {
  return useQuery({
    queryKey: queryKeys.users.detail(id),
    queryFn: () => userApi.getById(id),
    enabled: id > 0
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: userApi.create,
    onSuccess: (newUser) => {
      queryClient.setQueryData(
        queryKeys.users.detail(newUser.id),
        newUser
      );
      queryClient.invalidateQueries({
        queryKey: queryKeys.users.lists()
      });
    }
  });
}
```

## 12.4 Error Handling Type Safety

### Type-Safe Error Types

```typescript
// errors.ts
export class ApiError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export class ValidationError extends ApiError {
  constructor(
    public fields: Record<string, string[]>
  ) {
    super(400, 'VALIDATION_ERROR', 'Validation failed', fields);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends ApiError {
  constructor(resource: string, id: string | number) {
    super(404, 'NOT_FOUND', `${resource} with id ${id} not found`);
    this.name = 'NotFoundError';
  }
}

// Type guard
export function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}

export function isValidationError(error: unknown): error is ValidationError {
  return error instanceof ValidationError;
}

// Usage
async function fetchUser(id: number): Promise<User> {
  try {
    const response = await fetch(`/api/users/${id}`);
    
    if (!response.ok) {
      if (response.status === 404) {
        throw new NotFoundError('User', id);
      }
      
      if (response.status === 400) {
        const errorData = await response.json();
        throw new ValidationError(errorData.fields);
      }
      
      throw new ApiError(
        response.status,
        'UNKNOWN_ERROR',
        'An error occurred'
      );
    }
    
    const data = await response.json();
    return UserSchema.parse(data);
  } catch (error) {
    if (isApiError(error)) {
      throw error;
    }
    throw new ApiError(500, 'NETWORK_ERROR', 'Network error occurred');
  }
}

// Component error handling
function UserProfile({ userId }: { userId: number }) {
  const { data: user, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId)
  });

  if (error) {
    if (isNotFoundError(error)) {
      return <div>User not found</div>;
    }
    
    if (isValidationError(error)) {
      return (
        <div>
          {Object.entries(error.fields).map(([field, messages]) => (
            <div key={field}>
              {field}: {messages.join(', ')}
            </div>
          ))}
        </div>
      );
    }
    
    return <div>Error: {error.message}</div>;
  }

  return <div>{user?.name}</div>;
}
```

## 12.5 Type-Safe State Management

### Zustand with TypeScript

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set, get) => ({
        user: null,
        token: null,
        isAuthenticated: false,

        login: async (email, password) => {
          const response = await fetch('/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
          });
          
          const data = await response.json();
          const { user, token } = LoginResponseSchema.parse(data);
          
          set({ user, token, isAuthenticated: true });
        },

        logout: () => {
          set({ user: null, token: null, isAuthenticated: false });
        },

        updateUser: (updates) => {
          const currentUser = get().user;
          if (!currentUser) return;
          
          const updatedUser = { ...currentUser, ...updates };
          set({ user: updatedUser });
        }
      }),
      { name: 'auth-storage' }
    )
  )
);

// Selectors
export const useUser = () => useAuthStore(state => state.user);
export const useIsAuthenticated = () => useAuthStore(state => state.isAuthenticated);
```

## 12.6 GraphQL Type Safety

### Using GraphQL Code Generator

```typescript
// codegen.yml
// generates: src/generated/graphql.tsx

// Generated types
import { gql } from '@apollo/client';
import type { User, GetUserQuery, GetUserQueryVariables } from './generated/graphql';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      posts {
        id
        title
      }
    }
  }
`;

function UserProfile({ userId }: { userId: string }) {
  const { data, loading } = useQuery<GetUserQuery, GetUserQueryVariables>(
    GET_USER,
    { variables: { id: userId } }
  );

  if (loading) return <Spinner />;

  // data.user is fully typed!
  return (
    <div>
      <h1>{data?.user?.name}</h1>
      {data?.user?.posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}
```

## Real-World Scenario: API Integration

### The Challenge

Integrate with a third-party API:
- No TypeScript types provided
- Inconsistent response formats
- Need runtime validation
- Must handle errors gracefully

### Your Solution

```typescript
// 1. Define schemas with Zod
const ProductSchema = z.object({
  id: z.number(),
  name: z.string(),
  price: z.number(),
  // Handle inconsistent date formats
  createdAt: z.string().or(z.number()).transform(val => 
    typeof val === 'number' ? new Date(val) : new Date(val)
  )
});

// 2. Create type-safe API client
const apiClient = {
  getProducts: async (): Promise<Product[]> => {
    const response = await fetch('/api/products');
    const data = await response.json();
    return z.array(ProductSchema).parse(data);
  }
};

// 3. Use with React Query
function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: apiClient.getProducts,
    retry: (failureCount, error) => {
      if (isValidationError(error)) return false;
      return failureCount < 3;
    }
  });
}
```

## Chapter Exercise: Build Type-Safe API Layer

Create a complete type-safe API integration:

**Requirements:**
1. Define Zod schemas for all entities
2. Generate TypeScript types from schemas
3. Create type-safe API client
4. Implement error handling with custom error types
5. Add React Query integration
6. Handle loading/error states
7. Add optimistic updates

**Evaluation:**
- Full type safety from API to UI
- Runtime validation
- Error handling
- No `any` types

## Review Checklist

- [ ] Generate types from API schemas
- [ ] Use Zod for runtime validation
- [ ] Create type-safe query keys
- [ ] Implement custom error types
- [ ] Use type guards for error handling
- [ ] Infer types from schemas
- [ ] Handle edge cases type-safely

## Key Takeaways

1. **Runtime validation is essential** - Types are erased at runtime
2. **Zod bridges compile-time and runtime** - Single source of truth
3. **Type generation saves time** - OpenAPI, GraphQL, tRPC
4. **Error types prevent bugs** - Handle errors explicitly
5. **End-to-end type safety** - From API to UI
6. **Validate at boundaries** - API responses, user input
7. **Type inference reduces duplication** - Let TypeScript work

## Further Reading

- Zod documentation
- tRPC documentation
- OpenAPI TypeScript generator
- GraphQL Code Generator
- TypeScript strict mode guide

## Next Chapter

[Chapter 13: Testing Strategies](./13-testing-strategies.md)
