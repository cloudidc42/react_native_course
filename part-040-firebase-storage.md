# Part 040: Firebase Storage

## Firebase Storage คืออะไร?

Firebase Storage เป็นบริการเก็บไฟล์บน cloud (object storage) ที่:
- เก็บรูปภาพ, วิดีโอ, เสียง, เอกสาร
- Scale ได้อัตโนมัติ
- Secure ด้วย Security Rules
- รองรับ upload/download พร้อม progress tracking
- CDN ทั่วโลก

---

## Setup

```bash
npm install @react-native-firebase/storage

# สำหรับรับ images จาก gallery
npx expo install expo-image-picker expo-document-picker
```

---

## Upload Files/Images

### Upload รูปภาพจาก Gallery

```typescript
// src/services/firebase/storageService.ts
import storage from '@react-native-firebase/storage';
import auth from '@react-native-firebase/auth';

export interface UploadResult {
  url: string;
  path: string;
  size: number;
  contentType: string;
}

export const storageService = {
  // Upload รูปภาพ
  uploadImage: async (
    localUri: string,
    folder: string = 'images',
    onProgress?: (progress: number) => void
  ): Promise<UploadResult> => {
    const user = auth().currentUser;
    if (!user) throw new Error('Not authenticated');

    // สร้าง filename ไม่ซ้ำกัน
    const extension = localUri.split('.').pop() || 'jpg';
    const filename = `${folder}/${user.uid}/${Date.now()}.${extension}`;

    const reference = storage().ref(filename);

    // Upload task
    const task = reference.putFile(localUri);

    // Track progress
    if (onProgress) {
      task.on('state_changed', (snapshot) => {
        const progress =
          (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        onProgress(Math.round(progress));
      });
    }

    // รอ upload เสร็จ
    await task;

    // ดึง download URL
    const url = await reference.getDownloadURL();

    // ดึง metadata
    const metadata = await reference.getMetadata();

    return {
      url,
      path: filename,
      size: metadata.size,
      contentType: metadata.contentType || 'image/jpeg',
    };
  },

  // Upload Avatar โปรไฟล์
  uploadAvatar: async (
    localUri: string,
    onProgress?: (progress: number) => void
  ): Promise<string> => {
    const user = auth().currentUser;
    if (!user) throw new Error('Not authenticated');

    const filename = `avatars/${user.uid}/profile.jpg`;
    const reference = storage().ref(filename);

    const task = reference.putFile(localUri, {
      contentType: 'image/jpeg',
      customMetadata: {
        userId: user.uid,
        uploadedAt: new Date().toISOString(),
      },
    });

    if (onProgress) {
      task.on('state_changed', (snapshot) => {
        const progress =
          (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        onProgress(Math.round(progress));
      });
    }

    await task;
    return reference.getDownloadURL();
  },

  // Upload เอกสาร
  uploadDocument: async (
    localUri: string,
    mimeType: string,
    fileName: string,
    onProgress?: (progress: number) => void
  ): Promise<UploadResult> => {
    const user = auth().currentUser;
    if (!user) throw new Error('Not authenticated');

    const path = `documents/${user.uid}/${Date.now()}_${fileName}`;
    const reference = storage().ref(path);

    const task = reference.putFile(localUri, { contentType: mimeType });

    if (onProgress) {
      task.on('state_changed', (snapshot) => {
        const progress =
          (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        onProgress(Math.round(progress));
      });
    }

    await task;

    const url = await reference.getDownloadURL();
    const metadata = await reference.getMetadata();

    return {
      url,
      path,
      size: metadata.size,
      contentType: mimeType,
    };
  },

  // Upload หลายรูปพร้อมกัน
  uploadMultiple: async (
    localUris: string[],
    folder: string,
    onTotalProgress?: (progress: number) => void
  ): Promise<UploadResult[]> => {
    let completed = 0;
    const total = localUris.length;

    const results = await Promise.all(
      localUris.map((uri) =>
        storageService.uploadImage(uri, folder, () => {
          completed++;
          onTotalProgress?.((completed / total) * 100);
        })
      )
    );

    return results;
  },
};
```

---

## Download Files

```typescript
// src/services/firebase/downloadService.ts
import storage from '@react-native-firebase/storage';
import RNFS from 'react-native-fs';

export const downloadService = {
  // ดึง Download URL
  getDownloadUrl: async (storagePath: string): Promise<string> => {
    return storage().ref(storagePath).getDownloadURL();
  },

  // Download ไฟล์ลง device
  downloadToDevice: async (
    storagePath: string,
    localPath: string,
    onProgress?: (progress: number) => void
  ): Promise<string> => {
    const reference = storage().ref(storagePath);

    // Get download URL
    const url = await reference.getDownloadURL();

    // Download ไปยัง local file
    const result = await RNFS.downloadFile({
      fromUrl: url,
      toFile: localPath,
      progress: (res) => {
        if (onProgress) {
          const progress = (res.bytesWritten / res.contentLength) * 100;
          onProgress(Math.round(progress));
        }
      },
    }).promise;

    if (result.statusCode !== 200) {
      throw new Error('Download failed');
    }

    return localPath;
  },

  // ดึง metadata ของไฟล์
  getMetadata: async (storagePath: string) => {
    return storage().ref(storagePath).getMetadata();
  },
};
```

---

## Progress Tracking

### Custom Upload Hook

```typescript
// src/hooks/useUpload.ts
import { useState, useCallback, useRef } from 'react';
import storage, { FirebaseStorageTypes } from '@react-native-firebase/storage';
import auth from '@react-native-firebase/auth';

export type UploadStatus = 'idle' | 'uploading' | 'paused' | 'success' | 'error';

export interface UploadState {
  status: UploadStatus;
  progress: number;
  bytesTransferred: number;
  totalBytes: number;
  downloadUrl: string | null;
  error: string | null;
}

export function useUpload() {
  const [state, setState] = useState<UploadState>({
    status: 'idle',
    progress: 0,
    bytesTransferred: 0,
    totalBytes: 0,
    downloadUrl: null,
    error: null,
  });

  const taskRef = useRef<FirebaseStorageTypes.Task | null>(null);

  const upload = useCallback(
    async (localUri: string, storagePath: string) => {
      setState({
        status: 'uploading',
        progress: 0,
        bytesTransferred: 0,
        totalBytes: 0,
        downloadUrl: null,
        error: null,
      });

      const reference = storage().ref(storagePath);
      const task = reference.putFile(localUri);
      taskRef.current = task;

      // Track state changes
      task.on(
        storage.TaskEvent.STATE_CHANGED,
        (snapshot) => {
          const progress =
            (snapshot.bytesTransferred / snapshot.totalBytes) * 100;

          setState((prev) => ({
            ...prev,
            status:
              snapshot.state === storage.TaskState.PAUSED
                ? 'paused'
                : 'uploading',
            progress: Math.round(progress),
            bytesTransferred: snapshot.bytesTransferred,
            totalBytes: snapshot.totalBytes,
          }));
        },
        (error) => {
          // Upload error
          if (error.code !== 'storage/canceled') {
            setState((prev) => ({
              ...prev,
              status: 'error',
              error: error.message,
            }));
          }
        },
        async () => {
          // Upload complete
          const url = await reference.getDownloadURL();
          setState((prev) => ({
            ...prev,
            status: 'success',
            progress: 100,
            downloadUrl: url,
          }));
        }
      );

      try {
        await task;
      } catch (error: any) {
        if (error.code !== 'storage/canceled') {
          throw error;
        }
      }
    },
    []
  );

  const pause = useCallback(() => {
    taskRef.current?.pause();
  }, []);

  const resume = useCallback(() => {
    taskRef.current?.resume();
  }, []);

  const cancel = useCallback(() => {
    taskRef.current?.cancel();
    setState({
      status: 'idle',
      progress: 0,
      bytesTransferred: 0,
      totalBytes: 0,
      downloadUrl: null,
      error: null,
    });
  }, []);

  const reset = useCallback(() => {
    setState({
      status: 'idle',
      progress: 0,
      bytesTransferred: 0,
      totalBytes: 0,
      downloadUrl: null,
      error: null,
    });
  }, []);

  return { state, upload, pause, resume, cancel, reset };
}
```

---

## Delete Files

```typescript
// src/services/firebase/deleteService.ts
import storage from '@react-native-firebase/storage';

export const deleteService = {
  // ลบไฟล์จาก path
  deleteFile: async (storagePath: string): Promise<void> => {
    await storage().ref(storagePath).delete();
  },

  // ลบหลายไฟล์พร้อมกัน
  deleteMultiple: async (paths: string[]): Promise<void> => {
    await Promise.all(paths.map((path) => storage().ref(path).delete()));
  },

  // ลบ folder (ต้อง list files ก่อน)
  deleteFolder: async (folderPath: string): Promise<void> => {
    const result = await storage().ref(folderPath).listAll();

    await Promise.all([
      ...result.items.map((item) => item.delete()),
      ...result.prefixes.map((prefix) => deleteService.deleteFolder(prefix.fullPath)),
    ]);
  },
};
```

---

## Storage Security Rules

```javascript
// storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    // ฟังก์ชัน helper
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return request.auth != null && request.auth.uid == userId;
    }

    function isImageFile() {
      return request.resource.contentType.matches('image/.*');
    }

    function isValidSize(maxMB) {
      return request.resource.size < maxMB * 1024 * 1024;
    }

    // Avatars - แต่ละ user เข้าถึงของตัวเองเท่านั้น
    match /avatars/{userId}/{allPaths=**} {
      allow read: if true;  // ทุกคนดูได้
      allow write: if isOwner(userId)
        && isImageFile()
        && isValidSize(5);  // max 5MB
      allow delete: if isOwner(userId);
    }

    // Images - ดูได้ทุกคน แก้ไขได้เจ้าของ
    match /images/{userId}/{allPaths=**} {
      allow read: if true;
      allow write: if isOwner(userId)
        && isImageFile()
        && isValidSize(10);  // max 10MB
      allow delete: if isOwner(userId);
    }

    // Documents - เฉพาะเจ้าของ
    match /documents/{userId}/{allPaths=**} {
      allow read: if isOwner(userId);
      allow write: if isOwner(userId)
        && isValidSize(50);  // max 50MB
      allow delete: if isOwner(userId);
    }

    // Public files
    match /public/{allPaths=**} {
      allow read: if true;
      allow write: if isAuthenticated()
        && isValidSize(20);
    }
  }
}
```

---

## Workshop: Photo Gallery App

### Gallery Types

```typescript
// src/types/gallery.ts
export interface Photo {
  id: string;
  url: string;
  thumbnailUrl?: string;
  storagePath: string;
  caption?: string;
  width: number;
  height: number;
  size: number;
  uploadedAt: Date;
  userId: string;
  albumId?: string;
  tags: string[];
  likes: number;
  isPublic: boolean;
}

export interface Album {
  id: string;
  name: string;
  coverPhotoUrl?: string;
  photoCount: number;
  userId: string;
  createdAt: Date;
}
```

### Gallery Service

```typescript
// src/services/galleryService.ts
import storage from '@react-native-firebase/storage';
import firestore from '@react-native-firebase/firestore';
import auth from '@react-native-firebase/auth';
import { Photo, Album } from '../types/gallery';
import { manipulateAsync, SaveFormat } from 'expo-image-manipulator';

export const galleryService = {
  // Upload รูปพร้อมสร้าง thumbnail
  uploadPhoto: async (
    localUri: string,
    options: {
      caption?: string;
      albumId?: string;
      tags?: string[];
      isPublic?: boolean;
      onProgress?: (progress: number) => void;
    } = {}
  ): Promise<Photo> => {
    const user = auth().currentUser!;
    const { caption = '', albumId, tags = [], isPublic = false, onProgress } = options;

    // สร้าง thumbnail
    const thumbnail = await manipulateAsync(
      localUri,
      [{ resize: { width: 300 } }],
      { format: SaveFormat.JPEG, compress: 0.7 }
    );

    // Upload รูปต้นฉบับ
    const photoId = Date.now().toString();
    const extension = 'jpg';
    const photoPath = `images/${user.uid}/${photoId}.${extension}`;
    const thumbPath = `images/${user.uid}/${photoId}_thumb.${extension}`;

    // Upload ทั้ง 2 ไฟล์พร้อมกัน
    const [photoUpload, thumbUpload] = await Promise.all([
      (async () => {
        const ref = storage().ref(photoPath);
        const task = ref.putFile(localUri);
        if (onProgress) {
          task.on('state_changed', (snapshot) => {
            const p = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
            onProgress(Math.round(p * 0.8)); // 80% for main photo
          });
        }
        await task;
        return ref.getDownloadURL();
      })(),
      (async () => {
        const ref = storage().ref(thumbPath);
        await ref.putFile(thumbnail.uri);
        return ref.getDownloadURL();
      })(),
    ]);

    onProgress?.(90);

    // บันทึก metadata ลง Firestore
    const photoData: Omit<Photo, 'id'> = {
      url: photoUpload,
      thumbnailUrl: thumbUpload,
      storagePath: photoPath,
      caption,
      width: 0, // จะ update ทีหลัง
      height: 0,
      size: 0,
      uploadedAt: new Date(),
      userId: user.uid,
      albumId,
      tags,
      likes: 0,
      isPublic,
    };

    const docRef = await firestore()
      .collection('photos')
      .add(photoData);

    // อัปเดต album photo count
    if (albumId) {
      await firestore()
        .collection('albums')
        .doc(albumId)
        .update({
          photoCount: firestore.FieldValue.increment(1),
          coverPhotoUrl: photoUpload,
        });
    }

    onProgress?.(100);

    return { id: docRef.id, ...photoData };
  },

  // ดึงรูปทั้งหมดของ user
  getUserPhotos: async (userId: string): Promise<Photo[]> => {
    const snapshot = await firestore()
      .collection('photos')
      .where('userId', '==', userId)
      .orderBy('uploadedAt', 'desc')
      .get();

    return snapshot.docs.map((doc) => ({
      id: doc.id,
      ...doc.data(),
      uploadedAt: doc.data().uploadedAt?.toDate(),
    })) as Photo[];
  },

  // ลบรูป
  deletePhoto: async (photo: Photo): Promise<void> => {
    // ลบจาก Storage
    const deletions = [storage().ref(photo.storagePath).delete()];

    if (photo.thumbnailUrl) {
      const thumbPath = photo.storagePath.replace('.jpg', '_thumb.jpg');
      deletions.push(storage().ref(thumbPath).delete());
    }

    await Promise.all(deletions);

    // ลบจาก Firestore
    await firestore().collection('photos').doc(photo.id).delete();

    // อัปเดต album
    if (photo.albumId) {
      await firestore()
        .collection('albums')
        .doc(photo.albumId)
        .update({
          photoCount: firestore.FieldValue.increment(-1),
        });
    }
  },
};
```

### Photo Gallery Screen

```typescript
// src/screens/PhotoGalleryScreen.tsx
import React, { useState, useEffect } from 'react';
import {
  View,
  FlatList,
  Image,
  TouchableOpacity,
  Text,
  StyleSheet,
  Dimensions,
  Alert,
  ActivityIndicator,
  Modal,
} from 'react-native';
import * as ImagePicker from 'expo-image-picker';
import auth from '@react-native-firebase/auth';
import { galleryService } from '../services/galleryService';
import { useUpload } from '../hooks/useUpload';
import { Photo } from '../types/gallery';

const { width } = Dimensions.get('window');
const PHOTO_SIZE = (width - 4) / 3;

export const PhotoGalleryScreen: React.FC = () => {
  const [photos, setPhotos] = useState<Photo[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [selectedPhoto, setSelectedPhoto] = useState<Photo | null>(null);
  const { state: uploadState, upload, cancel, reset } = useUpload();

  const user = auth().currentUser!;

  useEffect(() => {
    loadPhotos();
  }, []);

  const loadPhotos = async () => {
    setIsLoading(true);
    try {
      const userPhotos = await galleryService.getUserPhotos(user.uid);
      setPhotos(userPhotos);
    } catch (error) {
      Alert.alert('ผิดพลาด', 'ไม่สามารถโหลดรูปภาพได้');
    } finally {
      setIsLoading(false);
    }
  };

  const handlePickImage = async () => {
    const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (status !== 'granted') {
      Alert.alert('ต้องการสิทธิ์', 'กรุณาอนุญาตการเข้าถึง photo library');
      return;
    }

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsMultipleSelection: true,
      quality: 0.8,
    });

    if (!result.canceled) {
      for (const asset of result.assets) {
        await handleUploadPhoto(asset.uri);
      }
    }
  };

  const handleTakePhoto = async () => {
    const { status } = await ImagePicker.requestCameraPermissionsAsync();
    if (status !== 'granted') {
      Alert.alert('ต้องการสิทธิ์', 'กรุณาอนุญาตการเข้าถึงกล้อง');
      return;
    }

    const result = await ImagePicker.launchCameraAsync({
      quality: 0.8,
    });

    if (!result.canceled) {
      await handleUploadPhoto(result.assets[0].uri);
    }
  };

  const handleUploadPhoto = async (uri: string) => {
    try {
      const photo = await galleryService.uploadPhoto(uri, {
        onProgress: (progress) => {
          // progress จะถูก track ผ่าน uploadState
        },
      });
      setPhotos((prev) => [photo, ...prev]);
    } catch (error: any) {
      Alert.alert('ผิดพลาด', error.message);
    }
  };

  const handleDeletePhoto = (photo: Photo) => {
    Alert.alert('ลบรูปภาพ', 'ต้องการลบรูปนี้?', [
      { text: 'ยกเลิก', style: 'cancel' },
      {
        text: 'ลบ',
        style: 'destructive',
        onPress: async () => {
          try {
            await galleryService.deletePhoto(photo);
            setPhotos((prev) => prev.filter((p) => p.id !== photo.id));
            setSelectedPhoto(null);
          } catch (error: any) {
            Alert.alert('ผิดพลาด', error.message);
          }
        },
      },
    ]);
  };

  const renderPhoto = ({ item }: { item: Photo }) => (
    <TouchableOpacity
      onPress={() => setSelectedPhoto(item)}
      style={styles.photoContainer}
    >
      <Image
        source={{ uri: item.thumbnailUrl || item.url }}
        style={styles.photo}
        resizeMode="cover"
      />
    </TouchableOpacity>
  );

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>
          📷 คลังรูปภาพ ({photos.length})
        </Text>
        <View style={styles.headerActions}>
          <TouchableOpacity
            onPress={handleTakePhoto}
            style={styles.headerButton}
          >
            <Text style={styles.headerButtonText}>📸</Text>
          </TouchableOpacity>
          <TouchableOpacity
            onPress={handlePickImage}
            style={styles.headerButton}
          >
            <Text style={styles.headerButtonText}>🖼️</Text>
          </TouchableOpacity>
        </View>
      </View>

      {/* Upload Progress */}
      {uploadState.status === 'uploading' && (
        <View style={styles.progressBar}>
          <View
            style={[
              styles.progressFill,
              { width: `${uploadState.progress}%` },
            ]}
          />
          <Text style={styles.progressText}>
            กำลังอัปโหลด {uploadState.progress}%
          </Text>
          <TouchableOpacity onPress={cancel} style={styles.cancelButton}>
            <Text style={styles.cancelText}>ยกเลิก</Text>
          </TouchableOpacity>
        </View>
      )}

      {/* Photos Grid */}
      {photos.length === 0 ? (
        <View style={styles.empty}>
          <Text style={styles.emptyIcon}>📷</Text>
          <Text style={styles.emptyTitle}>ยังไม่มีรูปภาพ</Text>
          <Text style={styles.emptySubtitle}>
            กดปุ่มด้านบนเพื่อเพิ่มรูปภาพ
          </Text>
        </View>
      ) : (
        <FlatList
          data={photos}
          keyExtractor={(item) => item.id}
          numColumns={3}
          renderItem={renderPhoto}
          contentContainerStyle={styles.grid}
          onRefresh={loadPhotos}
          refreshing={isLoading}
        />
      )}

      {/* Photo Detail Modal */}
      <Modal
        visible={!!selectedPhoto}
        transparent
        animationType="fade"
        onRequestClose={() => setSelectedPhoto(null)}
      >
        <View style={styles.modalOverlay}>
          <TouchableOpacity
            style={styles.modalClose}
            onPress={() => setSelectedPhoto(null)}
          >
            <Text style={styles.modalCloseText}>✕</Text>
          </TouchableOpacity>

          {selectedPhoto && (
            <View style={styles.modalContent}>
              <Image
                source={{ uri: selectedPhoto.url }}
                style={styles.modalImage}
                resizeMode="contain"
              />

              {selectedPhoto.caption ? (
                <Text style={styles.caption}>{selectedPhoto.caption}</Text>
              ) : null}

              <View style={styles.modalActions}>
                <TouchableOpacity
                  style={styles.actionButton}
                  onPress={() => handleDeletePhoto(selectedPhoto)}
                >
                  <Text style={styles.actionButtonText}>🗑️ ลบ</Text>
                </TouchableOpacity>
                <TouchableOpacity style={styles.actionButton}>
                  <Text style={styles.actionButtonText}>⬇️ ดาวน์โหลด</Text>
                </TouchableOpacity>
                <TouchableOpacity style={styles.actionButton}>
                  <Text style={styles.actionButtonText}>📤 แชร์</Text>
                </TouchableOpacity>
              </View>
            </View>
          )}
        </View>
      </Modal>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#111',
    paddingHorizontal: 16,
    paddingVertical: 12,
    paddingTop: 48,
  },
  headerTitle: { fontSize: 16, fontWeight: 'bold', color: '#fff' },
  headerActions: { flexDirection: 'row', gap: 8 },
  headerButton: {
    width: 40,
    height: 40,
    borderRadius: 20,
    backgroundColor: 'rgba(255,255,255,0.1)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  headerButtonText: { fontSize: 20 },
  progressBar: {
    backgroundColor: '#1a1a1a',
    padding: 12,
    flexDirection: 'row',
    alignItems: 'center',
  },
  progressFill: {
    position: 'absolute',
    left: 0,
    top: 0,
    bottom: 0,
    backgroundColor: '#6200EE',
    opacity: 0.3,
  },
  progressText: { flex: 1, color: '#fff', fontSize: 13 },
  cancelButton: { padding: 8 },
  cancelText: { color: '#FF5252', fontSize: 13 },
  grid: { gap: 2 },
  photoContainer: { margin: 1 },
  photo: { width: PHOTO_SIZE, height: PHOTO_SIZE },
  empty: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
  },
  emptyIcon: { fontSize: 64, marginBottom: 16 },
  emptyTitle: { fontSize: 18, color: '#fff', fontWeight: 'bold', marginBottom: 8 },
  emptySubtitle: { fontSize: 14, color: '#888' },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.95)',
    justifyContent: 'center',
  },
  modalClose: {
    position: 'absolute',
    top: 48,
    right: 16,
    zIndex: 10,
    width: 40,
    height: 40,
    borderRadius: 20,
    backgroundColor: 'rgba(255,255,255,0.2)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  modalCloseText: { color: '#fff', fontSize: 18 },
  modalContent: { flex: 1, justifyContent: 'center' },
  modalImage: { width: '100%', height: width },
  caption: {
    color: '#fff',
    fontSize: 14,
    textAlign: 'center',
    padding: 16,
  },
  modalActions: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    padding: 16,
    borderTopWidth: 1,
    borderTopColor: '#333',
  },
  actionButton: {
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 8,
    backgroundColor: 'rgba(255,255,255,0.1)',
  },
  actionButtonText: { color: '#fff', fontSize: 13 },
});
```

---

## Tips and Best Practices

### 1. Compress ก่อน Upload

```typescript
import { manipulateAsync, SaveFormat } from 'expo-image-manipulator';

const compressImage = async (uri: string, maxWidth: number = 1080) => {
  const result = await manipulateAsync(
    uri,
    [{ resize: { width: maxWidth } }],
    { format: SaveFormat.JPEG, compress: 0.8 }
  );
  return result.uri;
};
```

### 2. Cache URLs

```typescript
// เก็บ download URLs ใน AsyncStorage
import AsyncStorage from '@react-native-async-storage/async-storage';

const cacheService = {
  getUrl: async (path: string) => {
    const cached = await AsyncStorage.getItem(`storage_url_${path}`);
    return cached;
  },

  setUrl: async (path: string, url: string) => {
    await AsyncStorage.setItem(`storage_url_${path}`, url);
  },
};
```

### 3. Handle Network Errors

```typescript
const uploadWithRetry = async (
  uri: string,
  path: string,
  maxRetries: number = 3
) => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const ref = storage().ref(path);
      await ref.putFile(uri);
      return ref.getDownloadURL();
    } catch (error: any) {
      if (attempt === maxRetries) throw error;
      if (error.code === 'storage/retry-limit-exceeded') {
        await new Promise((r) => setTimeout(r, 2000 * attempt));
      } else {
        throw error;
      }
    }
  }
};
```

### 4. List Files

```typescript
const listUserPhotos = async (userId: string) => {
  const result = await storage().ref(`images/${userId}`).list({
    maxResults: 100,
  });

  const urls = await Promise.all(
    result.items.map((ref) => ref.getDownloadURL())
  );

  return urls;
};
```

---

## สรุป

Firebase Storage ครอบคลุม:

1. **Upload** - รูปภาพ, เอกสาร, วิดีโอ ด้วย progress tracking
2. **Download** - ดึง URL หรือ download ลง device
3. **Delete** - ลบไฟล์และ folder
4. **Security Rules** - ควบคุมสิทธิ์การเข้าถึง
5. **Workshop Photo Gallery** - แอปแกลเลอรี่สมบูรณ์

---

## สรุปบทที่ 031-040

ในส่วนนี้เราได้เรียนรู้เรื่อง State Management และ Backend Services:

| Part | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|--------------|
| 031 | Redux Toolkit | createSlice, configureStore |
| 032 | Redux Async | createAsyncThunk, RTK Query |
| 033 | Zustand | create, immer, persist |
| 034 | React Query | useQuery, useMutation |
| 035 | Axios | Interceptors, Error handling |
| 036 | JWT Auth | SecureStore, Token refresh |
| 037 | OAuth | Google, Facebook, Apple Sign-In |
| 038 | Firebase Auth | Email, Phone, Social login |
| 039 | Firestore | CRUD, Real-time, Queries |
| 040 | Firebase Storage | Upload, Download, Progress |

ในส่วนถัดไป (Part 041-050) จะเรียนเรื่อง Performance Optimization, Testing, และ App Deployment
