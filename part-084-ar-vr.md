# Part 084: AR/VR Features

## AR/VR ใน React Native

Augmented Reality (AR) และ Virtual Reality (VR) ใน mobile apps ช่วยสร้างประสบการณ์ที่น่าสนใจให้ผู้ใช้ เช่น overlay ข้อมูลบนโลกจริง (AR) หรือสภาพแวดล้อมเสมือนจริง (VR)

## Technologies ที่ใช้

- **ViroReact** - React wrapper สำหรับ AR/VR
- **ARKit** (iOS) - Apple's AR framework
- **ARCore** (Android) - Google's AR framework
- **React Native Vision Camera** - กล้องที่มี AR capability
- **Expo Camera** - กล้องสำหรับ Expo apps

---

## 1. ViroReact Setup

### 1.1 ติดตั้ง

```bash
npm install @viro-community/react-viro

# iOS
cd ios && pod install

# Android - ต้องเพิ่ม config
```

### 1.2 Android Configuration

```groovy
// android/app/build.gradle
android {
  defaultConfig {
    ndk {
      abiFilters 'armeabi-v7a', 'arm64-v8a'
    }
  }
}

dependencies {
  implementation 'com.viromedia:viro_renderer:+'
}
```

---

## 2. AR Scene Basics

### 2.1 สร้าง AR Scene ง่ายๆ

```tsx
// components/BasicARScene.tsx
import React, { useState, useRef } from 'react';
import {
  ViroARScene,
  ViroARSceneNavigator,
  ViroText,
  ViroBox,
  ViroSphere,
  ViroNode,
  ViroAnimations,
  ViroAmbientLight,
  ViroSpotLight,
  ViroMaterials
} from '@viro-community/react-viro';

// Register materials
ViroMaterials.createMaterials({
  blueMaterial: {
    diffuseColor: '#4A90E2'
  },
  metalMaterial: {
    lightingModel: 'PBR',
    diffuseColor: '#FFFFFF',
    roughness: 0.2,
    metalness: 0.8
  },
  woodTexture: {
    diffuseTexture: require('../assets/textures/wood.jpg'),
    normalTexture: require('../assets/textures/wood_normal.jpg')
  }
});

// Register animations
ViroAnimations.registerAnimations({
  rotate: {
    properties: {
      rotateY: '+=360'
    },
    duration: 3000
  },
  bounce: {
    properties: {
      positionY: ['-=0.1', '+=0.1']
    },
    duration: 500,
    easing: 'bounce'
  },
  fadeIn: {
    properties: {
      opacity: [0, 1]
    },
    duration: 1000
  }
});

const ARSceneContent: React.FC = () => {
  const [selectedObject, setSelectedObject] = useState<string | null>(null);
  const [objects, setObjects] = useState([
    { id: '1', type: 'box', position: [0, 0, -1], color: '#FF6B6B' },
    { id: '2', type: 'sphere', position: [0.5, 0, -1.5], color: '#4ECDC4' },
    { id: '3', type: 'text', position: [-0.5, 0.5, -1], text: 'Hello AR!' }
  ]);
  
  const handleObjectClick = (id: string) => {
    setSelectedObject(prev => prev === id ? null : id);
  };
  
  return (
    <ViroARScene>
      {/* Lighting */}
      <ViroAmbientLight color="#FFFFFF" intensity={200} />
      <ViroSpotLight
        position={[0, 3, 1]}
        color="#FFFFFF"
        direction={[0, -1, 0]}
        attenuationStartDistance={5}
        attenuationEndDistance={10}
        innerAngle={5}
        outerAngle={20}
      />
      
      {/* Info Text */}
      <ViroText
        text="แตะวัตถุเพื่อเลือก"
        position={[0, 1, -2]}
        scale={[0.5, 0.5, 0.5]}
        style={{ fontFamily: 'Arial', fontSize: 20, color: '#FFFFFF' }}
      />
      
      {/* 3D Box */}
      <ViroNode
        position={[0, 0, -1]}
        onClick={() => handleObjectClick('box')}
        animation={{
          name: selectedObject === 'box' ? 'rotate' : '',
          run: true,
          loop: true
        }}
      >
        <ViroBox
          height={0.3}
          width={0.3}
          length={0.3}
          materials={['blueMaterial']}
        />
        {selectedObject === 'box' && (
          <ViroText
            text="Box!"
            position={[0, 0.3, 0]}
            scale={[0.3, 0.3, 0.3]}
            style={{ color: '#FFF', fontSize: 20 }}
          />
        )}
      </ViroNode>
      
      {/* 3D Sphere */}
      <ViroNode
        position={[0.7, 0, -1.5]}
        onClick={() => handleObjectClick('sphere')}
        animation={{
          name: selectedObject === 'sphere' ? 'bounce' : '',
          run: true,
          loop: true
        }}
      >
        <ViroSphere
          radius={0.15}
          materials={['metalMaterial']}
        />
      </ViroNode>
    </ViroARScene>
  );
};

export const BasicARApp: React.FC = () => {
  return (
    <ViroARSceneNavigator
      autofocus={true}
      initialScene={{ scene: ARSceneContent }}
      style={{ flex: 1 }}
    />
  );
};
```

---

## 3. AR Image Markers

### 3.1 Image Tracking

```tsx
// components/ARImageMarker.tsx
import React, { useState } from 'react';
import {
  ViroARScene,
  ViroARSceneNavigator,
  ViroARImageMarker,
  ViroARTrackingTargets,
  Viro3DObject,
  ViroAmbientLight,
  ViroAnimations,
  ViroText,
  ViroNode
} from '@viro-community/react-viro';

// Register tracking targets (AR markers)
ViroARTrackingTargets.createTargets({
  businessCard: {
    source: require('../assets/markers/business_card.jpg'),
    orientation: 'Up',
    physicalWidth: 0.08 // 8cm wide
  },
  productBox: {
    source: require('../assets/markers/product.jpg'),
    orientation: 'Up',
    physicalWidth: 0.2 // 20cm wide
  }
});

ViroAnimations.registerAnimations({
  scaleUp: {
    properties: {
      scaleX: [0, 1],
      scaleY: [0, 1],
      scaleZ: [0, 1]
    },
    duration: 500,
    easing: 'easeInEaseOut'
  },
  float: {
    properties: {
      positionY: ['+=0.05', '-=0.05']
    },
    duration: 1500,
    easing: 'easeInEaseOut'
  }
});

interface CardInfo {
  name: string;
  title: string;
  company: string;
  email: string;
  phone: string;
}

const ARBusinessCardScene: React.FC<{ cardInfo: CardInfo }> = ({ cardInfo }) => {
  const [isVisible, setIsVisible] = useState(false);
  
  const handleAnchorFound = () => {
    setIsVisible(true);
  };
  
  const handleAnchorRemoved = () => {
    setIsVisible(false);
  };
  
  return (
    <ViroARScene>
      <ViroAmbientLight color="#FFFFFF" intensity={200} />
      
      <ViroARImageMarker
        target="businessCard"
        onAnchorFound={handleAnchorFound}
        onAnchorRemoved={handleAnchorRemoved}
      >
        {isVisible && (
          <ViroNode
            position={[0, 0.1, 0]}
            animation={{ name: 'scaleUp', run: true }}
          >
            {/* 3D Info Card */}
            <ViroNode
              position={[0, 0, 0]}
              animation={{ name: 'float', run: true, loop: true }}
            >
              {/* Name */}
              <ViroText
                text={cardInfo.name}
                position={[0, 0.15, 0]}
                scale={[0.2, 0.2, 0.2]}
                style={{
                  fontFamily: 'Arial',
                  fontSize: 30,
                  color: '#1a1a2e',
                  fontWeight: 'bold'
                }}
                textClipMode="clipToBounds"
              />
              
              {/* Title */}
              <ViroText
                text={cardInfo.title}
                position={[0, 0.05, 0]}
                scale={[0.15, 0.15, 0.15]}
                style={{
                  fontFamily: 'Arial',
                  fontSize: 20,
                  color: '#007AFF'
                }}
              />
              
              {/* Company */}
              <ViroText
                text={cardInfo.company}
                position={[0, -0.03, 0]}
                scale={[0.15, 0.15, 0.15]}
                style={{
                  fontFamily: 'Arial',
                  fontSize: 18,
                  color: '#666666'
                }}
              />
              
              {/* Contact Info */}
              <ViroText
                text={`📧 ${cardInfo.email}`}
                position={[0, -0.1, 0]}
                scale={[0.12, 0.12, 0.12]}
                style={{
                  fontFamily: 'Arial',
                  fontSize: 16,
                  color: '#333333'
                }}
              />
              
              <ViroText
                text={`📱 ${cardInfo.phone}`}
                position={[0, -0.17, 0]}
                scale={[0.12, 0.12, 0.12]}
                style={{
                  fontFamily: 'Arial',
                  fontSize: 16,
                  color: '#333333'
                }}
              />
            </ViroNode>
          </ViroNode>
        )}
      </ViroARImageMarker>
    </ViroARScene>
  );
};

// Main Component
const ARBusinessCard: React.FC = () => {
  const cardInfo: CardInfo = {
    name: 'สมชาย ดีมาก',
    title: 'Senior Developer',
    company: 'Tech Company Co., Ltd.',
    email: 'somchai@tech.com',
    phone: '081-234-5678'
  };
  
  return (
    <ViroARSceneNavigator
      autofocus={true}
      initialScene={{
        scene: () => <ARBusinessCardScene cardInfo={cardInfo} />
      }}
      style={{ flex: 1 }}
    />
  );
};

export default ARBusinessCard;
```

---

## 4. 3D Objects ใน AR

### 4.1 Loading 3D Models

```tsx
// components/AR3DProduct.tsx
import React, { useState, useRef } from 'react';
import {
  ViroARScene,
  ViroARSceneNavigator,
  Viro3DObject,
  ViroAmbientLight,
  ViroDirectionalLight,
  ViroNode,
  ViroAnimations
} from '@viro-community/react-viro';

ViroAnimations.registerAnimations({
  rotateProduct: {
    properties: { rotateY: '+=360' },
    duration: 8000
  },
  pulseScale: {
    properties: { scaleX: ['+=0.1', '-=0.1'], scaleY: ['+=0.1', '-=0.1'], scaleZ: ['+=0.1', '-=0.1'] },
    duration: 1000,
    easing: 'easeInEaseOut'
  }
});

interface ProductInfo {
  modelPath: string;
  resourcePath: string;
  name: string;
  description: string;
  price: number;
}

const ARProductViewer: React.FC<{ product: ProductInfo }> = ({ product }) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isSelected, setIsSelected] = useState(false);
  const [scale, setScale] = useState([0.3, 0.3, 0.3]);
  
  const handlePinch = (pinchState: number, scaleFactor: number) => {
    if (pinchState === 3) {
      const newScale = scale.map(s => Math.max(0.1, Math.min(s * scaleFactor, 2.0)));
      setScale(newScale);
    }
  };
  
  return (
    <ViroARScene>
      <ViroAmbientLight color="#FFFFFF" intensity={300} />
      <ViroDirectionalLight
        color="#FFFFFF"
        direction={[-1, -1, -1]}
        intensity={300}
      />
      
      <ViroNode
        position={[0, -0.3, -1.5]}
        onPinch={handlePinch}
        onClick={() => setIsSelected(prev => !prev)}
      >
        <Viro3DObject
          source={{ uri: product.modelPath }}
          resources={[{ uri: product.resourcePath }]}
          type="OBJ"
          scale={scale}
          position={[0, 0, 0]}
          onLoadStart={() => console.log('Loading model...')}
          onLoadEnd={() => {
            setIsLoaded(true);
            console.log('Model loaded!');
          }}
          onError={(error) => console.error('Model error:', error)}
          animation={{
            name: 'rotateProduct',
            run: !isSelected,
            loop: true
          }}
        />
        
        {isSelected && isLoaded && (
          <ViroNode position={[0, 0.5, 0]}>
            <ViroText
              text={product.name}
              style={{
                fontSize: 24,
                fontFamily: 'Arial',
                color: '#1a1a2e',
                fontWeight: 'bold'
              }}
              scale={[0.15, 0.15, 0.15]}
            />
            <ViroText
              text={`฿${product.price.toLocaleString()}`}
              position={[0, -0.1, 0]}
              style={{
                fontSize: 20,
                fontFamily: 'Arial',
                color: '#34C759'
              }}
              scale={[0.15, 0.15, 0.15]}
            />
            <ViroText
              text={product.description}
              position={[0, -0.2, 0]}
              style={{
                fontSize: 14,
                fontFamily: 'Arial',
                color: '#666666'
              }}
              scale={[0.1, 0.1, 0.1]}
              maxLines={2}
            />
          </ViroNode>
        )}
      </ViroNode>
    </ViroARScene>
  );
};
```

---

## 5. Workshop: AR Business Card App

### Step 1: App Structure

```tsx
// App.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  SafeAreaView,
  Platform,
  Alert
} from 'react-native';
import { check, request, PERMISSIONS, RESULTS } from 'react-native-permissions';
import ARBusinessCard from './components/ARBusinessCard';

const App: React.FC = () => {
  const [mode, setMode] = useState<'menu' | 'ar' | 'edit'>('menu');
  const [cardInfo, setCardInfo] = useState({
    name: 'ชื่อของคุณ',
    title: 'ตำแหน่งของคุณ',
    company: 'บริษัทของคุณ',
    email: 'email@example.com',
    phone: '081-000-0000'
  });
  
  const requestCameraPermission = async (): Promise<boolean> => {
    const permission = Platform.OS === 'ios'
      ? PERMISSIONS.IOS.CAMERA
      : PERMISSIONS.ANDROID.CAMERA;
    
    const result = await check(permission);
    
    if (result === RESULTS.GRANTED) return true;
    
    if (result === RESULTS.DENIED) {
      const requestResult = await request(permission);
      return requestResult === RESULTS.GRANTED;
    }
    
    return false;
  };
  
  const handleARMode = async () => {
    const hasPermission = await requestCameraPermission();
    if (hasPermission) {
      setMode('ar');
    } else {
      Alert.alert(
        'ต้องการ Permission',
        'แอปต้องการสิทธิ์ใช้กล้องสำหรับ AR',
        [{ text: 'ตกลง' }]
      );
    }
  };
  
  if (mode === 'ar') {
    return (
      <View style={{ flex: 1 }}>
        <ARBusinessCard cardInfo={cardInfo} />
        <SafeAreaView style={styles.arOverlay}>
          <TouchableOpacity
            style={styles.backButton}
            onPress={() => setMode('menu')}
          >
            <Text style={styles.backButtonText}>← กลับ</Text>
          </TouchableOpacity>
          <Text style={styles.arHint}>
            ชี้กล้องไปที่นามบัตรของคุณ
          </Text>
        </SafeAreaView>
      </View>
    );
  }
  
  return (
    <SafeAreaView style={styles.container}>
      <Text style={styles.title}>AR Business Card</Text>
      <Text style={styles.subtitle}>นามบัตรดิจิทัลด้วย AR</Text>
      
      {/* Card Preview */}
      <View style={styles.cardPreview}>
        <Text style={styles.previewName}>{cardInfo.name}</Text>
        <Text style={styles.previewTitle}>{cardInfo.title}</Text>
        <Text style={styles.previewCompany}>{cardInfo.company}</Text>
        <Text style={styles.previewContact}>{cardInfo.email}</Text>
        <Text style={styles.previewContact}>{cardInfo.phone}</Text>
      </View>
      
      <TouchableOpacity style={styles.arButton} onPress={handleARMode}>
        <Text style={styles.arButtonText}>เปิด AR View</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        style={styles.editButton}
        onPress={() => setMode('edit')}
      >
        <Text style={styles.editButtonText}>แก้ไขข้อมูล</Text>
      </TouchableOpacity>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
    padding: 20
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    textAlign: 'center',
    marginTop: 20,
    color: '#1a1a2e'
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
    marginBottom: 30
  },
  cardPreview: {
    backgroundColor: '#1a1a2e',
    borderRadius: 16,
    padding: 24,
    marginBottom: 30,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 8 },
    shadowOpacity: 0.3,
    shadowRadius: 12,
    elevation: 8
  },
  previewName: {
    fontSize: 22,
    fontWeight: 'bold',
    color: '#fff',
    marginBottom: 6
  },
  previewTitle: {
    fontSize: 16,
    color: '#007AFF',
    marginBottom: 4
  },
  previewCompany: {
    fontSize: 14,
    color: '#aaa',
    marginBottom: 16
  },
  previewContact: {
    fontSize: 14,
    color: '#ccc',
    marginBottom: 4
  },
  arButton: {
    backgroundColor: '#007AFF',
    paddingVertical: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginBottom: 12
  },
  arButtonText: {
    color: '#fff',
    fontSize: 18,
    fontWeight: '600'
  },
  editButton: {
    borderWidth: 2,
    borderColor: '#007AFF',
    paddingVertical: 16,
    borderRadius: 12,
    alignItems: 'center'
  },
  editButtonText: {
    color: '#007AFF',
    fontSize: 18,
    fontWeight: '600'
  },
  arOverlay: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingTop: 10
  },
  backButton: {
    backgroundColor: 'rgba(0,0,0,0.5)',
    padding: 10,
    borderRadius: 8
  },
  backButtonText: {
    color: '#fff',
    fontSize: 16
  },
  arHint: {
    color: '#fff',
    fontSize: 14,
    backgroundColor: 'rgba(0,0,0,0.5)',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 20
  }
});

export default App;
```

---

## Tips สำหรับ AR Development

1. **Lighting** - AR ต้องการแสงสว่างพอเพียง
2. **Marker quality** - ใช้ markers ที่มีรายละเอียดชัดเจน
3. **Performance** - 3D objects ควรมี polygon count ต่ำ
4. **User guidance** - แนะนำ user ว่าต้องทำอะไร
5. **Fallback** - รองรับกรณีที่ AR ไม่พร้อมใช้งาน

## สรุป

AR ใน React Native ช่วยสร้าง:
1. ประสบการณ์ใช้งานที่น่าสนใจ
2. Visualization ของสินค้าก่อนซื้อ
3. Navigation AR
4. Educational AR experiences
5. Marketing campaigns ที่สร้างสรรค์
