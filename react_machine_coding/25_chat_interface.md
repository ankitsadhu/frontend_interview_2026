# 25. Chat Interface

## Problem Description
Design a Chat Interface that includes a message list, an input area to send messages, automatic scroll-to-bottom on new messages, typing indicators, and timestamps.

**Key Concepts Tested:** `useRef` for DOM manipulation (scrolling), `useEffect` synchronization, array/list management, optimistic updates.

## React Component Implementation Outline

```tsx
import React, { useState, useRef, useEffect } from 'react';
import './ChatInterface.css';

interface Message {
  id: string;
  text: string;
  sender: 'me' | 'other';
  timestamp: Date;
}

export const ChatInterface: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputText, setInputText] = useState('');
  const [isTyping, setIsTyping] = useState(false);
  
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  // Scroll downwards when new messages / typing state change
  useEffect(() => {
    scrollToBottom();
  }, [messages, isTyping]);

  const handleSend = (e: React.FormEvent) => {
    e.preventDefault();
    if (!inputText.trim()) return;

    const newMsg: Message = {
      id: Date.now().toString(),
      text: inputText,
      sender: 'me',
      timestamp: new Date()
    };

    setMessages(prev => [...prev, newMsg]);
    setInputText('');
    
    // Simulate incoming API response
    setIsTyping(true);
    setTimeout(() => {
      setIsTyping(false);
      setMessages(prev => [...prev, {
        id: (Date.now() + 1).toString(),
        text: 'Hello, I process your request over the simulated network!',
        sender: 'other',
        timestamp: new Date()
      }]);
    }, 1500);
  };

  return (
    <div className="chat-container">
      <div className="message-list" ref={containerRef}>
        {messages.map((msg) => (
          <div key={msg.id} className={`message ${msg.sender}`}>
            <span className="text">{msg.text}</span>
            <span className="time">
              {msg.timestamp.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
            </span>
          </div>
        ))}
        {isTyping && <div className="typing-indicator">Other is typing...</div>}
        <div ref={messagesEndRef} />
      </div>

      <form onSubmit={handleSend} className="chat-input-area">
        <input 
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          placeholder="Type your message..."
          autoFocus
        />
        <button type="submit" disabled={!inputText.trim()}>Send</button>
      </form>
    </div>
  );
};
```

## Vitest Unit Test Cases

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { ChatInterface } from './ChatInterface';

// Mock scrollIntoView which isn't available in JSDOM
window.HTMLElement.prototype.scrollIntoView = vi.fn();

describe('ChatInterface Component', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.runOnlyPendingTimers();
    vi.useRealTimers();
  });

  it('renders input area correctly', () => {
    render(<ChatInterface />);
    expect(screen.getByPlaceholderText('Type your message...')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /send/i })).toBeDisabled();
  });

  it('adds user message to the list when sent', () => {
    render(<ChatInterface />);
    const input = screen.getByPlaceholderText('Type your message...');
    
    fireEvent.change(input, { target: { value: 'Hello world' } });
    fireEvent.submit(screen.getByRole('button', { name: /send/i }));
    
    expect(screen.getByText('Hello world')).toBeInTheDocument();
    expect(input).toHaveValue(''); // input cleared after submit
  });

  it('displays typing indicator and eventually receives delayed response', async () => {
    render(<ChatInterface />);
    
    const input = screen.getByPlaceholderText('Type your message...');
    fireEvent.change(input, { target: { value: 'Ping' } });
    fireEvent.click(screen.getByRole('button', { name: /send/i }));
    
    // Assert typing indicator instantly appearing
    expect(screen.getByText('Other is typing...')).toBeInTheDocument();
    
    vi.advanceTimersByTime(1500);
    
    await waitFor(() => {
      expect(screen.queryByText('Other is typing...')).not.toBeInTheDocument();
      expect(screen.getByText(/process your request over the simulated network/i)).toBeInTheDocument();
    });
  });
});
```

## MANG-Level Cross Questions

1. **Scroll UX Issue:** `scrollIntoView` forces the chat downwards on every message render. If the user was manually scrolling up to read old history and a new message arrives, the UI snaps away forcibly, losing their reading place. How do you implement "scroll to bottom *only if they are already near the bottom*"?
2. **IntersectionObserver vs Scroll Events:** How would you design lazy-fetching of older messages (pagination) when the user reaches the absolute top? Should you use `IntersectionObserver` or a `onScroll` listener? What are their respective performance impacts?
3. **Data Throttling (Chat Storms):** In a Twitch-style chat, you might receive 200+ WebSocket messages per second. Updating React state for every single message will freeze the thread. How would you design a "message queue/buffer logic" and throttle batched React renders?
4. **WebSocket Integration & Hydration:** How would you rewrite the component architecture to integrate seamlessly with an active WebSocket connection? How do you handle connection drops and sync/rehydrate missing messages when the network reconnects?
5. **Security (XSS Mitigation):** If product requirements dictate supporting rich text (Markdown, raw HTML tags) parsing in your chat, how do you render this content dangerously while ensuring ZERO Cross-Site Scripting (XSS) vulnerabilities?
