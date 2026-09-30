# Part 009: Event Handling และ User Interaction

## สารบัญ
1. [Event System ใน React Native](#event-system)
2. [onPress และ onLongPress](#onpress)
3. [onChange และ onChangeText](#onchange)
4. [onFocus และ onBlur](#onfocus)
5. [onScroll](#onscroll)
6. [Gesture Basics](#gestures)
7. [PanResponder](#panresponder)
8. [React Native Gesture Handler](#gesture-handler)
9. [Haptic Feedback](#haptic)
10. [Workshop: Interactive UI](#workshop)

---

## Event System

### Synthetic Events ใน React Native

React Native มี event system ที่คล้ายกับ React บน Web แต่มีความแตกต่าง

```tsx
// Event Object โครงสร้าง
type NativeEvent = {
  nativeEvent: {
    // ข้อมูล event ที่มาจาก Native
    target: number;         // ID ของ element
    timestamp: number;      // Unix timestamp
    // ... ขึ้นอยู่กับ event type
  };
};

// Example: Press Event
const handlePress = (event: GestureResponderEvent) => {
  const { nativeEvent } = event;
  console.log('Location X:', nativeEvent.locationX);
  console.log('Location Y:', nativeEvent.locationY);
  console.log('Page X:', nativeEvent.pageX);
  console.log('Page Y:', nativeEvent.pageY);
  console.log('Timestamp:', nativeEvent.timestamp);
};
```

### SyntheticEvent Pooling

```tsx
// React Native ไม่ใช้ event pooling แบบ React Web (v17+)
// แต่ควร Extract values ก่อนใช้ใน async context

// ✅ ปลอดภัย
const handlePress = (event: GestureResponderEvent) => {
  const { locationX, locationY } = event.nativeEvent;  // Extract ก่อน
  
  setTimeout(() => {
    console.log(locationX, locationY);  // ใช้ extracted values
  }, 100);
};
```

---

## onPress และ onLongPress

### onPress พื้นฐาน

```tsx
import { TouchableOpacity, Pressable, Text } from 'react-native';

// TouchableOpacity
<TouchableOpacity onPress={() => console.log('กด!')}>
  <Text>กด</Text>
</TouchableOpacity>

// Pressable (แนะนำ)
<Pressable onPress={() => console.log('กด!')}>
  <Text>กด</Text>
</Pressable>
```

### Long Press

```tsx
<TouchableOpacity
  onPress={() => console.log('Single press')}
  onLongPress={() => console.log('Long press!')}
  delayLongPress={800}    // default: 500ms
>
  <Text>กดค้างเพื่อดู options</Text>
</TouchableOpacity>

// Context Menu Pattern
const ItemWithContextMenu = ({ item }) => {
  const [showMenu, setShowMenu] = useState(false);

  return (
    <>
      <TouchableOpacity
        onPress={() => handleItemPress(item)}
        onLongPress={() => setShowMenu(true)}
        delayLongPress={500}
      >
        <View style={styles.item}>
          <Text>{item.name}</Text>
        </View>
      </TouchableOpacity>

      {/* Context Menu Modal */}
      <Modal visible={showMenu} transparent animationType="fade">
        <TouchableOpacity
          style={styles.overlay}
          onPress={() => setShowMenu(false)}
        >
          <View style={styles.contextMenu}>
            <TouchableOpacity style={styles.menuItem} onPress={() => handleEdit(item)}>
              <Text>✏️ แก้ไข</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.menuItem} onPress={() => handleShare(item)}>
              <Text>📤 แชร์</Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={[styles.menuItem, styles.deleteItem]}
              onPress={() => handleDelete(item)}
            >
              <Text style={{ color: '#FF3B30' }}>🗑️ ลบ</Text>
            </TouchableOpacity>
          </View>
        </TouchableOpacity>
      </Modal>
    </>
  );
};
```

### Press Event Details

```tsx
<Pressable
  onPressIn={(event) => {
    // เมื่อเริ่มกด (finger down)
    console.log('Press In at:', event.nativeEvent.locationX, event.nativeEvent.locationY);
  }}
  onPressOut={(event) => {
    // เมื่อปล่อย (finger up)
    console.log('Press Out');
  }}
  onPress={(event) => {
    // Short press (< delayLongPress)
    console.log('Press');
  }}
  onLongPress={(event) => {
    // Long press (>= delayLongPress)
    console.log('Long Press');
  }}
>
  <Text>Complex Press Events</Text>
</Pressable>
```

### Double Tap

```tsx
const DoubleTapButton = ({ onDoubleTap }) => {
  const lastTapRef = useRef<number>(0);
  const DOUBLE_TAP_DELAY = 300; // ms

  const handlePress = () => {
    const now = Date.now();
    const timeSinceLastTap = now - lastTapRef.current;

    if (timeSinceLastTap < DOUBLE_TAP_DELAY && timeSinceLastTap > 0) {
      onDoubleTap();
      lastTapRef.current = 0;
    } else {
      lastTapRef.current = now;
    }
  };

  return (
    <TouchableOpacity onPress={handlePress}>
      <Text>Double tap ที่นี่</Text>
    </TouchableOpacity>
  );
};
```

### hitSlop - เพิ่มพื้นที่กด

```tsx
// hitSlop ทำให้ touch area ใหญ่ขึ้นโดยไม่เปลี่ยน visual size
<TouchableOpacity
  style={{ width: 30, height: 30 }}  // Visual size เล็ก
  hitSlop={{ top: 15, bottom: 15, left: 15, right: 15 }}  // Touch area ใหญ่กว่า
  onPress={handleClose}
>
  <Text>✕</Text>
</TouchableOpacity>
```

---

## onChange และ onChangeText

### TextInput Events

```tsx
<TextInput
  // onChangeText: รับ string ตรงๆ (แนะนำ)
  onChangeText={(text: string) => {
    console.log('New text:', text);
    setValue(text);
  }}
  
  // onChange: รับ SyntheticEvent
  onChange={(event) => {
    const { text, eventCount, target } = event.nativeEvent;
    console.log('Text:', text);
  }}
/>
```

### Real-time Validation

```tsx
const EmailInput = () => {
  const [email, setEmail] = useState('');
  const [isValid, setIsValid] = useState<boolean | null>(null);

  const validateEmail = (text: string) => {
    if (!text) {
      setIsValid(null);
      return;
    }
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    setIsValid(emailRegex.test(text));
  };

  const handleChange = (text: string) => {
    setEmail(text);
    validateEmail(text);
  };

  const borderColor = isValid === null
    ? '#D1D5DB'
    : isValid ? '#34C759' : '#FF3B30';

  return (
    <View>
      <TextInput
        value={email}
        onChangeText={handleChange}
        placeholder="email@example.com"
        keyboardType="email-address"
        autoCapitalize="none"
        style={[styles.input, { borderColor }]}
      />
      {isValid === false && (
        <Text style={styles.errorText}>รูปแบบ Email ไม่ถูกต้อง</Text>
      )}
      {isValid === true && (
        <Text style={styles.successText}>✓ Email ถูกต้อง</Text>
      )}
    </View>
  );
};
```

### Debounced Search

```tsx
import React, { useState, useCallback, useEffect } from 'react';
import { TextInput, View, Text, ActivityIndicator } from 'react-native';

// Custom useDebounce hook
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

const SearchInput = () => {
  const [searchText, setSearchText] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  
  const debouncedSearch = useDebounce(searchText, 300);  // 300ms delay

  useEffect(() => {
    if (!debouncedSearch.trim()) {
      setResults([]);
      return;
    }

    const search = async () => {
      setLoading(true);
      try {
        // API call
        const data = await searchProducts(debouncedSearch);
        setResults(data);
      } catch (error) {
        console.error(error);
      } finally {
        setLoading(false);
      }
    };

    search();
  }, [debouncedSearch]);

  return (
    <View>
      <View style={styles.searchContainer}>
        <TextInput
          value={searchText}
          onChangeText={setSearchText}
          placeholder="ค้นหาสินค้า..."
          style={styles.searchInput}
          returnKeyType="search"
          clearButtonMode="while-editing"
        />
        {loading && <ActivityIndicator size="small" style={styles.loader} />}
      </View>
      
      {results.map(item => (
        <View key={item.id} style={styles.resultItem}>
          <Text>{item.name}</Text>
        </View>
      ))}
    </View>
  );
};
```

### Masked Input (Phone Number)

```tsx
const PhoneInput = () => {
  const [phone, setPhone] = useState('');
  const [displayValue, setDisplayValue] = useState('');

  const formatPhone = (text: string) => {
    // ลบทุกอย่างที่ไม่ใช่ตัวเลข
    const digits = text.replace(/\D/g, '');
    
    // จำกัดที่ 10 หลัก
    const limited = digits.slice(0, 10);
    
    // Format: 0XX-XXX-XXXX
    let formatted = '';
    if (limited.length <= 3) {
      formatted = limited;
    } else if (limited.length <= 6) {
      formatted = `${limited.slice(0, 3)}-${limited.slice(3)}`;
    } else {
      formatted = `${limited.slice(0, 3)}-${limited.slice(3, 6)}-${limited.slice(6)}`;
    }
    
    setPhone(limited);
    setDisplayValue(formatted);
  };

  return (
    <TextInput
      value={displayValue}
      onChangeText={formatPhone}
      placeholder="0XX-XXX-XXXX"
      keyboardType="phone-pad"
      maxLength={12}  // 10 digits + 2 dashes
    />
  );
};
```

---

## onFocus และ onBlur

```tsx
const FocusableInput = () => {
  const [isFocused, setIsFocused] = useState(false);

  return (
    <TextInput
      onFocus={() => setIsFocused(true)}
      onBlur={() => setIsFocused(false)}
      style={[
        styles.input,
        isFocused && styles.inputFocused,
      ]}
      placeholder="กดเพื่อพิมพ์"
    />
  );
};

// Animated Label Input
const AnimatedInput = () => {
  const [isFocused, setIsFocused] = useState(false);
  const [value, setValue] = useState('');
  const labelAnim = useRef(new Animated.Value(0)).current;

  const animateLabel = (focused: boolean) => {
    Animated.timing(labelAnim, {
      toValue: focused || value.length > 0 ? 1 : 0,
      duration: 200,
      useNativeDriver: false,
    }).start();
  };

  const handleFocus = () => {
    setIsFocused(true);
    animateLabel(true);
  };

  const handleBlur = () => {
    setIsFocused(false);
    animateLabel(false);
  };

  const labelTop = labelAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [14, -8],
  });

  const labelSize = labelAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [15, 12],
  });

  const labelColor = labelAnim.interpolate({
    inputRange: [0, 1],
    outputRange: ['#9CA3AF', '#007AFF'],
  });

  return (
    <View style={[styles.inputContainer, isFocused && styles.inputContainerFocused]}>
      <Animated.Text
        style={[
          styles.floatingLabel,
          {
            top: labelTop,
            fontSize: labelSize,
            color: labelColor,
          },
        ]}
      >
        Email
      </Animated.Text>
      <TextInput
        value={value}
        onChangeText={setValue}
        onFocus={handleFocus}
        onBlur={handleBlur}
        style={styles.animatedInput}
        keyboardType="email-address"
        autoCapitalize="none"
      />
    </View>
  );
};
```

---

## onScroll

```tsx
import { ScrollView, Animated } from 'react-native';

const ScrollableScreen = () => {
  const scrollY = useRef(new Animated.Value(0)).current;

  // Header opacity เปลี่ยนตาม scroll
  const headerOpacity = scrollY.interpolate({
    inputRange: [0, 100],
    outputRange: [0, 1],
    extrapolate: 'clamp',
  });

  // Header height เปลี่ยนตาม scroll
  const headerHeight = scrollY.interpolate({
    inputRange: [0, 100],
    outputRange: [200, 60],
    extrapolate: 'clamp',
  });

  return (
    <View style={{ flex: 1 }}>
      {/* Sticky Header */}
      <Animated.View
        style={[
          styles.header,
          {
            height: headerHeight,
            opacity: headerOpacity,
          },
        ]}
      >
        <Text style={styles.headerTitle}>ชื่อหน้า</Text>
      </Animated.View>

      {/* Scrollable Content */}
      <Animated.ScrollView
        onScroll={Animated.event(
          [{ nativeEvent: { contentOffset: { y: scrollY } } }],
          { useNativeDriver: false }
        )}
        scrollEventThrottle={16}  // ~60fps
      >
        {/* Content */}
      </Animated.ScrollView>
    </View>
  );
};
```

### Scroll Direction Detection

```tsx
const ScrollDirectionDetector = () => {
  const lastScrollY = useRef(0);
  const [isScrollingDown, setIsScrollingDown] = useState(false);

  const handleScroll = (event: NativeSyntheticEvent<NativeScrollEvent>) => {
    const currentScrollY = event.nativeEvent.contentOffset.y;
    
    if (currentScrollY > lastScrollY.current) {
      setIsScrollingDown(true);
    } else {
      setIsScrollingDown(false);
    }
    
    lastScrollY.current = currentScrollY;
  };

  return (
    <View style={{ flex: 1 }}>
      {/* Hide/Show Bottom Tab based on scroll direction */}
      {!isScrollingDown && (
        <View style={styles.bottomTab}>
          <Text>Tab Bar</Text>
        </View>
      )}
      
      <ScrollView
        onScroll={handleScroll}
        scrollEventThrottle={16}
      >
        {/* Content */}
      </ScrollView>
    </View>
  );
};
```

### Infinite Scroll

```tsx
const InfiniteScrollList = () => {
  const [items, setItems] = useState<Item[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    try {
      const newItems = await fetchItems(page);
      if (newItems.length === 0) {
        setHasMore(false);
      } else {
        setItems(prev => [...prev, ...newItems]);
        setPage(prev => prev + 1);
      }
    } finally {
      setLoading(false);
    }
  };

  const handleScroll = ({ nativeEvent }) => {
    const { layoutMeasurement, contentOffset, contentSize } = nativeEvent;
    const distanceFromBottom = contentSize.height - layoutMeasurement.height - contentOffset.y;
    
    if (distanceFromBottom < 200) {  // 200px จากล่าง
      loadMore();
    }
  };

  return (
    <ScrollView
      onScroll={handleScroll}
      scrollEventThrottle={400}
    >
      {items.map(item => (
        <ItemCard key={item.id} item={item} />
      ))}
      {loading && <ActivityIndicator />}
      {!hasMore && <Text style={{ textAlign: 'center', color: '#999' }}>ไม่มีข้อมูลเพิ่มเติม</Text>}
    </ScrollView>
  );
};
```

---

## Gestures

### Basic Touch Events

```tsx
// ทุก View รองรับ touch events (ผ่าน onStartShouldSetResponder)
<View
  onStartShouldSetResponder={() => true}   // เริ่ม respond
  onResponderGrant={(event) => {
    console.log('Touch granted');
  }}
  onResponderMove={(event) => {
    const { locationX, locationY } = event.nativeEvent;
    console.log(`Moving: ${locationX}, ${locationY}`);
  }}
  onResponderRelease={(event) => {
    console.log('Touch released');
  }}
>
  <Text>Touchable View</Text>
</View>
```

---

## PanResponder

PanResponder ใช้สำหรับ drag, swipe gestures

```tsx
import React, { useRef } from 'react';
import { View, PanResponder, Animated, StyleSheet } from 'react-native';

const DraggableCard = () => {
  const pan = useRef(new Animated.ValueXY()).current;

  const panResponder = useRef(
    PanResponder.create({
      // เมื่อ touch เริ่ม
      onStartShouldSetPanResponder: () => true,
      onStartShouldSetPanResponderCapture: () => true,
      
      // เมื่อ gesture เริ่ม
      onPanResponderGrant: () => {
        pan.setOffset({
          x: (pan.x as any)._value,
          y: (pan.y as any)._value,
        });
      },
      
      // เมื่อ drag
      onPanResponderMove: Animated.event(
        [null, { dx: pan.x, dy: pan.y }],
        { useNativeDriver: false }
      ),
      
      // เมื่อปล่อย
      onPanResponderRelease: () => {
        pan.flattenOffset();
        
        // Spring back to center
        Animated.spring(pan, {
          toValue: { x: 0, y: 0 },
          useNativeDriver: false,
        }).start();
      },
    })
  ).current;

  return (
    <View style={styles.container}>
      <Animated.View
        style={[
          styles.card,
          { transform: [{ translateX: pan.x }, { translateY: pan.y }] },
        ]}
        {...panResponder.panHandlers}
      >
        <Text style={styles.cardText}>ลากฉัน!</Text>
      </Animated.View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
  },
  card: {
    width: 150,
    height: 150,
    backgroundColor: '#007AFF',
    borderRadius: 16,
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 8,
  },
  cardText: {
    color: '#fff',
    fontSize: 18,
    fontWeight: 'bold',
  },
});

export default DraggableCard;
```

### Swipe to Delete

```tsx
const SwipeableItem = ({ item, onDelete }) => {
  const translateX = useRef(new Animated.Value(0)).current;
  const [isRevealed, setIsRevealed] = useState(false);
  
  const DELETE_THRESHOLD = -80;

  const panResponder = PanResponder.create({
    onStartShouldSetPanResponder: () => false,
    onMoveShouldSetPanResponder: (_, { dx, dy }) => {
      return Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > 5;
    },
    
    onPanResponderMove: (_, { dx }) => {
      if (dx < 0) {
        translateX.setValue(Math.max(dx, DELETE_THRESHOLD * 1.2));
      }
    },
    
    onPanResponderRelease: (_, { dx }) => {
      if (dx < DELETE_THRESHOLD / 2) {
        // Snap to reveal delete button
        Animated.spring(translateX, {
          toValue: DELETE_THRESHOLD,
          useNativeDriver: true,
        }).start();
        setIsRevealed(true);
      } else {
        // Snap back
        Animated.spring(translateX, {
          toValue: 0,
          useNativeDriver: true,
        }).start();
        setIsRevealed(false);
      }
    },
  });

  return (
    <View style={styles.swipeContainer}>
      {/* Delete action */}
      <View style={styles.deleteAction}>
        <TouchableOpacity
          style={styles.deleteButton}
          onPress={() => onDelete(item.id)}
        >
          <Text style={styles.deleteText}>ลบ</Text>
        </TouchableOpacity>
      </View>
      
      {/* Main content */}
      <Animated.View
        style={[styles.itemContent, { transform: [{ translateX }] }]}
        {...panResponder.panHandlers}
      >
        <Text style={styles.itemText}>{item.name}</Text>
      </Animated.View>
    </View>
  );
};
```

---

## React Native Gesture Handler

`react-native-gesture-handler` ให้ performance ดีกว่า PanResponder

```bash
npm install react-native-gesture-handler
cd ios && pod install
```

```tsx
// index.js - ต้อง wrap App ด้วย GestureHandlerRootView
import { GestureHandlerRootView } from 'react-native-gesture-handler';

const App = () => (
  <GestureHandlerRootView style={{ flex: 1 }}>
    <AppContent />
  </GestureHandlerRootView>
);
```

```tsx
import {
  GestureDetector,
  Gesture,
} from 'react-native-gesture-handler';
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withSpring,
} from 'react-native-reanimated';

const DraggableBox = () => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onBegin(() => {
      offsetX.value = translateX.value;
      offsetY.value = translateY.value;
    })
    .onUpdate((e) => {
      translateX.value = offsetX.value + e.translationX;
      translateY.value = offsetY.value + e.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });

  const tapGesture = Gesture.Tap()
    .onEnd(() => {
      console.log('Tapped!');
    });

  // Combine gestures
  const combinedGesture = Gesture.Simultaneous(panGesture, tapGesture);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }));

  return (
    <GestureDetector gesture={combinedGesture}>
      <Animated.View style={[styles.box, animatedStyle]}>
        <Text>Drag me!</Text>
      </Animated.View>
    </GestureDetector>
  );
};
```

---

## Haptic Feedback

Haptic feedback ให้ tactile response

```bash
# iOS: ใช้ built-in
# Android: ต้องติดตั้ง library

npm install react-native-haptic-feedback
# หรือ
npm install expo-haptics  # สำหรับ Expo
```

```tsx
import ReactNativeHapticFeedback from 'react-native-haptic-feedback';
// หรือ
import * as Haptics from 'expo-haptics';

const HapticButton = () => {
  const handlePress = () => {
    // Expo Haptics
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    
    // หรือ react-native-haptic-feedback
    ReactNativeHapticFeedback.trigger('impactMedium', {
      enableVibrateFallback: true,
      ignoreAndroidSystemSettings: false,
    });
  };

  return (
    <TouchableOpacity onPress={handlePress}>
      <Text>กดเพื่อรู้สึก Haptic</Text>
    </TouchableOpacity>
  );
};

// Haptic Types:
// 'selection'        - เบา
// 'impactLight'      - เบา
// 'impactMedium'     - ปานกลาง
// 'impactHeavy'      - หนัก
// 'notificationSuccess' - success
// 'notificationWarning' - warning
// 'notificationError'   - error
```

---

## Workshop: Interactive UI

### Workshop 9.1: Interactive Card Swiper

```tsx
import React, { useState, useRef } from 'react';
import {
  View, Text, PanResponder, Animated, StyleSheet,
  SafeAreaView, TouchableOpacity, Dimensions,
} from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');
const SWIPE_THRESHOLD = SCREEN_WIDTH * 0.3;

type Card = {
  id: string;
  name: string;
  age: number;
  bio: string;
  color: string;
};

const cards: Card[] = [
  { id: '1', name: 'สมชาย', age: 25, bio: 'ชอบเล่น React Native', color: '#FF6B6B' },
  { id: '2', name: 'สมหญิง', age: 23, bio: 'Flutter Developer', color: '#4ECDC4' },
  { id: '3', name: 'สมใจ', age: 27, bio: 'Full Stack Developer', color: '#45B7D1' },
  { id: '4', name: 'สมศรี', age: 24, bio: 'UI/UX Designer', color: '#96CEB4' },
];

const CardSwiper = () => {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [likedCards, setLikedCards] = useState<string[]>([]);
  const [dislikedCards, setDislikedCards] = useState<string[]>([]);
  const position = useRef(new Animated.ValueXY()).current;

  const rotate = position.x.interpolate({
    inputRange: [-SCREEN_WIDTH / 2, 0, SCREEN_WIDTH / 2],
    outputRange: ['-20deg', '0deg', '20deg'],
  });

  const likeOpacity = position.x.interpolate({
    inputRange: [-SCREEN_WIDTH / 4, 0, SCREEN_WIDTH / 4],
    outputRange: [0, 0, 1],
    extrapolate: 'clamp',
  });

  const dislikeOpacity = position.x.interpolate({
    inputRange: [-SCREEN_WIDTH / 4, 0, SCREEN_WIDTH / 4],
    outputRange: [1, 0, 0],
    extrapolate: 'clamp',
  });

  const panResponder = PanResponder.create({
    onStartShouldSetPanResponder: () => true,
    onPanResponderMove: Animated.event(
      [null, { dx: position.x, dy: position.y }],
      { useNativeDriver: false }
    ),
    onPanResponderRelease: (_, { dx }) => {
      if (dx > SWIPE_THRESHOLD) {
        swipeRight();
      } else if (dx < -SWIPE_THRESHOLD) {
        swipeLeft();
      } else {
        resetPosition();
      }
    },
  });

  const swipeRight = () => {
    Animated.timing(position, {
      toValue: { x: SCREEN_WIDTH + 100, y: 0 },
      duration: 300,
      useNativeDriver: false,
    }).start(() => {
      setLikedCards(prev => [...prev, cards[currentIndex].id]);
      nextCard();
    });
  };

  const swipeLeft = () => {
    Animated.timing(position, {
      toValue: { x: -SCREEN_WIDTH - 100, y: 0 },
      duration: 300,
      useNativeDriver: false,
    }).start(() => {
      setDislikedCards(prev => [...prev, cards[currentIndex].id]);
      nextCard();
    });
  };

  const resetPosition = () => {
    Animated.spring(position, {
      toValue: { x: 0, y: 0 },
      useNativeDriver: false,
    }).start();
  };

  const nextCard = () => {
    position.setValue({ x: 0, y: 0 });
    setCurrentIndex(prev => prev + 1);
  };

  if (currentIndex >= cards.length) {
    return (
      <SafeAreaView style={styles.container}>
        <View style={styles.emptyContainer}>
          <Text style={styles.emptyEmoji}>🎉</Text>
          <Text style={styles.emptyTitle}>ดูครบแล้ว!</Text>
          <Text style={styles.emptyStats}>
            ✅ Like: {likedCards.length} | ❌ Dislike: {dislikedCards.length}
          </Text>
          <TouchableOpacity
            style={styles.resetButton}
            onPress={() => {
              setCurrentIndex(0);
              setLikedCards([]);
              setDislikedCards([]);
            }}
          >
            <Text style={styles.resetButtonText}>เริ่มใหม่</Text>
          </TouchableOpacity>
        </View>
      </SafeAreaView>
    );
  }

  const currentCard = cards[currentIndex];

  return (
    <SafeAreaView style={styles.container}>
      {/* Stats */}
      <View style={styles.stats}>
        <Text style={styles.statText}>❌ {dislikedCards.length}</Text>
        <Text style={styles.statText}>✅ {likedCards.length}</Text>
      </View>

      {/* Cards Stack */}
      <View style={styles.cardsContainer}>
        {/* Next card (underneath) */}
        {currentIndex + 1 < cards.length && (
          <View style={[styles.card, styles.nextCard, { backgroundColor: cards[currentIndex + 1].color }]}>
            <Text style={styles.cardName}>{cards[currentIndex + 1].name}</Text>
          </View>
        )}

        {/* Current card */}
        <Animated.View
          style={[
            styles.card,
            {
              backgroundColor: currentCard.color,
              transform: [
                { translateX: position.x },
                { translateY: position.y },
                { rotate },
              ],
            },
          ]}
          {...panResponder.panHandlers}
        >
          {/* Like / Dislike overlay */}
          <Animated.View style={[styles.likeLabel, { opacity: likeOpacity }]}>
            <Text style={styles.likeText}>LIKE ✅</Text>
          </Animated.View>
          <Animated.View style={[styles.dislikeLabel, { opacity: dislikeOpacity }]}>
            <Text style={styles.dislikeText}>NOPE ❌</Text>
          </Animated.View>

          <Text style={styles.cardName}>{currentCard.name}</Text>
          <Text style={styles.cardAge}>อายุ {currentCard.age} ปี</Text>
          <Text style={styles.cardBio}>{currentCard.bio}</Text>
        </Animated.View>
      </View>

      {/* Action Buttons */}
      <View style={styles.actionButtons}>
        <TouchableOpacity
          style={[styles.actionButton, styles.dislikeButton]}
          onPress={swipeLeft}
        >
          <Text style={styles.actionButtonText}>❌</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.actionButton, styles.likeButton]}
          onPress={swipeRight}
        >
          <Text style={styles.actionButtonText}>✅</Text>
        </TouchableOpacity>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  stats: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    padding: 16,
  },
  statText: { fontSize: 16, fontWeight: '600' },
  cardsContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  card: {
    position: 'absolute',
    width: SCREEN_WIDTH * 0.85,
    height: SCREEN_WIDTH * 1.1,
    borderRadius: 20,
    padding: 24,
    justifyContent: 'flex-end',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 8 },
    shadowOpacity: 0.2,
    shadowRadius: 12,
    elevation: 8,
  },
  nextCard: {
    transform: [{ scale: 0.95 }],
    bottom: -10,
  },
  cardName: { fontSize: 28, fontWeight: 'bold', color: '#fff', marginBottom: 4 },
  cardAge: { fontSize: 18, color: 'rgba(255,255,255,0.8)', marginBottom: 8 },
  cardBio: { fontSize: 15, color: 'rgba(255,255,255,0.9)', lineHeight: 22 },
  likeLabel: {
    position: 'absolute',
    top: 24,
    left: 24,
    padding: 8,
    borderWidth: 3,
    borderColor: '#34C759',
    borderRadius: 8,
    transform: [{ rotate: '-15deg' }],
  },
  likeText: { fontSize: 20, fontWeight: 'bold', color: '#34C759' },
  dislikeLabel: {
    position: 'absolute',
    top: 24,
    right: 24,
    padding: 8,
    borderWidth: 3,
    borderColor: '#FF3B30',
    borderRadius: 8,
    transform: [{ rotate: '15deg' }],
  },
  dislikeText: { fontSize: 20, fontWeight: 'bold', color: '#FF3B30' },
  actionButtons: {
    flexDirection: 'row',
    justifyContent: 'center',
    gap: 40,
    paddingBottom: 40,
    paddingTop: 20,
  },
  actionButton: {
    width: 64,
    height: 64,
    borderRadius: 32,
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.2,
    shadowRadius: 6,
    elevation: 4,
  },
  likeButton: { backgroundColor: '#fff' },
  dislikeButton: { backgroundColor: '#fff' },
  actionButtonText: { fontSize: 28 },
  emptyContainer: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  emptyEmoji: { fontSize: 64, marginBottom: 16 },
  emptyTitle: { fontSize: 24, fontWeight: 'bold', color: '#333', marginBottom: 8 },
  emptyStats: { fontSize: 16, color: '#666', marginBottom: 24 },
  resetButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 32,
    paddingVertical: 14,
    borderRadius: 12,
  },
  resetButtonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});

export default CardSwiper;
```

### Workshop 9.2: แบบฝึกหัด

**แบบฝึกหัด 1:** สร้าง Drawing Pad:
- ใช้ PanResponder track การเคลื่อนที่ของนิ้ว
- วาด path ตาม gesture
- ปุ่ม "Clear" เพื่อล้าง

**แบบฝึกหัด 2:** Pinch to Zoom Image:
- ใช้ PanResponder กับ 2 นิ้ว
- Zoom in/out ตาม pinch gesture
- กลับมาขนาดเดิมเมื่อ double tap

**แบบฝึกหัด 3:** Pull to Reveal Secret:
- ScrollView ที่ pull ลงเพื่อ reveal content ที่ซ่อนอยู่
- Animated indicator แสดงความคืบหน้า

---

## Tips และ Best Practices

### 1. ใช้ useCallback สำหรับ Event Handlers

```tsx
// ✅ ป้องกัน re-render จาก handler ใหม่
const handlePress = useCallback(() => {
  doSomething(id);
}, [id]);

<TouchableOpacity onPress={handlePress}>
  <Text>กด</Text>
</TouchableOpacity>
```

### 2. Throttle Expensive Events

```tsx
import throttle from 'lodash/throttle';

// Throttle scroll handler ไม่ให้เรียกบ่อยเกิน
const handleScroll = useMemo(
  () => throttle((event) => {
    updateScrollPosition(event.nativeEvent.contentOffset.y);
  }, 100),
  []
);
```

### 3. Cleanup Event Listeners

```tsx
useEffect(() => {
  const subscription = SomeEventEmitter.addListener('event', handler);
  
  return () => {
    subscription.remove();  // ✅ Cleanup
  };
}, []);
```

### 4. Accessible Touch Targets

```tsx
// ✅ Touch target ต้องมีขนาดอย่างน้อย 44x44pt
<TouchableOpacity
  style={{ minWidth: 44, minHeight: 44, justifyContent: 'center', alignItems: 'center' }}
  // หรือ
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
>
  <SmallIcon size={20} />
</TouchableOpacity>
```

---

## สรุป Part 009

### ได้เรียนรู้

1. **Event System** - NativeEvent, SyntheticEvent
2. **onPress/onLongPress** - เวลา, hitSlop, double tap
3. **onChange/onChangeText** - text input, validation, debounce
4. **onFocus/onBlur** - animated input, keyboard
5. **onScroll** - animated header, direction detection, infinite scroll
6. **PanResponder** - drag, swipe to delete
7. **Gesture Handler** - modern gesture API
8. **Haptic Feedback** - tactile response

---

**ต่อไป → Part 010: Lists: FlatList, SectionList, ScrollView**
