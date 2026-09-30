# Part 013: Tab Navigator

## สารบัญ
1. createBottomTabNavigator
2. Tab Icons และ Labels
3. Tab Screen Options
4. Badge Notifications
5. Custom Tab Bar
6. Workshop: Social Media App Layout

---

## 1. createBottomTabNavigator

Tab Navigator แสดง navigation tabs ที่ด้านล่างของหน้าจอ เหมาะสำหรับ app ที่มี 3-5 section หลัก

### การติดตั้ง

```bash
npm install @react-navigation/bottom-tabs
```

### การใช้งานพื้นฐาน

```javascript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function HomeScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>หน้าหลัก</Text>
    </View>
  );
}

function SearchScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>ค้นหา</Text>
    </View>
  );
}

function ProfileScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>โปรไฟล์</Text>
    </View>
  );
}

export default function App() {
  return (
    <NavigationContainer>
      <Tab.Navigator>
        <Tab.Screen name="Home" component={HomeScreen} />
        <Tab.Screen name="Search" component={SearchScreen} />
        <Tab.Screen name="Profile" component={ProfileScreen} />
      </Tab.Navigator>
    </NavigationContainer>
  );
}
```

---

## 2. Tab Icons และ Labels

### ใช้ react-native-vector-icons

```bash
# ติดตั้ง icons
npm install @expo/vector-icons
# หรือ
npm install react-native-vector-icons
```

```javascript
import { Ionicons } from '@expo/vector-icons';

<Tab.Navigator
  screenOptions={({ route }) => ({
    // กำหนด icon ตาม route name
    tabBarIcon: ({ focused, color, size }) => {
      let iconName;

      if (route.name === 'Home') {
        iconName = focused ? 'home' : 'home-outline';
      } else if (route.name === 'Search') {
        iconName = focused ? 'search' : 'search-outline';
      } else if (route.name === 'Notifications') {
        iconName = focused ? 'notifications' : 'notifications-outline';
      } else if (route.name === 'Profile') {
        iconName = focused ? 'person' : 'person-outline';
      }

      return <Ionicons name={iconName} size={size} color={color} />;
    },
    
    // สีเมื่อ active/inactive
    tabBarActiveTintColor: '#6200EE',
    tabBarInactiveTintColor: '#999',
  })}
>
  <Tab.Screen name="Home" component={HomeScreen} />
  <Tab.Screen name="Search" component={SearchScreen} />
  <Tab.Screen name="Notifications" component={NotificationsScreen} />
  <Tab.Screen name="Profile" component={ProfileScreen} />
</Tab.Navigator>
```

### แยก options ในแต่ละ Screen

```javascript
<Tab.Screen
  name="Home"
  component={HomeScreen}
  options={{
    title: 'หน้าหลัก',
    tabBarLabel: 'หน้าหลัก',
    tabBarIcon: ({ focused, color, size }) => (
      <Ionicons 
        name={focused ? 'home' : 'home-outline'} 
        size={size} 
        color={color} 
      />
    ),
    tabBarActiveTintColor: '#6200EE',
  }}
/>
```

### ซ่อน Label

```javascript
// ซ่อน label ทุก tab
<Tab.Navigator
  screenOptions={{
    tabBarShowLabel: false,
    tabBarIconStyle: { marginTop: 6 },
  }}
>
```

### Custom Label Component

```javascript
<Tab.Navigator
  screenOptions={({ route }) => ({
    tabBarLabel: ({ focused, color }) => (
      <Text style={{ 
        color,
        fontSize: focused ? 13 : 11,
        fontWeight: focused ? 'bold' : 'normal',
      }}>
        {route.name}
      </Text>
    ),
  })}
>
```

---

## 3. Tab Screen Options

### Tab Bar Style

```javascript
<Tab.Navigator
  screenOptions={{
    // ===== Tab Bar Style =====
    tabBarStyle: {
      backgroundColor: '#FFF',
      borderTopWidth: 0,
      elevation: 10,
      shadowColor: '#000',
      shadowOffset: { width: 0, height: -2 },
      shadowOpacity: 0.1,
      shadowRadius: 4,
      height: 60,
      paddingBottom: 8,
      paddingTop: 4,
    },
    
    // ===== Tab Bar Colors =====
    tabBarActiveTintColor: '#6200EE',
    tabBarInactiveTintColor: '#BDBDBD',
    
    // ===== Tab Bar Background =====
    tabBarBackground: () => (
      <BlurView intensity={100} style={StyleSheet.absoluteFill} />
    ),
    
    // ===== Header =====
    headerShown: true,
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#FFF',
    
    // ===== Tab Bar Visibility =====
    tabBarHideOnKeyboard: true,  // ซ่อนเมื่อ keyboard เปิด
    
    // ===== Position =====
    tabBarPosition: 'bottom',  // 'bottom' (default)
  }}
>
```

### ซ่อน Tab Bar ใน Screen เฉพาะ

```javascript
// ซ่อน tab bar เมื่ออยู่ใน specific screen
// วิธีที่ 1: กำหนดใน screen options
<Tab.Screen
  name="Camera"
  component={CameraScreen}
  options={{ tabBarStyle: { display: 'none' } }}
/>

// วิธีที่ 2: Dynamic (ตาม route)
<Tab.Navigator
  screenOptions={({ route }) => ({
    tabBarStyle: route.name === 'Camera' 
      ? { display: 'none' }
      : { backgroundColor: '#FFF' },
  })}
>
```

---

## 4. Badge Notifications

### Basic Badge

```javascript
<Tab.Screen
  name="Notifications"
  component={NotificationsScreen}
  options={{
    tabBarBadge: 5,          // แสดง badge ตัวเลข
    tabBarBadgeStyle: {
      backgroundColor: '#E53935',
      color: '#FFF',
      fontSize: 10,
    },
  }}
/>
```

### Dynamic Badge จาก State

```javascript
import React, { useState, useEffect } from 'react';

function AppNavigator() {
  const [unreadCount, setUnreadCount] = useState(0);
  const [messageCount, setMessageCount] = useState(0);

  useEffect(() => {
    // Simulate fetching notification count
    const interval = setInterval(() => {
      setUnreadCount(Math.floor(Math.random() * 20));
      setMessageCount(Math.floor(Math.random() * 10));
    }, 5000);
    
    return () => clearInterval(interval);
  }, []);

  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen
        name="Messages"
        component={MessagesScreen}
        options={{
          tabBarBadge: messageCount > 0 ? messageCount : undefined,
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons
              name={focused ? 'chatbubble' : 'chatbubble-outline'}
              size={size}
              color={color}
            />
          ),
        }}
      />
      <Tab.Screen
        name="Notifications"
        component={NotificationsScreen}
        options={{
          tabBarBadge: unreadCount > 99 ? '99+' : unreadCount > 0 ? unreadCount : undefined,
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons
              name={focused ? 'notifications' : 'notifications-outline'}
              size={size}
              color={color}
            />
          ),
        }}
      />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

### Badge จาก Redux Store

```javascript
import { useSelector } from 'react-redux';

function AppNavigator() {
  const notificationCount = useSelector(state => state.notifications.unreadCount);
  
  return (
    <Tab.Navigator>
      <Tab.Screen
        name="Notifications"
        component={NotificationsScreen}
        options={{
          tabBarBadge: notificationCount > 0 ? notificationCount : undefined,
        }}
      />
    </Tab.Navigator>
  );
}
```

---

## 5. Custom Tab Bar

### สร้าง Custom Tab Bar ตั้งแต่ต้น

```javascript
import React from 'react';
import { 
  View, Text, TouchableOpacity, StyleSheet, 
  Platform, Dimensions 
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';

const { width } = Dimensions.get('window');

function CustomTabBar({ state, descriptors, navigation }) {
  const tabs = [
    { name: 'Home', icon: 'home', label: 'หน้าหลัก' },
    { name: 'Explore', icon: 'compass', label: 'สำรวจ' },
    { name: 'Post', icon: 'add-circle', label: '' },  // Special center button
    { name: 'Messages', icon: 'chatbubble', label: 'ข้อความ' },
    { name: 'Profile', icon: 'person', label: 'โปรไฟล์' },
  ];

  return (
    <View style={styles.container}>
      {state.routes.map((route, index) => {
        const { options } = descriptors[route.key];
        const isFocused = state.index === index;
        const tab = tabs[index];

        const onPress = () => {
          const event = navigation.emit({
            type: 'tabPress',
            target: route.key,
            canPreventDefault: true,
          });

          if (!isFocused && !event.defaultPrevented) {
            navigation.navigate(route.name);
          }
        };

        // Special center button (Post)
        if (route.name === 'Post') {
          return (
            <TouchableOpacity
              key={route.key}
              style={styles.centerButton}
              onPress={onPress}
            >
              <View style={styles.centerButtonInner}>
                <Ionicons name="add" size={30} color="#FFF" />
              </View>
            </TouchableOpacity>
          );
        }

        return (
          <TouchableOpacity
            key={route.key}
            style={styles.tabItem}
            onPress={onPress}
            activeOpacity={0.7}
          >
            <Ionicons
              name={isFocused ? tab.icon : `${tab.icon}-outline`}
              size={24}
              color={isFocused ? '#6200EE' : '#999'}
            />
            {tab.label !== '' && (
              <Text style={[
                styles.label,
                { color: isFocused ? '#6200EE' : '#999' }
              ]}>
                {tab.label}
              </Text>
            )}
            {/* Active indicator */}
            {isFocused && tab.label !== '' && (
              <View style={styles.activeIndicator} />
            )}
          </TouchableOpacity>
        );
      })}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    backgroundColor: '#FFF',
    borderTopWidth: 1,
    borderTopColor: '#F0F0F0',
    height: Platform.OS === 'ios' ? 80 : 60,
    paddingBottom: Platform.OS === 'ios' ? 20 : 0,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -3 },
    shadowOpacity: 0.1,
    shadowRadius: 5,
    elevation: 10,
  },
  tabItem: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingTop: 8,
  },
  label: {
    fontSize: 10,
    marginTop: 2,
    fontWeight: '500',
  },
  activeIndicator: {
    position: 'absolute',
    top: 0,
    width: 20,
    height: 3,
    backgroundColor: '#6200EE',
    borderRadius: 2,
  },
  centerButton: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    marginTop: -20,
  },
  centerButtonInner: {
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#6200EE',
    alignItems: 'center',
    justifyContent: 'center',
    shadowColor: '#6200EE',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.4,
    shadowRadius: 8,
    elevation: 8,
  },
});

// ใช้งาน Custom Tab Bar
<Tab.Navigator tabBar={(props) => <CustomTabBar {...props} />}>
  <Tab.Screen name="Home" component={HomeScreen} options={{ headerShown: false }} />
  <Tab.Screen name="Explore" component={ExploreScreen} options={{ headerShown: false }} />
  <Tab.Screen name="Post" component={PostScreen} options={{ headerShown: false }} />
  <Tab.Screen name="Messages" component={MessagesScreen} options={{ headerShown: false }} />
  <Tab.Screen name="Profile" component={ProfileScreen} options={{ headerShown: false }} />
</Tab.Navigator>
```

### Animated Tab Bar

```javascript
import Animated, { 
  useAnimatedStyle, 
  withTiming, 
  useSharedValue 
} from 'react-native-reanimated';

function AnimatedTabBar({ state, descriptors, navigation }) {
  return (
    <View style={styles.container}>
      {state.routes.map((route, index) => {
        const isFocused = state.index === index;
        
        return (
          <AnimatedTabItem
            key={route.key}
            isFocused={isFocused}
            route={route}
            onPress={() => navigation.navigate(route.name)}
          />
        );
      })}
    </View>
  );
}

function AnimatedTabItem({ isFocused, route, onPress }) {
  const scale = useSharedValue(isFocused ? 1.2 : 1);
  
  useEffect(() => {
    scale.value = withTiming(isFocused ? 1.2 : 1, { duration: 200 });
  }, [isFocused]);
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <TouchableOpacity style={styles.tabItem} onPress={onPress}>
      <Animated.View style={animatedStyle}>
        <Ionicons
          name={getIconName(route.name, isFocused)}
          size={24}
          color={isFocused ? '#6200EE' : '#999'}
        />
      </Animated.View>
    </TouchableOpacity>
  );
}
```

---

## Workshop: Social Media App Layout

### โครงสร้างแอป

```
SocialApp/
├── App.js
├── navigation/
│   ├── AppNavigator.js
│   ├── HomeStack.js
│   ├── SearchStack.js
│   └── ProfileStack.js
├── screens/
│   ├── home/
│   │   ├── FeedScreen.js
│   │   └── PostDetailScreen.js
│   ├── search/
│   │   └── SearchScreen.js
│   ├── post/
│   │   └── CreatePostScreen.js
│   ├── notifications/
│   │   └── NotificationsScreen.js
│   └── profile/
│       ├── ProfileScreen.js
│       └── EditProfileScreen.js
└── components/
    ├── PostCard.js
    ├── StoryList.js
    └── CustomTabBar.js
```

### App.js

```javascript
// App.js
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import AppNavigator from './navigation/AppNavigator';
import { NotificationProvider } from './context/NotificationContext';

export default function App() {
  return (
    <NotificationProvider>
      <NavigationContainer>
        <AppNavigator />
      </NavigationContainer>
    </NotificationProvider>
  );
}
```

### Notification Context

```javascript
// context/NotificationContext.js
import React, { createContext, useContext, useState } from 'react';

const NotificationContext = createContext();

export function NotificationProvider({ children }) {
  const [notifications, setNotifications] = useState([
    { id: 1, text: 'สมศรี ถูกใจโพสต์ของคุณ', read: false, time: '2 นาที' },
    { id: 2, text: 'มีผู้ติดตามใหม่ 5 คน', read: false, time: '10 นาที' },
    { id: 3, text: 'สมชาย คอมเมนต์รูปของคุณ', read: false, time: '1 ชม.' },
  ]);

  const unreadCount = notifications.filter(n => !n.read).length;

  const markAsRead = (id) => {
    setNotifications(prev =>
      prev.map(n => n.id === id ? { ...n, read: true } : n)
    );
  };

  const markAllAsRead = () => {
    setNotifications(prev => prev.map(n => ({ ...n, read: true })));
  };

  return (
    <NotificationContext.Provider value={{
      notifications,
      unreadCount,
      markAsRead,
      markAllAsRead,
    }}>
      {children}
    </NotificationContext.Provider>
  );
}

export const useNotifications = () => useContext(NotificationContext);
```

### AppNavigator

```javascript
// navigation/AppNavigator.js
import React from 'react';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { Ionicons } from '@expo/vector-icons';
import HomeStack from './HomeStack';
import SearchStack from './SearchStack';
import CreatePostScreen from '../screens/post/CreatePostScreen';
import NotificationsScreen from '../screens/notifications/NotificationsScreen';
import ProfileStack from './ProfileStack';
import CustomTabBar from '../components/CustomTabBar';
import { useNotifications } from '../context/NotificationContext';

const Tab = createBottomTabNavigator();

export default function AppNavigator() {
  const { unreadCount } = useNotifications();

  return (
    <Tab.Navigator
      tabBar={(props) => <CustomTabBar {...props} />}
      screenOptions={{ headerShown: false }}
    >
      <Tab.Screen 
        name="HomeStack" 
        component={HomeStack}
        options={{
          tabBarLabel: 'หน้าหลัก',
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons name={focused ? 'home' : 'home-outline'} size={size} color={color} />
          ),
        }}
      />
      <Tab.Screen 
        name="SearchStack" 
        component={SearchStack}
        options={{
          tabBarLabel: 'ค้นหา',
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons name={focused ? 'search' : 'search-outline'} size={size} color={color} />
          ),
        }}
      />
      <Tab.Screen 
        name="CreatePost" 
        component={CreatePostScreen}
        options={{
          tabBarLabel: '',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="add-circle" size={size + 10} color="#6200EE" />
          ),
        }}
      />
      <Tab.Screen 
        name="Notifications" 
        component={NotificationsScreen}
        options={{
          tabBarLabel: 'แจ้งเตือน',
          tabBarBadge: unreadCount > 0 ? unreadCount : undefined,
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons 
              name={focused ? 'notifications' : 'notifications-outline'} 
              size={size} 
              color={color} 
            />
          ),
        }}
      />
      <Tab.Screen 
        name="ProfileStack" 
        component={ProfileStack}
        options={{
          tabBarLabel: 'โปรไฟล์',
          tabBarIcon: ({ focused, color, size }) => (
            <Ionicons name={focused ? 'person' : 'person-outline'} size={size} color={color} />
          ),
        }}
      />
    </Tab.Navigator>
  );
}
```

### HomeStack Navigator

```javascript
// navigation/HomeStack.js
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import FeedScreen from '../screens/home/FeedScreen';
import PostDetailScreen from '../screens/home/PostDetailScreen';
import UserProfileScreen from '../screens/profile/UserProfileScreen';

const Stack = createNativeStackNavigator();

export default function HomeStack() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerStyle: { backgroundColor: '#FFF' },
        headerTintColor: '#212121',
        headerShadowVisible: false,
      }}
    >
      <Stack.Screen 
        name="Feed" 
        component={FeedScreen}
        options={{ title: 'SocialApp' }}
      />
      <Stack.Screen 
        name="PostDetail" 
        component={PostDetailScreen}
        options={{ title: 'โพสต์' }}
      />
      <Stack.Screen 
        name="UserProfile" 
        component={UserProfileScreen}
        options={({ route }) => ({ title: route.params.username })}
      />
    </Stack.Navigator>
  );
}
```

### FeedScreen

```javascript
// screens/home/FeedScreen.js
import React, { useState } from 'react';
import {
  View, Text, StyleSheet, FlatList,
  Image, TouchableOpacity, RefreshControl
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';

const MOCK_POSTS = [
  {
    id: '1',
    user: { name: 'สมชาย ใจดี', username: 'somchai', avatar: 'https://picsum.photos/id/10/50/50' },
    image: 'https://picsum.photos/id/100/400/300',
    caption: 'วันนี้อากาศดีมาก ออกไปเดินเล่นกัน 🌤️',
    likes: 234,
    comments: 12,
    time: '2 ชม. ที่แล้ว',
    liked: false,
    saved: false,
  },
  {
    id: '2',
    user: { name: 'สมศรี สุขใส', username: 'somsri', avatar: 'https://picsum.photos/id/20/50/50' },
    image: 'https://picsum.photos/id/200/400/300',
    caption: 'อาหารเย็นวันนี้ทำเองนะ อร่อยมากๆ 🍜',
    likes: 567,
    comments: 45,
    time: '4 ชม. ที่แล้ว',
    liked: true,
    saved: true,
  },
];

function PostCard({ post, onLike, onSave, onPress, onUserPress }) {
  return (
    <View style={styles.card}>
      {/* Header */}
      <TouchableOpacity style={styles.cardHeader} onPress={() => onUserPress(post.user)}>
        <Image source={{ uri: post.user.avatar }} style={styles.avatar} />
        <View>
          <Text style={styles.userName}>{post.user.name}</Text>
          <Text style={styles.time}>{post.time}</Text>
        </View>
        <TouchableOpacity style={styles.moreBtn}>
          <Ionicons name="ellipsis-horizontal" size={20} color="#999" />
        </TouchableOpacity>
      </TouchableOpacity>

      {/* Image */}
      <TouchableOpacity onPress={() => onPress(post)}>
        <Image source={{ uri: post.image }} style={styles.postImage} />
      </TouchableOpacity>

      {/* Actions */}
      <View style={styles.actions}>
        <View style={styles.leftActions}>
          <TouchableOpacity style={styles.actionBtn} onPress={() => onLike(post.id)}>
            <Ionicons
              name={post.liked ? 'heart' : 'heart-outline'}
              size={26}
              color={post.liked ? '#E53935' : '#212121'}
            />
          </TouchableOpacity>
          <TouchableOpacity style={styles.actionBtn} onPress={() => onPress(post)}>
            <Ionicons name="chatbubble-outline" size={24} color="#212121" />
          </TouchableOpacity>
          <TouchableOpacity style={styles.actionBtn}>
            <Ionicons name="paper-plane-outline" size={24} color="#212121" />
          </TouchableOpacity>
        </View>
        <TouchableOpacity onPress={() => onSave(post.id)}>
          <Ionicons
            name={post.saved ? 'bookmark' : 'bookmark-outline'}
            size={24}
            color="#212121"
          />
        </TouchableOpacity>
      </View>

      {/* Likes */}
      <View style={styles.cardContent}>
        <Text style={styles.likes}>{post.likes.toLocaleString()} ถูกใจ</Text>
        <Text style={styles.caption}>
          <Text style={styles.captionUser}>{post.user.username} </Text>
          {post.caption}
        </Text>
        {post.comments > 0 && (
          <TouchableOpacity onPress={() => onPress(post)}>
            <Text style={styles.commentsLink}>
              ดูความคิดเห็นทั้ง {post.comments} ความคิดเห็น
            </Text>
          </TouchableOpacity>
        )}
      </View>
    </View>
  );
}

export default function FeedScreen({ navigation }) {
  const [posts, setPosts] = useState(MOCK_POSTS);
  const [refreshing, setRefreshing] = useState(false);

  const handleLike = (postId) => {
    setPosts(prev => prev.map(p => {
      if (p.id === postId) {
        return {
          ...p,
          liked: !p.liked,
          likes: p.liked ? p.likes - 1 : p.likes + 1,
        };
      }
      return p;
    }));
  };

  const handleSave = (postId) => {
    setPosts(prev => prev.map(p =>
      p.id === postId ? { ...p, saved: !p.saved } : p
    ));
  };

  const handleRefresh = () => {
    setRefreshing(true);
    setTimeout(() => setRefreshing(false), 1500);
  };

  return (
    <FlatList
      data={posts}
      keyExtractor={item => item.id}
      renderItem={({ item }) => (
        <PostCard
          post={item}
          onLike={handleLike}
          onSave={handleSave}
          onPress={(post) => navigation.navigate('PostDetail', { post })}
          onUserPress={(user) => navigation.navigate('UserProfile', { 
            userId: user.username,
            username: user.name 
          })}
        />
      )}
      ListHeaderComponent={() => (
        <View style={styles.header}>
          <Text style={styles.headerTitle}>SocialApp</Text>
          <View style={styles.headerActions}>
            <TouchableOpacity style={styles.headerBtn}>
              <Ionicons name="heart-outline" size={26} color="#212121" />
            </TouchableOpacity>
            <TouchableOpacity style={styles.headerBtn}>
              <Ionicons name="paper-plane-outline" size={26} color="#212121" />
            </TouchableOpacity>
          </View>
        </View>
      )}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      showsVerticalScrollIndicator={false}
    />
  );
}

const styles = StyleSheet.create({
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#F0F0F0',
  },
  headerTitle: { fontSize: 22, fontWeight: 'bold', fontStyle: 'italic' },
  headerActions: { flexDirection: 'row' },
  headerBtn: { marginLeft: 16 },
  card: {
    backgroundColor: '#FFF',
    marginBottom: 1,
  },
  cardHeader: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
  },
  avatar: { width: 40, height: 40, borderRadius: 20, marginRight: 10 },
  userName: { fontSize: 14, fontWeight: 'bold', color: '#212121' },
  time: { fontSize: 12, color: '#999' },
  moreBtn: { marginLeft: 'auto' },
  postImage: { width: '100%', height: 300 },
  actions: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingHorizontal: 12,
    paddingVertical: 8,
  },
  leftActions: { flexDirection: 'row' },
  actionBtn: { marginRight: 12 },
  cardContent: { paddingHorizontal: 12, paddingBottom: 12 },
  likes: { fontWeight: 'bold', marginBottom: 4 },
  caption: { fontSize: 14, color: '#212121', lineHeight: 20 },
  captionUser: { fontWeight: 'bold' },
  commentsLink: { color: '#999', marginTop: 4, fontSize: 14 },
});
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ 3-5 tabs เท่านั้น (มากกว่านี้ UI รก)
- ใช้ icons ที่สื่อความหมายชัดเจน
- แสดง active state อย่างชัดเจน
- ใช้ badge เฉพาะเมื่อจำเป็น

❌ DON'T:
- ไม่ใช้ label ยาวเกินไปใน tab
- ไม่ซ่อน tab bar โดยไม่จำเป็น
- ไม่ใส่ tab ที่ผู้ใช้ไม่ได้ใช้บ่อย

📝 Pattern ที่แนะนำ:
- ใช้ Stack ใน Tab แต่ละ tab สำหรับ navigation ลึก
- เก็บ notification count ใน Context หรือ Redux
- ทำ Custom Tab Bar สำหรับ design พิเศษ
```

---

## สรุป

Tab Navigator เป็นส่วนสำคัญของ app layout ที่เราได้เรียนรู้:

1. **createBottomTabNavigator** - สร้างและใช้งาน tab navigation
2. **Tab Icons และ Labels** - ใช้ icons และปรับแต่ง labels
3. **Screen Options** - ปรับแต่ง tab bar style
4. **Badge Notifications** - แสดง notification count
5. **Custom Tab Bar** - สร้าง tab bar แบบ custom
6. **Workshop** - Social media app layout สมบูรณ์
