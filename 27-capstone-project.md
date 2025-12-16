# Chapter 27: Capstone Project - Building a Production-Ready SaaS Application

## Introduction

This is where everything comes together. You'll build a complete, production-ready application that demonstrates senior-level skills.

This is not a toy project. This is a real application you could deploy and use.

## Project Overview: TaskFlow - Team Task Management

Build a collaborative task management SaaS application with real-time updates, team collaboration, and enterprise features.

### Why This Project?

This project requires you to demonstrate:
- Complex state management
- Real-time collaboration
- Performance optimization
- Proper architecture
- Security best practices
- Testing strategies
- DevOps knowledge

### Demo Video Target

Your finished project should have:
- Polished UI/UX
- Smooth interactions
- Fast performance
- No obvious bugs
- Professional code quality

## Core Features

### Phase 1: Foundation (Week 1-2)

**Authentication & Authorization**
- Email/password registration
- Google OAuth
- Email verification
- Password reset
- Role-based access (Owner, Admin, Member)
- Session management

**Project Management**
- Create/edit/delete projects
- Project settings
- Archive projects
- Project roles

**Task Management**
- Create/edit/delete tasks
- Task descriptions (rich text)
- Due dates
- Assignees
- Labels/tags
- Task status workflow

**Technical Requirements:**
- Proper authentication flow
- Protected routes
- Secure API endpoints
- Form validation
- Error handling

### Phase 2: Collaboration (Week 3)

**Real-Time Updates**
- See when others edit tasks
- Live cursor positions
- Online/offline status
- Activity feed

**Comments & Discussion**
- Comment on tasks
- @mentions
- Comment threading
- Rich text support
- Edit history

**Team Management**
- Invite team members
- Manage permissions
- Remove members
- Transfer ownership

**Technical Requirements:**
- WebSocket connection
- Optimistic updates
- Conflict resolution
- Connection resilience

### Phase 3: Advanced Features (Week 4)

**Search & Filtering**
- Full-text search
- Filter by assignee, status, labels
- Saved filters
- Keyboard shortcuts

**Notifications**
- In-app notifications
- Email notifications (optional)
- Notification preferences
- Mark as read/unread

**Analytics Dashboard**
- Task completion trends
- Team productivity metrics
- Burndown charts
- Time tracking (bonus)

**Performance**
- Virtualized lists for 1000+ tasks
- Optimized re-renders
- Code splitting
- Image optimization
- Bundle analysis

**Technical Requirements:**
- Search implementation (client or server)
- Chart library integration
- Performance monitoring
- Analytics events

## Architecture Requirements

### Frontend Architecture

```
src/
├── app/                          # Next.js app router (or pages)
│   ├── (auth)/
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   ├── projects/
│   │   └── settings/
│   └── api/                      # API routes
│
├── features/                     # Feature modules
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   ├── types/
│   │   └── index.ts              # Public API
│   │
│   ├── projects/
│   ├── tasks/
│   ├── comments/
│   ├── notifications/
│   └── analytics/
│
├── shared/                       # Shared code
│   ├── components/               # UI components
│   │   ├── ui/                   # Base components
│   │   ├── forms/
│   │   └── layout/
│   │
│   ├── hooks/                    # Generic hooks
│   │   ├── useLocalStorage.ts
│   │   ├── useDebounce.ts
│   │   ├── useWebSocket.ts
│   │   └── useKeyboard.ts
│   │
│   ├── lib/                      # Utilities
│   │   ├── api-client.ts
│   │   ├── validation.ts
│   │   └── formatters.ts
│   │
│   └── types/                    # Shared types
│
├── styles/                       # Global styles
└── tests/                        # Test utilities
    ├── setup.ts
    ├── mocks/
    └── utils/
```

### State Management

Choose your approach and justify it in an ADR:
- Zustand for app state?
- React Query for server state?
- Context for theme/auth?
- URL state for filters?

### Backend (Choose One)

**Option A: Next.js API Routes**
- Simplest setup
- Collocated with frontend
- Limited by Vercel functions

**Option B: Express/Fastify**
- More control
- Better WebSocket support
- Separate deployment

**Option C: Supabase/Firebase**
- Fastest to production
- Real-time built-in
- Less control

Document your choice in ADR-001.

### Database

**Schema Design:**

```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  name VARCHAR(255) NOT NULL,
  avatar_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Projects table
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  owner_id UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Project members table
CREATE TABLE project_members (
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(50) NOT NULL, -- 'owner', 'admin', 'member'
  joined_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (project_id, user_id)
);

-- Tasks table
CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  title VARCHAR(500) NOT NULL,
  description TEXT,
  status VARCHAR(50) NOT NULL, -- 'todo', 'in_progress', 'done'
  assignee_id UUID REFERENCES users(id),
  due_date TIMESTAMP,
  position INTEGER NOT NULL,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Comments table
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id),
  content TEXT NOT NULL,
  parent_id UUID REFERENCES comments(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Add more tables as needed...
```

## Technical Implementation Guide

### 1. Authentication System

```typescript
// features/auth/hooks/useAuth.ts
export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check for existing session
    checkSession();
  }, []);

  const login = async (email: string, password: string) => {
    const response = await api.post('/auth/login', { email, password });
    setUser(response.user);
    // Store token
  };

  const logout = async () => {
    await api.post('/auth/logout');
    setUser(null);
  };

  return { user, loading, login, logout };
}

// features/auth/components/ProtectedRoute.tsx
export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth();

  if (loading) return <LoadingSpinner />;
  if (!user) return <Navigate to="/login" />;

  return <>{children}</>;
}
```

### 2. Real-Time WebSocket Setup

```typescript
// shared/hooks/useWebSocket.ts
export function useWebSocket<T>(url: string) {
  const [messages, setMessages] = useState<T[]>([]);
  const ws = useRef<WebSocket | null>(null);

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages(prev => [...prev, message]);
    };

    ws.current.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    return () => {
      ws.current?.close();
    };
  }, [url]);

  const send = (message: T) => {
    if (ws.current?.readyState === WebSocket.OPEN) {
      ws.current.send(JSON.stringify(message));
    }
  };

  return { messages, send };
}

// features/tasks/hooks/useRealtimeTasks.ts
export function useRealtimeTasks(projectId: string) {
  const { messages } = useWebSocket<TaskUpdate>(
    `wss://api.example.com/projects/${projectId}/tasks`
  );

  const queryClient = useQueryClient();

  useEffect(() => {
    messages.forEach(update => {
      // Optimistically update cache
      queryClient.setQueryData(['tasks', projectId], (old: Task[]) =>
        old.map(task =>
          task.id === update.taskId
            ? { ...task, ...update.changes }
            : task
        )
      );
    });
  }, [messages, projectId, queryClient]);
}
```

### 3. Optimistic Updates

```typescript
// features/tasks/hooks/useUpdateTask.ts
export function useUpdateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (update: TaskUpdate) =>
      api.patch(`/tasks/${update.id}`, update),

    // Optimistic update
    onMutate: async (update) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries(['tasks']);

      // Snapshot previous value
      const previous = queryClient.getQueryData(['tasks']);

      // Optimistically update
      queryClient.setQueryData(['tasks'], (old: Task[]) =>
        old.map(task =>
          task.id === update.id
            ? { ...task, ...update }
            : task
        )
      );

      return { previous };
    },

    // Rollback on error
    onError: (err, update, context) => {
      queryClient.setQueryData(['tasks'], context.previous);
      toast.error('Failed to update task');
    },

    // Refetch on success
    onSuccess: () => {
      queryClient.invalidateQueries(['tasks']);
    }
  });
}
```

### 4. Performance Optimization

```typescript
// features/tasks/components/TaskList.tsx
import { FixedSizeList } from 'react-window';

export function TaskList({ tasks }: { tasks: Task[] }) {
  const Row = memo(({ index, style }: { index: number; style: React.CSSProperties }) => {
    const task = tasks[index];
    return (
      <div style={style}>
        <TaskItem task={task} />
      </div>
    );
  });

  return (
    <FixedSizeList
      height={600}
      itemCount={tasks.length}
      itemSize={80}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// Memoize expensive computations
export function TaskStats({ tasks }: { tasks: Task[] }) {
  const stats = useMemo(() => ({
    total: tasks.length,
    completed: tasks.filter(t => t.status === 'done').length,
    overdue: tasks.filter(t => isOverdue(t)).length
  }), [tasks]);

  return <StatsDisplay stats={stats} />;
}
```

## Testing Requirements

### Unit Tests (70% coverage minimum)

```typescript
// features/tasks/hooks/useTaskFilters.test.ts
describe('useTaskFilters', () => {
  it('filters tasks by status', () => {
    const tasks = [
      { id: '1', status: 'todo', title: 'Task 1' },
      { id: '2', status: 'done', title: 'Task 2' }
    ];

    const { result } = renderHook(() =>
      useTaskFilters(tasks, { status: 'todo' })
    );

    expect(result.current.filteredTasks).toHaveLength(1);
    expect(result.current.filteredTasks[0].id).toBe('1');
  });

  it('filters tasks by assignee', () => {
    // ... more tests
  });
});
```

### Integration Tests

```typescript
// features/tasks/TaskBoard.test.tsx
describe('TaskBoard Integration', () => {
  it('user can create, edit, and delete task', async () => {
    render(<TaskBoard projectId="123" />);

    // Create task
    await userEvent.click(screen.getByRole('button', { name: /new task/i }));
    await userEvent.type(screen.getByLabelText(/title/i), 'New Task');
    await userEvent.click(screen.getByRole('button', { name: /create/i }));

    expect(await screen.findByText('New Task')).toBeInTheDocument();

    // Edit task
    await userEvent.click(screen.getByText('New Task'));
    await userEvent.clear(screen.getByLabelText(/title/i));
    await userEvent.type(screen.getByLabelText(/title/i), 'Updated Task');
    await userEvent.click(screen.getByRole('button', { name: /save/i }));

    expect(await screen.findByText('Updated Task')).toBeInTheDocument();

    // Delete task
    await userEvent.click(screen.getByRole('button', { name: /delete/i }));
    await userEvent.click(screen.getByRole('button', { name: /confirm/i }));

    expect(screen.queryByText('Updated Task')).not.toBeInTheDocument();
  });
});
```

### E2E Tests (Critical paths only)

```typescript
// e2e/task-management.spec.ts
test('complete task management flow', async ({ page }) => {
  // Login
  await page.goto('/login');
  await page.fill('[name=email]', 'test@example.com');
  await page.fill('[name=password]', 'password');
  await page.click('button[type=submit]');

  // Navigate to project
  await page.click('text=My Project');

  // Create task
  await page.click('button:has-text("New Task")');
  await page.fill('[name=title]', 'Complete project');
  await page.click('button:has-text("Create")');

  // Verify task appears
  await expect(page.locator('text=Complete project')).toBeVisible();

  // Mark as complete
  await page.click('text=Complete project');
  await page.selectOption('[name=status]', 'done');

  // Verify completion
  await expect(page.locator('[data-status=done]')).toContainText('Complete project');
});
```

## Evaluation Criteria

### Architecture (25%)
- [ ] Clear separation of concerns
- [ ] Feature-based organization
- [ ] Proper layering
- [ ] No circular dependencies
- [ ] Well-defined module boundaries
- [ ] ADRs for major decisions

### Code Quality (25%)
- [ ] TypeScript used effectively
- [ ] Consistent code style
- [ ] No code smells
- [ ] Proper error handling
- [ ] Meaningful variable names
- [ ] Comments where needed

### Performance (15%)
- [ ] First Contentful Paint < 1.5s
- [ ] Time to Interactive < 3s
- [ ] Lighthouse score > 90
- [ ] No unnecessary re-renders
- [ ] Optimized bundle size
- [ ] Lazy loading implemented

### Testing (15%)
- [ ] 70%+ code coverage
- [ ] Tests are maintainable
- [ ] Integration tests for features
- [ ] E2E tests for critical paths
- [ ] No flaky tests

### Security (10%)
- [ ] Input validation
- [ ] CSRF protection
- [ ] XSS prevention
- [ ] Secure authentication
- [ ] Environment variables
- [ ] No secrets in code

### UX/UI (10%)
- [ ] Responsive design
- [ ] Loading states
- [ ] Error states
- [ ] Smooth animations
- [ ] Keyboard navigation
- [ ] Accessible (WCAG AA)

## Bonus Challenges

### Advanced Features (Optional)
- [ ] Drag and drop task reordering
- [ ] Kanban board view
- [ ] Calendar view
- [ ] File attachments
- [ ] Dark mode
- [ ] Keyboard shortcuts
- [ ] Undo/redo
- [ ] Offline support
- [ ] Email notifications
- [ ] Slack integration

### DevOps (Bonus)
- [ ] CI/CD pipeline
- [ ] Automated tests in CI
- [ ] Preview deployments
- [ ] Error monitoring (Sentry)
- [ ] Analytics (Plausible/Umami)
- [ ] Performance monitoring
- [ ] Docker containerization

## Deliverables

### 1. Working Application
- Deployed to production (Vercel, Netlify, etc.)
- Public demo available
- Demo credentials provided

### 2. Documentation
```
docs/
├── README.md                  # Project overview
├── SETUP.md                   # Local development setup
├── ARCHITECTURE.md            # Architecture decisions
├── API.md                     # API documentation
└── TESTING.md                 # Testing approach
```

### 3. Architecture Decision Records
```
docs/adr/
├── 001-state-management.md
├── 002-real-time-approach.md
├── 003-database-choice.md
└── 004-authentication.md
```

### 4. Presentation
Prepare a 10-minute presentation covering:
- Problem you solved
- Architecture overview
- Key technical decisions
- Challenges and solutions
- What you learned

## Timeline

**Week 1: Foundation**
- Project setup
- Authentication
- Basic CRUD operations
- Database schema

**Week 2: Core Features**
- Task management
- Project management
- UI polish

**Week 3: Advanced Features**
- Real-time updates
- Search and filtering
- Notifications

**Week 4: Polish & Deploy**
- Performance optimization
- Testing
- Documentation
- Deployment

## Success Metrics

You've succeeded when:
- [ ] Application is deployed and works
- [ ] You can demo without encountering bugs
- [ ] Code review from a senior dev is positive
- [ ] You understand every technical decision
- [ ] You can explain trade-offs you made
- [ ] Tests give you confidence to refactor
- [ ] You're proud to show this in interviews

## Reflection Questions

After completing the project:

1. **Technical Decisions**
   - What would you do differently?
   - What decisions proved correct?
   - What surprised you?

2. **Challenges**
   - What was the hardest part?
   - How did you overcome obstacles?
   - What would you research more?

3. **Growth**
   - What new skills did you develop?
   - Where do you still feel weak?
   - What's your next learning goal?

4. **Production Readiness**
   - Is this actually production-ready?
   - What would you need to add for 1000 users?
   - What about 100,000 users?

## Final Thoughts

This project is not about perfection. It's about demonstrating:
- You can build complete features
- You make thoughtful technical decisions
- You write maintainable code
- You think about users and performance
- You can ship

When you're done, you'll have:
- A portfolio project
- Talking points for interviews
- Real experience with production concerns
- Confidence in your abilities

Now go build something great!

## Next Steps

[Chapter 28: Your Senior Developer Roadmap](./28-roadmap.md)
