# Part 085: Machine Learning - TensorFlow.js

## Machine Learning ใน React Native

การนำ Machine Learning มาใช้ใน React Native ช่วยให้แอปสามารถประมวลผลข้อมูล, จำแนกรูปภาพ, ตรวจจับวัตถุ, และทำการวิเคราะห์ข้อมูลได้โดยตรงบนอุปกรณ์ (on-device inference)

---

## 1. TensorFlow.js สำหรับ React Native

### 1.1 ติดตั้ง

```bash
npm install @tensorflow/tfjs
npm install @tensorflow/tfjs-react-native
npm install @tensorflow-models/mobilenet
npm install @tensorflow-models/coco-ssd
npm install expo-gl
npm install expo-camera

# iOS
cd ios && pod install
```

### 1.2 Setup TensorFlow.js

```typescript
// tensorflow/setup.ts
import * as tf from '@tensorflow/tfjs';
import '@tensorflow/tfjs-react-native';

let isInitialized = false;

export async function initializeTensorFlow(): Promise<void> {
  if (isInitialized) return;
  
  // Wait for TF to be ready
  await tf.ready();
  
  // Log backend info
  console.log('TensorFlow.js is ready!');
  console.log('Backend:', tf.getBackend());
  
  // Warm up model (optional)
  const warmupTensor = tf.zeros([1, 224, 224, 3]);
  warmupTensor.dispose();
  
  isInitialized = true;
}

export function getTFMemoryInfo() {
  const info = tf.memory();
  return {
    numTensors: info.numTensors,
    numDataBuffers: info.numDataBuffers,
    numBytes: info.numBytes,
    unreliable: info.unreliable
  };
}

export function disposeTensors(...tensors: tf.Tensor[]) {
  tensors.forEach(t => {
    if (t && !t.isDisposed) t.dispose();
  });
}
```

---

## 2. Pre-trained Models

### 2.1 MobileNet Image Classification

```typescript
// models/ImageClassifier.ts
import * as tf from '@tensorflow/tfjs';
import * as mobilenet from '@tensorflow-models/mobilenet';
import { decodeJpeg } from '@tensorflow/tfjs-react-native';

interface ClassificationResult {
  className: string;
  probability: number;
}

export class ImageClassifier {
  private model: mobilenet.MobileNet | null = null;
  private isLoading = false;
  
  async load(): Promise<void> {
    if (this.model || this.isLoading) return;
    
    this.isLoading = true;
    console.log('Loading MobileNet...');
    
    try {
      this.model = await mobilenet.load({
        version: 2,
        alpha: 1.0
      });
      console.log('MobileNet loaded!');
    } finally {
      this.isLoading = false;
    }
  }
  
  async classifyImage(
    imageData: Uint8Array,
    topK = 5
  ): Promise<ClassificationResult[]> {
    if (!this.model) {
      throw new Error('Model not loaded');
    }
    
    // Convert image data to tensor
    const imageTensor = decodeJpeg(imageData);
    
    try {
      // Classify
      const predictions = await this.model.classify(imageTensor, topK);
      
      return predictions.map(p => ({
        className: p.className,
        probability: p.probability
      }));
    } finally {
      imageTensor.dispose();
    }
  }
  
  async classifyFromBase64(
    base64: string,
    topK = 5
  ): Promise<ClassificationResult[]> {
    // Convert base64 to Uint8Array
    const binaryString = atob(base64);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
      bytes[i] = binaryString.charCodeAt(i);
    }
    
    return this.classifyImage(bytes, topK);
  }
  
  isLoaded(): boolean {
    return this.model !== null;
  }
  
  dispose() {
    // MobileNet doesn't expose direct dispose, but TF will handle cleanup
    this.model = null;
  }
}

// Singleton
export const imageClassifier = new ImageClassifier();
```

### 2.2 COCO-SSD Object Detection

```typescript
// models/ObjectDetector.ts
import * as tf from '@tensorflow/tfjs';
import * as cocoSsd from '@tensorflow-models/coco-ssd';
import { decodeJpeg } from '@tensorflow/tfjs-react-native';

export interface DetectedObject {
  class: string;
  score: number;
  bbox: [number, number, number, number]; // [x, y, width, height]
}

export class ObjectDetector {
  private model: cocoSsd.ObjectDetection | null = null;
  
  async load(): Promise<void> {
    if (this.model) return;
    
    console.log('Loading COCO-SSD...');
    this.model = await cocoSsd.load({
      base: 'mobilenet_v2'
    });
    console.log('COCO-SSD loaded!');
  }
  
  async detect(
    imageData: Uint8Array,
    maxDetections = 20,
    minScore = 0.5
  ): Promise<DetectedObject[]> {
    if (!this.model) {
      throw new Error('Model not loaded');
    }
    
    const imageTensor = decodeJpeg(imageData);
    
    try {
      const predictions = await this.model.detect(
        imageTensor,
        maxDetections,
        minScore
      );
      
      return predictions.map(p => ({
        class: p.class,
        score: p.score,
        bbox: p.bbox as [number, number, number, number]
      }));
    } finally {
      imageTensor.dispose();
    }
  }
  
  isLoaded(): boolean {
    return this.model !== null;
  }
}

export const objectDetector = new ObjectDetector();
```

---

## 3. Custom Models

### 3.1 Simple Custom Model

```typescript
// models/CustomModel.ts
import * as tf from '@tensorflow/tfjs';

export class SentimentAnalyzer {
  private model: tf.LayersModel | null = null;
  private vocabulary: Map<string, number> = new Map();
  private maxLength = 100;
  
  async loadFromUrl(modelUrl: string, vocabUrl: string): Promise<void> {
    // Load model
    this.model = await tf.loadLayersModel(modelUrl);
    
    // Load vocabulary
    const response = await fetch(vocabUrl);
    const vocabData = await response.json();
    this.vocabulary = new Map(Object.entries(vocabData));
    
    console.log('Sentiment analyzer loaded!');
  }
  
  private tokenize(text: string): number[] {
    const words = text.toLowerCase()
      .replace(/[^a-zA-Zก-๙\s]/g, '')
      .split(/\s+/)
      .filter(w => w.length > 0);
    
    const tokens = words.map(word => 
      this.vocabulary.get(word) ?? 0 // 0 = unknown
    );
    
    // Pad or truncate to maxLength
    if (tokens.length >= this.maxLength) {
      return tokens.slice(0, this.maxLength);
    }
    
    return [...tokens, ...Array(this.maxLength - tokens.length).fill(0)];
  }
  
  async analyze(text: string): Promise<{
    sentiment: 'positive' | 'negative' | 'neutral';
    confidence: number;
    scores: { positive: number; negative: number; neutral: number };
  }> {
    if (!this.model) throw new Error('Model not loaded');
    
    const tokens = this.tokenize(text);
    const inputTensor = tf.tensor2d([tokens], [1, this.maxLength]);
    
    try {
      const prediction = this.model.predict(inputTensor) as tf.Tensor;
      const scores = Array.from(await prediction.data()) as number[];
      prediction.dispose();
      
      const [negativeScore, neutralScore, positiveScore] = scores;
      const maxIndex = scores.indexOf(Math.max(...scores));
      const sentiments: Array<'negative' | 'neutral' | 'positive'> = [
        'negative', 'neutral', 'positive'
      ];
      
      return {
        sentiment: sentiments[maxIndex],
        confidence: scores[maxIndex],
        scores: {
          negative: negativeScore,
          neutral: neutralScore,
          positive: positiveScore
        }
      };
    } finally {
      inputTensor.dispose();
    }
  }
}
```

---

## 4. Image Classification Component

```tsx
// components/ImageClassifierComponent.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  ScrollView,
  Alert
} from 'react-native';
import { launchImageLibrary, launchCamera } from 'react-native-image-picker';
import RNFS from 'react-native-fs';
import { imageClassifier } from '../models/ImageClassifier';

interface ClassificationResult {
  className: string;
  probability: number;
}

const ImageClassifierComponent: React.FC = () => {
  const [imageUri, setImageUri] = useState<string | null>(null);
  const [results, setResults] = useState<ClassificationResult[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [modelLoaded, setModelLoaded] = useState(false);
  
  const loadModel = useCallback(async () => {
    setIsLoading(true);
    try {
      await imageClassifier.load();
      setModelLoaded(true);
    } catch (error) {
      Alert.alert('Error', 'ไม่สามารถโหลด model ได้');
    } finally {
      setIsLoading(false);
    }
  }, []);
  
  const classifyImage = useCallback(async (uri: string) => {
    if (!modelLoaded) {
      await loadModel();
    }
    
    setIsLoading(true);
    setResults([]);
    
    try {
      // Read image as base64
      const base64 = await RNFS.readFile(uri, 'base64');
      const predictions = await imageClassifier.classifyFromBase64(base64, 5);
      setResults(predictions);
    } catch (error) {
      Alert.alert('Error', 'ไม่สามารถจำแนกภาพได้');
    } finally {
      setIsLoading(false);
    }
  }, [modelLoaded, loadModel]);
  
  const pickImage = useCallback(async () => {
    launchImageLibrary(
      { mediaType: 'photo', quality: 0.8 },
      async (response) => {
        if (response.assets?.[0]?.uri) {
          const uri = response.assets[0].uri;
          setImageUri(uri);
          await classifyImage(uri);
        }
      }
    );
  }, [classifyImage]);
  
  const takePhoto = useCallback(async () => {
    launchCamera(
      { mediaType: 'photo', quality: 0.8 },
      async (response) => {
        if (response.assets?.[0]?.uri) {
          const uri = response.assets[0].uri;
          setImageUri(uri);
          await classifyImage(uri);
        }
      }
    );
  }, [classifyImage]);
  
  const formatProbability = (p: number) => `${(p * 100).toFixed(1)}%`;
  
  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>Image Classifier</Text>
      <Text style={styles.subtitle}>AI จำแนกรูปภาพด้วย MobileNet</Text>
      
      {/* Image Display */}
      {imageUri ? (
        <Image source={{ uri: imageUri }} style={styles.image} resizeMode="contain" />
      ) : (
        <View style={styles.imagePlaceholder}>
          <Text style={styles.placeholderText}>ยังไม่มีรูปภาพ</Text>
          <Text style={styles.placeholderSubtext}>เลือกหรือถ่ายรูปเพื่อจำแนก</Text>
        </View>
      )}
      
      {/* Buttons */}
      <View style={styles.buttonRow}>
        <TouchableOpacity
          style={[styles.button, styles.galleryButton]}
          onPress={pickImage}
          disabled={isLoading}
        >
          <Text style={styles.buttonText}>🖼️ เลือกรูป</Text>
        </TouchableOpacity>
        
        <TouchableOpacity
          style={[styles.button, styles.cameraButton]}
          onPress={takePhoto}
          disabled={isLoading}
        >
          <Text style={styles.buttonText}>📷 ถ่ายรูป</Text>
        </TouchableOpacity>
      </View>
      
      {/* Loading */}
      {isLoading && (
        <View style={styles.loadingContainer}>
          <ActivityIndicator size="large" color="#007AFF" />
          <Text style={styles.loadingText}>
            {modelLoaded ? 'กำลังจำแนก...' : 'กำลังโหลด AI model...'}
          </Text>
        </View>
      )}
      
      {/* Results */}
      {results.length > 0 && (
        <View style={styles.resultsContainer}>
          <Text style={styles.resultsTitle}>ผลการจำแนก</Text>
          
          {results.map((result, index) => (
            <View key={index} style={styles.resultRow}>
              <View style={styles.resultInfo}>
                <Text style={styles.className}>
                  {index + 1}. {result.className}
                </Text>
              </View>
              <View style={styles.probabilityContainer}>
                <View
                  style={[
                    styles.probabilityBar,
                    { width: `${result.probability * 100}%` }
                  ]}
                />
                <Text style={styles.probabilityText}>
                  {formatProbability(result.probability)}
                </Text>
              </View>
            </View>
          ))}
        </View>
      )}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    textAlign: 'center',
    marginTop: 20,
    marginBottom: 4
  },
  subtitle: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    marginBottom: 20
  },
  image: {
    width: '100%',
    height: 300,
    backgroundColor: '#ddd'
  },
  imagePlaceholder: {
    width: '100%',
    height: 250,
    backgroundColor: '#e0e0e0',
    justifyContent: 'center',
    alignItems: 'center'
  },
  placeholderText: { fontSize: 18, color: '#888', marginBottom: 8 },
  placeholderSubtext: { fontSize: 14, color: '#aaa' },
  buttonRow: {
    flexDirection: 'row',
    paddingHorizontal: 16,
    paddingVertical: 12,
    gap: 12
  },
  button: {
    flex: 1,
    padding: 14,
    borderRadius: 10,
    alignItems: 'center'
  },
  galleryButton: { backgroundColor: '#007AFF' },
  cameraButton: { backgroundColor: '#34C759' },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  loadingContainer: {
    alignItems: 'center',
    padding: 20
  },
  loadingText: { marginTop: 10, fontSize: 14, color: '#666' },
  resultsContainer: {
    margin: 16,
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3
  },
  resultsTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 12,
    color: '#1a1a1a'
  },
  resultRow: { marginBottom: 12 },
  resultInfo: { marginBottom: 4 },
  className: { fontSize: 15, color: '#333' },
  probabilityContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    height: 24,
    backgroundColor: '#f0f0f0',
    borderRadius: 12,
    overflow: 'hidden'
  },
  probabilityBar: {
    height: '100%',
    backgroundColor: '#007AFF',
    borderRadius: 12
  },
  probabilityText: {
    position: 'absolute',
    right: 8,
    fontSize: 13,
    fontWeight: '600',
    color: '#333'
  }
});

export default ImageClassifierComponent;
```

---

## 5. Workshop: Object Detection App

```tsx
// screens/ObjectDetectionScreen.tsx
import React, { useState, useRef, useCallback, useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  ScrollView,
  Dimensions
} from 'react-native';
import { Camera, useCameraDevices, useFrameProcessor } from 'react-native-vision-camera';
import { objectDetector, DetectedObject } from '../models/ObjectDetector';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

const LABEL_COLORS: Record<string, string> = {
  person: '#FF6B6B',
  car: '#4ECDC4',
  dog: '#45B7D1',
  cat: '#96E6A1',
  bottle: '#DDA0DD',
  default: '#007AFF'
};

const DetectionOverlay: React.FC<{
  objects: DetectedObject[];
  imageWidth: number;
  imageHeight: number;
}> = ({ objects, imageWidth, imageHeight }) => {
  const scaleX = SCREEN_WIDTH / imageWidth;
  const scaleY = (SCREEN_WIDTH * imageHeight / imageWidth) / imageHeight;
  
  return (
    <View style={StyleSheet.absoluteFillObject} pointerEvents="none">
      {objects.map((obj, index) => {
        const [x, y, width, height] = obj.bbox;
        const color = LABEL_COLORS[obj.class] || LABEL_COLORS.default;
        
        return (
          <View key={index}>
            {/* Bounding box */}
            <View
              style={[
                styles.boundingBox,
                {
                  left: x * scaleX,
                  top: y * scaleY,
                  width: width * scaleX,
                  height: height * scaleY,
                  borderColor: color
                }
              ]}
            />
            {/* Label */}
            <View
              style={[
                styles.labelContainer,
                {
                  left: x * scaleX,
                  top: y * scaleY - 24,
                  backgroundColor: color
                }
              ]}
            >
              <Text style={styles.labelText}>
                {obj.class} {(obj.score * 100).toFixed(0)}%
              </Text>
            </View>
          </View>
        );
      })}
    </View>
  );
};

const ObjectDetectionScreen: React.FC = () => {
  const [detections, setDetections] = useState<DetectedObject[]>([]);
  const [modelLoaded, setModelLoaded] = useState(false);
  const [isProcessing, setIsProcessing] = useState(false);
  const [fps, setFps] = useState(0);
  
  const devices = useCameraDevices();
  const device = devices.back;
  
  const frameCount = useRef(0);
  const lastFpsTime = useRef(Date.now());
  
  useEffect(() => {
    const loadModel = async () => {
      try {
        await objectDetector.load();
        setModelLoaded(true);
      } catch (error) {
        console.error('Failed to load model:', error);
      }
    };
    loadModel();
  }, []);
  
  // Process frame for detection
  const processFrame = useCallback(async (imageData: Uint8Array) => {
    if (!modelLoaded || isProcessing) return;
    
    setIsProcessing(true);
    try {
      const objects = await objectDetector.detect(imageData, 10, 0.5);
      setDetections(objects);
      
      // Update FPS
      frameCount.current++;
      const now = Date.now();
      const elapsed = now - lastFpsTime.current;
      if (elapsed >= 1000) {
        setFps(Math.round(frameCount.current * 1000 / elapsed));
        frameCount.current = 0;
        lastFpsTime.current = now;
      }
    } finally {
      setIsProcessing(false);
    }
  }, [modelLoaded, isProcessing]);
  
  if (!device) {
    return (
      <View style={styles.center}>
        <Text>กล้องไม่พร้อมใช้งาน</Text>
      </View>
    );
  }
  
  return (
    <View style={styles.container}>
      {/* Stats Overlay */}
      <View style={styles.statsOverlay}>
        <Text style={styles.statsText}>
          {modelLoaded ? `${fps} FPS | ${detections.length} objects` : 'Loading AI...'}
        </Text>
      </View>
      
      {/* Detection boxes would overlay camera */}
      <DetectionOverlay
        objects={detections}
        imageWidth={640}
        imageHeight={480}
      />
      
      {/* Results List */}
      <View style={styles.resultsList}>
        <Text style={styles.resultsHeader}>ตรวจพบ:</Text>
        {detections.length === 0 ? (
          <Text style={styles.noDetection}>ยังไม่พบวัตถุ</Text>
        ) : (
          detections.map((obj, i) => (
            <View key={i} style={styles.detectionItem}>
              <View
                style={[
                  styles.detectionDot,
                  { backgroundColor: LABEL_COLORS[obj.class] || LABEL_COLORS.default }
                ]}
              />
              <Text style={styles.detectionText}>
                {obj.class} - {(obj.score * 100).toFixed(1)}%
              </Text>
            </View>
          ))
        )}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  statsOverlay: {
    position: 'absolute',
    top: 50,
    left: 16,
    backgroundColor: 'rgba(0,0,0,0.6)',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 20,
    zIndex: 100
  },
  statsText: {
    color: '#0f0',
    fontFamily: 'monospace',
    fontSize: 12
  },
  boundingBox: {
    position: 'absolute',
    borderWidth: 2,
    borderRadius: 4
  },
  labelContainer: {
    position: 'absolute',
    paddingHorizontal: 6,
    paddingVertical: 2,
    borderRadius: 4
  },
  labelText: {
    color: '#fff',
    fontSize: 11,
    fontWeight: '600'
  },
  resultsList: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    backgroundColor: 'rgba(0,0,0,0.8)',
    padding: 16,
    maxHeight: 200
  },
  resultsHeader: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
    marginBottom: 8
  },
  noDetection: {
    color: '#aaa',
    fontSize: 14,
    textAlign: 'center',
    paddingVertical: 8
  },
  detectionItem: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 4
  },
  detectionDot: {
    width: 10,
    height: 10,
    borderRadius: 5,
    marginRight: 8
  },
  detectionText: {
    color: '#fff',
    fontSize: 14
  }
});

export default ObjectDetectionScreen;
```

---

## Tips สำหรับ ML บน Mobile

1. **Model size** - ใช้ quantized models เพื่อลดขนาด
2. **Inference frequency** - อย่า run inference ทุก frame ใช้ throttle
3. **Memory management** - dispose tensors เสมอหลังใช้
4. **Warm up** - warm up model ครั้งแรกเพื่อ JIT compilation
5. **Fallback** - มี fallback เมื่อ ML ล้มเหลว

## สรุป

TensorFlow.js ใน React Native ช่วยให้:
1. Run ML models on-device (ไม่ต้องส่งข้อมูลขึ้น server)
2. Privacy-preserving inference
3. Works offline
4. Low latency predictions
5. ใช้งาน pre-trained models ได้ทันที

ประยุกต์ใช้สำหรับ: image classification, object detection, face recognition, text analysis, speech recognition และอื่นๆ อีกมากมาย
