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

### The Problem

```javascript
// Junior approach - inflexible, hard to customize
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

### Senior Pattern: Compound Components

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

### Hands-On Exercise 2.1

Build a compound component system for a `Tabs` component:

**Requirements:**
1. `Tabs`, `TabList`, `Tab`, `TabPanels`, `TabPanel` components
2. Shared state for active tab
3. Keyboard navigation (arrow keys)
4. Support for controlled and uncontrolled usage
5. TypeScript types

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

## 2.2 Render Props Pattern

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

### Hands-On Exercise 2.2

Build a `<List>` component using render props:

**Requirements:**
1. Accepts array of data
2. Handles loading/error/empty states
3. Supports virtualization for large lists
4. Render props for item, loading, empty, error
5. Compare with custom hook approach

## 2.3 Props Getters Pattern

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

### Hands-On Exercise 2.3

Build a `useTooltip` hook with props getters:

**Requirements:**
1. `getTriggerProps` and `getTooltipProps` getters
2. Positioning logic (top, bottom, left, right)
3. Show/hide on hover with delay
4. Keyboard accessibility
5. Portal support for rendering

## 2.4 Control Props Pattern

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

### Hands-On Exercise 2.4

Build a flexible `DatePicker` component:

**Requirements:**
1. Supports controlled and uncontrolled modes
2. Props: `value`, `defaultValue`, `onChange`
3. Month/year navigation
4. Date range selection
5. Proper warnings in development for incorrect usage

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

## Review Checklist

- [ ] Implement compound components with shared context
- [ ] Choose between render props and custom hooks
- [ ] Build prop getters for complex components
- [ ] Create flexible controlled/uncontrolled components
- [ ] Explain trade-offs of each pattern
- [ ] Know when to use which pattern

## Key Takeaways

1. **Composition over configuration** - Flexible APIs beat large prop lists
2. **Think about use cases** - Design for how it will be used
3. **Progressive disclosure** - Simple things simple, complex things possible
4. **Consistency** - Use established patterns in your codebase
5. **Documentation** - Complex patterns need clear examples

## Next Chapter

[Chapter 3: Mastering React Hooks](./03-mastering-hooks.md)
