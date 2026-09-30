# Part 048: WebSocket และ Real-time Communication ใน React Native

## สารบัญ
1. [แนะนำ WebSocket](#introduction)
2. [WebSocket Basics](#websocket-basics)
3. [Socket.io-client](#socketio)
4. [Room System](#room-system)
5. [Reconnection Logic](#reconnection)
6. [Workshop: Real-time Chat App](#workshop)

---

## 1. แนะนำ WebSocket {#introduction}

WebSocket เป็น protocol สำหรับการสื่อสารแบบ real-time สองทิศทาง (full-duplex) ระหว่าง client และ server

### เปรียบเทียบ HTTP vs WebSocket

```
HTTP Request/Response:
Client --[Request]--> Server
Client <--[Response]-- Server
(ต้อง request ทุกครั้งที่ต้องการข้อมูลใหม่)

WebSocket:
Client <===[Connected]===> Server
Client <---[Message]------ Server  (server ส่งได้ทันที)
Client ------[Message]---> Server  (client ส่งได้ทันที)
(เชื่อมต่อค้างไว้ตลอด)
```

### ติดตั้ง

```bash
# Socket.io client
npm install socket.io-client

# Types
npm install @types/socket.io-client

# สำหรับ server (Node.js)
npm install socket.io
```

---

## 2. WebSocket Basics {#websocket-basics}

### การใช้งาน WebSocket API พื้นฐาน

```typescript
// src/services/WebSocketService.ts
class WebSocketService {
  private ws: WebSocket | null = null;
  private url: string;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private listeners: Map<string, Function[]> = new Map();
  private isConnecting = false;

  constructor(url: string) {
    this.url = url;
  }

  connect(): void {
    if (this.isConnecting || this.ws?.readyState === WebSocket.OPEN) return;

    this.isConnecting = true;
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.isConnecting = false;
      this.emit('connect', null);
    };

    this.ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        this.emit(data.type, data.payload);
      } catch (error) {
        console.error('Parse message error:', error);
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.emit('error', error);
    };

    this.ws.onclose = (event) => {
      console.log('WebSocket closed:', event.code, event.reason);
      this.isConnecting = false;
      this.emit('disconnect', { code: event.code, reason: event.reason });

      // Auto reconnect
      if (event.code !== 1000) { // 1000 = normal closure
        this.scheduleReconnect();
      }
    };
  }

  disconnect(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }
    this.ws?.close(1000, 'Client disconnect');
    this.ws = null;
  }

  send(type: string, payload: any): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({ type, payload }));
    } else {
      console.warn('WebSocket not connected');
    }
  }

  on(event: string, callback: Function): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event)!.push(callback);

    // Return unsubscribe function
    return () => {
      const callbacks = this.listeners.get(event) || [];
      this.listeners.set(event, callbacks.filter(cb => cb !== callback));
    };
  }

  private emit(event: string, data: any): void {
    const callbacks = this.listeners.get(event) || [];
    callbacks.forEach(cb => cb(data));
  }

  private scheduleReconnect(delay = 3000): void {
    this.reconnectTimer = setTimeout(() => {
      console.log('Attempting reconnect...');
      this.connect();
    }, delay);
  }

  get isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }
}

export default WebSocketService;
```

---

## 3. Socket.io-client {#socketio}

### การตั้งค่า Socket.io

```typescript
// src/services/SocketService.ts
import { io, Socket } from 'socket.io-client';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface ServerToClientEvents {
  message: (data: ChatMessage) => void;
  userOnline: (userId: string) => void;
  userOffline: (userId: string) => void;
  typing: (data: { userId: string; chatId: string }) => void;
  stopTyping: (data: { userId: string; chatId: string }) => void;
  messageRead: (data: { messageId: string; userId: string }) => void;
  notification: (data: Notification) => void;
}

interface ClientToServerEvents {
  joinRoom: (roomId: string) => void;
  leaveRoom: (roomId: string) => void;
  sendMessage: (data: SendMessageData) => void;
  typing: (chatId: string) => void;
  stopTyping: (chatId: string) => void;
  markAsRead: (messageId: string) => void;
}

interface ChatMessage {
  id: string;
  chatId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image' | 'file';
  timestamp: string;
  readBy: string[];
}

interface SendMessageData {
  chatId: string;
  content: string;
  type: 'text' | 'image' | 'file';
  fileUrl?: string;
}

class SocketService {
  private static instance: SocketService;
  private socket: Socket<ServerToClientEvents, ClientToServerEvents> | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  static getInstance(): SocketService {
    if (!SocketService.instance) {
      SocketService.instance = new SocketService();
    }
    return SocketService.instance;
  }

  async connect(userId: string): Promise<void> {
    const token = await AsyncStorage.getItem('auth_token');

    this.socket = io('https://api.example.com', {
      auth: { token, userId },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      timeout: 10000,
      autoConnect: true,
    });

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      console.log('Socket connected:', this.socket?.id);
      this.reconnectAttempts = 0;
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Socket disconnected:', reason);
      if (reason === 'io server disconnect') {
        // Server initiated disconnect - reconnect manually
        this.socket?.connect();
      }
    });

    this.socket.on('connect_error', (error) => {
      console.error('Connection error:', error);
      this.reconnectAttempts++;
      
      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        console.error('Max reconnect attempts reached');
        this.socket?.disconnect();
      }
    });
  }

  disconnect(): void {
    this.socket?.disconnect();
    this.socket = null;
  }

  // Room Management
  joinRoom(roomId: string): void {
    this.socket?.emit('joinRoom', roomId);
  }

  leaveRoom(roomId: string): void {
    this.socket?.emit('leaveRoom', roomId);
  }

  // Messages
  sendMessage(data: SendMessageData): void {
    this.socket?.emit('sendMessage', data);
  }

  onMessage(callback: (message: ChatMessage) => void): () => void {
    this.socket?.on('message', callback);
    return () => this.socket?.off('message', callback);
  }

  // Typing Indicators
  startTyping(chatId: string): void {
    this.socket?.emit('typing', chatId);
  }

  stopTyping(chatId: string): void {
    this.socket?.emit('stopTyping', chatId);
  }

  onTyping(callback: (data: { userId: string; chatId: string }) => void): () => void {
    this.socket?.on('typing', callback);
    return () => this.socket?.off('typing', callback);
  }

  // Online Status
  onUserOnline(callback: (userId: string) => void): () => void {
    this.socket?.on('userOnline', callback);
    return () => this.socket?.off('userOnline', callback);
  }

  onUserOffline(callback: (userId: string) => void): () => void {
    this.socket?.on('userOffline', callback);
    return () => this.socket?.off('userOffline', callback);
  }

  // Read Receipts
  markAsRead(messageId: string): void {
    this.socket?.emit('markAsRead', messageId);
  }

  onMessageRead(callback: (data: { messageId: string; userId: string }) => void): () => void {
    this.socket?.on('messageRead', callback);
    return () => this.socket?.off('messageRead', callback);
  }

  get isConnected(): boolean {
    return this.socket?.connected ?? false;
  }
}

export default SocketService.getInstance();
```

---

## 4. Room System {#room-system}

### Socket.io Server (Node.js)

```javascript
// server/index.js
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: '*',
    methods: ['GET', 'POST'],
  },
});

// JWT Authentication Middleware
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token;
    if (!token) throw new Error('No token');
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.userId;
    socket.username = decoded.username;
    next();
  } catch (error) {
    next(new Error('Authentication failed'));
  }
});

// Track online users
const onlineUsers = new Map();

io.on('connection', (socket) => {
  const { userId, username } = socket;
  console.log(`User connected: ${username} (${userId})`);

  // Track online status
  onlineUsers.set(userId, {
    socketId: socket.id,
    username,
    connectedAt: new Date(),
  });

  // Notify others of online status
  socket.broadcast.emit('userOnline', userId);

  // Join personal room (สำหรับรับ private messages)
  socket.join(`user:${userId}`);

  // Join Chat Room
  socket.on('joinRoom', async (roomId) => {
    try {
      // ตรวจสอบสิทธิ์
      const hasAccess = await checkRoomAccess(userId, roomId);
      if (!hasAccess) {
        socket.emit('error', { message: 'ไม่มีสิทธิ์เข้าถึง room นี้' });
        return;
      }

      socket.join(roomId);
      console.log(`${username} joined room: ${roomId}`);

      // แจ้ง members ใน room
      socket.to(roomId).emit('userJoined', {
        userId,
        username,
        timestamp: new Date(),
      });

      // ส่ง online members ไปให้ user
      const roomMembers = await getRoomMembers(roomId);
      const onlineMembers = roomMembers.filter(m => onlineUsers.has(m.id));
      socket.emit('roomMembers', onlineMembers);
    } catch (error) {
      socket.emit('error', { message: error.message });
    }
  });

  // Leave Room
  socket.on('leaveRoom', (roomId) => {
    socket.leave(roomId);
    socket.to(roomId).emit('userLeft', { userId, username });
  });

  // Send Message
  socket.on('sendMessage', async (data) => {
    try {
      const { chatId, content, type, fileUrl } = data;

      // ตรวจสอบสิทธิ์
      const hasAccess = await checkRoomAccess(userId, chatId);
      if (!hasAccess) {
        socket.emit('error', { message: 'ไม่มีสิทธิ์ส่งข้อความ' });
        return;
      }

      // บันทึกลง database
      const message = await saveMessage({
        chatId,
        senderId: userId,
        content,
        type,
        fileUrl,
      });

      // ส่งหา members ใน room
      io.to(chatId).emit('message', {
        ...message,
        sender: { id: userId, username },
      });

      // ส่ง push notification หา members ที่ offline
      await notifyOfflineMembers(chatId, message, userId);
    } catch (error) {
      socket.emit('error', { message: error.message });
    }
  });

  // Typing Events
  socket.on('typing', (chatId) => {
    socket.to(chatId).emit('typing', { userId, chatId });
  });

  socket.on('stopTyping', (chatId) => {
    socket.to(chatId).emit('stopTyping', { userId, chatId });
  });

  // Mark as Read
  socket.on('markAsRead', async (messageId) => {
    await markMessageAsRead(messageId, userId);
    const message = await getMessageById(messageId);
    io.to(message.chatId).emit('messageRead', { messageId, userId });
  });

  // Disconnect
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${username} (${reason})`);
    onlineUsers.delete(userId);
    socket.broadcast.emit('userOffline', userId);
  });
});

httpServer.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

---

## 5. Reconnection Logic {#reconnection}

### Reconnection Hook

```typescript
// src/hooks/useSocket.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import { AppState, AppStateStatus, NetInfo } from '@react-native-community/netinfo';
import SocketService from '../services/SocketService';

type ConnectionStatus = 'connected' | 'disconnected' | 'connecting' | 'reconnecting';

export function useSocket(userId: string) {
  const [status, setStatus] = useState<ConnectionStatus>('disconnected');
  const [connectionAttempt, setConnectionAttempt] = useState(0);
  const appState = useRef(AppState.currentState);
  const reconnectTimer = useRef<ReturnType<typeof setTimeout>>();

  const connect = useCallback(async () => {
    setStatus('connecting');
    try {
      await SocketService.connect(userId);
      setStatus('connected');
    } catch (error) {
      setStatus('disconnected');
      scheduleReconnect();
    }
  }, [userId]);

  const scheduleReconnect = useCallback((attempt = 1) => {
    const delay = Math.min(1000 * Math.pow(2, attempt), 30000); // exponential backoff
    console.log(`Reconnecting in ${delay}ms (attempt ${attempt})`);
    
    setStatus('reconnecting');
    reconnectTimer.current = setTimeout(() => {
      setConnectionAttempt(attempt);
      connect();
    }, delay);
  }, [connect]);

  // Handle App State changes
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (nextState: AppStateStatus) => {
      if (appState.current.match(/inactive|background/) && nextState === 'active') {
        // App กลับมา foreground
        if (!SocketService.isConnected) {
          connect();
        }
      } else if (nextState.match(/inactive|background/)) {
        // App ไป background - ไม่ disconnect เพื่อรับ messages
      }
      appState.current = nextState;
    });

    return () => subscription.remove();
  }, [connect]);

  // Handle Network changes
  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      if (state.isConnected && !SocketService.isConnected) {
        connect();
      }
    });

    return () => unsubscribe();
  }, [connect]);

  // Initial connection
  useEffect(() => {
    connect();

    return () => {
      if (reconnectTimer.current) {
        clearTimeout(reconnectTimer.current);
      }
      SocketService.disconnect();
    };
  }, []);

  return { status, connectionAttempt };
}
```

---

## 6. Workshop: Real-time Chat App {#workshop}

```typescript
// src/screens/ChatScreen.tsx
import React, { useState, useEffect, useCallback, useRef } from 'react';
import {
  View,
  Text,
  FlatList,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  Image,
  ActivityIndicator,
  Alert,
} from 'react-native';
import Icon from 'react-native-vector-icons/MaterialIcons';
import SocketService from '../services/SocketService';
import { useSocket } from '../hooks/useSocket';
import { useAuth } from '../hooks/useAuth';

interface Message {
  id: string;
  chatId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image';
  timestamp: Date;
  readBy: string[];
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
  sender?: {
    id: string;
    username: string;
    avatarUrl?: string;
  };
}

interface ChatUser {
  id: string;
  username: string;
  avatarUrl?: string;
  isOnline: boolean;
  isTyping: boolean;
  lastSeen?: Date;
}

interface ChatScreenProps {
  chatId: string;
  chatName: string;
  isGroup: boolean;
}

const ChatScreen: React.FC<ChatScreenProps> = ({ chatId, chatName, isGroup }) => {
  const { user } = useAuth();
  const { status: socketStatus } = useSocket(user.id);
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputText, setInputText] = useState('');
  const [isLoadingMore, setIsLoadingMore] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [chatUsers, setChatUsers] = useState<ChatUser[]>([]);
  const flatListRef = useRef<FlatList>(null);
  const typingTimer = useRef<ReturnType<typeof setTimeout>>();
  const isTyping = useRef(false);

  useEffect(() => {
    // Join room
    SocketService.joinRoom(chatId);

    // Load initial messages
    loadMessages();

    // Subscribe to events
    const unsubscribeMessage = SocketService.onMessage(handleNewMessage);
    const unsubscribeTyping = SocketService.onTyping(handleTyping);
    const unsubscribeStopTyping = SocketService.onTyping(handleStopTyping);
    const unsubscribeOnline = SocketService.onUserOnline(handleUserOnline);
    const unsubscribeOffline = SocketService.onUserOffline(handleUserOffline);
    const unsubscribeRead = SocketService.onMessageRead(handleMessageRead);

    return () => {
      SocketService.leaveRoom(chatId);
      unsubscribeMessage();
      unsubscribeTyping();
      unsubscribeStopTyping();
      unsubscribeOnline();
      unsubscribeOffline();
      unsubscribeRead();
    };
  }, [chatId]);

  const loadMessages = async (before?: string) => {
    try {
      if (before) setIsLoadingMore(true);

      const response = await fetch(
        `https://api.example.com/chats/${chatId}/messages${before ? `?before=${before}` : ''}`,
        { headers: { Authorization: `Bearer ${user.token}` } }
      );
      const data = await response.json();

      if (before) {
        setMessages(prev => [...data.messages.reverse(), ...prev]);
        setHasMore(data.hasMore);
        setIsLoadingMore(false);
      } else {
        setMessages(data.messages.reverse());
        setHasMore(data.hasMore);
      }
    } catch (error) {
      console.error('Load messages error:', error);
    }
  };

  const handleNewMessage = useCallback((message: Message) => {
    if (message.chatId !== chatId) return;

    setMessages(prev => {
      // ตรวจสอบว่ามี message นี้แล้วหรือยัง
      if (prev.find(m => m.id === message.id)) return prev;
      return [...prev, message];
    });

    // Mark as read
    SocketService.markAsRead(message.id);

    // Scroll to bottom
    setTimeout(() => {
      flatListRef.current?.scrollToEnd({ animated: true });
    }, 100);
  }, [chatId]);

  const handleTyping = useCallback(({ userId, chatId: typingChatId }: { userId: string; chatId: string }) => {
    if (typingChatId !== chatId) return;
    setChatUsers(prev =>
      prev.map(u => u.id === userId ? { ...u, isTyping: true } : u)
    );
  }, [chatId]);

  const handleStopTyping = useCallback(({ userId, chatId: typingChatId }: { userId: string; chatId: string }) => {
    if (typingChatId !== chatId) return;
    setChatUsers(prev =>
      prev.map(u => u.id === userId ? { ...u, isTyping: false } : u)
    );
  }, [chatId]);

  const handleUserOnline = useCallback((userId: string) => {
    setChatUsers(prev =>
      prev.map(u => u.id === userId ? { ...u, isOnline: true } : u)
    );
  }, []);

  const handleUserOffline = useCallback((userId: string) => {
    setChatUsers(prev =>
      prev.map(u => u.id === userId ? { ...u, isOnline: false, lastSeen: new Date() } : u)
    );
  }, []);

  const handleMessageRead = useCallback(({ messageId, userId }: { messageId: string; userId: string }) => {
    setMessages(prev =>
      prev.map(m =>
        m.id === messageId
          ? { ...m, readBy: [...m.readBy, userId] }
          : m
      )
    );
  }, []);

  const sendMessage = useCallback(() => {
    const content = inputText.trim();
    if (!content) return;

    // Optimistic update
    const tempId = `temp_${Date.now()}`;
    const tempMessage: Message = {
      id: tempId,
      chatId,
      senderId: user.id,
      content,
      type: 'text',
      timestamp: new Date(),
      readBy: [user.id],
      status: 'sending',
      sender: { id: user.id, username: user.username },
    };

    setMessages(prev => [...prev, tempMessage]);
    setInputText('');

    // Stop typing
    handleTypingInput('');

    // Send via socket
    SocketService.sendMessage({ chatId, content, type: 'text' });

    setTimeout(() => {
      flatListRef.current?.scrollToEnd({ animated: true });
    }, 100);
  }, [inputText, chatId, user]);

  const handleTypingInput = (text: string) => {
    setInputText(text);

    if (text.length > 0 && !isTyping.current) {
      isTyping.current = true;
      SocketService.startTyping(chatId);
    }

    if (typingTimer.current) {
      clearTimeout(typingTimer.current);
    }

    typingTimer.current = setTimeout(() => {
      if (isTyping.current) {
        isTyping.current = false;
        SocketService.stopTyping(chatId);
      }
    }, 1000);
  };

  const getTypingUsers = () => {
    return chatUsers.filter(u => u.isTyping && u.id !== user.id);
  };

  const formatTime = (date: Date) => {
    return new Date(date).toLocaleTimeString('th-TH', {
      hour: '2-digit',
      minute: '2-digit',
    });
  };

  const getMessageStatus = (message: Message) => {
    if (message.status === 'sending') return '⏳';
    if (message.status === 'failed') return '❌';
    if (message.readBy.length > 1) return '✓✓'; // Read
    return '✓'; // Sent
  };

  const renderMessage = ({ item, index }: { item: Message; index: number }) => {
    const isMyMessage = item.senderId === user.id;
    const showAvatar = !isMyMessage && (
      index === 0 || messages[index - 1]?.senderId !== item.senderId
    );
    const showSenderName = isGroup && !isMyMessage && showAvatar;

    return (
      <View style={[
        styles.messageRow,
        isMyMessage ? styles.myMessageRow : styles.theirMessageRow,
      ]}>
        {!isMyMessage && (
          <View style={styles.avatarSpace}>
            {showAvatar && (
              item.sender?.avatarUrl
                ? <Image source={{ uri: item.sender.avatarUrl }} style={styles.avatar} />
                : <View style={styles.avatarPlaceholder}>
                    <Text style={styles.avatarText}>
                      {item.sender?.username?.[0]?.toUpperCase()}
                    </Text>
                  </View>
            )}
          </View>
        )}

        <View style={[
          styles.messageBubble,
          isMyMessage ? styles.myBubble : styles.theirBubble,
        ]}>
          {showSenderName && (
            <Text style={styles.senderName}>{item.sender?.username}</Text>
          )}
          <Text style={[styles.messageText, isMyMessage && styles.myMessageText]}>
            {item.content}
          </Text>
          <View style={styles.messageFooter}>
            <Text style={[styles.messageTime, isMyMessage && styles.myMessageTime]}>
              {formatTime(item.timestamp)}
            </Text>
            {isMyMessage && (
              <Text style={styles.messageStatus}>{getMessageStatus(item)}</Text>
            )}
          </View>
        </View>
      </View>
    );
  };

  const typingUsers = getTypingUsers();

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}
    >
      {/* Connection Status */}
      {socketStatus !== 'connected' && (
        <View style={[
          styles.statusBanner,
          socketStatus === 'reconnecting' ? styles.reconnectingBanner : styles.disconnectedBanner,
        ]}>
          <ActivityIndicator size="small" color="white" />
          <Text style={styles.statusText}>
            {socketStatus === 'reconnecting' ? 'กำลังเชื่อมต่อใหม่...' : 'ไม่มีการเชื่อมต่อ'}
          </Text>
        </View>
      )}

      {/* Messages */}
      <FlatList
        ref={flatListRef}
        data={messages}
        renderItem={renderMessage}
        keyExtractor={item => item.id}
        contentContainerStyle={styles.messageList}
        onEndReached={() => {
          if (hasMore && !isLoadingMore && messages.length > 0) {
            loadMessages(messages[0].id);
          }
        }}
        onEndReachedThreshold={0.2}
        ListHeaderComponent={
          isLoadingMore ? <ActivityIndicator style={{ padding: 16 }} /> : null
        }
        ListEmptyComponent={
          <View style={styles.emptyChat}>
            <Text style={styles.emptyChatIcon}>💬</Text>
            <Text style={styles.emptyChatText}>เริ่มการสนทนา!</Text>
          </View>
        }
        inverted={false}
        onContentSizeChange={() => {
          flatListRef.current?.scrollToEnd({ animated: false });
        }}
      />

      {/* Typing Indicator */}
      {typingUsers.length > 0 && (
        <View style={styles.typingContainer}>
          <View style={styles.typingBubble}>
            <View style={styles.typingDots}>
              {[0, 1, 2].map(i => (
                <View key={i} style={[styles.typingDot, { animationDelay: `${i * 0.2}s` }]} />
              ))}
            </View>
          </View>
          <Text style={styles.typingText}>
            {typingUsers.map(u => u.username).join(', ')} กำลังพิมพ์...
          </Text>
        </View>
      )}

      {/* Input Area */}
      <View style={styles.inputContainer}>
        <TouchableOpacity style={styles.attachButton}>
          <Icon name="attach-file" size={24} color="#9E9E9E" />
        </TouchableOpacity>

        <TextInput
          style={styles.input}
          value={inputText}
          onChangeText={handleTypingInput}
          placeholder="พิมพ์ข้อความ..."
          multiline
          maxLength={1000}
          returnKeyType="default"
        />

        {inputText.trim().length > 0 ? (
          <TouchableOpacity style={styles.sendButton} onPress={sendMessage}>
            <Icon name="send" size={22} color="white" />
          </TouchableOpacity>
        ) : (
          <TouchableOpacity style={styles.micButton}>
            <Icon name="mic" size={24} color="#9E9E9E" />
          </TouchableOpacity>
        )}
      </View>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#ECEFF1' },
  statusBanner: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 8,
    gap: 8,
  },
  reconnectingBanner: { backgroundColor: '#FF9800' },
  disconnectedBanner: { backgroundColor: '#F44336' },
  statusText: { color: 'white', fontSize: 13 },
  messageList: { padding: 16, gap: 4 },
  messageRow: {
    flexDirection: 'row',
    marginVertical: 2,
    alignItems: 'flex-end',
    gap: 8,
  },
  myMessageRow: { justifyContent: 'flex-end' },
  theirMessageRow: { justifyContent: 'flex-start' },
  avatarSpace: { width: 32 },
  avatar: { width: 32, height: 32, borderRadius: 16 },
  avatarPlaceholder: {
    width: 32,
    height: 32,
    borderRadius: 16,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
  },
  avatarText: { color: 'white', fontWeight: 'bold', fontSize: 14 },
  messageBubble: {
    maxWidth: '75%',
    padding: 10,
    borderRadius: 16,
    elevation: 1,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  myBubble: {
    backgroundColor: '#2196F3',
    borderBottomRightRadius: 4,
  },
  theirBubble: {
    backgroundColor: 'white',
    borderBottomLeftRadius: 4,
  },
  senderName: { fontSize: 12, color: '#2196F3', fontWeight: 'bold', marginBottom: 2 },
  messageText: { fontSize: 15, color: '#333', lineHeight: 20 },
  myMessageText: { color: 'white' },
  messageFooter: { flexDirection: 'row', alignItems: 'center', justifyContent: 'flex-end', gap: 4, marginTop: 4 },
  messageTime: { fontSize: 11, color: '#9E9E9E' },
  myMessageTime: { color: 'rgba(255,255,255,0.7)' },
  messageStatus: { fontSize: 12, color: 'rgba(255,255,255,0.8)' },
  typingContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 6,
    gap: 8,
  },
  typingBubble: {
    backgroundColor: 'white',
    padding: 10,
    borderRadius: 16,
    borderBottomLeftRadius: 4,
  },
  typingDots: { flexDirection: 'row', gap: 4 },
  typingDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#9E9E9E',
  },
  typingText: { fontSize: 12, color: '#9E9E9E', fontStyle: 'italic' },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    padding: 8,
    paddingHorizontal: 12,
    backgroundColor: 'white',
    borderTopWidth: 1,
    borderTopColor: '#E0E0E0',
    gap: 8,
  },
  attachButton: { width: 40, height: 40, justifyContent: 'center', alignItems: 'center' },
  input: {
    flex: 1,
    backgroundColor: '#F5F5F5',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    maxHeight: 100,
    fontSize: 15,
  },
  sendButton: {
    width: 40,
    height: 40,
    borderRadius: 20,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
  },
  micButton: { width: 40, height: 40, justifyContent: 'center', alignItems: 'center' },
  emptyChat: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 48, gap: 12 },
  emptyChatIcon: { fontSize: 48 },
  emptyChatText: { color: '#9E9E9E', fontSize: 16 },
});

export default ChatScreen;
```

---

## Tips และ Best Practices

### 1. Message Queue สำหรับ Offline Support

```typescript
// Queue messages เมื่อ offline
class MessageQueue {
  private queue: SendMessageData[] = [];

  addToQueue(message: SendMessageData): void {
    this.queue.push(message);
    this.persistQueue();
  }

  async flushQueue(): Promise<void> {
    if (!SocketService.isConnected) return;
    
    const pendingMessages = [...this.queue];
    this.queue = [];
    
    for (const message of pendingMessages) {
      SocketService.sendMessage(message);
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }

  private persistQueue(): void {
    AsyncStorage.setItem('message_queue', JSON.stringify(this.queue));
  }
}
```

### 2. Exponential Backoff

```typescript
function getReconnectDelay(attempt: number): number {
  return Math.min(1000 * Math.pow(2, attempt), 30000);
}
// attempt 0: 1000ms
// attempt 1: 2000ms
// attempt 2: 4000ms
// attempt 3: 8000ms
// attempt 4: 16000ms
// attempt 5+: 30000ms (max)
```

### สรุป

- ใช้ Socket.io แทน WebSocket ดิบ เพราะมี features เพิ่มเติม
- Implement reconnection logic ด้วย exponential backoff
- Handle app state (foreground/background) อย่างเหมาะสม
- ใช้ Optimistic Updates เพื่อ UX ที่ดีขึ้น
- Queue messages เมื่อ offline
- Mark messages as read เมื่อ user เห็น
