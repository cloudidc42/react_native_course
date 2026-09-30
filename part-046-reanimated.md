# Part 046: Animations - Reanimated 2/3 ใน React Native

## สารบัญ
1. [แนะนำ Reanimated](#introduction)
2. [useSharedValue](#shared-value)
3. [useAnimatedStyle](#animated-style)
4. [withTiming, withSpring](#timing-spring)
5. [Worklets](#worklets)
6. [Complex Animations](#complex)
7. [Workshop: Swipe Cards](#workshop)

---

## 1. แนะนำ Reanimated {#introduction}

React Native Reanimated เป็น animation library ที่ทำงานบน UI thread โดยตรง ทำให้ได้ animation ที่ smooth กว่า Animated API มาก

### ข้อดีของ Reanimated

| Feature | Animated API | Reanimated 3 |
|---------|-------------|--------------|
| Performance | JS Thread | UI Thread |
| Gestures | PanResponder | Gesture Handler |
| Worklets | ไม่รองรับ | รองรับ |
| Complexity | ง่าย | ซับซ้อนกว่า |
| Bundle Size | เล็ก | ใหญ่กว่า |

### ติดตั้ง

```bash
npm install react-native-reanimated
cd ios && pod install

# babel.config.js ต้องเพิ่ม plugin
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    'react-native-reanimated/plugin', // ต้องอยู่ท้ายสุด
  ],
};
```

---

## 2. useSharedValue {#shared-value}

### การใช้งาน useSharedValue

```typescript
// src/components/SharedValueDemo.tsx
import React from 'react';
import { View, TouchableOpacity, Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withRepeat,
  withSequence,
  cancelAnimation,
  runOnJS,
} from 'react-native-reanimated';

const SharedValueDemo: React.FC = () => {
  // สร้าง Shared Value
  const opacity = useSharedValue(1);
  const scale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const rotation = useSharedValue(0);
  const backgroundColor = useSharedValue('#2196F3');

  // Animated Style
  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [
      { scale: scale.value },
      { translateX: translateX.value },
      { rotate: `${rotation.value}deg` },
    ],
  }));

  const runAnimation = () => {
    // เปลี่ยนค่าได้โดยตรงโดยไม่ต้องผ่าน Animated.timing
    opacity.value = withTiming(0.5, { duration: 500 });
    scale.value = withSpring(1.5);
    translateX.value = withTiming(100, { duration: 500 });
    rotation.value = withTiming(360, { duration: 500 });
  };

  const resetAnimation = () => {
    opacity.value = withTiming(1);
    scale.value = withSpring(1);
    translateX.value = withTiming(0);
    rotation.value = withTiming(0);
  };

  return (
    <View style={styles.container}>
      <Animated.View style={[styles.box, animatedStyle]}>
        <Text style={styles.boxText}>Reanimated!</Text>
      </Animated.View>

      <View style={styles.buttons}>
        <TouchableOpacity style={styles.btn} onPress={runAnimation}>
          <Text style={styles.btnText}>Animate</Text>
        </TouchableOpacity>
        <TouchableOpacity style={[styles.btn, { backgroundColor: '#4CAF50' }]} onPress={resetAnimation}>
          <Text style={styles.btnText}>Reset</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', gap: 24 },
  box: {
    width: 120,
    height: 120,
    backgroundColor: '#2196F3',
    borderRadius: 16,
    justifyContent: 'center',
    alignItems: 'center',
  },
  boxText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  buttons: { flexDirection: 'row', gap: 12 },
  btn: { backgroundColor: '#2196F3', paddingHorizontal: 20, paddingVertical: 12, borderRadius: 8 },
  btnText: { color: 'white', fontWeight: 'bold' },
});
```

### Derived Values

```typescript
import { useDerivedValue } from 'react-native-reanimated';

// สร้างค่าที่ derive มาจากค่าอื่น
const progress = useSharedValue(0);
const progressPercent = useDerivedValue(() => `${Math.round(progress.value * 100)}%`);
const color = useDerivedValue(() => {
  if (progress.value < 0.5) return '#F44336';
  if (progress.value < 0.8) return '#FF9800';
  return '#4CAF50';
});
```

---

## 3. useAnimatedStyle {#animated-style}

### useAnimatedStyle patterns

```typescript
// src/hooks/useAnimatedCard.ts
import {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

export function useAnimatedCard() {
  const pressed = useSharedValue(0);
  const hover = useSharedValue(0);

  const cardStyle = useAnimatedStyle(() => ({
    transform: [
      { scale: interpolate(pressed.value, [0, 1], [1, 0.95], Extrapolation.CLAMP) },
      {
        translateY: interpolate(hover.value, [0, 1], [0, -4], Extrapolation.CLAMP),
      },
    ],
    shadowOpacity: interpolate(hover.value, [0, 1], [0.1, 0.3], Extrapolation.CLAMP),
    elevation: interpolate(hover.value, [0, 1], [2, 8], Extrapolation.CLAMP),
  }));

  const handlePressIn = () => {
    pressed.value = withTiming(1, { duration: 100 });
  };

  const handlePressOut = () => {
    pressed.value = withTiming(0, { duration: 200 });
  };

  const handleHoverIn = () => {
    hover.value = withTiming(1, { duration: 200 });
  };

  const handleHoverOut = () => {
    hover.value = withTiming(0, { duration: 200 });
  };

  return { cardStyle, handlePressIn, handlePressOut, handleHoverIn, handleHoverOut };
}
```

### Scroll-based Animations

```typescript
// src/components/ScrollAnimations.tsx
import React from 'react';
import { Dimensions } from 'react-native';
import Animated, {
  useAnimatedScrollHandler,
  useAnimatedStyle,
  useSharedValue,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');
const HEADER_HEIGHT = 300;
const COLLAPSED_HEADER_HEIGHT = 60;

const ScrollAnimationsScreen: React.FC = () => {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  // Header collapse animation
  const headerStyle = useAnimatedStyle(() => ({
    height: interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT - COLLAPSED_HEADER_HEIGHT],
      [HEADER_HEIGHT, COLLAPSED_HEADER_HEIGHT],
      Extrapolation.CLAMP
    ),
  }));

  // Header image parallax
  const imageStyle = useAnimatedStyle(() => ({
    transform: [
      {
        translateY: interpolate(
          scrollY.value,
          [0, HEADER_HEIGHT],
          [0, -HEADER_HEIGHT / 2],
          Extrapolation.CLAMP
        ),
      },
    ],
  }));

  // Title opacity
  const titleOpacityStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      scrollY.value,
      [HEADER_HEIGHT - 100, HEADER_HEIGHT - 60],
      [0, 1],
      Extrapolation.CLAMP
    ),
  }));

  return (
    <Animated.View>
      {/* Collapsible Header */}
      <Animated.View style={[{ position: 'absolute', zIndex: 1 }, headerStyle]}>
        <Animated.Image
          source={{ uri: 'https://example.com/header.jpg' }}
          style={[{ width: '100%', height: HEADER_HEIGHT }, imageStyle]}
        />
        <Animated.Text style={[{ position: 'absolute', color: 'white' }, titleOpacityStyle]}>
          Header Title
        </Animated.Text>
      </Animated.View>

      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        contentContainerStyle={{ paddingTop: HEADER_HEIGHT }}
      >
        {/* Content */}
      </Animated.ScrollView>
    </Animated.View>
  );
};
```

---

## 4. withTiming, withSpring {#timing-spring}

### withTiming

```typescript
import {
  withTiming,
  withSpring,
  withDecay,
  withDelay,
  withRepeat,
  withSequence,
  Easing,
} from 'react-native-reanimated';

// withTiming พื้นฐาน
value.value = withTiming(100, {
  duration: 500,
  easing: Easing.out(Easing.quad),
});

// พร้อม callback
value.value = withTiming(100, { duration: 500 }, (finished) => {
  'worklet';
  if (finished) {
    console.log('Animation completed');
  }
});

// withDelay
value.value = withDelay(1000, withTiming(100, { duration: 500 }));

// withRepeat
value.value = withRepeat(
  withTiming(100, { duration: 500 }),
  -1, // -1 = infinite
  true // reverse: กลับมาค่าเดิม
);

// withSequence
value.value = withSequence(
  withTiming(100, { duration: 300 }),
  withTiming(50, { duration: 200 }),
  withTiming(80, { duration: 300 }),
);

// withSpring
value.value = withSpring(100, {
  damping: 15,
  stiffness: 150,
  overshootClamping: false,
  restDisplacementThreshold: 0.01,
  restSpeedThreshold: 2,
});
```

### Sequence Animations

```typescript
// src/components/SequenceAnimation.tsx
import React, { useEffect } from 'react';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withDelay,
  withSequence,
  withRepeat,
  Easing,
} from 'react-native-reanimated';

const HeartbeatAnimation: React.FC = () => {
  const scale = useSharedValue(1);

  useEffect(() => {
    scale.value = withRepeat(
      withSequence(
        withTiming(1.3, { duration: 100, easing: Easing.out(Easing.ease) }),
        withTiming(1, { duration: 100, easing: Easing.in(Easing.ease) }),
        withTiming(1.15, { duration: 80, easing: Easing.out(Easing.ease) }),
        withTiming(1, { duration: 80, easing: Easing.in(Easing.ease) }),
        withDelay(500, withTiming(1, { duration: 0 })),
      ),
      -1
    );
  }, []);

  const style = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return <Animated.Text style={style}>❤️</Animated.Text>;
};
```

---

## 5. Worklets {#worklets}

### การใช้งาน Worklets

```typescript
// Worklets คือ functions ที่ทำงานบน UI thread
// ต้องใส่ 'worklet' directive

// Function ปกติที่ทำงานบน UI thread
function clamp(value: number, min: number, max: number) {
  'worklet'; // ทำให้ function นี้ทำงาน UI thread
  return Math.min(Math.max(value, min), max);
}

// ใช้ใน useAnimatedStyle
const style = useAnimatedStyle(() => {
  const clampedOpacity = clamp(opacity.value, 0, 1); // ✅
  return { opacity: clampedOpacity };
});

// runOnJS - เรียก JS function จาก worklet
import { runOnJS } from 'react-native-reanimated';

const handleGestureEnd = () => {
  'worklet';
  // เรียก JS function จาก worklet
  runOnJS(setIsVisible)(true);
  runOnJS(console.log)('Gesture ended');
};

// runOnUI - เรียก worklet จาก JS thread
import { runOnUI } from 'react-native-reanimated';

const triggerAnimation = () => {
  runOnUI(() => {
    'worklet';
    value.value = withSpring(100);
  })();
};
```

### useAnimatedGestureHandler

```typescript
import { useAnimatedGestureHandler } from 'react-native-reanimated';
import { PanGestureHandler } from 'react-native-gesture-handler';

const DraggableItem: React.FC = () => {
  const x = useSharedValue(0);
  const y = useSharedValue(0);
  const scale = useSharedValue(1);

  const gestureHandler = useAnimatedGestureHandler({
    onStart: (_, context: { startX: number; startY: number }) => {
      'worklet';
      context.startX = x.value;
      context.startY = y.value;
      scale.value = withSpring(1.1);
    },
    onActive: (event, context) => {
      'worklet';
      x.value = context.startX + event.translationX;
      y.value = context.startY + event.translationY;
    },
    onEnd: () => {
      'worklet';
      scale.value = withSpring(1);
      x.value = withSpring(0);
      y.value = withSpring(0);
    },
  });

  const animStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: x.value },
      { translateY: y.value },
      { scale: scale.value },
    ],
  }));

  return (
    <PanGestureHandler onGestureEvent={gestureHandler}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: '#2196F3' }, animStyle]} />
    </PanGestureHandler>
  );
};
```

---

## 6. Complex Animations {#complex}

### Flip Card Animation

```typescript
// src/components/FlipCard.tsx
import React, { useState } from 'react';
import { View, TouchableWithoutFeedback, Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

interface FlipCardProps {
  front: React.ReactNode;
  back: React.ReactNode;
  width?: number;
  height?: number;
}

const FlipCard: React.FC<FlipCardProps> = ({
  front,
  back,
  width = 200,
  height = 280,
}) => {
  const [isFlipped, setIsFlipped] = useState(false);
  const rotation = useSharedValue(0);

  const flip = () => {
    if (isFlipped) {
      rotation.value = withTiming(0, { duration: 500 });
    } else {
      rotation.value = withTiming(180, { duration: 500 });
    }
    setIsFlipped(!isFlipped);
  };

  const frontStyle = useAnimatedStyle(() => {
    const rotateY = interpolate(
      rotation.value,
      [0, 90, 180],
      [0, -90, -180],
      Extrapolation.CLAMP
    );
    return {
      transform: [{ perspective: 1000 }, { rotateY: `${rotateY}deg` }],
      backfaceVisibility: 'hidden',
    };
  });

  const backStyle = useAnimatedStyle(() => {
    const rotateY = interpolate(
      rotation.value,
      [0, 90, 180],
      [180, 90, 0],
      Extrapolation.CLAMP
    );
    return {
      transform: [{ perspective: 1000 }, { rotateY: `${rotateY}deg` }],
      backfaceVisibility: 'hidden',
      position: 'absolute',
    };
  });

  return (
    <TouchableWithoutFeedback onPress={flip}>
      <View style={{ width, height }}>
        <Animated.View style={[styles.card, { width, height }, frontStyle]}>
          {front}
        </Animated.View>
        <Animated.View style={[styles.card, { width, height }, backStyle]}>
          {back}
        </Animated.View>
      </View>
    </TouchableWithoutFeedback>
  );
};

const styles = StyleSheet.create({
  card: {
    borderRadius: 16,
    overflow: 'hidden',
    elevation: 4,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
  },
});

export default FlipCard;
```

### Bottom Sheet Animation

```typescript
// src/components/AnimatedBottomSheet.tsx
import React, { useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Dimensions,
  TouchableWithoutFeedback,
} from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';
import { GestureDetector, Gesture } from 'react-native-gesture-handler';

const { height: SCREEN_HEIGHT } = Dimensions.get('window');
const MAX_TRANSLATE_Y = -SCREEN_HEIGHT + 50;

interface BottomSheetProps {
  children: React.ReactNode;
  snapPoints: number[];
  onClose?: () => void;
}

const AnimatedBottomSheet: React.FC<BottomSheetProps> = ({
  children,
  snapPoints,
  onClose,
}) => {
  const translateY = useSharedValue(0);
  const context = useSharedValue({ y: 0 });
  const isVisible = useSharedValue(false);

  const scrollTo = useCallback((destination: number) => {
    'worklet';
    isVisible.value = destination !== 0;
    translateY.value = withSpring(destination, { damping: 50 });
  }, []);

  const open = useCallback(() => {
    scrollTo(snapPoints[0]);
  }, [scrollTo, snapPoints]);

  const close = useCallback(() => {
    scrollTo(0);
    if (onClose) runOnJS(onClose)();
  }, [scrollTo, onClose]);

  const gesture = Gesture.Pan()
    .onStart(() => {
      context.value = { y: translateY.value };
    })
    .onUpdate((event) => {
      translateY.value = event.translationY + context.value.y;
      translateY.value = Math.max(translateY.value, MAX_TRANSLATE_Y);
    })
    .onEnd(() => {
      if (translateY.value > -SCREEN_HEIGHT / 3) {
        scrollTo(0);
        runOnJS(onClose || (() => {}))();
      } else if (translateY.value < -SCREEN_HEIGHT / 1.5) {
        scrollTo(MAX_TRANSLATE_Y);
      } else {
        scrollTo(snapPoints[0]);
      }
    });

  const bottomSheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  const backdropStyle = useAnimatedStyle(() => ({
    opacity: isVisible.value ? 0.5 : 0,
    pointerEvents: isVisible.value ? 'auto' : 'none',
  }));

  return (
    <>
      {/* Backdrop */}
      <Animated.View
        style={[
          StyleSheet.absoluteFillObject,
          { backgroundColor: 'black' },
          backdropStyle,
        ]}
      >
        <TouchableWithoutFeedback onPress={close}>
          <View style={{ flex: 1 }} />
        </TouchableWithoutFeedback>
      </Animated.View>

      {/* Bottom Sheet */}
      <GestureDetector gesture={gesture}>
        <Animated.View style={[styles.bottomSheet, bottomSheetStyle]}>
          <View style={styles.handle} />
          {children}
        </Animated.View>
      </GestureDetector>
    </>
  );
};

const styles = StyleSheet.create({
  bottomSheet: {
    height: SCREEN_HEIGHT,
    width: '100%',
    backgroundColor: 'white',
    position: 'absolute',
    top: SCREEN_HEIGHT,
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 16,
  },
  handle: {
    width: 40,
    height: 4,
    backgroundColor: '#E0E0E0',
    borderRadius: 2,
    alignSelf: 'center',
    marginBottom: 16,
  },
});

export default AnimatedBottomSheet;
```

---

## 7. Workshop: Swipe Cards {#workshop}

```typescript
// src/components/SwipeCards.tsx
import React, { useCallback, useState } from 'react';
import {
  View,
  Text,
  Image,
  StyleSheet,
  Dimensions,
  TouchableOpacity,
} from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Icon from 'react-native-vector-icons/MaterialIcons';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');
const SWIPE_THRESHOLD = SCREEN_WIDTH * 0.4;
const ROTATION_FACTOR = 15;

interface CardData {
  id: string;
  name: string;
  age: number;
  imageUrl: string;
  bio: string;
  tags: string[];
  distance: number;
}

interface SwipeCardProps {
  card: CardData;
  onSwipeLeft: (id: string) => void;
  onSwipeRight: (id: string) => void;
  onSuperLike: (id: string) => void;
  isTop: boolean;
  index: number;
}

const SwipeCard: React.FC<SwipeCardProps> = ({
  card,
  onSwipeLeft,
  onSwipeRight,
  onSuperLike,
  isTop,
  index,
}) => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(0.95 - index * 0.03);

  const gesture = Gesture.Pan()
    .enabled(isTop)
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY;
    })
    .onEnd((event) => {
      if (Math.abs(translateX.value) > SWIPE_THRESHOLD) {
        // Swipe left or right
        const direction = translateX.value > 0 ? 1 : -1;
        translateX.value = withTiming(
          direction * SCREEN_WIDTH * 1.5,
          { duration: 300 },
          () => {
            'worklet';
            if (direction > 0) runOnJS(onSwipeRight)(card.id);
            else runOnJS(onSwipeLeft)(card.id);
          }
        );
      } else if (translateY.value < -SCREEN_HEIGHT * 0.25) {
        // Super like (swipe up)
        translateY.value = withTiming(
          -SCREEN_HEIGHT * 1.5,
          { duration: 300 },
          () => {
            'worklet';
            runOnJS(onSuperLike)(card.id);
          }
        );
      } else {
        // Snap back
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
      }
    });

  const cardStyle = useAnimatedStyle(() => {
    const rotation = interpolate(
      translateX.value,
      [-SCREEN_WIDTH / 2, 0, SCREEN_WIDTH / 2],
      [-ROTATION_FACTOR, 0, ROTATION_FACTOR],
      Extrapolation.CLAMP
    );

    return {
      transform: [
        { translateX: translateX.value },
        { translateY: translateY.value },
        { rotate: `${rotation}deg` },
        { scale: scale.value },
      ],
    };
  });

  // Like/Nope indicator
  const likeOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [0, SWIPE_THRESHOLD],
      [0, 1],
      Extrapolation.CLAMP
    ),
  }));

  const nopeOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [-SWIPE_THRESHOLD, 0],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));

  const superLikeOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateY.value,
      [-SCREEN_HEIGHT * 0.25, 0],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.card, cardStyle]}>
        <Image source={{ uri: card.imageUrl }} style={styles.image} />

        {/* Indicators */}
        <Animated.View style={[styles.likeStamp, likeOpacity]}>
          <Text style={styles.likeText}>LIKE</Text>
        </Animated.View>

        <Animated.View style={[styles.nopeStamp, nopeOpacity]}>
          <Text style={styles.nopeText}>NOPE</Text>
        </Animated.View>

        <Animated.View style={[styles.superLikeStamp, superLikeOpacity]}>
          <Text style={styles.superLikeText}>SUPER LIKE</Text>
        </Animated.View>

        {/* Card Info */}
        <View style={styles.cardInfo}>
          <Text style={styles.name}>{card.name}, {card.age}</Text>
          <Text style={styles.distance}>{card.distance} กม. จากคุณ</Text>
          <Text style={styles.bio} numberOfLines={2}>{card.bio}</Text>

          <View style={styles.tags}>
            {card.tags.slice(0, 3).map(tag => (
              <View key={tag} style={styles.tag}>
                <Text style={styles.tagText}>{tag}</Text>
              </View>
            ))}
          </View>
        </View>
      </Animated.View>
    </GestureDetector>
  );
};

// Main Swipe Cards Component
const SwipeCards: React.FC = () => {
  const [cards, setCards] = useState<CardData[]>([
    {
      id: '1',
      name: 'สมใจ',
      age: 25,
      imageUrl: 'https://picsum.photos/400/600?random=1',
      bio: 'ชอบท่องเที่ยว ดูหนัง และกินอาหารอร่อย 🌍',
      tags: ['ท่องเที่ยว', 'ดูหนัง', 'อาหาร'],
      distance: 2.5,
    },
    {
      id: '2',
      name: 'มานะ',
      age: 28,
      imageUrl: 'https://picsum.photos/400/600?random=2',
      bio: 'นักดนตรีสมัครเล่น ชอบวิ่งตอนเช้า 🎸',
      tags: ['ดนตรี', 'วิ่ง', 'กีฬา'],
      distance: 5.1,
    },
    {
      id: '3',
      name: 'วิมล',
      age: 23,
      imageUrl: 'https://picsum.photos/400/600?random=3',
      bio: 'Barista ประจำคาเฟ่ ชอบถ่ายรูป 📷☕',
      tags: ['กาแฟ', 'ถ่ายรูป', 'ศิลปะ'],
      distance: 1.8,
    },
  ]);

  const [matches, setMatches] = useState<string[]>([]);
  const [stats, setStats] = useState({ likes: 0, nopes: 0, superLikes: 0 });

  const handleSwipeLeft = useCallback((id: string) => {
    setCards(prev => prev.filter(c => c.id !== id));
    setStats(prev => ({ ...prev, nopes: prev.nopes + 1 }));
  }, []);

  const handleSwipeRight = useCallback((id: string) => {
    setCards(prev => prev.filter(c => c.id !== id));
    setMatches(prev => [...prev, id]);
    setStats(prev => ({ ...prev, likes: prev.likes + 1 }));
  }, []);

  const handleSuperLike = useCallback((id: string) => {
    setCards(prev => prev.filter(c => c.id !== id));
    setMatches(prev => [...prev, id]);
    setStats(prev => ({ ...prev, superLikes: prev.superLikes + 1 }));
  }, []);

  return (
    <View style={styles.container}>
      {/* Stats */}
      <View style={styles.stats}>
        <Text style={styles.statItem}>👎 {stats.nopes}</Text>
        <Text style={styles.statItem}>❤️ {stats.likes}</Text>
        <Text style={styles.statItem}>⭐ {stats.superLikes}</Text>
      </View>

      {/* Cards Stack */}
      <View style={styles.cardsContainer}>
        {cards.length === 0 ? (
          <View style={styles.emptyState}>
            <Text style={styles.emptyIcon}>🎉</Text>
            <Text style={styles.emptyTitle}>ไม่มีการ์ดแล้ว!</Text>
            <Text style={styles.emptySubtitle}>คุณดูทุกคนในพื้นที่แล้ว</Text>
          </View>
        ) : (
          [...cards].reverse().map((card, index) => (
            <SwipeCard
              key={card.id}
              card={card}
              onSwipeLeft={handleSwipeLeft}
              onSwipeRight={handleSwipeRight}
              onSuperLike={handleSuperLike}
              isTop={index === cards.length - 1}
              index={cards.length - 1 - index}
            />
          ))
        )}
      </View>

      {/* Action Buttons */}
      {cards.length > 0 && (
        <View style={styles.actionButtons}>
          <TouchableOpacity
            style={[styles.actionBtn, styles.nopeBtn]}
            onPress={() => handleSwipeLeft(cards[cards.length - 1].id)}
          >
            <Icon name="close" size={32} color="#F44336" />
          </TouchableOpacity>

          <TouchableOpacity
            style={[styles.actionBtn, styles.superLikeBtn]}
            onPress={() => handleSuperLike(cards[cards.length - 1].id)}
          >
            <Icon name="star" size={28} color="#2196F3" />
          </TouchableOpacity>

          <TouchableOpacity
            style={[styles.actionBtn, styles.likeBtn]}
            onPress={() => handleSwipeRight(cards[cards.length - 1].id)}
          >
            <Icon name="favorite" size={32} color="#4CAF50" />
          </TouchableOpacity>
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  stats: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    padding: 16,
    backgroundColor: 'white',
  },
  statItem: { fontSize: 16, fontWeight: 'bold' },
  cardsContainer: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 16,
  },
  card: {
    position: 'absolute',
    width: SCREEN_WIDTH - 32,
    height: SCREEN_HEIGHT * 0.65,
    borderRadius: 20,
    overflow: 'hidden',
    backgroundColor: 'white',
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.2,
    shadowRadius: 6,
  },
  image: { width: '100%', height: '65%' },
  likeStamp: {
    position: 'absolute',
    top: 40,
    left: 20,
    borderWidth: 3,
    borderColor: '#4CAF50',
    borderRadius: 8,
    padding: 8,
    transform: [{ rotate: '-20deg' }],
  },
  likeText: { color: '#4CAF50', fontWeight: 'bold', fontSize: 32 },
  nopeStamp: {
    position: 'absolute',
    top: 40,
    right: 20,
    borderWidth: 3,
    borderColor: '#F44336',
    borderRadius: 8,
    padding: 8,
    transform: [{ rotate: '20deg' }],
  },
  nopeText: { color: '#F44336', fontWeight: 'bold', fontSize: 32 },
  superLikeStamp: {
    position: 'absolute',
    top: 40,
    alignSelf: 'center',
    borderWidth: 3,
    borderColor: '#2196F3',
    borderRadius: 8,
    padding: 8,
  },
  superLikeText: { color: '#2196F3', fontWeight: 'bold', fontSize: 24 },
  cardInfo: { padding: 16 },
  name: { fontSize: 24, fontWeight: 'bold', marginBottom: 4 },
  distance: { color: '#9E9E9E', fontSize: 13, marginBottom: 8 },
  bio: { color: '#666', fontSize: 14, lineHeight: 20, marginBottom: 12 },
  tags: { flexDirection: 'row', gap: 8 },
  tag: { backgroundColor: '#E3F2FD', paddingHorizontal: 10, paddingVertical: 4, borderRadius: 16 },
  tagText: { color: '#1565C0', fontSize: 12, fontWeight: '500' },
  emptyState: { alignItems: 'center', gap: 12 },
  emptyIcon: { fontSize: 64 },
  emptyTitle: { fontSize: 24, fontWeight: 'bold' },
  emptySubtitle: { color: '#666', fontSize: 16 },
  actionButtons: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 24,
    padding: 20,
    backgroundColor: 'white',
  },
  actionBtn: {
    width: 60,
    height: 60,
    borderRadius: 30,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'white',
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 4,
  },
  nopeBtn: { borderWidth: 2, borderColor: '#F44336' },
  likeBtn: { borderWidth: 2, borderColor: '#4CAF50' },
  superLikeBtn: { width: 50, height: 50, borderRadius: 25, borderWidth: 2, borderColor: '#2196F3' },
});

export default SwipeCards;
```

---

## Tips และ Best Practices

### 1. เลือกใช้ Reanimated หรือ Animated API

```typescript
// ใช้ Animated API เมื่อ:
// - Animation ง่ายๆ เช่น fade, simple transform
// - ไม่ต้องการ gesture integration ที่ซับซ้อน

// ใช้ Reanimated เมื่อ:
// - ต้องการ performance สูง
// - มี gesture handler ที่ซับซ้อน
// - ต้องการ layout animations
// - มี animations หลายอย่างพร้อมกัน
```

### 2. Layout Animations

```typescript
import { Layout, FadeIn, FadeOut, SlideInRight } from 'react-native-reanimated';

const ListItem = ({ item }) => (
  <Animated.View
    entering={FadeIn.duration(300)}
    exiting={FadeOut.duration(300)}
    layout={Layout.springify()}
  >
    <Text>{item.name}</Text>
  </Animated.View>
);
```

### สรุป

- Reanimated ทำงานบน UI thread ทำให้ animation smooth กว่า
- ใช้ worklets สำหรับ logic ที่ต้องทำงานบน UI thread
- runOnJS เมื่อต้องการเรียก JS function จาก worklet
- withSpring สำหรับ natural feel, withTiming สำหรับ precise control
- Layout animations ช่วยทำให้การเพิ่ม/ลบ element มี animation โดยอัตโนมัติ
