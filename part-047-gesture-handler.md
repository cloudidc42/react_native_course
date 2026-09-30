# Part 047: Gesture Handler ใน React Native

## สารบัญ
1. [แนะนำ Gesture Handler](#introduction)
2. [PanGestureHandler](#pan-gesture)
3. [PinchGestureHandler](#pinch-gesture)
4. [TapGestureHandler](#tap-gesture)
5. [LongPressGestureHandler](#longpress-gesture)
6. [Combined Gestures](#combined-gestures)
7. [Workshop: Draggable Elements](#workshop)

---

## 1. แนะนำ Gesture Handler {#introduction}

React Native Gesture Handler เป็น library สำหรับจัดการ gestures บน native thread ทำให้ได้ performance สูงกว่า PanResponder

### ติดตั้ง

```bash
npm install react-native-gesture-handler
cd ios && pod install
```

### Setup

```typescript
// index.js หรือ App.tsx (ต้อง wrap ด้วย GestureHandlerRootView)
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      {/* rest of your app */}
    </GestureHandlerRootView>
  );
}
```

### Gesture Types

| Gesture | คำอธิบาย |
|---------|---------|
| Tap | แตะครั้งเดียว |
| LongPress | กดค้าง |
| Pan | ลากนิ้ว |
| Pinch | บีบ/ขยาย |
| Rotation | หมุน |
| Fling | กวาด |
| ForceTouchGesture | 3D Touch (iOS) |

---

## 2. PanGestureHandler {#pan-gesture}

### PanGestureHandler พื้นฐาน

```typescript
// src/components/PanGestureDemo.tsx
import React from 'react';
import { View, Text, StyleSheet, Dimensions } from 'react-native';
import { PanGestureHandler, PanGestureHandlerGestureEvent } from 'react-native-gesture-handler';
import Animated, {
  useAnimatedGestureHandler,
  useAnimatedStyle,
  useSharedValue,
  withSpring,
  withDecay,
  clamp,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');
const BOX_SIZE = 80;

type GestureContext = {
  startX: number;
  startY: number;
};

const PanGestureDemo: React.FC = () => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);

  const gestureHandler = useAnimatedGestureHandler<
    PanGestureHandlerGestureEvent,
    GestureContext
  >({
    onStart: (_, context) => {
      context.startX = translateX.value;
      context.startY = translateY.value;
      scale.value = withSpring(1.1);
    },
    onActive: (event, context) => {
      // จำกัดการเคลื่อนที่ให้อยู่ใน screen
      const maxX = SCREEN_WIDTH / 2 - BOX_SIZE / 2;
      const maxY = SCREEN_HEIGHT / 2 - BOX_SIZE / 2;

      translateX.value = clamp(
        context.startX + event.translationX,
        -maxX,
        maxX
      );
      translateY.value = clamp(
        context.startY + event.translationY,
        -maxY,
        maxY
      );
    },
    onEnd: (event) => {
      scale.value = withSpring(1);

      // Decay animation (momentum)
      translateX.value = withDecay({
        velocity: event.velocityX,
        clamp: [-SCREEN_WIDTH / 2 + BOX_SIZE / 2, SCREEN_WIDTH / 2 - BOX_SIZE / 2],
      });
      translateY.value = withDecay({
        velocity: event.velocityY,
        clamp: [-SCREEN_HEIGHT / 2 + BOX_SIZE / 2, SCREEN_HEIGHT / 2 - BOX_SIZE / 2],
      });
    },
  });

  const animStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <View style={styles.container}>
      <PanGestureHandler onGestureEvent={gestureHandler}>
        <Animated.View style={[styles.box, animStyle]}>
          <Text style={styles.boxText}>ลากฉัน!</Text>
        </Animated.View>
      </PanGestureHandler>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5F5F5',
  },
  box: {
    width: BOX_SIZE,
    height: BOX_SIZE,
    backgroundColor: '#2196F3',
    borderRadius: 12,
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.3,
    shadowRadius: 5,
  },
  boxText: { color: 'white', fontWeight: 'bold', fontSize: 11 },
});

export default PanGestureDemo;
```

### Gesture Handler v2 API (แนะนำ)

```typescript
// src/components/GestureV2Demo.tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

const GestureV2Demo: React.FC = () => {
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);
  const startX = useSharedValue(0);
  const startY = useSharedValue(0);

  const pan = Gesture.Pan()
    .minDistance(0)
    .onBegin(() => {
      startX.value = offsetX.value;
      startY.value = offsetY.value;
    })
    .onUpdate((event) => {
      offsetX.value = startX.value + event.translationX;
      offsetY.value = startY.value + event.translationY;
    })
    .onEnd(() => {
      offsetX.value = withSpring(0);
      offsetY.value = withSpring(0);
    });

  const style = useAnimatedStyle(() => ({
    transform: [
      { translateX: offsetX.value },
      { translateY: offsetY.value },
    ],
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[{ width: 80, height: 80, backgroundColor: '#2196F3', borderRadius: 12 }, style]} />
    </GestureDetector>
  );
};
```

---

## 3. PinchGestureHandler {#pinch-gesture}

```typescript
// src/components/PinchGestureDemo.tsx
import React, { useRef } from 'react';
import { View, Image, StyleSheet } from 'react-native';
import {
  PinchGestureHandler,
  PanGestureHandler,
  State,
  GestureHandlerRootView,
} from 'react-native-gesture-handler';
import Animated, {
  useAnimatedGestureHandler,
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

const ImageViewer: React.FC<{ imageUri: string }> = ({ imageUri }) => {
  const scale = useSharedValue(1);
  const focalX = useSharedValue(0);
  const focalY = useSharedValue(0);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const lastScale = useSharedValue(1);
  const lastTranslateX = useSharedValue(0);
  const lastTranslateY = useSharedValue(0);

  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = lastScale.value * event.scale;
      focalX.value = event.focalX;
      focalY.value = event.focalY;
    })
    .onEnd(() => {
      // Clamp scale
      if (scale.value < 1) {
        scale.value = withSpring(1);
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        lastScale.value = 1;
      } else if (scale.value > 4) {
        scale.value = withSpring(4);
        lastScale.value = 4;
      } else {
        lastScale.value = scale.value;
      }
    });

  const panGesture = Gesture.Pan()
    .enabled(scale.value > 1)
    .onUpdate((event) => {
      translateX.value = lastTranslateX.value + event.translationX;
      translateY.value = lastTranslateY.value + event.translationY;
    })
    .onEnd(() => {
      lastTranslateX.value = translateX.value;
      lastTranslateY.value = translateY.value;
    });

  // Double tap to reset
  const doubleTap = Gesture.Tap()
    .numberOfTaps(2)
    .onEnd(() => {
      if (scale.value !== 1) {
        scale.value = withSpring(1);
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        lastScale.value = 1;
        lastTranslateX.value = 0;
        lastTranslateY.value = 0;
      } else {
        scale.value = withSpring(2);
        lastScale.value = 2;
      }
    });

  const composed = Gesture.Simultaneous(pinchGesture, panGesture, doubleTap);

  const imageStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <View style={styles.container}>
      <GestureDetector gesture={composed}>
        <Animated.Image
          source={{ uri: imageUri }}
          style={[styles.image, imageStyle]}
          resizeMode="contain"
        />
      </GestureDetector>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: 'black',
    overflow: 'hidden',
  },
  image: {
    flex: 1,
    width: '100%',
  },
});

export default ImageViewer;
```

---

## 4. TapGestureHandler {#tap-gesture}

```typescript
// src/components/TapGestureDemo.tsx
import React, { useState } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  withSequence,
  runOnJS,
} from 'react-native-reanimated';

const TapGestureDemo: React.FC = () => {
  const [tapCount, setTapCount] = useState(0);
  const scale = useSharedValue(1);
  const opacity = useSharedValue(1);
  const heartScale = useSharedValue(0);
  const heartOpacity = useSharedValue(0);

  const incrementTap = () => setTapCount(prev => prev + 1);
  const showDoubleTapFeedback = () => {
    heartScale.value = withSequence(
      withSpring(1.5),
      withSpring(1),
      withTiming(0, { duration: 800 })
    );
    heartOpacity.value = withSequence(
      withTiming(1, { duration: 200 }),
      withTiming(0, { duration: 800 })
    );
  };

  // Single Tap
  const singleTap = Gesture.Tap()
    .maxDuration(250)
    .onEnd(() => {
      scale.value = withSequence(
        withTiming(0.95, { duration: 100 }),
        withSpring(1)
      );
      runOnJS(incrementTap)();
    });

  // Double Tap
  const doubleTap = Gesture.Tap()
    .numberOfTaps(2)
    .maxDuration(250)
    .onEnd(() => {
      runOnJS(showDoubleTapFeedback)();
      opacity.value = withSequence(
        withTiming(0.6, { duration: 100 }),
        withTiming(1, { duration: 200 })
      );
    });

  // Exclusive - ให้ double tap มีความสำคัญกว่า single tap
  const composed = Gesture.Exclusive(doubleTap, singleTap);

  const boxStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    opacity: opacity.value,
  }));

  const heartStyle = useAnimatedStyle(() => ({
    transform: [{ scale: heartScale.value }],
    opacity: heartOpacity.value,
    position: 'absolute',
  }));

  return (
    <View style={styles.container}>
      <Text style={styles.tapCount}>แตะ: {tapCount} ครั้ง</Text>
      <Text style={styles.instruction}>แตะ 2 ครั้งเพื่อ Like</Text>

      <GestureDetector gesture={composed}>
        <Animated.View style={[styles.card, boxStyle]}>
          <Text style={styles.cardText}>แตะที่นี่</Text>
          <Text style={styles.cardSubText}>แตะ 1 ครั้ง = นับจำนวน</Text>
          <Text style={styles.cardSubText}>แตะ 2 ครั้ง = Like</Text>

          <Animated.Text style={[styles.heart, heartStyle]}>❤️</Animated.Text>
        </Animated.View>
      </GestureDetector>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', gap: 16, padding: 20 },
  tapCount: { fontSize: 24, fontWeight: 'bold' },
  instruction: { color: '#666', marginBottom: 8 },
  card: {
    width: 260,
    height: 180,
    backgroundColor: 'white',
    borderRadius: 20,
    justifyContent: 'center',
    alignItems: 'center',
    gap: 8,
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.2,
    shadowRadius: 6,
  },
  cardText: { fontSize: 20, fontWeight: 'bold' },
  cardSubText: { color: '#666', fontSize: 13 },
  heart: { fontSize: 60 },
});
```

---

## 5. LongPressGestureHandler {#longpress-gesture}

```typescript
// src/components/LongPressMenu.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Modal,
  TouchableOpacity,
} from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,
} from 'react-native-reanimated';
import * as Haptics from 'expo-haptics';

interface ContextMenuItem {
  icon: string;
  label: string;
  action: () => void;
  destructive?: boolean;
}

interface LongPressMenuProps {
  children: React.ReactNode;
  menuItems: ContextMenuItem[];
}

const LongPressMenu: React.FC<LongPressMenuProps> = ({ children, menuItems }) => {
  const [showMenu, setShowMenu] = useState(false);
  const [menuPosition, setMenuPosition] = useState({ x: 0, y: 0 });
  const scale = useSharedValue(1);
  const opacity = useSharedValue(1);

  const triggerHaptic = () => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
  };

  const openMenu = (x: number, y: number) => {
    setMenuPosition({ x, y });
    setShowMenu(true);
  };

  const longPress = Gesture.LongPress()
    .minDuration(500)
    .onBegin(() => {
      scale.value = withSpring(0.95);
    })
    .onStart((event) => {
      runOnJS(triggerHaptic)();
      runOnJS(openMenu)(event.absoluteX, event.absoluteY);
    })
    .onFinalize(() => {
      scale.value = withSpring(1);
    });

  const animStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    opacity: opacity.value,
  }));

  return (
    <>
      <GestureDetector gesture={longPress}>
        <Animated.View style={animStyle}>
          {children}
        </Animated.View>
      </GestureDetector>

      <Modal
        visible={showMenu}
        transparent
        animationType="fade"
        onRequestClose={() => setShowMenu(false)}
      >
        <TouchableOpacity
          style={styles.backdrop}
          onPress={() => setShowMenu(false)}
          activeOpacity={1}
        >
          <View
            style={[
              styles.menu,
              {
                top: Math.min(menuPosition.y, 400),
                left: Math.min(menuPosition.x, 200),
              },
            ]}
          >
            {menuItems.map((item, index) => (
              <TouchableOpacity
                key={index}
                style={[
                  styles.menuItem,
                  index < menuItems.length - 1 && styles.menuItemBorder,
                ]}
                onPress={() => {
                  setShowMenu(false);
                  item.action();
                }}
              >
                <Text style={styles.menuIcon}>{item.icon}</Text>
                <Text style={[
                  styles.menuLabel,
                  item.destructive && styles.destructiveLabel,
                ]}>
                  {item.label}
                </Text>
              </TouchableOpacity>
            ))}
          </View>
        </TouchableOpacity>
      </Modal>
    </>
  );
};

const styles = StyleSheet.create({
  backdrop: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.2)',
  },
  menu: {
    position: 'absolute',
    backgroundColor: 'white',
    borderRadius: 12,
    width: 200,
    elevation: 8,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    overflow: 'hidden',
  },
  menuItem: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 14,
    gap: 12,
  },
  menuItemBorder: {
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: '#E0E0E0',
  },
  menuIcon: { fontSize: 20 },
  menuLabel: { fontSize: 15, flex: 1 },
  destructiveLabel: { color: '#F44336' },
});

export default LongPressMenu;
```

---

## 6. Combined Gestures {#combined-gestures}

### Gesture Composition

```typescript
// Gesture.Simultaneous - ทำงานพร้อมกัน
const simultaneousGesture = Gesture.Simultaneous(
  Gesture.Pan(),
  Gesture.Pinch(),
  Gesture.Rotation()
);

// Gesture.Exclusive - ทำงานอย่างใดอย่างหนึ่ง
const exclusiveGesture = Gesture.Exclusive(
  Gesture.Tap().numberOfTaps(2),  // ลองก่อน
  Gesture.Tap()                    // ถ้าไม่ใช่ double tap ก็ใช้ single
);

// Gesture.Race - อันไหนสำเร็จก่อน
const raceGesture = Gesture.Race(
  Gesture.Pan().minDistance(10),
  Gesture.LongPress()
);
```

### Image Transform (Pan + Pinch + Rotation)

```typescript
// src/components/TransformableImage.tsx
import React from 'react';
import { View, StyleSheet, Dimensions } from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

const TransformableImage: React.FC<{ imageUri: string }> = ({ imageUri }) => {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);
  const rotation = useSharedValue(0);
  const savedRotation = useSharedValue(0);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const savedX = useSharedValue(0);
  const savedY = useSharedValue(0);

  const pinch = Gesture.Pinch()
    .onUpdate((e) => {
      scale.value = savedScale.value * e.scale;
    })
    .onEnd(() => {
      savedScale.value = scale.value;
      if (scale.value < 1) {
        scale.value = withSpring(1);
        savedScale.value = 1;
      }
    });

  const rotate = Gesture.Rotation()
    .onUpdate((e) => {
      rotation.value = savedRotation.value + e.rotation;
    })
    .onEnd(() => {
      savedRotation.value = rotation.value;
    });

  const pan = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = savedX.value + e.translationX;
      translateY.value = savedY.value + e.translationY;
    })
    .onEnd(() => {
      savedX.value = translateX.value;
      savedY.value = translateY.value;
    });

  const doubleTap = Gesture.Tap()
    .numberOfTaps(2)
    .onEnd(() => {
      scale.value = withSpring(1);
      savedScale.value = 1;
      rotation.value = withSpring(0);
      savedRotation.value = 0;
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
      savedX.value = 0;
      savedY.value = 0;
    });

  const combined = Gesture.Simultaneous(
    pinch,
    rotate,
    pan,
    doubleTap
  );

  const animStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
      { rotate: `${rotation.value}rad` },
    ],
  }));

  return (
    <View style={styles.container}>
      <GestureDetector gesture={combined}>
        <Animated.Image
          source={{ uri: imageUri }}
          style={[styles.image, animStyle]}
          resizeMode="contain"
        />
      </GestureDetector>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#1a1a1a', overflow: 'hidden' },
  image: { width: SCREEN_WIDTH, height: SCREEN_WIDTH },
});

export default TransformableImage;
```

---

## 7. Workshop: Draggable Elements {#workshop}

```typescript
// src/components/DraggableList.tsx
import React, { useState, useCallback } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,
  useAnimatedReaction,
} from 'react-native-reanimated';

const ITEM_HEIGHT = 72;

interface DraggableItem {
  id: string;
  text: string;
  color: string;
}

interface DraggableListItemProps {
  item: DraggableItem;
  index: number;
  totalItems: number;
  onOrderChange: (fromIndex: number, toIndex: number) => void;
}

const DraggableListItem: React.FC<DraggableListItemProps> = ({
  item,
  index,
  totalItems,
  onOrderChange,
}) => {
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);
  const zIndex = useSharedValue(0);
  const opacity = useSharedValue(1);
  const isDragging = useSharedValue(false);

  const currentIndex = useSharedValue(index);
  const originalY = index * ITEM_HEIGHT;

  const updateOrder = (from: number, to: number) => {
    onOrderChange(from, to);
  };

  const pan = Gesture.Pan()
    .onBegin(() => {
      isDragging.value = true;
      scale.value = withSpring(1.05);
      zIndex.value = 100;
      opacity.value = withTiming(0.9);
    })
    .onUpdate((event) => {
      translateY.value = event.translationY;

      // คำนวณ index ใหม่
      const newIndex = Math.round(
        (originalY + event.translationY) / ITEM_HEIGHT
      );
      currentIndex.value = Math.max(0, Math.min(totalItems - 1, newIndex));
    })
    .onEnd(() => {
      isDragging.value = false;
      scale.value = withSpring(1);
      zIndex.value = 0;
      opacity.value = withTiming(1);

      // Snap to position
      const finalY = (currentIndex.value - index) * ITEM_HEIGHT;
      translateY.value = withSpring(finalY, { damping: 20 });

      if (currentIndex.value !== index) {
        runOnJS(updateOrder)(index, currentIndex.value);
      }
    });

  const style = useAnimatedStyle(() => ({
    transform: [
      { translateY: translateY.value },
      { scale: scale.value },
    ],
    zIndex: zIndex.value,
    opacity: opacity.value,
    elevation: isDragging.value ? 10 : 0,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: isDragging.value ? 5 : 0 },
    shadowOpacity: isDragging.value ? 0.3 : 0,
    shadowRadius: isDragging.value ? 10 : 0,
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[styles.item, { borderLeftColor: item.color }, style]}>
        <View style={[styles.colorDot, { backgroundColor: item.color }]} />
        <Text style={styles.itemText}>{item.text}</Text>
        <Text style={styles.dragHandle}>☰</Text>
      </Animated.View>
    </GestureDetector>
  );
};

const DraggableList: React.FC = () => {
  const [items, setItems] = useState<DraggableItem[]>([
    { id: '1', text: 'งานเร่งด่วน A', color: '#F44336' },
    { id: '2', text: 'งานทั่วไป B', color: '#FF9800' },
    { id: '3', text: 'งานปานกลาง C', color: '#FFC107' },
    { id: '4', text: 'งานไม่เร่งด่วน D', color: '#4CAF50' },
    { id: '5', text: 'งานยืดหยุ่น E', color: '#2196F3' },
  ]);

  const handleOrderChange = useCallback((fromIndex: number, toIndex: number) => {
    setItems(prev => {
      const newItems = [...prev];
      const [removed] = newItems.splice(fromIndex, 1);
      newItems.splice(toIndex, 0, removed);
      return newItems;
    });
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>ลำดับความสำคัญ</Text>
      <Text style={styles.subtitle}>ลากเพื่อเรียงลำดับ</Text>

      <View style={styles.list}>
        {items.map((item, index) => (
          <DraggableListItem
            key={item.id}
            item={item}
            index={index}
            totalItems={items.length}
            onOrderChange={handleOrderChange}
          />
        ))}
      </View>

      <View style={styles.orderDisplay}>
        <Text style={styles.orderTitle}>ลำดับปัจจุบัน:</Text>
        {items.map((item, index) => (
          <Text key={item.id} style={styles.orderItem}>
            {index + 1}. {item.text}
          </Text>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#F5F5F5' },
  title: { fontSize: 22, fontWeight: 'bold', marginBottom: 4 },
  subtitle: { color: '#666', marginBottom: 20 },
  list: { gap: 8 },
  item: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 12,
    borderLeftWidth: 4,
    gap: 12,
  },
  colorDot: {
    width: 12,
    height: 12,
    borderRadius: 6,
  },
  itemText: { flex: 1, fontSize: 15, fontWeight: '500' },
  dragHandle: { fontSize: 20, color: '#9E9E9E' },
  orderDisplay: {
    marginTop: 24,
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 12,
    gap: 4,
  },
  orderTitle: { fontWeight: 'bold', marginBottom: 8 },
  orderItem: { color: '#333', fontSize: 14 },
});

export default DraggableList;
```

---

## Tips และ Best Practices

### 1. Haptic Feedback

```typescript
import * as Haptics from 'expo-haptics';

// เพิ่ม haptic feedback ให้ gesture
const longPress = Gesture.LongPress()
  .onStart(() => {
    runOnJS(Haptics.impactAsync)(Haptics.ImpactFeedbackStyle.Medium);
  });
```

### 2. Gesture Priority

```typescript
// ใช้ requireExternalGestureToFail เพื่อจัดการ priority
const parentGesture = Gesture.Pan();
const childGesture = Gesture.Pan()
  .requireExternalGestureToFail(parentGesture);
```

### 3. ScrollView กับ PanGesture

```typescript
// แก้ปัญหา conflict ระหว่าง ScrollView และ Pan gesture
import { NativeViewGestureHandler } from 'react-native-gesture-handler';

const scrollRef = useRef(null);

<NativeViewGestureHandler ref={scrollRef}>
  <ScrollView>
    <GestureDetector gesture={panGesture.requireExternalGestureToFail(scrollRef)}>
      <View />
    </GestureDetector>
  </ScrollView>
</NativeViewGestureHandler>
```

### สรุป

- ใช้ Gesture Handler v2 API (GestureDetector + Gesture) แทน v1
- GestureHandlerRootView ต้อง wrap แอปทั้งหมด
- Gesture.Simultaneous สำหรับ gestures ที่ทำงานพร้อมกัน
- Gesture.Exclusive สำหรับ gestures ที่แข่งกัน
- เพิ่ม haptic feedback เพื่อประสบการณ์ที่ดีขึ้น
- จัดการ scroll view conflict อย่างระวัง
