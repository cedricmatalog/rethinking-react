# Chapter 15: Testing Complex Interactions

## Introduction

Junior developers struggle to test complex user interactions. Senior developers know how to test drag-and-drop, real-time updates, accessibility, animations, and other advanced scenarios that make applications feel polished and professional.

## Learning Objectives

- Test drag-and-drop interactions
- Handle real-time features (WebSockets, SSE)
- Test file uploads and downloads
- Verify accessibility in tests
- Test keyboard navigation
- Handle animations and transitions
- Test third-party integrations
- Mock browser APIs effectively

## 15.1 Testing Drag-and-Drop

### Basic Drag-and-Drop Testing

```typescript
// DragDropList.test.tsx
import { DndProvider } from 'react-dnd';
import { HTML5Backend } from 'react-dnd-html5-backend';
import { render, fireEvent } from '@testing-library/react';

function renderWithDnd(ui: React.ReactElement) {
  return render(
    <DndProvider backend={HTML5Backend}>
      {ui}
    </DndProvider>
  );
}

describe('Drag and Drop List', () => {
  it('reorders items on drag and drop', async () => {
    const onReorder = jest.fn();
    const { getByText } = renderWithDnd(
      <DraggableList
        items={['Item 1', 'Item 2', 'Item 3']}
        onReorder={onReorder}
      />
    );

    const item1 = getByText('Item 1');
    const item3 = getByText('Item 3');

    // Simulate drag and drop
    fireEvent.dragStart(item1);
    fireEvent.dragEnter(item3);
    fireEvent.dragOver(item3);
    fireEvent.drop(item3);
    fireEvent.dragEnd(item1);

    expect(onReorder).toHaveBeenCalledWith(['Item 2', 'Item 3', 'Item 1']);
  });

  it('shows drop indicator during drag', async () => {
    const { getByText, container } = renderWithDnd(
      <DraggableList items={['Item 1', 'Item 2']} />
    );

    const item1 = getByText('Item 1');
    const item2 = getByText('Item 2');

    fireEvent.dragStart(item1);
    fireEvent.dragEnter(item2);

    // Check for drop indicator
    expect(container.querySelector('.drop-indicator')).toBeInTheDocument();

    fireEvent.dragEnd(item1);

    // Drop indicator removed
    expect(container.querySelector('.drop-indicator')).not.toBeInTheDocument();
  });

  it('prevents dropping in invalid zones', async () => {
    const onReorder = jest.fn();
    const { getByText, getByTestId } = renderWithDnd(
      <DragDropZones onReorder={onReorder} />
    );

    const item = getByText('Draggable Item');
    const invalidZone = getByTestId('invalid-drop-zone');

    fireEvent.dragStart(item);
    fireEvent.dragOver(invalidZone);
    fireEvent.drop(invalidZone);

    expect(onReorder).not.toHaveBeenCalled();
  });
});
```

### Advanced Drag-and-Drop with User Events

```typescript
import { pointerDown, pointerMove, pointerUp } from '@testing-library/user-event/dist/pointer';

describe('Kanban Board Drag and Drop', () => {
  it('moves card between columns', async () => {
    const { getByText } = render(<KanbanBoard />);

    const card = getByText('Task 1');
    const todoColumn = getByText('To Do').parentElement!;
    const inProgressColumn = getByText('In Progress').parentElement!;

    // Get initial positions
    const cardRect = card.getBoundingClientRect();
    const targetRect = inProgressColumn.getBoundingClientRect();

    // Simulate pointer drag
    await pointerDown(card, {
      clientX: cardRect.left + cardRect.width / 2,
      clientY: cardRect.top + cardRect.height / 2
    });

    await pointerMove(card, {
      clientX: targetRect.left + targetRect.width / 2,
      clientY: targetRect.top + 50
    });

    await pointerUp(card);

    // Verify card moved
    await waitFor(() => {
      expect(inProgressColumn).toContainElement(card);
      expect(todoColumn).not.toContainElement(card);
    });
  });

  it('supports multi-select drag', async () => {
    const { getByText } = render(<KanbanBoard />);

    const card1 = getByText('Task 1');
    const card2 = getByText('Task 2');

    // Select multiple cards
    await userEvent.click(card1, { ctrlKey: true });
    await userEvent.click(card2, { ctrlKey: true });

    // Drag one of them
    const targetColumn = getByText('Done').parentElement!;
    await dragTo(card1, targetColumn);

    // Both should move
    await waitFor(() => {
      expect(targetColumn).toContainElement(card1);
      expect(targetColumn).toContainElement(card2);
    });
  });
});
```

## 15.2 Testing Real-Time Features

### WebSocket Testing

```typescript
// WebSocketProvider.test.tsx
import { WebSocket, Server } from 'mock-socket';

describe('WebSocket Integration', () => {
  let mockServer: Server;

  beforeEach(() => {
    mockServer = new Server('ws://localhost:8080');
  });

  afterEach(() => {
    mockServer.close();
  });

  it('connects to WebSocket and receives messages', async () => {
    const { getByText } = render(
      <WebSocketProvider url="ws://localhost:8080">
        <ChatRoom />
      </WebSocketProvider>
    );

    // Wait for connection
    await waitFor(() => {
      expect(getByText('Connected')).toBeInTheDocument();
    });

    // Server sends message
    mockServer.emit('message', JSON.stringify({
      type: 'chat',
      user: 'John',
      message: 'Hello!'
    }));

    // Message appears
    await waitFor(() => {
      expect(getByText('John: Hello!')).toBeInTheDocument();
    });
  });

  it('sends messages through WebSocket', async () => {
    const onMessage = jest.fn();
    mockServer.on('message', onMessage);

    const { getByRole, getByLabelText } = render(
      <WebSocketProvider url="ws://localhost:8080">
        <ChatRoom />
      </WebSocketProvider>
    );

    await waitFor(() => {
      expect(getByText('Connected')).toBeInTheDocument();
    });

    // Send message
    await userEvent.type(getByLabelText('Message'), 'Test message');
    await userEvent.click(getByRole('button', { name: /send/i }));

    // Verify sent to server
    expect(onMessage).toHaveBeenCalledWith(
      expect.stringContaining('Test message')
    );
  });

  it('handles reconnection on disconnect', async () => {
    const { getByText } = render(
      <WebSocketProvider url="ws://localhost:8080">
        <ChatRoom />
      </WebSocketProvider>
    );

    await waitFor(() => {
      expect(getByText('Connected')).toBeInTheDocument();
    });

    // Simulate disconnect
    mockServer.close();

    await waitFor(() => {
      expect(getByText('Disconnected')).toBeInTheDocument();
    });

    // Reconnect
    mockServer = new Server('ws://localhost:8080');

    await waitFor(() => {
      expect(getByText('Connected')).toBeInTheDocument();
    });
  });

  it('handles message queue during disconnect', async () => {
    const { getByLabelText, getByRole } = render(
      <WebSocketProvider url="ws://localhost:8080">
        <ChatRoom />
      </WebSocketProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('Connected')).toBeInTheDocument();
    });

    // Disconnect
    mockServer.close();

    // Try to send message while disconnected
    await userEvent.type(getByLabelText('Message'), 'Queued message');
    await userEvent.click(getByRole('button', { name: /send/i }));

    // Reconnect
    const onMessage = jest.fn();
    mockServer = new Server('ws://localhost:8080');
    mockServer.on('message', onMessage);

    // Queued message should be sent
    await waitFor(() => {
      expect(onMessage).toHaveBeenCalledWith(
        expect.stringContaining('Queued message')
      );
    });
  });
});
```

### Server-Sent Events (SSE) Testing

```typescript
describe('Server-Sent Events', () => {
  it('receives real-time notifications', async () => {
    // Mock EventSource
    const mockEventSource = {
      addEventListener: jest.fn(),
      removeEventListener: jest.fn(),
      close: jest.fn()
    };

    global.EventSource = jest.fn(() => mockEventSource) as any;

    const { getByText } = render(<NotificationProvider />);

    // Get the message handler
    const messageHandler = mockEventSource.addEventListener.mock.calls
      .find(call => call[0] === 'message')?.[1];

    // Simulate server event
    messageHandler?.({
      data: JSON.stringify({
        type: 'notification',
        message: 'New message from John'
      })
    });

    expect(await screen.findByText('New message from John'))
      .toBeInTheDocument();
  });

  it('handles connection errors', async () => {
    const mockEventSource = {
      addEventListener: jest.fn(),
      removeEventListener: jest.fn(),
      close: jest.fn()
    };

    global.EventSource = jest.fn(() => mockEventSource) as any;

    render(<NotificationProvider />);

    // Get error handler
    const errorHandler = mockEventSource.addEventListener.mock.calls
      .find(call => call[0] === 'error')?.[1];

    // Simulate error
    errorHandler?.(new Event('error'));

    expect(await screen.findByText(/connection error/i))
      .toBeInTheDocument();
  });
});
```

## 15.3 Testing File Operations

### File Upload Testing

```typescript
describe('File Upload', () => {
  it('uploads single file', async () => {
    const onUpload = jest.fn();
    const { getByLabelText } = render(
      <FileUpload onUpload={onUpload} />
    );

    const file = new File(['hello'], 'hello.txt', { type: 'text/plain' });
    const input = getByLabelText('Upload file') as HTMLInputElement;

    await userEvent.upload(input, file);

    expect(input.files).toHaveLength(1);
    expect(input.files?.[0]).toBe(file);

    await waitFor(() => {
      expect(onUpload).toHaveBeenCalledWith(
        expect.objectContaining({
          name: 'hello.txt',
          type: 'text/plain'
        })
      );
    });
  });

  it('uploads multiple files', async () => {
    const { getByLabelText } = render(<FileUpload multiple />);

    const files = [
      new File(['file1'], 'file1.txt', { type: 'text/plain' }),
      new File(['file2'], 'file2.txt', { type: 'text/plain' })
    ];

    const input = getByLabelText('Upload files') as HTMLInputElement;
    await userEvent.upload(input, files);

    expect(input.files).toHaveLength(2);
    expect(screen.getByText('file1.txt')).toBeInTheDocument();
    expect(screen.getByText('file2.txt')).toBeInTheDocument();
  });

  it('validates file type', async () => {
    const { getByLabelText } = render(
      <FileUpload accept="image/*" />
    );

    const file = new File(['content'], 'document.pdf', {
      type: 'application/pdf'
    });

    const input = getByLabelText('Upload file') as HTMLInputElement;
    await userEvent.upload(input, file);

    expect(await screen.findByText(/invalid file type/i))
      .toBeInTheDocument();
  });

  it('validates file size', async () => {
    const { getByLabelText } = render(
      <FileUpload maxSize={1024} /> // 1KB max
    );

    // Create file larger than 1KB
    const largeContent = 'x'.repeat(2000);
    const file = new File([largeContent], 'large.txt', {
      type: 'text/plain'
    });

    const input = getByLabelText('Upload file') as HTMLInputElement;
    await userEvent.upload(input, file);

    expect(await screen.findByText(/file too large/i))
      .toBeInTheDocument();
  });

  it('shows upload progress', async () => {
    const { getByLabelText } = render(<FileUpload />);

    const file = new File(['content'], 'file.txt', { type: 'text/plain' });
    const input = getByLabelText('Upload file') as HTMLInputElement;

    await userEvent.upload(input, file);

    // Progress bar appears
    const progressBar = await screen.findByRole('progressbar');
    expect(progressBar).toBeInTheDocument();

    // Eventually completes
    await waitFor(() => {
      expect(progressBar).toHaveAttribute('aria-valuenow', '100');
    });
  });
});
```

### Drag-and-Drop File Upload

```typescript
describe('Drag and Drop File Upload', () => {
  it('accepts files dropped on drop zone', async () => {
    const onUpload = jest.fn();
    const { getByTestId } = render(
      <DropZone onUpload={onUpload} />
    );

    const dropZone = getByTestId('drop-zone');
    const file = new File(['content'], 'dropped.txt', {
      type: 'text/plain'
    });

    // Create drag event with files
    const dataTransfer = new DataTransfer();
    dataTransfer.items.add(file);

    fireEvent.dragEnter(dropZone, { dataTransfer });
    fireEvent.dragOver(dropZone, { dataTransfer });
    fireEvent.drop(dropZone, { dataTransfer });

    await waitFor(() => {
      expect(onUpload).toHaveBeenCalledWith(
        expect.arrayContaining([
          expect.objectContaining({ name: 'dropped.txt' })
        ])
      );
    });
  });

  it('highlights drop zone on drag over', () => {
    const { getByTestId } = render(<DropZone />);
    const dropZone = getByTestId('drop-zone');

    fireEvent.dragEnter(dropZone);

    expect(dropZone).toHaveClass('drag-over');

    fireEvent.dragLeave(dropZone);

    expect(dropZone).not.toHaveClass('drag-over');
  });
});
```

### File Download Testing

```typescript
describe('File Download', () => {
  it('downloads file on button click', async () => {
    // Mock URL.createObjectURL
    const mockUrl = 'blob:mock-url';
    global.URL.createObjectURL = jest.fn(() => mockUrl);

    // Mock link click
    const mockLink = {
      click: jest.fn(),
      setAttribute: jest.fn()
    };
    jest.spyOn(document, 'createElement').mockReturnValue(mockLink as any);

    const { getByRole } = render(<ExportButton data={mockData} />);

    await userEvent.click(getByRole('button', { name: /download/i }));

    expect(mockLink.setAttribute).toHaveBeenCalledWith('href', mockUrl);
    expect(mockLink.setAttribute).toHaveBeenCalledWith('download', expect.any(String));
    expect(mockLink.click).toHaveBeenCalled();
  });

  it('generates CSV file correctly', async () => {
    const data = [
      { name: 'John', age: 30 },
      { name: 'Jane', age: 25 }
    ];

    const { getByRole } = render(<ExportButton data={data} format="csv" />);

    // Capture Blob content
    let blobContent = '';
    global.URL.createObjectURL = jest.fn((blob: Blob) => {
      const reader = new FileReader();
      reader.onload = () => {
        blobContent = reader.result as string;
      };
      reader.readAsText(blob);
      return 'blob:mock';
    });

    await userEvent.click(getByRole('button', { name: /download/i }));

    await waitFor(() => {
      expect(blobContent).toContain('name,age');
      expect(blobContent).toContain('John,30');
      expect(blobContent).toContain('Jane,25');
    });
  });
});
```

## 15.4 Accessibility Testing

### Screen Reader Testing

```typescript
describe('Accessibility', () => {
  it('has proper ARIA labels', () => {
    const { getByRole } = render(<NavigationMenu />);

    const nav = getByRole('navigation');
    expect(nav).toHaveAttribute('aria-label', 'Main navigation');

    const menuButton = getByRole('button', { name: /menu/i });
    expect(menuButton).toHaveAttribute('aria-expanded', 'false');
    expect(menuButton).toHaveAttribute('aria-controls', expect.any(String));
  });

  it('announces dynamic content changes', async () => {
    const { getByRole } = render(<NotificationSystem />);

    const liveRegion = getByRole('status');
    expect(liveRegion).toHaveAttribute('aria-live', 'polite');

    // Trigger notification
    fireEvent.click(getByRole('button', { name: /notify/i }));

    await waitFor(() => {
      expect(liveRegion).toHaveTextContent('New notification received');
    });
  });

  it('manages focus correctly in modals', async () => {
    const { getByRole } = render(<ModalDialog />);

    const openButton = getByRole('button', { name: /open dialog/i });
    await userEvent.click(openButton);

    const dialog = getByRole('dialog');
    expect(dialog).toBeInTheDocument();

    // Focus should be inside dialog
    expect(document.activeElement).toBeInstanceOf(HTMLElement);
    expect(dialog).toContainElement(document.activeElement);

    // Close dialog
    await userEvent.keyboard('{Escape}');

    // Focus returns to trigger
    expect(document.activeElement).toBe(openButton);
  });

  it('supports keyboard navigation in dropdown', async () => {
    const { getByRole } = render(<Dropdown options={['Option 1', 'Option 2']} />);

    const button = getByRole('button');

    // Open with keyboard
    await userEvent.keyboard('{Tab}');
    expect(button).toHaveFocus();

    await userEvent.keyboard('{Enter}');

    const listbox = getByRole('listbox');
    expect(listbox).toBeVisible();

    // Navigate with arrows
    await userEvent.keyboard('{ArrowDown}');
    expect(getByRole('option', { name: 'Option 1' })).toHaveClass('focused');

    await userEvent.keyboard('{ArrowDown}');
    expect(getByRole('option', { name: 'Option 2' })).toHaveClass('focused');

    // Select with Enter
    await userEvent.keyboard('{Enter}');
    expect(button).toHaveTextContent('Option 2');
  });
});
```

### Using jest-axe for Automated Accessibility Testing

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Accessibility Compliance', () => {
  it('has no accessibility violations', async () => {
    const { container } = render(<ComplexForm />);
    const results = await axe(container);

    expect(results).toHaveNoViolations();
  });

  it('form inputs have labels', async () => {
    const { container } = render(<LoginForm />);

    const results = await axe(container, {
      rules: {
        'label': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });

  it('color contrast meets WCAG AA', async () => {
    const { container } = render(<Button variant="primary">Click me</Button>);

    const results = await axe(container, {
      rules: {
        'color-contrast': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });

  it('images have alt text', async () => {
    const { container } = render(<ImageGallery />);

    const results = await axe(container, {
      rules: {
        'image-alt': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });
});
```

## 15.5 Testing Keyboard Navigation

### Complex Keyboard Interactions

```typescript
describe('Keyboard Navigation', () => {
  it('navigates through table with arrow keys', async () => {
    const { getByRole } = render(<DataTable data={mockData} />);

    const firstCell = getByRole('cell', { name: mockData[0].name });
    firstCell.focus();

    // Move right
    await userEvent.keyboard('{ArrowRight}');
    expect(document.activeElement).toHaveTextContent(mockData[0].age);

    // Move down
    await userEvent.keyboard('{ArrowDown}');
    expect(document.activeElement).toHaveTextContent(mockData[1].age);

    // Move left
    await userEvent.keyboard('{ArrowLeft}');
    expect(document.activeElement).toHaveTextContent(mockData[1].name);
  });

  it('supports vim-style navigation (hjkl)', async () => {
    const { getByRole } = render(<Editor />);

    const editor = getByRole('textbox');
    editor.focus();

    // Enter command mode
    await userEvent.keyboard('{Escape}');

    // Navigate with hjkl
    await userEvent.keyboard('j'); // down
    await userEvent.keyboard('j');
    await userEvent.keyboard('k'); // up
    await userEvent.keyboard('l'); // right
    await userEvent.keyboard('h'); // left

    // Verify cursor position
    const cursorPosition = (editor as any).selectionStart;
    expect(cursorPosition).toBe(expectedPosition);
  });

  it('handles keyboard shortcuts', async () => {
    const onSave = jest.fn();
    const { getByRole } = render(<TextEditor onSave={onSave} />);

    const editor = getByRole('textbox');
    await userEvent.type(editor, 'Some text');

    // Ctrl+S to save
    await userEvent.keyboard('{Control>}s{/Control}');

    expect(onSave).toHaveBeenCalledWith('Some text');
  });

  it('supports tab navigation with skip links', async () => {
    const { getByText } = render(<Layout />);

    // Tab to skip link
    await userEvent.keyboard('{Tab}');
    expect(document.activeElement).toHaveTextContent('Skip to main content');

    // Activate skip link
    await userEvent.keyboard('{Enter}');

    // Focus moved to main content
    const main = getByRole('main');
    expect(main).toContainElement(document.activeElement);
  });
});
```

## 15.6 Testing Animations and Transitions

### Testing CSS Animations

```typescript
describe('Animations', () => {
  it('applies animation class', () => {
    const { getByTestId } = render(<AnimatedBox />);

    const box = getByTestId('animated-box');

    expect(box).toHaveClass('fade-in');
  });

  it('removes animation class after completion', async () => {
    const { getByTestId } = render(<AnimatedBox duration={100} />);

    const box = getByTestId('animated-box');

    expect(box).toHaveClass('fade-in');

    // Wait for animation to complete
    await waitFor(() => {
      expect(box).not.toHaveClass('fade-in');
    }, { timeout: 200 });
  });

  it('calls callback after animation', async () => {
    const onComplete = jest.fn();
    const { getByTestId } = render(
      <AnimatedBox onComplete={onComplete} duration={50} />
    );

    expect(onComplete).not.toHaveBeenCalled();

    await waitFor(() => {
      expect(onComplete).toHaveBeenCalled();
    }, { timeout: 100 });
  });
});
```

### Testing React Spring Animations

```typescript
import { act } from '@testing-library/react';

describe('React Spring Animations', () => {
  it('animates values', async () => {
    const { getByTestId } = render(<SpringBox />);

    const box = getByTestId('spring-box');

    // Initial state
    expect(box).toHaveStyle({ opacity: '0' });

    // Fast-forward animation
    await act(async () => {
      jest.advanceTimersByTime(1000);
    });

    // Final state
    expect(box).toHaveStyle({ opacity: '1' });
  });

  it('respects prefers-reduced-motion', () => {
    // Mock matchMedia
    window.matchMedia = jest.fn().mockImplementation(query => ({
      matches: query === '(prefers-reduced-motion: reduce)',
      media: query,
      addEventListener: jest.fn(),
      removeEventListener: jest.fn()
    }));

    const { getByTestId } = render(<AnimatedComponent />);

    const element = getByTestId('animated-element');

    // Should skip animation
    expect(element).toHaveStyle({ opacity: '1' });
    expect(element).not.toHaveClass('animating');
  });
});
```

## 15.7 Testing Third-Party Integrations

### Mocking Payment Providers (Stripe)

```typescript
describe('Stripe Integration', () => {
  beforeEach(() => {
    // Mock Stripe
    (window as any).Stripe = jest.fn(() => ({
      elements: jest.fn(() => ({
        create: jest.fn(() => ({
          mount: jest.fn(),
          on: jest.fn(),
          destroy: jest.fn()
        }))
      })),
      confirmCardPayment: jest.fn()
    }));
  });

  it('initializes Stripe elements', () => {
    render(<CheckoutForm />);

    expect((window as any).Stripe).toHaveBeenCalledWith(
      expect.any(String) // API key
    );
  });

  it('handles successful payment', async () => {
    const mockConfirm = jest.fn().mockResolvedValue({
      paymentIntent: { status: 'succeeded' }
    });

    (window as any).Stripe = jest.fn(() => ({
      elements: jest.fn(() => ({
        create: jest.fn(() => ({
          mount: jest.fn(),
          on: jest.fn()
        }))
      })),
      confirmCardPayment: mockConfirm
    }));

    const { getByRole } = render(<CheckoutForm />);

    await userEvent.click(getByRole('button', { name: /pay/i }));

    await waitFor(() => {
      expect(screen.getByText(/payment successful/i)).toBeInTheDocument();
    });
  });

  it('handles payment errors', async () => {
    const mockConfirm = jest.fn().mockResolvedValue({
      error: { message: 'Card declined' }
    });

    (window as any).Stripe = jest.fn(() => ({
      elements: jest.fn(() => ({
        create: jest.fn(() => ({
          mount: jest.fn(),
          on: jest.fn()
        }))
      })),
      confirmCardPayment: mockConfirm
    }));

    const { getByRole } = render(<CheckoutForm />);

    await userEvent.click(getByRole('button', { name: /pay/i }));

    await waitFor(() => {
      expect(screen.getByText(/card declined/i)).toBeInTheDocument();
    });
  });
});
```

### Mocking Google Maps

```typescript
describe('Google Maps Integration', () => {
  beforeEach(() => {
    // Mock Google Maps API
    (window as any).google = {
      maps: {
        Map: jest.fn(),
        Marker: jest.fn(),
        LatLng: jest.fn((lat, lng) => ({ lat, lng })),
        event: {
          addListener: jest.fn()
        }
      }
    };
  });

  it('initializes map with correct center', () => {
    render(<MapComponent center={{ lat: 40.7128, lng: -74.0060 }} />);

    expect((window as any).google.maps.Map).toHaveBeenCalledWith(
      expect.any(HTMLElement),
      expect.objectContaining({
        center: { lat: 40.7128, lng: -74.0060 }
      })
    );
  });

  it('adds markers to map', () => {
    const markers = [
      { lat: 40.7128, lng: -74.0060, title: 'New York' },
      { lat: 34.0522, lng: -118.2437, title: 'Los Angeles' }
    ];

    render(<MapComponent markers={markers} />);

    expect((window as any).google.maps.Marker).toHaveBeenCalledTimes(2);
  });
});
```

## Real-World Scenario: Testing Rich Text Editor

### The Challenge

Test a complex rich text editor with:
- Keyboard shortcuts
- Drag-and-drop
- File uploads (images)
- Toolbar interactions
- Accessibility

### Senior Approach

```typescript
describe('Rich Text Editor', () => {
  describe('Text Formatting', () => {
    it('applies bold formatting with keyboard shortcut', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');
      await userEvent.type(editor, 'Hello World');

      // Select "World"
      editor.setSelectionRange(6, 11);

      // Ctrl+B for bold
      await userEvent.keyboard('{Control>}b{/Control}');

      const html = editor.innerHTML;
      expect(html).toContain('<strong>World</strong>');
    });

    it('applies formatting via toolbar', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');
      await userEvent.type(editor, 'Test');

      editor.setSelectionRange(0, 4);

      await userEvent.click(getByRole('button', { name: /italic/i }));

      expect(editor.innerHTML).toContain('<em>Test</em>');
    });
  });

  describe('Image Upload', () => {
    it('inserts image via file upload', async () => {
      const { getByLabelText, getByRole } = render(<RichTextEditor />);

      const file = new File(['image'], 'test.png', { type: 'image/png' });
      const input = getByLabelText('Upload image') as HTMLInputElement;

      await userEvent.upload(input, file);

      await waitFor(() => {
        const editor = getByRole('textbox');
        expect(editor.querySelector('img')).toBeInTheDocument();
      });
    });

    it('inserts image via drag and drop', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');
      const file = new File(['image'], 'dropped.png', { type: 'image/png' });

      const dataTransfer = new DataTransfer();
      dataTransfer.items.add(file);

      fireEvent.drop(editor, { dataTransfer });

      await waitFor(() => {
        expect(editor.querySelector('img')).toBeInTheDocument();
      });
    });
  });

  describe('Accessibility', () => {
    it('announces formatting changes to screen readers', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');
      const status = getByRole('status');

      await userEvent.type(editor, 'Text');
      editor.setSelectionRange(0, 4);

      await userEvent.keyboard('{Control>}b{/Control}');

      expect(status).toHaveTextContent('Bold applied');
    });

    it('supports keyboard navigation in toolbar', async () => {
      const { getAllByRole } = render(<RichTextEditor />);

      const toolbarButtons = getAllByRole('button', {
        name: /bold|italic|underline/i
      });

      // Tab to first button
      await userEvent.keyboard('{Tab}');
      expect(toolbarButtons[0]).toHaveFocus();

      // Arrow to next button
      await userEvent.keyboard('{ArrowRight}');
      expect(toolbarButtons[1]).toHaveFocus();
    });

    it('has no accessibility violations', async () => {
      const { container } = render(<RichTextEditor />);
      const results = await axe(container);

      expect(results).toHaveNoViolations();
    });
  });

  describe('Complex Interactions', () => {
    it('supports undo/redo', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');

      await userEvent.type(editor, 'First');
      await userEvent.type(editor, ' Second');

      // Undo
      await userEvent.keyboard('{Control>}z{/Control}');
      expect(editor).toHaveTextContent('First');

      // Redo
      await userEvent.keyboard('{Control>}{Shift>}z{/Shift}{/Control}');
      expect(editor).toHaveTextContent('First Second');
    });

    it('maintains formatting when pasting', async () => {
      const { getByRole } = render(<RichTextEditor />);

      const editor = getByRole('textbox');

      // Paste HTML
      const clipboardData = new DataTransfer();
      clipboardData.setData('text/html', '<strong>Bold text</strong>');

      fireEvent.paste(editor, { clipboardData });

      expect(editor.innerHTML).toContain('<strong>Bold text</strong>');
    });
  });
});
```

## Chapter Exercise: Build Complete Test Suite

Create comprehensive tests for a complex interactive feature:

**Requirements:**
1. Test all user interactions (keyboard, mouse, touch)
2. Accessibility testing with jest-axe
3. File upload/download testing
4. Real-time features (if applicable)
5. Animation testing
6. Third-party integration mocking
7. Edge cases and error scenarios

**Evaluation:**
- All interaction methods tested
- Accessibility compliance
- No flaky tests
- Fast execution
- Maintainable test code

## Review Checklist

- [ ] Drag-and-drop tested properly
- [ ] WebSocket/real-time features tested
- [ ] File operations tested (upload/download)
- [ ] Accessibility verified with jest-axe
- [ ] Keyboard navigation tested
- [ ] Animations handled correctly
- [ ] Third-party APIs mocked
- [ ] Complex interactions covered

## Key Takeaways

1. **Complex interactions need testing** - Don't skip hard-to-test features
2. **Accessibility is testable** - Use jest-axe and manual testing
3. **Mock browser APIs strategically** - WebSockets, file APIs, etc.
4. **Test keyboard navigation** - Not just mouse clicks
5. **Handle animations in tests** - Fast-forward or mock
6. **Third-party integrations** - Always mock external services
7. **Real-time features** - Use mock servers for WebSocket testing

## Further Reading

- Testing Library: Complex Interactions
- jest-axe documentation
- WebSocket testing strategies
- ARIA Authoring Practices Guide
- React DnD testing guide

## Next Chapter

[Chapter 16: Build Optimization](./16-build-optimization.md)
