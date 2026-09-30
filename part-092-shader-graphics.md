# Part 092: Shader and Graphics ใน React Native

## บทนำ

Graphics programming ใน React Native ช่วยให้สร้าง visual effects ที่ซับซ้อนได้ เราจะเรียนรู้การใช้ React Native Skia, OpenGL ES, Custom Shaders, และ 2D Graphics

## หัวข้อที่จะเรียน

1. React Native Skia
2. OpenGL ES กับ react-native-gl
3. Custom Shaders (GLSL)
4. 2D Graphics
5. Workshop: Custom Visual Effects

---

## 1. React Native Skia

React Native Skia เป็น high-performance 2D graphics library ที่ใช้ Skia engine เดียวกับ Flutter

### การติดตั้ง

```bash
npm install @shopify/react-native-skia
cd ios && pod install
```

### พื้นฐาน Skia

```typescript
import React from 'react';
import {
  Canvas,
  Circle,
  Fill,
  Path,
  Rect,
  Text,
  useFont,
  Group,
  Paint,
  Blur,
  LinearGradient,
  vec,
  useValue,
  useTiming,
  interpolate,
  Easing,
} from '@shopify/react-native-skia';

// วาด Circle ง่ายๆ
const BasicCircle: React.FC = () => {
  return (
    <Canvas style={{ width: 200, height: 200 }}>
      <Fill color="white" />
      <Circle cx={100} cy={100} r={80} color="#2196F3" />
      <Circle cx={100} cy={100} r={60} color="#64B5F6" />
      <Circle cx={100} cy={100} r={40} color="#BBDEFB" />
    </Canvas>
  );
};

// Gradient Background
const GradientBackground: React.FC<{ width: number; height: number }> = ({
  width,
  height,
}) => {
  return (
    <Canvas style={{ width, height }}>
      <Rect x={0} y={0} width={width} height={height}>
        <LinearGradient
          start={vec(0, 0)}
          end={vec(width, height)}
          colors={['#667eea', '#764ba2']}
        />
      </Rect>
    </Canvas>
  );
};

// Animated Circle
const AnimatedSkiaCircle: React.FC = () => {
  const progress = useTiming({ from: 0, to: 1, loop: true, easing: Easing.inOut(Easing.ease) });

  const r = useValue(0);
  
  r.value = interpolate(progress.value, [0, 1], [20, 80]);

  return (
    <Canvas style={{ width: 200, height: 200 }}>
      <Fill color="#1a1a2e" />
      <Circle cx={100} cy={100} r={r} color="#e94560" />
    </Canvas>
  );
};
```

### components/SkiaCard.tsx

```typescript
import React from 'react';
import {
  Canvas,
  RoundedRect,
  LinearGradient,
  vec,
  Shadow,
  Text,
  useFont,
  Group,
  Paint,
  Path,
  Skia,
} from '@shopify/react-native-skia';
import { StyleSheet } from 'react-native';

interface SkiaCardProps {
  width: number;
  height: number;
  title: string;
  value: string;
  gradientStart: string;
  gradientEnd: string;
}

const SkiaCard: React.FC<SkiaCardProps> = ({
  width, height, title, value, gradientStart, gradientEnd
}) => {
  const font = useFont(require('./fonts/Inter-Bold.ttf'), 24);
  const smallFont = useFont(require('./fonts/Inter-Regular.ttf'), 14);

  if (!font || !smallFont) return null;

  return (
    <Canvas style={{ width, height }}>
      {/* Card Background */}
      <RoundedRect x={0} y={0} width={width} height={height} r={20}>
        <LinearGradient
          start={vec(0, 0)}
          end={vec(width, height)}
          colors={[gradientStart, gradientEnd]}
        />
        <Shadow dx={0} dy={8} blur={20} color="rgba(0,0,0,0.3)" />
      </RoundedRect>

      {/* Title */}
      <Text
        x={24}
        y={50}
        text={title}
        font={smallFont}
        color="rgba(255,255,255,0.8)"
      />

      {/* Value */}
      <Text
        x={24}
        y={90}
        text={value}
        font={font}
        color="white"
      />

      {/* Decorative circle */}
      <Group opacity={0.15}>
        <Paint color="white">
          <Paint.BlendMode mode="screen" />
        </Paint>
        <Circle cx={width - 30} cy={30} r={60} color="white" />
        <Circle cx={width - 10} cy={height - 20} r={80} color="white" />
      </Group>
    </Canvas>
  );
};

export default SkiaCard;
```

### components/SkiaChart.tsx

```typescript
import React from 'react';
import {
  Canvas,
  Path,
  Skia,
  Group,
  LinearGradient,
  vec,
  useValue,
  useTiming,
  interpolate,
  Paint,
  Circle,
  Text,
  useFont,
} from '@shopify/react-native-skia';

interface DataPoint {
  x: number;
  y: number;
  label: string;
}

interface SkiaLineChartProps {
  data: DataPoint[];
  width: number;
  height: number;
  padding?: number;
}

const SkiaLineChart: React.FC<SkiaLineChartProps> = ({
  data, width, height, padding = 30
}) => {
  const font = useFont(require('./fonts/Inter-Regular.ttf'), 12);
  const progress = useTiming({ from: 0, to: 1, duration: 1500, easing: Easing.inOut(Easing.ease) });

  if (data.length === 0) return null;

  // คำนวณ scale
  const chartWidth = width - padding * 2;
  const chartHeight = height - padding * 2;
  const maxY = Math.max(...data.map(d => d.y));
  const minY = Math.min(...data.map(d => d.y));

  const scaleX = (index: number) =>
    padding + (index / (data.length - 1)) * chartWidth;
  const scaleY = (value: number) =>
    padding + chartHeight - ((value - minY) / (maxY - minY)) * chartHeight;

  // สร้าง path
  const linePath = Skia.Path.Make();
  const areaPath = Skia.Path.Make();

  data.forEach((point, index) => {
    const x = scaleX(index);
    const y = scaleY(point.y);

    if (index === 0) {
      linePath.moveTo(x, y);
      areaPath.moveTo(x, height - padding);
      areaPath.lineTo(x, y);
    } else {
      // Smooth curve
      const prevX = scaleX(index - 1);
      const prevY = scaleY(data[index - 1].y);
      const cpX = (prevX + x) / 2;
      
      linePath.cubicTo(cpX, prevY, cpX, y, x, y);
      areaPath.cubicTo(cpX, prevY, cpX, y, x, y);
    }
  });

  // ปิด area path
  areaPath.lineTo(scaleX(data.length - 1), height - padding);
  areaPath.close();

  return (
    <Canvas style={{ width, height }}>
      {/* Area fill */}
      <Path path={areaPath} style="fill" opacity={0.3}>
        <LinearGradient
          start={vec(0, padding)}
          end={vec(0, height - padding)}
          colors={['#2196F3', 'transparent']}
        />
      </Path>

      {/* Line */}
      <Path
        path={linePath}
        style="stroke"
        strokeWidth={3}
        strokeCap="round"
        strokeJoin="round"
        color="#2196F3"
      />

      {/* Data points */}
      {data.map((point, index) => (
        <Group key={index}>
          <Circle
            cx={scaleX(index)}
            cy={scaleY(point.y)}
            r={5}
            color="white"
          />
          <Circle
            cx={scaleX(index)}
            cy={scaleY(point.y)}
            r={4}
            color="#2196F3"
          />
        </Group>
      ))}

      {/* Labels */}
      {font && data.map((point, index) => (
        <Text
          key={index}
          x={scaleX(index) - 10}
          y={height - 5}
          text={point.label}
          font={font}
          color="#666"
        />
      ))}
    </Canvas>
  );
};

export default SkiaLineChart;
```

---

## 2. OpenGL ES กับ react-native-gl

```bash
npm install @react-three/fiber @react-three/drei
# หรือสำหรับ native approach
npm install react-native-gl
```

### components/GLCanvas.tsx

```typescript
import React, { useRef, useEffect } from 'react';
import { GLView } from 'expo-gl';

// Basic WebGL Triangle
const GLTriangle: React.FC = () => {
  const onContextCreate = async (gl: WebGLRenderingContext) => {
    // Vertex Shader
    const vertexShaderSource = `
      attribute vec2 a_position;
      attribute vec3 a_color;
      varying vec3 v_color;
      
      void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
        v_color = a_color;
      }
    `;

    // Fragment Shader
    const fragmentShaderSource = `
      precision mediump float;
      varying vec3 v_color;
      
      void main() {
        gl_FragColor = vec4(v_color, 1.0);
      }
    `;

    // Compile shaders
    const vertexShader = gl.createShader(gl.VERTEX_SHADER)!;
    gl.shaderSource(vertexShader, vertexShaderSource);
    gl.compileShader(vertexShader);

    const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER)!;
    gl.shaderSource(fragmentShader, fragmentShaderSource);
    gl.compileShader(fragmentShader);

    // Link program
    const program = gl.createProgram()!;
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    gl.useProgram(program);

    // Vertices (x, y, r, g, b)
    const vertices = new Float32Array([
       0.0,  0.5,  1.0, 0.0, 0.0, // top - red
      -0.5, -0.5,  0.0, 1.0, 0.0, // bottom left - green
       0.5, -0.5,  0.0, 0.0, 1.0, // bottom right - blue
    ]);

    // Create buffer
    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

    // Set up attributes
    const STRIDE = 5 * Float32Array.BYTES_PER_ELEMENT;
    
    const positionLoc = gl.getAttribLocation(program, 'a_position');
    gl.enableVertexAttribArray(positionLoc);
    gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, STRIDE, 0);

    const colorLoc = gl.getAttribLocation(program, 'a_color');
    gl.enableVertexAttribArray(colorLoc);
    gl.vertexAttribPointer(colorLoc, 3, gl.FLOAT, false, STRIDE, 
      2 * Float32Array.BYTES_PER_ELEMENT);

    // Draw
    gl.clearColor(0, 0, 0, 1);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLES, 0, 3);
    
    gl.endFrameEXP?.();
  };

  return (
    <GLView
      style={{ width: 300, height: 300 }}
      onContextCreate={onContextCreate}
    />
  );
};
```

---

## 3. Custom GLSL Shaders

### Ripple Effect Shader

```typescript
import { Canvas, Shader, Skia, useImage } from '@shopify/react-native-skia';
import { useValue, useTiming } from '@shopify/react-native-skia';

const RIPPLE_SHADER = `
  uniform shader image;
  uniform float time;
  uniform vec2 resolution;
  uniform vec2 center;
  
  vec4 main(vec2 fragCoord) {
    vec2 uv = fragCoord / resolution;
    vec2 dir = uv - center;
    float dist = length(dir);
    
    float wave = sin(dist * 30.0 - time * 5.0) * 0.02;
    float strength = 1.0 / (1.0 + dist * 15.0);
    
    vec2 offset = normalize(dir) * wave * strength;
    
    return image.eval((uv + offset) * resolution);
  }
`;

const RippleEffect: React.FC<{ imageSource: any; width: number; height: number }> = ({
  imageSource, width, height
}) => {
  const image = useImage(imageSource);
  const time = useTiming({ from: 0, to: 100, loop: true, duration: 3000 });

  if (!image) return null;

  const shader = Skia.RuntimeEffect.Make(RIPPLE_SHADER)!;

  return (
    <Canvas style={{ width, height }}>
      <Shader
        source={shader}
        uniforms={{
          time: time,
          resolution: [width, height],
          center: [0.5, 0.5],
        }}
      >
        {/* shader children */}
      </Shader>
    </Canvas>
  );
};

// Glow Effect
const GLOW_SHADER = `
  uniform shader image;
  uniform float intensity;
  uniform vec3 glowColor;
  
  vec4 main(vec2 fragCoord) {
    vec4 color = image.eval(fragCoord);
    float brightness = dot(color.rgb, vec3(0.299, 0.587, 0.114));
    
    if (brightness > 0.7) {
      vec3 glow = glowColor * brightness * intensity;
      color.rgb += glow;
    }
    
    return color;
  }
`;

// Chromatic Aberration
const CHROMATIC_SHADER = `
  uniform shader image;
  uniform vec2 resolution;
  uniform float offset;
  
  vec4 main(vec2 fragCoord) {
    vec2 uv = fragCoord / resolution;
    vec2 dir = uv - vec2(0.5);
    float dist = length(dir);
    
    vec2 rOffset = dir * offset * 0.01;
    vec2 gOffset = dir * offset * 0.005;
    vec2 bOffset = dir * offset * 0.0;
    
    float r = image.eval((uv + rOffset) * resolution).r;
    float g = image.eval((uv + gOffset) * resolution).g;
    float b = image.eval((uv + bOffset) * resolution).b;
    
    return vec4(r, g, b, 1.0);
  }
`;
```

---

## 4. 2D Graphics Drawing App

### screens/DrawingCanvas.tsx

```typescript
import React, { useState, useCallback } from 'react';
import { View, StyleSheet, TouchableOpacity, Text } from 'react-native';
import {
  Canvas,
  Path,
  Skia,
  useCanvasRef,
  useTouchHandler,
  PaintStyle,
  StrokeCap,
  StrokeJoin,
} from '@shopify/react-native-skia';
import { GestureHandlerRootView } from 'react-native-gesture-handler';

interface Stroke {
  path: ReturnType<typeof Skia.Path.Make>;
  color: string;
  strokeWidth: number;
}

const COLORS = ['#000000', '#f44336', '#2196F3', '#4CAF50', '#FF9800', '#9C27B0', '#FFFFFF'];

const DrawingCanvas: React.FC = () => {
  const [strokes, setStrokes] = useState<Stroke[]>([]);
  const [currentStroke, setCurrentStroke] = useState<Stroke | null>(null);
  const [selectedColor, setSelectedColor] = useState('#000000');
  const [strokeWidth, setStrokeWidth] = useState(4);
  const [isErasing, setIsErasing] = useState(false);
  const canvasRef = useCanvasRef();

  const touchHandler = useTouchHandler({
    onStart: ({ x, y }) => {
      const newPath = Skia.Path.Make();
      newPath.moveTo(x, y);
      const stroke: Stroke = {
        path: newPath,
        color: isErasing ? '#FFFFFF' : selectedColor,
        strokeWidth: isErasing ? 20 : strokeWidth,
      };
      setCurrentStroke(stroke);
    },
    onActive: ({ x, y }) => {
      if (currentStroke) {
        currentStroke.path.lineTo(x, y);
        setCurrentStroke({ ...currentStroke });
      }
    },
    onEnd: () => {
      if (currentStroke) {
        setStrokes(prev => [...prev, currentStroke]);
        setCurrentStroke(null);
      }
    },
  });

  const handleUndo = () => {
    setStrokes(prev => prev.slice(0, -1));
  };

  const handleClear = () => {
    setStrokes([]);
    setCurrentStroke(null);
  };

  const handleSave = async () => {
    const image = canvasRef.current?.makeImageSnapshot();
    if (image) {
      const data = image.encodeToBase64();
      // บันทึกหรือแชร์รูปภาพ
      console.log('Saved drawing:', data.substring(0, 50) + '...');
    }
  };

  return (
    <View style={styles.container}>
      {/* Toolbar */}
      <View style={styles.toolbar}>
        <TouchableOpacity onPress={handleUndo} style={styles.toolButton}>
          <Text>↩️</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={handleClear} style={styles.toolButton}>
          <Text>🗑️</Text>
        </TouchableOpacity>
        <TouchableOpacity
          onPress={() => setIsErasing(!isErasing)}
          style={[styles.toolButton, isErasing && styles.activeToolButton]}
        >
          <Text>⬜</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={handleSave} style={styles.toolButton}>
          <Text>💾</Text>
        </TouchableOpacity>
      </View>

      {/* Stroke Width */}
      <View style={styles.strokeWidths}>
        {[2, 4, 8, 16].map(w => (
          <TouchableOpacity
            key={w}
            style={[styles.strokeWidthButton, strokeWidth === w && styles.activeStroke]}
            onPress={() => setStrokeWidth(w)}
          >
            <View style={[styles.strokePreview, {
              height: w, width: w * 4, borderRadius: w / 2
            }]} />
          </TouchableOpacity>
        ))}
      </View>

      {/* Color Palette */}
      <View style={styles.colorPalette}>
        {COLORS.map(color => (
          <TouchableOpacity
            key={color}
            style={[
              styles.colorDot,
              { backgroundColor: color },
              selectedColor === color && styles.selectedColor,
              color === '#FFFFFF' && styles.whiteBorder,
            ]}
            onPress={() => {
              setSelectedColor(color);
              setIsErasing(false);
            }}
          />
        ))}
      </View>

      {/* Canvas */}
      <GestureHandlerRootView style={styles.canvasContainer}>
        <Canvas
          ref={canvasRef}
          style={styles.canvas}
          onTouch={touchHandler}
        >
          {/* Background */}
          <Path
            path={Skia.Path.Make().addRect(Skia.XYWHRect(0, 0, 400, 600))}
            color="white"
          />

          {/* Completed strokes */}
          {strokes.map((stroke, index) => (
            <Path
              key={index}
              path={stroke.path}
              color={stroke.color}
              style="stroke"
              strokeWidth={stroke.strokeWidth}
              strokeCap={StrokeCap.Round}
              strokeJoin={StrokeJoin.Round}
            />
          ))}

          {/* Current stroke */}
          {currentStroke && (
            <Path
              path={currentStroke.path}
              color={currentStroke.color}
              style="stroke"
              strokeWidth={currentStroke.strokeWidth}
              strokeCap={StrokeCap.Round}
              strokeJoin={StrokeJoin.Round}
            />
          )}
        </Canvas>
      </GestureHandlerRootView>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  toolbar: {
    flexDirection: 'row', padding: 8, backgroundColor: 'white',
    justifyContent: 'center', gap: 8, borderBottomWidth: 1, borderBottomColor: '#e0e0e0',
  },
  toolButton: {
    width: 44, height: 44, borderRadius: 22, backgroundColor: '#f5f5f5',
    justifyContent: 'center', alignItems: 'center',
  },
  activeToolButton: { backgroundColor: '#bbdefb' },
  strokeWidths: {
    flexDirection: 'row', padding: 8, backgroundColor: 'white',
    justifyContent: 'center', gap: 8, alignItems: 'center',
  },
  strokeWidthButton: {
    padding: 8, borderRadius: 8, borderWidth: 1, borderColor: 'transparent',
  },
  activeStroke: { borderColor: '#2196F3' },
  strokePreview: { backgroundColor: '#333' },
  colorPalette: {
    flexDirection: 'row', padding: 12, backgroundColor: 'white',
    justifyContent: 'center', gap: 10,
  },
  colorDot: { width: 32, height: 32, borderRadius: 16 },
  selectedColor: { borderWidth: 3, borderColor: '#333' },
  whiteBorder: { borderWidth: 1, borderColor: '#ddd' },
  canvasContainer: { flex: 1, margin: 16 },
  canvas: {
    flex: 1, backgroundColor: 'white', borderRadius: 12,
    shadowColor: '#000', shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.15, shadowRadius: 8, elevation: 4,
  },
});

export default DrawingCanvas;
```

---

## Workshop: Custom Visual Effects

### Glassmorphism Effect

```typescript
import { Canvas, RoundedRect, BlurMask, Group } from '@shopify/react-native-skia';

const GlassmorphismCard: React.FC<{
  width: number;
  height: number;
  children: React.ReactNode;
}> = ({ width, height, children }) => {
  return (
    <View>
      <Canvas style={{ position: 'absolute', width, height }}>
        <Group>
          <RoundedRect
            x={0} y={0} width={width} height={height} r={20}
            color="rgba(255,255,255,0.15)"
          >
            <BlurMask blur={10} style="normal" />
          </RoundedRect>
          <RoundedRect
            x={0} y={0} width={width} height={height} r={20}
            color="transparent"
            style="stroke"
            strokeWidth={1}
          />
        </Group>
      </Canvas>
      <View style={{ width, height, padding: 16 }}>
        {children}
      </View>
    </View>
  );
};
```

---

## Workshop Exercises

1. **Photo Filter App** - เพิ่ม filters (grayscale, sepia, blur) ให้รูปภาพ
2. **Signature Capture** - จับลายเซ็นผู้ใช้ด้วย smooth drawing
3. **Game Board** - สร้าง board game ด้วย Skia
4. **Data Visualization** - สร้าง custom chart ที่ไม่มีใน library ทั่วไป

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **React Native Skia** - 2D graphics ประสิทธิภาพสูง
2. **OpenGL ES** - low-level graphics programming
3. **GLSL Shaders** - custom visual effects
4. **Drawing App** - สร้าง canvas drawing application
