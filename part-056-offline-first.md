# Part 056: Offline First Apps ใน React Native

## Offline-First Strategy

Offline-First คือแนวคิดในการออกแบบแอปที่ให้ทำงานได้อย่างสมบูรณ์แม้ไม่มี internet connection แล้วค่อย sync ข้อมูลเมื่อมี network กลับมา

### ทำไมต้อง Offline First?

```
1. UX ดีกว่า - ไม่มี loading spinner ตลอดเวลา
2. รองรับ network ที่ไม่เสถียร (รถไฟใต้ดิน, ลิฟต์)
3. ประหยัด data - ไม่ fetch ข้อมูลซ้ำๆ
4. Fast startup - โหลดจาก cache ก่อน
5. Reliability - ใช้ได้แม้ไม่มี network
```

### Architecture แบบ Offline First

```
User Action → Local Storage → Sync Queue → Server
     ↑                ↓
     └──── Read ◄─────┘
```

---

## Network Detection

```typescript
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';

// Installation
// npm install @react-native-community/netinfo

class NetworkManager {
  private static isConnected = true;
  private static listeners: ((connected: boolean) => void)[] = [];
  private static unsubscribe: (() => void) | null = null;

  static initialize(): void {
    this.unsubscribe = NetInfo.addEventListener((state: NetInfoState) => {
      const connected = !!(state.isConnected && state.isInternetReachable);
      
      if (connected !== this.isConnected) {
        this.isConnected = connected;
        this.notifyListeners(connected);
        
        if (connected) {
          console.log('[Network] Back online');
        } else {
          console.log('[Network] Gone offline');
        }
      }
    });
  }

  static addListener(listener: (connected: boolean) => void): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  static async getStatus(): Promise<boolean> {
    const state = await NetInfo.fetch();
    return !!(state.isConnected && state.isInternetReachable);
  }

  static isOnline(): boolean {
    return this.isConnected;
  }

  static destroy(): void {
    this.unsubscribe?.();
  }

  private static notifyListeners(connected: boolean): void {
    this.listeners.forEach(listener => listener(connected));
  }
}

// React Hook สำหรับ Network Status
import { useState, useEffect } from 'react';

export const useNetworkStatus = () => {
  const [isOnline, setIsOnline] = useState(true);
  const [networkType, setNetworkType] = useState<string>('unknown');

  useEffect(() => {
    const checkInitialStatus = async () => {
      const state = await NetInfo.fetch();
      setIsOnline(!!(state.isConnected && state.isInternetReachable));
      setNetworkType(state.type);
    };

    checkInitialStatus();

    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(!!(state.isConnected && state.isInternetReachable));
      setNetworkType(state.type);
    });

    return () => unsubscribe();
  }, []);

  return { isOnline, networkType };
};
```

---

## Queue Operations

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';

type OperationType = 'CREATE' | 'UPDATE' | 'DELETE';

interface QueuedOperation {
  id: string;
  type: OperationType;
  resource: string;       // 'notes', 'tasks', 'comments'
  resourceId: string;     // ID ของ resource ที่ถูกแก้ไข
  data: any;
  createdAt: number;
  retryCount: number;
  maxRetries: number;
  priority: 'low' | 'normal' | 'high';
}

class OperationQueue {
  private static STORAGE_KEY = '@operation_queue';
  private static queue: QueuedOperation[] = [];
  private static isProcessing = false;

  static async load(): Promise<void> {
    try {
      const stored = await AsyncStorage.getItem(this.STORAGE_KEY);
      if (stored) {
        this.queue = JSON.parse(stored);
      }
    } catch (error) {
      console.error('Failed to load queue:', error);
    }
  }

  static async enqueue(
    operation: Omit<QueuedOperation, 'id' | 'createdAt' | 'retryCount'>
  ): Promise<string> {
    const id = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    const queuedOp: QueuedOperation = {
      ...operation,
      id,
      createdAt: Date.now(),
      retryCount: 0,
    };

    // เพิ่มตาม priority
    if (operation.priority === 'high') {
      this.queue.unshift(queuedOp);
    } else {
      this.queue.push(queuedOp);
    }

    await this.save();

    // ลอง process ทันที
    this.process();

    return id;
  }

  static async process(): Promise<void> {
    if (this.isProcessing) return;

    const networkState = await NetInfo.fetch();
    if (!networkState.isConnected || !networkState.isInternetReachable) {
      console.log('[Queue] Offline, skipping process');
      return;
    }

    this.isProcessing = true;
    
    const toProcess = [...this.queue].sort((a, b) => {
      const priorityOrder = { high: 0, normal: 1, low: 2 };
      return priorityOrder[a.priority] - priorityOrder[b.priority];
    });

    for (const operation of toProcess) {
      try {
        await this.executeOperation(operation);
        
        // ลบ operation ที่สำเร็จ
        this.queue = this.queue.filter(op => op.id !== operation.id);
        await this.save();
      } catch (error) {
        console.error(`[Queue] Operation ${operation.id} failed:`, error);
        
        const index = this.queue.findIndex(op => op.id === operation.id);
        if (index !== -1) {
          this.queue[index].retryCount++;
          
          if (this.queue[index].retryCount >= operation.maxRetries) {
            console.error(`[Queue] Operation ${operation.id} exceeded max retries`);
            this.queue.splice(index, 1);
          }
          
          await this.save();
        }
      }
    }

    this.isProcessing = false;
  }

  private static async executeOperation(op: QueuedOperation): Promise<any> {
    const methodMap: Record<OperationType, string> = {
      CREATE: 'POST',
      UPDATE: 'PUT',
      DELETE: 'DELETE',
    };

    const url = `https://api.example.com/${op.resource}${
      op.type !== 'CREATE' ? `/${op.resourceId}` : ''
    }`;

    const response = await fetch(url, {
      method: methodMap[op.type],
      headers: {
        'Content-Type': 'application/json',
      },
      body: op.type !== 'DELETE' ? JSON.stringify(op.data) : undefined,
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return response.json();
  }

  private static async save(): Promise<void> {
    await AsyncStorage.setItem(this.STORAGE_KEY, JSON.stringify(this.queue));
  }

  static getPendingCount(): number {
    return this.queue.length;
  }

  static getPendingOperations(): QueuedOperation[] {
    return [...this.queue];
  }

  static async clear(): Promise<void> {
    this.queue = [];
    await this.save();
  }
}
```

---

## Sync on Reconnect

```typescript
import { useEffect, useRef, useCallback } from 'react';
import NetInfo from '@react-native-community/netinfo';

const useSyncOnReconnect = (syncFn: () => Promise<void>) => {
  const wasOffline = useRef(false);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(async state => {
      const isNowOnline = !!(state.isConnected && state.isInternetReachable);
      
      if (isNowOnline && wasOffline.current) {
        console.log('[Sync] Back online, syncing...');
        wasOffline.current = false;
        
        try {
          await syncFn();
        } catch (error) {
          console.error('[Sync] Failed to sync:', error);
        }
      } else if (!isNowOnline) {
        wasOffline.current = true;
      }
    });

    return () => unsubscribe();
  }, [syncFn]);
};
```

---

## Conflict Resolution

```typescript
interface ConflictResolutionStrategy {
  resolve: (local: any, server: any) => any;
}

class ConflictResolver {
  // Last Write Wins - ใช้ timestamp ล่าสุด
  static lastWriteWins(): ConflictResolutionStrategy {
    return {
      resolve: (local, server) => {
        return local.updatedAt > server.updatedAt ? local : server;
      },
    };
  }

  // Server Wins - server เสมอชนะ
  static serverWins(): ConflictResolutionStrategy {
    return {
      resolve: (local, server) => server,
    };
  }

  // Client Wins - client เสมอชนะ
  static clientWins(): ConflictResolutionStrategy {
    return {
      resolve: (local, server) => local,
    };
  }

  // Merge - รวมข้อมูลจากทั้งสอง
  static merge(mergeFields: string[]): ConflictResolutionStrategy {
    return {
      resolve: (local, server) => {
        const merged = { ...server };
        for (const field of mergeFields) {
          if (local[field] !== undefined) {
            merged[field] = local[field];
          }
        }
        return merged;
      },
    };
  }

  // Manual - ให้ผู้ใช้เลือก
  static manual(onConflict: (local: any, server: any) => Promise<any>): ConflictResolutionStrategy {
    return {
      resolve: async (local, server) => {
        return onConflict(local, server);
      },
    };
  }
}

// ตัวอย่าง: Merge note changes
const resolveNoteConflict = async (localNote: any, serverNote: any) => {
  // ถ้า content เหมือนกัน ให้ใช้ timestamp ล่าสุด
  if (localNote.content === serverNote.content) {
    return ConflictResolver.lastWriteWins().resolve(localNote, serverNote);
  }
  
  // ถ้า content ต่างกัน ให้ผู้ใช้เลือก
  return new Promise(resolve => {
    Alert.alert(
      'ข้อมูลขัดแย้งกัน',
      'มีการแก้ไขทั้งบน device และ server กรุณาเลือกเวอร์ชันที่ต้องการ',
      [
        {
          text: 'ใช้ข้อมูลบน Device',
          onPress: () => resolve(localNote),
        },
        {
          text: 'ใช้ข้อมูลจาก Server',
          onPress: () => resolve(serverNote),
        },
      ]
    );
  });
};
```

---

## Workshop: Offline Notes App

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  Alert,
  ActivityIndicator,
  Modal,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';

interface Note {
  id: string;
  title: string;
  content: string;
  createdAt: string;
  updatedAt: string;
  syncStatus: 'synced' | 'pending' | 'error' | 'conflict';
  localVersion: number;
  serverVersion?: number;
  isDeleted?: boolean;
}

const NOTES_STORAGE_KEY = '@offline_notes';
const API_BASE = 'https://api.example.com';

const OfflineNotesApp: React.FC = () => {
  const [notes, setNotes] = useState<Note[]>([]);
  const [isOnline, setIsOnline] = useState(true);
  const [isSyncing, setIsSyncing] = useState(false);
  const [showEditor, setShowEditor] = useState(false);
  const [editingNote, setEditingNote] = useState<Partial<Note> | null>(null);
  const [pendingCount, setPendingCount] = useState(0);
  const [networkType, setNetworkType] = useState('unknown');

  useEffect(() => {
    loadNotes();
    
    const unsubscribe = NetInfo.addEventListener(state => {
      const connected = !!(state.isConnected && state.isInternetReachable);
      setIsOnline(connected);
      setNetworkType(state.type);
      
      if (connected) {
        syncNotes();
      }
    });

    return () => unsubscribe();
  }, []);

  const loadNotes = async () => {
    try {
      const stored = await AsyncStorage.getItem(NOTES_STORAGE_KEY);
      if (stored) {
        const loadedNotes: Note[] = JSON.parse(stored);
        const activeNotes = loadedNotes.filter(n => !n.isDeleted);
        setNotes(activeNotes);
        setPendingCount(activeNotes.filter(n => n.syncStatus !== 'synced').length);
      }
    } catch (error) {
      console.error('Failed to load notes:', error);
    }
  };

  const saveNotes = async (updatedNotes: Note[]) => {
    try {
      await AsyncStorage.setItem(NOTES_STORAGE_KEY, JSON.stringify(updatedNotes));
      setPendingCount(updatedNotes.filter(n => n.syncStatus !== 'synced' && !n.isDeleted).length);
    } catch (error) {
      console.error('Failed to save notes:', error);
    }
  };

  const createNote = async () => {
    if (!editingNote?.title?.trim()) {
      Alert.alert('ข้อผิดพลาด', 'กรุณาใส่ชื่อ note');
      return;
    }

    const now = new Date().toISOString();
    const newNote: Note = {
      id: editingNote.id || `local-${Date.now()}`,
      title: editingNote.title,
      content: editingNote.content || '',
      createdAt: now,
      updatedAt: now,
      syncStatus: 'pending',
      localVersion: 1,
    };

    const isEditing = !!editingNote.id;
    let updatedNotes: Note[];
    
    if (isEditing) {
      updatedNotes = notes.map(n =>
        n.id === newNote.id
          ? { ...newNote, createdAt: n.createdAt, localVersion: n.localVersion + 1 }
          : n
      );
    } else {
      updatedNotes = [...notes, newNote];
    }

    setNotes(updatedNotes);
    await saveNotes(updatedNotes);
    setShowEditor(false);
    setEditingNote(null);

    if (isOnline) {
      syncNotes(updatedNotes);
    }
  };

  const deleteNote = async (noteId: string) => {
    Alert.alert(
      'ลบ Note',
      'คุณแน่ใจว่าต้องการลบ note นี้?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ลบ',
          style: 'destructive',
          onPress: async () => {
            const updatedNotes = notes.map(n =>
              n.id === noteId
                ? { ...n, isDeleted: true, syncStatus: 'pending' as const, updatedAt: new Date().toISOString() }
                : n
            );
            
            const activeNotes = updatedNotes.filter(n => !n.isDeleted);
            setNotes(activeNotes);
            await saveNotes(updatedNotes); // เก็บ deleted records ไว้ sync ด้วย
            
            if (isOnline) {
              syncNotes(updatedNotes);
            }
          },
        },
      ]
    );
  };

  const syncNotes = useCallback(async (notesToSync?: Note[]) => {
    if (isSyncing) return;
    
    const networkState = await NetInfo.fetch();
    if (!networkState.isConnected || !networkState.isInternetReachable) return;
    
    setIsSyncing(true);
    
    try {
      const allNotes = notesToSync || JSON.parse(
        await AsyncStorage.getItem(NOTES_STORAGE_KEY) || '[]'
      );
      
      const pendingNotes = allNotes.filter((n: Note) => n.syncStatus === 'pending');
      
      if (pendingNotes.length === 0) {
        setIsSyncing(false);
        return;
      }
      
      const updatedNotes = [...allNotes];
      
      for (const note of pendingNotes) {
        try {
          if (note.isDeleted) {
            // Delete on server
            if (!note.id.startsWith('local-')) {
              await fetch(`${API_BASE}/notes/${note.id}`, { method: 'DELETE' });
            }
            // ลบจาก local storage ด้วย
            const idx = updatedNotes.findIndex(n => n.id === note.id);
            if (idx !== -1) updatedNotes.splice(idx, 1);
          } else if (note.id.startsWith('local-')) {
            // Create on server
            const response = await fetch(`${API_BASE}/notes`, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ title: note.title, content: note.content }),
            });
            
            if (response.ok) {
              const serverNote = await response.json();
              const idx = updatedNotes.findIndex(n => n.id === note.id);
              if (idx !== -1) {
                updatedNotes[idx] = {
                  ...updatedNotes[idx],
                  id: serverNote.id, // ใช้ server ID
                  syncStatus: 'synced',
                  serverVersion: serverNote.version || 1,
                };
              }
            }
          } else {
            // Update on server
            const response = await fetch(`${API_BASE}/notes/${note.id}`, {
              method: 'PUT',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                title: note.title,
                content: note.content,
                version: note.serverVersion,
              }),
            });
            
            if (response.ok) {
              const idx = updatedNotes.findIndex(n => n.id === note.id);
              if (idx !== -1) {
                updatedNotes[idx] = {
                  ...updatedNotes[idx],
                  syncStatus: 'synced',
                };
              }
            } else if (response.status === 409) {
              // Conflict!
              const serverNote = await response.json();
              const idx = updatedNotes.findIndex(n => n.id === note.id);
              if (idx !== -1) {
                updatedNotes[idx] = {
                  ...updatedNotes[idx],
                  syncStatus: 'conflict',
                  serverVersion: serverNote.version,
                };
              }
            }
          }
        } catch (error) {
          const idx = updatedNotes.findIndex(n => n.id === note.id);
          if (idx !== -1) {
            updatedNotes[idx] = { ...updatedNotes[idx], syncStatus: 'error' };
          }
        }
      }
      
      await AsyncStorage.setItem(NOTES_STORAGE_KEY, JSON.stringify(updatedNotes));
      const activeNotes = updatedNotes.filter(n => !n.isDeleted);
      setNotes(activeNotes);
      setPendingCount(activeNotes.filter(n => n.syncStatus !== 'synced').length);
    } catch (error) {
      console.error('Sync failed:', error);
    } finally {
      setIsSyncing(false);
    }
  }, [isSyncing]);

  const getSyncStatusIcon = (status: Note['syncStatus']) => {
    switch (status) {
      case 'synced': return '✅';
      case 'pending': return '⏳';
      case 'error': return '❌';
      case 'conflict': return '⚠️';
    }
  };

  const getSyncStatusColor = (status: Note['syncStatus']) => {
    switch (status) {
      case 'synced': return '#4CAF50';
      case 'pending': return '#FF9800';
      case 'error': return '#F44336';
      case 'conflict': return '#9C27B0';
    }
  };

  const openEditor = (note?: Note) => {
    setEditingNote(note || { title: '', content: '' });
    setShowEditor(true);
  };

  return (
    <View style={offlineStyles.container}>
      {/* Network Status Bar */}
      <View style={[
        offlineStyles.networkBar,
        { backgroundColor: isOnline ? '#4CAF50' : '#F44336' }
      ]}>
        <Text style={offlineStyles.networkText}>
          {isOnline ? `🌐 ออนไลน์ (${networkType})` : '📴 ออฟไลน์ - การเปลี่ยนแปลงจะถูก sync เมื่อมี connection'}
        </Text>
        {pendingCount > 0 && (
          <Text style={offlineStyles.pendingBadge}>{pendingCount}</Text>
        )}
      </View>

      {/* Header */}
      <View style={offlineStyles.header}>
        <Text style={offlineStyles.title}>Notes ({notes.length})</Text>
        <View style={offlineStyles.headerActions}>
          {isSyncing && <ActivityIndicator size="small" color="#2196F3" style={{ marginRight: 10 }} />}
          {pendingCount > 0 && isOnline && (
            <TouchableOpacity
              style={offlineStyles.syncButton}
              onPress={() => syncNotes()}
            >
              <Text style={offlineStyles.syncButtonText}>Sync</Text>
            </TouchableOpacity>
          )}
          <TouchableOpacity
            style={offlineStyles.addButton}
            onPress={() => openEditor()}
          >
            <Text style={offlineStyles.addButtonText}>+</Text>
          </TouchableOpacity>
        </View>
      </View>

      {/* Notes List */}
      <FlatList
        data={notes}
        keyExtractor={item => item.id}
        contentContainerStyle={offlineStyles.listContent}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={[
              offlineStyles.noteCard,
              item.syncStatus === 'conflict' && offlineStyles.conflictCard,
            ]}
            onPress={() => openEditor(item)}
            onLongPress={() => deleteNote(item.id)}
          >
            <View style={offlineStyles.noteHeader}>
              <Text style={offlineStyles.noteTitle} numberOfLines={1}>
                {item.title}
              </Text>
              <View style={offlineStyles.syncStatusBadge}>
                <Text>{getSyncStatusIcon(item.syncStatus)}</Text>
              </View>
            </View>
            <Text style={offlineStyles.noteContent} numberOfLines={2}>
              {item.content || '(ไม่มีเนื้อหา)'}
            </Text>
            <Text style={offlineStyles.noteDate}>
              {new Date(item.updatedAt).toLocaleString('th-TH')}
            </Text>
            {item.syncStatus === 'conflict' && (
              <View style={offlineStyles.conflictBanner}>
                <Text style={offlineStyles.conflictText}>⚠️ ข้อมูลขัดแย้ง - กดเพื่อแก้ไข</Text>
              </View>
            )}
          </TouchableOpacity>
        )}
        ListEmptyComponent={
          <View style={offlineStyles.emptyContainer}>
            <Text style={offlineStyles.emptyIcon}>📝</Text>
            <Text style={offlineStyles.emptyText}>ยังไม่มี notes</Text>
            <Text style={offlineStyles.emptySubText}>
              กด + เพื่อสร้าง note ใหม่{'\n'}
              ใช้งานได้แม้ออฟไลน์ 📴
            </Text>
          </View>
        }
      />

      {/* Editor Modal */}
      <Modal
        visible={showEditor}
        animationType="slide"
        onRequestClose={() => setShowEditor(false)}
      >
        <View style={offlineStyles.editorContainer}>
          <View style={offlineStyles.editorHeader}>
            <TouchableOpacity onPress={() => setShowEditor(false)}>
              <Text style={offlineStyles.cancelText}>ยกเลิก</Text>
            </TouchableOpacity>
            <Text style={offlineStyles.editorTitle}>
              {editingNote?.id ? 'แก้ไข Note' : 'Note ใหม่'}
            </Text>
            <TouchableOpacity onPress={createNote}>
              <Text style={offlineStyles.saveText}>บันทึก</Text>
            </TouchableOpacity>
          </View>
          
          <TextInput
            style={offlineStyles.titleInput}
            value={editingNote?.title || ''}
            onChangeText={text => setEditingNote(prev => ({ ...prev, title: text }))}
            placeholder="ชื่อ note"
            fontSize={20}
            fontWeight="bold"
          />
          
          <TextInput
            style={offlineStyles.contentInput}
            value={editingNote?.content || ''}
            onChangeText={text => setEditingNote(prev => ({ ...prev, content: text }))}
            placeholder="เนื้อหา..."
            multiline
            textAlignVertical="top"
            fontSize={15}
          />
        </View>
      </Modal>
    </View>
  );
};

const offlineStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#FFFDE7' },
  networkBar: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingHorizontal: 15,
    paddingVertical: 8,
  },
  networkText: { color: 'white', fontSize: 13, flex: 1 },
  pendingBadge: {
    backgroundColor: 'rgba(255,255,255,0.3)',
    color: 'white',
    fontSize: 12,
    fontWeight: 'bold',
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 10,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 20,
    paddingBottom: 10,
  },
  title: { fontSize: 24, fontWeight: 'bold', color: '#333' },
  headerActions: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  syncButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 15,
  },
  syncButtonText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
  addButton: {
    backgroundColor: '#FF6F00',
    width: 36,
    height: 36,
    borderRadius: 18,
    justifyContent: 'center',
    alignItems: 'center',
  },
  addButtonText: { color: 'white', fontSize: 24, lineHeight: 28 },
  listContent: { padding: 15, paddingTop: 5 },
  noteCard: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 15,
    marginBottom: 12,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 3,
  },
  conflictCard: {
    borderWidth: 2,
    borderColor: '#9C27B0',
  },
  noteHeader: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 8 },
  noteTitle: { fontSize: 16, fontWeight: 'bold', color: '#333', flex: 1 },
  syncStatusBadge: { marginLeft: 8 },
  noteContent: { fontSize: 14, color: '#666', marginBottom: 8 },
  noteDate: { fontSize: 11, color: '#bbb' },
  conflictBanner: {
    backgroundColor: '#F3E5F5',
    padding: 6,
    borderRadius: 6,
    marginTop: 8,
  },
  conflictText: { fontSize: 12, color: '#7B1FA2' },
  emptyContainer: { alignItems: 'center', paddingTop: 60 },
  emptyIcon: { fontSize: 60, marginBottom: 15 },
  emptyText: { fontSize: 20, fontWeight: 'bold', color: '#555', marginBottom: 8 },
  emptySubText: { fontSize: 14, color: '#999', textAlign: 'center', lineHeight: 22 },
  editorContainer: { flex: 1, backgroundColor: 'white' },
  editorHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 15,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  cancelText: { color: '#F44336', fontSize: 16 },
  editorTitle: { fontSize: 17, fontWeight: 'bold', color: '#333' },
  saveText: { color: '#2196F3', fontSize: 16, fontWeight: 'bold' },
  titleInput: {
    padding: 20,
    paddingBottom: 10,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
    color: '#333',
  },
  contentInput: {
    flex: 1,
    padding: 20,
    color: '#444',
  },
});

export default OfflineNotesApp;
```

---

## Tips และ Best Practices

### 1. Optimistic Updates
```typescript
// อัปเดต UI ก่อน แล้วค่อย sync
const updateNoteOptimistically = async (noteId: string, updates: Partial<Note>) => {
  // อัปเดต UI ทันที
  setNotes(prev => prev.map(n => n.id === noteId ? { ...n, ...updates } : n));
  
  try {
    // Sync กับ server
    await syncNote(noteId, updates);
  } catch (error) {
    // ถ้า fail ให้ rollback
    setNotes(prev => prev.map(n => n.id === noteId ? previousNote : n));
    Alert.alert('ข้อผิดพลาด', 'ไม่สามารถบันทึกได้');
  }
};
```

### 2. Background Sync
```typescript
import BackgroundFetch from 'react-native-background-fetch';

BackgroundFetch.configure({
  minimumFetchInterval: 15,
  stopOnTerminate: false,
}, async (taskId) => {
  await OperationQueue.process();
  BackgroundFetch.finish(taskId);
});
```

---

## สรุป

Offline First Apps ต้องการ:
- **Local Storage**: เก็บข้อมูลใน device (AsyncStorage/SQLite/Realm)
- **Operation Queue**: เก็บ operations ที่รอ sync
- **Conflict Resolution**: จัดการ conflict ระหว่าง local/server
- **Network Detection**: รู้ว่าออนไลน์หรือออฟไลน์อยู่
- **Background Sync**: sync เมื่อมี network
- **Optimistic Updates**: อัปเดต UI ทันทีโดยไม่รอ server
