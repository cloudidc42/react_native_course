# Part 042: Camera และ Image Picker ใน React Native

## สารบัญ
1. [แนะนำ Camera และ Image Picker](#introduction)
2. [react-native-camera](#react-native-camera)
3. [react-native-image-picker](#react-native-image-picker)
4. [Expo Camera](#expo-camera)
5. [Photo Compression](#photo-compression)
6. [Upload to Server](#upload-to-server)
7. [Workshop: Profile Picture Upload](#workshop)

---

## 1. แนะนำ Camera และ Image Picker {#introduction}

การทำงานกับ Camera และ Image ใน React Native เป็นส่วนสำคัญของแอปมือถือสมัยใหม่

### ตัวเลือกที่นิยมใช้

| Library | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| react-native-camera | ฟีเจอร์ครบ | ขนาดใหญ่ |
| react-native-image-picker | เบา ใช้ง่าย | ฟีเจอร์น้อย |
| Expo Camera | Simple API | ต้องใช้ Expo |
| react-native-vision-camera | ประสิทธิภาพสูง | ซับซ้อน |

### ติดตั้ง Dependencies

```bash
# react-native-image-picker (แนะนำสำหรับการเลือกรูป)
npm install react-native-image-picker

# react-native-vision-camera (แนะนำสำหรับถ่ายภาพ)
npm install react-native-vision-camera

# Image compression
npm install react-native-image-resizer

# สำหรับ Expo
npx expo install expo-camera expo-image-picker expo-media-library
```

---

## 2. react-native-camera / Vision Camera {#react-native-camera}

### การตั้งค่า Permissions

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

```xml
<!-- ios/YourApp/Info.plist -->
<key>NSCameraUsageDescription</key>
<string>แอปต้องการเข้าถึงกล้องเพื่อถ่ายรูป</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>แอปต้องการเข้าถึงรูปภาพของคุณ</string>
<key>NSMicrophoneUsageDescription</key>
<string>แอปต้องการเข้าถึงไมโครโฟนเพื่อบันทึกวิดีโอ</string>
```

### Camera Component ด้วย Vision Camera

```typescript
// src/components/CameraComponent.tsx
import React, { useRef, useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Dimensions,
  Alert,
} from 'react-native';
import {
  Camera,
  useCameraDevice,
  useCameraPermission,
  useFrameProcessor,
  CameraDevice,
} from 'react-native-vision-camera';
import Icon from 'react-native-vector-icons/MaterialIcons';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');

interface CameraComponentProps {
  onPhotoTaken: (path: string) => void;
  onClose: () => void;
}

const CameraComponent: React.FC<CameraComponentProps> = ({ onPhotoTaken, onClose }) => {
  const camera = useRef<Camera>(null);
  const [cameraPosition, setCameraPosition] = useState<'front' | 'back'>('back');
  const [flash, setFlash] = useState<'off' | 'on' | 'auto'>('off');
  const [isRecording, setIsRecording] = useState(false);
  const [zoom, setZoom] = useState(0);

  const { hasPermission, requestPermission } = useCameraPermission();
  const device = useCameraDevice(cameraPosition);

  // ขอ Permission
  React.useEffect(() => {
    if (!hasPermission) {
      requestPermission();
    }
  }, [hasPermission, requestPermission]);

  // ถ่ายภาพ
  const takePhoto = useCallback(async () => {
    if (!camera.current) return;

    try {
      const photo = await camera.current.takePhoto({
        flash,
        qualityPrioritization: 'quality',
        enableAutoRedEyeReduction: true,
        enableAutoDistortionCorrection: true,
      });

      console.log('Photo taken:', photo.path);
      onPhotoTaken(`file://${photo.path}`);
    } catch (error) {
      console.error('Take photo error:', error);
      Alert.alert('Error', 'ไม่สามารถถ่ายรูปได้');
    }
  }, [camera, flash, onPhotoTaken]);

  // สลับกล้องหน้า/หลัง
  const toggleCamera = useCallback(() => {
    setCameraPosition(prev => prev === 'back' ? 'front' : 'back');
  }, []);

  // สลับ Flash
  const toggleFlash = useCallback(() => {
    setFlash(prev => {
      if (prev === 'off') return 'on';
      if (prev === 'on') return 'auto';
      return 'off';
    });
  }, []);

  const getFlashIcon = () => {
    switch (flash) {
      case 'on': return 'flash-on';
      case 'auto': return 'flash-auto';
      default: return 'flash-off';
    }
  };

  if (!hasPermission) {
    return (
      <View style={styles.permissionContainer}>
        <Icon name="camera-alt" size={64} color="#9E9E9E" />
        <Text style={styles.permissionText}>ต้องการสิทธิ์เข้าถึงกล้อง</Text>
        <TouchableOpacity style={styles.permissionButton} onPress={requestPermission}>
          <Text style={styles.permissionButtonText}>อนุญาต</Text>
        </TouchableOpacity>
      </View>
    );
  }

  if (!device) {
    return (
      <View style={styles.permissionContainer}>
        <Text style={styles.permissionText}>ไม่พบกล้อง</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <Camera
        ref={camera}
        style={StyleSheet.absoluteFill}
        device={device}
        isActive={true}
        photo={true}
        zoom={zoom}
        enableZoomGesture={true}
      />

      {/* Controls Overlay */}
      <View style={styles.overlay}>
        {/* Top Controls */}
        <View style={styles.topControls}>
          <TouchableOpacity style={styles.controlButton} onPress={onClose}>
            <Icon name="close" size={28} color="white" />
          </TouchableOpacity>

          <TouchableOpacity style={styles.controlButton} onPress={toggleFlash}>
            <Icon name={getFlashIcon()} size={28} color="white" />
          </TouchableOpacity>
        </View>

        {/* Grid Lines */}
        <View style={styles.gridContainer}>
          <View style={styles.gridLineHorizontal1} />
          <View style={styles.gridLineHorizontal2} />
          <View style={styles.gridLineVertical1} />
          <View style={styles.gridLineVertical2} />
        </View>

        {/* Bottom Controls */}
        <View style={styles.bottomControls}>
          <TouchableOpacity style={styles.galleryButton}>
            <Icon name="photo-library" size={28} color="white" />
          </TouchableOpacity>

          {/* Shutter Button */}
          <TouchableOpacity
            style={styles.shutterButton}
            onPress={takePhoto}
            activeOpacity={0.7}
          >
            <View style={styles.shutterInner} />
          </TouchableOpacity>

          <TouchableOpacity style={styles.flipButton} onPress={toggleCamera}>
            <Icon name="flip-camera-ios" size={28} color="white" />
          </TouchableOpacity>
        </View>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: 'black',
  },
  overlay: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'space-between',
  },
  topControls: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    padding: 20,
    paddingTop: 50,
  },
  controlButton: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  gridContainer: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'center',
    alignItems: 'center',
  },
  gridLineHorizontal1: {
    position: 'absolute',
    top: '33%',
    width: '100%',
    height: 1,
    backgroundColor: 'rgba(255,255,255,0.3)',
  },
  gridLineHorizontal2: {
    position: 'absolute',
    top: '66%',
    width: '100%',
    height: 1,
    backgroundColor: 'rgba(255,255,255,0.3)',
  },
  gridLineVertical1: {
    position: 'absolute',
    left: '33%',
    height: '100%',
    width: 1,
    backgroundColor: 'rgba(255,255,255,0.3)',
  },
  gridLineVertical2: {
    position: 'absolute',
    left: '66%',
    height: '100%',
    width: 1,
    backgroundColor: 'rgba(255,255,255,0.3)',
  },
  bottomControls: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    alignItems: 'center',
    padding: 20,
    paddingBottom: 40,
    backgroundColor: 'rgba(0,0,0,0.3)',
  },
  galleryButton: {
    width: 50,
    height: 50,
    justifyContent: 'center',
    alignItems: 'center',
  },
  shutterButton: {
    width: 80,
    height: 80,
    borderRadius: 40,
    backgroundColor: 'white',
    justifyContent: 'center',
    alignItems: 'center',
  },
  shutterInner: {
    width: 68,
    height: 68,
    borderRadius: 34,
    backgroundColor: 'white',
    borderWidth: 2,
    borderColor: '#333',
  },
  flipButton: {
    width: 50,
    height: 50,
    justifyContent: 'center',
    alignItems: 'center',
  },
  permissionContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    gap: 16,
    backgroundColor: '#F5F5F5',
  },
  permissionText: {
    fontSize: 16,
    color: '#666',
  },
  permissionButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 32,
    paddingVertical: 12,
    borderRadius: 8,
  },
  permissionButtonText: {
    color: 'white',
    fontWeight: 'bold',
  },
});

export default CameraComponent;
```

---

## 3. react-native-image-picker {#react-native-image-picker}

### การเลือกรูปภาพจาก Gallery หรือถ่ายจากกล้อง

```typescript
// src/hooks/useImagePicker.ts
import { useCallback } from 'react';
import {
  launchCamera,
  launchImageLibrary,
  ImagePickerResponse,
  MediaType,
  ImageLibraryOptions,
  CameraOptions,
} from 'react-native-image-picker';
import { Alert, ActionSheetIOS, Platform } from 'react-native';

interface PickedImage {
  uri: string;
  fileName: string;
  type: string;
  fileSize: number;
  width: number;
  height: number;
}

export function useImagePicker() {
  const commonOptions = {
    mediaType: 'photo' as MediaType,
    quality: 0.8 as any,
    maxWidth: 1920,
    maxHeight: 1920,
    includeBase64: false,
    includeExif: true,
  };

  const pickFromCamera = useCallback((): Promise<PickedImage | null> => {
    return new Promise((resolve) => {
      const options: CameraOptions = {
        ...commonOptions,
        saveToPhotos: true,
        cameraType: 'back',
      };

      launchCamera(options, (response: ImagePickerResponse) => {
        if (response.didCancel) {
          resolve(null);
          return;
        }

        if (response.errorCode) {
          if (response.errorCode === 'camera_unavailable') {
            Alert.alert('ข้อผิดพลาด', 'ไม่พบกล้องในอุปกรณ์นี้');
          } else if (response.errorCode === 'permission') {
            Alert.alert('ต้องการสิทธิ์', 'กรุณาอนุญาตการเข้าถึงกล้อง');
          } else {
            Alert.alert('ข้อผิดพลาด', response.errorMessage || 'เกิดข้อผิดพลาด');
          }
          resolve(null);
          return;
        }

        if (response.assets && response.assets[0]) {
          const asset = response.assets[0];
          resolve({
            uri: asset.uri!,
            fileName: asset.fileName || `photo_${Date.now()}.jpg`,
            type: asset.type || 'image/jpeg',
            fileSize: asset.fileSize || 0,
            width: asset.width || 0,
            height: asset.height || 0,
          });
        }
      });
    });
  }, []);

  const pickFromGallery = useCallback((multiple = false): Promise<PickedImage[] | null> => {
    return new Promise((resolve) => {
      const options: ImageLibraryOptions = {
        ...commonOptions,
        selectionLimit: multiple ? 10 : 1,
      };

      launchImageLibrary(options, (response: ImagePickerResponse) => {
        if (response.didCancel) {
          resolve(null);
          return;
        }

        if (response.errorCode) {
          Alert.alert('ข้อผิดพลาด', response.errorMessage || 'เกิดข้อผิดพลาด');
          resolve(null);
          return;
        }

        if (response.assets) {
          const images = response.assets.map(asset => ({
            uri: asset.uri!,
            fileName: asset.fileName || `image_${Date.now()}.jpg`,
            type: asset.type || 'image/jpeg',
            fileSize: asset.fileSize || 0,
            width: asset.width || 0,
            height: asset.height || 0,
          }));
          resolve(images);
        }
      });
    });
  }, []);

  const showImagePickerOptions = useCallback((): Promise<PickedImage | null> => {
    return new Promise((resolve) => {
      if (Platform.OS === 'ios') {
        ActionSheetIOS.showActionSheetWithOptions(
          {
            options: ['ยกเลิก', 'ถ่ายรูป', 'เลือกจากคลัง'],
            cancelButtonIndex: 0,
          },
          async (buttonIndex) => {
            if (buttonIndex === 1) {
              resolve(await pickFromCamera());
            } else if (buttonIndex === 2) {
              const images = await pickFromGallery(false);
              resolve(images ? images[0] : null);
            } else {
              resolve(null);
            }
          }
        );
      } else {
        Alert.alert(
          'เลือกรูปภาพ',
          '',
          [
            { text: 'ยกเลิก', style: 'cancel', onPress: () => resolve(null) },
            { text: 'ถ่ายรูป', onPress: async () => resolve(await pickFromCamera()) },
            {
              text: 'เลือกจากคลัง',
              onPress: async () => {
                const images = await pickFromGallery(false);
                resolve(images ? images[0] : null);
              },
            },
          ]
        );
      }
    });
  }, [pickFromCamera, pickFromGallery]);

  return { pickFromCamera, pickFromGallery, showImagePickerOptions };
}
```

---

## 4. Expo Camera {#expo-camera}

### การใช้งาน Expo Camera

```typescript
// src/screens/ExpoCameraScreen.tsx
import React, { useState, useRef, useEffect } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Image } from 'react-native';
import { Camera, CameraType, FlashMode, CameraView } from 'expo-camera';
import * as MediaLibrary from 'expo-media-library';
import * as ImageManipulator from 'expo-image-manipulator';

interface ExpoImageCaptureProps {
  onCapture: (uri: string) => void;
}

const ExpoCameraCapture: React.FC<ExpoImageCaptureProps> = ({ onCapture }) => {
  const [permission, requestPermission] = Camera.useCameraPermissions();
  const [mediaPermission, requestMediaPermission] = MediaLibrary.usePermissions();
  const [type, setType] = useState<CameraType>(CameraType.back);
  const [flash, setFlash] = useState<FlashMode>(FlashMode.off);
  const [capturedImage, setCapturedImage] = useState<string | null>(null);
  const cameraRef = useRef<CameraView>(null);

  const takePicture = async () => {
    if (!cameraRef.current) return;

    const photo = await cameraRef.current.takePictureAsync({
      quality: 0.8,
      base64: false,
      exif: true,
    });

    if (photo) {
      // Manipulate image
      const manipulated = await ImageManipulator.manipulateAsync(
        photo.uri,
        [{ resize: { width: 1080 } }],
        { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
      );

      setCapturedImage(manipulated.uri);
    }
  };

  const saveToGallery = async () => {
    if (!capturedImage) return;

    const asset = await MediaLibrary.createAssetAsync(capturedImage);
    await MediaLibrary.createAlbumAsync('MyApp', asset, false);
  };

  const confirmCapture = () => {
    if (capturedImage) {
      onCapture(capturedImage);
      setCapturedImage(null);
    }
  };

  if (!permission?.granted) {
    return (
      <View style={styles.permissionContainer}>
        <Text>ต้องการสิทธิ์เข้าถึงกล้อง</Text>
        <TouchableOpacity onPress={requestPermission}>
          <Text>อนุญาต</Text>
        </TouchableOpacity>
      </View>
    );
  }

  if (capturedImage) {
    return (
      <View style={styles.previewContainer}>
        <Image source={{ uri: capturedImage }} style={styles.preview} />
        <View style={styles.previewActions}>
          <TouchableOpacity onPress={() => setCapturedImage(null)}>
            <Text style={styles.retakeText}>ถ่ายใหม่</Text>
          </TouchableOpacity>
          <TouchableOpacity onPress={saveToGallery}>
            <Text style={styles.saveText}>บันทึก</Text>
          </TouchableOpacity>
          <TouchableOpacity onPress={confirmCapture} style={styles.confirmButton}>
            <Text style={styles.confirmText}>ใช้รูปนี้</Text>
          </TouchableOpacity>
        </View>
      </View>
    );
  }

  return (
    <CameraView
      ref={cameraRef}
      style={styles.camera}
      facing={type}
      flash={flash}
    >
      <View style={styles.cameraControls}>
        <TouchableOpacity
          onPress={() => setType(t =>
            t === CameraType.back ? CameraType.front : CameraType.back
          )}
        >
          <Text style={styles.flipText}>สลับกล้อง</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.captureButton} onPress={takePicture} />

        <TouchableOpacity
          onPress={() => setFlash(f =>
            f === FlashMode.off ? FlashMode.on : FlashMode.off
          )}
        >
          <Text style={styles.flashText}>
            {flash === FlashMode.off ? 'แฟลชปิด' : 'แฟลชเปิด'}
          </Text>
        </TouchableOpacity>
      </View>
    </CameraView>
  );
};

const styles = StyleSheet.create({
  permissionContainer: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  camera: { flex: 1 },
  cameraControls: {
    position: 'absolute',
    bottom: 40,
    left: 0,
    right: 0,
    flexDirection: 'row',
    justifyContent: 'space-around',
    alignItems: 'center',
    paddingHorizontal: 20,
  },
  captureButton: {
    width: 70,
    height: 70,
    borderRadius: 35,
    backgroundColor: 'white',
    borderWidth: 5,
    borderColor: '#ccc',
  },
  flipText: { color: 'white', fontSize: 16 },
  flashText: { color: 'white', fontSize: 16 },
  previewContainer: { flex: 1 },
  preview: { flex: 1 },
  previewActions: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    padding: 20,
    backgroundColor: 'black',
  },
  retakeText: { color: 'white', fontSize: 16 },
  saveText: { color: '#4CAF50', fontSize: 16 },
  confirmButton: { backgroundColor: '#2196F3', padding: 12, borderRadius: 8 },
  confirmText: { color: 'white', fontWeight: 'bold' },
});

export default ExpoCameraCapture;
```

---

## 5. Photo Compression {#photo-compression}

### การ Compress รูปภาพก่อน Upload

```typescript
// src/utils/imageUtils.ts
import ImageResizer from 'react-native-image-resizer';
import { Platform } from 'react-native';

interface CompressOptions {
  maxWidth?: number;
  maxHeight?: number;
  quality?: number;
  format?: 'JPEG' | 'PNG' | 'WEBP';
  rotation?: number;
  outputPath?: string;
}

interface CompressResult {
  uri: string;
  name: string;
  size: number;
  width: number;
  height: number;
}

export async function compressImage(
  imageUri: string,
  options: CompressOptions = {}
): Promise<CompressResult> {
  const {
    maxWidth = 1080,
    maxHeight = 1080,
    quality = 80,
    format = 'JPEG',
    rotation = 0,
  } = options;

  try {
    const result = await ImageResizer.createResizedImage(
      imageUri,
      maxWidth,
      maxHeight,
      format,
      quality,
      rotation,
      undefined,
      false,
      { mode: 'contain', onlyScaleDown: true }
    );

    console.log(`Compressed: ${result.size} bytes (${result.width}x${result.height})`);

    return {
      uri: result.uri,
      name: result.name,
      size: result.size,
      width: result.width,
      height: result.height,
    };
  } catch (error) {
    console.error('Compression error:', error);
    throw error;
  }
}

// คำนวณ quality ตาม target size
export function calculateQuality(
  originalSize: number,
  targetSizeKB: number
): number {
  const targetBytes = targetSizeKB * 1024;
  const ratio = targetBytes / originalSize;
  
  if (ratio >= 1) return 90; // ไม่ต้อง compress มาก
  if (ratio >= 0.5) return 70;
  if (ratio >= 0.25) return 50;
  return 30;
}

// Resize และ crop ให้เป็น square (สำหรับ profile picture)
export async function makeSquarePhoto(
  imageUri: string,
  size: number = 400
): Promise<CompressResult> {
  return await compressImage(imageUri, {
    maxWidth: size,
    maxHeight: size,
    quality: 85,
    format: 'JPEG',
  });
}

// ตรวจสอบขนาดไฟล์
export function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

---

## 6. Upload to Server {#upload-to-server}

### การ Upload รูปภาพขึ้น Server

```typescript
// src/services/ImageUploadService.ts
import { Platform } from 'react-native';

interface UploadOptions {
  uri: string;
  fileName?: string;
  type?: string;
  fieldName?: string;
  additionalData?: Record<string, string>;
  onProgress?: (progress: number) => void;
}

interface UploadResult {
  url: string;
  publicId?: string;
  thumbnailUrl?: string;
}

class ImageUploadService {
  private static baseUrl = 'https://api.example.com';

  static async uploadImage(options: UploadOptions): Promise<UploadResult> {
    const {
      uri,
      fileName = `image_${Date.now()}.jpg`,
      type = 'image/jpeg',
      fieldName = 'image',
      additionalData = {},
      onProgress,
    } = options;

    return new Promise((resolve, reject) => {
      const formData = new FormData();
      
      formData.append(fieldName, {
        uri: Platform.OS === 'android' ? uri : uri.replace('file://', ''),
        name: fileName,
        type,
      } as any);

      // เพิ่ม data เพิ่มเติม
      Object.entries(additionalData).forEach(([key, value]) => {
        formData.append(key, value);
      });

      const xhr = new XMLHttpRequest();
      
      // Track progress
      xhr.upload.addEventListener('progress', (event) => {
        if (event.lengthComputable) {
          const progress = Math.round((event.loaded / event.total) * 100);
          onProgress?.(progress);
        }
      });

      xhr.addEventListener('load', () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          const response = JSON.parse(xhr.responseText);
          resolve(response);
        } else {
          reject(new Error(`Upload failed: ${xhr.status}`));
        }
      });

      xhr.addEventListener('error', () => {
        reject(new Error('Network error during upload'));
      });

      xhr.addEventListener('timeout', () => {
        reject(new Error('Upload timeout'));
      });

      xhr.open('POST', `${this.baseUrl}/upload`);
      xhr.setRequestHeader('Authorization', `Bearer ${this.getAuthToken()}`);
      xhr.timeout = 60000; // 60 seconds timeout
      xhr.send(formData);
    });
  }

  // Upload หลายรูปพร้อมกัน
  static async uploadMultipleImages(
    images: UploadOptions[],
    onTotalProgress?: (progress: number) => void
  ): Promise<UploadResult[]> {
    let completedCount = 0;
    const totalImages = images.length;

    const results = await Promise.all(
      images.map(async (image, index) => {
        const result = await this.uploadImage({
          ...image,
          onProgress: (imageProgress) => {
            const totalProgress = Math.round(
              ((completedCount + imageProgress / 100) / totalImages) * 100
            );
            onTotalProgress?.(totalProgress);
          },
        });
        completedCount++;
        return result;
      })
    );

    return results;
  }

  // Upload ไป Cloudinary
  static async uploadToCloudinary(
    imageUri: string,
    options: {
      cloudName: string;
      uploadPreset: string;
      folder?: string;
      transformation?: string;
    }
  ): Promise<UploadResult> {
    const formData = new FormData();
    
    formData.append('file', {
      uri: imageUri,
      name: `upload_${Date.now()}.jpg`,
      type: 'image/jpeg',
    } as any);
    formData.append('upload_preset', options.uploadPreset);
    
    if (options.folder) {
      formData.append('folder', options.folder);
    }

    const response = await fetch(
      `https://api.cloudinary.com/v1_1/${options.cloudName}/image/upload`,
      {
        method: 'POST',
        body: formData,
      }
    );

    if (!response.ok) {
      throw new Error('Cloudinary upload failed');
    }

    const data = await response.json();
    
    return {
      url: data.secure_url,
      publicId: data.public_id,
      thumbnailUrl: data.secure_url.replace('/upload/', '/upload/w_200,h_200,c_fill/'),
    };
  }

  private static getAuthToken(): string {
    // ดึง token จาก storage
    return '';
  }
}

export default ImageUploadService;
```

---

## 7. Workshop: Profile Picture Upload {#workshop}

### โปรเจกต์: ระบบ Profile Picture ครบวงจร

```typescript
// src/screens/ProfilePictureScreen.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  Alert,
  Animated,
} from 'react-native';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { useImagePicker } from '../hooks/useImagePicker';
import { compressImage, formatFileSize } from '../utils/imageUtils';
import ImageUploadService from '../services/ImageUploadService';

interface ProfilePictureScreenProps {
  currentAvatarUrl?: string;
  onAvatarUpdated: (url: string) => void;
}

type UploadState = 'idle' | 'selecting' | 'compressing' | 'uploading' | 'success' | 'error';

const ProfilePictureScreen: React.FC<ProfilePictureScreenProps> = ({
  currentAvatarUrl,
  onAvatarUpdated,
}) => {
  const [avatarUri, setAvatarUri] = useState<string | null>(currentAvatarUrl || null);
  const [uploadState, setUploadState] = useState<UploadState>('idle');
  const [uploadProgress, setUploadProgress] = useState(0);
  const [fileInfo, setFileInfo] = useState<{ originalSize: number; compressedSize: number } | null>(null);
  const progressAnim = new Animated.Value(0);

  const { showImagePickerOptions } = useImagePicker();

  const handleSelectImage = useCallback(async () => {
    setUploadState('selecting');

    try {
      const image = await showImagePickerOptions();
      if (!image) {
        setUploadState('idle');
        return;
      }

      // Compress
      setUploadState('compressing');
      const originalSize = image.fileSize;

      const compressed = await compressImage(image.uri, {
        maxWidth: 400,
        maxHeight: 400,
        quality: 85,
        format: 'JPEG',
      });

      setFileInfo({
        originalSize,
        compressedSize: compressed.size,
      });

      setAvatarUri(compressed.uri);

      // Upload
      setUploadState('uploading');
      setUploadProgress(0);

      const result = await ImageUploadService.uploadImage({
        uri: compressed.uri,
        fileName: `avatar_${Date.now()}.jpg`,
        type: 'image/jpeg',
        fieldName: 'avatar',
        onProgress: (progress) => {
          setUploadProgress(progress);
          Animated.timing(progressAnim, {
            toValue: progress / 100,
            duration: 300,
            useNativeDriver: false,
          }).start();
        },
      });

      setUploadState('success');
      onAvatarUpdated(result.url);

      Alert.alert('สำเร็จ', 'อัพเดตรูปโปรไฟล์เรียบร้อยแล้ว');
    } catch (error) {
      console.error('Upload error:', error);
      setUploadState('error');
      Alert.alert('เกิดข้อผิดพลาด', 'ไม่สามารถอัพโหลดรูปได้ กรุณาลองใหม่');
    }
  }, [showImagePickerOptions, onAvatarUpdated]);

  const progressWidth = progressAnim.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '100%'],
  });

  return (
    <View style={styles.container}>
      <Text style={styles.title}>รูปโปรไฟล์</Text>

      {/* Avatar Display */}
      <TouchableOpacity
        style={styles.avatarContainer}
        onPress={uploadState === 'idle' || uploadState === 'success' || uploadState === 'error'
          ? handleSelectImage
          : undefined
        }
        activeOpacity={0.8}
      >
        {avatarUri ? (
          <Image source={{ uri: avatarUri }} style={styles.avatar} />
        ) : (
          <View style={styles.avatarPlaceholder}>
            <Icon name="person" size={60} color="#9E9E9E" />
          </View>
        )}

        {/* Overlay เมื่อ idle */}
        {(uploadState === 'idle' || uploadState === 'success') && (
          <View style={styles.avatarOverlay}>
            <Icon name="camera-alt" size={28} color="white" />
            <Text style={styles.overlayText}>เปลี่ยนรูป</Text>
          </View>
        )}

        {/* Loading State */}
        {(uploadState === 'compressing' || uploadState === 'selecting') && (
          <View style={styles.loadingOverlay}>
            <ActivityIndicator size="large" color="white" />
          </View>
        )}
      </TouchableOpacity>

      {/* Upload Progress */}
      {uploadState === 'uploading' && (
        <View style={styles.progressContainer}>
          <Text style={styles.progressText}>กำลังอัพโหลด... {uploadProgress}%</Text>
          <View style={styles.progressBar}>
            <Animated.View
              style={[styles.progressFill, { width: progressWidth }]}
            />
          </View>
        </View>
      )}

      {/* File Info */}
      {fileInfo && uploadState === 'success' && (
        <View style={styles.fileInfo}>
          <Icon name="check-circle" size={20} color="#4CAF50" />
          <Text style={styles.fileInfoText}>
            ขนาดลดจาก {formatFileSize(fileInfo.originalSize)} เป็น {formatFileSize(fileInfo.compressedSize)}
            {' '}(ลด {Math.round((1 - fileInfo.compressedSize / fileInfo.originalSize) * 100)}%)
          </Text>
        </View>
      )}

      {/* State Messages */}
      <Text style={styles.stateText}>
        {uploadState === 'idle' && 'แตะที่รูปเพื่อเปลี่ยน'}
        {uploadState === 'selecting' && 'กำลังเลือกรูปภาพ...'}
        {uploadState === 'compressing' && 'กำลังบีบอัดรูปภาพ...'}
        {uploadState === 'uploading' && `กำลังอัพโหลด... ${uploadProgress}%`}
        {uploadState === 'success' && '✓ อัพเดตสำเร็จ'}
        {uploadState === 'error' && '✗ เกิดข้อผิดพลาด - กรุณาลองใหม่'}
      </Text>

      {/* Error Retry */}
      {uploadState === 'error' && (
        <TouchableOpacity style={styles.retryButton} onPress={handleSelectImage}>
          <Icon name="refresh" size={20} color="white" />
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      )}

      {/* Image Guidelines */}
      <View style={styles.guidelines}>
        <Text style={styles.guidelinesTitle}>คำแนะนำการใช้รูปโปรไฟล์</Text>
        <Text style={styles.guideline}>• ใช้รูปที่มีความชัดเจน</Text>
        <Text style={styles.guideline}>• ขนาดรูปที่รองรับ: JPG, PNG</Text>
        <Text style={styles.guideline}>• ขนาดไฟล์สูงสุด: 5 MB</Text>
        <Text style={styles.guideline}>• แนะนำความละเอียด 400x400 pixels</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#F5F5F5',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 32,
  },
  avatarContainer: {
    width: 150,
    height: 150,
    borderRadius: 75,
    overflow: 'hidden',
    marginBottom: 16,
    backgroundColor: '#E0E0E0',
    elevation: 4,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
  },
  avatar: {
    width: '100%',
    height: '100%',
  },
  avatarPlaceholder: {
    width: '100%',
    height: '100%',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#E0E0E0',
  },
  avatarOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.4)',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 4,
  },
  overlayText: {
    color: 'white',
    fontSize: 14,
    fontWeight: 'bold',
  },
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  progressContainer: {
    width: '100%',
    marginTop: 16,
    gap: 8,
  },
  progressText: {
    textAlign: 'center',
    color: '#666',
  },
  progressBar: {
    height: 8,
    backgroundColor: '#E0E0E0',
    borderRadius: 4,
    overflow: 'hidden',
  },
  progressFill: {
    height: '100%',
    backgroundColor: '#2196F3',
    borderRadius: 4,
  },
  fileInfo: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    marginTop: 12,
    backgroundColor: '#E8F5E9',
    padding: 8,
    borderRadius: 8,
  },
  fileInfoText: {
    color: '#388E3C',
    fontSize: 13,
    flex: 1,
  },
  stateText: {
    marginTop: 12,
    color: '#666',
    fontSize: 14,
  },
  retryButton: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#F44336',
    padding: 12,
    borderRadius: 8,
    marginTop: 12,
    gap: 8,
  },
  retryText: {
    color: 'white',
    fontWeight: 'bold',
  },
  guidelines: {
    width: '100%',
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 12,
    marginTop: 24,
    gap: 6,
  },
  guidelinesTitle: {
    fontWeight: 'bold',
    marginBottom: 4,
    fontSize: 15,
  },
  guideline: {
    color: '#666',
    fontSize: 13,
  },
});

export default ProfilePictureScreen;
```

---

## Tips และ Best Practices

### 1. จัดการ Memory Leaks

```typescript
// ล้างภาพชั่วคราวหลังใช้งาน
import RNFS from 'react-native-fs';

async function cleanupTempImages(uris: string[]): Promise<void> {
  for (const uri of uris) {
    if (uri.startsWith(RNFS.TemporaryDirectoryPath)) {
      try {
        await RNFS.unlink(uri);
      } catch (error) {
        // ไม่ต้อง throw เพราะไฟล์อาจถูกลบไปแล้ว
      }
    }
  }
}
```

### 2. Image Caching

```typescript
// ใช้ FastImage สำหรับ caching
import FastImage from 'react-native-fast-image';

const CachedAvatar = ({ url }: { url: string }) => (
  <FastImage
    style={styles.avatar}
    source={{
      uri: url,
      priority: FastImage.priority.normal,
      cache: FastImage.cacheControl.immutable,
    }}
    resizeMode={FastImage.resizeMode.cover}
  />
);
```

### 3. Validation ก่อน Upload

```typescript
function validateImage(image: PickedImage): string | null {
  // ตรวจสอบขนาดไฟล์
  const maxSize = 5 * 1024 * 1024; // 5MB
  if (image.fileSize > maxSize) {
    return 'ขนาดไฟล์ใหญ่เกินไป (สูงสุด 5MB)';
  }

  // ตรวจสอบ type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
  if (!allowedTypes.includes(image.type)) {
    return 'รองรับเฉพาะ JPG, PNG, WEBP เท่านั้น';
  }

  // ตรวจสอบขนาดรูป
  if (image.width < 100 || image.height < 100) {
    return 'รูปภาพเล็กเกินไป (ขั้นต่ำ 100x100 pixels)';
  }

  return null; // Valid
}
```

### สรุป

- ใช้ react-native-image-picker สำหรับเลือกรูปจาก gallery
- ใช้ react-native-vision-camera สำหรับ custom camera
- Compress รูปก่อน upload เสมอเพื่อประหยัด bandwidth
- แสดง progress ให้ user เห็นระหว่าง upload
- Validate ขนาดและประเภทไฟล์ก่อน upload
- ใช้ FastImage สำหรับ caching รูปที่ดาวน์โหลด
