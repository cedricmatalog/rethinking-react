# Chapter 14: Integration & E2E Testing

## Introduction

Junior developers write isolated unit tests and call it done. Senior developers understand that real bugs happen when systems interact, and they build comprehensive integration and end-to-end test suites to catch issues before users do.

## Learning Objectives

- Build effective integration tests
- Implement end-to-end testing with Playwright/Cypress
- Test API integrations properly
- Handle async operations in tests
- Mock external dependencies strategically
- Create maintainable test suites
- Balance test coverage with execution time

## 14.1 Integration Testing Fundamentals

### Testing React Query Integration

```typescript
// UserProfile.test.tsx
import { renderWithClient } from './test-utils';
import { QueryClient } from '@tanstack/react-query';
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({
        id: Number(id),
        name: 'John Doe',
        email: 'john@example.com'
      })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('UserProfile Integration', () => {
  it('fetches and displays user data', async () => {
    const { findByText } = renderWithClient(
      <UserProfile userId={1} />
    );

    // Wait for API call and render
    expect(await findByText('John Doe')).toBeInTheDocument();
    expect(await findByText('john@example.com')).toBeInTheDocument();
  });

  it('handles API errors gracefully', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    const { findByText } = renderWithClient(
      <UserProfile userId={1} />
    );

    expect(await findByText(/error/i)).toBeInTheDocument();
  });

  it('refetches on retry', async () => {
    let callCount = 0;

    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        callCount++;
        if (callCount === 1) {
          return res(ctx.status(500));
        }
        return res(ctx.json({ id: 1, name: 'John Doe' }));
      })
    );

    const { findByText, getByRole } = renderWithClient(
      <UserProfile userId={1} />
    );

    await findByText(/error/i);

    const retryButton = getByRole('button', { name: /retry/i });
    fireEvent.click(retryButton);

    expect(await findByText('John Doe')).toBeInTheDocument();
    expect(callCount).toBe(2);
  });
});

// test-utils.tsx
export function renderWithClient(
  ui: React.ReactElement,
  client?: QueryClient
) {
  const queryClient = client || new QueryClient({
    defaultOptions: {
      queries: {
        retry: false, // Disable retries in tests
        cacheTime: 0  // Disable caching
      }
    }
  });

  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}
```

### Testing Context Integration

```typescript
// AuthContext.test.tsx
describe('Auth Integration', () => {
  it('provides authentication state to children', async () => {
    const { getByText, getByLabelText } = render(
      <AuthProvider>
        <LoginForm />
        <ProtectedContent />
      </AuthProvider>
    );

    // Initially not authenticated
    expect(getByText('Please log in')).toBeInTheDocument();

    // Perform login
    fireEvent.change(getByLabelText('Email'), {
      target: { value: 'user@example.com' }
    });
    fireEvent.change(getByLabelText('Password'), {
      target: { value: 'password123' }
    });
    fireEvent.click(getByText('Login'));

    // Wait for authentication
    await waitFor(() => {
      expect(getByText('Welcome back!')).toBeInTheDocument();
    });
  });

  it('persists auth state across remounts', async () => {
    const { unmount, rerender } = render(
      <AuthProvider>
        <UserStatus />
      </AuthProvider>
    );

    // Login
    await userEvent.type(screen.getByLabelText('Email'), 'user@example.com');
    await userEvent.click(screen.getByText('Login'));

    await waitFor(() => {
      expect(screen.getByText('Logged in')).toBeInTheDocument();
    });

    // Unmount and remount
    unmount();

    render(
      <AuthProvider>
        <UserStatus />
      </AuthProvider>
    );

    // Should still be logged in
    expect(screen.getByText('Logged in')).toBeInTheDocument();
  });
});
```

## 14.2 Testing Component Interactions

### Testing Form Submission Flow

```typescript
describe('Registration Flow', () => {
  it('completes full registration process', async () => {
    const onSuccess = jest.fn();

    const { getByLabelText, getByRole } = renderWithClient(
      <RegistrationForm onSuccess={onSuccess} />
    );

    // Fill out form
    await userEvent.type(
      getByLabelText('Username'),
      'johndoe'
    );
    await userEvent.type(
      getByLabelText('Email'),
      'john@example.com'
    );
    await userEvent.type(
      getByLabelText('Password'),
      'SecurePass123'
    );
    await userEvent.type(
      getByLabelText('Confirm Password'),
      'SecurePass123'
    );
    await userEvent.click(
      getByLabelText('I accept the terms')
    );

    // Submit
    await userEvent.click(
      getByRole('button', { name: /register/i })
    );

    // Wait for API call
    await waitFor(() => {
      expect(onSuccess).toHaveBeenCalledWith({
        id: expect.any(Number),
        username: 'johndoe',
        email: 'john@example.com'
      });
    });
  });

  it('shows validation errors on invalid input', async () => {
    const { getByLabelText, getByRole, findByText } = renderWithClient(
      <RegistrationForm />
    );

    await userEvent.type(getByLabelText('Username'), 'ab'); // Too short
    await userEvent.type(getByLabelText('Email'), 'invalid-email');
    await userEvent.type(getByLabelText('Password'), 'weak');

    await userEvent.click(getByRole('button', { name: /register/i }));

    expect(await findByText(/username must be at least 3 characters/i))
      .toBeInTheDocument();
    expect(await findByText(/invalid email/i)).toBeInTheDocument();
    expect(await findByText(/password must be at least 8 characters/i))
      .toBeInTheDocument();
  });
});
```

### Testing Multi-Step Wizards

```typescript
describe('Checkout Wizard', () => {
  it('progresses through all steps', async () => {
    const { getByRole, getByLabelText } = renderWithClient(
      <CheckoutWizard />
    );

    // Step 1: Shipping Info
    await userEvent.type(getByLabelText('Address'), '123 Main St');
    await userEvent.type(getByLabelText('City'), 'New York');
    await userEvent.click(getByRole('button', { name: /next/i }));

    // Step 2: Payment
    await waitFor(() => {
      expect(getByLabelText('Card Number')).toBeInTheDocument();
    });

    await userEvent.type(getByLabelText('Card Number'), '4111111111111111');
    await userEvent.click(getByRole('button', { name: /next/i }));

    // Step 3: Review
    await waitFor(() => {
      expect(screen.getByText('123 Main St')).toBeInTheDocument();
      expect(screen.getByText('****1111')).toBeInTheDocument();
    });

    // Submit
    await userEvent.click(getByRole('button', { name: /place order/i }));

    await waitFor(() => {
      expect(screen.getByText(/order confirmed/i)).toBeInTheDocument();
    });
  });

  it('allows going back to previous steps', async () => {
    const { getByRole } = renderWithClient(<CheckoutWizard />);

    // Go to step 2
    await userEvent.click(getByRole('button', { name: /next/i }));

    // Go back
    await userEvent.click(getByRole('button', { name: /back/i }));

    // Should be on step 1
    expect(screen.getByLabelText('Address')).toBeInTheDocument();
  });
});
```

## 14.3 End-to-End Testing with Playwright

### Setup and Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### User Journey Tests

```typescript
// e2e/user-journey.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Complete User Journey', () => {
  test('new user registration and first purchase', async ({ page }) => {
    // Navigate to homepage
    await page.goto('/');

    // Register
    await page.click('text=Sign Up');
    await page.fill('[name="email"]', 'newuser@example.com');
    await page.fill('[name="password"]', 'SecurePass123');
    await page.fill('[name="confirmPassword"]', 'SecurePass123');
    await page.click('button:has-text("Create Account")');

    // Wait for redirect to dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('text=Welcome')).toBeVisible();

    // Browse products
    await page.click('nav >> text=Products');
    await expect(page).toHaveURL('/products');

    // Search for product
    await page.fill('[placeholder="Search products"]', 'laptop');
    await page.press('[placeholder="Search products"]', 'Enter');

    // Click first product
    await page.click('.product-card >> nth=0');

    // Add to cart
    await page.click('button:has-text("Add to Cart")');
    await expect(page.locator('.cart-count')).toHaveText('1');

    // Go to cart
    await page.click('.cart-icon');
    await expect(page).toHaveURL('/cart');

    // Proceed to checkout
    await page.click('button:has-text("Checkout")');

    // Fill shipping info
    await page.fill('[name="address"]', '123 Main St');
    await page.fill('[name="city"]', 'New York');
    await page.fill('[name="zipCode"]', '10001');
    await page.click('button:has-text("Continue")');

    // Fill payment info (using test card)
    await page.fill('[name="cardNumber"]', '4242424242424242');
    await page.fill('[name="expiry"]', '12/25');
    await page.fill('[name="cvv"]', '123');

    // Place order
    await page.click('button:has-text("Place Order")');

    // Verify success
    await expect(page.locator('text=Order Confirmed')).toBeVisible();
    await expect(page.locator('.order-number')).toBeVisible();
  });

  test('handles out of stock items', async ({ page }) => {
    await page.goto('/products/out-of-stock-item');

    const addToCartButton = page.locator('button:has-text("Add to Cart")');
    await expect(addToCartButton).toBeDisabled();
    await expect(page.locator('text=Out of Stock')).toBeVisible();
  });
});
```

### Page Object Model

```typescript
// e2e/pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.page.fill('[name="email"]', email);
    await this.page.fill('[name="password"]', password);
    await this.page.click('button:has-text("Login")');
  }

  async getErrorMessage() {
    return this.page.locator('.error-message').textContent();
  }
}

// e2e/pages/DashboardPage.ts
export class DashboardPage {
  constructor(private page: Page) {}

  async expectToBeVisible() {
    await expect(this.page).toHaveURL('/dashboard');
    await expect(this.page.locator('h1')).toContainText('Dashboard');
  }

  async getUserName() {
    return this.page.locator('.user-name').textContent();
  }
}

// Usage in tests
test('login flow with page objects', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await dashboardPage.expectToBeVisible();
  expect(await dashboardPage.getUserName()).toBe('John Doe');
});
```

## 14.4 Testing API Integration

### MSW (Mock Service Worker) Setup

```typescript
// mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  // GET requests
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;

    return res(
      ctx.status(200),
      ctx.json({
        id: Number(id),
        name: 'John Doe',
        email: 'john@example.com'
      })
    );
  }),

  // POST requests
  rest.post('/api/users', async (req, res, ctx) => {
    const body = await req.json();

    // Validate request
    if (!body.email || !body.name) {
      return res(
        ctx.status(400),
        ctx.json({ error: 'Missing required fields' })
      );
    }

    return res(
      ctx.status(201),
      ctx.json({
        id: Math.random(),
        ...body
      })
    );
  }),

  // Error simulation
  rest.get('/api/unstable', (req, res, ctx) => {
    return res.networkError('Network error');
  }),

  // Delayed responses
  rest.get('/api/slow', (req, res, ctx) => {
    return res(
      ctx.delay(2000),
      ctx.json({ data: 'slow response' })
    );
  })
];

// mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### Advanced MSW Patterns

```typescript
describe('API Integration Advanced', () => {
  it('handles pagination correctly', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        const page = req.url.searchParams.get('page');
        const limit = req.url.searchParams.get('limit');

        const users = generateUsers(100);
        const start = (Number(page) - 1) * Number(limit);
        const end = start + Number(limit);

        return res(
          ctx.json({
            data: users.slice(start, end),
            total: users.length,
            page: Number(page),
            hasMore: end < users.length
          })
        );
      })
    );

    const { findByText, getByRole } = renderWithClient(<UserList />);

    // First page
    expect(await findByText('User 1')).toBeInTheDocument();

    // Load more
    await userEvent.click(getByRole('button', { name: /load more/i }));

    expect(await findByText('User 21')).toBeInTheDocument();
  });

  it('handles race conditions', async () => {
    let requestCount = 0;

    server.use(
      rest.get('/api/search', async (req, res, ctx) => {
        const query = req.url.searchParams.get('q');
        requestCount++;
        const currentRequest = requestCount;

        // Simulate varying response times
        await new Promise(resolve =>
          setTimeout(resolve, currentRequest === 1 ? 500 : 100)
        );

        // Only return if this is the latest request
        if (currentRequest === requestCount) {
          return res(ctx.json({ results: [`Result for ${query}`] }));
        }

        return res(ctx.status(499)); // Client Closed Request
      })
    );

    const { getByRole } = renderWithClient(<Search />);

    // Type quickly to create race condition
    await userEvent.type(getByRole('searchbox'), 'first');
    await userEvent.clear(getByRole('searchbox'));
    await userEvent.type(getByRole('searchbox'), 'second');

    // Should only show results for latest query
    await waitFor(() => {
      expect(screen.getByText('Result for second')).toBeInTheDocument();
      expect(screen.queryByText('Result for first')).not.toBeInTheDocument();
    });
  });
});
```

## 14.5 Testing Async Patterns

### Testing Optimistic Updates

```typescript
describe('Optimistic Updates', () => {
  it('shows optimistic update, then reverts on error', async () => {
    server.use(
      rest.post('/api/todos', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    const { getByRole, getByText, queryByText } = renderWithClient(
      <TodoList />
    );

    // Add todo
    await userEvent.type(getByRole('textbox'), 'New todo');
    await userEvent.click(getByRole('button', { name: /add/i }));

    // Optimistically added
    expect(getByText('New todo')).toBeInTheDocument();

    // Wait for revert
    await waitFor(() => {
      expect(queryByText('New todo')).not.toBeInTheDocument();
    });

    // Error shown
    expect(getByText(/failed to add/i)).toBeInTheDocument();
  });

  it('persists optimistic update on success', async () => {
    server.use(
      rest.post('/api/todos', async (req, res, ctx) => {
        const body = await req.json();
        return res(
          ctx.status(201),
          ctx.json({ id: '123', ...body })
        );
      })
    );

    const { getByRole, getByText } = renderWithClient(<TodoList />);

    await userEvent.type(getByRole('textbox'), 'New todo');
    await userEvent.click(getByRole('button', { name: /add/i }));

    // Still there after API call
    await waitFor(() => {
      expect(getByText('New todo')).toBeInTheDocument();
    });
  });
});
```

### Testing Infinite Scroll

```typescript
describe('Infinite Scroll', () => {
  it('loads more items on scroll', async () => {
    const { container } = renderWithClient(<InfiniteList />);

    // Initial items loaded
    await waitFor(() => {
      expect(screen.getAllByRole('listitem')).toHaveLength(20);
    });

    // Scroll to bottom
    const scrollContainer = container.querySelector('.scroll-container')!;
    fireEvent.scroll(scrollContainer, {
      target: { scrollY: scrollContainer.scrollHeight }
    });

    // More items loaded
    await waitFor(() => {
      expect(screen.getAllByRole('listitem')).toHaveLength(40);
    });
  });

  it('shows loading indicator while fetching', async () => {
    const { container } = renderWithClient(<InfiniteList />);

    await waitFor(() => {
      expect(screen.getAllByRole('listitem')).toHaveLength(20);
    });

    const scrollContainer = container.querySelector('.scroll-container')!;
    fireEvent.scroll(scrollContainer, {
      target: { scrollY: scrollContainer.scrollHeight }
    });

    // Loading indicator appears
    expect(screen.getByRole('status')).toBeInTheDocument();

    // Then disappears
    await waitFor(() => {
      expect(screen.queryByRole('status')).not.toBeInTheDocument();
    });
  });
});
```

## 14.6 Visual Regression Testing

### Using Playwright Screenshots

```typescript
// e2e/visual.spec.ts
test.describe('Visual Regression', () => {
  test('homepage looks correct', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveScreenshot('homepage.png');
  });

  test('dark mode theme', async ({ page }) => {
    await page.goto('/');
    await page.click('[aria-label="Toggle dark mode"]');

    await expect(page).toHaveScreenshot('homepage-dark.png');
  });

  test('responsive mobile view', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');

    await expect(page).toHaveScreenshot('homepage-mobile.png');
  });

  test('component variations', async ({ page }) => {
    await page.goto('/components/buttons');

    // Test different states
    await expect(
      page.locator('.button-primary')
    ).toHaveScreenshot('button-primary.png');

    await page.hover('.button-primary');
    await expect(
      page.locator('.button-primary')
    ).toHaveScreenshot('button-primary-hover.png');
  });
});
```

## Real-World Scenario: Testing E-commerce Checkout

### The Challenge

Test a complex checkout flow:
- Multi-step wizard
- Payment processing
- Inventory checks
- Email notifications
- Error handling

### Junior Approach

```typescript
// Shallow, incomplete tests
test('checkout works', async () => {
  render(<Checkout />);
  fireEvent.click(screen.getByText('Buy Now'));
  expect(screen.getByText('Success')).toBeInTheDocument();
});
```

### Senior Approach

```typescript
describe('E-commerce Checkout Flow', () => {
  describe('Happy Path', () => {
    it('completes full checkout process', async () => {
      // Setup: Add items to cart
      const { getByRole, getByLabelText } = renderWithClient(
        <App />,
        { initialCart: [{ id: 1, quantity: 2 }] }
      );

      // Step 1: Navigate to checkout
      await userEvent.click(getByRole('link', { name: /cart/i }));
      await userEvent.click(getByRole('button', { name: /checkout/i }));

      // Step 2: Shipping information
      await userEvent.type(getByLabelText('Full Name'), 'John Doe');
      await userEvent.type(getByLabelText('Address'), '123 Main St');
      await userEvent.type(getByLabelText('City'), 'New York');
      await userEvent.selectOptions(getByLabelText('State'), 'NY');
      await userEvent.type(getByLabelText('ZIP'), '10001');
      await userEvent.click(getByRole('button', { name: /continue/i }));

      // Step 3: Payment
      await waitFor(() => {
        expect(getByLabelText('Card Number')).toBeInTheDocument();
      });

      await userEvent.type(getByLabelText('Card Number'), '4242424242424242');
      await userEvent.type(getByLabelText('Expiry'), '12/25');
      await userEvent.type(getByLabelText('CVV'), '123');

      // Step 4: Review and submit
      await userEvent.click(getByRole('button', { name: /review order/i }));

      await waitFor(() => {
        expect(screen.getByText('123 Main St')).toBeInTheDocument();
        expect(screen.getByText('****4242')).toBeInTheDocument();
      });

      await userEvent.click(getByRole('button', { name: /place order/i }));

      // Verify success
      await waitFor(() => {
        expect(screen.getByText(/order confirmed/i)).toBeInTheDocument();
        expect(screen.getByText(/order #/i)).toBeInTheDocument();
      });

      // Verify cart is cleared
      expect(screen.getByText('0 items')).toBeInTheDocument();
    });
  });

  describe('Error Scenarios', () => {
    it('handles payment decline', async () => {
      server.use(
        rest.post('/api/payment', (req, res, ctx) => {
          return res(
            ctx.status(402),
            ctx.json({ error: 'Card declined' })
          );
        })
      );

      // ... fill out form ...

      await userEvent.click(getByRole('button', { name: /place order/i }));

      await waitFor(() => {
        expect(screen.getByText(/card declined/i)).toBeInTheDocument();
      });

      // User can try again
      expect(getByRole('button', { name: /try again/i })).toBeEnabled();
    });

    it('handles out of stock during checkout', async () => {
      server.use(
        rest.post('/api/checkout', (req, res, ctx) => {
          return res(
            ctx.status(409),
            ctx.json({
              error: 'Item out of stock',
              itemId: 1
            })
          );
        })
      );

      // ... complete checkout ...

      await waitFor(() => {
        expect(screen.getByText(/out of stock/i)).toBeInTheDocument();
      });

      // Item removed from cart
      expect(screen.queryByText('Item 1')).not.toBeInTheDocument();
    });

    it('recovers from network errors', async () => {
      let attempts = 0;

      server.use(
        rest.post('/api/checkout', (req, res, ctx) => {
          attempts++;
          if (attempts === 1) {
            return res.networkError('Network error');
          }
          return res(ctx.json({ orderId: '123' }));
        })
      );

      // ... complete checkout ...

      // First attempt fails
      await waitFor(() => {
        expect(screen.getByText(/network error/i)).toBeInTheDocument();
      });

      // Retry succeeds
      await userEvent.click(getByRole('button', { name: /retry/i }));

      await waitFor(() => {
        expect(screen.getByText(/order confirmed/i)).toBeInTheDocument();
      });
    });
  });

  describe('Validation', () => {
    it('validates shipping information', async () => {
      const { getByRole } = renderWithClient(<Checkout />);

      await userEvent.click(getByRole('button', { name: /continue/i }));

      expect(await screen.findByText(/full name is required/i))
        .toBeInTheDocument();
      expect(screen.getByText(/address is required/i)).toBeInTheDocument();
    });

    it('validates payment information', async () => {
      // ... navigate to payment step ...

      await userEvent.type(getByLabelText('Card Number'), '1234'); // Invalid
      await userEvent.click(getByRole('button', { name: /review/i }));

      expect(await screen.findByText(/invalid card number/i))
        .toBeInTheDocument();
    });
  });
});
```

## Chapter Exercise: Build Complete Test Suite

Create a comprehensive test suite for a feature:

**Requirements:**
1. Unit tests for individual components
2. Integration tests for component interactions
3. API integration tests with MSW
4. E2E tests for critical user flows
5. Visual regression tests
6. Error scenario coverage
7. Performance tests (loading states, timeouts)

**Evaluation Criteria:**
- Test coverage > 80%
- All critical paths tested
- Error scenarios handled
- Tests are maintainable
- Fast execution time
- No flaky tests

## Review Checklist

- [ ] Integration tests cover component interactions
- [ ] E2E tests cover critical user journeys
- [ ] API mocking with MSW
- [ ] Async operations tested properly
- [ ] Error scenarios covered
- [ ] Visual regression tests for UI changes
- [ ] Page Object Model for maintainability
- [ ] Fast, reliable test execution

## Key Takeaways

1. **Integration tests catch real bugs** - Unit tests aren't enough
2. **E2E tests verify user journeys** - Test what users actually do
3. **Mock external dependencies** - Use MSW for API mocking
4. **Test async carefully** - Handle loading, errors, race conditions
5. **Page Objects improve maintainability** - DRY principle for tests
6. **Visual regression prevents UI bugs** - Automated screenshot comparison
7. **Balance coverage and speed** - Optimize test execution

## Further Reading

- Playwright documentation
- React Testing Library best practices
- MSW (Mock Service Worker) guides
- Kent C. Dodds: Testing Implementation Details
- Martin Fowler: Test Pyramid

## Next Chapter

[Chapter 15: Testing Complex Interactions](./15-testing-complex-interactions.md)
