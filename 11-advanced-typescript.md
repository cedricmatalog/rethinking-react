# Chapter 11: Advanced TypeScript for React

## Introduction

Junior developers use TypeScript for basic type checking. Senior developers leverage TypeScript's advanced features to catch bugs at compile time and create self-documenting APIs.

## Learning Objectives

- Master generics in React components
- Use advanced utility types effectively
- Implement type-safe patterns
- Create discriminated unions
- Leverage type inference
- Build type-safe hooks and contexts
- Handle complex prop types

## 11.1 Generics in React

### Generic Components

```typescript
// Generic List component
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <div>
      {items.map(item => (
        <div key={keyExtractor(item)}>
          {renderItem(item)}
        </div>
      ))}
    </div>
  );
}

// Usage - type is inferred
interface User {
  id: number;
  name: string;
}

<List
  items={users} // TypeScript knows items is User[]
  renderItem={(user) => <div>{user.name}</div>} // user is typed as User
  keyExtractor={(user) => user.id}
/>
```

### Generic Hooks

```typescript
// Type-safe localStorage hook
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((prev: T) => T)) => {
    try {
      const valueToStore = value instanceof Function
        ? value(storedValue)
        : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
}

// Usage - fully typed
const [user, setUser] = useLocalStorage<User>('user', { id: 0, name: '' });
setUser({ id: 1, name: 'John' }); // ✓ Typed
setUser({ id: 1 }); // ✗ Error: missing 'name'
```

## 11.2 Advanced Utility Types

### Built-in Utilities

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial - all properties optional
type UserUpdate = Partial<User>;
const update: UserUpdate = { name: 'John' }; // ✓ Valid

// Required - all properties required
type RequiredUser = Required<Partial<User>>; // Removes all optionals

// Pick - select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;
const preview: UserPreview = { id: 1, name: 'John' }; // ✓ Valid

// Omit - exclude properties
type UserWithoutPassword = Omit<User, 'password'>;
const safeUser: UserWithoutPassword = { 
  id: 1, 
  name: 'John',
  email: 'john@example.com',
  createdAt: new Date()
}; // password not allowed

// Record - create object type
type UserRoles = Record<number, 'admin' | 'user' | 'guest'>;
const roles: UserRoles = {
  1: 'admin',
  2: 'user'
}; // ✓ Valid
```

### Custom Utility Types

```typescript
// Make specific properties required
type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

interface FormData {
  name?: string;
  email?: string;
  age?: number;
}

type ValidatedForm = RequireFields<FormData, 'name' | 'email'>;
// name and email are required, age is optional

// Make properties nullable
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
// All properties can be null

// Deep Partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? DeepPartial<T[P]>
    : T[P];
};

interface Config {
  api: {
    url: string;
    timeout: number;
  };
  features: {
    darkMode: boolean;
  };
}

const partialConfig: DeepPartial<Config> = {
  api: { url: 'https://api.com' } // timeout optional
  // features optional
};
```

## 11.3 Discriminated Unions

### Type-Safe State Machines

```typescript
// API state with discriminated unions
type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function DataComponent() {
  const [state, setState] = useState<ApiState<User>>({ status: 'idle' });

  // TypeScript knows which properties exist based on status
  if (state.status === 'loading') {
    return <Spinner />;
  }

  if (state.status === 'error') {
    return <Error message={state.error.message} />; // error property exists
  }

  if (state.status === 'success') {
    return <UserCard user={state.data} />; // data property exists
  }

  return <button onClick={loadData}>Load</button>;
}
```

### Action Types

```typescript
// Type-safe reducer actions
type Action =
  | { type: 'SET_USER'; payload: User }
  | { type: 'CLEAR_USER' }
  | { type: 'UPDATE_FIELD'; field: keyof User; value: string }
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: User }
  | { type: 'FETCH_ERROR'; error: string };

function userReducer(state: UserState, action: Action): UserState {
  switch (action.type) {
    case 'SET_USER':
      return { ...state, user: action.payload }; // payload is typed as User

    case 'CLEAR_USER':
      return { ...state, user: null }; // no payload needed

    case 'UPDATE_FIELD':
      return {
        ...state,
        user: state.user
          ? { ...state.user, [action.field]: action.value }
          : null
      }; // field and value are typed

    case 'FETCH_ERROR':
      return { ...state, error: action.error }; // error is string

    default:
      // TypeScript ensures all cases are handled
      const _exhaustive: never = action;
      return state;
  }
}
```

## 11.4 Type Inference

### Inferring from Props

```typescript
// Infer component props from implementation
function Button({ variant, size, children, ...props }: {
  variant: 'primary' | 'secondary';
  size: 'small' | 'large';
  children: React.ReactNode;
} & React.ButtonHTMLAttributes<HTMLButtonElement>) {
  return <button {...props}>{children}</button>;
}

// Extract props type
type ButtonProps = React.ComponentProps<typeof Button>;

// Use inferred type
const props: ButtonProps = {
  variant: 'primary',
  size: 'large',
  children: 'Click',
  onClick: () => {}
};
```

### Inferring Generic Types

```typescript
// TypeScript infers generic types from usage
function createStore<T>(initialState: T) {
  // implementation
  return {
    getState: () => initialState,
    setState: (newState: T) => {}
  };
}

// Type is inferred as { count: number }
const store = createStore({ count: 0 });
store.setState({ count: 1 }); // ✓ Valid
store.setState({ count: 'invalid' }); // ✗ Error
```

## Review Checklist

- [ ] Use generics for reusable components
- [ ] Leverage utility types (Partial, Pick, Omit)
- [ ] Create discriminated unions for state machines
- [ ] Let TypeScript infer types when possible
- [ ] Create custom utility types for common patterns
- [ ] Use type guards effectively
- [ ] Ensure exhaustive type checking

## Key Takeaways

1. **Generics enable reusability** - Write once, type-safe everywhere
2. **Utility types reduce boilerplate** - Use built-in TypeScript utilities
3. **Discriminated unions prevent bugs** - Type-safe state machines
4. **Inference is powerful** - Let TypeScript do the work
5. **Types document code** - Self-documenting APIs
6. **Strictness catches bugs** - Enable strict mode
7. **Advanced types = better DX** - Invest in type infrastructure

## Further Reading

- TypeScript Handbook: Advanced Types
- React TypeScript Cheatsheet
- Type Challenges (github.com/type-challenges)

## Next Chapter

[Chapter 12: Type-Safe APIs & Data Flow](./12-type-safe-apis.md)
