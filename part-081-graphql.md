# Part 081: GraphQL

## GraphQL คืออะไร?

GraphQL คือ query language สำหรับ API และ runtime ที่ใช้ execute queries กับ data ที่มีอยู่ พัฒนาโดย Facebook ช่วยให้ client ขอ data ที่ต้องการได้แม่นยำ ไม่มากไม่น้อย

## GraphQL vs REST

| Feature | REST | GraphQL |
|---------|------|---------|
| Data fetching | Multiple endpoints | Single endpoint |
| Over-fetching | มักเกิดขึ้น | ไม่เกิด - ขอเฉพาะ fields ที่ต้องการ |
| Under-fetching | มักเกิดขึ้น | ไม่เกิด - ดึงทุกอย่างได้ในครั้งเดียว |
| Type system | ไม่ required | Built-in |
| Real-time | ต้องใช้ WebSocket แยก | Built-in Subscriptions |

---

## 1. GraphQL Basics

### 1.1 Schema Definition

```graphql
# schema.graphql

# Types
type User {
  id: ID!
  username: String!
  email: String!
  displayName: String
  avatar: String
  bio: String
  createdAt: String!
  posts: [Post!]!
  followers: [User!]!
  following: [User!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  imageUrl: String
  author: User!
  likes: Int!
  comments: [Comment!]!
  tags: [String!]!
  createdAt: String!
  updatedAt: String!
}

type Comment {
  id: ID!
  content: String!
  author: User!
  post: Post!
  createdAt: String!
}

# Input types
input CreatePostInput {
  title: String!
  content: String!
  imageUrl: String
  tags: [String!]
}

input UpdatePostInput {
  id: ID!
  title: String
  content: String
  imageUrl: String
  tags: [String!]
}

input CreateCommentInput {
  postId: ID!
  content: String!
}

# Queries
type Query {
  # User queries
  me: User
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  
  # Post queries
  post(id: ID!): Post
  posts(limit: Int, offset: Int, filter: PostFilterInput): PostConnection!
  feedPosts(limit: Int, after: String): PostConnection!
  userPosts(userId: ID!, limit: Int, offset: Int): [Post!]!
}

input PostFilterInput {
  authorId: ID
  tags: [String!]
  search: String
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Mutations
type Mutation {
  # Auth
  login(email: String!, password: String!): AuthPayload!
  register(email: String!, password: String!, username: String!): AuthPayload!
  logout: Boolean!
  
  # Posts
  createPost(input: CreatePostInput!): Post!
  updatePost(input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  likePost(id: ID!): Post!
  unlikePost(id: ID!): Post!
  
  # Comments
  createComment(input: CreateCommentInput!): Comment!
  deleteComment(id: ID!): Boolean!
  
  # User
  updateProfile(input: UpdateProfileInput!): User!
  followUser(id: ID!): Boolean!
  unfollowUser(id: ID!): Boolean!
}

input UpdateProfileInput {
  displayName: String
  bio: String
  avatar: String
}

type AuthPayload {
  token: String!
  user: User!
}

# Subscriptions
type Subscription {
  newPost: Post!
  postLiked(postId: ID!): Post!
  newComment(postId: ID!): Comment!
  newFollower: User!
}
```

---

## 2. Apollo Client Setup

### 2.1 ติดตั้ง Dependencies

```bash
npm install @apollo/client graphql
npm install @react-native-async-storage/async-storage
```

### 2.2 Apollo Client Configuration

```typescript
// apollo/client.ts
import {
  ApolloClient,
  InMemoryCache,
  createHttpLink,
  split,
  from,
  ApolloLink
} from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { onError } from '@apollo/client/link/error';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';
import AsyncStorage from '@react-native-async-storage/async-storage';

// HTTP Link
const httpLink = createHttpLink({
  uri: 'https://api.myapp.com/graphql'
});

// WebSocket Link for Subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: 'wss://api.myapp.com/graphql',
    connectionParams: async () => {
      const token = await AsyncStorage.getItem('auth_token');
      return {
        Authorization: token ? `Bearer ${token}` : ''
      };
    }
  })
);

// Auth Link - add token to headers
const authLink = setContext(async (_, { headers }) => {
  const token = await AsyncStorage.getItem('auth_token');
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : ''
    }
  };
});

// Error Link
const errorLink = onError(({ graphQLErrors, networkError, operation, forward }) => {
  if (graphQLErrors) {
    for (const { message, locations, path, extensions } of graphQLErrors) {
      console.error(
        `[GraphQL error]: Message: ${message}, Location: ${locations}, Path: ${path}`
      );
      
      // Handle auth errors
      if (extensions?.code === 'UNAUTHENTICATED') {
        AsyncStorage.removeItem('auth_token');
        // Navigate to login
      }
    }
  }
  
  if (networkError) {
    console.error(`[Network error]: ${networkError}`);
  }
});

// Split link - use WebSocket for subscriptions, HTTP for others
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  from([authLink, httpLink])
);

// Cache configuration
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        feedPosts: {
          // Pagination cache policy
          keyArgs: false,
          merge(existing = { edges: [] }, incoming) {
            return {
              ...incoming,
              edges: [...(existing.edges || []), ...incoming.edges]
            };
          }
        }
      }
    },
    Post: {
      fields: {
        // Custom field policies
        likes: {
          merge: false
        }
      }
    }
  }
});

// Create Apollo Client
export const apolloClient = new ApolloClient({
  link: from([errorLink, splitLink]),
  cache,
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-and-network',
      errorPolicy: 'all'
    },
    query: {
      fetchPolicy: 'network-only',
      errorPolicy: 'all'
    }
  }
});
```

---

## 3. Queries

### 3.1 Query Definitions

```typescript
// graphql/queries.ts
import { gql } from '@apollo/client';

// Fragment - ใช้ซ้ำได้
export const USER_FRAGMENT = gql`
  fragment UserFragment on User {
    id
    username
    displayName
    avatar
    bio
  }
`;

export const POST_FRAGMENT = gql`
  fragment PostFragment on Post {
    id
    title
    content
    imageUrl
    likes
    createdAt
    author {
      ...UserFragment
    }
    tags
  }
  ${USER_FRAGMENT}
`;

// Queries
export const GET_ME = gql`
  query GetMe {
    me {
      ...UserFragment
      email
      posts {
        id
        title
        createdAt
      }
      followers {
        id
        username
      }
      following {
        id
        username
      }
    }
  }
  ${USER_FRAGMENT}
`;

export const GET_FEED = gql`
  query GetFeed($limit: Int, $after: String) {
    feedPosts(limit: $limit, after: $after) {
      edges {
        node {
          ...PostFragment
          comments {
            id
            content
            author {
              ...UserFragment
            }
          }
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
      totalCount
    }
  }
  ${POST_FRAGMENT}
  ${USER_FRAGMENT}
`;

export const GET_POST = gql`
  query GetPost($id: ID!) {
    post(id: $id) {
      ...PostFragment
      comments {
        id
        content
        createdAt
        author {
          ...UserFragment
        }
      }
    }
  }
  ${POST_FRAGMENT}
  ${USER_FRAGMENT}
`;

export const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      ...UserFragment
      posts {
        ...PostFragment
      }
      followers {
        ...UserFragment
      }
      following {
        ...UserFragment
      }
    }
  }
  ${USER_FRAGMENT}
  ${POST_FRAGMENT}
`;
```

### 3.2 Mutations

```typescript
// graphql/mutations.ts
import { gql } from '@apollo/client';
import { USER_FRAGMENT, POST_FRAGMENT } from './queries';

export const LOGIN = gql`
  mutation Login($email: String!, $password: String!) {
    login(email: $email, password: $password) {
      token
      user {
        ...UserFragment
        email
      }
    }
  }
  ${USER_FRAGMENT}
`;

export const REGISTER = gql`
  mutation Register($email: String!, $password: String!, $username: String!) {
    register(email: $email, password: $password, username: $username) {
      token
      user {
        ...UserFragment
        email
      }
    }
  }
  ${USER_FRAGMENT}
`;

export const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      ...PostFragment
    }
  }
  ${POST_FRAGMENT}
  ${USER_FRAGMENT}
`;

export const LIKE_POST = gql`
  mutation LikePost($id: ID!) {
    likePost(id: $id) {
      id
      likes
    }
  }
`;

export const CREATE_COMMENT = gql`
  mutation CreateComment($input: CreateCommentInput!) {
    createComment(input: $input) {
      id
      content
      createdAt
      author {
        ...UserFragment
      }
    }
  }
  ${USER_FRAGMENT}
`;

export const FOLLOW_USER = gql`
  mutation FollowUser($id: ID!) {
    followUser(id: $id)
  }
`;
```

### 3.3 Subscriptions

```typescript
// graphql/subscriptions.ts
import { gql } from '@apollo/client';
import { POST_FRAGMENT, USER_FRAGMENT } from './queries';

export const NEW_POST_SUBSCRIPTION = gql`
  subscription OnNewPost {
    newPost {
      ...PostFragment
    }
  }
  ${POST_FRAGMENT}
  ${USER_FRAGMENT}
`;

export const POST_LIKED_SUBSCRIPTION = gql`
  subscription OnPostLiked($postId: ID!) {
    postLiked(postId: $postId) {
      id
      likes
    }
  }
`;

export const NEW_COMMENT_SUBSCRIPTION = gql`
  subscription OnNewComment($postId: ID!) {
    newComment(postId: $postId) {
      id
      content
      createdAt
      author {
        ...UserFragment
      }
    }
  }
  ${USER_FRAGMENT}
`;
```

---

## 4. Workshop: GraphQL API App

### Step 1: Setup Provider

```tsx
// App.tsx
import React from 'react';
import { ApolloProvider } from '@apollo/client';
import { apolloClient } from './apollo/client';
import AppNavigator from './navigation/AppNavigator';

export default function App() {
  return (
    <ApolloProvider client={apolloClient}>
      <AppNavigator />
    </ApolloProvider>
  );
}
```

### Step 2: Feed Screen with Queries

```tsx
// screens/FeedScreen.tsx
import React, { useCallback } from 'react';
import {
  View,
  FlatList,
  StyleSheet,
  ActivityIndicator,
  Text,
  RefreshControl
} from 'react-native';
import { useQuery, useMutation, useSubscription } from '@apollo/client';
import { GET_FEED, LIKE_POST, NEW_POST_SUBSCRIPTION } from '../graphql';
import PostCard from '../components/PostCard';

const ITEMS_PER_PAGE = 10;

const FeedScreen: React.FC = () => {
  const { data, loading, error, fetchMore, refetch } = useQuery(GET_FEED, {
    variables: { limit: ITEMS_PER_PAGE },
    notifyOnNetworkStatusChange: true
  });
  
  const [likePost] = useMutation(LIKE_POST, {
    update(cache, { data: { likePost } }) {
      // Optimistic update
      cache.modify({
        id: cache.identify({ __typename: 'Post', id: likePost.id }),
        fields: {
          likes: () => likePost.likes,
          isLiked: () => true
        }
      });
    }
  });
  
  // Subscribe to new posts
  useSubscription(NEW_POST_SUBSCRIPTION, {
    onData: ({ client, data }) => {
      // Add new post to beginning of feed
      client.cache.modify({
        fields: {
          feedPosts: (existing = { edges: [] }) => ({
            ...existing,
            edges: [
              { node: data.data.newPost, cursor: data.data.newPost.id, __typename: 'PostEdge' },
              ...existing.edges
            ]
          })
        }
      });
    }
  });
  
  const loadMore = useCallback(() => {
    const pageInfo = data?.feedPosts?.pageInfo;
    if (!pageInfo?.hasNextPage) return;
    
    fetchMore({
      variables: {
        limit: ITEMS_PER_PAGE,
        after: pageInfo.endCursor
      }
    });
  }, [data, fetchMore]);
  
  const handleLike = useCallback((postId: string) => {
    likePost({ variables: { id: postId } });
  }, [likePost]);
  
  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.error}>เกิดข้อผิดพลาด: {error.message}</Text>
      </View>
    );
  }
  
  const posts = data?.feedPosts?.edges?.map((edge: any) => edge.node) ?? [];
  
  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <PostCard
          post={item}
          onLike={() => handleLike(item.id)}
        />
      )}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      refreshControl={
        <RefreshControl
          refreshing={loading && !data}
          onRefresh={() => refetch()}
        />
      }
      ListFooterComponent={
        loading && data ? <ActivityIndicator style={styles.loader} /> : null
      }
      ListEmptyComponent={
        !loading ? (
          <View style={styles.center}>
            <Text style={styles.emptyText}>ยังไม่มีโพสต์</Text>
          </View>
        ) : null
      }
    />
  );
};

const styles = StyleSheet.create({
  center: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20
  },
  error: { color: '#FF3B30', textAlign: 'center' },
  loader: { marginVertical: 16 },
  emptyText: { fontSize: 16, color: '#666' }
});

export default FeedScreen;
```

### Step 3: Create Post Screen

```tsx
// screens/CreatePostScreen.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ScrollView,
  StyleSheet,
  Alert,
  KeyboardAvoidingView,
  Platform
} from 'react-native';
import { useMutation } from '@apollo/client';
import { CREATE_POST, GET_FEED } from '../graphql';

const CreatePostScreen: React.FC<{ navigation: any }> = ({ navigation }) => {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [tags, setTags] = useState('');
  
  const [createPost, { loading }] = useMutation(CREATE_POST, {
    // Update feed cache after creating post
    update(cache, { data: { createPost } }) {
      const existingData = cache.readQuery({ query: GET_FEED, variables: { limit: 10 } }) as any;
      
      if (existingData) {
        cache.writeQuery({
          query: GET_FEED,
          variables: { limit: 10 },
          data: {
            feedPosts: {
              ...existingData.feedPosts,
              edges: [
                { node: createPost, cursor: createPost.id, __typename: 'PostEdge' },
                ...existingData.feedPosts.edges
              ]
            }
          }
        });
      }
    },
    onCompleted: () => {
      Alert.alert('สำเร็จ', 'สร้างโพสต์แล้ว');
      navigation.goBack();
    },
    onError: (error) => {
      Alert.alert('ผิดพลาด', error.message);
    }
  });
  
  const handleSubmit = useCallback(() => {
    if (!title.trim() || !content.trim()) {
      Alert.alert('ผิดพลาด', 'กรุณากรอกหัวข้อและเนื้อหา');
      return;
    }
    
    createPost({
      variables: {
        input: {
          title: title.trim(),
          content: content.trim(),
          tags: tags.split(',').map(t => t.trim()).filter(Boolean)
        }
      }
    });
  }, [title, content, tags, createPost]);
  
  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView style={styles.scroll}>
        <Text style={styles.label}>หัวข้อ *</Text>
        <TextInput
          style={styles.input}
          value={title}
          onChangeText={setTitle}
          placeholder="หัวข้อโพสต์"
          maxLength={100}
        />
        
        <Text style={styles.label}>เนื้อหา *</Text>
        <TextInput
          style={[styles.input, styles.contentInput]}
          value={content}
          onChangeText={setContent}
          placeholder="เขียนเนื้อหาโพสต์ของคุณ..."
          multiline
          maxLength={5000}
        />
        
        <Text style={styles.label}>แท็ก (คั่นด้วย comma)</Text>
        <TextInput
          style={styles.input}
          value={tags}
          onChangeText={setTags}
          placeholder="react-native, mobile, development"
        />
        
        <TouchableOpacity
          style={[styles.button, loading && styles.buttonDisabled]}
          onPress={handleSubmit}
          disabled={loading}
        >
          <Text style={styles.buttonText}>
            {loading ? 'กำลังสร้าง...' : 'สร้างโพสต์'}
          </Text>
        </TouchableOpacity>
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  scroll: { flex: 1, padding: 16 },
  label: {
    fontSize: 14,
    fontWeight: '600',
    color: '#333',
    marginBottom: 6,
    marginTop: 12
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    backgroundColor: '#fff'
  },
  contentInput: { minHeight: 150, textAlignVertical: 'top' },
  button: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 24,
    marginBottom: 40
  },
  buttonDisabled: { backgroundColor: '#ccc' },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' }
});

export default CreatePostScreen;
```

---

## Tips สำหรับ GraphQL

1. **Use fragments** - ลด code duplication
2. **Proper caching** - configure InMemoryCache อย่างระมัดระวัง
3. **Error handling** - handle both network และ GraphQL errors
4. **Optimistic updates** - ทำให้ UI รู้สึกเร็วขึ้น
5. **Subscriptions sparingly** - ใช้เมื่อ real-time จำเป็นจริงๆ

## สรุป

GraphQL ช่วยให้:
1. ดึงข้อมูลได้แม่นยำ ไม่ over-fetch หรือ under-fetch
2. Type safety ด้วย schema
3. Real-time updates ด้วย subscriptions
4. Better developer experience
5. Easier API versioning
