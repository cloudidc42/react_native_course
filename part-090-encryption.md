# Part 090: Encryption ใน React Native

## บทนำ

การเข้ารหัสข้อมูล (Encryption) เป็นส่วนสำคัญในการปกป้องข้อมูลผู้ใช้ เราจะเรียนรู้การใช้ AES, RSA encryption, การเก็บ keys อย่างปลอดภัย และสร้าง encrypted messaging app

## หัวข้อที่จะเรียน

1. AES Encryption
2. RSA Encryption  
3. react-native-crypto
4. Secure Key Storage
5. Workshop: Encrypted Messaging App

---

## 1. การติดตั้ง Crypto Libraries

```bash
# CryptoJS - pure JavaScript
npm install crypto-js
npm install @types/crypto-js

# react-native-crypto (Node.js crypto API)
npm install react-native-crypto
npm install react-native-randombytes

# สำหรับ Keychain
npm install react-native-keychain

# สำหรับ key generation
npm install react-native-rsa-native
```

### ตั้งค่า react-native-crypto

```javascript
// shim.js
if (typeof __dirname === 'undefined') global.__dirname = '/'
if (typeof __filename === 'undefined') global.__filename = ''
if (typeof process === 'undefined') {
  global.process = require('process')
} else {
  const bProcess = require('process')
  for (var p in bProcess) {
    if (!(p in process)) {
      process[p] = bProcess[p]
    }
  }
}

process.browser = false
if (typeof Buffer === 'undefined') global.Buffer = require('buffer').Buffer

// setup react-native-randombytes
global.crypto = require('crypto')

require('react-native-randombytes')
```

---

## 2. AES Encryption

AES (Advanced Encryption Standard) เป็น symmetric encryption ที่ใช้ key เดียวกันทั้งเข้ารหัสและถอดรหัส

### utils/aesEncryption.ts

```typescript
import CryptoJS from 'crypto-js';

interface EncryptedData {
  ciphertext: string;
  iv: string;
  salt: string;
}

class AESEncryption {
  private static readonly KEY_SIZE = 256 / 32; // 256 bits
  private static readonly ITERATIONS = 10000; // PBKDF2 iterations

  // สร้าง key จาก password ด้วย PBKDF2
  static deriveKey(password: string, salt?: string): {
    key: CryptoJS.lib.WordArray;
    salt: string;
  } {
    const saltWords = salt
      ? CryptoJS.enc.Hex.parse(salt)
      : CryptoJS.lib.WordArray.random(128 / 8); // 16 bytes salt

    const key = CryptoJS.PBKDF2(password, saltWords, {
      keySize: AESEncryption.KEY_SIZE,
      iterations: AESEncryption.ITERATIONS,
      hasher: CryptoJS.algo.SHA256,
    });

    return {
      key,
      salt: CryptoJS.enc.Hex.stringify(saltWords),
    };
  }

  // เข้ารหัสข้อความ
  static encrypt(plaintext: string, password: string): EncryptedData {
    const { key, salt } = AESEncryption.deriveKey(password);
    const iv = CryptoJS.lib.WordArray.random(128 / 8); // 16 bytes IV

    const encrypted = CryptoJS.AES.encrypt(plaintext, key, {
      iv: iv,
      mode: CryptoJS.mode.CBC,
      padding: CryptoJS.pad.Pkcs7,
    });

    return {
      ciphertext: encrypted.toString(),
      iv: CryptoJS.enc.Hex.stringify(iv),
      salt,
    };
  }

  // ถอดรหัสข้อความ
  static decrypt(encryptedData: EncryptedData, password: string): string {
    const { key } = AESEncryption.deriveKey(password, encryptedData.salt);
    const iv = CryptoJS.enc.Hex.parse(encryptedData.iv);

    const decrypted = CryptoJS.AES.decrypt(encryptedData.ciphertext, key, {
      iv: iv,
      mode: CryptoJS.mode.CBC,
      padding: CryptoJS.pad.Pkcs7,
    });

    return decrypted.toString(CryptoJS.enc.Utf8);
  }

  // เข้ารหัสด้วย key โดยตรง (สำหรับ session encryption)
  static encryptWithKey(plaintext: string, keyHex: string): EncryptedData {
    const key = CryptoJS.enc.Hex.parse(keyHex);
    const iv = CryptoJS.lib.WordArray.random(128 / 8);

    const encrypted = CryptoJS.AES.encrypt(plaintext, key, {
      iv: iv,
      mode: CryptoJS.mode.GCM, // GCM ปลอดภัยกว่า CBC
    });

    return {
      ciphertext: encrypted.ciphertext.toString(CryptoJS.enc.Hex),
      iv: CryptoJS.enc.Hex.stringify(iv),
      salt: '',
    };
  }

  // เข้ารหัสไฟล์ (binary data)
  static encryptFile(fileData: ArrayBuffer, password: string): EncryptedData {
    const wordArray = CryptoJS.lib.WordArray.create(fileData as any);
    const { key, salt } = AESEncryption.deriveKey(password);
    const iv = CryptoJS.lib.WordArray.random(128 / 8);

    const encrypted = CryptoJS.AES.encrypt(wordArray, key, {
      iv,
      mode: CryptoJS.mode.CBC,
      padding: CryptoJS.pad.Pkcs7,
    });

    return {
      ciphertext: encrypted.toString(),
      iv: CryptoJS.enc.Hex.stringify(iv),
      salt,
    };
  }

  // สร้าง AES key แบบสุ่ม
  static generateKey(bits: 128 | 192 | 256 = 256): string {
    return CryptoJS.lib.WordArray.random(bits / 8).toString(CryptoJS.enc.Hex);
  }

  // HMAC สำหรับ message authentication
  static hmac(message: string, key: string): string {
    return CryptoJS.HmacSHA256(message, key).toString();
  }

  // ตรวจสอบ HMAC
  static verifyHmac(message: string, key: string, expectedHmac: string): boolean {
    const computed = AESEncryption.hmac(message, key);
    // Constant time comparison เพื่อป้องกัน timing attacks
    if (computed.length !== expectedHmac.length) return false;
    let result = 0;
    for (let i = 0; i < computed.length; i++) {
      result |= computed.charCodeAt(i) ^ expectedHmac.charCodeAt(i);
    }
    return result === 0;
  }
}

export default AESEncryption;
```

---

## 3. RSA Encryption

RSA เป็น asymmetric encryption ใช้ public key เข้ารหัส และ private key ถอดรหัส

### utils/rsaEncryption.ts

```typescript
import RsaNative from 'react-native-rsa-native';

interface RSAKeyPair {
  publicKey: string;
  privateKey: string;
}

class RSAEncryption {
  // สร้าง key pair
  static async generateKeyPair(bits: 2048 | 4096 = 2048): Promise<RSAKeyPair> {
    const keys = await RsaNative.generateKeys(bits);
    return {
      publicKey: keys.public,
      privateKey: keys.private,
    };
  }

  // เข้ารหัสด้วย public key
  static async encrypt(message: string, publicKey: string): Promise<string> {
    return await RsaNative.encrypt(message, publicKey);
  }

  // ถอดรหัสด้วย private key
  static async decrypt(ciphertext: string, privateKey: string): Promise<string> {
    return await RsaNative.decrypt(ciphertext, privateKey);
  }

  // Sign ข้อความด้วย private key
  static async sign(message: string, privateKey: string): Promise<string> {
    return await RsaNative.sign(message, privateKey, 'SHA256withRSA');
  }

  // ตรวจสอบ signature
  static async verify(
    message: string,
    signature: string,
    publicKey: string
  ): Promise<boolean> {
    return await RsaNative.verify(message, signature, publicKey, 'SHA256withRSA');
  }

  // Hybrid Encryption: RSA + AES
  // ใช้ RSA เข้ารหัส AES key, ใช้ AES เข้ารหัสข้อมูลจริง
  static async hybridEncrypt(
    plaintext: string,
    recipientPublicKey: string
  ): Promise<{
    encryptedKey: string;
    encryptedData: any;
  }> {
    // สร้าง AES key ชั่วคราว
    const aesKey = AESEncryption.generateKey(256);

    // เข้ารหัส AES key ด้วย RSA public key
    const encryptedKey = await RSAEncryption.encrypt(aesKey, recipientPublicKey);

    // เข้ารหัสข้อมูลจริงด้วย AES
    const encryptedData = AESEncryption.encryptWithKey(plaintext, aesKey);

    return { encryptedKey, encryptedData };
  }

  // ถอดรหัส Hybrid
  static async hybridDecrypt(
    encryptedKey: string,
    encryptedData: any,
    recipientPrivateKey: string
  ): Promise<string> {
    // ถอดรหัส AES key
    const aesKey = await RSAEncryption.decrypt(encryptedKey, recipientPrivateKey);

    // ถอดรหัสข้อมูลด้วย AES key
    // Note: ต้องปรับ AESEncryption.decryptWithKey ด้วย
    return AESEncryption.decrypt(encryptedData, aesKey);
  }
}

// ต้อง import AESEncryption
import AESEncryption from './aesEncryption';

export default RSAEncryption;
```

---

## 4. Secure Key Storage

### utils/keyManager.ts

```typescript
import * as Keychain from 'react-native-keychain';
import AESEncryption from './aesEncryption';
import RSAEncryption from './rsaEncryption';

const KEY_SERVICES = {
  RSA_KEYS: 'app_rsa_keys',
  AES_MASTER_KEY: 'app_aes_master_key',
  SESSION_KEY: 'app_session_key',
};

class KeyManager {
  private static instance: KeyManager;
  private memoryCache: Map<string, string> = new Map();

  static getInstance(): KeyManager {
    if (!KeyManager.instance) {
      KeyManager.instance = new KeyManager();
    }
    return KeyManager.instance;
  }

  // สร้างและเก็บ RSA key pair
  async initializeRSAKeys(): Promise<{ publicKey: string }> {
    // ตรวจสอบว่ามี keys อยู่แล้วหรือไม่
    const existing = await this.getRSAKeys();
    if (existing) {
      return { publicKey: existing.publicKey };
    }

    // สร้าง key pair ใหม่
    const keyPair = await RSAEncryption.generateKeyPair(2048);

    // เก็บไว้ใน Keychain (encrypted)
    await Keychain.setGenericPassword(
      'rsa_keypair',
      JSON.stringify(keyPair),
      {
        service: KEY_SERVICES.RSA_KEYS,
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY_OR_DEVICE_PASSCODE,
        accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      }
    );

    // Cache public key ในหน่วยความจำ (ไม่มีปัญหาด้าน security)
    this.memoryCache.set('publicKey', keyPair.publicKey);

    return { publicKey: keyPair.publicKey };
  }

  // ดึง RSA keys
  async getRSAKeys(): Promise<{ publicKey: string; privateKey: string } | null> {
    try {
      const result = await Keychain.getGenericPassword({
        service: KEY_SERVICES.RSA_KEYS,
      });

      if (!result) return null;
      return JSON.parse(result.password);
    } catch {
      return null;
    }
  }

  // สร้าง Master Key จาก user password
  async initializeMasterKey(userPassword: string): Promise<void> {
    const { key, salt } = AESEncryption.deriveKey(userPassword);
    const keyHex = key.toString();

    // เก็บ salt (ไม่ sensitive)
    await Keychain.setGenericPassword('salt', salt, {
      service: `${KEY_SERVICES.AES_MASTER_KEY}_salt`,
    });

    // เก็บ derived key ใน Keychain
    await Keychain.setGenericPassword('master_key', keyHex, {
      service: KEY_SERVICES.AES_MASTER_KEY,
      accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY_OR_DEVICE_PASSCODE,
      accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
    });

    // Cache ในหน่วยความจำสำหรับ session
    this.memoryCache.set('masterKey', keyHex);
  }

  // ดึง Master Key
  async getMasterKey(): Promise<string | null> {
    // ลอง cache ก่อน
    const cached = this.memoryCache.get('masterKey');
    if (cached) return cached;

    try {
      const result = await Keychain.getGenericPassword({
        service: KEY_SERVICES.AES_MASTER_KEY,
        authenticationPrompt: {
          title: 'ยืนยันตัวตน',
          subtitle: 'เข้าถึงกุญแจเข้ารหัส',
        },
      });

      if (!result) return null;
      this.memoryCache.set('masterKey', result.password);
      return result.password;
    } catch {
      return null;
    }
  }

  // สร้าง Session Key สำหรับ session นี้
  async createSessionKey(): Promise<string> {
    const sessionKey = AESEncryption.generateKey(256);
    await Keychain.setGenericPassword('session_key', sessionKey, {
      service: KEY_SERVICES.SESSION_KEY,
      accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
    });
    this.memoryCache.set('sessionKey', sessionKey);
    return sessionKey;
  }

  // ล้าง cache เมื่อ logout
  clearMemoryCache(): void {
    this.memoryCache.clear();
  }

  // ลบ keys ทั้งหมด
  async deleteAllKeys(): Promise<void> {
    await Promise.all([
      Keychain.resetGenericPassword({ service: KEY_SERVICES.RSA_KEYS }),
      Keychain.resetGenericPassword({ service: KEY_SERVICES.AES_MASTER_KEY }),
      Keychain.resetGenericPassword({ service: KEY_SERVICES.SESSION_KEY }),
    ]);
    this.clearMemoryCache();
  }
}

export default KeyManager;
```

---

## 5. Workshop: Encrypted Messaging App

### สถาปัตยกรรม

```
User A                           User B
  |                                 |
  |-- [Encrypt with B's Public Key] |
  |                                 |
  |---- Encrypted Message --------> |
  |                                 |
  |              [Decrypt with B's Private Key]
  |                                 |
```

### stores/chatStore.ts

```typescript
import { create } from 'zustand';
import AESEncryption from '../utils/aesEncryption';
import RSAEncryption from '../utils/rsaEncryption';
import KeyManager from '../utils/keyManager';

interface Message {
  id: string;
  senderId: string;
  recipientId: string;
  encryptedContent: string;
  encryptedKey: string; // AES key encrypted with recipient's RSA key
  iv: string;
  salt: string;
  signature: string;
  timestamp: number;
  isRead: boolean;
}

interface DecryptedMessage extends Omit<Message, 'encryptedContent'> {
  content: string;
  isVerified: boolean; // signature verified
}

interface ChatState {
  messages: DecryptedMessage[];
  sendMessage: (
    content: string,
    recipientPublicKey: string,
    recipientId: string
  ) => Promise<void>;
  receiveMessage: (message: Message) => Promise<void>;
  loadMessages: (chatId: string) => Promise<void>;
}

const useChatStore = create<ChatState>((set, get) => ({
  messages: [],

  sendMessage: async (content, recipientPublicKey, recipientId) => {
    const keyManager = KeyManager.getInstance();
    const myKeys = await keyManager.getRSAKeys();
    
    if (!myKeys) throw new Error('ไม่พบ encryption keys');

    // Hybrid Encrypt
    const { encryptedKey, encryptedData } = await RSAEncryption.hybridEncrypt(
      content,
      recipientPublicKey
    );

    // Sign message
    const signature = await RSAEncryption.sign(content, myKeys.privateKey);

    const message: Message = {
      id: generateId(),
      senderId: getCurrentUserId(),
      recipientId,
      encryptedContent: encryptedData.ciphertext,
      encryptedKey,
      iv: encryptedData.iv,
      salt: encryptedData.salt,
      signature,
      timestamp: Date.now(),
      isRead: false,
    };

    // ส่งไปยัง server
    await sendMessageToServer(message);

    // เพิ่มในหน้าจอ (decrypted)
    set(state => ({
      messages: [...state.messages, {
        ...message,
        content,
        isVerified: true,
      }],
    }));
  },

  receiveMessage: async (message) => {
    const keyManager = KeyManager.getInstance();
    const myKeys = await keyManager.getRSAKeys();
    
    if (!myKeys) return;

    try {
      // ถอดรหัส AES key
      const aesKey = await RSAEncryption.decrypt(message.encryptedKey, myKeys.privateKey);

      // ถอดรหัสข้อความ
      const content = AESEncryption.decrypt({
        ciphertext: message.encryptedContent,
        iv: message.iv,
        salt: message.salt,
      }, aesKey);

      // ตรวจสอบ signature (ต้องมี sender's public key)
      const senderPublicKey = await fetchUserPublicKey(message.senderId);
      const isVerified = await RSAEncryption.verify(
        content,
        message.signature,
        senderPublicKey
      );

      set(state => ({
        messages: [...state.messages, {
          ...message,
          content,
          isVerified,
        }],
      }));
    } catch (error) {
      console.error('Failed to decrypt message:', error);
    }
  },

  loadMessages: async (chatId) => {
    // โหลดและถอดรหัส messages จาก storage
  },
}));

// Helper functions
const generateId = () => Math.random().toString(36).substr(2, 9);
const getCurrentUserId = () => 'user_123'; // จาก auth store
const sendMessageToServer = async (message: Message) => {
  await fetch('/api/messages', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(message),
  });
};
const fetchUserPublicKey = async (userId: string): Promise<string> => {
  const response = await fetch(`/api/users/${userId}/public-key`);
  const { publicKey } = await response.json();
  return publicKey;
};

export default useChatStore;
```

### screens/ChatScreen.tsx

```typescript
import React, { useState, useEffect, useRef } from 'react';
import {
  View,
  Text,
  TextInput,
  FlatList,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import useChatStore from '../stores/chatStore';

const ChatScreen: React.FC<{
  route: { params: { userId: string; userName: string; publicKey: string } };
}> = ({ route }) => {
  const { userId, userName, publicKey } = route.params;
  const [input, setInput] = useState('');
  const flatListRef = useRef<FlatList>(null);
  const { messages, sendMessage } = useChatStore();

  const handleSend = async () => {
    if (!input.trim()) return;
    
    await sendMessage(input.trim(), publicKey, userId);
    setInput('');
  };

  const renderMessage = ({ item }: any) => {
    const isMyMessage = item.senderId !== userId;
    
    return (
      <View style={[styles.messageBubble, isMyMessage ? styles.myMessage : styles.theirMessage]}>
        <Text style={[styles.messageText, isMyMessage && styles.myMessageText]}>
          {item.content}
        </Text>
        <View style={styles.messageFooter}>
          <Text style={styles.messageTime}>
            {new Date(item.timestamp).toLocaleTimeString('th-TH', {
              hour: '2-digit', minute: '2-digit'
            })}
          </Text>
          {item.isVerified && (
            <Text style={styles.verifiedBadge}>🔒</Text>
          )}
        </View>
      </View>
    );
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <View style={styles.header}>
        <Text style={styles.headerName}>{userName}</Text>
        <Text style={styles.encryptedLabel}>🔐 End-to-End Encrypted</Text>
      </View>

      <FlatList
        ref={flatListRef}
        data={messages}
        keyExtractor={item => item.id}
        renderItem={renderMessage}
        onContentSizeChange={() => flatListRef.current?.scrollToEnd()}
        contentContainerStyle={styles.messageList}
      />

      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          value={input}
          onChangeText={setInput}
          placeholder="พิมพ์ข้อความ..."
          multiline
          maxLength={1000}
        />
        <TouchableOpacity
          style={[styles.sendButton, !input.trim() && styles.sendButtonDisabled]}
          onPress={handleSend}
          disabled={!input.trim()}
        >
          <Text style={styles.sendIcon}>➤</Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f0f0f0' },
  header: {
    backgroundColor: '#1a237e', padding: 16, paddingTop: 50,
    alignItems: 'center',
  },
  headerName: { color: 'white', fontSize: 18, fontWeight: 'bold' },
  encryptedLabel: { color: 'rgba(255,255,255,0.7)', fontSize: 12, marginTop: 2 },
  messageList: { padding: 16 },
  messageBubble: {
    maxWidth: '75%', padding: 12, borderRadius: 16, marginBottom: 8,
  },
  myMessage: {
    alignSelf: 'flex-end', backgroundColor: '#1a237e',
    borderBottomRightRadius: 4,
  },
  theirMessage: {
    alignSelf: 'flex-start', backgroundColor: 'white',
    borderBottomLeftRadius: 4,
    shadowColor: '#000', shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1, shadowRadius: 2, elevation: 1,
  },
  messageText: { fontSize: 15, color: '#333', lineHeight: 20 },
  myMessageText: { color: 'white' },
  messageFooter: { flexDirection: 'row', alignItems: 'center', marginTop: 4, gap: 4 },
  messageTime: { fontSize: 11, color: 'rgba(255,255,255,0.7)' },
  verifiedBadge: { fontSize: 10 },
  inputContainer: {
    flexDirection: 'row', padding: 12, backgroundColor: 'white',
    alignItems: 'flex-end', borderTopWidth: 1, borderTopColor: '#e0e0e0',
  },
  input: {
    flex: 1, borderWidth: 1, borderColor: '#ddd', borderRadius: 24,
    paddingHorizontal: 16, paddingVertical: 10, maxHeight: 120, fontSize: 15,
  },
  sendButton: {
    width: 44, height: 44, backgroundColor: '#1a237e', borderRadius: 22,
    justifyContent: 'center', alignItems: 'center', marginLeft: 8,
  },
  sendButtonDisabled: { backgroundColor: '#bdbdbd' },
  sendIcon: { color: 'white', fontSize: 18 },
});

export default ChatScreen;
```

---

## 6. Hash Functions

```typescript
import CryptoJS from 'crypto-js';

class HashUtils {
  // SHA-256
  static sha256(data: string): string {
    return CryptoJS.SHA256(data).toString(CryptoJS.enc.Hex);
  }

  // SHA-512
  static sha512(data: string): string {
    return CryptoJS.SHA512(data).toString(CryptoJS.enc.Hex);
  }

  // MD5 (ไม่ควรใช้สำหรับ security)
  static md5(data: string): string {
    return CryptoJS.MD5(data).toString(CryptoJS.enc.Hex);
  }

  // Password Hash ด้วย bcrypt-like approach
  static hashPassword(password: string): { hash: string; salt: string } {
    const salt = CryptoJS.lib.WordArray.random(128 / 8).toString(CryptoJS.enc.Hex);
    const hash = CryptoJS.PBKDF2(password, salt, {
      keySize: 256 / 32,
      iterations: 100000,
      hasher: CryptoJS.algo.SHA512,
    }).toString(CryptoJS.enc.Hex);

    return { hash, salt };
  }

  // ตรวจสอบ password
  static verifyPassword(password: string, hash: string, salt: string): boolean {
    const computed = CryptoJS.PBKDF2(password, salt, {
      keySize: 256 / 32,
      iterations: 100000,
      hasher: CryptoJS.algo.SHA512,
    }).toString(CryptoJS.enc.Hex);

    // Constant time comparison
    if (computed.length !== hash.length) return false;
    let result = 0;
    for (let i = 0; i < computed.length; i++) {
      result |= computed.charCodeAt(i) ^ hash.charCodeAt(i);
    }
    return result === 0;
  }

  // สร้าง checksum สำหรับไฟล์
  static fileChecksum(data: string | ArrayBuffer): string {
    if (typeof data === 'string') {
      return CryptoJS.SHA256(data).toString();
    }
    const wordArray = CryptoJS.lib.WordArray.create(data as any);
    return CryptoJS.SHA256(wordArray).toString();
  }
}

export default HashUtils;
```

---

## Best Practices

### 1. Key Rotation

```typescript
class KeyRotation {
  static async rotateKeys(): Promise<void> {
    const keyManager = KeyManager.getInstance();
    
    // สร้าง key pair ใหม่
    const newKeys = await RSAEncryption.generateKeyPair(2048);
    
    // อัปเดต keys ใน Keychain
    await keyManager.storeNewKeys(newKeys);
    
    // แจ้ง server ถึง public key ใหม่
    await updatePublicKeyOnServer(newKeys.publicKey);
    
    // Re-encrypt ข้อมูลเก่าด้วย key ใหม่
    await migrateEncryptedData();
  }
}
```

### 2. Zero-Knowledge Architecture

```typescript
// ไม่ส่ง plaintext หรือ key ไปยัง server เลย
// ทุกอย่างเข้ารหัสที่ client-side ก่อนส่ง

const uploadEncryptedFile = async (file: Blob, serverPublicKey: string) => {
  // เข้ารหัสก่อนส่ง
  const fileBuffer = await file.arrayBuffer();
  const aesKey = AESEncryption.generateKey(256);
  const encryptedFile = AESEncryption.encryptFile(fileBuffer, aesKey);
  
  // เข้ารหัส AES key ด้วย server's public key
  const encryptedKey = await RSAEncryption.encrypt(aesKey, serverPublicKey);
  
  // ส่ง encrypted data เท่านั้น
  await fetch('/api/files/upload', {
    method: 'POST',
    body: JSON.stringify({
      encryptedFile,
      encryptedKey,
    }),
  });
};
```

---

## Workshop Exercises

1. **Encrypted Notes App** - บันทึกข้อความเข้ารหัสที่เปิดด้วย biometric
2. **Secure File Transfer** - ส่งไฟล์แบบ end-to-end encrypted
3. **Key Exchange Protocol** - implement Diffie-Hellman key exchange
4. **Digital Signature** - sign และ verify documents

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **AES Encryption** - symmetric encryption สำหรับข้อมูล
2. **RSA Encryption** - asymmetric encryption สำหรับ key exchange
3. **Hybrid Encryption** - ผสม RSA + AES เพื่อประสิทธิภาพ
4. **Key Management** - เก็บ keys อย่างปลอดภัยด้วย Keychain
5. **Encrypted Messaging** - สร้าง end-to-end encrypted chat
