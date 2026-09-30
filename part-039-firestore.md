# Part 039: Firestore Database

## Firestore Basics

### Firestore คืออะไร?

Firestore เป็น NoSQL document database จาก Firebase ที่:
- เก็บข้อมูลแบบ **Documents** ใน **Collections**
- รองรับ **Real-time updates** ผ่าน listeners
- มี **Offline support** built-in
- Scale ได้อัตโนมัติ
- มี security rules ที่ยืดหยุ่น

### Data Model

```
Firestore
├── users/           ← Collection
│   ├── user123/     ← Document
│   │   ├── name: "สมชาย"
│   │   ├── email: "somchai@example.com"
│   │   └── createdAt: Timestamp
│   └── user456/
│       └── ...
├── posts/
│   ├── post001/
│   │   ├── title: "หัวข้อ"
│   │   ├── body: "เนื้อหา"
│   │   ├── authorId: "user123"
│   │   └── comments/    ← Subcollection
│   │       ├── comment1/
│   │       └── ...
│   └── ...
```

---

## การติดตั้ง

```bash
npm install @react-native-firebase/app @react-native-firebase/firestore

# หรือ Web SDK
npm install firebase
```

### Initialize

```typescript
// src/services/firebase/firestore.ts
import firestore from '@react-native-firebase/firestore';

export const db = firestore();

// Collections
export const COLLECTIONS = {
  USERS: 'users',
  POSTS: 'posts',
  MESSAGES: 'messages',
  ROOMS: 'rooms',
} as const;
```

---

## CRUD Operations

### Create (เพิ่มข้อมูล)

```typescript
// src/services/firebase/postsService.ts
import firestore from '@react-native-firebase/firestore';
import auth from '@react-native-firebase/auth';

export interface Post {
  id?: string;
  title: string;
  body: string;
  authorId: string;
  authorName: string;
  tags: string[];
  likes: number;
  createdAt: firestore.Timestamp;
  updatedAt: firestore.Timestamp;
}

export const postsService = {
  // สร้าง Post
  create: async (data: Omit<Post, 'id' | 'createdAt' | 'updatedAt' | 'likes'>) => {
    const now = firestore.Timestamp.now();

    const postData: Omit<Post, 'id'> = {
      ...data,
      likes: 0,
      createdAt: now,
      updatedAt: now,
    };

    // addDoc - สร้าง document ด้วย auto ID
    const docRef = await firestore().collection('posts').add(postData);

    // หรือ setDoc - กำหนด ID เอง
    // await firestore().collection('posts').doc('my-custom-id').set(postData);

    return { id: docRef.id, ...postData };
  },

  // อ่าน Posts ทั้งหมด
  getAll: async (): Promise<Post[]> => {
    const snapshot = await firestore()
      .collection('posts')
      .orderBy('createdAt', 'desc')
      .limit(20)
      .get();

    return snapshot.docs.map((doc) => ({
      id: doc.id,
      ...doc.data(),
    })) as Post[];
  },

  // อ่าน Post เดียว
  getById: async (postId: string): Promise<Post | null> => {
    const doc = await firestore().collection('posts').doc(postId).get();

    if (!doc.exists) return null;

    return { id: doc.id, ...doc.data() } as Post;
  },

  // อัปเดต Post
  update: async (postId: string, updates: Partial<Post>) => {
    await firestore()
      .collection('posts')
      .doc(postId)
      .update({
        ...updates,
        updatedAt: firestore.Timestamp.now(),
      });
  },

  // ลบ Post
  delete: async (postId: string) => {
    await firestore().collection('posts').doc(postId).delete();
  },

  // Like Post (atomic increment)
  like: async (postId: string) => {
    await firestore()
      .collection('posts')
      .doc(postId)
      .update({
        likes: firestore.FieldValue.increment(1),
      });
  },
};
```

---

## Real-time Listeners

### onSnapshot - Listen to Changes

```typescript
// src/hooks/useRealtimePosts.ts
import { useState, useEffect, useRef } from 'react';
import firestore from '@react-native-firebase/firestore';
import { Post } from '../services/firebase/postsService';

export function useRealtimePosts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const unsubscribeRef = useRef<(() => void) | null>(null);

  useEffect(() => {
    setIsLoading(true);

    // Subscribe to changes
    unsubscribeRef.current = firestore()
      .collection('posts')
      .orderBy('createdAt', 'desc')
      .limit(50)
      .onSnapshot(
        (snapshot) => {
          const newPosts = snapshot.docs.map((doc) => ({
            id: doc.id,
            ...doc.data(),
          })) as Post[];

          setPosts(newPosts);
          setIsLoading(false);
        },
        (err) => {
          setError(err.message);
          setIsLoading(false);
        }
      );

    // Cleanup - unsubscribe เมื่อ component unmount
    return () => {
      unsubscribeRef.current?.();
    };
  }, []);

  return { posts, isLoading, error };
}
```

### Listen to Specific Document

```typescript
export function useRealtimePost(postId: string) {
  const [post, setPost] = useState<Post | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    if (!postId) return;

    const unsubscribe = firestore()
      .collection('posts')
      .doc(postId)
      .onSnapshot((doc) => {
        if (doc.exists) {
          setPost({ id: doc.id, ...doc.data() } as Post);
        } else {
          setPost(null);
        }
        setIsLoading(false);
      });

    return unsubscribe;
  }, [postId]);

  return { post, isLoading };
}
```

---

## Queries และ Filters

```typescript
// src/services/firebase/queryExamples.ts
import firestore from '@react-native-firebase/firestore';

export const queryExamples = {
  // =====================
  // Basic Queries
  // =====================

  // Where filter
  getPublishedPosts: () =>
    firestore()
      .collection('posts')
      .where('status', '==', 'published')
      .get(),

  // Multiple conditions
  getRecentHighLikedPosts: () =>
    firestore()
      .collection('posts')
      .where('likes', '>=', 100)
      .orderBy('likes', 'desc')
      .orderBy('createdAt', 'desc')
      .limit(10)
      .get(),

  // Array contains
  getPostsByTag: (tag: string) =>
    firestore()
      .collection('posts')
      .where('tags', 'array-contains', tag)
      .get(),

  // Array contains any
  getPostsByAnyTag: (tags: string[]) =>
    firestore()
      .collection('posts')
      .where('tags', 'array-contains-any', tags)
      .get(),

  // In array
  getPostsByIds: (ids: string[]) =>
    firestore()
      .collection('posts')
      .where(firestore.FieldPath.documentId(), 'in', ids)
      .get(),

  // =====================
  // Pagination
  // =====================

  // เริ่มหน้าแรก
  getFirstPage: () =>
    firestore()
      .collection('posts')
      .orderBy('createdAt', 'desc')
      .limit(10)
      .get(),

  // หน้าถัดไป (startAfter)
  getNextPage: (lastDoc: any) =>
    firestore()
      .collection('posts')
      .orderBy('createdAt', 'desc')
      .startAfter(lastDoc)
      .limit(10)
      .get(),

  // =====================
  // Aggregate Queries
  // =====================

  // นับจำนวน
  countPosts: async () => {
    const snapshot = await firestore()
      .collection('posts')
      .count()
      .get();
    return snapshot.data().count;
  },
};
```

---

## Subcollections

```typescript
// src/services/firebase/commentsService.ts
import firestore from '@react-native-firebase/firestore';

export interface Comment {
  id?: string;
  text: string;
  authorId: string;
  authorName: string;
  authorAvatar?: string;
  createdAt: firestore.Timestamp;
}

export const commentsService = {
  // เพิ่ม comment ใน post
  add: async (postId: string, comment: Omit<Comment, 'id' | 'createdAt'>) => {
    const docRef = await firestore()
      .collection('posts')
      .doc(postId)
      .collection('comments')  // Subcollection
      .add({
        ...comment,
        createdAt: firestore.Timestamp.now(),
      });

    return docRef.id;
  },

  // ดู comments ทั้งหมดของ post
  getByPost: async (postId: string): Promise<Comment[]> => {
    const snapshot = await firestore()
      .collection('posts')
      .doc(postId)
      .collection('comments')
      .orderBy('createdAt', 'asc')
      .get();

    return snapshot.docs.map((doc) => ({
      id: doc.id,
      ...doc.data(),
    })) as Comment[];
  },

  // Listen to comments
  listenToComments: (
    postId: string,
    onUpdate: (comments: Comment[]) => void
  ) => {
    return firestore()
      .collection('posts')
      .doc(postId)
      .collection('comments')
      .orderBy('createdAt', 'asc')
      .onSnapshot((snapshot) => {
        const comments = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        })) as Comment[];
        onUpdate(comments);
      });
  },

  delete: async (postId: string, commentId: string) => {
    await firestore()
      .collection('posts')
      .doc(postId)
      .collection('comments')
      .doc(commentId)
      .delete();
  },
};
```

---

## Transactions และ Batch Writes

### Transaction

```typescript
// ใช้สำหรับ operations ที่ต้อง atomic
const transferLikes = async (fromPostId: string, toPostId: string) => {
  await firestore().runTransaction(async (transaction) => {
    const fromRef = firestore().collection('posts').doc(fromPostId);
    const toRef = firestore().collection('posts').doc(toPostId);

    const fromDoc = await transaction.get(fromRef);
    const toDoc = await transaction.get(toRef);

    if (!fromDoc.exists || !toDoc.exists) {
      throw new Error('Post not found');
    }

    const fromLikes = fromDoc.data()?.likes || 0;

    transaction.update(fromRef, { likes: fromLikes - 1 });
    transaction.update(toRef, {
      likes: firestore.FieldValue.increment(1),
    });
  });
};
```

### Batch Write

```typescript
// ใช้สำหรับ หลาย operations พร้อมกัน
const batchDeletePosts = async (postIds: string[]) => {
  const batch = firestore().batch();

  postIds.forEach((id) => {
    const postRef = firestore().collection('posts').doc(id);
    batch.delete(postRef);
  });

  await batch.commit();
};

const batchCreatePosts = async (posts: Omit<Post, 'id'>[]) => {
  const batch = firestore().batch();

  posts.forEach((post) => {
    const postRef = firestore().collection('posts').doc();
    batch.set(postRef, post);
  });

  await batch.commit();
};
```

---

## Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users collection
    match /users/{userId} {
      // อ่านได้ทุกคน
      allow read: if true;
      // เขียนได้เฉพาะเจ้าของ
      allow write: if request.auth != null && request.auth.uid == userId;
    }

    // Posts collection
    match /posts/{postId} {
      // อ่านได้ถ้า published หรือเป็นเจ้าของ
      allow read: if resource.data.status == 'published'
        || (request.auth != null && request.auth.uid == resource.data.authorId);

      // สร้างได้ถ้า login แล้ว
      allow create: if request.auth != null
        && request.resource.data.authorId == request.auth.uid
        && request.resource.data.title is string
        && request.resource.data.title.size() > 0;

      // แก้ไขได้เฉพาะเจ้าของ
      allow update: if request.auth != null
        && request.auth.uid == resource.data.authorId
        && !request.resource.data.diff(resource.data).affectedKeys()
            .hasAny(['authorId', 'createdAt']);

      // ลบได้เฉพาะเจ้าของหรือ admin
      allow delete: if request.auth != null
        && (request.auth.uid == resource.data.authorId
          || get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin');

      // Comments subcollection
      match /comments/{commentId} {
        allow read: if true;
        allow create: if request.auth != null
          && request.resource.data.authorId == request.auth.uid;
        allow delete: if request.auth != null
          && request.auth.uid == resource.data.authorId;
      }
    }

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    function isAdmin() {
      return isAuthenticated()
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
  }
}
```

---

## Workshop: Real-time Chat

### Chat Types

```typescript
// src/types/chat.ts
import firestore from '@react-native-firebase/firestore';

export interface ChatRoom {
  id?: string;
  name: string;
  type: 'public' | 'private';
  members: string[];
  lastMessage?: string;
  lastMessageAt?: firestore.Timestamp;
  createdAt: firestore.Timestamp;
  createdBy: string;
  unreadCount?: { [userId: string]: number };
}

export interface Message {
  id?: string;
  text: string;
  senderId: string;
  senderName: string;
  senderAvatar?: string;
  createdAt: firestore.Timestamp;
  readBy: string[];
  type: 'text' | 'image' | 'file';
  fileUrl?: string;
  replyTo?: string;
}
```

### Chat Service

```typescript
// src/services/firebase/chatService.ts
import firestore from '@react-native-firebase/firestore';
import auth from '@react-native-firebase/auth';
import { ChatRoom, Message } from '../../types/chat';

export const chatService = {
  // สร้าง room
  createRoom: async (name: string, type: ChatRoom['type']) => {
    const user = auth().currentUser!;
    const now = firestore.Timestamp.now();

    const roomData: Omit<ChatRoom, 'id'> = {
      name,
      type,
      members: [user.uid],
      createdAt: now,
      createdBy: user.uid,
    };

    const docRef = await firestore().collection('rooms').add(roomData);
    return { id: docRef.id, ...roomData };
  },

  // ส่งข้อความ
  sendMessage: async (roomId: string, text: string) => {
    const user = auth().currentUser!;
    const now = firestore.Timestamp.now();

    const batch = firestore().batch();

    // เพิ่มข้อความ
    const messageRef = firestore()
      .collection('rooms')
      .doc(roomId)
      .collection('messages')
      .doc();

    batch.set(messageRef, {
      text,
      senderId: user.uid,
      senderName: user.displayName || 'Anonymous',
      senderAvatar: user.photoURL || null,
      createdAt: now,
      readBy: [user.uid],
      type: 'text',
    });

    // อัปเดต lastMessage ของ room
    const roomRef = firestore().collection('rooms').doc(roomId);
    batch.update(roomRef, {
      lastMessage: text,
      lastMessageAt: now,
    });

    await batch.commit();
    return messageRef.id;
  },

  // Listen to messages
  listenToMessages: (
    roomId: string,
    onMessages: (messages: Message[]) => void
  ) => {
    return firestore()
      .collection('rooms')
      .doc(roomId)
      .collection('messages')
      .orderBy('createdAt', 'asc')
      .limitToLast(50)
      .onSnapshot((snapshot) => {
        const messages = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        })) as Message[];
        onMessages(messages);
      });
  },

  // อ่านข้อความ (mark as read)
  markAsRead: async (roomId: string, messageId: string) => {
    const userId = auth().currentUser?.uid;
    if (!userId) return;

    await firestore()
      .collection('rooms')
      .doc(roomId)
      .collection('messages')
      .doc(messageId)
      .update({
        readBy: firestore.FieldValue.arrayUnion(userId),
      });
  },

  // Listen to rooms ของ user
  listenToRooms: (onRooms: (rooms: ChatRoom[]) => void) => {
    const userId = auth().currentUser?.uid;
    if (!userId) return () => {};

    return firestore()
      .collection('rooms')
      .where('members', 'array-contains', userId)
      .orderBy('lastMessageAt', 'desc')
      .onSnapshot((snapshot) => {
        const rooms = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        })) as ChatRoom[];
        onRooms(rooms);
      });
  },
};
```

### Chat Screen

```typescript
// src/screens/ChatScreen.tsx
import React, { useState, useEffect, useRef } from 'react';
import {
  View,
  FlatList,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  Image,
} from 'react-native';
import auth from '@react-native-firebase/auth';
import { chatService } from '../services/firebase/chatService';
import { Message } from '../types/chat';
import firestore from '@react-native-firebase/firestore';

interface ChatScreenProps {
  route: { params: { roomId: string; roomName: string } };
}

export const ChatScreen: React.FC<ChatScreenProps> = ({ route }) => {
  const { roomId, roomName } = route.params;
  const currentUser = auth().currentUser!;
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputText, setInputText] = useState('');
  const [isSending, setIsSending] = useState(false);
  const flatListRef = useRef<FlatList>(null);

  useEffect(() => {
    const unsubscribe = chatService.listenToMessages(roomId, (msgs) => {
      setMessages(msgs);
      setTimeout(() => {
        flatListRef.current?.scrollToEnd({ animated: true });
      }, 100);
    });

    return unsubscribe;
  }, [roomId]);

  const handleSend = async () => {
    if (!inputText.trim() || isSending) return;

    const text = inputText.trim();
    setInputText('');
    setIsSending(true);

    try {
      await chatService.sendMessage(roomId, text);
    } catch (error) {
      console.error('Send error:', error);
      setInputText(text); // Restore text on error
    } finally {
      setIsSending(false);
    }
  };

  const formatTime = (timestamp: firestore.Timestamp) => {
    const date = timestamp.toDate();
    return date.toLocaleTimeString('th-TH', {
      hour: '2-digit',
      minute: '2-digit',
    });
  };

  const renderMessage = ({ item, index }: { item: Message; index: number }) => {
    const isOwn = item.senderId === currentUser.uid;
    const prevMessage = messages[index - 1];
    const showAvatar = !isOwn &&
      (!prevMessage || prevMessage.senderId !== item.senderId);

    return (
      <View
        style={[
          styles.messageRow,
          isOwn ? styles.ownMessageRow : styles.otherMessageRow,
        ]}
      >
        {/* Avatar */}
        {!isOwn && (
          <View style={styles.avatarContainer}>
            {showAvatar ? (
              <View style={styles.avatar}>
                <Text style={styles.avatarText}>
                  {item.senderName[0]?.toUpperCase()}
                </Text>
              </View>
            ) : (
              <View style={styles.avatarPlaceholder} />
            )}
          </View>
        )}

        {/* Bubble */}
        <View
          style={[
            styles.bubble,
            isOwn ? styles.ownBubble : styles.otherBubble,
          ]}
        >
          {/* Sender name */}
          {!isOwn && showAvatar && (
            <Text style={styles.senderName}>{item.senderName}</Text>
          )}

          {/* Message text */}
          <Text style={[styles.messageText, isOwn && styles.ownMessageText]}>
            {item.text}
          </Text>

          {/* Time and read status */}
          <View style={styles.messageFooter}>
            <Text style={[styles.timeText, isOwn && styles.ownTimeText]}>
              {formatTime(item.createdAt)}
            </Text>
            {isOwn && (
              <Text style={styles.readStatus}>
                {item.readBy.length > 1 ? '✓✓' : '✓'}
              </Text>
            )}
          </View>
        </View>
      </View>
    );
  };

  const renderDateSeparator = (date: string) => (
    <View style={styles.dateSeparator}>
      <View style={styles.dateLine} />
      <Text style={styles.dateText}>{date}</Text>
      <View style={styles.dateLine} />
    </View>
  );

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>{roomName}</Text>
        <Text style={styles.headerSubtitle}>
          {messages.length} ข้อความ
        </Text>
      </View>

      {/* Messages */}
      <FlatList
        ref={flatListRef}
        data={messages}
        keyExtractor={(item) => item.id!}
        renderItem={renderMessage}
        contentContainerStyle={styles.messagesList}
        onContentSizeChange={() =>
          flatListRef.current?.scrollToEnd({ animated: false })
        }
      />

      {/* Input */}
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      >
        <View style={styles.inputContainer}>
          <TextInput
            value={inputText}
            onChangeText={setInputText}
            placeholder="พิมพ์ข้อความ..."
            style={styles.input}
            multiline
            maxLength={1000}
            onSubmitEditing={handleSend}
            blurOnSubmit={false}
          />
          <TouchableOpacity
            onPress={handleSend}
            style={[
              styles.sendButton,
              (!inputText.trim() || isSending) && styles.disabledSend,
            ]}
            disabled={!inputText.trim() || isSending}
          >
            <Text style={styles.sendIcon}>➤</Text>
          </TouchableOpacity>
        </View>
      </KeyboardAvoidingView>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F0F2F5' },
  header: {
    backgroundColor: '#6200EE',
    padding: 16,
    paddingTop: 48,
  },
  headerTitle: { fontSize: 18, fontWeight: 'bold', color: '#fff' },
  headerSubtitle: { fontSize: 12, color: 'rgba(255,255,255,0.7)' },
  messagesList: { padding: 16, paddingBottom: 8 },
  messageRow: {
    flexDirection: 'row',
    marginBottom: 4,
    alignItems: 'flex-end',
  },
  ownMessageRow: { justifyContent: 'flex-end' },
  otherMessageRow: { justifyContent: 'flex-start' },
  avatarContainer: { marginRight: 8, width: 32 },
  avatar: {
    width: 32,
    height: 32,
    borderRadius: 16,
    backgroundColor: '#6200EE',
    alignItems: 'center',
    justifyContent: 'center',
  },
  avatarText: { color: '#fff', fontWeight: 'bold' },
  avatarPlaceholder: { width: 32 },
  bubble: {
    maxWidth: '75%',
    padding: 10,
    borderRadius: 16,
  },
  ownBubble: {
    backgroundColor: '#6200EE',
    borderBottomRightRadius: 4,
  },
  otherBubble: {
    backgroundColor: '#fff',
    borderBottomLeftRadius: 4,
    elevation: 1,
  },
  senderName: {
    fontSize: 11,
    fontWeight: 'bold',
    color: '#6200EE',
    marginBottom: 2,
  },
  messageText: { fontSize: 15, color: '#333', lineHeight: 20 },
  ownMessageText: { color: '#fff' },
  messageFooter: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'flex-end',
    marginTop: 4,
    gap: 4,
  },
  timeText: { fontSize: 10, color: '#999' },
  ownTimeText: { color: 'rgba(255,255,255,0.7)' },
  readStatus: { fontSize: 10, color: 'rgba(255,255,255,0.7)' },
  dateSeparator: {
    flexDirection: 'row',
    alignItems: 'center',
    marginVertical: 12,
    gap: 8,
  },
  dateLine: { flex: 1, height: 1, backgroundColor: '#E0E0E0' },
  dateText: { fontSize: 11, color: '#999' },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    padding: 12,
    backgroundColor: '#fff',
    borderTopWidth: 1,
    borderTopColor: '#E0E0E0',
    gap: 8,
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    maxHeight: 100,
    fontSize: 15,
    backgroundColor: '#F9F9F9',
  },
  sendButton: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: '#6200EE',
    alignItems: 'center',
    justifyContent: 'center',
  },
  disabledSend: { backgroundColor: '#E0E0E0' },
  sendIcon: { color: '#fff', fontSize: 18 },
});
```

---

## Performance Tips

### 1. ใช้ Pagination เสมอ

```typescript
// ❌ ไม่ดี - โหลดทั้งหมด
const snapshot = await firestore().collection('posts').get();

// ✅ ดีกว่า - โหลดทีละหน้า
const PAGE_SIZE = 10;
const snapshot = await firestore()
  .collection('posts')
  .orderBy('createdAt', 'desc')
  .limit(PAGE_SIZE)
  .get();
```

### 2. Index สำหรับ Complex Queries

```json
// firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "posts",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

### 3. ใช้ Select เฉพาะ fields ที่ต้องการ

```typescript
// โหลดเฉพาะ field ที่ต้องการ
const snapshot = await firestore()
  .collection('posts')
  .select('title', 'createdAt', 'authorName')
  .get();
```

---

## สรุป

Firestore Database:

1. **CRUD** - Create, Read, Update, Delete documents
2. **Real-time Listeners** - `onSnapshot` สำหรับ live updates
3. **Queries** - where, orderBy, limit, pagination
4. **Subcollections** - nested data structure
5. **Transactions/Batch** - atomic operations
6. **Security Rules** - ควบคุมการเข้าถึง

ใน Part 040 จะเรียนเรื่อง **Firebase Storage** สำหรับ upload รูปภาพและไฟล์
