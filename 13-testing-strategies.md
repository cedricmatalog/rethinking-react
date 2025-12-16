# Chapter 13: Testing Strategies

## Introduction

Junior developers write tests to check if code works. Senior developers write tests to enable confident refactoring, document behavior, and catch regressions.

The difference is in the strategy, not the tools.

## Learning Objectives

- Understand the testing pyramid
- Write tests that don't break on refactoring
- Test user behavior, not implementation
- Know what to test and what to skip
- Build confidence in your test suite

## 13.1 The Testing Mindset Shift

### Junior vs Senior Testing Approach

| Junior Developer | Senior Developer |
|-----------------|------------------|
| Tests implementation details | Tests user behavior |
| 100% coverage goal | Meaningful coverage goal |
| Tests after writing code | Tests guide design (TDD when appropriate) |
| Fragile tests that break on refactors | Resilient tests that survive refactors |
| Tests components in isolation | Tests features integrated |

### Example: Testing a Counter

```javascript
// Junior test - testing implementation
test('clicking button increments count state', () => {
  const { result } = renderHook(() => useState(0));
  const [count, setCount] = result.current;

  act(() => {
    setCount(count + 1);
  });

  expect(result.current[0]).toBe(1);
});

// Senior test - testing user behavior
test('user can increment counter', () => {
  render(<Counter />);

  expect(screen.getByText(/count: 0/i)).toBeInTheDocument();

  userEvent.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
});
```

The senior test:
- Tests what users see and do
- Doesn't care about useState vs useReducer
- Survives refactoring
- Documents behavior

## 13.2 The Testing Pyramid

```
        /\
       /  \
      /E2E \          Few (5-10%)
     /------\         - Test critical user flows
    /  Integ \        - Slow, expensive
   /----------\       - Test happy paths
  / Unit Tests \      Many (70-80%)
 /--------------\     - Test logic, edge cases
                      - Fast, cheap
```

### What to Test at Each Level

**Unit Tests (70-80%)**
- Pure functions
- Custom hooks
- Utilities
- Business logic
- Edge cases

**Integration Tests (15-25%)**
- Features working together
- API integration
- State management
- User workflows

**E2E Tests (5-10%)**
- Critical user paths
- Authentication flow
- Checkout flow
- Core business flows

### Common Mistake: Inverted Pyramid

```javascript
// DON'T: Too many E2E tests
describe('TodoApp E2E', () => {
  test('adding a todo', () => {/* browser test */});
  test('editing a todo', () => {/* browser test */});
  test('deleting a todo', () => {/* browser test */});
  test('filtering todos', () => {/* browser test */});
  test('marking complete', () => {/* browser test */});
  // ... 50 more E2E tests (slow, flaky)
});

// DO: E2E for critical path, unit/integration for details
describe('TodoApp E2E', () => {
  test('user can manage todos (add, edit, complete)', () => {
    // One test for the happy path
  });
});

describe('TodoApp Integration', () => {
  test('adding a todo');
  test('editing a todo');
  test('deleting a todo');
  // ... more focused integration tests
});
```

## 13.3 Testing User Behavior

### The Testing Library Philosophy

> "The more your tests resemble the way your software is used, the more confidence they can give you."

```javascript
// BAD: Testing implementation
test('updates state when input changes', () => {
  const { getByTestId } = render(<LoginForm />);
  const input = getByTestId('email-input');

  fireEvent.change(input, { target: { value: 'test@example.com' } });

  // Testing internal state - implementation detail!
  expect(input.value).toBe('test@example.com');
});

// GOOD: Testing behavior
test('user can log in with email and password', async () => {
  const handleLogin = jest.fn();
  render(<LoginForm onLogin={handleLogin} />);

  // User actions
  await userEvent.type(
    screen.getByRole('textbox', { name: /email/i }),
    'test@example.com'
  );
  await userEvent.type(
    screen.getByLabelText(/password/i),
    'password123'
  );
  await userEvent.click(
    screen.getByRole('button', { name: /log in/i })
  );

  // User expectations
  expect(handleLogin).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123'
  });
});
```

### Query Priority

Use queries in this order:

1. **Accessible queries (best)**
   - `getByRole` - buttons, links, inputs
   - `getByLabelText` - form inputs
   - `getByPlaceholderText` - inputs
   - `getByText` - non-interactive elements

2. **Semantic queries (okay)**
   - `getByAltText` - images
   - `getByTitle` - title attribute

3. **Test IDs (last resort)**
   - `getByTestId` - when nothing else works

```javascript
// Prefer this
screen.getByRole('button', { name: /submit/i })

// Over this
screen.getByTestId('submit-button')
```

### Hands-On Exercise 13.3

Refactor these tests to test behavior instead of implementation:

```javascript
// Current tests (implementation-focused)
test('sets loading to true when fetching');
test('sets error state on API failure');
test('updates data state on success');
test('calls useEffect cleanup on unmount');

// Your task: Write behavior-focused tests
// Hint: What does the user see/do?
```

## 13.4 Testing Async Code

### The Challenge

```javascript
// Component with async behavior
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <Spinner />;
  if (!user) return <div>User not found</div>;
  return <div>{user.name}</div>;
}
```

### Junior Approach: Wait for arbitrary time

```javascript
// FRAGILE: Timing-dependent
test('shows user after loading', async () => {
  render(<UserProfile userId="123" />);

  // Wait random time
  await new Promise(resolve => setTimeout(resolve, 1000));

  expect(screen.getByText('John Doe')).toBeInTheDocument();
});
```

### Senior Approach: Wait for specific changes

```javascript
// ROBUST: Wait for actual changes
test('shows user after loading', async () => {
  render(<UserProfile userId="123" />);

  // Initial loading state
  expect(screen.getByRole('progressbar')).toBeInTheDocument();

  // Wait for user to appear
  expect(await screen.findByText('John Doe')).toBeInTheDocument();

  // Spinner should be gone
  expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
});
```

### Mocking API Calls

```javascript
// Mock at the service layer
import { fetchUser } from '@/services/userService';

jest.mock('@/services/userService');

test('displays user data', async () => {
  fetchUser.mockResolvedValue({
    id: '123',
    name: 'John Doe',
    email: 'john@example.com'
  });

  render(<UserProfile userId="123" />);

  expect(await screen.findByText('John Doe')).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
});

test('shows error message on failure', async () => {
  fetchUser.mockRejectedValue(new Error('Network error'));

  render(<UserProfile userId="123" />);

  expect(
    await screen.findByText(/failed to load user/i)
  ).toBeInTheDocument();
});
```

### Hands-On Exercise 13.4

Test a component that:
1. Fetches data on mount
2. Shows loading spinner
3. Handles success and error states
4. Allows retry on error
5. Cancels requests on unmount

Write comprehensive tests using proper async patterns.

## 13.5 Testing Custom Hooks

### When to Test Hooks

| Test the Hook | Test Through Component |
|--------------|----------------------|
| Complex logic | Simple hooks |
| Reused widely | One-off hooks |
| Pure computation | Tightly coupled to UI |
| State machines | Basic useState wrappers |

### Example: Testing a Pagination Hook

```javascript
// hooks/usePagination.js
function usePagination({ items, itemsPerPage = 10 }) {
  const [currentPage, setCurrentPage] = useState(1);

  const totalPages = Math.ceil(items.length / itemsPerPage);
  const startIndex = (currentPage - 1) * itemsPerPage;
  const endIndex = startIndex + itemsPerPage;
  const currentItems = items.slice(startIndex, endIndex);

  const goToPage = (page) => {
    setCurrentPage(Math.max(1, Math.min(page, totalPages)));
  };

  const nextPage = () => goToPage(currentPage + 1);
  const prevPage = () => goToPage(currentPage - 1);

  return {
    currentItems,
    currentPage,
    totalPages,
    nextPage,
    prevPage,
    goToPage,
    hasNext: currentPage < totalPages,
    hasPrev: currentPage > 1
  };
}

// hooks/usePagination.test.js
import { renderHook, act } from '@testing-library/react';
import { usePagination } from './usePagination';

describe('usePagination', () => {
  const items = Array.from({ length: 25 }, (_, i) => `Item ${i + 1}`);

  test('returns first page by default', () => {
    const { result } = renderHook(() =>
      usePagination({ items, itemsPerPage: 10 })
    );

    expect(result.current.currentPage).toBe(1);
    expect(result.current.currentItems).toHaveLength(10);
    expect(result.current.currentItems[0]).toBe('Item 1');
  });

  test('navigates to next page', () => {
    const { result } = renderHook(() =>
      usePagination({ items, itemsPerPage: 10 })
    );

    act(() => {
      result.current.nextPage();
    });

    expect(result.current.currentPage).toBe(2);
    expect(result.current.currentItems[0]).toBe('Item 11');
  });

  test('cannot go beyond last page', () => {
    const { result } = renderHook(() =>
      usePagination({ items, itemsPerPage: 10 })
    );

    act(() => {
      result.current.goToPage(999);
    });

    expect(result.current.currentPage).toBe(3); // totalPages = 3
    expect(result.current.hasNext).toBe(false);
  });

  test('handles page boundaries correctly', () => {
    const { result } = renderHook(() =>
      usePagination({ items, itemsPerPage: 10 })
    );

    expect(result.current.hasPrev).toBe(false);
    expect(result.current.hasNext).toBe(true);

    act(() => {
      result.current.goToPage(3);
    });

    expect(result.current.hasPrev).toBe(true);
    expect(result.current.hasNext).toBe(false);
  });
});
```

### Hands-On Exercise 13.5

Test a `useForm` hook that handles:
- Field registration
- Validation
- Error messages
- Submission
- Reset functionality

## 13.6 Testing Patterns for Common Scenarios

### Testing Forms

```javascript
test('validates email before submission', async () => {
  const handleSubmit = jest.fn();
  render(<RegistrationForm onSubmit={handleSubmit} />);

  // Submit without filling
  await userEvent.click(screen.getByRole('button', { name: /sign up/i }));

  // Should show validation errors
  expect(screen.getByText(/email is required/i)).toBeInTheDocument();
  expect(handleSubmit).not.toHaveBeenCalled();

  // Fill with invalid email
  await userEvent.type(
    screen.getByRole('textbox', { name: /email/i }),
    'invalid-email'
  );
  await userEvent.click(screen.getByRole('button', { name: /sign up/i }));

  expect(screen.getByText(/invalid email/i)).toBeInTheDocument();
  expect(handleSubmit).not.toHaveBeenCalled();

  // Fill with valid email
  await userEvent.clear(screen.getByRole('textbox', { name: /email/i }));
  await userEvent.type(
    screen.getByRole('textbox', { name: /email/i }),
    'test@example.com'
  );
  await userEvent.click(screen.getByRole('button', { name: /sign up/i }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com'
  });
});
```

### Testing Modal Interactions

```javascript
test('user can open and close modal', async () => {
  render(<App />);

  // Modal not visible initially
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();

  // Open modal
  await userEvent.click(screen.getByRole('button', { name: /open modal/i }));

  // Modal appears
  expect(screen.getByRole('dialog')).toBeInTheDocument();
  expect(screen.getByText(/modal content/i)).toBeInTheDocument();

  // Close with X button
  await userEvent.click(screen.getByRole('button', { name: /close/i }));

  // Modal disappears
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});

test('modal closes on escape key', async () => {
  render(<App />);

  await userEvent.click(screen.getByRole('button', { name: /open modal/i }));
  expect(screen.getByRole('dialog')).toBeInTheDocument();

  await userEvent.keyboard('{Escape}');

  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});
```

### Testing Lists and Filtering

```javascript
test('user can filter products by category', async () => {
  render(<ProductCatalog />);

  // All products shown initially
  expect(screen.getAllByRole('article')).toHaveLength(10);

  // Filter by electronics
  await userEvent.click(screen.getByRole('button', { name: /electronics/i }));

  // Only electronics shown
  const products = screen.getAllByRole('article');
  expect(products).toHaveLength(3);
  products.forEach(product => {
    expect(product).toHaveTextContent(/electronics/i);
  });
});
```

### Testing Error Boundaries

```javascript
test('error boundary catches and displays error', () => {
  const ThrowError = () => {
    throw new Error('Test error');
  };

  // Suppress console.error for this test
  const consoleSpy = jest.spyOn(console, 'error').mockImplementation();

  render(
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <ThrowError />
    </ErrorBoundary>
  );

  expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();

  consoleSpy.mockRestore();
});
```

## 13.7 What NOT to Test

### Don't Test Implementation Details

```javascript
// DON'T test useState, useEffect, etc.
test('sets loading to true', () => {
  const { result } = renderHook(() => useData());
  expect(result.current.loading).toBe(true); // Implementation detail
});

// DO test user-visible behavior
test('shows loading spinner while fetching', () => {
  render(<DataView />);
  expect(screen.getByRole('progressbar')).toBeInTheDocument();
});
```

### Don't Test Third-Party Libraries

```javascript
// DON'T test if React Router works
test('navigate changes route', () => {
  // Testing react-router-dom internals
});

// DO test your navigation logic
test('clicking logo returns to home page', async () => {
  render(<App />);
  await userEvent.click(screen.getByRole('link', { name: /home/i }));
  expect(screen.getByRole('heading', { name: /home/i })).toBeInTheDocument();
});
```

### Don't Test Trivial Code

```javascript
// DON'T test simple mappers
test('formatDate formats date correctly', () => {
  expect(formatDate(date)).toBe('01/01/2024');
});

// Unless it has business logic
test('formatOrderDate shows relative time for recent orders', () => {
  // This has business rules worth testing
});
```

## Real-World Scenario: Inheriting Untested Code

### The Challenge

You inherit a 10,000-line React app with:
- 0% test coverage
- Complex state management
- Lots of edge cases
- Legacy patterns
- Frequent bugs

You cannot rewrite it. You need to add tests incrementally.

### Your Strategy

1. **Start with critical paths** - What breaks most often?
2. **Test before fixing bugs** - Write failing test, then fix
3. **Test new features** - 100% coverage for new code
4. **Add integration tests** - Cover main workflows
5. **Refactor with confidence** - Tests enable refactoring

### Discussion Questions
- Where would you start?
- How do you prioritize what to test?
- How do you convince stakeholders to invest in tests?
- What metrics matter?

## Chapter Exercise: Test-Driven Feature Development

Build a feature using TDD:

**Feature:** Shopping cart with:
- Add/remove items
- Update quantities
- Apply discount codes
- Calculate totals
- Persist to localStorage

**Process:**
1. Write failing test
2. Write minimal code to pass
3. Refactor
4. Repeat

**Evaluation:**
- All tests pass
- Good coverage of edge cases
- Tests survive refactoring
- Tests are readable

## Review Checklist

- [ ] Understand the testing pyramid
- [ ] Write behavior-focused tests
- [ ] Test async code properly
- [ ] Know when to test hooks vs components
- [ ] Avoid testing implementation details
- [ ] Use appropriate queries (getByRole, etc.)
- [ ] Write maintainable, resilient tests

## Key Takeaways

1. **Test behavior, not implementation** - Tests should survive refactoring
2. **Follow the testing pyramid** - Mostly unit, some integration, few E2E
3. **Use the right tools** - @testing-library/react for components
4. **Test user flows** - What users see and do matters
5. **Don't chase 100% coverage** - Chase confidence
6. **Write tests that document** - Tests should explain behavior

## Further Reading

- Testing Library documentation and guiding principles
- Kent C. Dodds: Testing blog posts
- Martin Fowler: Test Pyramid
- "Test-Driven Development with React" book

## Next Chapter

[Chapter 14: Integration & E2E Testing](./14-integration-e2e.md)
