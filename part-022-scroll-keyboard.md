# Part 022: ScrollView, KeyboardAvoidingView, SafeAreaView

## บทนำ

การจัดการ layout ใน React Native มีความซับซ้อนกว่า web เนื่องจากต้องคำนึงถึง device หลากหลายรุ่น ขนาดหน้าจอ keyboard ที่ pop up ขึ้นมา และ safe areas ของ notch/home indicator ในบทนี้เราจะเรียนรู้ component สำคัญ 3 ตัวที่ช่วยแก้ปัญหาเหล่านี้

## สารบัญ

1. ScrollView vs FlatList
2. Horizontal ScrollView
3. KeyboardAvoidingView
4. SafeAreaView
5. StatusBar
6. Workshop: Chat Interface

---

## 1. ScrollView vs FlatList

### ScrollView

`ScrollView` เป็น component ที่ render เนื้อหาทั้งหมดในครั้งเดียว เหมาะสำหรับเนื้อหาที่ไม่มากนัก

**เมื่อไหร่ควรใช้ ScrollView:**
- เนื้อหาน้อยกว่า 20-30 items
- ต้องการ scroll ทั้งแนวตั้งและแนวนอนพร้อมกัน
- Layout ซับซ้อนที่ไม่ใช่ list
- Form ที่มีหลาย field

```jsx
import React from 'react';
import {
  ScrollView,
  View,
  Text,
  StyleSheet,
  RefreshControl,
} from 'react-native';

const ScrollViewExample = () => {
  const [refreshing, setRefreshing] = React.useState(false);

  const onRefresh = React.useCallback(() => {
    setRefreshing(true);
    // จำลองการดึงข้อมูล
    setTimeout(() => setRefreshing(false), 2000);
  }, []);

  return (
    <ScrollView
      style={styles.scroll}
      contentContainerStyle={styles.content}
      showsVerticalScrollIndicator={false}
      bounces={true}                    // iOS: bounce effect
      overScrollMode="always"           // Android: over-scroll effect
      keyboardShouldPersistTaps="handled"
      refreshControl={
        <RefreshControl
          refreshing={refreshing}
          onRefresh={onRefresh}
          tintColor="#007AFF"
          title="กำลังโหลด..."
          titleColor="#007AFF"
        />
      }
    >
      {Array.from({ length: 20 }, (_, i) => (
        <View key={i} style={styles.card}>
          <Text style={styles.cardTitle}>รายการที่ {i + 1}</Text>
          <Text style={styles.cardBody}>
            นี่คือเนื้อหาของรายการที่ {i + 1} ซึ่งอาจมีข้อความยาวหลายบรรทัด
          </Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  scroll: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  content: {
    padding: 16,
    gap: 12,
    paddingBottom: 32,
  },
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  cardTitle: {
    fontSize: 16,
    fontWeight: '600',
    marginBottom: 8,
  },
  cardBody: {
    color: '#666',
    lineHeight: 22,
  },
});
```

### FlatList

`FlatList` เป็น virtual list ที่ render เฉพาะ item ที่แสดงบนหน้าจอ เหมาะสำหรับ list ขนาดใหญ่

**เมื่อไหร่ควรใช้ FlatList:**
- มี items มากกว่า 30 รายการ
- ข้อมูลที่ load เพิ่มได้ (infinite scroll)
- Performance เป็นสิ่งสำคัญ

```jsx
import React, { useState, useCallback } from 'react';
import {
  FlatList,
  View,
  Text,
  ActivityIndicator,
  StyleSheet,
} from 'react-native';

const FlatListExample = () => {
  const [data, setData] = useState(
    Array.from({ length: 20 }, (_, i) => ({ id: String(i), title: `Item ${i + 1}` }))
  );
  const [loading, setLoading] = useState(false);
  const [refreshing, setRefreshing] = useState(false);
  const [page, setPage] = useState(1);

  // Load more data (infinite scroll)
  const loadMore = useCallback(() => {
    if (loading) return;
    setLoading(true);
    setTimeout(() => {
      const newItems = Array.from({ length: 10 }, (_, i) => ({
        id: String(data.length + i),
        title: `Item ${data.length + i + 1}`,
      }));
      setData((prev) => [...prev, ...newItems]);
      setPage((prev) => prev + 1);
      setLoading(false);
    }, 1500);
  }, [loading, data.length]);

  const onRefresh = useCallback(() => {
    setRefreshing(true);
    setTimeout(() => {
      setData(
        Array.from({ length: 20 }, (_, i) => ({ id: String(i), title: `Item ${i + 1}` }))
      );
      setRefreshing(false);
    }, 1500);
  }, []);

  const renderItem = useCallback(({ item, index }) => (
    <View style={styles.item}>
      <View style={styles.avatar}>
        <Text style={styles.avatarText}>{index + 1}</Text>
      </View>
      <Text style={styles.itemTitle}>{item.title}</Text>
    </View>
  ), []);

  const keyExtractor = useCallback((item) => item.id, []);

  const ListFooter = () => (
    loading ? (
      <View style={styles.footer}>
        <ActivityIndicator color="#007AFF" />
        <Text style={styles.footerText}>กำลังโหลด...</Text>
      </View>
    ) : null
  );

  const ListEmpty = () => (
    <View style={styles.empty}>
      <Text>ไม่มีข้อมูล</Text>
    </View>
  );

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      ListFooterComponent={ListFooter}
      ListEmptyComponent={ListEmpty}
      refreshing={refreshing}
      onRefresh={onRefresh}
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={10}
      initialNumToRender={15}
    />
  );
};

const styles = StyleSheet.create({
  item: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  avatar: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  avatarText: { color: '#fff', fontWeight: 'bold' },
  itemTitle: { fontSize: 16, color: '#333' },
  footer: {
    padding: 20,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 8,
  },
  footerText: { color: '#666' },
  empty: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 40 },
});
```

### ตารางเปรียบเทียบ ScrollView vs FlatList

| Feature | ScrollView | FlatList |
|---------|-----------|---------|
| Render | ทั้งหมดพร้อมกัน | เฉพาะที่มองเห็น |
| Performance | ดีกับ items น้อย | ดีกับ items มาก |
| Memory | ใช้มาก | ประหยัด |
| Infinite Scroll | ต้องทำเอง | รองรับในตัว |
| Pull to Refresh | ใช้ RefreshControl | รองรับในตัว |

---

## 2. Horizontal ScrollView

```jsx
import React, { useRef } from 'react';
import {
  View,
  ScrollView,
  Text,
  TouchableOpacity,
  Dimensions,
  StyleSheet,
} from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

// Horizontal Cards (เช่น Categories)
const HorizontalCards = () => {
  const categories = [
    { id: '1', icon: '📱', name: 'มือถือ' },
    { id: '2', icon: '💻', name: 'คอมพิวเตอร์' },
    { id: '3', icon: '🎧', name: 'หูฟัง' },
    { id: '4', icon: '⌚', name: 'นาฬิกา' },
    { id: '5', icon: '📷', name: 'กล้อง' },
    { id: '6', icon: '🎮', name: 'เกมส์' },
  ];

  return (
    <ScrollView
      horizontal
      showsHorizontalScrollIndicator={false}
      contentContainerStyle={styles.horizontal}
    >
      {categories.map((cat) => (
        <TouchableOpacity key={cat.id} style={styles.catCard}>
          <Text style={styles.catIcon}>{cat.icon}</Text>
          <Text style={styles.catName}>{cat.name}</Text>
        </TouchableOpacity>
      ))}
    </ScrollView>
  );
};

// Image Carousel (Snap Scrolling)
const ImageCarousel = () => {
  const scrollRef = useRef(null);
  const [currentIndex, setCurrentIndex] = React.useState(0);

  const images = [
    { id: '1', bg: '#FF6B6B', text: 'โปรโมชั่น 1' },
    { id: '2', bg: '#4ECDC4', text: 'โปรโมชั่น 2' },
    { id: '3', bg: '#45B7D1', text: 'โปรโมชั่น 3' },
    { id: '4', bg: '#96CEB4', text: 'โปรโมชั่น 4' },
  ];

  const handleScroll = (event) => {
    const index = Math.round(
      event.nativeEvent.contentOffset.x / SCREEN_WIDTH
    );
    setCurrentIndex(index);
  };

  return (
    <View>
      <ScrollView
        ref={scrollRef}
        horizontal
        pagingEnabled            // snap to page
        showsHorizontalScrollIndicator={false}
        onScroll={handleScroll}
        scrollEventThrottle={16}
      >
        {images.map((img) => (
          <View
            key={img.id}
            style={[styles.slide, { width: SCREEN_WIDTH, backgroundColor: img.bg }]}
          >
            <Text style={styles.slideText}>{img.text}</Text>
          </View>
        ))}
      </ScrollView>

      {/* Dots indicator */}
      <View style={styles.dots}>
        {images.map((_, index) => (
          <View
            key={index}
            style={[
              styles.dot,
              index === currentIndex && styles.activeDot,
            ]}
          />
        ))}
      </View>
    </View>
  );
};

// Tabbar with ScrollView
const ScrollableTabs = ({ tabs, activeTab, onTabPress }) => {
  const scrollRef = useRef(null);

  return (
    <ScrollView
      ref={scrollRef}
      horizontal
      showsHorizontalScrollIndicator={false}
      contentContainerStyle={styles.tabsContainer}
    >
      {tabs.map((tab, index) => (
        <TouchableOpacity
          key={tab.id}
          style={[styles.tab, activeTab === tab.id && styles.activeTab]}
          onPress={() => onTabPress(tab.id)}
        >
          <Text
            style={[
              styles.tabText,
              activeTab === tab.id && styles.activeTabText,
            ]}
          >
            {tab.label}
          </Text>
        </TouchableOpacity>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  horizontal: {
    paddingHorizontal: 16,
    gap: 12,
    paddingVertical: 8,
  },
  catCard: {
    alignItems: 'center',
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    width: 80,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 2,
  },
  catIcon: { fontSize: 28, marginBottom: 8 },
  catName: { fontSize: 12, color: '#333' },
  slide: {
    height: 180,
    justifyContent: 'center',
    alignItems: 'center',
  },
  slideText: { fontSize: 24, fontWeight: 'bold', color: '#fff' },
  dots: {
    flexDirection: 'row',
    justifyContent: 'center',
    gap: 6,
    marginTop: 12,
  },
  dot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#ddd',
  },
  activeDot: { backgroundColor: '#007AFF', width: 20 },
  tabsContainer: {
    paddingHorizontal: 16,
    gap: 8,
  },
  tab: {
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 20,
    backgroundColor: '#f0f0f0',
  },
  activeTab: { backgroundColor: '#007AFF' },
  tabText: { color: '#666', fontSize: 14 },
  activeTabText: { color: '#fff', fontWeight: '600' },
});
```

---

## 3. KeyboardAvoidingView

`KeyboardAvoidingView` ช่วยป้องกันไม่ให้ keyboard บัง input fields

### Props หลัก

| Prop | ค่า | ความหมาย |
|------|-----|-----------|
| behavior | 'height' \| 'padding' \| 'position' | วิธีการ adjust |
| keyboardVerticalOffset | number | ระยะห่างจาก keyboard |
| enabled | boolean | เปิด/ปิดการทำงาน |

```jsx
import React, { useRef } from 'react';
import {
  View,
  TextInput,
  KeyboardAvoidingView,
  Platform,
  TouchableWithoutFeedback,
  Keyboard,
  ScrollView,
  StyleSheet,
} from 'react-native';

// Pattern พื้นฐาน
const BasicKAV = () => {
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 60 : 0}
    >
      <TouchableWithoutFeedback onPress={Keyboard.dismiss}>
        <View style={{ flex: 1, justifyContent: 'flex-end', padding: 20 }}>
          <TextInput
            style={styles.input}
            placeholder="พิมพ์ข้อความ..."
          />
        </View>
      </TouchableWithoutFeedback>
    </KeyboardAvoidingView>
  );
};

// Form ที่มีหลาย TextInput
const LoginForm = () => {
  const passwordRef = useRef(null);

  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <ScrollView
        contentContainerStyle={{ flexGrow: 1 }}
        keyboardShouldPersistTaps="handled"
      >
        <View style={styles.formContainer}>
          <Text style={styles.formTitle}>เข้าสู่ระบบ</Text>

          <View style={styles.inputGroup}>
            <Text style={styles.label}>อีเมล</Text>
            <TextInput
              style={styles.input}
              placeholder="email@example.com"
              keyboardType="email-address"
              autoCapitalize="none"
              returnKeyType="next"
              onSubmitEditing={() => passwordRef.current?.focus()}
            />
          </View>

          <View style={styles.inputGroup}>
            <Text style={styles.label}>รหัสผ่าน</Text>
            <TextInput
              ref={passwordRef}
              style={styles.input}
              placeholder="รหัสผ่าน"
              secureTextEntry
              returnKeyType="done"
              onSubmitEditing={Keyboard.dismiss}
            />
          </View>

          <TouchableOpacity style={styles.submitBtn}>
            <Text style={styles.submitText}>เข้าสู่ระบบ</Text>
          </TouchableOpacity>
        </View>
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  formContainer: {
    flex: 1,
    padding: 24,
    justifyContent: 'center',
  },
  formTitle: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 32,
  },
  inputGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', color: '#333', marginBottom: 8 },
  input: {
    borderWidth: 1,
    borderColor: '#e0e0e0',
    borderRadius: 10,
    padding: 14,
    fontSize: 16,
    backgroundColor: '#fff',
  },
  submitBtn: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 8,
  },
  submitText: { color: '#fff', fontSize: 18, fontWeight: '600' },
});
```

### KeyboardAvoidingView สำหรับ Chat

```jsx
const ChatInput = () => {
  const [message, setMessage] = useState('');

  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}
    >
      <View style={chatStyles.inputContainer}>
        <TextInput
          style={chatStyles.chatInput}
          placeholder="พิมพ์ข้อความ..."
          value={message}
          onChangeText={setMessage}
          multiline
          maxLength={500}
        />
        <TouchableOpacity
          style={[chatStyles.sendBtn, !message && chatStyles.sendBtnDisabled]}
          disabled={!message.trim()}
          onPress={() => {
            console.log('Send:', message);
            setMessage('');
          }}
        >
          <Text style={chatStyles.sendIcon}>➤</Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
};

const chatStyles = StyleSheet.create({
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    padding: 12,
    backgroundColor: '#fff',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    gap: 8,
  },
  chatInput: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#e0e0e0',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    maxHeight: 100,
    fontSize: 16,
  },
  sendBtn: {
    width: 40,
    height: 40,
    borderRadius: 20,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  sendBtnDisabled: { backgroundColor: '#ccc' },
  sendIcon: { color: '#fff', fontSize: 18 },
});
```

---

## 4. SafeAreaView

`SafeAreaView` ช่วยจัดการ safe areas บนอุปกรณ์ที่มี notch, Dynamic Island หรือ home indicator

### ติดตั้ง react-native-safe-area-context (แนะนำ)

```bash
npx expo install react-native-safe-area-context
```

```jsx
import React from 'react';
import {
  View,
  Text,
  StyleSheet,
  Platform,
} from 'react-native';
import { SafeAreaView, SafeAreaProvider, useSafeAreaInsets } from 'react-native-safe-area-context';

// Pattern 1: ใช้ SafeAreaView โดยตรง
const SafeAreaExample = () => {
  return (
    <SafeAreaProvider>
      <SafeAreaView style={{ flex: 1, backgroundColor: '#fff' }} edges={['top', 'bottom']}>
        <View style={{ padding: 16 }}>
          <Text>เนื้อหาจะอยู่ใน safe area</Text>
        </View>
      </SafeAreaView>
    </SafeAreaProvider>
  );
};

// Pattern 2: ใช้ useSafeAreaInsets hook (แนะนำสำหรับ custom layouts)
const CustomSafeArea = () => {
  const insets = useSafeAreaInsets();

  return (
    <View
      style={{
        flex: 1,
        paddingTop: insets.top,
        paddingBottom: insets.bottom,
        paddingLeft: insets.left,
        paddingRight: insets.right,
      }}
    >
      {/* Header ที่รับรู้ safe area */}
      <View
        style={{
          height: 44 + insets.top,
          paddingTop: insets.top,
          backgroundColor: '#007AFF',
          justifyContent: 'center',
          alignItems: 'center',
        }}
      >
        <Text style={{ color: '#fff', fontSize: 18, fontWeight: 'bold' }}>
          My App
        </Text>
      </View>

      {/* Content */}
      <View style={{ flex: 1, padding: 16 }}>
        <Text>เนื้อหา</Text>
      </View>

      {/* Footer / Tab Bar */}
      <View
        style={{
          height: 60 + insets.bottom,
          paddingBottom: insets.bottom,
          backgroundColor: '#fff',
          borderTopWidth: 1,
          borderTopColor: '#e0e0e0',
          flexDirection: 'row',
        }}
      >
        {/* Tab items */}
      </View>
    </View>
  );
};

// Pattern 3: SafeAreaView เฉพาะส่วน
const PartialSafeArea = () => {
  const insets = useSafeAreaInsets();

  return (
    <View style={{ flex: 1 }}>
      {/* Full-bleed header image */}
      <View style={{ height: 200, backgroundColor: '#007AFF' }}>
        {/* ภาพเต็มๆ */}
      </View>

      {/* Content with safe area padding */}
      <View style={{ flex: 1, paddingBottom: insets.bottom }}>
        <Text style={{ padding: 16 }}>เนื้อหา</Text>
      </View>
    </View>
  );
};
```

---

## 5. StatusBar

`StatusBar` ใช้จัดการ status bar บนของอุปกรณ์

```jsx
import React from 'react';
import { View, StatusBar, Platform } from 'react-native';

// Basic StatusBar
const StatusBarExample = () => {
  return (
    <View style={{ flex: 1 }}>
      <StatusBar
        barStyle="dark-content"     // 'default' | 'light-content' | 'dark-content'
        backgroundColor="#fff"      // Android only
        translucent={false}         // Android: overlay content
        hidden={false}              // ซ่อน status bar
        animated={true}             // animate การเปลี่ยนแปลง
      />
      {/* content */}
    </View>
  );
};

// Status Bar ที่เปลี่ยนตาม screen
const DynamicStatusBar = ({ isDark }) => {
  return (
    <StatusBar
      barStyle={isDark ? 'light-content' : 'dark-content'}
      backgroundColor={isDark ? '#1a1a1a' : '#fff'}
    />
  );
};

// Translucent status bar (Android)
const TranslucentHeader = () => {
  const insets = useSafeAreaInsets();

  return (
    <View style={{ flex: 1 }}>
      <StatusBar translucent backgroundColor="transparent" barStyle="light-content" />
      
      {/* Header ที่ extend ใต้ status bar */}
      <View
        style={{
          height: 200,
          backgroundColor: '#007AFF',
          paddingTop: insets.top,
          justifyContent: 'flex-end',
          padding: 16,
        }}
      >
        <Text style={{ color: '#fff', fontSize: 28, fontWeight: 'bold' }}>
          Header
        </Text>
      </View>
    </View>
  );
};
```

---

## 6. Workshop: Chat Interface

สร้าง Chat Interface ที่สมจริงพร้อม keyboard handling

```jsx
import React, { useState, useRef, useCallback, useEffect } from 'react';
import {
  View,
  Text,
  FlatList,
  TextInput,
  TouchableOpacity,
  KeyboardAvoidingView,
  Platform,
  StyleSheet,
  SafeAreaView,
  StatusBar,
  Image,
  Animated,
} from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

// ข้อมูลตัวอย่าง
const INITIAL_MESSAGES = [
  { id: '1', text: 'สวัสดีครับ!', sender: 'other', time: '10:00', read: true },
  { id: '2', text: 'สวัสดีค่ะ 😊', sender: 'me', time: '10:01', read: true },
  { id: '3', text: 'วันนี้ว่างไหมครับ ไปกินข้าวด้วยกันได้ไหม?', sender: 'other', time: '10:02', read: true },
  { id: '4', text: 'ว่างค่ะ ไปที่ไหนดี?', sender: 'me', time: '10:03', read: true },
  { id: '5', text: 'ร้านอาหารญี่ปุ่นแถวสยามเป็นไงครับ?', sender: 'other', time: '10:04', read: true },
];

// Message Bubble Component
const MessageBubble = ({ message, showAvatar }) => {
  const isMe = message.sender === 'me';
  const fadeAnim = useRef(new Animated.Value(0)).current;
  const slideAnim = useRef(new Animated.Value(20)).current;

  useEffect(() => {
    Animated.parallel([
      Animated.timing(fadeAnim, {
        toValue: 1,
        duration: 200,
        useNativeDriver: true,
      }),
      Animated.timing(slideAnim, {
        toValue: 0,
        duration: 200,
        useNativeDriver: true,
      }),
    ]).start();
  }, []);

  return (
    <Animated.View
      style={[
        styles.messageRow,
        isMe ? styles.myMessageRow : styles.otherMessageRow,
        { opacity: fadeAnim, transform: [{ translateY: slideAnim }] },
      ]}
    >
      {/* Avatar สำหรับข้อความของคนอื่น */}
      {!isMe && (
        <View style={styles.avatarContainer}>
          {showAvatar ? (
            <View style={styles.avatar}>
              <Text style={styles.avatarText}>A</Text>
            </View>
          ) : (
            <View style={{ width: 36 }} />
          )}
        </View>
      )}

      <View style={[styles.bubble, isMe ? styles.myBubble : styles.otherBubble]}>
        <Text style={[styles.messageText, isMe && styles.myMessageText]}>
          {message.text}
        </Text>
        <View style={styles.messageMeta}>
          <Text style={[styles.timeText, isMe && styles.myTimeText]}>
            {message.time}
          </Text>
          {isMe && (
            <Text style={styles.readIcon}>
              {message.read ? '✓✓' : '✓'}
            </Text>
          )}
        </View>
      </View>
    </Animated.View>
  );
};

// Date Separator
const DateSeparator = ({ date }) => (
  <View style={styles.dateSeparator}>
    <View style={styles.dateLine} />
    <Text style={styles.dateText}>{date}</Text>
    <View style={styles.dateLine} />
  </View>
);

// Typing Indicator
const TypingIndicator = () => {
  const dot1 = useRef(new Animated.Value(0)).current;
  const dot2 = useRef(new Animated.Value(0)).current;
  const dot3 = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    const animate = (dot, delay) => {
      Animated.loop(
        Animated.sequence([
          Animated.delay(delay),
          Animated.timing(dot, { toValue: 1, duration: 300, useNativeDriver: true }),
          Animated.timing(dot, { toValue: 0, duration: 300, useNativeDriver: true }),
          Animated.delay(600),
        ])
      ).start();
    };

    animate(dot1, 0);
    animate(dot2, 200);
    animate(dot3, 400);
  }, []);

  return (
    <View style={styles.typingContainer}>
      <View style={styles.typingBubble}>
        {[dot1, dot2, dot3].map((dot, i) => (
          <Animated.View
            key={i}
            style={[styles.typingDot, {
              opacity: dot,
              transform: [{ scale: dot.interpolate({ inputRange: [0, 1], outputRange: [0.7, 1] }) }],
            }]}
          />
        ))}
      </View>
    </View>
  );
};

// Main Chat Screen
const ChatScreen = () => {
  const [messages, setMessages] = useState(INITIAL_MESSAGES);
  const [inputText, setInputText] = useState('');
  const [isTyping, setIsTyping] = useState(false);
  const flatListRef = useRef(null);
  const inputRef = useRef(null);
  const insets = useSafeAreaInsets();

  // จำลองการตอบกลับ
  const simulateReply = (userMessage) => {
    setIsTyping(true);
    setTimeout(() => {
      setIsTyping(false);
      const reply = {
        id: String(Date.now() + 1),
        text: getAutoReply(userMessage),
        sender: 'other',
        time: new Date().toLocaleTimeString('th-TH', { hour: '2-digit', minute: '2-digit' }),
        read: false,
      };
      setMessages((prev) => [...prev, reply]);
    }, 1500 + Math.random() * 1000);
  };

  const getAutoReply = (msg) => {
    const replies = [
      'ได้เลยครับ! 😊',
      'ขอบคุณนะครับ',
      'แน่นอนครับ ไม่มีปัญหา',
      'โอเคครับ เดี๋ยวคุยกันใหม่',
      'เข้าใจแล้วครับ',
    ];
    return replies[Math.floor(Math.random() * replies.length)];
  };

  const sendMessage = useCallback(() => {
    if (!inputText.trim()) return;

    const newMessage = {
      id: String(Date.now()),
      text: inputText.trim(),
      sender: 'me',
      time: new Date().toLocaleTimeString('th-TH', { hour: '2-digit', minute: '2-digit' }),
      read: false,
    };

    setMessages((prev) => [...prev, newMessage]);
    setInputText('');
    simulateReply(inputText);

    // Scroll to bottom
    setTimeout(() => {
      flatListRef.current?.scrollToEnd({ animated: true });
    }, 100);
  }, [inputText]);

  const renderMessage = useCallback(({ item, index }) => {
    const prevMessage = messages[index - 1];
    const showAvatar = !prevMessage || prevMessage.sender !== item.sender;

    return (
      <MessageBubble
        key={item.id}
        message={item}
        showAvatar={showAvatar}
      />
    );
  }, [messages]);

  return (
    <View style={[chatStyles.container, { paddingTop: insets.top }]}>
      <StatusBar barStyle="dark-content" backgroundColor="#fff" />

      {/* Chat Header */}
      <View style={chatStyles.header}>
        <TouchableOpacity style={chatStyles.backBtn}>
          <Text style={{ fontSize: 18, color: '#007AFF' }}>◀</Text>
        </TouchableOpacity>

        <View style={chatStyles.headerInfo}>
          <View style={chatStyles.headerAvatar}>
            <Text style={chatStyles.headerAvatarText}>A</Text>
            <View style={chatStyles.onlineDot} />
          </View>
          <View>
            <Text style={chatStyles.headerName}>Arisa</Text>
            <Text style={chatStyles.headerStatus}>ออนไลน์อยู่</Text>
          </View>
        </View>

        <TouchableOpacity style={chatStyles.callBtn}>
          <Text style={{ fontSize: 20 }}>📞</Text>
        </TouchableOpacity>
      </View>

      {/* Messages */}
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        keyboardVerticalOffset={Platform.OS === 'ios' ? 0 : 0}
      >
        <FlatList
          ref={flatListRef}
          data={messages}
          renderItem={renderMessage}
          keyExtractor={(item) => item.id}
          contentContainerStyle={chatStyles.messageList}
          showsVerticalScrollIndicator={false}
          onContentSizeChange={() => flatListRef.current?.scrollToEnd({ animated: false })}
          ListHeaderComponent={<DateSeparator date="วันนี้" />}
          ListFooterComponent={isTyping ? <TypingIndicator /> : null}
        />

        {/* Input Area */}
        <View style={[chatStyles.inputBar, { paddingBottom: insets.bottom + 8 }]}>
          <TouchableOpacity style={chatStyles.attachBtn}>
            <Text style={{ fontSize: 20 }}>➕</Text>
          </TouchableOpacity>

          <TextInput
            ref={inputRef}
            style={chatStyles.textInput}
            placeholder="พิมพ์ข้อความ..."
            value={inputText}
            onChangeText={setInputText}
            multiline
            maxLength={500}
            returnKeyType="default"
          />

          {inputText.trim() ? (
            <TouchableOpacity style={chatStyles.sendButton} onPress={sendMessage}>
              <Text style={chatStyles.sendButtonText}>➤</Text>
            </TouchableOpacity>
          ) : (
            <TouchableOpacity style={chatStyles.micBtn}>
              <Text style={{ fontSize: 20 }}>🎤</Text>
            </TouchableOpacity>
          )}
        </View>
      </KeyboardAvoidingView>
    </View>
  );
};

const chatStyles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 12,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.05,
    shadowRadius: 4,
    elevation: 3,
  },
  backBtn: { marginRight: 8 },
  headerInfo: {
    flex: 1,
    flexDirection: 'row',
    alignItems: 'center',
    gap: 10,
  },
  headerAvatar: { position: 'relative' },
  headerAvatarText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
  onlineDot: {
    position: 'absolute',
    bottom: 0,
    right: 0,
    width: 12,
    height: 12,
    borderRadius: 6,
    backgroundColor: '#34C759',
    borderWidth: 2,
    borderColor: '#fff',
  },
  headerName: { fontSize: 16, fontWeight: '600' },
  headerStatus: { fontSize: 12, color: '#34C759' },
  callBtn: {},
  messageList: {
    padding: 16,
    gap: 4,
    paddingBottom: 8,
  },
  messageRow: {
    flexDirection: 'row',
    marginVertical: 2,
  },
  myMessageRow: {
    justifyContent: 'flex-end',
  },
  otherMessageRow: {
    justifyContent: 'flex-start',
  },
  avatarContainer: { marginRight: 8, justifyContent: 'flex-end' },
  avatar: {
    width: 36,
    height: 36,
    borderRadius: 18,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  avatarText: { color: '#fff', fontWeight: 'bold' },
  bubble: {
    maxWidth: '75%',
    paddingHorizontal: 14,
    paddingVertical: 10,
    borderRadius: 18,
  },
  myBubble: {
    backgroundColor: '#007AFF',
    borderBottomRightRadius: 4,
  },
  otherBubble: {
    backgroundColor: '#fff',
    borderBottomLeftRadius: 4,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 2,
    elevation: 1,
  },
  messageText: { fontSize: 16, color: '#333', lineHeight: 22 },
  myMessageText: { color: '#fff' },
  messageMeta: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'flex-end',
    gap: 4,
    marginTop: 4,
  },
  timeText: { fontSize: 11, color: '#999' },
  myTimeText: { color: 'rgba(255,255,255,0.7)' },
  readIcon: { fontSize: 11, color: 'rgba(255,255,255,0.8)' },
  dateSeparator: {
    flexDirection: 'row',
    alignItems: 'center',
    marginVertical: 12,
    gap: 8,
  },
  dateLine: { flex: 1, height: 1, backgroundColor: '#e0e0e0' },
  dateText: { fontSize: 12, color: '#999', paddingHorizontal: 8 },
  typingContainer: { flexDirection: 'row', padding: 8, paddingLeft: 52 },
  typingBubble: {
    backgroundColor: '#fff',
    borderRadius: 18,
    borderBottomLeftRadius: 4,
    paddingHorizontal: 16,
    paddingVertical: 12,
    flexDirection: 'row',
    gap: 4,
    alignItems: 'center',
  },
  typingDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#999',
  },
  inputBar: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    paddingHorizontal: 12,
    paddingTop: 8,
    backgroundColor: '#fff',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    gap: 8,
  },
  attachBtn: {
    width: 36,
    height: 36,
    justifyContent: 'center',
    alignItems: 'center',
  },
  textInput: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#e0e0e0',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    maxHeight: 100,
    fontSize: 16,
    backgroundColor: '#f9f9f9',
  },
  sendButton: {
    width: 36,
    height: 36,
    borderRadius: 18,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  sendButtonText: { color: '#fff', fontSize: 16 },
  micBtn: {
    width: 36,
    height: 36,
    justifyContent: 'center',
    alignItems: 'center',
  },
});

export default ChatScreen;
```

---

## Tips และ Best Practices

### 1. ScrollView Performance
```jsx
// ✅ ดี: ใช้ getItemLayout เพื่อ performance
<FlatList
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
/>

// ✅ ดี: ปิด scroll indicators ที่ไม่จำเป็น
<ScrollView showsVerticalScrollIndicator={false} />
```

### 2. Keyboard Handling
```jsx
// ✅ รูปแบบแนะนำสำหรับ iOS
<KeyboardAvoidingView behavior="padding" keyboardVerticalOffset={44}>

// ✅ รูปแบบแนะนำสำหรับ Android
<KeyboardAvoidingView behavior="height">
```

### 3. SafeAreaView
```jsx
// ✅ ใช้ SafeAreaProvider ที่ root ของ app
// ✅ ใช้ useSafeAreaInsets สำหรับ custom layouts
// ✅ ระบุ edges ที่ต้องการ เพื่อควบคุมว่าด้าน ไหนที่ต้อง pad
<SafeAreaView edges={['top']}>  // เฉพาะด้านบน
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- ความแตกต่างระหว่าง `ScrollView` และ `FlatList`
- การสร้าง horizontal scroll และ carousel
- การใช้ `KeyboardAvoidingView` เพื่อป้องกัน keyboard บัง input
- การจัดการ safe areas ด้วย `SafeAreaView`
- การควบคุม `StatusBar`
- Workshop: Chat interface ที่สมบูรณ์

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Touchable Components และ Pressable
