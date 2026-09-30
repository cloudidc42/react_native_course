# Part 045: Animations - Animated API ใน React Native

## สารบัญ
1. [แนะนำ Animated API](#introduction)
2. [Animated.Value](#animated-value)
3. [Animated.timing, spring, decay](#timing-spring-decay)
4. [Animated.parallel, sequence, stagger](#parallel-sequence)
5. [useNativeDriver](#native-driver)
6. [Interpolation](#interpolation)
7. [Workshop: Animated Loading Screen](#workshop)

---

## 1. แนะนำ Animated API {#introduction}

Animated API เป็น built-in animation system ของ React Native ที่ช่วยสร้าง animations ที่ smooth และประสิทธิภาพสูง

### หลักการทำงาน

```
Animated API
├── Values (ตัวแปร animation)
│   ├── Animated.Value (single value)
│   └── Animated.ValueXY (X and Y)
├── Animations (ประเภท animation)
│   ├── timing (เส้นตรง)
│   ├── spring (เด้ง)
│   └── decay (ค่อยๆ ช้าลง)
└── Combinations
    ├── parallel (พร้อมกัน)
    ├── sequence (ทีละอย่าง)
    └── stagger (ทยอย)
```

### Animated Components

```typescript
// Component ที่รองรับ Animation
Animated.View
Animated.Text
Animated.Image
Animated.ScrollView
Animated.FlatList
Animated.SectionList

// สร้าง Animated Component เอง
const AnimatedTouchable = Animated.createAnimatedComponent(TouchableOpacity);
const AnimatedTextInput = Animated.createAnimatedComponent(TextInput);
```

---

## 2. Animated.Value {#animated-value}

### การสร้างและใช้งาน Animated.Value

```typescript
// src/components/AnimatedDemo.tsx
import React, { useRef, useEffect } from 'react';
import { Animated, View, StyleSheet, Text } from 'react-native';

const BasicAnimatedDemo: React.FC = () => {
  // สร้าง Animated.Value
  const fadeAnim = useRef(new Animated.Value(0)).current;
  const scaleAnim = useRef(new Animated.Value(1)).current;
  const translateX = useRef(new Animated.Value(0)).current;
  const positionXY = useRef(new Animated.ValueXY({ x: 0, y: 0 })).current;

  useEffect(() => {
    // เริ่ม animation เมื่อ component mount
    Animated.timing(fadeAnim, {
      toValue: 1,
      duration: 1000,
      useNativeDriver: true,
    }).start();
  }, []);

  // อ่านค่าปัจจุบัน (ไม่แนะนำใช้บ่อย)
  const currentValue = (fadeAnim as any)._value;

  // Reset animation
  const resetAnimation = () => {
    fadeAnim.setValue(0);
    scaleAnim.setValue(1);
  };

  // Track ค่าเปลี่ยนแปลง
  useEffect(() => {
    const id = fadeAnim.addListener(({ value }) => {
      console.log('Current opacity:', value);
    });
    return () => fadeAnim.removeListener(id);
  }, []);

  return (
    <View style={styles.container}>
      <Animated.View
        style={[
          styles.box,
          {
            opacity: fadeAnim,
            transform: [
              { scale: scaleAnim },
              { translateX: translateX },
            ],
          },
        ]}
      >
        <Text style={styles.boxText}>Animated Box</Text>
      </Animated.View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  box: {
    width: 100,
    height: 100,
    backgroundColor: '#2196F3',
    borderRadius: 12,
    justifyContent: 'center',
    alignItems: 'center',
  },
  boxText: { color: 'white', fontWeight: 'bold' },
});
```

### Animated.ValueXY สำหรับการเคลื่อนที่ 2D

```typescript
// src/hooks/useDraggable.ts
import { useRef } from 'react';
import { Animated, PanResponder } from 'react-native';

export function useDraggable() {
  const position = useRef(new Animated.ValueXY()).current;

  const panResponder = useRef(
    PanResponder.create({
      onStartShouldSetPanResponder: () => true,
      onMoveShouldSetPanResponder: () => true,

      onPanResponderGrant: () => {
        position.setOffset({
          x: (position.x as any)._value,
          y: (position.y as any)._value,
        });
        position.setValue({ x: 0, y: 0 });
      },

      onPanResponderMove: Animated.event(
        [null, { dx: position.x, dy: position.y }],
        { useNativeDriver: false }
      ),

      onPanResponderRelease: () => {
        position.flattenOffset();
        
        // Spring กลับไปจุดเริ่มต้น
        Animated.spring(position, {
          toValue: { x: 0, y: 0 },
          useNativeDriver: false,
          bounciness: 10,
        }).start();
      },
    })
  ).current;

  return { position, panResponder };
}
```

---

## 3. Animated.timing, spring, decay {#timing-spring-decay}

### Animated.timing

```typescript
// src/animations/timingExamples.ts
import { Animated, Easing } from 'react-native';

// Linear
const linearAnimation = (value: Animated.Value) =>
  Animated.timing(value, {
    toValue: 1,
    duration: 500,
    easing: Easing.linear,
    useNativeDriver: true,
  });

// Ease In/Out
const easeAnimation = (value: Animated.Value) =>
  Animated.timing(value, {
    toValue: 1,
    duration: 500,
    easing: Easing.inOut(Easing.ease),
    useNativeDriver: true,
  });

// Bounce
const bounceAnimation = (value: Animated.Value) =>
  Animated.timing(value, {
    toValue: 1,
    duration: 800,
    easing: Easing.bounce,
    useNativeDriver: true,
  });

// Elastic
const elasticAnimation = (value: Animated.Value) =>
  Animated.timing(value, {
    toValue: 1,
    duration: 1000,
    easing: Easing.elastic(2),
    useNativeDriver: true,
  });

// Custom Bezier Curve
const bezierAnimation = (value: Animated.Value) =>
  Animated.timing(value, {
    toValue: 1,
    duration: 600,
    easing: Easing.bezier(0.25, 0.1, 0.25, 1),
    useNativeDriver: true,
  });
```

### Animated.spring

```typescript
// Spring configurations
const springConfigs = {
  // Wobbly (ขยับมาก)
  wobbly: {
    stiffness: 180,
    damping: 12,
    useNativeDriver: true,
  },
  
  // Gentle (นุ่มนวล)
  gentle: {
    stiffness: 120,
    damping: 14,
    useNativeDriver: true,
  },
  
  // Stiff (แข็ง)
  stiff: {
    stiffness: 400,
    damping: 30,
    useNativeDriver: true,
  },
  
  // Preset
  preset: {
    tension: 40,
    friction: 7,
    useNativeDriver: true,
  },
};

function springToValue(value: Animated.Value, toValue: number, config = springConfigs.gentle) {
  Animated.spring(value, {
    toValue,
    ...config,
  }).start();
}
```

### Animated.decay

```typescript
// Decay (ค่อยๆ ช้าลงจากความเร็วเริ่มต้น)
function throwAnimation(velocity: { x: number; y: number }, position: Animated.ValueXY) {
  Animated.decay(position, {
    velocity,
    deceleration: 0.997,
    useNativeDriver: true,
  }).start(({ finished }) => {
    if (finished) {
      console.log('Decay animation completed');
    }
  });
}
```

---

## 4. Animated.parallel, sequence, stagger {#parallel-sequence}

```typescript
// src/components/CombinedAnimations.tsx
import React, { useRef, useEffect } from 'react';
import { Animated, View, StyleSheet, TouchableOpacity, Text } from 'react-native';

const CombinedAnimations: React.FC = () => {
  const animations = Array.from({ length: 5 }, () => ({
    opacity: useRef(new Animated.Value(0)).current,
    scale: useRef(new Animated.Value(0.5)).current,
    translateY: useRef(new Animated.Value(50)).current,
  }));

  // Parallel - ทำพร้อมกัน
  const runParallel = () => {
    const anim = animations[0];
    Animated.parallel([
      Animated.timing(anim.opacity, {
        toValue: 1,
        duration: 500,
        useNativeDriver: true,
      }),
      Animated.spring(anim.scale, {
        toValue: 1,
        useNativeDriver: true,
      }),
      Animated.timing(anim.translateY, {
        toValue: 0,
        duration: 500,
        useNativeDriver: true,
      }),
    ]).start();
  };

  // Sequence - ทีละอย่าง
  const runSequence = () => {
    const anim = animations[1];
    Animated.sequence([
      Animated.timing(anim.opacity, {
        toValue: 1,
        duration: 300,
        useNativeDriver: true,
      }),
      Animated.spring(anim.scale, {
        toValue: 1,
        useNativeDriver: true,
      }),
      Animated.timing(anim.translateY, {
        toValue: 0,
        duration: 300,
        useNativeDriver: true,
      }),
    ]).start();
  };

  // Stagger - ทยอยทีละอย่าง
  const runStagger = () => {
    Animated.stagger(
      100, // delay ระหว่าง animations
      animations.map(anim =>
        Animated.parallel([
          Animated.timing(anim.opacity, {
            toValue: 1,
            duration: 400,
            useNativeDriver: true,
          }),
          Animated.spring(anim.scale, {
            toValue: 1,
            useNativeDriver: true,
          }),
          Animated.timing(anim.translateY, {
            toValue: 0,
            duration: 400,
            useNativeDriver: true,
          }),
        ])
      )
    ).start();
  };

  // Reset
  const resetAnimations = () => {
    animations.forEach(anim => {
      anim.opacity.setValue(0);
      anim.scale.setValue(0.5);
      anim.translateY.setValue(50);
    });
  };

  const colors = ['#F44336', '#2196F3', '#4CAF50', '#FF9800', '#9C27B0'];

  return (
    <View style={styles.container}>
      <View style={styles.items}>
        {animations.map((anim, index) => (
          <Animated.View
            key={index}
            style={[
              styles.item,
              { backgroundColor: colors[index] },
              {
                opacity: anim.opacity,
                transform: [
                  { scale: anim.scale },
                  { translateY: anim.translateY },
                ],
              },
            ]}
          >
            <Text style={styles.itemText}>{index + 1}</Text>
          </Animated.View>
        ))}
      </View>

      <View style={styles.buttons}>
        <TouchableOpacity style={styles.btn} onPress={resetAnimations}>
          <Text style={styles.btnText}>Reset</Text>
        </TouchableOpacity>
        <TouchableOpacity style={[styles.btn, { backgroundColor: '#F44336' }]} onPress={runParallel}>
          <Text style={styles.btnText}>Parallel</Text>
        </TouchableOpacity>
        <TouchableOpacity style={[styles.btn, { backgroundColor: '#4CAF50' }]} onPress={runSequence}>
          <Text style={styles.btnText}>Sequence</Text>
        </TouchableOpacity>
        <TouchableOpacity style={[styles.btn, { backgroundColor: '#FF9800' }]} onPress={runStagger}>
          <Text style={styles.btnText}>Stagger</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', padding: 20 },
  items: {
    flexDirection: 'row',
    justifyContent: 'center',
    gap: 12,
    marginBottom: 40,
  },
  item: {
    width: 50,
    height: 50,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  itemText: { color: 'white', fontWeight: 'bold' },
  buttons: { flexDirection: 'row', flexWrap: 'wrap', gap: 8, justifyContent: 'center' },
  btn: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 16,
    paddingVertical: 10,
    borderRadius: 8,
  },
  btnText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
});

export default CombinedAnimations;
```

---

## 5. useNativeDriver {#native-driver}

### Native Driver vs JavaScript Driver

```typescript
// ❌ ไม่ใช้ Native Driver - ช้ากว่า
Animated.timing(value, {
  toValue: 1,
  duration: 300,
  useNativeDriver: false, // ทำงานบน JS thread
}).start();

// ✅ ใช้ Native Driver - เร็วกว่า
Animated.timing(value, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // ทำงานบน UI thread โดยตรง
}).start();

// Properties ที่รองรับ Native Driver:
// ✅ opacity, transform (translateX, translateY, scale, rotate)
// ❌ width, height, backgroundColor, top, left, margin, padding
// ❌ flex, position, borderRadius (บาง platform)

// ตัวอย่างที่ถูกต้อง
const animStyle = {
  opacity: fadeAnim,              // ✅
  transform: [
    { translateX: slideAnim },    // ✅
    { scale: scaleAnim },         // ✅
    { rotate: rotateAnim.interpolate({
        inputRange: [0, 1],
        outputRange: ['0deg', '360deg'],
    })},                           // ✅
  ],
};

// ตัวอย่างที่ใช้ Native Driver ไม่ได้
const nonNativeStyle = {
  width: widthAnim,               // ❌ ต้องใช้ useNativeDriver: false
  backgroundColor: colorAnim,     // ❌
};
```

### Performance Tips

```typescript
// 1. ใช้ shouldRasterizeIOS และ renderToHardwareTextureAndroid
<Animated.View
  style={[styles.box, animatedStyle]}
  shouldRasterizeIOS={true}
  renderToHardwareTextureAndroid={true}
>
  <ComplexComponent />
</Animated.View>

// 2. หลีกเลี่ยงการ re-render ระหว่าง animation
// ใช้ useRef แทน useState สำหรับ Animated.Value
const goodWay = useRef(new Animated.Value(0)).current;    // ✅
const badWay = useState(new Animated.Value(0))[0];       // ❌

// 3. หยุด animation ก่อน unmount
useEffect(() => {
  const anim = Animated.loop(
    Animated.sequence([...])
  );
  anim.start();
  return () => anim.stop(); // ✅ หยุดเมื่อ unmount
}, []);
```

---

## 6. Interpolation {#interpolation}

### การใช้ Interpolation

```typescript
// src/components/InterpolationExamples.tsx
import React, { useRef, useEffect } from 'react';
import { Animated, View, StyleSheet } from 'react-native';

const InterpolationExamples: React.FC = () => {
  const anim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.timing(anim, {
        toValue: 1,
        duration: 2000,
        useNativeDriver: false,
      })
    ).start();
  }, []);

  // Color interpolation
  const backgroundColor = anim.interpolate({
    inputRange: [0, 0.5, 1],
    outputRange: ['#FF5722', '#2196F3', '#4CAF50'],
  });

  // Rotation
  const rotation = anim.interpolate({
    inputRange: [0, 1],
    outputRange: ['0deg', '360deg'],
  });

  // Scale with extrapolation
  const scale = anim.interpolate({
    inputRange: [0, 0.5, 1],
    outputRange: [1, 1.5, 1],
    extrapolate: 'clamp', // ไม่ให้ค่าเกิน range
  });

  // Width (0-100%)
  const widthPercent = anim.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '100%'],
  });

  // Opacity based on scroll
  const scrollAnim = useRef(new Animated.Value(0)).current;
  const headerOpacity = scrollAnim.interpolate({
    inputRange: [0, 100],
    outputRange: [0, 1],
    extrapolate: 'clamp',
  });

  const headerTranslateY = scrollAnim.interpolate({
    inputRange: [0, 100],
    outputRange: [-60, 0],
    extrapolate: 'clamp',
  });

  return (
    <View style={styles.container}>
      {/* Color animation */}
      <Animated.View style={[styles.box, { backgroundColor }]}>
      </Animated.View>

      {/* Rotation */}
      <Animated.View style={[styles.box, { transform: [{ rotate: rotation }] }]}>
      </Animated.View>

      {/* Scale */}
      <Animated.View style={[styles.box, { transform: [{ scale }] }]}>
      </Animated.View>

      {/* Progress Bar */}
      <View style={styles.progressContainer}>
        <Animated.View style={[styles.progressBar, { width: widthPercent }]} />
      </View>

      {/* Scroll-based Parallax Header */}
      <Animated.View
        style={[
          styles.header,
          {
            opacity: headerOpacity,
            transform: [{ translateY: headerTranslateY }],
          },
        ]}
      >
      </Animated.View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, gap: 20, padding: 20 },
  box: { width: 80, height: 80, borderRadius: 12 },
  progressContainer: {
    height: 8,
    backgroundColor: '#E0E0E0',
    borderRadius: 4,
    overflow: 'hidden',
  },
  progressBar: { height: '100%', backgroundColor: '#2196F3', borderRadius: 4 },
  header: { height: 60, backgroundColor: '#2196F3', position: 'absolute', top: 0, left: 0, right: 0 },
});
```

---

## 7. Workshop: Animated Loading Screen {#workshop}

```typescript
// src/screens/LoadingScreen.tsx
import React, { useRef, useEffect } from 'react';
import { Animated, View, StyleSheet, Text, Dimensions } from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

// Dot loader
const DotLoader: React.FC = () => {
  const dots = Array.from({ length: 3 }, () => useRef(new Animated.Value(0)).current);

  useEffect(() => {
    const animations = dots.map((dot, index) =>
      Animated.loop(
        Animated.sequence([
          Animated.delay(index * 200),
          Animated.timing(dot, {
            toValue: 1,
            duration: 400,
            useNativeDriver: true,
          }),
          Animated.timing(dot, {
            toValue: 0,
            duration: 400,
            useNativeDriver: true,
          }),
        ])
      )
    );

    Animated.parallel(animations).start();
    return () => animations.forEach(a => a.stop());
  }, []);

  return (
    <View style={dotStyles.container}>
      {dots.map((dot, index) => (
        <Animated.View
          key={index}
          style={[
            dotStyles.dot,
            {
              opacity: dot,
              transform: [
                {
                  translateY: dot.interpolate({
                    inputRange: [0, 1],
                    outputRange: [0, -10],
                  }),
                },
              ],
            },
          ]}
        />
      ))}
    </View>
  );
};

const dotStyles = StyleSheet.create({
  container: { flexDirection: 'row', gap: 8 },
  dot: { width: 12, height: 12, borderRadius: 6, backgroundColor: '#2196F3' },
});

// Skeleton loader
const SkeletonLoader: React.FC<{ width?: number; height?: number; borderRadius?: number }> = ({
  width = '100%',
  height = 20,
  borderRadius = 4,
}) => {
  const shimmer = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.timing(shimmer, {
        toValue: 1,
        duration: 1500,
        useNativeDriver: false,
      })
    ).start();
  }, []);

  const shimmerTranslateX = shimmer.interpolate({
    inputRange: [0, 1],
    outputRange: [-SCREEN_WIDTH, SCREEN_WIDTH],
  });

  return (
    <View style={[skeletonStyles.container, { width: width as any, height, borderRadius }]}>
      <Animated.View
        style={[
          skeletonStyles.shimmer,
          { transform: [{ translateX: shimmerTranslateX }] },
        ]}
      />
    </View>
  );
};

const skeletonStyles = StyleSheet.create({
  container: {
    backgroundColor: '#E0E0E0',
    overflow: 'hidden',
  },
  shimmer: {
    width: '50%',
    height: '100%',
    backgroundColor: 'rgba(255,255,255,0.6)',
    position: 'absolute',
  },
});

// Pulse animation
const PulseAnimation: React.FC<{ size?: number; color?: string }> = ({
  size = 80,
  color = '#2196F3',
}) => {
  const pulseAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.sequence([
        Animated.timing(pulseAnim, {
          toValue: 1,
          duration: 1000,
          useNativeDriver: true,
        }),
        Animated.timing(pulseAnim, {
          toValue: 0,
          duration: 1000,
          useNativeDriver: true,
        }),
      ])
    ).start();
  }, []);

  const pulseScale = pulseAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [1, 1.3],
  });

  const pulseOpacity = pulseAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [0.8, 0],
  });

  return (
    <View style={{ width: size, height: size, justifyContent: 'center', alignItems: 'center' }}>
      {/* Pulse ring */}
      <Animated.View
        style={{
          position: 'absolute',
          width: size,
          height: size,
          borderRadius: size / 2,
          backgroundColor: color,
          opacity: pulseOpacity,
          transform: [{ scale: pulseScale }],
        }}
      />
      {/* Center dot */}
      <View
        style={{
          width: size * 0.5,
          height: size * 0.5,
          borderRadius: size * 0.25,
          backgroundColor: color,
        }}
      />
    </View>
  );
};

// Spinner
const SpinnerAnimation: React.FC<{ size?: number; color?: string }> = ({
  size = 40,
  color = '#2196F3',
}) => {
  const spinAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.timing(spinAnim, {
        toValue: 1,
        duration: 1000,
        useNativeDriver: true,
      })
    ).start();
  }, []);

  const spin = spinAnim.interpolate({
    inputRange: [0, 1],
    outputRange: ['0deg', '360deg'],
  });

  return (
    <Animated.View
      style={{
        width: size,
        height: size,
        borderRadius: size / 2,
        borderWidth: 3,
        borderColor: '#E0E0E0',
        borderTopColor: color,
        transform: [{ rotate: spin }],
      }}
    />
  );
};

// Main Loading Screen
const AnimatedLoadingScreen: React.FC<{ onFinish: () => void }> = ({ onFinish }) => {
  const logoScale = useRef(new Animated.Value(0)).current;
  const logoOpacity = useRef(new Animated.Value(0)).current;
  const textOpacity = useRef(new Animated.Value(0)).current;
  const progressAnim = useRef(new Animated.Value(0)).current;
  const containerOpacity = useRef(new Animated.Value(1)).current;

  useEffect(() => {
    // Entrance animation
    Animated.sequence([
      // 1. Logo appears
      Animated.parallel([
        Animated.spring(logoScale, {
          toValue: 1,
          useNativeDriver: true,
          tension: 50,
          friction: 7,
        }),
        Animated.timing(logoOpacity, {
          toValue: 1,
          duration: 600,
          useNativeDriver: true,
        }),
      ]),
      // 2. Text appears
      Animated.timing(textOpacity, {
        toValue: 1,
        duration: 400,
        useNativeDriver: true,
      }),
      // 3. Progress bar
      Animated.timing(progressAnim, {
        toValue: 1,
        duration: 2000,
        useNativeDriver: false,
      }),
    ]).start(() => {
      // 4. Fade out
      Animated.timing(containerOpacity, {
        toValue: 0,
        duration: 500,
        useNativeDriver: true,
      }).start(() => {
        onFinish();
      });
    });
  }, []);

  const progressWidth = progressAnim.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '100%'],
  });

  return (
    <Animated.View style={[loadingStyles.container, { opacity: containerOpacity }]}>
      {/* Logo */}
      <Animated.View
        style={[
          loadingStyles.logoContainer,
          {
            opacity: logoOpacity,
            transform: [{ scale: logoScale }],
          },
        ]}
      >
        <Text style={loadingStyles.logoEmoji}>🚀</Text>
      </Animated.View>

      {/* App Name */}
      <Animated.Text style={[loadingStyles.appName, { opacity: textOpacity }]}>
        MyApp
      </Animated.Text>

      {/* Tagline */}
      <Animated.Text style={[loadingStyles.tagline, { opacity: textOpacity }]}>
        กำลังโหลด...
      </Animated.Text>

      {/* Progress Bar */}
      <View style={loadingStyles.progressContainer}>
        <Animated.View style={[loadingStyles.progressBar, { width: progressWidth }]} />
      </View>

      {/* Skeleton Content Preview */}
      <View style={loadingStyles.skeletonContainer}>
        <View style={loadingStyles.skeletonRow}>
          <SkeletonLoader width={50} height={50} borderRadius={25} />
          <View style={{ flex: 1, gap: 8 }}>
            <SkeletonLoader height={14} />
            <SkeletonLoader width="60%" height={12} />
          </View>
        </View>
        <SkeletonLoader height={120} borderRadius={8} />
        <SkeletonLoader height={14} />
        <SkeletonLoader width="80%" height={12} />
        <SkeletonLoader width="60%" height={12} />
      </View>
    </Animated.View>
  );
};

const loadingStyles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: 'white',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
  },
  logoContainer: {
    width: 100,
    height: 100,
    borderRadius: 50,
    backgroundColor: '#E3F2FD',
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 20,
  },
  logoEmoji: { fontSize: 50 },
  appName: { fontSize: 32, fontWeight: 'bold', color: '#333', marginBottom: 8 },
  tagline: { fontSize: 16, color: '#666', marginBottom: 32 },
  progressContainer: {
    width: '80%',
    height: 4,
    backgroundColor: '#E0E0E0',
    borderRadius: 2,
    overflow: 'hidden',
    marginBottom: 40,
  },
  progressBar: { height: '100%', backgroundColor: '#2196F3', borderRadius: 2 },
  skeletonContainer: { width: '100%', gap: 12 },
  skeletonRow: { flexDirection: 'row', gap: 12, alignItems: 'center' },
});

export { DotLoader, SkeletonLoader, PulseAnimation, SpinnerAnimation, AnimatedLoadingScreen };
```

---

## Tips และ Best Practices

### 1. หลีกเลี่ยง Performance Issues

```typescript
// ❌ Don't - Animated.Value ใน state
const [animValue] = useState(new Animated.Value(0));

// ✅ Do - ใช้ useRef
const animValue = useRef(new Animated.Value(0)).current;

// ❌ Don't - สร้าง style object ใหม่ทุกครั้ง render
<Animated.View style={{ opacity: fadeAnim, transform: [{ scale: scaleAnim }] }}>

// ✅ Do - ใช้ useMemo
const animStyle = useMemo(() => ({
  opacity: fadeAnim,
  transform: [{ scale: scaleAnim }],
}), []);
<Animated.View style={animStyle}>
```

### 2. Gesture + Animation

```typescript
// ผสม PanResponder กับ Animation
const pan = useRef(new Animated.ValueXY()).current;
const panResponder = PanResponder.create({
  onMoveShouldSetPanResponder: () => true,
  onPanResponderMove: Animated.event(
    [null, { dx: pan.x, dy: pan.y }],
    { useNativeDriver: false }
  ),
  onPanResponderRelease: () => {
    Animated.spring(pan, {
      toValue: { x: 0, y: 0 },
      useNativeDriver: false,
    }).start();
  },
});
```

### สรุป

- ใช้ useNativeDriver: true เสมอเมื่อ animate opacity และ transform
- useRef สำหรับ Animated.Value ไม่ใช่ useState
- Stagger ให้ผลที่สวยงามสำหรับ list animations
- Interpolation ช่วยให้ค่าเดียว animate ได้หลายอย่าง
- หยุด animation เมื่อ component unmount เพื่อป้องกัน memory leaks
