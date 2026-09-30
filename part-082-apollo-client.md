# Part 082: Apollo Client

## Apollo Client คืออะไร?

Apollo Client คือ comprehensive state management library สำหรับ JavaScript ที่ช่วยจัดการ data จาก GraphQL API รวมถึง caching, optimistic updates, real-time subscriptions, และ local state management

---

## 1. Setup Apollo Client

### 1.1 ติดตั้ง

```bash
npm install @apollo/client graphql
npm install @react-native-async-storage/async-storage
```

### 1.2 Advanced Client Setup

```typescript
// apollo/ApolloConfig.ts
import {
  ApolloClient,
  InMemoryCache,
  createHttpLink,
  from,
  split
} from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { RetryLink } from '@apollo/client/link/retry';
import { onError } from '@apollo/client/link/error';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';
import AsyncStorage from '@react-native-async-storage/async-storage';

const TOKEN_KEY = '@auth_token';

// Token management
export const tokenManager = {
  async get(): Promise<string | null> {
    return AsyncStorage.getItem(TOKEN_KEY);
  },
  async set(token: string): Promise<void> {
    return AsyncStorage.setItem(TOKEN_KEY, token);
  },
  async remove(): Promise<void> {
    return AsyncStorage.removeItem(TOKEN_KEY);
  }
};

// HTTP Link
const httpLink = createHttpLink({
  uri: process.env.GRAPHQL_URL || 'https://api.example.com/graphql',
  fetch: (uri, options) => {
    // Add timeout
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 30000);
    
    return fetch(uri, {
      ...options,
      signal: controller.signal
    }).finally(() => clearTimeout(timeoutId));
  }
});

// Auth Link
const authLink = setContext(async (_, { headers }) => {
  const token = await tokenManager.get();
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : ''
    }
  };
});

// Retry Link
const retryLink = new RetryLink({
  delay: {
    initial: 300,
    max: 3000,
    jitter: true
  },
  attempts: {
    max: 3,
    retryIf: (error) => {
      return error.networkError !== null;
    }
  }
});

// Error Link
const errorLink = onError(({ graphQLErrors, networkError, operation, forward }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, extensions }) => {
      console.error('[GraphQL Error]:', message);
      
      if (extensions?.code === 'UNAUTHENTICATED') {
        // Handle token expiration
        tokenManager.remove().then(() => {
          // Redirect to login
        });
      }
    });
  }
  
  if (networkError) {
    console.error('[Network Error]:', networkError);
    
    if ('statusCode' in networkError) {
      if (networkError.statusCode === 401) {
        tokenManager.remove();
      }
    }
  }
});

// WebSocket Link for subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: process.env.GRAPHQL_WS_URL || 'wss://api.example.com/graphql',
    connectionParams: async () => {
      const token = await tokenManager.get();
      return { Authorization: token ? `Bearer ${token}` : '' };
    },
    on: {
      connected: () => console.log('WebSocket connected'),
      closed: () => console.log('WebSocket closed'),
      error: (error) => console.error('WebSocket error:', error)
    }
  })
);

// Split link
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  from([retryLink, authLink, httpLink])
);

// Cache
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        // Pagination
        posts: {
          keyArgs: ['filter'],
          merge(existing = [], incoming, { args }) {
            // If loading first page, replace
            if (!args?.after) return incoming;
            // Otherwise append
            return [...existing, ...incoming];
          }
        },
        // User feed
        feedPosts: {
          keyArgs: false,
          merge(existing = { edges: [], pageInfo: null }, incoming) {
            return {
              ...incoming,
              edges: [
                ...(existing.edges || []),
                ...(incoming.edges || [])
              ]
            };
          }
        }
      }
    },
    Post: {
      fields: {
        likes: {
          read(existing = 0) {
            return existing;
          }
        },
        isLiked: {
          read(existing = false) {
            return existing;
          }
        }
      }
    },
    User: {
      fields: {
        isFollowing: {
          read(existing = false) {
            return existing;
          }
        }
      }
    }
  }
});

export const apolloClient = new ApolloClient({
  link: from([errorLink, splitLink]),
  cache,
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-and-network',
      nextFetchPolicy: 'cache-first',
      errorPolicy: 'all'
    },
    query: {
      fetchPolicy: 'cache-first',
      errorPolicy: 'all'
    },
    mutate: {
      errorPolicy: 'all'
    }
  },
  connectToDevTools: __DEV__
});
```

---

## 2. useQuery Hook

### 2.1 Basic useQuery

```tsx
// hooks/useUserProfile.ts
import { useQuery, gql } from '@apollo/client';

const GET_USER_PROFILE = gql`
  query GetUserProfile($userId: ID!) {
    user(id: $userId) {
      id
      username
      displayName
      avatar
      bio
      postsCount
      followersCount
      followingCount
      isFollowing
    }
  }
`;

interface UserProfile {
  id: string;
  username: string;
  displayName: string | null;
  avatar: string | null;
  bio: string | null;
  postsCount: number;
  followersCount: number;
  followingCount: number;
  isFollowing: boolean;
}

export function useUserProfile(userId: string) {
  const { data, loading, error, refetch } = useQuery<{
    user: UserProfile;
  }>(GET_USER_PROFILE, {
    variables: { userId },
    skip: !userId
  });
  
  return {
    user: data?.user ?? null,
    loading,
    error,
    refetch
  };
}
```

### 2.2 Lazy Query

```typescript
// hooks/useSearchUsers.ts
import { useLazyQuery, gql } from '@apollo/client';
import { useCallback, useState } from 'react';

const SEARCH_USERS = gql`
  query SearchUsers($query: String!, $limit: Int) {
    users(search: $query, limit: $limit) {
      id
      username
      displayName
      avatar
    }
  }
`;

export function useSearchUsers() {
  const [searchQuery, setSearchQuery] = useState('');
  
  const [search, { data, loading, error }] = useLazyQuery(SEARCH_USERS, {
    fetchPolicy: 'cache-and-network'
  });
  
  const handleSearch = useCallback((query: string) => {
    setSearchQuery(query);
    if (query.trim().length >= 2) {
      search({
        variables: { query: query.trim(), limit: 20 }
      });
    }
  }, [search]);
  
  return {
    searchQuery,
    results: data?.users ?? [],
    loading,
    error,
    handleSearch
  };
}
```

---

## 3. useMutation Hook

### 3.1 Mutations with Optimistic Updates

```typescript
// hooks/useLikePost.ts
import { useMutation, gql, useApolloClient } from '@apollo/client';

const LIKE_POST = gql`
  mutation LikePost($postId: ID!) {
    likePost(id: $postId) {
      id
      likes
      isLiked
    }
  }
`;

const UNLIKE_POST = gql`
  mutation UnlikePost($postId: ID!) {
    unlikePost(id: $postId) {
      id
      likes
      isLiked
    }
  }
`;

export function useLikePost() {
  const client = useApolloClient();
  
  const [like] = useMutation(LIKE_POST);
  const [unlike] = useMutation(UNLIKE_POST);
  
  const toggleLike = async (postId: string, currentlyLiked: boolean, currentLikes: number) => {
    const mutation = currentlyLiked ? unlike : like;
    
    // Optimistic update
    client.cache.modify({
      id: client.cache.identify({ __typename: 'Post', id: postId }),
      fields: {
        isLiked: () => !currentlyLiked,
        likes: () => currentlyLiked ? currentLikes - 1 : currentLikes + 1
      }
    });
    
    try {
      await mutation({
        variables: { postId }
      });
    } catch (error) {
      // Revert optimistic update on error
      client.cache.modify({
        id: client.cache.identify({ __typename: 'Post', id: postId }),
        fields: {
          isLiked: () => currentlyLiked,
          likes: () => currentLikes
        }
      });
      throw error;
    }
  };
  
  return { toggleLike };
}
```

### 3.2 Complex Mutation with Cache Update

```typescript
// hooks/useCreateComment.ts
import { useMutation, gql } from '@apollo/client';

const CREATE_COMMENT = gql`
  mutation CreateComment($postId: ID!, $content: String!) {
    createComment(input: { postId: $postId, content: $content }) {
      id
      content
      createdAt
      author {
        id
        username
        avatar
      }
    }
  }
`;

const GET_POST_COMMENTS = gql`
  query GetPostComments($postId: ID!) {
    post(id: $postId) {
      id
      comments {
        id
        content
        createdAt
        author {
          id
          username
          avatar
        }
      }
    }
  }
`;

export function useCreateComment(postId: string) {
  const [createComment, { loading }] = useMutation(CREATE_COMMENT, {
    update(cache, { data: { createComment: newComment } }) {
      // Read existing post data
      const existingData = cache.readQuery<any>({
        query: GET_POST_COMMENTS,
        variables: { postId }
      });
      
      if (existingData) {
        // Write updated data
        cache.writeQuery({
          query: GET_POST_COMMENTS,
          variables: { postId },
          data: {
            post: {
              ...existingData.post,
              comments: [...existingData.post.comments, newComment]
            }
          }
        });
      }
    }
  });
  
  const addComment = async (content: string) => {
    if (!content.trim()) return;
    
    await createComment({
      variables: { postId, content: content.trim() },
      // Optimistic response
      optimisticResponse: {
        createComment: {
          __typename: 'Comment',
          id: `temp-${Date.now()}`,
          content: content.trim(),
          createdAt: new Date().toISOString(),
          author: {
            __typename: 'User',
            id: 'current-user',
            username: 'You',
            avatar: null
          }
        }
      }
    });
  };
  
  return { addComment, loading };
}
```

---

## 4. Cache Management

### 4.1 Manual Cache Operations

```typescript
// cache/CacheManager.ts
import { ApolloClient, gql } from '@apollo/client';

export class CacheManager {
  constructor(private client: ApolloClient<any>) {}
  
  // อ่านจาก cache
  readPost(postId: string) {
    return this.client.readFragment({
      id: `Post:${postId}`,
      fragment: gql`
        fragment PostData on Post {
          id
          title
          likes
          isLiked
        }
      `
    });
  }
  
  // เขียนลง cache
  updatePostLikes(postId: string, likes: number, isLiked: boolean) {
    this.client.writeFragment({
      id: `Post:${postId}`,
      fragment: gql`
        fragment UpdatePostLikes on Post {
          likes
          isLiked
        }
      `,
      data: { likes, isLiked }
    });
  }
  
  // ลบ item จาก list ใน cache
  removePostFromCache(postId: string) {
    this.client.cache.modify({
      fields: {
        feedPosts: (existingRef, { readField }) => {
          return {
            ...existingRef,
            edges: existingRef.edges.filter(
              (edge: any) => readField('id', readField('node', edge)) !== postId
            )
          };
        }
      }
    });
    
    // Evict from cache
    this.client.cache.evict({ id: `Post:${postId}` });
    this.client.cache.gc(); // Garbage collect
  }
  
  // Reset cache (เช่น หลัง logout)
  async resetCache() {
    await this.client.resetStore();
    // หรือ clear โดยไม่ refetch
    await this.client.clearStore();
  }
}
```

---

## 5. Workshop: Social Feed กับ Apollo

### Step 1: PostCard Component

```tsx
// components/PostCard.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  Alert
} from 'react-native';
import { useLikePost } from '../hooks/useLikePost';
import { useNavigation } from '@react-navigation/native';

interface Post {
  id: string;
  title: string;
  content: string;
  imageUrl: string | null;
  likes: number;
  isLiked: boolean;
  author: {
    id: string;
    username: string;
    avatar: string | null;
    displayName: string | null;
  };
  createdAt: string;
  tags: string[];
}

const PostCard: React.FC<{ post: Post }> = ({ post }) => {
  const navigation = useNavigation<any>();
  const { toggleLike } = useLikePost();
  const [isAnimating, setIsAnimating] = useState(false);
  
  const handleLike = useCallback(async () => {
    if (isAnimating) return;
    setIsAnimating(true);
    
    try {
      await toggleLike(post.id, post.isLiked, post.likes);
    } catch (error) {
      Alert.alert('ผิดพลาด', 'ไม่สามารถ like ได้');
    } finally {
      setTimeout(() => setIsAnimating(false), 300);
    }
  }, [post, toggleLike, isAnimating]);
  
  const handleAuthorPress = useCallback(() => {
    navigation.navigate('UserProfile', { userId: post.author.id });
  }, [post.author.id, navigation]);
  
  const handlePostPress = useCallback(() => {
    navigation.navigate('PostDetail', { postId: post.id });
  }, [post.id, navigation]);
  
  const formatDate = (dateStr: string) => {
    const date = new Date(dateStr);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffMins = Math.floor(diffMs / 60000);
    
    if (diffMins < 1) return 'เมื่อกี้นี้';
    if (diffMins < 60) return `${diffMins} นาทีที่แล้ว`;
    if (diffMins < 1440) return `${Math.floor(diffMins / 60)} ชั่วโมงที่แล้ว`;
    return `${Math.floor(diffMins / 1440)} วันที่แล้ว`;
  };
  
  return (
    <View style={styles.card}>
      {/* Author Header */}
      <TouchableOpacity style={styles.authorRow} onPress={handleAuthorPress}>
        {post.author.avatar ? (
          <Image source={{ uri: post.author.avatar }} style={styles.avatar} />
        ) : (
          <View style={[styles.avatar, styles.avatarPlaceholder]}>
            <Text style={styles.avatarInitial}>
              {post.author.username[0].toUpperCase()}
            </Text>
          </View>
        )}
        <View style={styles.authorInfo}>
          <Text style={styles.authorName}>
            {post.author.displayName || post.author.username}
          </Text>
          <Text style={styles.date}>{formatDate(post.createdAt)}</Text>
        </View>
      </TouchableOpacity>
      
      {/* Content */}
      <TouchableOpacity onPress={handlePostPress}>
        <Text style={styles.title}>{post.title}</Text>
        <Text style={styles.content} numberOfLines={3}>
          {post.content}
        </Text>
        
        {post.imageUrl && (
          <Image
            source={{ uri: post.imageUrl }}
            style={styles.postImage}
            resizeMode="cover"
          />
        )}
      </TouchableOpacity>
      
      {/* Tags */}
      {post.tags.length > 0 && (
        <View style={styles.tags}>
          {post.tags.map(tag => (
            <View key={tag} style={styles.tag}>
              <Text style={styles.tagText}>#{tag}</Text>
            </View>
          ))}
        </View>
      )}
      
      {/* Actions */}
      <View style={styles.actions}>
        <TouchableOpacity
          style={styles.actionButton}
          onPress={handleLike}
        >
          <Text style={[styles.likeIcon, post.isLiked && styles.liked]}>
            {post.isLiked ? '❤️' : '🤍'}
          </Text>
          <Text style={[styles.actionText, post.isLiked && styles.likedText]}>
            {post.likes}
          </Text>
        </TouchableOpacity>
        
        <TouchableOpacity
          style={styles.actionButton}
          onPress={handlePostPress}
        >
          <Text style={styles.actionIcon}>💬</Text>
          <Text style={styles.actionText}>แสดงความคิดเห็น</Text>
        </TouchableOpacity>
        
        <TouchableOpacity style={styles.actionButton}>
          <Text style={styles.actionIcon}>📤</Text>
          <Text style={styles.actionText}>แชร์</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    marginHorizontal: 0,
    marginVertical: 4,
    paddingVertical: 12,
    borderBottomWidth: 8,
    borderBottomColor: '#f0f0f0'
  },
  authorRow: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    marginBottom: 12
  },
  avatar: {
    width: 44,
    height: 44,
    borderRadius: 22,
    marginRight: 10
  },
  avatarPlaceholder: {
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center'
  },
  avatarInitial: {
    color: '#fff',
    fontSize: 18,
    fontWeight: 'bold'
  },
  authorInfo: { flex: 1 },
  authorName: { fontSize: 15, fontWeight: '600' },
  date: { fontSize: 12, color: '#999', marginTop: 2 },
  title: {
    fontSize: 17,
    fontWeight: '700',
    paddingHorizontal: 16,
    marginBottom: 8
  },
  content: {
    fontSize: 15,
    color: '#444',
    lineHeight: 22,
    paddingHorizontal: 16,
    marginBottom: 12
  },
  postImage: {
    width: '100%',
    height: 250,
    marginBottom: 12
  },
  tags: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    paddingHorizontal: 16,
    marginBottom: 12,
    gap: 6
  },
  tag: {
    backgroundColor: '#e8f0fe',
    paddingHorizontal: 8,
    paddingVertical: 4,
    borderRadius: 12
  },
  tagText: { fontSize: 12, color: '#007AFF' },
  actions: {
    flexDirection: 'row',
    paddingHorizontal: 16,
    paddingTop: 8,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0'
  },
  actionButton: {
    flexDirection: 'row',
    alignItems: 'center',
    marginRight: 20,
    gap: 4
  },
  likeIcon: { fontSize: 18 },
  liked: {},
  actionIcon: { fontSize: 18 },
  actionText: { fontSize: 14, color: '#666' },
  likedText: { color: '#FF3B30' }
});

export default PostCard;
```

### Step 2: Full Feed with Subscriptions

```tsx
// screens/SocialFeedScreen.tsx
import React, { useCallback, useEffect } from 'react';
import {
  View,
  FlatList,
  StyleSheet,
  RefreshControl,
  Text,
  TouchableOpacity
} from 'react-native';
import { useQuery, useSubscription, gql, useApolloClient } from '@apollo/client';
import PostCard from '../components/PostCard';
import { useNavigation } from '@react-navigation/native';

const FEED_QUERY = gql`
  query FeedQuery($limit: Int, $after: String) {
    feedPosts(limit: $limit, after: $after) {
      edges {
        node {
          id
          title
          content
          imageUrl
          likes
          isLiked
          createdAt
          tags
          author {
            id
            username
            displayName
            avatar
          }
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;

const NEW_POST_SUB = gql`
  subscription {
    newPost {
      id
      title
      content
      imageUrl
      likes
      isLiked
      createdAt
      tags
      author {
        id
        username
        displayName
        avatar
      }
    }
  }
`;

const SocialFeedScreen: React.FC = () => {
  const navigation = useNavigation<any>();
  const client = useApolloClient();
  
  const { data, loading, fetchMore, refetch } = useQuery(FEED_QUERY, {
    variables: { limit: 10 },
    notifyOnNetworkStatusChange: true
  });
  
  // Subscribe to new posts
  useSubscription(NEW_POST_SUB, {
    onData: ({ client, data }) => {
      if (!data.data?.newPost) return;
      const newPost = data.data.newPost;
      
      // Prepend to feed
      client.cache.modify({
        fields: {
          feedPosts: (existing) => ({
            ...existing,
            edges: [
              {
                __typename: 'PostEdge',
                node: newPost,
                cursor: newPost.id
              },
              ...existing.edges
            ]
          })
        }
      });
    }
  });
  
  const loadMore = useCallback(() => {
    const pageInfo = data?.feedPosts?.pageInfo;
    if (!pageInfo?.hasNextPage || loading) return;
    
    fetchMore({
      variables: {
        limit: 10,
        after: pageInfo.endCursor
      }
    });
  }, [data, fetchMore, loading]);
  
  const posts = data?.feedPosts?.edges?.map((e: any) => e.node) ?? [];
  
  return (
    <View style={styles.container}>
      <FlatList
        data={posts}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <PostCard post={item} />}
        onEndReached={loadMore}
        onEndReachedThreshold={0.5}
        refreshControl={
          <RefreshControl
            refreshing={loading && posts.length === 0}
            onRefresh={refetch}
          />
        }
        ListHeaderComponent={
          <View style={styles.header}>
            <Text style={styles.headerTitle}>Feed</Text>
            <TouchableOpacity
              style={styles.createButton}
              onPress={() => navigation.navigate('CreatePost')}
            >
              <Text style={styles.createButtonText}>+ สร้างโพสต์</Text>
            </TouchableOpacity>
          </View>
        }
        ItemSeparatorComponent={() => <View style={styles.separator} />}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f0f0f0' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0'
  },
  headerTitle: { fontSize: 20, fontWeight: 'bold' },
  createButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20
  },
  createButtonText: { color: '#fff', fontSize: 14, fontWeight: '600' },
  separator: { height: 8, backgroundColor: '#f0f0f0' }
});

export default SocialFeedScreen;
```

---

## Tips สำหรับ Apollo Client

1. **Cache correctly** - configure typePolicies ให้ถูกต้องตั้งแต่ต้น
2. **Optimistic updates** - ทำให้ UI รู้สึกเร็วขึ้นมาก
3. **Error handling** - handle errors ทั้งระดับ GraphQL และ Network
4. **Fragments** - ใช้ fragments เพื่อ reuse
5. **Subscriptions carefully** - อย่า subscribe ทุกอย่าง ใช้เมื่อจำเป็น

## สรุป

Apollo Client เป็น powerful tool สำหรับ GraphQL ที่:
1. จัดการ data fetching และ caching อัตโนมัติ
2. มี optimistic updates built-in
3. รองรับ real-time subscriptions
4. ทำงานได้ดีกับ TypeScript
5. มี devtools ที่ดีเยี่ยม
