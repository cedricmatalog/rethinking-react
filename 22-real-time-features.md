# Chapter 22: Real-Time Features

## Introduction

Junior developers poll servers every few seconds for updates, wasting bandwidth and creating lag. Senior developers implement true real-time features with WebSockets or Server-Sent Events, building responsive applications with live updates, presence tracking, and collaborative features.

## Learning Objectives

- Implement WebSocket connections
- Use Server-Sent Events (SSE) appropriately
- Build real-time notifications
- Create collaborative features
- Implement presence/online status
- Handle connection reliability
- Optimize real-time performance
- Build real-time chat applications

## 22.1 WebSocket Fundamentals

### Basic WebSocket Implementation

```typescript
// services/websocket.ts
export class WebSocketService {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;
  private listeners: Map<string, Set<(data: any) => void>> = new Map();

  connect(url: string, token?: string) {
    const wsUrl = token ? `${url}?token=${token}` : url;

    this.ws = new WebSocket(wsUrl);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
      this.emit('connected', null);
    };

    this.ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);
        this.emit(message.type, message.payload);
      } catch (error) {
        console.error('Failed to parse WebSocket message:', error);
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.emit('error', error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket disconnected');
      this.emit('disconnected', null);
      this.attemptReconnect(url, token);
    };
  }

  private attemptReconnect(url: string, token?: string) {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);

    setTimeout(() => {
      console.log(`Reconnecting... (Attempt ${this.reconnectAttempts})`);
      this.connect(url, token);
    }, delay);
  }

  send(type: string, payload: any) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({ type, payload }));
    } else {
      console.warn('WebSocket is not connected');
    }
  }

  on(event: string, callback: (data: any) => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);
  }

  off(event: string, callback: (data: any) => void) {
    this.listeners.get(event)?.delete(callback);
  }

  private emit(event: string, data: any) {
    this.listeners.get(event)?.forEach((callback) => callback(data));
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }

  isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }
}

export const wsService = new WebSocketService();
```

### React Hook for WebSocket

```typescript
// hooks/useWebSocket.ts
import { useEffect, useCallback, useState } from 'react';
import { wsService } from '@/services/websocket';

export function useWebSocket<T = any>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const handleConnected = () => setIsConnected(true);
    const handleDisconnected = () => setIsConnected(false);
    const handleError = (err: Error) => setError(err);

    wsService.on('connected', handleConnected);
    wsService.on('disconnected', handleDisconnected);
    wsService.on('error', handleError);

    if (!wsService.isConnected()) {
      wsService.connect(url);
    }

    return () => {
      wsService.off('connected', handleConnected);
      wsService.off('disconnected', handleDisconnected);
      wsService.off('error', handleError);
    };
  }, [url]);

  const subscribe = useCallback((event: string, callback: (data: T) => void) => {
    wsService.on(event, callback);
    return () => wsService.off(event, callback);
  }, []);

  const send = useCallback((type: string, payload: any) => {
    wsService.send(type, payload);
  }, []);

  return {
    data,
    isConnected,
    error,
    subscribe,
    send,
  };
}
```

## 22.2 Real-Time Chat Application

### Chat Context and State

```typescript
// contexts/ChatContext.tsx
interface Message {
  id: string;
  userId: string;
  username: string;
  text: string;
  timestamp: number;
}

interface ChatContextType {
  messages: Message[];
  sendMessage: (text: string) => void;
  isConnected: boolean;
  typingUsers: Set<string>;
}

const ChatContext = createContext<ChatContextType | undefined>(undefined);

export function ChatProvider({ children, roomId }: { children: React.ReactNode; roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [typingUsers, setTypingUsers] = useState<Set<string>>(new Set());
  const { isConnected, subscribe, send } = useWebSocket(
    `${import.meta.env.VITE_WS_URL}/chat/${roomId}`
  );

  useEffect(() => {
    const unsubscribeMessage = subscribe('message', (message: Message) => {
      setMessages((prev) => [...prev, message]);
    });

    const unsubscribeTyping = subscribe('user_typing', ({ userId, username }: any) => {
      setTypingUsers((prev) => new Set(prev).add(username));

      // Remove typing indicator after 3 seconds
      setTimeout(() => {
        setTypingUsers((prev) => {
          const newSet = new Set(prev);
          newSet.delete(username);
          return newSet;
        });
      }, 3000);
    });

    const unsubscribeHistory = subscribe('message_history', (history: Message[]) => {
      setMessages(history);
    });

    // Request message history
    send('get_history', { roomId });

    return () => {
      unsubscribeMessage();
      unsubscribeTyping();
      unsubscribeHistory();
    };
  }, [roomId, subscribe, send]);

  const sendMessage = useCallback(
    (text: string) => {
      send('send_message', {
        roomId,
        text,
        timestamp: Date.now(),
      });
    },
    [roomId, send]
  );

  const notifyTyping = useCallback(() => {
    send('typing', { roomId });
  }, [roomId, send]);

  return (
    <ChatContext.Provider
      value={{
        messages,
        sendMessage,
        isConnected,
        typingUsers,
      }}
    >
      {children}
    </ChatContext.Provider>
  );
}

export function useChat() {
  const context = useContext(ChatContext);
  if (!context) {
    throw new Error('useChat must be used within ChatProvider');
  }
  return context;
}
```

### Chat UI Components

```typescript
// components/ChatRoom.tsx
function ChatRoom() {
  const { messages, sendMessage, isConnected, typingUsers } = useChat();
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  return (
    <div className="chat-room">
      {!isConnected && (
        <div className="connection-status">
          Reconnecting...
        </div>
      )}

      <MessageList messages={messages} />

      {typingUsers.size > 0 && (
        <TypingIndicator users={Array.from(typingUsers)} />
      )}

      <div ref={messagesEndRef} />

      <MessageInput onSend={sendMessage} />
    </div>
  );
}

// components/MessageInput.tsx
function MessageInput({ onSend }: { onSend: (text: string) => void }) {
  const [text, setText] = useState('');
  const { send } = useWebSocket();
  const typingTimeoutRef = useRef<NodeJS.Timeout>();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setText(e.target.value);

    // Notify typing
    send('typing', {});

    // Clear previous timeout
    if (typingTimeoutRef.current) {
      clearTimeout(typingTimeoutRef.current);
    }

    // Stop typing indicator after 2 seconds of inactivity
    typingTimeoutRef.current = setTimeout(() => {
      send('stop_typing', {});
    }, 2000);
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (text.trim()) {
      onSend(text);
      setText('');
      send('stop_typing', {});
    }
  };

  return (
    <form onSubmit={handleSubmit} className="message-input">
      <input
        type="text"
        value={text}
        onChange={handleChange}
        placeholder="Type a message..."
      />
      <button type="submit">Send</button>
    </form>
  );
}

// components/TypingIndicator.tsx
function TypingIndicator({ users }: { users: string[] }) {
  if (users.length === 0) return null;

  const text =
    users.length === 1
      ? `${users[0]} is typing...`
      : users.length === 2
      ? `${users[0]} and ${users[1]} are typing...`
      : `${users[0]} and ${users.length - 1} others are typing...`;

  return (
    <div className="typing-indicator">
      <span>{text}</span>
      <span className="dots">
        <span>.</span>
        <span>.</span>
        <span>.</span>
      </span>
    </div>
  );
}
```

## 22.3 Server-Sent Events (SSE)

### SSE Implementation

```typescript
// hooks/useServerSentEvents.ts
export function useServerSentEvents<T = any>(url: string, options?: {
  withCredentials?: boolean;
}) {
  const [data, setData] = useState<T | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const eventSourceRef = useRef<EventSource | null>(null);

  useEffect(() => {
    const eventSource = new EventSource(url, {
      withCredentials: options?.withCredentials,
    });

    eventSourceRef.current = eventSource;

    eventSource.onopen = () => {
      setIsConnected(true);
      setError(null);
    };

    eventSource.onmessage = (event) => {
      try {
        const parsedData = JSON.parse(event.data);
        setData(parsedData);
      } catch (err) {
        console.error('Failed to parse SSE data:', err);
      }
    };

    eventSource.onerror = (err) => {
      setIsConnected(false);
      setError(err as Error);
    };

    return () => {
      eventSource.close();
    };
  }, [url, options?.withCredentials]);

  const subscribe = useCallback(
    (eventType: string, callback: (data: T) => void) => {
      const eventSource = eventSourceRef.current;
      if (!eventSource) return;

      const handler = (event: MessageEvent) => {
        try {
          const parsedData = JSON.parse(event.data);
          callback(parsedData);
        } catch (err) {
          console.error('Failed to parse event data:', err);
        }
      };

      eventSource.addEventListener(eventType, handler);

      return () => {
        eventSource.removeEventListener(eventType, handler);
      };
    },
    []
  );

  return {
    data,
    isConnected,
    error,
    subscribe,
  };
}
```

### Real-Time Notifications with SSE

```typescript
// components/NotificationCenter.tsx
interface Notification {
  id: string;
  type: 'info' | 'warning' | 'success' | 'error';
  title: string;
  message: string;
  timestamp: number;
}

function NotificationCenter() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const { subscribe, isConnected } = useServerSentEvents('/api/notifications/stream');

  useEffect(() => {
    const unsubscribe = subscribe('notification', (notification: Notification) => {
      setNotifications((prev) => [notification, ...prev]);

      // Show toast
      toast[notification.type](notification.message);

      // Auto-remove after 5 seconds
      setTimeout(() => {
        setNotifications((prev) =>
          prev.filter((n) => n.id !== notification.id)
        );
      }, 5000);
    });

    return unsubscribe;
  }, [subscribe]);

  return (
    <div className="notification-center">
      <div className="notification-header">
        <h3>Notifications</h3>
        {!isConnected && <span className="disconnected">Disconnected</span>}
      </div>

      <div className="notification-list">
        {notifications.map((notification) => (
          <NotificationCard
            key={notification.id}
            notification={notification}
            onDismiss={() =>
              setNotifications((prev) =>
                prev.filter((n) => n.id !== notification.id)
              )
            }
          />
        ))}

        {notifications.length === 0 && (
          <div className="empty">No new notifications</div>
        )}
      </div>
    </div>
  );
}
```

## 22.4 Presence and Online Status

### Presence Tracking

```typescript
// hooks/usePresence.ts
interface UserPresence {
  userId: string;
  username: string;
  status: 'online' | 'away' | 'offline';
  lastSeen: number;
}

export function usePresence(roomId: string) {
  const [presence, setPresence] = useState<Map<string, UserPresence>>(new Map());
  const { subscribe, send, isConnected } = useWebSocket(
    `${import.meta.env.VITE_WS_URL}/presence`
  );

  useEffect(() => {
    if (!isConnected) return;

    // Join room
    send('join_room', { roomId });

    const unsubscribePresence = subscribe('presence_update', ({
      userId,
      username,
      status,
      lastSeen,
    }: UserPresence) => {
      setPresence((prev) => {
        const newPresence = new Map(prev);
        if (status === 'offline') {
          newPresence.delete(userId);
        } else {
          newPresence.set(userId, { userId, username, status, lastSeen });
        }
        return newPresence;
      });
    });

    const unsubscribeInitial = subscribe('initial_presence', (users: UserPresence[]) => {
      setPresence(new Map(users.map((user) => [user.userId, user])));
    });

    // Update presence on activity
    const updatePresence = () => {
      send('update_presence', { status: 'online' });
    };

    const setAway = () => {
      send('update_presence', { status: 'away' });
    };

    // Update on activity
    window.addEventListener('mousemove', updatePresence);
    window.addEventListener('keypress', updatePresence);

    // Set away after 5 minutes of inactivity
    const awayTimer = setTimeout(setAway, 5 * 60 * 1000);

    return () => {
      send('leave_room', { roomId });
      unsubscribePresence();
      unsubscribeInitial();
      window.removeEventListener('mousemove', updatePresence);
      window.removeEventListener('keypress', updatePresence);
      clearTimeout(awayTimer);
    };
  }, [roomId, isConnected, subscribe, send]);

  const onlineUsers = Array.from(presence.values()).filter(
    (user) => user.status === 'online'
  );

  const awayUsers = Array.from(presence.values()).filter(
    (user) => user.status === 'away'
  );

  return {
    presence,
    onlineUsers,
    awayUsers,
    totalUsers: presence.size,
  };
}

// components/OnlineIndicator.tsx
function OnlineIndicator({ userId }: { userId: string }) {
  const { presence } = usePresence('global');
  const user = presence.get(userId);

  if (!user || user.status === 'offline') {
    return <span className="status offline">Offline</span>;
  }

  return (
    <span className={`status ${user.status}`}>
      <span className="dot" />
      {user.status}
    </span>
  );
}
```

## 22.5 Collaborative Features

### Collaborative Cursor Tracking

```typescript
// hooks/useCollaborativeCursors.ts
interface Cursor {
  userId: string;
  username: string;
  x: number;
  y: number;
  color: string;
}

export function useCollaborativeCursors(documentId: string) {
  const [cursors, setCursors] = useState<Map<string, Cursor>>(new Map());
  const { subscribe, send, isConnected } = useWebSocket(
    `${import.meta.env.VITE_WS_URL}/collab/${documentId}`
  );
  const throttledSendRef = useRef<ReturnType<typeof throttle>>();

  useEffect(() => {
    throttledSendRef.current = throttle((x: number, y: number) => {
      send('cursor_move', { x, y });
    }, 50);
  }, [send]);

  useEffect(() => {
    if (!isConnected) return;

    const unsubscribeCursor = subscribe('cursor_update', ({
      userId,
      username,
      x,
      y,
      color,
    }: Cursor) => {
      setCursors((prev) => {
        const newCursors = new Map(prev);
        newCursors.set(userId, { userId, username, x, y, color });
        return newCursors;
      });
    });

    const unsubscribeLeft = subscribe('user_left', ({ userId }: { userId: string }) => {
      setCursors((prev) => {
        const newCursors = new Map(prev);
        newCursors.delete(userId);
        return newCursors;
      });
    });

    const handleMouseMove = (e: MouseEvent) => {
      throttledSendRef.current?.(e.clientX, e.clientY);
    };

    document.addEventListener('mousemove', handleMouseMove);

    return () => {
      document.removeEventListener('mousemove', handleMouseMove);
      unsubscribeCursor();
      unsubscribeLeft();
    };
  }, [isConnected, subscribe]);

  return Array.from(cursors.values());
}

// components/CollaborativeCursors.tsx
function CollaborativeCursors() {
  const cursors = useCollaborativeCursors('doc-123');

  return (
    <>
      {cursors.map((cursor) => (
        <div
          key={cursor.userId}
          className="cursor"
          style={{
            left: cursor.x,
            top: cursor.y,
            borderColor: cursor.color,
          }}
        >
          <svg
            width="20"
            height="20"
            viewBox="0 0 20 20"
            fill={cursor.color}
          >
            <path d="M0 0 L0 16 L5 11 L8 16 L10 15 L7 10 L12 10 Z" />
          </svg>
          <span className="cursor-label">{cursor.username}</span>
        </div>
      ))}
    </>
  );
}
```

### Operational Transformation for Text Collaboration

```typescript
// lib/ot.ts - Simplified Operational Transformation
export interface Operation {
  type: 'insert' | 'delete' | 'retain';
  count?: number;
  text?: string;
}

export class OTClient {
  private localVersion = 0;
  private serverVersion = 0;
  private pendingOps: Operation[] = [];

  applyOperation(text: string, op: Operation): string {
    if (op.type === 'insert') {
      return text + op.text;
    } else if (op.type === 'delete') {
      return text.slice(0, -op.count!);
    }
    return text;
  }

  sendOperation(op: Operation) {
    this.pendingOps.push(op);
    this.localVersion++;

    // Send to server
    wsService.send('operation', {
      op,
      version: this.serverVersion,
    });
  }

  receiveOperation(op: Operation, version: number) {
    if (version !== this.serverVersion + 1) {
      console.error('Version mismatch, resyncing...');
      return;
    }

    // Transform pending operations against received operation
    this.pendingOps = this.pendingOps.map((pendingOp) =>
      this.transform(pendingOp, op)
    );

    this.serverVersion++;
    return op;
  }

  private transform(op1: Operation, op2: Operation): Operation {
    // Simplified transformation logic
    // In production, use a library like ShareDB
    return op1;
  }
}
```

## 22.6 Connection Reliability

### Heartbeat and Reconnection

```typescript
// Enhanced WebSocket service with heartbeat
export class ReliableWebSocketService extends WebSocketService {
  private heartbeatInterval?: NodeJS.Timeout;
  private heartbeatTimeout?: NodeJS.Timeout;
  private missedHeartbeats = 0;

  connect(url: string, token?: string) {
    super.connect(url, token);

    // Start heartbeat
    this.startHeartbeat();
  }

  private startHeartbeat() {
    // Send ping every 30 seconds
    this.heartbeatInterval = setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: 'ping' }));

        // Expect pong within 5 seconds
        this.heartbeatTimeout = setTimeout(() => {
          this.missedHeartbeats++;

          if (this.missedHeartbeats >= 3) {
            console.warn('Missed 3 heartbeats, reconnecting...');
            this.ws?.close();
          }
        }, 5000);
      }
    }, 30000);

    // Listen for pong
    this.on('pong', () => {
      this.missedHeartbeats = 0;
      if (this.heartbeatTimeout) {
        clearTimeout(this.heartbeatTimeout);
      }
    });
  }

  disconnect() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
    if (this.heartbeatTimeout) {
      clearTimeout(this.heartbeatTimeout);
    }
    super.disconnect();
  }
}
```

### Offline Queue

```typescript
// hooks/useOfflineQueue.ts
export function useOfflineQueue() {
  const [queue, setQueue] = useState<Array<{ type: string; payload: any }>>([]);
  const { isConnected, send } = useWebSocket();

  const enqueue = useCallback((type: string, payload: any) => {
    if (isConnected) {
      send(type, payload);
    } else {
      setQueue((prev) => [...prev, { type, payload }]);

      // Persist to localStorage
      localStorage.setItem('ws_queue', JSON.stringify(queue));
    }
  }, [isConnected, send, queue]);

  useEffect(() => {
    if (isConnected && queue.length > 0) {
      // Send queued messages
      queue.forEach(({ type, payload }) => {
        send(type, payload);
      });

      setQueue([]);
      localStorage.removeItem('ws_queue');
    }
  }, [isConnected, queue, send]);

  useEffect(() => {
    // Restore queue from localStorage
    const stored = localStorage.getItem('ws_queue');
    if (stored) {
      try {
        setQueue(JSON.parse(stored));
      } catch (err) {
        console.error('Failed to restore queue:', err);
      }
    }
  }, []);

  return { enqueue, queueLength: queue.length };
}
```

## Real-World Scenario: Building Collaborative Whiteboard

### The Challenge

Build a real-time collaborative whiteboard:
- Multiple users drawing simultaneously
- Cursor tracking
- Presence indicators
- Undo/redo
- Connection resilience

### Senior Approach

```typescript
// Complete implementation

// 1. WebSocket connection with heartbeat
const ws = new ReliableWebSocketService();

// 2. Operational transformation for conflict resolution
const otClient = new OTClient();

// 3. Cursor tracking
const cursors = useCollaborativeCursors('whiteboard-1');

// 4. Drawing operations
const draw = (operation: DrawOperation) => {
  // Apply locally for instant feedback
  applyDrawOperation(operation);

  // Send to server
  otClient.sendOperation(operation);
};

// 5. Receive remote operations
useEffect(() => {
  const unsubscribe = subscribe('draw_operation', (op) => {
    const transformed = otClient.receiveOperation(op);
    applyDrawOperation(transformed);
  });

  return unsubscribe;
}, []);

// 6. Offline queue
const { enqueue } = useOfflineQueue();

// 7. Presence tracking
const { onlineUsers } = usePresence('whiteboard-1');
```

## Chapter Exercise: Build Real-Time Feature

Create a real-time collaborative application:

**Requirements:**
1. WebSocket connection with reconnection
2. Real-time data updates
3. Presence/online status
4. Typing indicators (for chat) OR cursor tracking (for collaboration)
5. Connection status UI
6. Offline queue
7. Heartbeat mechanism
8. Error handling

**Bonus:**
- Operational transformation
- Collaborative editing
- Conflict resolution
- Performance optimization

## Review Checklist

- [ ] WebSocket connection reliable
- [ ] Reconnection logic implemented
- [ ] Heartbeat/ping-pong working
- [ ] Real-time updates functional
- [ ] Presence tracking accurate
- [ ] Connection status shown to user
- [ ] Offline queue implemented
- [ ] Error handling in place
- [ ] Performance optimized
- [ ] Memory leaks prevented

## Key Takeaways

1. **WebSockets for bidirectional communication** - Better than polling
2. **SSE for server-to-client updates** - Simpler when appropriate
3. **Implement reconnection logic** - Connections will drop
4. **Use heartbeats** - Detect dead connections
5. **Queue messages when offline** - Better UX
6. **Presence requires careful state management** - Handle joins/leaves
7. **Throttle high-frequency events** - Cursor movements, typing

## Further Reading

- WebSocket API documentation
- Server-Sent Events specification
- Operational Transformation algorithms
- Socket.io documentation
- Real-time collaboration patterns

## Next Chapter

[Chapter 23: Code Review Excellence](./23-code-review-excellence.md)
