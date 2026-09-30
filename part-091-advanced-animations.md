# Part 091: Advanced Animations ใน React Native

## บทนำ

Animation ที่ดีทำให้ app รู้สึก polished และน่าใช้งาน เราจะเรียนรู้ Lottie animations, SVG animations, Particle systems, Physics-based animations และสร้าง Onboarding animation

## หัวข้อที่จะเรียน

1. Lottie Animations
2. SVG Animations
3. Particle Systems
4. Physics-based Animations
5. Workshop: Onboarding Animation

---

## 1. Lottie Animations

Lottie ช่วยให้เราใช้ animation จาก Adobe After Effects ได้โดยตรงใน React Native

### การติดตั้ง

```bash
npm install lottie-react-native
cd ios && pod install
```

### การใช้งานพื้นฐาน

```typescript
import LottieView from 'lottie-react-native';
import { useRef, useEffect } from 'react';

const LoadingAnimation: React.FC = () => {
  const animationRef = useRef<LottieView>(null);

  useEffect(() => {
    animationRef.current?.play();
  }, []);

  return (
    <LottieView
      ref={animationRef}
      source={require('./animations/loading.json')}
      autoPlay
      loop
      style={{ width: 200, height: 200 }}
    />
  );
};
```

### components/LottieButton.tsx

```typescript
import React, { useRef, useState } from 'react';
import { TouchableOpacity, Text, StyleSheet, View } from 'react-native';
import LottieView from 'lottie-react-native';

interface LottieButtonProps {
  onPress: () => void;
  label: string;
  animationSource: any;
  animationSize?: number;
}

const LottieButton: React.FC<LottieButtonProps> = ({
  onPress, label, animationSource, animationSize = 60
}) => {
  const animRef = useRef<LottieView>(null);
  const [isAnimating, setIsAnimating] = useState(false);

  const handlePress = () => {
    setIsAnimating(true);
    animRef.current?.play(0, 60); // เล่น 0-60 frame
    
    setTimeout(() => {
      setIsAnimating(false);
      onPress();
    }, 1000);
  };

  return (
    <TouchableOpacity
      style={styles.button}
      onPress={handlePress}
      activeOpacity={0.8}
    >
      <LottieView
        ref={animRef}
        source={animationSource}
        style={{ width: animationSize, height: animationSize }}
        loop={false}
        autoPlay={false}
      />
      <Text style={styles.label}>{label}</Text>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#fff',
    borderRadius: 16,
    padding: 16,
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.15,
    shadowRadius: 8,
    elevation: 4,
    minWidth: 120,
  },
  label: {
    marginTop: 8,
    fontSize: 14,
    fontWeight: '600',
    color: '#333',
  },
});

export default LottieButton;
```

### components/LottieSuccess.tsx (Reusable)

```typescript
import React, { forwardRef, useImperativeHandle, useRef } from 'react';
import { View, StyleSheet } from 'react-native';
import LottieView from 'lottie-react-native';

export interface LottieSuccessRef {
  play: () => void;
  reset: () => void;
}

const LottieSuccess = forwardRef<LottieSuccessRef>((_, ref) => {
  const animRef = useRef<LottieView>(null);

  useImperativeHandle(ref, () => ({
    play: () => animRef.current?.play(),
    reset: () => animRef.current?.reset(),
  }));

  return (
    <View style={styles.container}>
      <LottieView
        ref={animRef}
        source={require('./success.json')}
        style={styles.animation}
        loop={false}
        autoPlay={false}
      />
    </View>
  );
});

const styles = StyleSheet.create({
  container: { alignItems: 'center', justifyContent: 'center' },
  animation: { width: 150, height: 150 },
});

export default LottieSuccess;
```

---

## 2. SVG Animations

### การติดตั้ง

```bash
npm install react-native-svg
npm install react-native-reanimated
cd ios && pod install
```

### components/AnimatedSVGPath.tsx

```typescript
import React, { useEffect } from 'react';
import Animated, {
  useSharedValue,
  useAnimatedProps,
  withTiming,
  withRepeat,
  withSequence,
  Easing,
} from 'react-native-reanimated';
import { View, StyleSheet } from 'react-native';
import Svg, { Path, Circle, G } from 'react-native-svg';

const AnimatedPath = Animated.createAnimatedComponent(Path);
const AnimatedCircle = Animated.createAnimatedComponent(Circle);
const AnimatedG = Animated.createAnimatedComponent(G);

// Loading Spinner SVG Animation
const SVGLoadingSpinner: React.FC<{ size?: number; color?: string }> = ({
  size = 60,
  color = '#2196F3',
}) => {
  const rotation = useSharedValue(0);

  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, {
        duration: 1000,
        easing: Easing.linear,
      }),
      -1
    );
  }, []);

  const animatedProps = useAnimatedProps(() => ({
    rotation: rotation.value,
    originX: size / 2,
    originY: size / 2,
  }));

  return (
    <View style={{ width: size, height: size }}>
      <Svg width={size} height={size} viewBox={`0 0 ${size} ${size}`}>
        <AnimatedG animatedProps={animatedProps}>
          <Circle
            cx={size / 2}
            cy={size / 2}
            r={size / 2 - 4}
            stroke="#e0e0e0"
            strokeWidth={4}
            fill="none"
          />
          <Path
            d={`M ${size / 2} 4 A ${size / 2 - 4} ${size / 2 - 4} 0 0 1 ${size - 4} ${size / 2}`}
            stroke={color}
            strokeWidth={4}
            strokeLinecap="round"
            fill="none"
          />
        </AnimatedG>
      </Svg>
    </View>
  );
};

// Heart Beat Animation
const AnimatedHeart: React.FC = () => {
  const scale = useSharedValue(1);
  const opacity = useSharedValue(1);

  useEffect(() => {
    scale.value = withRepeat(
      withSequence(
        withTiming(1.3, { duration: 300, easing: Easing.out(Easing.quad) }),
        withTiming(1, { duration: 300, easing: Easing.in(Easing.quad) }),
        withTiming(1.2, { duration: 200 }),
        withTiming(1, { duration: 200 })
      ),
      -1,
      false
    );
  }, []);

  const heartStyle = useAnimatedProps(() => ({
    scale: scale.value,
  }));

  return (
    <View style={{ alignItems: 'center' }}>
      <Svg width={80} height={80} viewBox="0 0 100 100">
        <AnimatedPath
          d="M 50 85 C 20 60 5 40 5 25 C 5 12 15 5 25 5 C 35 5 45 12 50 20 C 55 12 65 5 75 5 C 85 5 95 12 95 25 C 95 40 80 60 50 85 Z"
          fill="#f44336"
          animatedProps={heartStyle}
          origin="50, 50"
        />
      </Svg>
    </View>
  );
};

// Progress Circle
const SVGProgressCircle: React.FC<{
  progress: number;
  size?: number;
  strokeWidth?: number;
}> = ({ progress, size = 120, strokeWidth = 10 }) => {
  const animatedProgress = useSharedValue(0);
  const radius = (size - strokeWidth) / 2;
  const circumference = 2 * Math.PI * radius;

  useEffect(() => {
    animatedProgress.value = withTiming(progress, { duration: 1000 });
  }, [progress]);

  const animatedProps = useAnimatedProps(() => ({
    strokeDashoffset: circumference * (1 - animatedProgress.value / 100),
  }));

  return (
    <View style={{ width: size, height: size }}>
      <Svg width={size} height={size} viewBox={`0 0 ${size} ${size}`}>
        {/* Background circle */}
        <Circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          stroke="#e0e0e0"
          strokeWidth={strokeWidth}
          fill="none"
        />
        {/* Progress circle */}
        <AnimatedCircle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          stroke="#2196F3"
          strokeWidth={strokeWidth}
          fill="none"
          strokeLinecap="round"
          strokeDasharray={circumference}
          animatedProps={animatedProps}
          rotation="-90"
          origin={`${size / 2}, ${size / 2}`}
        />
      </Svg>
    </View>
  );
};

export { SVGLoadingSpinner, AnimatedHeart, SVGProgressCircle };
```

---

## 3. Particle System

### components/ParticleSystem.tsx

```typescript
import React, { useEffect, useRef } from 'react';
import { View, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  withDelay,
  runOnJS,
  cancelAnimation,
} from 'react-native-reanimated';

const { width, height } = Dimensions.get('window');

interface Particle {
  id: number;
  x: number;
  y: number;
  size: number;
  color: string;
  velocityX: number;
  velocityY: number;
  opacity: number;
  rotation: number;
}

const PARTICLE_COLORS = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7', '#DDA0DD'];

interface ParticleSystemProps {
  count?: number;
  autoPlay?: boolean;
}

const ParticleView: React.FC<{
  particle: Particle;
  duration: number;
}> = ({ particle, duration }) => {
  const translateX = useSharedValue(particle.x);
  const translateY = useSharedValue(particle.y);
  const opacity = useSharedValue(1);
  const scale = useSharedValue(1);
  const rotation = useSharedValue(particle.rotation);

  useEffect(() => {
    translateX.value = withTiming(
      particle.x + particle.velocityX * 100,
      { duration }
    );
    translateY.value = withTiming(
      particle.y + particle.velocityY * 100,
      { duration }
    );
    opacity.value = withTiming(0, { duration });
    scale.value = withTiming(0, { duration });
    rotation.value = withTiming(particle.rotation + 360, { duration });
  }, []);

  const style = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
      { rotate: `${rotation.value}deg` },
    ],
    opacity: opacity.value,
  }));

  return (
    <Animated.View
      style={[
        styles.particle,
        {
          width: particle.size,
          height: particle.size,
          borderRadius: particle.size / 2,
          backgroundColor: particle.color,
        },
        style,
      ]}
    />
  );
};

const ConfettiSystem: React.FC<{
  x: number;
  y: number;
  isActive: boolean;
  onComplete?: () => void;
}> = ({ x, y, isActive, onComplete }) => {
  const [particles, setParticles] = React.useState<Particle[]>([]);

  useEffect(() => {
    if (!isActive) return;

    const newParticles: Particle[] = Array.from({ length: 30 }, (_, i) => ({
      id: i,
      x,
      y,
      size: Math.random() * 10 + 5,
      color: PARTICLE_COLORS[Math.floor(Math.random() * PARTICLE_COLORS.length)],
      velocityX: (Math.random() - 0.5) * 10,
      velocityY: -(Math.random() * 8 + 2),
      opacity: 1,
      rotation: Math.random() * 360,
    }));

    setParticles(newParticles);

    const timeout = setTimeout(() => {
      setParticles([]);
      onComplete?.();
    }, 2000);

    return () => clearTimeout(timeout);
  }, [isActive]);

  return (
    <View style={StyleSheet.absoluteFillObject} pointerEvents="none">
      {particles.map(particle => (
        <ParticleView
          key={particle.id}
          particle={particle}
          duration={2000}
        />
      ))}
    </View>
  );
};

// Floating Bubbles
const FloatingBubbles: React.FC = () => {
  const bubbles = Array.from({ length: 8 }, (_, i) => ({
    id: i,
    x: Math.random() * (width - 40),
    size: Math.random() * 30 + 20,
    color: PARTICLE_COLORS[i % PARTICLE_COLORS.length],
    duration: Math.random() * 3000 + 2000,
    delay: Math.random() * 2000,
  }));

  return (
    <View style={StyleSheet.absoluteFillObject} pointerEvents="none">
      {bubbles.map(bubble => (
        <FloatingBubble key={bubble.id} {...bubble} />
      ))}
    </View>
  );
};

const FloatingBubble: React.FC<{
  id: number;
  x: number;
  size: number;
  color: string;
  duration: number;
  delay: number;
}> = ({ x, size, color, duration, delay }) => {
  const translateY = useSharedValue(height + size);
  const opacity = useSharedValue(0.7);

  useEffect(() => {
    translateY.value = withDelay(
      delay,
      withRepeat(
        withTiming(-size, { duration }),
        -1
      )
    );
  }, []);

  const style = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
    opacity: opacity.value,
  }));

  return (
    <Animated.View
      style={[
        {
          position: 'absolute',
          left: x,
          width: size,
          height: size,
          borderRadius: size / 2,
          backgroundColor: color,
        },
        style,
      ]}
    />
  );
};

const styles = StyleSheet.create({
  particle: {
    position: 'absolute',
  },
});

export { ConfettiSystem, FloatingBubbles };
```

---

## 4. Physics-based Animations

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withDecay,
  useAnimatedGestureHandler,
  runOnJS,
} from 'react-native-reanimated';
import {
  PanGestureHandler,
  GestureHandlerRootView,
} from 'react-native-gesture-handler';

// Draggable Card with Physics
const PhysicsCard: React.FC = () => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);

  const gestureHandler = useAnimatedGestureHandler({
    onStart: (_, context: any) => {
      context.startX = translateX.value;
      context.startY = translateY.value;
      scale.value = withSpring(1.05);
    },
    onActive: (event, context) => {
      translateX.value = context.startX + event.translationX;
      translateY.value = context.startY + event.translationY;
    },
    onEnd: (event) => {
      scale.value = withSpring(1);
      
      // Physics decay - momentum
      translateX.value = withDecay({
        velocity: event.velocityX,
        clamp: [-200, 200],
      });
      translateY.value = withDecay({
        velocity: event.velocityY,
        clamp: [-400, 400],
      });
    },
  });

  const cardStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <GestureHandlerRootView>
      <PanGestureHandler onGestureEvent={gestureHandler}>
        <Animated.View style={[physicsStyles.card, cardStyle]}>
          <Animated.Text style={physicsStyles.cardText}>
            ลากฉันไป!
          </Animated.Text>
        </Animated.View>
      </PanGestureHandler>
    </GestureHandlerRootView>
  );
};

// Spring Animation
const SpringBall: React.FC = () => {
  const scale = useSharedValue(1);
  const translateY = useSharedValue(0);

  const bounce = () => {
    scale.value = withSpring(0.8, {}, () => {
      scale.value = withSpring(1, {
        damping: 3,
        stiffness: 200,
        mass: 1,
      });
    });

    translateY.value = withSpring(-100, {
      damping: 5,
      stiffness: 150,
    }, () => {
      translateY.value = withSpring(0, {
        damping: 5,
        stiffness: 150,
      });
    });
  };

  const style = useAnimatedStyle(() => ({
    transform: [
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <Animated.View style={[physicsStyles.ball, style]}>
      <Animated.Text
        onPress={bounce}
        style={physicsStyles.ballText}
      >
        🏀
      </Animated.Text>
    </Animated.View>
  );
};

const physicsStyles = StyleSheet.create({
  card: {
    width: 200, height: 120, backgroundColor: '#2196F3',
    borderRadius: 16, justifyContent: 'center', alignItems: 'center',
    shadowColor: '#000', shadowOffset: { width: 0, height: 8 },
    shadowOpacity: 0.3, shadowRadius: 12, elevation: 8,
  },
  cardText: { color: 'white', fontSize: 16, fontWeight: '600' },
  ball: { alignItems: 'center' },
  ballText: { fontSize: 60 },
});

import { StyleSheet } from 'react-native';
```

---

## 5. Workshop: Onboarding Animation

### screens/OnboardingScreen.tsx

```typescript
import React, { useState, useRef } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Dimensions,
  TouchableOpacity,
  FlatList,
} from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useAnimatedScrollHandler,
  interpolate,
  Extrapolation,
  withSpring,
  withTiming,
} from 'react-native-reanimated';
import LottieView from 'lottie-react-native';

const { width, height } = Dimensions.get('window');

const ONBOARDING_DATA = [
  {
    id: '1',
    title: 'ยินดีต้อนรับ!',
    description: 'แอปที่ดีที่สุดสำหรับการจัดการชีวิตประจำวัน',
    animation: require('./animations/welcome.json'),
    backgroundColor: '#667eea',
    textColor: 'white',
  },
  {
    id: '2',
    title: 'ติดตามทุกอย่าง',
    description: 'บันทึกและติดตามกิจกรรมต่างๆ ได้ง่าย',
    animation: require('./animations/track.json'),
    backgroundColor: '#f093fb',
    textColor: 'white',
  },
  {
    id: '3',
    title: 'ร่วมกันกับเพื่อน',
    description: 'แชร์และทำงานร่วมกับทีมของคุณ',
    animation: require('./animations/social.json'),
    backgroundColor: '#4facfe',
    textColor: 'white',
  },
  {
    id: '4',
    title: 'เริ่มต้นได้เลย!',
    description: 'สร้างบัญชีใหม่หรือเข้าสู่ระบบ',
    animation: require('./animations/start.json'),
    backgroundColor: '#43e97b',
    textColor: 'white',
  },
];

const OnboardingItem: React.FC<{
  item: typeof ONBOARDING_DATA[0];
  index: number;
  scrollX: Animated.SharedValue<number>;
}> = ({ item, index, scrollX }) => {
  const animRef = useRef<LottieView>(null);

  const containerStyle = useAnimatedStyle(() => {
    const inputRange = [(index - 1) * width, index * width, (index + 1) * width];
    const scale = interpolate(
      scrollX.value,
      inputRange,
      [0.8, 1, 0.8],
      Extrapolation.CLAMP
    );
    const opacity = interpolate(
      scrollX.value,
      inputRange,
      [0.5, 1, 0.5],
      Extrapolation.CLAMP
    );

    return { transform: [{ scale }], opacity };
  });

  const textStyle = useAnimatedStyle(() => {
    const inputRange = [(index - 1) * width, index * width, (index + 1) * width];
    const translateY = interpolate(
      scrollX.value,
      inputRange,
      [50, 0, -50],
      Extrapolation.CLAMP
    );
    const opacity = interpolate(
      scrollX.value,
      inputRange,
      [0, 1, 0],
      Extrapolation.CLAMP
    );

    return { transform: [{ translateY }], opacity };
  });

  return (
    <View style={[styles.slide, { backgroundColor: item.backgroundColor }]}>
      <Animated.View style={[styles.animationContainer, containerStyle]}>
        <LottieView
          ref={animRef}
          source={item.animation}
          style={styles.animation}
          autoPlay
          loop
        />
      </Animated.View>

      <Animated.View style={[styles.textContainer, textStyle]}>
        <Text style={[styles.title, { color: item.textColor }]}>
          {item.title}
        </Text>
        <Text style={[styles.description, { color: `${item.textColor}CC` }]}>
          {item.description}
        </Text>
      </Animated.View>
    </View>
  );
};

const OnboardingDots: React.FC<{
  count: number;
  currentIndex: number;
  scrollX: Animated.SharedValue<number>;
}> = ({ count, scrollX }) => {
  return (
    <View style={styles.dotsContainer}>
      {Array.from({ length: count }, (_, i) => {
        const dotStyle = useAnimatedStyle(() => {
          const inputRange = [(i - 1) * width, i * width, (i + 1) * width];
          const dotWidth = interpolate(
            scrollX.value,
            inputRange,
            [8, 24, 8],
            Extrapolation.CLAMP
          );
          const opacity = interpolate(
            scrollX.value,
            inputRange,
            [0.5, 1, 0.5],
            Extrapolation.CLAMP
          );

          return { width: dotWidth, opacity };
        });

        return (
          <Animated.View
            key={i}
            style={[styles.dot, dotStyle]}
          />
        );
      })}
    </View>
  );
};

const OnboardingScreen: React.FC<{ onComplete: () => void }> = ({ onComplete }) => {
  const [currentIndex, setCurrentIndex] = useState(0);
  const scrollX = useSharedValue(0);
  const flatListRef = useRef<FlatList>(null);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollX.value = event.contentOffset.x;
    },
  });

  const handleNext = () => {
    if (currentIndex < ONBOARDING_DATA.length - 1) {
      flatListRef.current?.scrollToIndex({
        index: currentIndex + 1,
        animated: true,
      });
      setCurrentIndex(prev => prev + 1);
    } else {
      onComplete();
    }
  };

  const handleSkip = () => {
    onComplete();
  };

  const isLastSlide = currentIndex === ONBOARDING_DATA.length - 1;

  return (
    <View style={styles.container}>
      {!isLastSlide && (
        <TouchableOpacity style={styles.skipButton} onPress={handleSkip}>
          <Text style={styles.skipText}>ข้าม</Text>
        </TouchableOpacity>
      )}

      <Animated.FlatList
        ref={flatListRef}
        data={ONBOARDING_DATA}
        keyExtractor={item => item.id}
        horizontal
        pagingEnabled
        showsHorizontalScrollIndicator={false}
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        onMomentumScrollEnd={e => {
          const index = Math.round(e.nativeEvent.contentOffset.x / width);
          setCurrentIndex(index);
        }}
        renderItem={({ item, index }) => (
          <OnboardingItem
            item={item}
            index={index}
            scrollX={scrollX}
          />
        )}
      />

      <View style={styles.footer}>
        <OnboardingDots
          count={ONBOARDING_DATA.length}
          currentIndex={currentIndex}
          scrollX={scrollX}
        />

        <TouchableOpacity style={styles.nextButton} onPress={handleNext}>
          <Text style={styles.nextButtonText}>
            {isLastSlide ? 'เริ่มใช้งาน' : 'ถัดไป'}
          </Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  skipButton: {
    position: 'absolute', top: 50, right: 20, zIndex: 10,
    padding: 8,
  },
  skipText: { color: 'rgba(255,255,255,0.8)', fontSize: 16 },
  slide: { width, height, justifyContent: 'center', alignItems: 'center' },
  animationContainer: { width: width * 0.7, height: width * 0.7 },
  animation: { width: '100%', height: '100%' },
  textContainer: { padding: 24, alignItems: 'center' },
  title: { fontSize: 32, fontWeight: 'bold', textAlign: 'center', marginBottom: 12 },
  description: { fontSize: 16, textAlign: 'center', lineHeight: 24 },
  footer: {
    position: 'absolute', bottom: 50, width: '100%',
    alignItems: 'center', paddingHorizontal: 24,
  },
  dotsContainer: { flexDirection: 'row', marginBottom: 24, gap: 6, alignItems: 'center' },
  dot: { height: 8, borderRadius: 4, backgroundColor: 'white' },
  nextButton: {
    backgroundColor: 'white', paddingHorizontal: 40, paddingVertical: 16,
    borderRadius: 50, width: '100%', alignItems: 'center',
    shadowColor: '#000', shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.2, shadowRadius: 8, elevation: 4,
  },
  nextButtonText: { fontSize: 18, fontWeight: 'bold', color: '#333' },
});

export default OnboardingScreen;
```

---

## Workshop Exercises

1. **Skeleton Loading** - สร้าง skeleton animation สำหรับ content loading
2. **Morphing Shapes** - SVG shapes ที่ transform เป็นรูปอื่น
3. **Parallax Scroll** - background เคลื่อนที่ต่างความเร็วจาก foreground
4. **Pull to Refresh** - custom pull-to-refresh ด้วย Lottie animation

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Lottie** - ใช้ animation จาก After Effects
2. **SVG Animation** - animate shapes และ paths
3. **Particle System** - confetti, bubbles effects
4. **Physics Animation** - spring, decay animations
5. **Onboarding** - สร้าง onboarding flow ที่สวยงาม
