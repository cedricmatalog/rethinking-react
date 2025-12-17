# Chapter 2: Advanced Component Patterns

## Introduction

Junior developers write components that work. Senior developers write components that scale, compose, and adapt to changing requirements without breaking.

This chapter covers battle-tested patterns used in production applications and libraries.

## Learning Objectives

- Master compound components pattern
- Understand render props and their use cases
- Implement flexible APIs with props getters
- Build extensible components with control props
- Know when to use each pattern

## 2.1 Compound Components

### Junior Perspective
"I pass everything as props - title, content, footer. Simple!"

### Senior Perspective
"Compound components use React Context to share state between parent and children, creating flexible, composable APIs that scale with complexity."

### The Problem: Prop Explosion

Imagine you're building a Modal component. The junior approach seems simple at first:

```javascript
// Junior approach - starts simple...
function Modal({ title, content, footer }) {
  return (
    <div className="modal">
      <div className="modal-header">{title}</div>
      <div className="modal-content">{content}</div>
      <div className="modal-footer">{footer}</div>
    </div>
  );
}

// Usage is limited
<Modal
  title="Delete Item"
  content="Are you sure?"
  footer={<button>Delete</button>}
/>
```

**But then requirements change:**

"We need a close button in the header"
→ Add `showCloseButton` prop

"Sometimes we need an icon in the header"
→ Add `headerIcon` prop

"The content needs custom padding sometimes"
→ Add `contentClassName` prop

"We need to support multiple footer buttons with different layouts"
→ Add `footerButtons` array prop... wait, how do we handle click handlers?

**After 6 months:**

```javascript
<Modal
  title="Delete Item"
  headerIcon={<WarningIcon />}
  showCloseButton={true}
  onClose={handleClose}
  content={<CustomContent />}
  contentClassName="custom-padding"
  footer={[
    { text: 'Cancel', onClick: handleCancel, variant: 'secondary' },
    { text: 'Delete', onClick: handleDelete, variant: 'danger' }
  ]}
  footerAlign="right"
  size="medium"
  closeOnOverlayClick={true}
  closeOnEsc={true}
/>
```

**The problem:** Every customization needs a new prop. The API becomes rigid and hard to use.

**Visual representation:**

```
Junior Approach (Prop Drilling):
┌─────────────────────────────────────┐
│          <Modal />                  │
│                                     │
│  Props in: {                        │
│    title,                           │
│    content,                         │
│    footer,                          │
│    headerIcon,                      │
│    showCloseButton,                 │
│    contentClassName,                │
│    footerButtons,                   │
│    footerAlign,                     │
│    size,                            │
│    ... 15 more props                │
│  }                                  │
│                                     │
│  ├─> Modal renders all parts        │
│  └─> Users have no control          │
└─────────────────────────────────────┘
```

### Senior Pattern: Compound Components

**Visual representation:**

```
Compound Components (Composition):
┌──────────────────────────────────────────────┐
│         <Modal isOpen onClose>               │
│                                              │
│   Context provides: { isOpen, onClose }     │
│                ↓                             │
│   ┌────────────────────────────┐            │
│   │   <Modal.Header>            │            │
│   │     {children}              │ ← Full control
│   │     <CloseButton />         │            │
│   └────────────────────────────┘            │
│                ↓                             │
│   ┌────────────────────────────┐            │
│   │   <Modal.Body>              │            │
│   │     <YourComponent />       │ ← Any JSX!
│   │     <AnotherComponent />    │            │
│   └────────────────────────────┘            │
│                ↓                             │
│   ┌────────────────────────────┐            │
│   │   <Modal.Footer>            │            │
│   │     <button>Cancel</button> │ ← Compose freely
│   │     <button>Delete</button> │            │
│   └────────────────────────────┘            │
│                                              │
│  Each child can:                             │
│  ✅ Access shared state via Context          │
│  ✅ Render any children                      │
│  ✅ Be used or omitted                       │
│  ✅ Be reordered                             │
└──────────────────────────────────────────────┘
```

**The implementation:**

```javascript
// Context for internal state sharing
const ModalContext = createContext();

function Modal({ children, isOpen, onClose }) {
  const value = { isOpen, onClose };

  if (!isOpen) return null;

  return (
    <ModalContext.Provider value={value}>
      <div className="modal-overlay" onClick={onClose}>
        <div className="modal" onClick={e => e.stopPropagation()}>
          {children}
        </div>
      </div>
    </ModalContext.Provider>
  );
}

Modal.Header = function ModalHeader({ children }) {
  const { onClose } = useContext(ModalContext);
  return (
    <div className="modal-header">
      {children}
      <button onClick={onClose}>×</button>
    </div>
  );
};

Modal.Body = function ModalBody({ children }) {
  return <div className="modal-body">{children}</div>;
};

Modal.Footer = function ModalFooter({ children }) {
  return <div className="modal-footer">{children}</div>;
};

// Flexible usage
<Modal isOpen={isOpen} onClose={close}>
  <Modal.Header>
    <h2>Delete Item</h2>
  </Modal.Header>
  <Modal.Body>
    <p>Are you sure you want to delete this item?</p>
    <WarningMessage />
  </Modal.Body>
  <Modal.Footer>
    <button onClick={close}>Cancel</button>
    <button onClick={handleDelete}>Delete</button>
  </Modal.Footer>
</Modal>
```

### Benefits
- Flexible composition
- Clear hierarchy
- Shared implicit state
- Better API discoverability

### Exercise 2.1: Building Compound Components

**Challenge:** Build a compound component system for a `Tabs` component that demonstrates the power of composition.

**Requirements:**
1. Components: `Tabs`, `TabList`, `Tab`, `TabPanels`, `TabPanel`
2. Shared state for active tab (using Context)
3. Keyboard navigation (arrow keys to switch tabs)
4. Support both controlled and uncontrolled usage
5. Accessible (ARIA attributes)

**Example Usage:**
```javascript
<Tabs defaultIndex={0}>
  <TabList>
    <Tab>Profile</Tab>
    <Tab>Settings</Tab>
    <Tab>Notifications</Tab>
  </TabList>
  <TabPanels>
    <TabPanel><Profile /></TabPanel>
    <TabPanel><Settings /></TabPanel>
    <TabPanel><Notifications /></TabPanel>
  </TabPanels>
</Tabs>
```

**Think About:**
- How will `Tab` components know their index?
- How will `TabPanel` components know if they should render?
- What state needs to be shared via Context?
- How do you prevent users from using `Tab` outside of `TabList`?

<details>
<summary>💡 Hint: Getting started</summary>

**Start with the Context:**
1. Create `TabsContext` with `{ activeIndex, setActiveIndex }`
2. The `Tabs` component provides this context
3. Each child component consumes what it needs

**For automatic indexing:**
- Use `React.Children.map` to assign indices to `Tab` and `TabPanel` components
- Or use a registration pattern where children register themselves

**For keyboard navigation:**
- Add `onKeyDown` handler to `TabList`
- Handle `ArrowLeft` and `ArrowRight` keys
- Focus the newly active tab

</details>

<details>
<summary>✅ Solution</summary>

```javascript
import { createContext, useContext, useState, Children, cloneElement } from 'react';

// 1. Create Context
const TabsContext = createContext();

function useTabs() {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('Tabs compounds must be used within <Tabs>');
  }
  return context;
}

// 2. Main Tabs component
function Tabs({ children, defaultIndex = 0, index: controlledIndex, onChange }) {
  const [uncontrolledIndex, setUncontrolledIndex] = useState(defaultIndex);

  // Support controlled/uncontrolled
  const isControlled = controlledIndex !== undefined;
  const activeIndex = isControlled ? controlledIndex : uncontrolledIndex;

  const setActiveIndex = (newIndex) => {
    onChange?.(newIndex);
    if (!isControlled) {
      setUncontrolledIndex(newIndex);
    }
  };

  const value = { activeIndex, setActiveIndex };

  return (
    <TabsContext.Provider value={value}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// 3. TabList component
Tabs.TabList = function TabList({ children }) {
  const { activeIndex, setActiveIndex } = useTabs();

  const handleKeyDown = (e) => {
    const count = Children.count(children);

    if (e.key === 'ArrowRight') {
      e.preventDefault();
      setActiveIndex((activeIndex + 1) % count);
    } else if (e.key === 'ArrowLeft') {
      e.preventDefault();
      setActiveIndex((activeIndex - 1 + count) % count);
    }
  };

  return (
    <div role="tablist" onKeyDown={handleKeyDown} className="tab-list">
      {Children.map(children, (child, index) =>
        cloneElement(child, { index })
      )}
    </div>
  );
};

// 4. Tab component
Tabs.Tab = function Tab({ children, index }) {
  const { activeIndex, setActiveIndex } = useTabs();
  const isActive = activeIndex === index;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      tabIndex={isActive ? 0 : -1}
      onClick={() => setActiveIndex(index)}
      className={isActive ? 'tab tab-active' : 'tab'}
    >
      {children}
    </button>
  );
};

// 5. TabPanels component
Tabs.TabPanels = function TabPanels({ children }) {
  return (
    <div className="tab-panels">
      {Children.map(children, (child, index) =>
        cloneElement(child, { index })
      )}
    </div>
  );
};

// 6. TabPanel component
Tabs.TabPanel = function TabPanel({ children, index }) {
  const { activeIndex } = useTabs();

  if (index !== activeIndex) return null;

  return (
    <div role="tabpanel" className="tab-panel">
      {children}
    </div>
  );
};

export default Tabs;
```

**Key Observations:**
- Context eliminates prop drilling - `Tab` and `TabPanel` access shared state directly
- `useTabs` hook enforces that compounds are used correctly
- `React.Children.map` + `cloneElement` automatically assigns indices
- Keyboard navigation is centralized in `TabList`
- Supports both controlled (`index` + `onChange`) and uncontrolled (`defaultIndex`) usage

**What You Learned:**
- Compound components share state through Context
- Sub-components can be simple - complexity lives in the parent
- `cloneElement` allows injecting props into children
- Custom hooks can enforce usage rules
- Flexible APIs support multiple use cases

</details>

<details>
<summary>📚 Deep Dive: Why not just use props?</summary>

**Why compound components are better than props:**

```javascript
// Props approach - rigid
<Tabs
  tabs={[
    { label: 'Profile', content: <Profile /> },
    { label: 'Settings', content: <Settings /> }
  ]}
/>
// Problem: Can't add icons, badges, or custom styling to tabs

// Compound approach - flexible
<Tabs>
  <TabList>
    <Tab><UserIcon /> Profile <Badge>3</Badge></Tab>
    <Tab><SettingsIcon /> Settings</Tab>
  </TabList>
  <TabPanels>
    <TabPanel><Profile /></TabPanel>
    <TabPanel><Settings /></TabPanel>
  </TabPanels>
</Tabs>
// ✅ Full control over rendering
```

**When to use compound components:**
- UI components with multiple related parts (Tabs, Accordion, Select)
- Need for flexible composition
- Shared state between parts
- Library/design system components

**When NOT to use:**
- Simple components with no shared state
- When prop-based API is simpler
- Performance-critical components (Context can cause re-renders)

</details>

---

## 2.2 Render Props Pattern

**🧠 Quick Recall (from 2.1):** Before we dive in, test your retention: What problem do compound components solve? How do child components access shared state from the parent?

<details>
<summary>Check your answer</summary>

**Answer:** Compound components solve prop explosion - instead of passing 15+ props for customization, you give users full control through composition.

**How state is shared:** Through React Context. The parent component (`<Tabs>`) provides context, and child components (`<Tab>`, `<TabPanel>`) consume it using `useContext`.

**Example:**
```javascript
// Parent provides
<TabsContext.Provider value={{ activeIndex, setActiveIndex }}>

// Children consume
const { activeIndex } = useContext(TabsContext);
```

Good! Now you're ready to learn about render props, another pattern for code reuse.
</details>

---

### The Problem: Sharing Logic

```javascript
// Junior: Copy-paste logic across components
function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser().then(setUser).finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  return <div>{user.name}</div>;
}

function UserSettings() {
  // Same logic repeated
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser().then(setUser).finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  return <div>{/* settings */}</div>;
}
```

### Senior Pattern: Render Props

```javascript
// Reusable data fetching logic
function DataFetcher({ url, render, renderLoading, renderError }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  if (loading) return renderLoading();
  if (error) return renderError(error);
  return render(data);
}

// Clean usage
function UserProfile() {
  return (
    <DataFetcher
      url="/api/user"
      renderLoading={() => <Spinner />}
      renderError={(error) => <ErrorBanner error={error} />}
      render={(user) => <div>{user.name}</div>}
    />
  );
}
```

### Modern Alternative: Custom Hooks

```javascript
// Even better - custom hook
function useDataFetcher(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) setData(data);
      })
      .catch(err => {
        if (!cancelled) setError(err);
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile() {
  const { data: user, loading, error } = useDataFetcher('/api/user');

  if (loading) return <Spinner />;
  if (error) return <ErrorBanner error={error} />;
  return <div>{user.name}</div>;
}
```

### When to Use Each

| Render Props | Custom Hooks |
|-------------|--------------|
| Component needs JSX injection points | Pure logic sharing |
| Multiple render variations | Single render output |
| Working with legacy code | Modern codebases |
| Library development | Application code |

---

## 💥 Real War Story: The Over-Engineered Component Library

**Company:** SaaS platform (200+ developers, 15 product teams)
**Date:** Q2 2022
**Pattern Misuse:** Compound components used for EVERYTHING

### What Happened

The design system team built a component library using compound components for every single component - even simple ones.

```javascript
// Their Button component - seriously
<Button variant="primary" size="large">
  <Button.Icon><CheckIcon /></Button.Icon>
  <Button.Text>Save Changes</Button.Text>
  <Button.LoadingSpinner />
</Button>

// vs what it should have been
<Button variant="primary" size="large" icon={<CheckIcon />}>
  Save Changes
</Button>
```

**Why they did it:**
- "Compound components are a senior pattern!"
- "It's more flexible!"
- "Other libraries do it this way!"

### The Impact

**Developer Experience Disaster:**
- Simple buttons required 5-10 lines of code
- New devs couldn't figure out basic usage
- Every team built wrapper components to simplify the API
- Documentation was 50+ pages for basic components

**Performance Impact:**
- Every sub-component had its own Context consumer
- Simple `<Button>` had 4 Context reads for 4 sub-components
- 3,247 unnecessary re-renders found in production profiling
- Page load times increased by 200-400ms

**Maintenance Nightmare:**
- Breaking changes in every minor release
- 89 open GitHub issues about "too complex"
- Teams forked the library and made their own simpler versions

### The 1-Week Refactor

They rewrote the library following this principle:

> "Use the simplest pattern that solves the problem. Compound components are for components with MULTIPLE RELATED PARTS that need SHARED STATE."

**New guidelines:**

```javascript
// ✅ Good use of compound components
<Select>
  <Select.Trigger>{selectedValue}</Select.Trigger>
  <Select.Options>
    <Select.Option value="1">Option 1</Select.Option>
    <Select.Option value="2">Option 2</Select.Option>
  </Select.Options>
</Select>
// Reason: Multiple parts, shared state (isOpen, selectedValue)

// ❌ Bad use of compound components
<Button>
  <Button.Text>Click me</Button.Text>
</Button>
// Reason: No shared state needed, just props!

// ✅ Simple prop-based API instead
<Button>Click me</Button>
```

**Results after refactor:**
- 80% reduction in component code
- Documentation went from 50 pages → 12 pages
- Zero complaints about complexity
- Performance improved (fewer Context reads)
- Adoption rate increased 3x

### Lessons Learned

From their post-mortem:

> "We cargo-culted a pattern we saw in popular libraries without understanding WHEN to use it. Just because a pattern is 'senior' doesn't mean it's right for every situation."

**What they changed:**
1. **Pattern selection checklist:**
   - Does this need shared state between parts? → Compound components
   - Does this need logic reuse? → Custom hook
   - Does this need render flexibility? → Render props
   - Is it just a simple component? → Props!

2. **Code review guidelines:**
   - "Justify compound components in PR description"
   - "Default to simplest pattern"
   - "If you need docs to explain basic usage, API is too complex"

3. **Documentation:**
   - Added "When to use this pattern" to every pattern doc
   - Created decision tree for pattern selection

---

**The Key Insight:**

```
Junior Developer:
"I learned compound components, so I'll use them everywhere!"

Senior Developer:
"I know compound components, render props, and custom hooks.
 For this Button, props are simplest. For this Select, compound
 components make sense because..."
```

**The rule:** Choose the pattern based on the problem, not based on what you just learned.

---

### Exercise 2.2: Render Props vs Custom Hooks

**Challenge:** Build a `<List>` component that handles data fetching and rendering with flexible state management.

**Requirements:**
1. Accepts array of data or a fetch URL
2. Handles loading, error, and empty states
3. Render props for: item, loading, empty, error
4. Then refactor to use a custom hook instead
5. Compare both approaches

**Think About:**
- When would render props be better than a custom hook?
- How do you handle the loading state in both patterns?
- Which approach feels more natural to use?

<details>
<summary>💡 Hint: Start with render props</summary>

**Structure:**
```javascript
<List
  data={items}
  renderItem={(item) => <div>{item.name}</div>}
  renderLoading={() => <Spinner />}
  renderEmpty={() => <EmptyState />}
  renderError={(error) => <ErrorBanner error={error} />}
/>
```

**For custom hook version:**
- Extract the logic to `useList(items)`
- Return `{ data, loading, error, isEmpty }`
- Let the component handle rendering

</details>

<details>
<summary>✅ Solution: Both Approaches</summary>

**Approach 1: Render Props**

```javascript
function List({ data, renderItem, renderLoading, renderEmpty, renderError }) {
  const [items, setItems] = useState(data);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // If data is a URL, fetch it
  useEffect(() => {
    if (typeof data === 'string') {
      setLoading(true);
      fetch(data)
        .then(res => res.json())
        .then(setItems)
        .catch(setError)
        .finally(() => setLoading(false));
    }
  }, [data]);

  if (loading) return renderLoading();
  if (error) return renderError(error);
  if (!items || items.length === 0) return renderEmpty();

  return (
    <ul>
      {items.map((item, index) => (
        <li key={item.id || index}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  data="/api/users"
  renderItem={(user) => <UserCard user={user} />}
  renderLoading={() => <Spinner />}
  renderEmpty={() => <div>No users found</div>}
  renderError={(err) => <div>Error: {err.message}</div>}
/>
```

**Approach 2: Custom Hook**

```javascript
function useList(dataOrUrl) {
  const [items, setItems] = useState(Array.isArray(dataOrUrl) ? dataOrUrl : []);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (typeof dataOrUrl === 'string') {
      setLoading(true);
      fetch(dataOrUrl)
        .then(res => res.json())
        .then(setItems)
        .catch(setError)
        .finally(() => setLoading(false));
    }
  }, [dataOrUrl]);

  return {
    items,
    loading,
    error,
    isEmpty: !items || items.length === 0
  };
}

// Usage - much cleaner!
function UserList() {
  const { items, loading, error, isEmpty } = useList('/api/users');

  if (loading) return <Spinner />;
  if (error) return <div>Error: {error.message}</div>;
  if (isEmpty) return <div>No users found</div>;

  return (
    <ul>
      {items.map(user => (
        <li key={user.id}>
          <UserCard user={user} />
        </li>
      ))}
    </ul>
  );
}
```

**Comparison:**

| Aspect | Render Props | Custom Hook |
|--------|-------------|-------------|
| **Boilerplate** | More (4 render functions) | Less (destructure result) |
| **Flexibility** | High (different renders per state) | Medium (component decides) |
| **Readability** | Verbose | Clean and linear |
| **Reusability** | Component-level | Logic-level |
| **Modern?** | Legacy pattern | Modern standard |

**When to use render props here:**
- If you need VERY different rendering for different use cases
- Building a library where users need maximum control

**When to use custom hook:**
- 99% of the time in modern React!
- Cleaner, more composable, easier to test

**Key Insight:** Custom hooks have mostly replaced render props for logic sharing. Use render props only when you truly need JSX injection points.

</details>

---

## ✅ Quick Knowledge Check: Render Props vs Hooks

**Question 1:** This render prop component has a performance issue. What's wrong?

```javascript
<DataFetcher
  url="/api/user"
  render={(data) => {
    const processed = expensiveProcess(data); // Heavy computation
    return <UserProfile data={processed} />;
  }}
/>
```

<details>
<summary>Show answer</summary>

**Problem:** `expensiveProcess` runs on EVERY render of the parent component, not just when data changes!

**Why:**
- Inline arrow function creates a new function reference each render
- DataFetcher might re-render even if `data` hasn't changed
- No memoization of the expensive computation

**Fix with custom hook:**
```javascript
function UserProfile() {
  const { data } = useDataFetcher('/api/user');
  const processed = useMemo(() => expensiveProcess(data), [data]);
  return <UserProfile data={processed} />;
}
```

**Lesson:** Hooks give you better control over optimizations (useMemo, useCallback).

</details>

---

**Question 2:** When is render props better than a custom hook?

<details>
<summary>Show answer</summary>

**Render props are better when:**

1. **Multiple JSX injection points needed:**
   ```javascript
   <InfiniteScroll
     renderItem={(item) => <Card data={item} />}
     renderLoading={() => <Skeleton />}
     renderEmpty={() => <EmptyState />}
     renderFooter={() => <LoadMoreButton />}
   />
   // Can't easily do this with hooks!
   ```

2. **Component library with maximum flexibility**
3. **Each usage needs radically different rendering**

**Custom hooks are better for:**
- Pure logic sharing
- Modern codebases
- Better performance optimization
- Cleaner code

**The trend:** 90% of use cases prefer custom hooks now.

</details>

---

## 2.3 Props Getters Pattern

**🧠 Quick Recall (from 2.2):** Before we continue, test your retention: What's the main advantage of custom hooks over render props? When might you still use render props?

<details>
<summary>Check your answer</summary>

**Advantage of custom hooks:**
- Cleaner, more readable code
- Better composability (can use multiple hooks)
- More idiomatic in modern React
- Easier to test

**When to still use render props:**
- Component needs multiple JSX injection points
- Building a library with maximum flexibility
- Working with legacy code

Now let's learn about props getters - a pattern that works great WITH custom hooks!
</details>

---

### The Problem: Complex Props Spreading

```javascript
// Junior: Manual prop spreading is error-prone
function Dropdown({ items }) {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState(0);

  return (
    <div>
      <button
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={(e) => {
          if (e.key === 'ArrowDown') setSelectedIndex(i => i + 1);
          if (e.key === 'ArrowUp') setSelectedIndex(i => i - 1);
          if (e.key === 'Enter') setIsOpen(!isOpen);
        }}
        aria-expanded={isOpen}
        aria-haspopup="listbox"
      >
        {items[selectedIndex]}
      </button>
      {isOpen && (
        <ul role="listbox">
          {items.map((item, i) => (
            <li
              key={i}
              onClick={() => {
                setSelectedIndex(i);
                setIsOpen(false);
              }}
              onKeyDown={(e) => {/* more handling */}}
              role="option"
              aria-selected={i === selectedIndex}
            >
              {item}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Senior Pattern: Props Getters

```javascript
function useDropdown({ items, defaultIndex = 0 }) {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState(defaultIndex);

  const getToggleButtonProps = ({ onClick, onKeyDown, ...props } = {}) => ({
    onClick: (e) => {
      onClick?.(e);
      setIsOpen(!isOpen);
    },
    onKeyDown: (e) => {
      onKeyDown?.(e);
      if (e.key === 'ArrowDown') {
        e.preventDefault();
        setIsOpen(true);
        setSelectedIndex(i => Math.min(i + 1, items.length - 1));
      }
      if (e.key === 'ArrowUp') {
        e.preventDefault();
        setIsOpen(true);
        setSelectedIndex(i => Math.max(i - 1, 0));
      }
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        setIsOpen(!isOpen);
      }
    },
    'aria-expanded': isOpen,
    'aria-haspopup': 'listbox',
    ...props
  });

  const getItemProps = ({ index, onClick, ...props } = {}) => ({
    onClick: (e) => {
      onClick?.(e);
      setSelectedIndex(index);
      setIsOpen(false);
    },
    role: 'option',
    'aria-selected': index === selectedIndex,
    ...props
  });

  return {
    isOpen,
    selectedIndex,
    getToggleButtonProps,
    getItemProps
  };
}

// Clean usage
function Dropdown({ items }) {
  const { isOpen, selectedIndex, getToggleButtonProps, getItemProps } =
    useDropdown({ items });

  return (
    <div>
      <button {...getToggleButtonProps()}>
        {items[selectedIndex]}
      </button>
      {isOpen && (
        <ul role="listbox">
          {items.map((item, index) => (
            <li key={index} {...getItemProps({ index })}>
              {item}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Benefits
- Encapsulates complex prop logic
- Allows prop composition
- User can override props
- Accessibility baked in

### Exercise 2.3: Building Props Getters

**Challenge:** Build a `useTooltip` hook that uses props getters to handle all the complex tooltip behavior.

**Requirements:**
1. `getTriggerProps` and `getTooltipProps` getter functions
2. Show/hide on hover with configurable delay
3. Keyboard accessibility (show on focus, hide on blur)
4. Positioning logic (top, bottom, left, right)
5. Props should be composable (user can add their own handlers)

**Think About:**
- What props does the trigger element need?
- What props does the tooltip element need?
- How do you merge user's onClick with your onClick?
- How do you calculate tooltip position?

<details>
<summary>💡 Hint: Structure your hook</summary>

**Return object:**
```javascript
{
  isOpen,
  getTriggerProps: ({ onClick, onMouseEnter, onFocus, ...props }) => ({...}),
  getTooltipProps: ({ style, ...props }) => ({...})
}
```

**Trigger needs:**
- Mouse enter/leave handlers
- Focus/blur handlers
- ARIA attributes

**Tooltip needs:**
- Role and ARIA attributes
- Positioning styles
- Conditional rendering based on `isOpen`

</details>

<details>
<summary>✅ Solution</summary>

```javascript
import { useState, useRef } from 'react';

function useTooltip({ delay = 200, position = 'top' } = {}) {
  const [isOpen, setIsOpen] = useState(false);
  const timeoutRef = useRef(null);
  const triggerRef = useRef(null);

  const show = () => {
    clearTimeout(timeoutRef.current);
    timeoutRef.current = setTimeout(() => setIsOpen(true), delay);
  };

  const hide = () => {
    clearTimeout(timeoutRef.current);
    setIsOpen(false);
  };

  const getTriggerProps = ({
    onMouseEnter,
    onMouseLeave,
    onFocus,
    onBlur,
    ...props
  } = {}) => ({
    ref: triggerRef,
    'aria-describedby': isOpen ? 'tooltip' : undefined,
    onMouseEnter: (e) => {
      onMouseEnter?.(e);
      show();
    },
    onMouseLeave: (e) => {
      onMouseLeave?.(e);
      hide();
    },
    onFocus: (e) => {
      onFocus?.(e);
      show();
    },
    onBlur: (e) => {
      onBlur?.(e);
      hide();
    },
    ...props
  });

  const getTooltipProps = ({ style, ...props } = {}) => {
    // Calculate position based on trigger element
    const triggerRect = triggerRef.current?.getBoundingClientRect();
    const positions = {
      top: {
        top: (triggerRect?.top || 0) - 8,
        left: (triggerRect?.left || 0) + (triggerRect?.width || 0) / 2,
        transform: 'translate(-50%, -100%)'
      },
      bottom: {
        top: (triggerRect?.bottom || 0) + 8,
        left: (triggerRect?.left || 0) + (triggerRect?.width || 0) / 2,
        transform: 'translate(-50%, 0)'
      },
      // Add left/right as needed
    };

    return {
      id: 'tooltip',
      role: 'tooltip',
      style: {
        position: 'fixed',
        ...positions[position],
        ...style
      },
      ...props
    };
  };

  return {
    isOpen,
    getTriggerProps,
    getTooltipProps
  };
}

// Usage
function TooltipExample() {
  const { isOpen, getTriggerProps, getTooltipProps } = useTooltip({
    delay: 300,
    position: 'top'
  });

  return (
    <>
      <button {...getTriggerProps()}>
        Hover me
      </button>
      {isOpen && (
        <div {...getTooltipProps({ style: { background: 'black', color: 'white', padding: 8 } })}>
          This is a tooltip!
        </div>
      )}
    </>
  );
}
```

**Key Observations:**
- Props getters accept user's props and merge them
- User's handlers called FIRST, then ours (composition!)
- ARIA attributes baked in for accessibility
- Position calculated dynamically based on trigger
- Clean API - user just spreads the props

**What You Learned:**
- Props getters encapsulate complex prop logic
- Always compose, never replace user's handlers
- Use `ref` to access DOM elements for positioning
- Accessibility can be built-in without user effort

</details>

---

## 2.4 Control Props Pattern

**🧠 Quick Recall (from 2.3):** Before learning about control props, test yourself: Why do props getters accept user props as arguments? What happens if you don't compose event handlers?

<details>
<summary>Check your answer</summary>

**Why accept user props:**
- So users can add their own onClick, onMouseEnter, etc.
- Enables composition, not replacement
- Makes the API flexible

**If you don't compose handlers:**
```javascript
// Bad - overwrites user's onClick
const getProps = () => ({ onClick: myHandler });

<button {...getProps()} onClick={userHandler}>
  // Only userHandler runs! (spread order matters)
```

**Fix:**
```javascript
const getProps = ({ onClick, ...props } = {}) => ({
  onClick: (e) => {
    onClick?.(e);  // User's first
    myHandler(e);  // Then ours
  },
  ...props
});
```

Good! Now let's learn how to make components work both controlled and uncontrolled.
</details>

---

### Controlled vs Uncontrolled Components (Advanced)

```javascript
// Junior: Either fully controlled or fully uncontrolled
function UncontrolledAccordion({ defaultOpenIndexes = [] }) {
  const [openIndexes, setOpenIndexes] = useState(defaultOpenIndexes);
  // Always uses internal state
}

function ControlledAccordion({ openIndexes, onToggle }) {
  // Must be controlled from outside
}

// Senior: Flexible - supports both!
function Accordion({
  openIndexes: controlledOpenIndexes,
  defaultOpenIndexes = [],
  onToggle
}) {
  const [uncontrolledOpenIndexes, setUncontrolledOpenIndexes] =
    useState(defaultOpenIndexes);

  // Use controlled value if provided, otherwise use internal
  const isControlled = controlledOpenIndexes !== undefined;
  const openIndexes = isControlled
    ? controlledOpenIndexes
    : uncontrolledOpenIndexes;

  const handleToggle = (index) => {
    const newIndexes = openIndexes.includes(index)
      ? openIndexes.filter(i => i !== index)
      : [...openIndexes, index];

    // Call external handler if provided
    onToggle?.(newIndexes);

    // Update internal state if uncontrolled
    if (!isControlled) {
      setUncontrolledOpenIndexes(newIndexes);
    }
  };

  return (/* accordion UI */);
}

// Usage: Uncontrolled
<Accordion defaultOpenIndexes={[0]} />

// Usage: Controlled
const [openIndexes, setOpenIndexes] = useState([0]);
<Accordion openIndexes={openIndexes} onToggle={setOpenIndexes} />
```

### Exercise 2.4: Controlled/Uncontrolled Component

**Challenge:** Build a flexible `Input` component that works in both controlled and uncontrolled modes, with proper warnings when used incorrectly.

**Requirements:**
1. Support both `value` + `onChange` (controlled) and `defaultValue` (uncontrolled)
2. Detect if component is controlled or uncontrolled
3. Warn in development if user:
   - Provides both `value` and `defaultValue`
   - Switches from controlled to uncontrolled (or vice versa)
4. Handle the state correctly in both modes

**Think About:**
- How do you detect if a component is controlled?
- What happens if `value` changes from `undefined` to a string?
- When should you use `useRef` vs `useState` for tracking?

<details>
<summary>💡 Hint: Track the control mode</summary>

**Key pattern:**
```javascript
const isControlled = value !== undefined;
const displayValue = isControlled ? value : internalValue;
```

**For warnings:**
- Use `useRef` to remember the initial control mode
- Use `useEffect` to detect changes
- Only warn in `process.env.NODE_ENV !== 'production'`

</details>

<details>
<summary>✅ Solution</summary>

```javascript
import { useState, useRef, useEffect } from 'react';

function Input({ value, defaultValue = '', onChange, ...props }) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = value !== undefined;
  const isControlledRef = useRef(isControlled);

  // Warn if both value and defaultValue provided
  useEffect(() => {
    if (process.env.NODE_ENV !== 'production') {
      if (value !== undefined && defaultValue !== undefined) {
        console.warn(
          'Input: You provided both `value` and `defaultValue`. ' +
          'Use `value` for controlled or `defaultValue` for uncontrolled, but not both.'
        );
      }
    }
  }, []); // Only check on mount

  // Warn if control mode changes
  useEffect(() => {
    if (process.env.NODE_ENV !== 'production') {
      const wasControlled = isControlledRef.current;
      const isNowControlled = value !== undefined;

      if (wasControlled !== isNowControlled) {
        console.error(
          `Input: A component is changing from ${wasControlled ? 'controlled' : 'uncontrolled'} ` +
          `to ${isNowControlled ? 'controlled' : 'uncontrolled'}. ` +
          `This is likely caused by the value changing from ${wasControlled ? 'defined to undefined' : 'undefined to defined'}, ` +
          `which should not happen. Decide between using a controlled or uncontrolled input for the lifetime of the component.`
        );
      }
    }
  }, [value]);

  // Determine which value to display
  const displayValue = isControlled ? value : internalValue;

  const handleChange = (e) => {
    const newValue = e.target.value;

    // Always call user's onChange if provided
    onChange?.(e);

    // Only update internal state if uncontrolled
    if (!isControlled) {
      setInternalValue(newValue);
    }
  };

  return (
    <input
      {...props}
      value={displayValue}
      onChange={handleChange}
    />
  );
}

export default Input;
```

**Usage Examples:**

```javascript
// Uncontrolled mode
<Input defaultValue="Hello" />

// Controlled mode
const [value, setValue] = useState('');
<Input value={value} onChange={(e) => setValue(e.target.value)} />

// ❌ Wrong - will warn
<Input value={value} defaultValue="hello" onChange={setValue} />

// ❌ Wrong - will error
const [value, setValue] = useState(undefined);
<Input value={value} onChange={(e) => setValue(e.target.value)} />
// If value changes to a string, component switches modes!
```

**Key Observations:**
- `useRef` remembers the initial control mode (doesn't cause re-renders)
- `isControlled` checked on every render
- Warnings only in development (`process.env.NODE_ENV`)
- Internal state only updated when uncontrolled
- User's `onChange` called in both modes

**What You Learned:**
- Controlled: Component state managed by parent
- Uncontrolled: Component manages its own state
- Control mode should never change during lifecycle
- Always warn users when they make mistakes
- `useRef` for values that don't trigger renders

</details>

<details>
<summary>📚 Deep Dive: When to use each mode?</summary>

**Use controlled when:**
- You need to validate on every keystroke
- Value affects other UI (character counter, live preview)
- Multiple inputs depend on each other
- You need to programmatically set the value
- Building forms with libraries (React Hook Form, Formik)

**Use uncontrolled when:**
- Simple forms where you only need the value on submit
- Performance is critical (no re-render on every keystroke)
- Integrating with non-React code (jQuery plugins, etc.)
- File inputs (always uncontrolled in React)

**The React team recommends:**
- Default to controlled for most use cases
- Use uncontrolled for simple forms or file inputs
- Never switch modes mid-lifecycle!

</details>

---

## ✅ Quick Knowledge Check: All Patterns

Test your understanding of all 4 patterns before moving on!

**Question 1:** You're building a `Select` component. It has a trigger button, a dropdown menu, and multiple options. Which pattern should you use?

<details>
<summary>Show answer</summary>

**Answer:** Compound Components

**Why:**
- Multiple related parts (Trigger, Menu, Option)
- Shared state needed (isOpen, selectedValue)
- Users need flexibility in rendering

```javascript
<Select value={value} onChange={setValue}>
  <Select.Trigger>{value || 'Choose...'}</Select.Trigger>
  <Select.Menu>
    <Select.Option value="1">Option 1</Select.Option>
    <Select.Option value="2">Option 2</Select.Option>
  </Select.Menu>
</Select>
```

</details>

---

**Question 2:** You need to share data fetching logic across 10 components. Should you use render props or a custom hook?

<details>
<summary>Show answer</summary>

**Answer:** Custom Hook

**Why:**
- Pure logic sharing (no UI structure)
- Modern React best practice
- Cleaner, more composable
- Easier to test

```javascript
function useData(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  // ... fetch logic
  return { data, loading };
}

// Each component uses it
const { data, loading } = useData('/api/users');
```

</details>

---

**Question 3:** Your `useDropdown` hook needs to provide onClick, onKeyDown, and aria-* props to a button. What pattern should you use?

<details>
<summary>Show answer</summary>

**Answer:** Props Getters

**Why:**
- Complex prop logic that needs to be reused
- Users might want to add their own handlers
- Encapsulates accessibility

```javascript
const { getToggleProps } = useDropdown();

<button {...getToggleProps({ onClick: myCustomHandler })}>
  // Both dropdown logic AND custom handler run!
</button>
```

</details>

---

## Real-World Scenario: Building a UI Library

### The Challenge
You're building a component library used by 50+ teams. Your components must:
- Work in different contexts
- Allow customization without breaking
- Be accessible by default
- Have intuitive APIs

### Your Task
Design and implement a `Select` component that:
1. Uses compound components for flexibility
2. Implements props getters for accessibility
3. Supports controlled/uncontrolled modes
4. Handles keyboard navigation
5. Works with form libraries
6. Is fully accessible

### Discussion Questions
- How do you balance flexibility and simplicity?
- When should you restrict usage?
- How do you handle breaking API changes?
- How do you document complex patterns?

## Chapter Exercise: Build a Form Builder

Create a flexible form system using patterns from this chapter:

**Requirements:**
1. Compound components: `Form`, `Field`, `ErrorMessage`, `Submit`
2. Props getters for common input props
3. Control props for form state
4. Support for validation
5. Works with any UI library

**Example Usage:**
```javascript
<Form onSubmit={handleSubmit}>
  <Field name="email" validate={emailValidator}>
    {({ getInputProps, error }) => (
      <>
        <input {...getInputProps({ type: 'email' })} />
        {error && <ErrorMessage>{error}</ErrorMessage>}
      </>
    )}
  </Field>
  <Submit>Register</Submit>
</Form>
```

## 🚫 Common Mistakes Gallery

**Real patterns from real code reviews.** Each mistake comes from production code.

### Mistake #1: Using Compound Components Without Context

```javascript
// ❌ BAD - No shared state, just prop drilling disguised
function Card({ children }) {
  return <div className="card">{children}</div>;
}

Card.Header = ({ children }) => <div className="header">{children}</div>;
Card.Body = ({ children }) => <div className="body">{children}</div>;
```

**Why it's wrong:**
- No shared state = no need for compound pattern
- Just added complexity with no benefit
- Simple props would work better

**✅ Fix:**
```javascript
// Just use a simple component with children
function Card({ header, children }) {
  return (
    <div className="card">
      <div className="header">{header}</div>
      <div className="body">{children}</div>
    </div>
  );
}
```

---

### Mistake #2: Forgetting the Context Boundary Check

```javascript
// ❌ BAD - No safety check
function Tab() {
  const { activeIndex } = useContext(TabsContext); // Crashes if used outside Tabs!
  // ...
}
```

**What happens:**
User tries `<Tab>` outside of `<Tabs>` → `useContext` returns `undefined` → App crashes

**✅ Fix:**
```javascript
function useTabs() {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('<Tab> must be used within <Tabs>');
  }
  return context;
}

function Tab() {
  const { activeIndex } = useTabs(); // Safe!
  // ...
}
```

---

### Mistake #3: Props Getters That Don't Compose

```javascript
// ❌ BAD - Overwrites user's onClick
function useDropdown() {
  const getButtonProps = () => ({
    onClick: () => setIsOpen(!isOpen), // User can't add their own onClick!
  });
  return { getButtonProps };
}

// User tries to add logging
<button {...getButtonProps()} onClick={() => console.log('clicked')}>
  // Only console.log runs, dropdown doesn't open!
</button>
```

**✅ Fix:**
```javascript
function useDropdown() {
  const getButtonProps = ({ onClick, ...props } = {}) => ({
    onClick: (e) => {
      onClick?.(e); // Call user's onClick first
      setIsOpen(!isOpen); // Then our logic
    },
    ...props // Spread remaining props
  });
  return { getButtonProps };
}
```

---

### Mistake #4: Controlled Component Without Warning

```javascript
// ❌ BAD - Silently fails if user mixes controlled/uncontrolled
function Input({ value, defaultValue, onChange }) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const displayValue = value ?? internalValue;
  // User changes from controlled to uncontrolled → broken!
}
```

**✅ Fix:**
```javascript
function Input({ value, defaultValue, onChange }) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = value !== undefined;

  // Warn in development
  useEffect(() => {
    if (process.env.NODE_ENV !== 'production') {
      if (isControlled && defaultValue !== undefined) {
        console.warn(
          'Input: Cannot use both `value` and `defaultValue`. Use one or the other.'
        );
      }
    }
  }, [isControlled, defaultValue]);

  const displayValue = isControlled ? value : internalValue;
  // ...
}
```

---

### Mistake #5: Render Props With Stale Closures

```javascript
// ❌ BAD - Captures old onClick value
function DataLoader({ url, render }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(res => res.json()).then(setData);
  }, [url]);

  return render(data); // If render function changes, we don't re-render!
}

// Usage
const handleClick = () => console.log(user.id);
<DataLoader
  url="/api/user"
  render={(user) => <button onClick={handleClick}>...</button>}
/>
// handleClick has old user.id!
```

**✅ Fix:**
```javascript
// Option 1: Use custom hook instead
function useDataLoader(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch(url).then(res => res.json()).then(setData);
  }, [url]);
  return data;
}

// Usage - handleClick always has latest user
const user = useDataLoader('/api/user');
const handleClick = () => console.log(user.id); // ✓ Latest value
<button onClick={handleClick}>...</button>

// Option 2: If you must use render props, pass callbacks in the render function
<DataLoader
  url="/api/user"
  render={(user) => (
    <button onClick={() => console.log(user.id)}>  {/* ✓ Inline, always fresh */}
      ...
    </button>
  )}
/>
```

---

### Mistake #6: Over-Engineering Simple Use Cases

```javascript
// ❌ BAD - This doesn't need ANY advanced pattern!
function Loading({ isLoading, children }) {
  if (isLoading) return <Spinner />;
  return children;
}

// Someone made this with render props for NO REASON
function Loading({ isLoading, renderLoading, renderChildren }) {
  if (isLoading) return renderLoading();
  return renderChildren();
}
```

**The pattern decision tree:**

```
Do I need shared state between parts?
  ├─ YES → Compound Components
  └─ NO
      └─ Do I need to reuse logic?
          ├─ YES → Custom Hook
          └─ NO → Just use props!
```

**✅ When in doubt:** Start with props. Refactor to patterns only when props become unwieldy.

---

## 🧠 Cumulative Review: Test Your Complete Understanding

**Research shows:** Testing yourself multiple times with spaced intervals dramatically improves retention. Let's review all 4 sections together.

**Instructions:** Try to answer from memory first. These questions mix concepts from the beginning (2.1) with recent sections (2.4).

### Question 1: Pattern Selection (from 2.1-2.4)

You need to build a `ColorPicker` component. Users should be able to:
- Click a button to open a color palette
- Select from preset colors or enter custom hex
- See a preview of the selected color

**Which pattern(s) would you use and why?**

<details>
<summary>✅ Answer</summary>

**Best approach:** Combination of compound components + props getters

```javascript
// Compound components for flexible composition
<ColorPicker value={color} onChange={setColor}>
  <ColorPicker.Trigger />      {/* Button to open */}
  <ColorPicker.Palette>         {/* The popup */}
    <ColorPicker.Presets />     {/* Preset colors */}
    <ColorPicker.CustomInput /> {/* Custom hex input */}
    <ColorPicker.Preview />     {/* Preview swatch */}
  </ColorPicker.Palette>
</ColorPicker>

// With props getters for accessibility
function useColorPicker({ value, onChange }) {
  const [isOpen, setIsOpen] = useState(false);

  const getTriggerProps = ({ onClick, ...props } = {}) => ({
    onClick: (e) => {
      onClick?.(e);
      setIsOpen(!isOpen);
    },
    'aria-expanded': isOpen,
    'aria-haspopup': 'dialog',
    ...props
  });

  return { isOpen, value, onChange, getTriggerProps };
}
```

**Why this combination:**
- Compound components: Multiple related parts (Trigger, Palette, Presets, etc.)
- Shared state needed: `isOpen`, `value`
- Props getters: Complex accessibility props (ARIA attributes, keyboard handling)
- Control props: Supports both controlled (`value` + `onChange`) and uncontrolled (`defaultValue`)

**Why NOT other patterns:**
- ❌ Just props: Too many props for all the customization options
- ❌ Render props alone: Awkward for this multi-part UI
- ❌ Just custom hook: Need visual structure, not just logic

</details>

---

### Question 2: Compound Components (from 2.1)

This code has a bug. What's wrong?

```javascript
const TabsContext = createContext();

function Tabs({ children }) {
  const [activeIndex, setActiveIndex] = useState(0);
  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext.Provider>
  );
}

function Tab({ index, children }) {
  const { setActiveIndex } = useContext(TabsContext);
  return <button onClick={() => setActiveIndex(index)}>{children}</button>;
}

// Usage
<Tabs>
  <Tab index={0}>Tab 1</Tab>
  <Tab>Tab 2</Tab>  {/* Bug here! */}
</Tabs>
```

<details>
<summary>✅ Answer</summary>

**Bug:** Second `<Tab>` is missing the `index` prop!

**What happens:**
- `index` is `undefined`
- Clicking Tab 2 calls `setActiveIndex(undefined)`
- Active state breaks

**Better design - auto-assign indices:**
```javascript
Tabs.TabList = function TabList({ children }) {
  return (
    <div role="tablist">
      {Children.map(children, (child, index) =>
        cloneElement(child, { index })  // Auto-inject index
      )}
    </div>
  );
};

// Usage - no manual indices!
<Tabs>
  <Tabs.TabList>
    <Tab>Tab 1</Tab>  {/* Gets index=0 */}
    <Tab>Tab 2</Tab>  {/* Gets index=1 */}
  </Tabs.TabList>
</Tabs>
```

**Lesson:** Design APIs that prevent user errors. Auto-indexing is better than manual.

</details>

---

### Question 3: Render Props vs Custom Hooks (from 2.2)

When should you use a render prop instead of a custom hook?

<details>
<summary>✅ Answer</summary>

**Use render props when:**
1. **Multiple render variations needed**
   ```javascript
   // Render props - different loading states
   <DataFetcher
     url="/api/user"
     renderLoading={() => <Skeleton />}
     renderError={(err) => <ErrorBanner error={err} />}
     render={(data) => <UserProfile data={data} />}
   />
   ```

2. **Component needs JSX injection points**
   ```javascript
   <InfiniteList
     items={items}
     renderItem={(item) => <CustomCard item={item} />}
     renderEmpty={() => <EmptyState />}
     renderFooter={() => <LoadMoreButton />}
   />
   ```

**Use custom hooks when:**
1. **Pure logic sharing** (no rendering decisions)
   ```javascript
   const { data, loading, error } = useFetch('/api/user');
   ```

2. **Modern codebase** (hooks are more idiomatic)
3. **Multiple hooks can be composed**

**The trend:** Custom hooks have largely replaced render props. Use render props only when you truly need JSX injection.

</details>

---

### Question 4: Props Getters (from 2.3)

Why does this props getter break?

```javascript
function useDropdown() {
  const [isOpen, setIsOpen] = useState(false);

  const getToggleProps = () => ({
    onClick: () => setIsOpen(!isOpen),
    onKeyDown: (e) => {
      if (e.key === 'Enter') setIsOpen(!isOpen);
    }
  });

  return { getToggleProps };
}

// User tries to add custom onClick
<button {...getToggleProps()} onClick={handleCustomClick}>
  // Custom click doesn't run!
</button>
```

<details>
<summary>✅ Answer</summary>

**Problem:** Props spread order! User's `onClick` comes AFTER `...getToggleProps()`, so it gets overwritten.

```javascript
<button {...getToggleProps()} onClick={handleCustomClick}>
//      ^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^
//      This runs              This overwrites it!
```

**Fix 1 - Compose callbacks:**
```javascript
const getToggleProps = ({ onClick, onKeyDown, ...props } = {}) => ({
  onClick: (e) => {
    onClick?.(e);  // Call user's onClick first
    setIsOpen(!isOpen);  // Then ours
  },
  onKeyDown: (e) => {
    onKeyDown?.(e);  // Call user's handler
    if (e.key === 'Enter') setIsOpen(!isOpen);
  },
  ...props
});
```

**Fix 2 - Accept overrides as argument:**
```javascript
<button {...getToggleProps({ onClick: handleCustomClick })}>
```

**Lesson:** Props getters must support composition, not replacement.

</details>

---

### Question 5: Control Props (from 2.4)

This controlled/uncontrolled component has a bug. What happens?

```javascript
function Accordion({ openIndexes, defaultOpenIndexes = [], onToggle }) {
  const [internal, setInternal] = useState(defaultOpenIndexes);
  const isControlled = openIndexes !== undefined;
  const current = isControlled ? openIndexes : internal;

  const toggle = (index) => {
    const newIndexes = current.includes(index)
      ? current.filter(i => i !== index)
      : [...current, index];

    onToggle?.(newIndexes);
    if (!isControlled) setInternal(newIndexes);
  };

  // ...
}

// Usage - user changes from uncontrolled to controlled
const [indexes, setIndexes] = useState(undefined);
<Accordion
  openIndexes={indexes}  // undefined at first
  onToggle={setIndexes}
/>
// User clicks a button that sets indexes to [0]
<button onClick={() => setIndexes([0])}>Open first</button>
```

<details>
<summary>✅ Answer</summary>

**Bug:** Component switches from uncontrolled to controlled at runtime!

**What happens:**
1. Initially: `openIndexes={undefined}` → uncontrolled mode, uses `internal` state
2. User opens a section → `internal = [2]`
3. Button click sets `indexes = [0]`
4. Component becomes controlled → switches to `openIndexes=[0]`
5. **Lost state!** The `internal=[2]` is abandoned

**Fix - Warn in development:**
```javascript
const isControlled = useRef(openIndexes !== undefined);

useEffect(() => {
  if (process.env.NODE_ENV !== 'production') {
    const nowControlled = openIndexes !== undefined;
    if (isControlled.current !== nowControlled) {
      console.error(
        `Accordion is changing from ${isControlled.current ? 'controlled' : 'uncontrolled'} ` +
        `to ${nowControlled ? 'controlled' : 'uncontrolled'}. ` +
        `Decide between using openIndexes or defaultOpenIndexes, but not both.`
      );
    }
  }
}, [openIndexes]);
```

**Better user code:**
```javascript
// Start controlled from the beginning
const [indexes, setIndexes] = useState([]);  // Not undefined!
<Accordion openIndexes={indexes} onToggle={setIndexes} />
```

**Lesson:** Components should stay controlled or uncontrolled for their entire lifecycle. Warn users when they violate this.

</details>

---

### Question 6: Integrating Everything

You're building a `DataTable` component for a component library. It needs:
- Sortable columns
- Selectable rows
- Custom cell rendering
- Pagination
- Works controlled or uncontrolled

**Design the API using patterns from this chapter. Which patterns would you combine?**

<details>
<summary>✅ Answer</summary>

**Recommended approach:** Combine ALL patterns!

```javascript
// 1. Compound Components for structure
<DataTable data={data} onSort={handleSort}>
  <DataTable.Header>
    <DataTable.Column sortKey="name">Name</DataTable.Column>
    <DataTable.Column sortKey="email">Email</DataTable.Column>
    <DataTable.Column>Actions</DataTable.Column>
  </DataTable.Header>

  <DataTable.Body>
    {({ row, index }) => (  {/* 2. Render prop for custom cells */}
      <DataTable.Row key={row.id} index={index}>
        <DataTable.Cell>{row.name}</DataTable.Cell>
        <DataTable.Cell>{row.email}</DataTable.Cell>
        <DataTable.Cell>
          <button>Edit</button>
        </DataTable.Cell>
      </DataTable.Row>
    )}
  </DataTable.Body>

  <DataTable.Pagination />
</DataTable>

// 3. Props getters for row selection
function useDataTable({ data, selectedRows, onSelect }) {
  const getRowProps = ({ rowId, ...props } = {}) => ({
    onClick: () => onSelect(rowId),
    'aria-selected': selectedRows.includes(rowId),
    role: 'row',
    ...props
  });

  return { getRowProps, /* ... */ };
}

// 4. Control props for selection & sorting
// Controlled
<DataTable
  selectedRows={selected}
  onSelect={setSelected}
  sortBy={sortConfig}
  onSort={setSortConfig}
/>

// Uncontrolled
<DataTable
  defaultSelectedRows={[]}
  defaultSortBy={{ key: 'name', direction: 'asc' }}
/>
```

**Why this combination works:**
- **Compound components:** Multiple related parts (Header, Body, Row, Cell, Pagination)
- **Render props:** Custom cell rendering without prop explosion
- **Props getters:** Accessibility & selection logic encapsulated
- **Control props:** Flexible controlled/uncontrolled usage

**Key concept:** Real-world components often need MULTIPLE patterns working together!

</details>

---

## Review Checklist

- [ ] Implement compound components with shared context
- [ ] Choose between render props and custom hooks
- [ ] Build prop getters for complex components
- [ ] Create flexible controlled/uncontrolled components
- [ ] Explain trade-offs of each pattern
- [ ] Know when to use which pattern
- [ ] Recognize when NOT to use advanced patterns

## Key Takeaways

1. **Composition over configuration** - Flexible APIs beat large prop lists
2. **Think about use cases** - Design for how it will be used
3. **Progressive disclosure** - Simple things simple, complex things possible
4. **Consistency** - Use established patterns in your codebase
5. **Documentation** - Complex patterns need clear examples

## Next Chapter

[Chapter 3: Mastering React Hooks](./03-mastering-hooks.md)
