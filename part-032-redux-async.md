# Part 032: Redux Thunk และ Async Actions

## ทำไมต้องใช้ Async Actions?

Redux ปกติทำงานแบบ synchronous แต่ใน real-world app เราต้องการ:
- Fetch ข้อมูลจาก API
- บันทึกข้อมูลลง server
- จัดการ loading/error states
- Cancel requests เมื่อ component unmount

### Redux Thunk คืออะไร?

Redux Thunk เป็น middleware ที่อนุญาตให้ dispatch **functions** แทน plain objects

```
// ปกติ: dispatch action object
dispatch({ type: 'increment' })

// กับ Thunk: dispatch function ที่รับ dispatch และ getState
dispatch((dispatch, getState) => {
  // async logic ที่นี่
  fetch('/api/data').then(data => dispatch(setData(data)));
});
```

RTK มี `createAsyncThunk` ที่ทำให้การสร้าง async thunks ง่ายขึ้นมาก

---

## createAsyncThunk

### โครงสร้างพื้นฐาน

```typescript
import { createAsyncThunk } from '@reduxjs/toolkit';

const fetchTodos = createAsyncThunk(
  'todos/fetchAll',          // action type prefix
  async (arg, thunkAPI) => { // payload creator
    const response = await fetch('/api/todos');
    if (!response.ok) {
      return thunkAPI.rejectWithValue('Failed to fetch');
    }
    return response.json();
  }
);
```

`createAsyncThunk` สร้าง 3 action types อัตโนมัติ:
- `todos/fetchAll/pending` - กำลัง fetch
- `todos/fetchAll/fulfilled` - fetch สำเร็จ
- `todos/fetchAll/rejected` - fetch ล้มเหลว

---

## extraReducers

ใช้ `extraReducers` เพื่อจัดการ async action states

```typescript
// src/store/slices/postsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

// Types
export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

interface PostsState {
  items: Post[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  selectedPost: Post | null;
}

// Async Thunks
export const fetchPosts = createAsyncThunk<Post[], void>(
  'posts/fetchAll',
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch(
        'https://jsonplaceholder.typicode.com/posts'
      );
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return response.json();
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }
);

export const fetchPostById = createAsyncThunk<Post, number>(
  'posts/fetchById',
  async (postId, { rejectWithValue }) => {
    try {
      const response = await fetch(
        `https://jsonplaceholder.typicode.com/posts/${postId}`
      );
      if (!response.ok) {
        throw new Error('Post not found');
      }
      return response.json();
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }
);

export const createPost = createAsyncThunk<
  Post,
  { title: string; body: string; userId: number }
>(
  'posts/create',
  async (postData, { rejectWithValue }) => {
    try {
      const response = await fetch(
        'https://jsonplaceholder.typicode.com/posts',
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(postData),
        }
      );
      if (!response.ok) {
        throw new Error('Failed to create post');
      }
      return response.json();
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }
);

export const updatePost = createAsyncThunk<
  Post,
  { id: number; title: string; body: string }
>(
  'posts/update',
  async ({ id, ...data }, { rejectWithValue }) => {
    try {
      const response = await fetch(
        `https://jsonplaceholder.typicode.com/posts/${id}`,
        {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        }
      );
      if (!response.ok) {
        throw new Error('Failed to update post');
      }
      return response.json();
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }
);

export const deletePost = createAsyncThunk<number, number>(
  'posts/delete',
  async (postId, { rejectWithValue }) => {
    try {
      const response = await fetch(
        `https://jsonplaceholder.typicode.com/posts/${postId}`,
        { method: 'DELETE' }
      );
      if (!response.ok) {
        throw new Error('Failed to delete post');
      }
      return postId;
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  }
);

// Initial State
const initialState: PostsState = {
  items: [],
  status: 'idle',
  error: null,
  selectedPost: null,
};

// Slice
const postsSlice = createSlice({
  name: 'posts',
  initialState,
  reducers: {
    selectPost: (state, action: PayloadAction<Post | null>) => {
      state.selectedPost = action.payload;
    },
    clearError: (state) => {
      state.error = null;
    },
  },
  // จัดการ async thunk actions
  extraReducers: (builder) => {
    // Fetch All Posts
    builder
      .addCase(fetchPosts.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchPosts.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload as string;
      });

    // Fetch Post By Id
    builder
      .addCase(fetchPostById.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchPostById.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.selectedPost = action.payload;
      })
      .addCase(fetchPostById.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload as string;
      });

    // Create Post
    builder
      .addCase(createPost.fulfilled, (state, action) => {
        state.items.unshift(action.payload);
      })
      .addCase(createPost.rejected, (state, action) => {
        state.error = action.payload as string;
      });

    // Update Post
    builder.addCase(updatePost.fulfilled, (state, action) => {
      const index = state.items.findIndex(
        (post) => post.id === action.payload.id
      );
      if (index !== -1) {
        state.items[index] = action.payload;
      }
    });

    // Delete Post
    builder.addCase(deletePost.fulfilled, (state, action) => {
      state.items = state.items.filter((post) => post.id !== action.payload);
    });
  },
});

export const { selectPost, clearError } = postsSlice.actions;
export default postsSlice.reducer;

// Selectors
export const selectAllPosts = (state: { posts: PostsState }) =>
  state.posts.items;
export const selectPostsStatus = (state: { posts: PostsState }) =>
  state.posts.status;
export const selectPostsError = (state: { posts: PostsState }) =>
  state.posts.error;
export const selectSelectedPost = (state: { posts: PostsState }) =>
  state.posts.selectedPost;
```

---

## Loading/Error States Pattern

### รูปแบบมาตรฐาน

```typescript
interface AsyncState<T> {
  data: T | null;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

// Helper function
function createAsyncInitialState<T>(data: T | null = null): AsyncState<T> {
  return {
    data,
    status: 'idle',
    error: null,
  };
}
```

### Generic Loading Component

```typescript
// src/components/LoadingState.tsx
import React from 'react';
import { View, Text, ActivityIndicator, StyleSheet, TouchableOpacity } from 'react-native';

interface LoadingStateProps {
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error?: string | null;
  onRetry?: () => void;
  children: React.ReactNode;
}

export const LoadingState: React.FC<LoadingStateProps> = ({
  status,
  error,
  onRetry,
  children,
}) => {
  if (status === 'loading') {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
        <Text style={styles.loadingText}>กำลังโหลด...</Text>
      </View>
    );
  }

  if (status === 'failed') {
    return (
      <View style={styles.center}>
        <Text style={styles.errorIcon}>❌</Text>
        <Text style={styles.errorText}>{error || 'เกิดข้อผิดพลาด'}</Text>
        {onRetry && (
          <TouchableOpacity onPress={onRetry} style={styles.retryButton}>
            <Text style={styles.retryText}>ลองใหม่</Text>
          </TouchableOpacity>
        )}
      </View>
    );
  }

  return <>{children}</>;
};

const styles = StyleSheet.create({
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 20,
  },
  loadingText: {
    marginTop: 12,
    fontSize: 16,
    color: '#666',
  },
  errorIcon: {
    fontSize: 48,
    marginBottom: 12,
  },
  errorText: {
    fontSize: 16,
    color: '#F44336',
    textAlign: 'center',
    marginBottom: 16,
  },
  retryButton: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  retryText: {
    color: '#fff',
    fontWeight: 'bold',
  },
});
```

---

## RTK Query เบื้องต้น

RTK Query เป็น data fetching tool ที่ทรงพลัง built-in อยู่ใน Redux Toolkit

### ทำไม RTK Query ดีกว่า createAsyncThunk?

| Feature | createAsyncThunk | RTK Query |
|---------|-----------------|-----------|
| Cache | Manual | Automatic |
| Refetch | Manual | Automatic |
| Optimistic updates | Manual | Built-in |
| Polling | Manual | Built-in |
| Code amount | Many lines | Few lines |

### สร้าง API Service ด้วย RTK Query

```typescript
// src/store/services/postsApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

export interface CreatePostRequest {
  title: string;
  body: string;
  userId: number;
}

export const postsApi = createApi({
  reducerPath: 'postsApi',
  baseQuery: fetchBaseQuery({
    baseUrl: 'https://jsonplaceholder.typicode.com',
    // เพิ่ม headers
    prepareHeaders: (headers, { getState }) => {
      // ดึง token จาก store
      const token = (getState() as any).auth?.token;
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    },
  }),
  // Tag types สำหรับ cache invalidation
  tagTypes: ['Post'],
  endpoints: (builder) => ({
    // GET /posts
    getPosts: builder.query<Post[], void>({
      query: () => '/posts',
      providesTags: ['Post'],
    }),

    // GET /posts/:id
    getPostById: builder.query<Post, number>({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),

    // POST /posts
    createPost: builder.mutation<Post, CreatePostRequest>({
      query: (post) => ({
        url: '/posts',
        method: 'POST',
        body: post,
      }),
      invalidatesTags: ['Post'],
    }),

    // PUT /posts/:id
    updatePost: builder.mutation<Post, { id: number } & Partial<Post>>({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: 'PUT',
        body: patch,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'Post', id }],
    }),

    // DELETE /posts/:id
    deletePost: builder.mutation<void, number>({
      query: (id) => ({
        url: `/posts/${id}`,
        method: 'DELETE',
      }),
      invalidatesTags: ['Post'],
    }),
  }),
});

// Export hooks
export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

### เพิ่ม RTK Query ลง Store

```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { postsApi } from './services/postsApi';

export const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
    // ... other reducers
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),
});
```

### ใช้งาน RTK Query ใน Component

```typescript
// src/screens/PostsScreen.tsx
import React, { useState } from 'react';
import {
  View,
  FlatList,
  Text,
  TouchableOpacity,
  TextInput,
  StyleSheet,
  ActivityIndicator,
  Alert,
} from 'react-native';
import {
  useGetPostsQuery,
  useCreatePostMutation,
  useDeletePostMutation,
  Post,
} from '../store/services/postsApi';

export const PostsScreen: React.FC = () => {
  const [newTitle, setNewTitle] = useState('');
  const [newBody, setNewBody] = useState('');

  // Query
  const {
    data: posts,
    isLoading,
    isError,
    error,
    refetch,
    isFetching,
  } = useGetPostsQuery();

  // Mutations
  const [createPost, { isLoading: isCreating }] = useCreatePostMutation();
  const [deletePost] = useDeletePostMutation();

  const handleCreate = async () => {
    if (!newTitle.trim()) return;

    try {
      await createPost({
        title: newTitle,
        body: newBody,
        userId: 1,
      }).unwrap(); // .unwrap() throws error if rejected

      setNewTitle('');
      setNewBody('');
    } catch (err) {
      Alert.alert('Error', 'ไม่สามารถสร้าง post ได้');
    }
  };

  const handleDelete = async (id: number) => {
    try {
      await deletePost(id).unwrap();
    } catch (err) {
      Alert.alert('Error', 'ไม่สามารถลบ post ได้');
    }
  };

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  if (isError) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>เกิดข้อผิดพลาด</Text>
        <TouchableOpacity onPress={refetch} style={styles.retryButton}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  const renderPost = ({ item }: { item: Post }) => (
    <View style={styles.postCard}>
      <Text style={styles.postTitle}>{item.title}</Text>
      <Text style={styles.postBody} numberOfLines={2}>
        {item.body}
      </Text>
      <TouchableOpacity
        onPress={() => handleDelete(item.id)}
        style={styles.deleteButton}
      >
        <Text style={styles.deleteText}>ลบ</Text>
      </TouchableOpacity>
    </View>
  );

  return (
    <View style={styles.container}>
      {/* Create Form */}
      <View style={styles.form}>
        <TextInput
          value={newTitle}
          onChangeText={setNewTitle}
          placeholder="หัวข้อ post"
          style={styles.input}
        />
        <TextInput
          value={newBody}
          onChangeText={setNewBody}
          placeholder="เนื้อหา"
          style={[styles.input, styles.bodyInput]}
          multiline
        />
        <TouchableOpacity
          onPress={handleCreate}
          style={styles.createButton}
          disabled={isCreating}
        >
          {isCreating ? (
            <ActivityIndicator color="#fff" />
          ) : (
            <Text style={styles.createButtonText}>สร้าง Post</Text>
          )}
        </TouchableOpacity>
      </View>

      {/* Refresh indicator */}
      {isFetching && (
        <View style={styles.refreshBar}>
          <ActivityIndicator size="small" color="#6200EE" />
          <Text style={styles.refreshText}>กำลังอัปเดต...</Text>
        </View>
      )}

      {/* Posts List */}
      <FlatList
        data={posts?.slice(0, 20)} // แสดงแค่ 20 รายการ
        keyExtractor={(item) => item.id.toString()}
        renderItem={renderPost}
        contentContainerStyle={styles.listContent}
        onRefresh={refetch}
        refreshing={isFetching && !isLoading}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  form: {
    backgroundColor: '#fff',
    padding: 16,
    margin: 16,
    borderRadius: 12,
    elevation: 2,
  },
  input: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 8,
    padding: 12,
    marginBottom: 8,
    fontSize: 14,
  },
  bodyInput: { height: 80, textAlignVertical: 'top' },
  createButton: {
    backgroundColor: '#6200EE',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  createButtonText: { color: '#fff', fontWeight: 'bold' },
  refreshBar: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 8,
    backgroundColor: '#EDE7F6',
  },
  refreshText: { marginLeft: 8, color: '#6200EE', fontSize: 12 },
  listContent: { paddingHorizontal: 16, paddingBottom: 20 },
  postCard: {
    backgroundColor: '#fff',
    padding: 16,
    marginVertical: 4,
    borderRadius: 8,
    elevation: 1,
  },
  postTitle: { fontSize: 16, fontWeight: 'bold', marginBottom: 4 },
  postBody: { fontSize: 14, color: '#666', marginBottom: 8 },
  deleteButton: {
    alignSelf: 'flex-end',
    backgroundColor: '#FF5252',
    paddingHorizontal: 12,
    paddingVertical: 4,
    borderRadius: 4,
  },
  deleteText: { color: '#fff', fontSize: 12 },
  errorText: { fontSize: 16, color: '#F44336', marginBottom: 12 },
  retryButton: { backgroundColor: '#6200EE', padding: 12, borderRadius: 8 },
  retryText: { color: '#fff' },
});
```

---

## Advanced Thunk Patterns

### การส่ง Multiple Dispatches

```typescript
export const loginUser = createAsyncThunk<
  User,
  { email: string; password: string }
>(
  'auth/login',
  async (credentials, { dispatch, getState, rejectWithValue }) => {
    try {
      // 1. Set loading
      dispatch(setLoading(true));

      // 2. Fetch token
      const tokenResponse = await authApi.login(credentials);

      // 3. Save token
      await SecureStore.setItemAsync('token', tokenResponse.token);

      // 4. Fetch user profile
      const user = await userApi.getProfile(tokenResponse.token);

      // 5. Load user's data
      dispatch(fetchUserTodos(user.id));

      return user;
    } catch (error) {
      return rejectWithValue(
        error instanceof Error ? error.message : 'Login failed'
      );
    } finally {
      dispatch(setLoading(false));
    }
  }
);
```

### Thunk Cancellation

```typescript
const fetchDataThunk = createAsyncThunk(
  'data/fetch',
  async (_, { signal, rejectWithValue }) => {
    try {
      const response = await fetch('/api/data', { signal });
      return response.json();
    } catch (error) {
      if (error.name === 'AbortError') {
        return rejectWithValue('Request cancelled');
      }
      return rejectWithValue('Request failed');
    }
  }
);

// ใน component
useEffect(() => {
  const promise = dispatch(fetchDataThunk());

  return () => {
    promise.abort(); // Cancel เมื่อ unmount
  };
}, []);
```

### Conditional Thunk

```typescript
export const fetchPostsIfNeeded = createAsyncThunk(
  'posts/fetchIfNeeded',
  async (_, { getState, dispatch }) => {
    const state = getState() as RootState;

    // ตรวจสอบว่า data เก่าเกินไปหรือเปล่า
    const lastFetch = state.posts.lastFetch;
    const isOld = !lastFetch || Date.now() - lastFetch > 5 * 60 * 1000; // 5 นาที

    if (state.posts.status !== 'loading' && isOld) {
      return dispatch(fetchPosts());
    }
  }
);
```

---

## Workshop: Fetch Data จาก API

### Project: News App

```typescript
// src/store/slices/newsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

export interface NewsArticle {
  id: string;
  title: string;
  description: string;
  url: string;
  imageUrl: string;
  publishedAt: string;
  source: string;
  category: string;
}

interface NewsState {
  articles: NewsArticle[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  page: number;
  hasMore: boolean;
  selectedCategory: string;
  bookmarks: string[];
}

const CATEGORIES = ['general', 'technology', 'sports', 'business', 'health'];

// Mock API call (แทน real API)
const mockFetchNews = async (
  category: string,
  page: number
): Promise<NewsArticle[]> => {
  await new Promise((resolve) => setTimeout(resolve, 1000)); // Simulate delay

  return Array.from({ length: 10 }, (_, i) => ({
    id: `${category}-${page}-${i}`,
    title: `ข่าว${category} หน้า ${page} #${i + 1}`,
    description: `รายละเอียดข่าว${category}ที่น่าสนใจ`,
    url: `https://example.com/news/${category}-${page}-${i}`,
    imageUrl: `https://picsum.photos/seed/${category}${page}${i}/400/200`,
    publishedAt: new Date().toISOString(),
    source: 'Thai News',
    category,
  }));
};

export const fetchNews = createAsyncThunk<
  { articles: NewsArticle[]; page: number; hasMore: boolean },
  { category: string; page: number; refresh?: boolean }
>(
  'news/fetch',
  async ({ category, page }, { rejectWithValue }) => {
    try {
      const articles = await mockFetchNews(category, page);
      return {
        articles,
        page,
        hasMore: page < 5, // Max 5 pages for mock
      };
    } catch (error) {
      return rejectWithValue('ไม่สามารถโหลดข่าวได้');
    }
  }
);

const initialState: NewsState = {
  articles: [],
  status: 'idle',
  error: null,
  page: 1,
  hasMore: true,
  selectedCategory: 'general',
  bookmarks: [],
};

const newsSlice = createSlice({
  name: 'news',
  initialState,
  reducers: {
    setCategory: (state, action: PayloadAction<string>) => {
      state.selectedCategory = action.payload;
      state.articles = [];
      state.page = 1;
      state.hasMore = true;
      state.status = 'idle';
    },
    toggleBookmark: (state, action: PayloadAction<string>) => {
      const id = action.payload;
      if (state.bookmarks.includes(id)) {
        state.bookmarks = state.bookmarks.filter((b) => b !== id);
      } else {
        state.bookmarks.push(id);
      }
    },
    resetNews: (state) => {
      state.articles = [];
      state.page = 1;
      state.hasMore = true;
      state.status = 'idle';
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchNews.pending, (state, action) => {
        state.status = 'loading';
        state.error = null;
        // ถ้าเป็น refresh (page 1) ล้าง articles ก่อน
        if (action.meta.arg.page === 1) {
          state.articles = [];
        }
      })
      .addCase(fetchNews.fulfilled, (state, action) => {
        state.status = 'succeeded';
        const { articles, page, hasMore } = action.payload;

        if (page === 1) {
          state.articles = articles;
        } else {
          state.articles.push(...articles);
        }

        state.page = page;
        state.hasMore = hasMore;
      })
      .addCase(fetchNews.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload as string;
      });
  },
});

export const { setCategory, toggleBookmark, resetNews } = newsSlice.actions;
export default newsSlice.reducer;

export const selectNews = (state: { news: NewsState }) => state.news.articles;
export const selectNewsStatus = (state: { news: NewsState }) =>
  state.news.status;
export const selectNewsError = (state: { news: NewsState }) => state.news.error;
export const selectHasMore = (state: { news: NewsState }) => state.news.hasMore;
export const selectCurrentPage = (state: { news: NewsState }) => state.news.page;
export const selectBookmarks = (state: { news: NewsState }) =>
  state.news.bookmarks;
export const selectSelectedCategory = (state: { news: NewsState }) =>
  state.news.selectedCategory;
```

### News Screen

```typescript
// src/screens/NewsScreen.tsx
import React, { useEffect } from 'react';
import {
  View,
  FlatList,
  Text,
  TouchableOpacity,
  Image,
  StyleSheet,
  ActivityIndicator,
  ScrollView,
  RefreshControl,
} from 'react-native';
import { useAppDispatch, useAppSelector } from '../store/hooks';
import {
  fetchNews,
  setCategory,
  toggleBookmark,
  selectNews,
  selectNewsStatus,
  selectHasMore,
  selectCurrentPage,
  selectSelectedCategory,
  selectBookmarks,
  NewsArticle,
} from '../store/slices/newsSlice';

const CATEGORIES = [
  { id: 'general', label: 'ทั่วไป' },
  { id: 'technology', label: 'เทคโนโลยี' },
  { id: 'sports', label: 'กีฬา' },
  { id: 'business', label: 'ธุรกิจ' },
  { id: 'health', label: 'สุขภาพ' },
];

export const NewsScreen: React.FC = () => {
  const dispatch = useAppDispatch();
  const articles = useAppSelector(selectNews);
  const status = useAppSelector(selectNewsStatus);
  const hasMore = useAppSelector(selectHasMore);
  const currentPage = useAppSelector(selectCurrentPage);
  const selectedCategory = useAppSelector(selectSelectedCategory);
  const bookmarks = useAppSelector(selectBookmarks);

  // Load initial data
  useEffect(() => {
    dispatch(fetchNews({ category: selectedCategory, page: 1 }));
  }, [selectedCategory]);

  const handleRefresh = () => {
    dispatch(fetchNews({ category: selectedCategory, page: 1, refresh: true }));
  };

  const handleLoadMore = () => {
    if (status !== 'loading' && hasMore) {
      dispatch(
        fetchNews({ category: selectedCategory, page: currentPage + 1 })
      );
    }
  };

  const handleCategoryChange = (category: string) => {
    dispatch(setCategory(category));
  };

  const handleBookmark = (articleId: string) => {
    dispatch(toggleBookmark(articleId));
  };

  const renderArticle = ({ item }: { item: NewsArticle }) => (
    <View style={styles.articleCard}>
      <Image
        source={{ uri: item.imageUrl }}
        style={styles.articleImage}
        resizeMode="cover"
      />
      <View style={styles.articleContent}>
        <Text style={styles.articleSource}>{item.source}</Text>
        <Text style={styles.articleTitle}>{item.title}</Text>
        <Text style={styles.articleDescription} numberOfLines={2}>
          {item.description}
        </Text>
        <View style={styles.articleFooter}>
          <Text style={styles.publishedAt}>
            {new Date(item.publishedAt).toLocaleDateString('th-TH')}
          </Text>
          <TouchableOpacity
            onPress={() => handleBookmark(item.id)}
            style={styles.bookmarkButton}
          >
            <Text style={styles.bookmarkIcon}>
              {bookmarks.includes(item.id) ? '🔖' : '🏷️'}
            </Text>
          </TouchableOpacity>
        </View>
      </View>
    </View>
  );

  const renderFooter = () => {
    if (!hasMore) {
      return (
        <View style={styles.endMessage}>
          <Text style={styles.endText}>โหลดครบแล้ว</Text>
        </View>
      );
    }
    if (status === 'loading' && articles.length > 0) {
      return (
        <View style={styles.loadingMore}>
          <ActivityIndicator color="#6200EE" />
        </View>
      );
    }
    return null;
  };

  return (
    <View style={styles.container}>
      {/* Category Tabs */}
      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        style={styles.categoryContainer}
        contentContainerStyle={styles.categoryContent}
      >
        {CATEGORIES.map((cat) => (
          <TouchableOpacity
            key={cat.id}
            onPress={() => handleCategoryChange(cat.id)}
            style={[
              styles.categoryTab,
              selectedCategory === cat.id && styles.activeCategoryTab,
            ]}
          >
            <Text
              style={[
                styles.categoryText,
                selectedCategory === cat.id && styles.activeCategoryText,
              ]}
            >
              {cat.label}
            </Text>
          </TouchableOpacity>
        ))}
      </ScrollView>

      {/* Initial Loading */}
      {status === 'loading' && articles.length === 0 ? (
        <View style={styles.center}>
          <ActivityIndicator size="large" color="#6200EE" />
          <Text style={styles.loadingText}>กำลังโหลดข่าว...</Text>
        </View>
      ) : (
        <FlatList
          data={articles}
          keyExtractor={(item) => item.id}
          renderItem={renderArticle}
          onEndReached={handleLoadMore}
          onEndReachedThreshold={0.5}
          ListFooterComponent={renderFooter}
          refreshControl={
            <RefreshControl
              refreshing={status === 'loading' && articles.length > 0}
              onRefresh={handleRefresh}
              colors={['#6200EE']}
            />
          }
          contentContainerStyle={styles.listContent}
        />
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  loadingText: { marginTop: 12, color: '#666' },
  categoryContainer: {
    backgroundColor: '#fff',
    elevation: 2,
    maxHeight: 50,
  },
  categoryContent: { paddingHorizontal: 12 },
  categoryTab: {
    paddingHorizontal: 16,
    paddingVertical: 12,
    marginRight: 4,
  },
  activeCategoryTab: {
    borderBottomWidth: 2,
    borderBottomColor: '#6200EE',
  },
  categoryText: { fontSize: 14, color: '#666' },
  activeCategoryText: { color: '#6200EE', fontWeight: 'bold' },
  listContent: { padding: 16 },
  articleCard: {
    backgroundColor: '#fff',
    borderRadius: 12,
    marginBottom: 16,
    overflow: 'hidden',
    elevation: 2,
  },
  articleImage: { width: '100%', height: 180 },
  articleContent: { padding: 12 },
  articleSource: { fontSize: 12, color: '#6200EE', fontWeight: 'bold', marginBottom: 4 },
  articleTitle: { fontSize: 16, fontWeight: 'bold', marginBottom: 6 },
  articleDescription: { fontSize: 14, color: '#666', lineHeight: 20 },
  articleFooter: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginTop: 8,
  },
  publishedAt: { fontSize: 12, color: '#999' },
  bookmarkButton: { padding: 4 },
  bookmarkIcon: { fontSize: 20 },
  loadingMore: { padding: 20, alignItems: 'center' },
  endMessage: { padding: 20, alignItems: 'center' },
  endText: { color: '#999' },
});
```

---

## Tips and Best Practices

### 1. ใช้ unwrap() สำหรับ error handling

```typescript
const handleSubmit = async () => {
  try {
    const result = await dispatch(createPost(data)).unwrap();
    // Success! result คือ payload
    navigation.goBack();
  } catch (error) {
    // Error! error คือ rejectWithValue payload
    Alert.alert('Error', error as string);
  }
};
```

### 2. Track Request Status แยกกัน

```typescript
interface PostsState {
  items: Post[];
  fetchStatus: 'idle' | 'loading' | 'succeeded' | 'failed';
  createStatus: 'idle' | 'loading' | 'succeeded' | 'failed';
  deleteStatus: { [id: number]: 'loading' | 'succeeded' | 'failed' };
}
```

### 3. Retry Logic

```typescript
export const fetchWithRetry = createAsyncThunk(
  'data/fetchWithRetry',
  async (_, { dispatch, rejectWithValue }) => {
    const MAX_RETRIES = 3;

    for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
      try {
        const response = await fetch('/api/data');
        return response.json();
      } catch (error) {
        if (attempt === MAX_RETRIES) {
          return rejectWithValue('Failed after 3 attempts');
        }
        // Wait before retry: 1s, 2s, 4s (exponential backoff)
        await new Promise((r) => setTimeout(r, Math.pow(2, attempt - 1) * 1000));
      }
    }
  }
);
```

---

## สรุป

Redux Async Actions ด้วย `createAsyncThunk`:

1. **createAsyncThunk** - สร้าง pending/fulfilled/rejected actions อัตโนมัติ
2. **extraReducers** - จัดการ async states ใน slice
3. **rejectWithValue** - ส่ง custom error payload
4. **RTK Query** - สำหรับ data fetching แบบ advanced พร้อม caching
5. **unwrap()** - handle result/error หลัง dispatch

ใน Part 033 จะเรียนเรื่อง **Zustand** ซึ่งเป็น state management ที่เบากว่าและง่ายกว่า Redux
