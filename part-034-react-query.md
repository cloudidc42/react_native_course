# Part 034: React Query / TanStack Query

## ทำไม React Query ดีกว่า useState?

เมื่อ fetch ข้อมูลจาก API ด้วย useState ปกติ:

```typescript
// ❌ แบบที่ต้องเขียน manual ด้วย useState
const [data, setData] = useState(null);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  let cancelled = false;
  setIsLoading(true);

  fetch('/api/posts')
    .then(res => res.json())
    .then(data => {
      if (!cancelled) {
        setData(data);
        setIsLoading(false);
      }
    })
    .catch(err => {
      if (!cancelled) {
        setError(err);
        setIsLoading(false);
      }
    });

  return () => { cancelled = true; };
}, []);
```

ปัญหาของวิธีนี้:
- ต้องเขียนโค้ดซ้ำในทุก component
- ไม่มี caching
- ไม่มี background refetch
- ต้อง handle race conditions เอง
- ไม่มี deduplication

### React Query แก้ทุกปัญหา

```typescript
// ✅ แบบที่ React Query ทำให้ง่ายขึ้น
const { data, isLoading, error } = useQuery({
  queryKey: ['posts'],
  queryFn: () => fetch('/api/posts').then(res => res.json()),
});
```

React Query จัดการให้อัตโนมัติ:
- Caching
- Background refetch
- Loading/error states
- Stale data handling
- Request deduplication
- Window focus refetch
- Optimistic updates

---

## ติดตั้ง TanStack Query

```bash
npm install @tanstack/react-query

# ถ้าต้องการ DevTools
npm install @tanstack/react-query-devtools
```

### Setup QueryClient

```typescript
// App.tsx
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 นาที
      gcTime: 1000 * 60 * 10,   // 10 นาที (garbage collection time)
      retry: 2,                   // retry 2 ครั้ง
      refetchOnWindowFocus: false, // ปิดสำหรับ mobile
    },
    mutations: {
      retry: 0,
    },
  },
});

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AppNavigator />
    </QueryClientProvider>
  );
}
```

---

## useQuery - ดึงข้อมูล

### พื้นฐาน

```typescript
import { useQuery } from '@tanstack/react-query';

// API function
const fetchUser = async (userId: number) => {
  const response = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`);
  if (!response.ok) throw new Error('Network response was not ok');
  return response.json();
};

// ใน Component
function UserProfile({ userId }: { userId: number }) {
  const {
    data: user,           // ข้อมูล
    isLoading,            // กำลังโหลดครั้งแรก
    isFetching,           // กำลัง fetch (รวม background fetch)
    isError,              // มี error
    error,                // error object
    isSuccess,            // สำเร็จ
    refetch,              // function สำหรับ refetch manually
    status,               // 'pending' | 'success' | 'error'
    fetchStatus,          // 'fetching' | 'paused' | 'idle'
    dataUpdatedAt,        // เวลาที่ข้อมูลอัปเดต
  } = useQuery({
    queryKey: ['user', userId],  // unique key สำหรับ cache
    queryFn: () => fetchUser(userId),
    enabled: userId > 0,         // เงื่อนไขในการ fetch
    staleTime: 1000 * 60,       // ข้อมูลไม่เก่าใน 1 นาที
    gcTime: 1000 * 60 * 5,     // เก็บ cache 5 นาที
    refetchInterval: 1000 * 30, // refetch ทุก 30 วินาที
    retry: 3,                   // retry 3 ครั้งถ้าล้มเหลว
    retryDelay: 1000,           // รอ 1 วินาทีก่อน retry
  });

  if (isLoading) return <ActivityIndicator />;
  if (isError) return <Text>Error: {error.message}</Text>;

  return <Text>{user?.name}</Text>;
}
```

### Query Keys

Query keys เป็น array ที่ใช้ระบุ cache entry

```typescript
// Simple key
useQuery({ queryKey: ['posts'], queryFn: fetchPosts });

// Key with ID
useQuery({ queryKey: ['post', postId], queryFn: () => fetchPost(postId) });

// Key with filters
useQuery({
  queryKey: ['posts', { status: 'published', author: userId }],
  queryFn: () => fetchPosts({ status: 'published', author: userId }),
});

// Dependent queries
const { data: user } = useQuery({ queryKey: ['user', userId], queryFn: ... });
const { data: posts } = useQuery({
  queryKey: ['posts', user?.id],
  queryFn: () => fetchUserPosts(user!.id),
  enabled: !!user?.id,  // เปิดใช้เฉพาะเมื่อมี user
});
```

---

## useMutation - เปลี่ยนข้อมูล

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface CreatePostData {
  title: string;
  body: string;
}

interface Post {
  id: number;
  title: string;
  body: string;
}

const createPost = async (data: CreatePostData): Promise<Post> => {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create post');
  return response.json();
};

function CreatePostForm() {
  const queryClient = useQueryClient();
  const [title, setTitle] = useState('');
  const [body, setBody] = useState('');

  const mutation = useMutation({
    mutationFn: createPost,
    onSuccess: (newPost) => {
      // Invalidate posts list cache
      queryClient.invalidateQueries({ queryKey: ['posts'] });
      Alert.alert('สำเร็จ', 'สร้าง post แล้ว');
    },
    onError: (error) => {
      Alert.alert('ผิดพลาด', error.message);
    },
    onSettled: () => {
      // ทำงานทั้ง success และ error
      console.log('Mutation settled');
    },
  });

  return (
    <View>
      <TextInput value={title} onChangeText={setTitle} placeholder="หัวข้อ" />
      <TextInput value={body} onChangeText={setBody} placeholder="เนื้อหา" />
      <TouchableOpacity
        onPress={() => mutation.mutate({ title, body })}
        disabled={mutation.isPending}
      >
        {mutation.isPending ? (
          <ActivityIndicator />
        ) : (
          <Text>สร้าง Post</Text>
        )}
      </TouchableOpacity>
    </View>
  );
}
```

---

## Query Invalidation

การ invalidate ทำให้ query ถือว่า stale และ refetch

```typescript
const queryClient = useQueryClient();

// Invalidate queries แบบต่าง ๆ
queryClient.invalidateQueries({ queryKey: ['posts'] });           // ทุก posts queries
queryClient.invalidateQueries({ queryKey: ['posts', postId] });  // เฉพาะ post นี้
queryClient.invalidateQueries({ queryKey: ['user'] });            // ทุก user queries

// Refetch ทันที (ไม่รอ stale)
queryClient.refetchQueries({ queryKey: ['posts'] });

// ลบ cache
queryClient.removeQueries({ queryKey: ['posts'] });

// อัปเดต cache โดยตรง (optimistic update)
queryClient.setQueryData(['posts'], (old: Post[]) => [...old, newPost]);

// ดึงข้อมูลจาก cache
const cachedPosts = queryClient.getQueryData<Post[]>(['posts']);
```

---

## Caching Strategies

### staleTime และ gcTime

```typescript
// staleTime = เวลาที่ข้อมูลถือว่า "fresh" ไม่ต้อง refetch
// gcTime = เวลาที่ cache ถูกลบออกจาก memory

// Aggressive caching - ข้อมูลไม่ค่อยเปลี่ยน
useQuery({
  queryKey: ['categories'],
  queryFn: fetchCategories,
  staleTime: Infinity,  // ไม่เก่าเลย
  gcTime: Infinity,      // เก็บตลอดไป
});

// Default caching - ข้อมูลปกติ
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 1000 * 60 * 5, // fresh 5 นาที
  gcTime: 1000 * 60 * 10,   // เก็บ 10 นาที
});

// Real-time data - ข้อมูลเปลี่ยนบ่อย
useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  staleTime: 0,               // เก่าทันที
  refetchInterval: 1000 * 30, // refetch ทุก 30 วินาที
});
```

---

## Workshop: Blog App ด้วย React Query

### API Service Layer

```typescript
// src/services/api.ts
const BASE_URL = 'https://jsonplaceholder.typicode.com';

export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

export interface Comment {
  id: number;
  postId: number;
  name: string;
  email: string;
  body: string;
}

export interface User {
  id: number;
  name: string;
  email: string;
  username: string;
}

// Posts
export const postsApi = {
  getAll: async (): Promise<Post[]> => {
    const res = await fetch(`${BASE_URL}/posts`);
    if (!res.ok) throw new Error('Failed to fetch posts');
    return res.json();
  },

  getById: async (id: number): Promise<Post> => {
    const res = await fetch(`${BASE_URL}/posts/${id}`);
    if (!res.ok) throw new Error('Post not found');
    return res.json();
  },

  getByUser: async (userId: number): Promise<Post[]> => {
    const res = await fetch(`${BASE_URL}/posts?userId=${userId}`);
    if (!res.ok) throw new Error('Failed to fetch user posts');
    return res.json();
  },

  create: async (data: Omit<Post, 'id'>): Promise<Post> => {
    const res = await fetch(`${BASE_URL}/posts`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error('Failed to create post');
    return res.json();
  },

  update: async ({ id, ...data }: Post): Promise<Post> => {
    const res = await fetch(`${BASE_URL}/posts/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error('Failed to update post');
    return res.json();
  },

  delete: async (id: number): Promise<void> => {
    const res = await fetch(`${BASE_URL}/posts/${id}`, { method: 'DELETE' });
    if (!res.ok) throw new Error('Failed to delete post');
  },
};

// Comments
export const commentsApi = {
  getByPost: async (postId: number): Promise<Comment[]> => {
    const res = await fetch(`${BASE_URL}/posts/${postId}/comments`);
    if (!res.ok) throw new Error('Failed to fetch comments');
    return res.json();
  },
};

// Users
export const usersApi = {
  getAll: async (): Promise<User[]> => {
    const res = await fetch(`${BASE_URL}/users`);
    if (!res.ok) throw new Error('Failed to fetch users');
    return res.json();
  },

  getById: async (id: number): Promise<User> => {
    const res = await fetch(`${BASE_URL}/users/${id}`);
    if (!res.ok) throw new Error('User not found');
    return res.json();
  },
};
```

### Custom Hooks

```typescript
// src/hooks/usePosts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { postsApi, Post } from '../services/api';

// Query Keys constants
export const QUERY_KEYS = {
  posts: ['posts'] as const,
  post: (id: number) => ['posts', id] as const,
  userPosts: (userId: number) => ['posts', 'user', userId] as const,
  comments: (postId: number) => ['comments', postId] as const,
  users: ['users'] as const,
  user: (id: number) => ['users', id] as const,
};

// Hooks สำหรับ Posts
export function usePosts() {
  return useQuery({
    queryKey: QUERY_KEYS.posts,
    queryFn: postsApi.getAll,
    select: (data) => data.slice(0, 20), // เอาแค่ 20 รายการ
  });
}

export function usePost(id: number) {
  return useQuery({
    queryKey: QUERY_KEYS.post(id),
    queryFn: () => postsApi.getById(id),
    enabled: id > 0,
  });
}

export function useUserPosts(userId: number) {
  return useQuery({
    queryKey: QUERY_KEYS.userPosts(userId),
    queryFn: () => postsApi.getByUser(userId),
    enabled: userId > 0,
  });
}

export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.create,
    onSuccess: (newPost) => {
      // Optimistic update
      queryClient.setQueryData<Post[]>(QUERY_KEYS.posts, (old = []) => [
        newPost,
        ...old,
      ]);
    },
    onError: () => {
      queryClient.invalidateQueries({ queryKey: QUERY_KEYS.posts });
    },
  });
}

export function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.update,
    onMutate: async (updatedPost) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: QUERY_KEYS.post(updatedPost.id) });

      // Snapshot previous value
      const previousPost = queryClient.getQueryData<Post>(
        QUERY_KEYS.post(updatedPost.id)
      );

      // Optimistically update
      queryClient.setQueryData(QUERY_KEYS.post(updatedPost.id), updatedPost);

      return { previousPost };
    },
    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousPost) {
        queryClient.setQueryData(
          QUERY_KEYS.post(variables.id),
          context.previousPost
        );
      }
    },
    onSettled: (data, error, variables) => {
      queryClient.invalidateQueries({ queryKey: QUERY_KEYS.post(variables.id) });
    },
  });
}

export function useDeletePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.delete,
    onSuccess: (_, deletedId) => {
      queryClient.setQueryData<Post[]>(QUERY_KEYS.posts, (old = []) =>
        old.filter((post) => post.id !== deletedId)
      );
      queryClient.removeQueries({ queryKey: QUERY_KEYS.post(deletedId) });
    },
  });
}
```

### Posts List Screen

```typescript
// src/screens/PostsListScreen.tsx
import React, { useState } from 'react';
import {
  View,
  FlatList,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  RefreshControl,
  TextInput,
  Alert,
} from 'react-native';
import { usePosts, useCreatePost, useDeletePost } from '../hooks/usePosts';
import { Post } from '../services/api';

interface PostItemProps {
  post: Post;
  onPress: () => void;
  onDelete: () => void;
}

const PostItem: React.FC<PostItemProps> = ({ post, onPress, onDelete }) => (
  <TouchableOpacity onPress={onPress} style={styles.postItem}>
    <Text style={styles.postTitle} numberOfLines={2}>{post.title}</Text>
    <Text style={styles.postBody} numberOfLines={3}>{post.body}</Text>
    <View style={styles.postFooter}>
      <Text style={styles.postId}>#{post.id}</Text>
      <TouchableOpacity onPress={onDelete} style={styles.deleteBtn}>
        <Text style={styles.deleteBtnText}>ลบ</Text>
      </TouchableOpacity>
    </View>
  </TouchableOpacity>
);

export const PostsListScreen: React.FC = ({ navigation }: any) => {
  const [newTitle, setNewTitle] = useState('');
  const [showForm, setShowForm] = useState(false);

  const { data: posts, isLoading, isError, error, refetch, isFetching } =
    usePosts();
  const createMutation = useCreatePost();
  const deleteMutation = useDeletePost();

  const handleCreate = () => {
    if (!newTitle.trim()) return;
    createMutation.mutate(
      { title: newTitle, body: 'Post body', userId: 1 },
      {
        onSuccess: () => {
          setNewTitle('');
          setShowForm(false);
        },
      }
    );
  };

  const handleDelete = (postId: number) => {
    Alert.alert('ยืนยัน', 'ต้องการลบ post นี้?', [
      { text: 'ยกเลิก', style: 'cancel' },
      {
        text: 'ลบ',
        style: 'destructive',
        onPress: () => deleteMutation.mutate(postId),
      },
    ]);
  };

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
        <Text style={styles.loadingText}>กำลังโหลด posts...</Text>
      </View>
    );
  }

  if (isError) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>❌ {error?.message}</Text>
        <TouchableOpacity onPress={() => refetch()} style={styles.retryBtn}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>📝 Blog Posts</Text>
        <TouchableOpacity
          onPress={() => setShowForm(!showForm)}
          style={styles.addBtn}
        >
          <Text style={styles.addBtnText}>{showForm ? '✕' : '+'}</Text>
        </TouchableOpacity>
      </View>

      {/* Create Form */}
      {showForm && (
        <View style={styles.createForm}>
          <TextInput
            value={newTitle}
            onChangeText={setNewTitle}
            placeholder="หัวข้อ post ใหม่"
            style={styles.input}
          />
          <TouchableOpacity
            onPress={handleCreate}
            style={styles.submitBtn}
            disabled={createMutation.isPending}
          >
            {createMutation.isPending ? (
              <ActivityIndicator color="#fff" size="small" />
            ) : (
              <Text style={styles.submitBtnText}>สร้าง</Text>
            )}
          </TouchableOpacity>
        </View>
      )}

      {/* Posts List */}
      <FlatList
        data={posts}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => (
          <PostItem
            post={item}
            onPress={() => navigation.navigate('PostDetail', { postId: item.id })}
            onDelete={() => handleDelete(item.id)}
          />
        )}
        refreshControl={
          <RefreshControl
            refreshing={isFetching && !isLoading}
            onRefresh={refetch}
            colors={['#6200EE']}
          />
        }
        contentContainerStyle={styles.listContent}
        ListEmptyComponent={
          <View style={styles.center}>
            <Text style={styles.emptyText}>ยังไม่มี posts</Text>
          </View>
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 20 },
  loadingText: { marginTop: 12, color: '#666' },
  errorText: { fontSize: 16, color: '#F44336', marginBottom: 12, textAlign: 'center' },
  retryBtn: { backgroundColor: '#6200EE', padding: 12, borderRadius: 8 },
  retryText: { color: '#fff', fontWeight: 'bold' },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    backgroundColor: '#6200EE',
    padding: 16,
  },
  headerTitle: { fontSize: 20, fontWeight: 'bold', color: '#fff' },
  addBtn: {
    width: 36,
    height: 36,
    borderRadius: 18,
    backgroundColor: 'rgba(255,255,255,0.3)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  addBtnText: { color: '#fff', fontSize: 20, fontWeight: 'bold' },
  createForm: {
    backgroundColor: '#fff',
    padding: 16,
    flexDirection: 'row',
    gap: 8,
    elevation: 2,
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 8,
    padding: 10,
    fontSize: 14,
  },
  submitBtn: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 16,
    borderRadius: 8,
    justifyContent: 'center',
    minWidth: 60,
    alignItems: 'center',
  },
  submitBtnText: { color: '#fff', fontWeight: 'bold' },
  listContent: { padding: 16 },
  postItem: {
    backgroundColor: '#fff',
    padding: 16,
    marginBottom: 12,
    borderRadius: 8,
    elevation: 1,
  },
  postTitle: { fontSize: 16, fontWeight: 'bold', marginBottom: 6 },
  postBody: { fontSize: 14, color: '#666', lineHeight: 20, marginBottom: 8 },
  postFooter: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },
  postId: { fontSize: 12, color: '#999' },
  deleteBtn: { backgroundColor: '#FF5252', paddingHorizontal: 12, paddingVertical: 4, borderRadius: 4 },
  deleteBtnText: { color: '#fff', fontSize: 12 },
  emptyText: { color: '#999', fontSize: 16 },
});
```

### Post Detail Screen (Parallel Queries)

```typescript
// src/screens/PostDetailScreen.tsx
import React from 'react';
import {
  View,
  Text,
  ScrollView,
  StyleSheet,
  ActivityIndicator,
} from 'react-native';
import { useQuery, useQueries } from '@tanstack/react-query';
import { postsApi, commentsApi, usersApi } from '../services/api';
import { QUERY_KEYS } from '../hooks/usePosts';

export const PostDetailScreen: React.FC = ({ route }: any) => {
  const { postId } = route.params;

  // Parallel queries - fetch ข้อมูลพร้อมกัน
  const results = useQueries({
    queries: [
      {
        queryKey: QUERY_KEYS.post(postId),
        queryFn: () => postsApi.getById(postId),
      },
      {
        queryKey: QUERY_KEYS.comments(postId),
        queryFn: () => commentsApi.getByPost(postId),
      },
    ],
  });

  const [postQuery, commentsQuery] = results;

  // Dependent query - fetch user หลังจากได้ post
  const userQuery = useQuery({
    queryKey: QUERY_KEYS.user(postQuery.data?.userId || 0),
    queryFn: () => usersApi.getById(postQuery.data!.userId),
    enabled: !!postQuery.data?.userId,
  });

  const isLoading =
    postQuery.isLoading || commentsQuery.isLoading || userQuery.isLoading;

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  const { data: post } = postQuery;
  const { data: comments } = commentsQuery;
  const { data: author } = userQuery;

  return (
    <ScrollView style={styles.container} contentContainerStyle={styles.content}>
      {/* Post */}
      <View style={styles.postCard}>
        <Text style={styles.title}>{post?.title}</Text>
        {author && (
          <View style={styles.authorRow}>
            <Text style={styles.authorName}>✍️ {author.name}</Text>
            <Text style={styles.authorEmail}>{author.email}</Text>
          </View>
        )}
        <Text style={styles.body}>{post?.body}</Text>
      </View>

      {/* Comments */}
      <View style={styles.commentsSection}>
        <Text style={styles.commentsTitle}>
          💬 ความคิดเห็น ({comments?.length || 0})
        </Text>
        {comments?.map((comment) => (
          <View key={comment.id} style={styles.commentCard}>
            <Text style={styles.commentName}>{comment.name}</Text>
            <Text style={styles.commentEmail}>{comment.email}</Text>
            <Text style={styles.commentBody}>{comment.body}</Text>
          </View>
        ))}
      </View>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  content: { padding: 16 },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  postCard: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    marginBottom: 16,
    elevation: 2,
  },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  authorRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 12, gap: 8 },
  authorName: { fontSize: 14, fontWeight: 'bold', color: '#6200EE' },
  authorEmail: { fontSize: 12, color: '#999' },
  body: { fontSize: 15, lineHeight: 24, color: '#333' },
  commentsSection: { marginBottom: 20 },
  commentsTitle: { fontSize: 18, fontWeight: 'bold', marginBottom: 12 },
  commentCard: {
    backgroundColor: '#fff',
    borderRadius: 8,
    padding: 12,
    marginBottom: 8,
    elevation: 1,
  },
  commentName: { fontSize: 13, fontWeight: 'bold', marginBottom: 2 },
  commentEmail: { fontSize: 11, color: '#6200EE', marginBottom: 6 },
  commentBody: { fontSize: 13, color: '#666', lineHeight: 18 },
});
```

---

## Advanced Features

### Infinite Queries (Pagination)

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

const fetchPagedPosts = async ({ pageParam = 1 }: { pageParam?: number }) => {
  const res = await fetch(
    `https://api.example.com/posts?page=${pageParam}&limit=10`
  );
  const data = await res.json();
  return {
    posts: data.posts,
    nextPage: data.hasMore ? pageParam + 1 : undefined,
  };
};

function InfinitePostsList() {
  const {
    data,
    isLoading,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: fetchPagedPosts,
    getNextPageParam: (lastPage) => lastPage.nextPage,
    initialPageParam: 1,
  });

  // รวม pages ทั้งหมด
  const allPosts = data?.pages.flatMap((page) => page.posts) ?? [];

  return (
    <FlatList
      data={allPosts}
      keyExtractor={(item) => item.id.toString()}
      renderItem={({ item }) => <PostItem post={item} />}
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={() =>
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  );
}
```

### Prefetching

```typescript
const queryClient = useQueryClient();

// Prefetch ข้อมูลก่อนที่จะต้องใช้
const handlePostHover = (postId: number) => {
  queryClient.prefetchQuery({
    queryKey: QUERY_KEYS.post(postId),
    queryFn: () => postsApi.getById(postId),
    staleTime: 1000 * 60, // ไม่ prefetch ถ้าข้อมูลยังไม่เก่า
  });
};
```

---

## Tips and Best Practices

### 1. จัดระเบียบ Query Keys

```typescript
export const queryKeys = {
  all: ['root'] as const,
  posts: {
    all: () => [...queryKeys.all, 'posts'] as const,
    lists: () => [...queryKeys.posts.all(), 'list'] as const,
    list: (filters: object) => [...queryKeys.posts.lists(), filters] as const,
    details: () => [...queryKeys.posts.all(), 'detail'] as const,
    detail: (id: number) => [...queryKeys.posts.details(), id] as const,
  },
};

// ใช้งาน
useQuery({ queryKey: queryKeys.posts.list({ status: 'active' }), ... });

// Invalidate ทุก posts queries
queryClient.invalidateQueries({ queryKey: queryKeys.posts.all() });
```

### 2. Error Boundary

```typescript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ error, resetErrorBoundary }) => (
            <View>
              <Text>{error.message}</Text>
              <Button onPress={resetErrorBoundary} title="ลองใหม่" />
            </View>
          )}
        >
          <PostsList />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

---

## สรุป

React Query / TanStack Query:

1. **useQuery** - ดึงข้อมูลพร้อม caching อัตโนมัติ
2. **useMutation** - เปลี่ยนข้อมูล + invalidate cache
3. **useInfiniteQuery** - pagination แบบ infinite scroll
4. **Query Keys** - ระบุ cache uniquely
5. **Optimistic Updates** - update UI ก่อน server response
6. **Prefetching** - โหลดข้อมูลล่วงหน้า

ใน Part 035 จะเรียนเรื่อง **Axios** สำหรับ HTTP requests ที่ advanced ขึ้น
