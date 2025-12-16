# Chapter 10: Building Design Systems

## Introduction

Junior developers use component libraries. Senior developers build component systems that scale across teams and products.

This chapter teaches you to create design systems that developers love to use.

## Learning Objectives

- Understand design system fundamentals
- Build scalable component libraries
- Implement theming systems
- Create comprehensive documentation
- Version and publish design systems
- Establish contribution guidelines
- Measure adoption and success

## 10.1 What is a Design System?

### Beyond Component Libraries

```javascript
// Just a component library - NO
const Button = ({ children, onClick }) => (
  <button onClick={onClick}>{children}</button>
);

// Design system - YES
// Includes:
// - Design tokens (colors, spacing, typography)
// - Components with variants
// - Usage guidelines
// - Accessibility standards
// - Code examples
// - Design files (Figma/Sketch)
// - Version history
// - Contribution process
```

### Layers of a Design System

```
┌─────────────────────────────────────┐
│     Design Principles               │  Philosophy & values
├─────────────────────────────────────┤
│     Design Tokens                   │  Colors, spacing, typography
├─────────────────────────────────────┤
│     Base Components                 │  Button, Input, etc.
├─────────────────────────────────────┤
│     Composite Components            │  Forms, Cards, etc.
├─────────────────────────────────────┤
│     Patterns & Templates            │  Page layouts, workflows
├─────────────────────────────────────┤
│     Documentation                   │  How to use everything
└─────────────────────────────────────┘
```

## 10.2 Design Tokens

### The Foundation

```typescript
// tokens/colors.ts
export const colors = {
  // Primitives - raw values
  blue50: '#eff6ff',
  blue100: '#dbeafe',
  blue500: '#3b82f6',
  blue900: '#1e3a8a',

  red50: '#fef2f2',
  red500: '#ef4444',
  red900: '#7f1d1d',

  gray50: '#f9fafb',
  gray500: '#6b7280',
  gray900: '#111827',

  // Semantic - meaning-based
  primary: '#3b82f6', // blue500
  primaryHover: '#1e3a8a', // blue900
  danger: '#ef4444', // red500
  dangerHover: '#7f1d1d', // red900

  textPrimary: '#111827', // gray900
  textSecondary: '#6b7280', // gray500

  bgPrimary: '#ffffff',
  bgSecondary: '#f9fafb', // gray50
  border: '#e5e7eb'
} as const;

// tokens/spacing.ts
export const spacing = {
  0: '0',
  1: '0.25rem', // 4px
  2: '0.5rem',  // 8px
  3: '0.75rem', // 12px
  4: '1rem',    // 16px
  6: '1.5rem',  // 24px
  8: '2rem',    // 32px
  12: '3rem',   // 48px
  16: '4rem'    // 64px
} as const;

// tokens/typography.ts
export const typography = {
  fontFamily: {
    sans: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
    mono: '"SF Mono", Monaco, "Cascadia Code", monospace'
  },
  fontSize: {
    xs: '0.75rem',    // 12px
    sm: '0.875rem',   // 14px
    base: '1rem',     // 16px
    lg: '1.125rem',   // 18px
    xl: '1.25rem',    // 20px
    '2xl': '1.5rem',  // 24px
    '3xl': '1.875rem' // 30px
  },
  fontWeight: {
    normal: 400,
    medium: 500,
    semibold: 600,
    bold: 700
  },
  lineHeight: {
    tight: 1.25,
    normal: 1.5,
    relaxed: 1.75
  }
} as const;
```

### Using Tokens in Components

```typescript
// components/Button/Button.tsx
import { colors, spacing, typography } from '@/tokens';

const buttonStyles = {
  base: {
    fontFamily: typography.fontFamily.sans,
    fontSize: typography.fontSize.base,
    fontWeight: typography.fontWeight.medium,
    padding: `${spacing[2]} ${spacing[4]}`,
    borderRadius: spacing[2],
    border: 'none',
    cursor: 'pointer',
    transition: 'all 0.2s'
  },
  variants: {
    primary: {
      backgroundColor: colors.primary,
      color: colors.bgPrimary,
      ':hover': {
        backgroundColor: colors.primaryHover
      }
    },
    secondary: {
      backgroundColor: colors.bgSecondary,
      color: colors.textPrimary,
      border: `1px solid ${colors.border}`
    },
    danger: {
      backgroundColor: colors.danger,
      color: colors.bgPrimary,
      ':hover': {
        backgroundColor: colors.dangerHover
      }
    }
  }
};
```

## 10.3 Component API Design

### Composable vs Monolithic

```typescript
// BAD: Monolithic API - hard to extend
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  icon?: 'left' | 'right';
  iconName?: string;
  loading?: boolean;
  disabled?: boolean;
  fullWidth?: boolean;
  // ... 20 more props
}

// GOOD: Composable API
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
  children: React.ReactNode;
}

interface ButtonIconProps {
  name: string;
  position?: 'left' | 'right';
}

// Usage - flexible
<Button variant="primary">
  <Button.Icon name="save" position="left" />
  Save
</Button>

<Button variant="danger">
  Delete
  {isDeleting && <Spinner />}
</Button>
```

### Flexible Styling APIs

```typescript
// Multiple styling approaches

// 1. Variant-based
<Button variant="primary" size="large">Click</Button>

// 2. Compound props
<Button primary large>Click</Button>

// 3. CSS-in-JS props
<Button bg="primary" px={4} py={2}>Click</Button>

// 4. Style prop (escape hatch)
<Button style={{ backgroundColor: 'purple' }}>Click</Button>

// 5. ClassName prop (for customization)
<Button className="my-custom-button">Click</Button>

// Best: Support variants + className for extensibility
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  className?: string; // For customization
  style?: React.CSSProperties; // Escape hatch
}

function Button({ variant = 'primary', size = 'medium', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        styles.base,
        styles[variant],
        styles[size],
        className // User customization
      )}
      {...props}
    />
  );
}
```

## 10.4 Theming System

### CSS Variables Approach

```css
/* themes/light.css */
:root[data-theme="light"] {
  --color-primary: #3b82f6;
  --color-bg: #ffffff;
  --color-text: #111827;
  --spacing-unit: 4px;
}

/* themes/dark.css */
:root[data-theme="dark"] {
  --color-primary: #60a5fa;
  --color-bg: #111827;
  --color-text: #f9fafb;
  --spacing-unit: 4px;
}

/* Component uses CSS variables */
.button {
  background-color: var(--color-primary);
  color: var(--color-text);
  padding: calc(var(--spacing-unit) * 2);
}
```

### Theme Context Approach

```typescript
// Theme provider
interface Theme {
  colors: typeof colors;
  spacing: typeof spacing;
  typography: typeof typography;
}

const ThemeContext = createContext<Theme | null>(null);

export function ThemeProvider({ theme, children }: {
  theme: Theme;
  children: React.ReactNode
}) {
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const theme = useContext(ThemeContext);
  if (!theme) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return theme;
}

// Component uses theme
function Button({ children }: { children: React.ReactNode }) {
  const theme = useTheme();

  return (
    <button
      style={{
        backgroundColor: theme.colors.primary,
        color: theme.colors.bgPrimary,
        padding: `${theme.spacing[2]} ${theme.spacing[4]}`
      }}
    >
      {children}
    </button>
  );
}

// Usage with different themes
import { lightTheme, darkTheme } from './themes';

function App() {
  const [isDark, setIsDark] = useState(false);

  return (
    <ThemeProvider theme={isDark ? darkTheme : lightTheme}>
      <Button>Themed Button</Button>
    </ThemeProvider>
  );
}
```

### Styled Components Theming

```typescript
import styled, { ThemeProvider } from 'styled-components';

const theme = {
  colors: {
    primary: '#3b82f6',
    danger: '#ef4444'
  },
  spacing: [0, 4, 8, 16, 24, 32, 48]
};

const Button = styled.button<{ variant?: 'primary' | 'danger' }>`
  background-color: ${props =>
    props.variant === 'danger'
      ? props.theme.colors.danger
      : props.theme.colors.primary
  };
  padding: ${props => props.theme.spacing[2]}px ${props => props.theme.spacing[4]}px;
  border: none;
  border-radius: 4px;
  color: white;
  cursor: pointer;

  &:hover {
    opacity: 0.9;
  }
`;

function App() {
  return (
    <ThemeProvider theme={theme}>
      <Button>Primary</Button>
      <Button variant="danger">Danger</Button>
    </ThemeProvider>
  );
}
```

## 10.5 Documentation

### Component Documentation

```typescript
/**
 * Button component
 *
 * A flexible button component that supports multiple variants and sizes.
 *
 * @example
 * ```tsx
 * <Button variant="primary" size="large" onClick={handleClick}>
 *   Click me
 * </Button>
 * ```
 *
 * @example With icon
 * ```tsx
 * <Button variant="primary">
 *   <Icon name="save" />
 *   Save
 * </Button>
 * ```
 */
export interface ButtonProps {
  /**
   * Visual style variant
   * @default "primary"
   */
  variant?: 'primary' | 'secondary' | 'danger';

  /**
   * Size of the button
   * @default "medium"
   */
  size?: 'small' | 'medium' | 'large';

  /**
   * Disabled state
   * @default false
   */
  disabled?: boolean;

  /**
   * Click handler
   */
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;

  /**
   * Button content
   */
  children: React.ReactNode;
}

export function Button({
  variant = 'primary',
  size = 'medium',
  disabled = false,
  onClick,
  children
}: ButtonProps) {
  // Implementation
}
```

### Storybook Integration

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger'],
      description: 'Visual style of the button'
    },
    size: {
      control: 'select',
      options: ['small', 'medium', 'large']
    }
  }
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Primary Button'
  }
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Secondary Button'
  }
};

export const Danger: Story = {
  args: {
    variant: 'danger',
    children: 'Danger Button'
  }
};

export const AllSizes: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', alignItems: 'center' }}>
      <Button size="small">Small</Button>
      <Button size="medium">Medium</Button>
      <Button size="large">Large</Button>
    </div>
  )
};

export const WithIcon: Story = {
  render: () => (
    <Button variant="primary">
      <span style={{ marginRight: '0.5rem' }}>💾</span>
      Save
    </Button>
  )
};

export const Loading: Story = {
  render: () => (
    <Button variant="primary" disabled>
      <Spinner size="small" />
      Loading...
    </Button>
  )
};
```

## 10.6 Accessibility

### ARIA Attributes

```typescript
interface ButtonProps {
  'aria-label'?: string;
  'aria-describedby'?: string;
  'aria-pressed'?: boolean;
  'aria-expanded'?: boolean;
}

// Icon-only button needs label
<Button aria-label="Save document">
  <SaveIcon />
</Button>

// Toggle button needs pressed state
<Button
  aria-pressed={isActive}
  onClick={() => setIsActive(!isActive)}
>
  {isActive ? 'Active' : 'Inactive'}
</Button>

// Dropdown button needs expanded state
<Button
  aria-expanded={isOpen}
  aria-haspopup="menu"
  onClick={() => setIsOpen(!isOpen)}
>
  Menu
</Button>
```

### Keyboard Navigation

```typescript
function Button({ onClick, ...props }: ButtonProps) {
  const handleKeyDown = (e: React.KeyboardEvent) => {
    // Space and Enter should trigger button
    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault();
      onClick?.(e as any);
    }
  };

  return (
    <button
      onClick={onClick}
      onKeyDown={handleKeyDown}
      {...props}
    />
  );
}

// Custom interactive element needs keyboard support
function CustomButton({ onClick }: { onClick: () => void }) {
  return (
    <div
      role="button"
      tabIndex={0}
      onClick={onClick}
      onKeyDown={(e) => {
        if (e.key === ' ' || e.key === 'Enter') {
          e.preventDefault();
          onClick();
        }
      }}
    >
      Click me
    </div>
  );
}
```

### Focus Management

```typescript
function Modal({ isOpen, onClose }: ModalProps) {
  const previousFocus = useRef<HTMLElement>();
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      // Save previous focus
      previousFocus.current = document.activeElement as HTMLElement;

      // Focus first element in modal
      const firstFocusable = modalRef.current?.querySelector(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      ) as HTMLElement;
      firstFocusable?.focus();
    } else {
      // Restore focus
      previousFocus.current?.focus();
    }
  }, [isOpen]);

  // Trap focus inside modal
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Escape') {
      onClose();
    }

    if (e.key === 'Tab') {
      const focusableElements = modalRef.current?.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );

      if (!focusableElements) return;

      const firstElement = focusableElements[0] as HTMLElement;
      const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
  };

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      onKeyDown={handleKeyDown}
    >
      {/* Modal content */}
    </div>
  );
}
```

## 10.7 Versioning and Publishing

### Semantic Versioning

```json
{
  "name": "@company/design-system",
  "version": "2.3.1",
  "description": "Company design system",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc && vite build",
    "test": "vitest",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

### Changelog

```markdown
# Changelog

## [2.3.1] - 2024-01-15

### Fixed
- Button hover state in dark mode
- Modal focus trap on Safari
- TypeScript types for Select component

## [2.3.0] - 2024-01-10

### Added
- New `DatePicker` component
- Dark mode support for all components
- `useMediaQuery` hook

### Changed
- Button API now supports `as` prop for polymorphism
- Improved Input accessibility with better labels

### Deprecated
- `LegacyModal` - Use `Modal` instead

## [2.2.0] - 2024-01-01

### Breaking Changes
- Removed `Button.Group` - use flex layout instead
- Changed `spacing` token values (4px → 8px base)

### Migration Guide
```tsx
// Before
<Button.Group>
  <Button>One</Button>
  <Button>Two</Button>
</Button.Group>

// After
<div style={{ display: 'flex', gap: '8px' }}>
  <Button>One</Button>
  <Button>Two</Button>
</div>
```
```

### Publishing Process

```bash
# 1. Update version
npm version patch # 2.3.0 → 2.3.1
npm version minor # 2.3.1 → 2.4.0
npm version major # 2.4.0 → 3.0.0

# 2. Update CHANGELOG.md

# 3. Build
npm run build
npm run test
npm run build-storybook

# 4. Publish
npm publish --access public

# 5. Tag release
git tag v2.3.1
git push --tags

# 6. Create GitHub release
gh release create v2.3.1 --notes "See CHANGELOG.md"
```

## Real-World Scenario: Scaling a Design System

### The Challenge

Your design system is used by 10 teams:
- Components are inconsistent
- Teams create custom versions
- No clear contribution process
- Documentation is outdated
- Breaking changes cause issues

### Your Plan

1. **Audit current usage**
2. **Establish governance**
3. **Create contribution guidelines**
4. **Improve documentation**
5. **Set up automated testing**
6. **Version carefully**
7. **Communicate changes**

## Chapter Exercise: Build a Component Library

Create a mini design system with:

**Requirements:**
1. Design tokens (colors, spacing, typography)
2. 5 base components (Button, Input, Select, Modal, Card)
3. Theming system (light/dark)
4. Storybook documentation
5. TypeScript types
6. Accessibility compliance
7. Version 1.0.0 published to npm

**Evaluation:**
- Component API quality
- Documentation completeness
- Accessibility score
- Type safety
- Bundle size

## Review Checklist

- [ ] Understand design system layers
- [ ] Create design tokens
- [ ] Design composable component APIs
- [ ] Implement theming system
- [ ] Write comprehensive documentation
- [ ] Ensure accessibility
- [ ] Version and publish correctly
- [ ] Establish contribution process

## Key Takeaways

1. **Tokens first** - Foundation for consistency
2. **API design matters** - Make it easy to use correctly
3. **Document everything** - Examples, props, guidelines
4. **Accessibility is required** - Not optional
5. **Version carefully** - Breaking changes hurt teams
6. **Composability > Configuration** - Flexible APIs
7. **Governance is key** - At scale, you need process

## Further Reading

- Design Systems by Alla Kholmatova
- Storybook documentation
- WAI-ARIA Authoring Practices
- Atomic Design by Brad Frost
- Design Tokens W3C specification

## Next Chapter

[Chapter 11: Advanced TypeScript for React](./11-advanced-typescript.md)
