# Part 049: File System และ Documents ใน React Native

## สารบัญ
1. [แนะนำ File System](#introduction)
2. [react-native-fs](#react-native-fs)
3. [Read/Write Files](#read-write)
4. [Directory Operations](#directory-ops)
5. [Document Picker](#document-picker)
6. [File Sharing](#file-sharing)
7. [Workshop: Note Taking App](#workshop)

---

## 1. แนะนำ File System {#introduction}

การทำงานกับ File System ใน React Native ช่วยให้แอปสามารถอ่าน เขียน และจัดการไฟล์บนอุปกรณ์ได้

### ตำแหน่งไฟล์สำคัญ

```typescript
import RNFS from 'react-native-fs';

// iOS Paths
RNFS.DocumentDirectoryPath     // ~/Documents (sync กับ iCloud ได้)
RNFS.CachesDirectoryPath       // ~/Library/Caches (ถูกลบได้โดย OS)
RNFS.TemporaryDirectoryPath    // ~/tmp
RNFS.MainBundlePath            // App bundle
RNFS.LibraryDirectoryPath      // ~/Library

// Android Paths
RNFS.DocumentDirectoryPath     // /data/data/[package]/files
RNFS.CachesDirectoryPath       // /data/data/[package]/cache
RNFS.ExternalDirectoryPath     // /sdcard/Android/data/[package]/files
RNFS.ExternalCachesDirectoryPath // /sdcard/Android/data/[package]/cache
RNFS.DownloadDirectoryPath     // /sdcard/Download
```

### ติดตั้ง

```bash
npm install react-native-fs
cd ios && pod install

# Document Picker
npm install react-native-document-picker

# File sharing
npm install react-native-share

# Expo
npx expo install expo-file-system expo-document-picker expo-sharing
```

---

## 2. react-native-fs {#react-native-fs}

### File Operations

```typescript
// src/services/FileSystemService.ts
import RNFS from 'react-native-fs';
import { Platform } from 'react-native';

class FileSystemService {
  // อ่านไฟล์ text
  static async readTextFile(filePath: string): Promise<string | null> {
    try {
      const exists = await RNFS.exists(filePath);
      if (!exists) {
        console.log('File not found:', filePath);
        return null;
      }

      const content = await RNFS.readFile(filePath, 'utf8');
      return content;
    } catch (error) {
      console.error('Read file error:', error);
      return null;
    }
  }

  // เขียนไฟล์ text
  static async writeTextFile(
    filePath: string,
    content: string,
    append = false
  ): Promise<boolean> {
    try {
      if (append) {
        await RNFS.appendFile(filePath, content, 'utf8');
      } else {
        await RNFS.writeFile(filePath, content, 'utf8');
      }
      return true;
    } catch (error) {
      console.error('Write file error:', error);
      return false;
    }
  }

  // อ่านไฟล์เป็น base64
  static async readFileAsBase64(filePath: string): Promise<string | null> {
    try {
      return await RNFS.readFile(filePath, 'base64');
    } catch (error) {
      console.error('Read base64 error:', error);
      return null;
    }
  }

  // เขียนไฟล์จาก base64
  static async writeBase64File(filePath: string, base64: string): Promise<boolean> {
    try {
      await RNFS.writeFile(filePath, base64, 'base64');
      return true;
    } catch (error) {
      console.error('Write base64 error:', error);
      return false;
    }
  }

  // อ่านไฟล์ JSON
  static async readJsonFile<T>(filePath: string): Promise<T | null> {
    try {
      const content = await this.readTextFile(filePath);
      if (!content) return null;
      return JSON.parse(content) as T;
    } catch (error) {
      console.error('Read JSON error:', error);
      return null;
    }
  }

  // เขียนไฟล์ JSON
  static async writeJsonFile(filePath: string, data: any): Promise<boolean> {
    try {
      const content = JSON.stringify(data, null, 2);
      return await this.writeTextFile(filePath, content);
    } catch (error) {
      console.error('Write JSON error:', error);
      return false;
    }
  }

  // ลบไฟล์
  static async deleteFile(filePath: string): Promise<boolean> {
    try {
      const exists = await RNFS.exists(filePath);
      if (!exists) return true; // ไม่มีก็ถือว่าสำเร็จ

      await RNFS.unlink(filePath);
      return true;
    } catch (error) {
      console.error('Delete file error:', error);
      return false;
    }
  }

  // คัดลอกไฟล์
  static async copyFile(source: string, dest: string): Promise<boolean> {
    try {
      // สร้าง directory ถ้าไม่มี
      const destDir = dest.substring(0, dest.lastIndexOf('/'));
      await RNFS.mkdir(destDir);

      await RNFS.copyFile(source, dest);
      return true;
    } catch (error) {
      console.error('Copy file error:', error);
      return false;
    }
  }

  // ย้ายไฟล์
  static async moveFile(source: string, dest: string): Promise<boolean> {
    try {
      await RNFS.moveFile(source, dest);
      return true;
    } catch (error) {
      console.error('Move file error:', error);
      return false;
    }
  }

  // ดาวน์โหลดไฟล์
  static async downloadFile(
    url: string,
    destPath: string,
    onProgress?: (progress: number) => void
  ): Promise<{ success: boolean; statusCode?: number }> {
    try {
      const result = await RNFS.downloadFile({
        fromUrl: url,
        toFile: destPath,
        progress: (res) => {
          const progress = (res.bytesWritten / res.contentLength) * 100;
          onProgress?.(Math.round(progress));
        },
        progressDivider: 1,
        background: true, // Android background download
        discretionary: true, // iOS
      }).promise;

      return {
        success: result.statusCode >= 200 && result.statusCode < 300,
        statusCode: result.statusCode,
      };
    } catch (error) {
      console.error('Download error:', error);
      return { success: false };
    }
  }

  // ดึงข้อมูลไฟล์
  static async getFileInfo(filePath: string): Promise<RNFS.StatResult | null> {
    try {
      const exists = await RNFS.exists(filePath);
      if (!exists) return null;
      return await RNFS.stat(filePath);
    } catch (error) {
      console.error('File stat error:', error);
      return null;
    }
  }

  // ขนาดไฟล์ที่อ่านได้
  static formatFileSize(bytes: number): string {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    if (bytes < 1024 * 1024 * 1024) return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
    return `${(bytes / (1024 * 1024 * 1024)).toFixed(1)} GB`;
  }
}

export default FileSystemService;
```

---

## 3. Read/Write Files {#read-write}

### File Manager Hook

```typescript
// src/hooks/useFileManager.ts
import { useState, useCallback } from 'react';
import RNFS from 'react-native-fs';
import FileSystemService from '../services/FileSystemService';

interface FileOperationState {
  loading: boolean;
  error: string | null;
  progress: number;
}

export function useFileManager() {
  const [state, setState] = useState<FileOperationState>({
    loading: false,
    error: null,
    progress: 0,
  });

  const getDocumentPath = (filename: string): string => {
    return `${RNFS.DocumentDirectoryPath}/${filename}`;
  };

  const getCachePath = (filename: string): string => {
    return `${RNFS.CachesDirectoryPath}/${filename}`;
  };

  // บันทึก Note
  const saveNote = useCallback(async (
    id: string,
    title: string,
    content: string
  ): Promise<boolean> => {
    setState(prev => ({ ...prev, loading: true, error: null }));

    try {
      const note = {
        id,
        title,
        content,
        updatedAt: new Date().toISOString(),
      };

      const path = getDocumentPath(`notes/${id}.json`);
      const success = await FileSystemService.writeJsonFile(path, note);

      setState(prev => ({ ...prev, loading: false }));
      return success;
    } catch (error) {
      setState(prev => ({ ...prev, loading: false, error: String(error) }));
      return false;
    }
  }, []);

  // อ่าน Note
  const loadNote = useCallback(async (id: string) => {
    setState(prev => ({ ...prev, loading: true }));

    try {
      const path = getDocumentPath(`notes/${id}.json`);
      const note = await FileSystemService.readJsonFile(path);
      setState(prev => ({ ...prev, loading: false }));
      return note;
    } catch (error) {
      setState(prev => ({ ...prev, loading: false, error: String(error) }));
      return null;
    }
  }, []);

  // ดาวน์โหลดไฟล์
  const downloadFile = useCallback(async (
    url: string,
    filename: string
  ): Promise<string | null> => {
    const destPath = getCachePath(filename);
    setState(prev => ({ ...prev, loading: true, progress: 0 }));

    const result = await FileSystemService.downloadFile(
      url,
      destPath,
      (progress) => setState(prev => ({ ...prev, progress }))
    );

    setState(prev => ({ ...prev, loading: false, progress: 100 }));

    return result.success ? destPath : null;
  }, []);

  return {
    ...state,
    saveNote,
    loadNote,
    downloadFile,
    getDocumentPath,
    getCachePath,
  };
}
```

---

## 4. Directory Operations {#directory-ops}

```typescript
// src/services/DirectoryService.ts
import RNFS from 'react-native-fs';

interface FileItem {
  name: string;
  path: string;
  isDirectory: boolean;
  size: number;
  modified: Date;
  type?: string;
}

class DirectoryService {
  // สร้าง Directory
  static async createDirectory(dirPath: string): Promise<boolean> {
    try {
      await RNFS.mkdir(dirPath);
      return true;
    } catch (error) {
      console.error('Create directory error:', error);
      return false;
    }
  }

  // ลิสต์ไฟล์ใน Directory
  static async listDirectory(dirPath: string): Promise<FileItem[]> {
    try {
      const exists = await RNFS.exists(dirPath);
      if (!exists) return [];

      const items = await RNFS.readDir(dirPath);
      return items.map(item => ({
        name: item.name,
        path: item.path,
        isDirectory: item.isDirectory(),
        size: item.size,
        modified: new Date(item.mtime || Date.now()),
        type: item.isFile() ? this.getFileType(item.name) : 'directory',
      }));
    } catch (error) {
      console.error('List directory error:', error);
      return [];
    }
  }

  // ลบ Directory
  static async deleteDirectory(dirPath: string): Promise<boolean> {
    try {
      await RNFS.unlink(dirPath);
      return true;
    } catch (error) {
      console.error('Delete directory error:', error);
      return false;
    }
  }

  // ล้าง Cache
  static async clearCache(): Promise<{ freed: number }> {
    try {
      const cacheDir = RNFS.CachesDirectoryPath;
      const items = await RNFS.readDir(cacheDir);
      let freed = 0;

      for (const item of items) {
        const stat = await RNFS.stat(item.path);
        freed += stat.size;
        await RNFS.unlink(item.path);
      }

      return { freed };
    } catch (error) {
      console.error('Clear cache error:', error);
      return { freed: 0 };
    }
  }

  // คำนวณขนาด Directory
  static async getDirectorySize(dirPath: string): Promise<number> {
    try {
      const items = await RNFS.readDir(dirPath);
      let totalSize = 0;

      for (const item of items) {
        if (item.isFile()) {
          totalSize += item.size;
        } else {
          totalSize += await this.getDirectorySize(item.path);
        }
      }

      return totalSize;
    } catch (error) {
      return 0;
    }
  }

  // ค้นหาไฟล์
  static async searchFiles(
    dirPath: string,
    query: string,
    recursive = true
  ): Promise<FileItem[]> {
    const results: FileItem[] = [];

    try {
      const items = await RNFS.readDir(dirPath);

      for (const item of items) {
        if (item.name.toLowerCase().includes(query.toLowerCase())) {
          results.push({
            name: item.name,
            path: item.path,
            isDirectory: item.isDirectory(),
            size: item.size,
            modified: new Date(item.mtime || Date.now()),
            type: item.isFile() ? this.getFileType(item.name) : 'directory',
          });
        }

        if (recursive && item.isDirectory()) {
          const subResults = await this.searchFiles(item.path, query, true);
          results.push(...subResults);
        }
      }
    } catch (error) {
      console.error('Search error:', error);
    }

    return results;
  }

  // ตรวจสอบ type จากนามสกุล
  static getFileType(filename: string): string {
    const ext = filename.split('.').pop()?.toLowerCase() || '';

    const typeMap: Record<string, string> = {
      // Documents
      pdf: 'pdf', doc: 'word', docx: 'word', xls: 'excel', xlsx: 'excel',
      ppt: 'powerpoint', pptx: 'powerpoint', txt: 'text', md: 'markdown',
      // Images
      jpg: 'image', jpeg: 'image', png: 'image', gif: 'image', webp: 'image',
      // Audio
      mp3: 'audio', wav: 'audio', aac: 'audio', m4a: 'audio',
      // Video
      mp4: 'video', mov: 'video', avi: 'video', mkv: 'video',
      // Code
      js: 'code', ts: 'code', json: 'data', xml: 'data',
    };

    return typeMap[ext] || 'other';
  }
}

export default DirectoryService;
```

---

## 5. Document Picker {#document-picker}

```typescript
// src/hooks/useDocumentPicker.ts
import { useCallback } from 'react';
import DocumentPicker, {
  DocumentPickerResponse,
  types,
} from 'react-native-document-picker';
import { Alert } from 'react-native';

interface PickedDocument {
  uri: string;
  name: string;
  type: string;
  size: number;
}

export function useDocumentPicker() {
  // เลือกเอกสาร
  const pickDocument = useCallback(async (
    allowedTypes: string[] = [types.allFiles]
  ): Promise<PickedDocument | null> => {
    try {
      const result = await DocumentPicker.pickSingle({
        type: allowedTypes,
        copyTo: 'documentDirectory', // คัดลอกไปยัง app's documents
      });

      return {
        uri: result.fileCopyUri || result.uri,
        name: result.name || `document_${Date.now()}`,
        type: result.type || 'application/octet-stream',
        size: result.size || 0,
      };
    } catch (error) {
      if (!DocumentPicker.isCancel(error)) {
        Alert.alert('ข้อผิดพลาด', 'ไม่สามารถเลือกไฟล์ได้');
        console.error('Pick document error:', error);
      }
      return null;
    }
  }, []);

  // เลือกหลายเอกสาร
  const pickMultipleDocuments = useCallback(async (
    allowedTypes: string[] = [types.allFiles]
  ): Promise<PickedDocument[]> => {
    try {
      const results = await DocumentPicker.pickMultiple({
        type: allowedTypes,
        copyTo: 'documentDirectory',
      });

      return results.map(result => ({
        uri: result.fileCopyUri || result.uri,
        name: result.name || `document_${Date.now()}`,
        type: result.type || 'application/octet-stream',
        size: result.size || 0,
      }));
    } catch (error) {
      if (!DocumentPicker.isCancel(error)) {
        console.error('Pick multiple documents error:', error);
      }
      return [];
    }
  }, []);

  // เลือกไฟล์รูปภาพ
  const pickImage = () => pickDocument([types.images]);

  // เลือกไฟล์ PDF
  const pickPDF = () => pickDocument([types.pdf]);

  // เลือกไฟล์ Office
  const pickOfficeDocument = () => pickDocument([
    types.doc, types.docx, types.xls, types.xlsx, types.ppt, types.pptx
  ]);

  // เลือกไฟล์เสียง/วิดีโอ
  const pickMedia = () => pickDocument([types.audio, types.video]);

  return {
    pickDocument,
    pickMultipleDocuments,
    pickImage,
    pickPDF,
    pickOfficeDocument,
    pickMedia,
  };
}
```

---

## 6. File Sharing {#file-sharing}

```typescript
// src/utils/fileSharing.ts
import Share from 'react-native-share';
import RNFS from 'react-native-fs';
import { Platform } from 'react-native';

interface ShareFileOptions {
  filePath: string;
  title?: string;
  message?: string;
  type?: string;
}

export async function shareFile(options: ShareFileOptions): Promise<void> {
  const { filePath, title, message, type } = options;

  try {
    // อ่านไฟล์เป็น base64
    const base64 = await RNFS.readFile(filePath, 'base64');
    const fileType = type || getContentType(filePath);

    await Share.open({
      title: title || 'แชร์ไฟล์',
      message,
      url: `data:${fileType};base64,${base64}`,
      filename: filePath.split('/').pop(),
      type: fileType,
      showAppsToView: true,
      failOnCancel: false,
    });
  } catch (error) {
    if ((error as any).message !== 'User did not share') {
      throw error;
    }
  }
}

export async function shareText(text: string, title?: string): Promise<void> {
  await Share.open({
    title: title || 'แชร์',
    message: text,
    failOnCancel: false,
  });
}

export async function shareImage(imageUri: string, message?: string): Promise<void> {
  try {
    let url = imageUri;

    // Convert file:// to base64 if needed
    if (!imageUri.startsWith('http')) {
      const base64 = await RNFS.readFile(imageUri.replace('file://', ''), 'base64');
      url = `data:image/jpeg;base64,${base64}`;
    }

    await Share.open({
      url,
      message,
      failOnCancel: false,
    });
  } catch (error) {
    throw error;
  }
}

function getContentType(filePath: string): string {
  const ext = filePath.split('.').pop()?.toLowerCase() || '';
  const contentTypes: Record<string, string> = {
    pdf: 'application/pdf',
    doc: 'application/msword',
    docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    xls: 'application/vnd.ms-excel',
    xlsx: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    jpg: 'image/jpeg',
    jpeg: 'image/jpeg',
    png: 'image/png',
    gif: 'image/gif',
    mp3: 'audio/mp3',
    mp4: 'video/mp4',
    txt: 'text/plain',
  };
  return contentTypes[ext] || 'application/octet-stream';
}
```

---

## 7. Workshop: Note Taking App {#workshop}

```typescript
// src/screens/NotesScreen.tsx
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  TextInput,
  StyleSheet,
  Alert,
  Modal,
  ScrollView,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import RNFS from 'react-native-fs';
import Icon from 'react-native-vector-icons/MaterialIcons';
import FileSystemService from '../services/FileSystemService';
import { shareText } from '../utils/fileSharing';

interface Note {
  id: string;
  title: string;
  content: string;
  tags: string[];
  color: string;
  createdAt: string;
  updatedAt: string;
  isPinned: boolean;
  attachments: Attachment[];
}

interface Attachment {
  id: string;
  name: string;
  path: string;
  type: string;
  size: number;
}

const NOTE_COLORS = ['#FFFFFF', '#FFF9C4', '#E8F5E9', '#E3F2FD', '#FCE4EC', '#F3E5F5'];
const NOTES_DIR = `${RNFS.DocumentDirectoryPath}/notes`;

const NotesScreen: React.FC = () => {
  const [notes, setNotes] = useState<Note[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedNote, setSelectedNote] = useState<Note | null>(null);
  const [isEditing, setIsEditing] = useState(false);
  const [editTitle, setEditTitle] = useState('');
  const [editContent, setEditContent] = useState('');
  const [editColor, setEditColor] = useState('#FFFFFF');
  const [viewMode, setViewMode] = useState<'grid' | 'list'>('grid');

  useEffect(() => {
    initializeNotesDirectory();
    loadAllNotes();
  }, []);

  const initializeNotesDirectory = async () => {
    const exists = await RNFS.exists(NOTES_DIR);
    if (!exists) {
      await RNFS.mkdir(NOTES_DIR);
    }
  };

  const loadAllNotes = async () => {
    try {
      const exists = await RNFS.exists(NOTES_DIR);
      if (!exists) return;

      const files = await RNFS.readDir(NOTES_DIR);
      const noteFiles = files.filter(f => f.name.endsWith('.json'));

      const loadedNotes = await Promise.all(
        noteFiles.map(async (file) => {
          const content = await RNFS.readFile(file.path, 'utf8');
          return JSON.parse(content) as Note;
        })
      );

      // Sort: pinned first, then by updatedAt
      const sortedNotes = loadedNotes.sort((a, b) => {
        if (a.isPinned && !b.isPinned) return -1;
        if (!a.isPinned && b.isPinned) return 1;
        return new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime();
      });

      setNotes(sortedNotes);
    } catch (error) {
      console.error('Load notes error:', error);
    }
  };

  const saveNote = async (note: Note): Promise<void> => {
    const filePath = `${NOTES_DIR}/${note.id}.json`;
    await RNFS.writeFile(filePath, JSON.stringify(note, null, 2), 'utf8');
  };

  const createNote = useCallback(async () => {
    const newNote: Note = {
      id: `note_${Date.now()}`,
      title: editTitle.trim() || 'โน้ตใหม่',
      content: editContent,
      tags: [],
      color: editColor,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      isPinned: false,
      attachments: [],
    };

    await saveNote(newNote);
    setNotes(prev => [newNote, ...prev]);
    setIsEditing(false);
    resetEditForm();
  }, [editTitle, editContent, editColor]);

  const updateNote = useCallback(async (note: Note) => {
    const updatedNote = {
      ...note,
      title: editTitle.trim() || 'โน้ตใหม่',
      content: editContent,
      color: editColor,
      updatedAt: new Date().toISOString(),
    };

    await saveNote(updatedNote);
    setNotes(prev => prev.map(n => n.id === note.id ? updatedNote : n));
    setSelectedNote(null);
    setIsEditing(false);
    resetEditForm();
  }, [editTitle, editContent, editColor]);

  const deleteNote = async (noteId: string) => {
    Alert.alert(
      'ลบโน้ต',
      'ต้องการลบโน้ตนี้หรือไม่?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ลบ',
          style: 'destructive',
          onPress: async () => {
            const filePath = `${NOTES_DIR}/${noteId}.json`;
            await RNFS.unlink(filePath);
            setNotes(prev => prev.filter(n => n.id !== noteId));
            setSelectedNote(null);
          },
        },
      ]
    );
  };

  const togglePin = async (note: Note) => {
    const updatedNote = { ...note, isPinned: !note.isPinned };
    await saveNote(updatedNote);
    setNotes(prev => {
      const updated = prev.map(n => n.id === note.id ? updatedNote : n);
      return updated.sort((a, b) => {
        if (a.isPinned && !b.isPinned) return -1;
        if (!a.isPinned && b.isPinned) return 1;
        return 0;
      });
    });
  };

  const shareNote = async (note: Note) => {
    const text = `${note.title}\n\n${note.content}`;
    await shareText(text, note.title);
  };

  const exportNote = async (note: Note) => {
    const exportPath = `${RNFS.DocumentDirectoryPath}/exports/${note.id}.txt`;
    const exportDir = `${RNFS.DocumentDirectoryPath}/exports`;

    // สร้าง exports directory
    const exists = await RNFS.exists(exportDir);
    if (!exists) await RNFS.mkdir(exportDir);

    const content = `${note.title}\n${'='.repeat(40)}\n\n${note.content}\n\nสร้างเมื่อ: ${new Date(note.createdAt).toLocaleDateString('th-TH')}\nแก้ไขล่าสุด: ${new Date(note.updatedAt).toLocaleDateString('th-TH')}`;

    await RNFS.writeFile(exportPath, content, 'utf8');
    Alert.alert('สำเร็จ', `บันทึกไฟล์ที่ ${exportPath}`);
  };

  const resetEditForm = () => {
    setEditTitle('');
    setEditContent('');
    setEditColor('#FFFFFF');
  };

  const openNoteForEdit = (note: Note) => {
    setSelectedNote(note);
    setEditTitle(note.title);
    setEditContent(note.content);
    setEditColor(note.color);
    setIsEditing(true);
  };

  const filteredNotes = notes.filter(note =>
    note.title.toLowerCase().includes(searchQuery.toLowerCase()) ||
    note.content.toLowerCase().includes(searchQuery.toLowerCase())
  );

  const formatDate = (dateStr: string) => {
    const date = new Date(dateStr);
    const now = new Date();
    const diff = now.getTime() - date.getTime();
    const days = Math.floor(diff / 86400000);

    if (days === 0) return date.toLocaleTimeString('th-TH', { hour: '2-digit', minute: '2-digit' });
    if (days === 1) return 'เมื่อวาน';
    if (days < 7) return `${days} วันที่แล้ว`;
    return date.toLocaleDateString('th-TH');
  };

  const renderNote = ({ item }: { item: Note }) => (
    <TouchableOpacity
      style={[
        styles.noteCard,
        { backgroundColor: item.color },
        viewMode === 'grid' && styles.gridCard,
        item.isPinned && styles.pinnedCard,
      ]}
      onPress={() => openNoteForEdit(item)}
      onLongPress={() => {
        Alert.alert(item.title, '', [
          { text: 'ยกเลิก', style: 'cancel' },
          {
            text: item.isPinned ? 'เลิกปักหมุด' : 'ปักหมุด',
            onPress: () => togglePin(item),
          },
          { text: 'แชร์', onPress: () => shareNote(item) },
          { text: 'Export', onPress: () => exportNote(item) },
          {
            text: 'ลบ',
            style: 'destructive',
            onPress: () => deleteNote(item.id),
          },
        ]);
      }}
    >
      {item.isPinned && (
        <Icon name="push-pin" size={14} color="#FF9800" style={styles.pinIcon} />
      )}
      <Text style={styles.noteTitle} numberOfLines={2}>{item.title}</Text>
      <Text style={styles.noteContent} numberOfLines={viewMode === 'grid' ? 4 : 2}>
        {item.content}
      </Text>
      <View style={styles.noteFooter}>
        <Text style={styles.noteDate}>{formatDate(item.updatedAt)}</Text>
        {item.attachments.length > 0 && (
          <Icon name="attach-file" size={14} color="#9E9E9E" />
        )}
      </View>
    </TouchableOpacity>
  );

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>โน้ตของฉัน</Text>
        <View style={styles.headerActions}>
          <TouchableOpacity onPress={() => setViewMode(v => v === 'grid' ? 'list' : 'grid')}>
            <Icon
              name={viewMode === 'grid' ? 'view-list' : 'grid-view'}
              size={24}
              color="#333"
            />
          </TouchableOpacity>
        </View>
      </View>

      {/* Search */}
      <View style={styles.searchContainer}>
        <Icon name="search" size={20} color="#9E9E9E" />
        <TextInput
          style={styles.searchInput}
          placeholder="ค้นหาโน้ต..."
          value={searchQuery}
          onChangeText={setSearchQuery}
        />
        {searchQuery.length > 0 && (
          <TouchableOpacity onPress={() => setSearchQuery('')}>
            <Icon name="close" size={20} color="#9E9E9E" />
          </TouchableOpacity>
        )}
      </View>

      {/* Stats */}
      <View style={styles.stats}>
        <Text style={styles.statsText}>
          {filteredNotes.length} โน้ต
          {notes.filter(n => n.isPinned).length > 0 &&
            ` • ${notes.filter(n => n.isPinned).length} ปักหมุด`
          }
        </Text>
      </View>

      {/* Notes List */}
      <FlatList
        data={filteredNotes}
        renderItem={renderNote}
        keyExtractor={item => item.id}
        numColumns={viewMode === 'grid' ? 2 : 1}
        key={viewMode} // Force re-render เมื่อ viewMode เปลี่ยน
        contentContainerStyle={styles.notesList}
        ListEmptyComponent={
          <View style={styles.emptyState}>
            <Text style={styles.emptyIcon}>📝</Text>
            <Text style={styles.emptyTitle}>
              {searchQuery ? 'ไม่พบโน้ต' : 'ยังไม่มีโน้ต'}
            </Text>
            <Text style={styles.emptySubtitle}>
              {searchQuery ? 'ลองค้นหาด้วยคำอื่น' : 'กดปุ่ม + เพื่อสร้างโน้ตแรก'}
            </Text>
          </View>
        }
      />

      {/* FAB */}
      <TouchableOpacity
        style={styles.fab}
        onPress={() => {
          setSelectedNote(null);
          resetEditForm();
          setIsEditing(true);
        }}
      >
        <Icon name="add" size={28} color="white" />
      </TouchableOpacity>

      {/* Edit Modal */}
      <Modal
        visible={isEditing}
        animationType="slide"
        presentationStyle="pageSheet"
        onRequestClose={() => setIsEditing(false)}
      >
        <KeyboardAvoidingView
          style={{ flex: 1, backgroundColor: editColor }}
          behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        >
          {/* Modal Header */}
          <View style={styles.modalHeader}>
            <TouchableOpacity onPress={() => setIsEditing(false)}>
              <Icon name="close" size={24} color="#333" />
            </TouchableOpacity>
            <Text style={styles.modalTitle}>
              {selectedNote ? 'แก้ไขโน้ต' : 'โน้ตใหม่'}
            </Text>
            <TouchableOpacity
              onPress={() => selectedNote ? updateNote(selectedNote) : createNote()}
            >
              <Text style={styles.saveButton}>บันทึก</Text>
            </TouchableOpacity>
          </View>

          <ScrollView style={styles.modalContent}>
            {/* Title */}
            <TextInput
              style={styles.titleInput}
              placeholder="หัวข้อ..."
              value={editTitle}
              onChangeText={setEditTitle}
              maxLength={100}
              multiline
            />

            {/* Content */}
            <TextInput
              style={styles.contentInput}
              placeholder="เริ่มเขียน..."
              value={editContent}
              onChangeText={setEditContent}
              multiline
              textAlignVertical="top"
            />
          </ScrollView>

          {/* Color Picker */}
          <View style={styles.colorPicker}>
            <Text style={styles.colorPickerLabel}>สีพื้นหลัง:</Text>
            {NOTE_COLORS.map(color => (
              <TouchableOpacity
                key={color}
                style={[
                  styles.colorOption,
                  { backgroundColor: color },
                  editColor === color && styles.selectedColor,
                ]}
                onPress={() => setEditColor(color)}
              />
            ))}
          </View>
        </KeyboardAvoidingView>
      </Modal>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: 'white',
  },
  headerTitle: { fontSize: 24, fontWeight: 'bold' },
  headerActions: { flexDirection: 'row', gap: 12 },
  searchContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    margin: 16,
    marginTop: 8,
    padding: 10,
    borderRadius: 12,
    gap: 8,
    elevation: 1,
  },
  searchInput: { flex: 1, fontSize: 15 },
  stats: { paddingHorizontal: 16, marginBottom: 8 },
  statsText: { color: '#9E9E9E', fontSize: 13 },
  notesList: { padding: 8, paddingBottom: 80 },
  noteCard: {
    flex: 1,
    margin: 6,
    padding: 14,
    borderRadius: 12,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 3,
    minHeight: 100,
  },
  gridCard: { maxWidth: '48%' },
  pinnedCard: { borderWidth: 1, borderColor: '#FF9800' },
  pinIcon: { position: 'absolute', top: 8, right: 8 },
  noteTitle: { fontSize: 15, fontWeight: 'bold', marginBottom: 6, color: '#1a1a1a' },
  noteContent: { fontSize: 13, color: '#666', lineHeight: 18, flex: 1 },
  noteFooter: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginTop: 8 },
  noteDate: { fontSize: 11, color: '#9E9E9E' },
  emptyState: { alignItems: 'center', padding: 60, gap: 12 },
  emptyIcon: { fontSize: 56 },
  emptyTitle: { fontSize: 18, fontWeight: 'bold', color: '#666' },
  emptySubtitle: { color: '#9E9E9E', textAlign: 'center' },
  fab: {
    position: 'absolute',
    right: 24,
    bottom: 24,
    width: 60,
    height: 60,
    borderRadius: 30,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 8,
    shadowColor: '#2196F3',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.4,
    shadowRadius: 8,
  },
  modalHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    paddingTop: Platform.OS === 'ios' ? 20 : 16,
    borderBottomWidth: 1,
    borderBottomColor: 'rgba(0,0,0,0.1)',
  },
  modalTitle: { fontSize: 17, fontWeight: 'bold' },
  saveButton: { color: '#2196F3', fontWeight: 'bold', fontSize: 16 },
  modalContent: { flex: 1, padding: 16 },
  titleInput: { fontSize: 22, fontWeight: 'bold', marginBottom: 12, color: '#1a1a1a' },
  contentInput: { fontSize: 16, color: '#333', lineHeight: 24, minHeight: 300 },
  colorPicker: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: 'rgba(255,255,255,0.8)',
    gap: 8,
  },
  colorPickerLabel: { fontSize: 13, color: '#666' },
  colorOption: {
    width: 32,
    height: 32,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: '#E0E0E0',
  },
  selectedColor: { borderWidth: 3, borderColor: '#2196F3' },
});

export default NotesScreen;
```

---

## Tips และ Best Practices

### 1. File Encryption

```typescript
// เข้ารหัสไฟล์ sensitive data
import CryptoJS from 'crypto-js';

async function saveEncryptedFile(path: string, data: any, key: string): Promise<void> {
  const json = JSON.stringify(data);
  const encrypted = CryptoJS.AES.encrypt(json, key).toString();
  await RNFS.writeFile(path, encrypted, 'utf8');
}

async function readEncryptedFile<T>(path: string, key: string): Promise<T | null> {
  const encrypted = await RNFS.readFile(path, 'utf8');
  const decrypted = CryptoJS.AES.decrypt(encrypted, key).toString(CryptoJS.enc.Utf8);
  return JSON.parse(decrypted) as T;
}
```

### 2. Auto-save

```typescript
// Auto-save ทุก 30 วินาที
const autoSaveTimer = useRef<ReturnType<typeof setInterval>>();

useEffect(() => {
  autoSaveTimer.current = setInterval(() => {
    if (editContent !== originalContent) {
      saveNote();
    }
  }, 30000);

  return () => {
    if (autoSaveTimer.current) clearInterval(autoSaveTimer.current);
  };
}, [editContent, originalContent]);
```

### สรุป

- ใช้ DocumentDirectoryPath สำหรับไฟล์ที่ user สร้าง
- ใช้ CachesDirectoryPath สำหรับไฟล์ชั่วคราว
- ตรวจสอบ exists ก่อนอ่านไฟล์เสมอ
- จัดการ permissions สำหรับ external storage บน Android
- ใช้ JSON.stringify/parse สำหรับเก็บข้อมูล structured
- Export ไฟล์ให้ user ผ่าน Share API
