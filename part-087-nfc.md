# Part 087: NFC ใน React Native

## บทนำ

NFC (Near Field Communication) เป็นเทคโนโลยีการสื่อสารไร้สายระยะสั้นที่ใช้กันอย่างแพร่หลายในการชำระเงิน, การเข้าถึงอาคาร, และการแลกเปลี่ยนข้อมูล เราจะเรียนรู้การใช้ `react-native-nfc-manager` เพื่อทำงานกับ NFC tags

## สิ่งที่จะเรียนรู้

1. react-native-nfc-manager basics
2. การอ่าน NFC tags
3. การเขียน NFC tags
4. NFC payments integration
5. Workshop: NFC Tag Reader/Writer App

---

## 1. การติดตั้ง react-native-nfc-manager

```bash
npm install react-native-nfc-manager
# หรือ
yarn add react-native-nfc-manager
```

### ตั้งค่า Android

แก้ไข `android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  
  <!-- NFC Permission -->
  <uses-permission android:name="android.permission.NFC" />
  <uses-feature android:name="android.hardware.nfc" android:required="true" />

  <application ...>
    <activity
      android:name=".MainActivity"
      ...>
      
      <!-- NFC Intent Filter -->
      <intent-filter>
        <action android:name="android.nfc.action.NDEF_DISCOVERED" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
      </intent-filter>
      
      <intent-filter>
        <action android:name="android.nfc.action.TAG_DISCOVERED" />
        <category android:name="android.intent.category.DEFAULT" />
      </intent-filter>
      
    </activity>
  </application>
</manifest>
```

### ตั้งค่า iOS

แก้ไข `ios/[ProjectName]/Info.plist`:

```xml
<key>NFCReaderUsageDescription</key>
<string>แอปนี้ต้องการเข้าถึง NFC เพื่ออ่านและเขียน NFC tags</string>
```

เพิ่ม NFC Entitlement ใน `ios/[ProjectName]/[ProjectName].entitlements`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.nfc.readersession.formats</key>
  <array>
    <string>NDEF</string>
    <string>TAG</string>
  </array>
</dict>
</plist>
```

---

## 2. การตั้งค่าและตรวจสอบ NFC

### utils/nfcManager.ts

```typescript
import NfcManager, { NfcTech, Ndef, NfcEvents } from 'react-native-nfc-manager';

class NFCManager {
  private static instance: NFCManager;
  private initialized = false;

  static getInstance(): NFCManager {
    if (!NFCManager.instance) {
      NFCManager.instance = new NFCManager();
    }
    return NFCManager.instance;
  }

  async initialize(): Promise<boolean> {
    if (this.initialized) return true;
    
    try {
      await NfcManager.start();
      this.initialized = true;
      return true;
    } catch (error) {
      console.error('NFC initialization error:', error);
      return false;
    }
  }

  async isSupported(): Promise<boolean> {
    return await NfcManager.isSupported();
  }

  async isEnabled(): Promise<boolean> {
    return await NfcManager.isEnabled();
  }

  async cancelRequest(): Promise<void> {
    await NfcManager.cancelTechnologyRequest();
  }

  getManager() {
    return NfcManager;
  }
}

export default NFCManager;
```

---

## 3. การอ่าน NFC Tags

### hooks/useNFCReader.ts

```typescript
import { useState, useCallback } from 'react';
import NfcManager, { NfcTech, Ndef } from 'react-native-nfc-manager';

export type NFCTagType = 'NDEF' | 'MifareClassic' | 'MifareUltralight' | 'Unknown';

export interface NFCTag {
  id: string;
  type: NFCTagType;
  techTypes: string[];
  ndefRecords?: NDEFRecord[];
}

export interface NDEFRecord {
  type: string;
  payload: string;
  language?: string;
  mimeType?: string;
}

export const useNFCReader = () => {
  const [isReading, setIsReading] = useState(false);
  const [lastTag, setLastTag] = useState<NFCTag | null>(null);
  const [error, setError] = useState<string | null>(null);

  const readTag = useCallback(async (): Promise<NFCTag | null> => {
    setIsReading(true);
    setError(null);
    
    try {
      // ขอเทคโนโลยี NDEF
      await NfcManager.requestTechnology(NfcTech.Ndef);
      
      // อ่าน tag
      const tag = await NfcManager.getTag();
      
      if (!tag) {
        throw new Error('ไม่พบ NFC tag');
      }

      const nfcTag: NFCTag = {
        id: tag.id || '',
        type: 'NDEF',
        techTypes: tag.techTypes || [],
        ndefRecords: [],
      };

      // อ่าน NDEF records
      if (tag.ndefMessage && tag.ndefMessage.length > 0) {
        nfcTag.ndefRecords = tag.ndefMessage.map(record => {
          const decoded = decodeNDEFRecord(record);
          return decoded;
        });
      }

      setLastTag(nfcTag);
      return nfcTag;
      
    } catch (ex: any) {
      if (ex.message !== 'UserCancel') {
        setError(ex.message || 'เกิดข้อผิดพลาดในการอ่าน');
      }
      return null;
    } finally {
      setIsReading(false);
      await NfcManager.cancelTechnologyRequest();
    }
  }, []);

  const decodeNDEFRecord = (record: any): NDEFRecord => {
    const tnf = record.tnf;
    const type = record.type 
      ? String.fromCharCode(...record.type) 
      : '';
    
    // Text Record (TNF=1, Type="T")
    if (tnf === 1 && type === 'T') {
      const payload = record.payload;
      const encoding = payload[0] & 0x80 ? 'UTF-16' : 'UTF-8';
      const languageLength = payload[0] & 0x3F;
      const language = String.fromCharCode(...payload.slice(1, 1 + languageLength));
      const text = encoding === 'UTF-8'
        ? String.fromCharCode(...payload.slice(1 + languageLength))
        : new TextDecoder('utf-16').decode(new Uint8Array(payload.slice(1 + languageLength)));
      
      return {
        type: 'text',
        payload: text,
        language,
      };
    }
    
    // URI Record (TNF=1, Type="U")
    if (tnf === 1 && type === 'U') {
      const payload = record.payload;
      const prefixCode = payload[0];
      const uriPrefixes = [
        '', 'http://www.', 'https://www.', 'http://', 'https://',
        'tel:', 'mailto:', 'ftp://anonymous:anonymous@', 'ftp://ftp.',
        'ftps://', 'sftp://', 'smb://', 'nfs://', 'ftp://', 'dav://',
        'news:', 'telnet://', 'imap:', 'rtsp://', 'urn:', 'pop:',
        'sip:', 'sips:', 'tftp:', 'btspp://', 'btl2cap://', 'btgoep://',
        'tcpobex://', 'irdaobex://', 'file://', 'urn:epc:id:', 'urn:epc:tag:',
        'urn:epc:pat:', 'urn:epc:raw:', 'urn:epc:', 'urn:nfc:',
      ];
      
      const prefix = uriPrefixes[prefixCode] || '';
      const uri = prefix + String.fromCharCode(...payload.slice(1));
      
      return {
        type: 'uri',
        payload: uri,
      };
    }
    
    // MIME Type Record (TNF=2)
    if (tnf === 2) {
      const mimeType = String.fromCharCode(...record.type);
      const payloadStr = String.fromCharCode(...record.payload);
      return {
        type: 'mime',
        payload: payloadStr,
        mimeType,
      };
    }
    
    return {
      type: 'unknown',
      payload: JSON.stringify(record),
    };
  };

  return {
    isReading,
    lastTag,
    error,
    readTag,
  };
};
```

---

## 4. การเขียน NFC Tags

### hooks/useNFCWriter.ts

```typescript
import { useState, useCallback } from 'react';
import NfcManager, { NfcTech, Ndef } from 'react-native-nfc-manager';

export type NFCWriteType = 'text' | 'url' | 'contact' | 'wifi' | 'custom';

export interface NFCWriteData {
  type: NFCWriteType;
  content: string;
  language?: string;
}

export const useNFCWriter = () => {
  const [isWriting, setIsWriting] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState(false);

  const writeTag = useCallback(async (data: NFCWriteData): Promise<boolean> => {
    setIsWriting(true);
    setError(null);
    setSuccess(false);
    
    try {
      await NfcManager.requestTechnology(NfcTech.Ndef);
      
      let bytes: number[];
      
      switch (data.type) {
        case 'text':
          bytes = Ndef.encodeMessage([
            Ndef.textRecord(data.content, data.language || 'th')
          ]);
          break;
          
        case 'url':
          bytes = Ndef.encodeMessage([
            Ndef.uriRecord(data.content)
          ]);
          break;
          
        case 'contact':
          // vCard format
          const vcard = formatVCard(data.content);
          bytes = Ndef.encodeMessage([
            Ndef.mimeMediaRecord('text/vcard', vcard)
          ]);
          break;
          
        case 'wifi':
          // WiFi credential format
          bytes = Ndef.encodeMessage([
            Ndef.wifiSimpleRecord(data.content)
          ]);
          break;
          
        case 'custom':
          // JSON data
          bytes = Ndef.encodeMessage([
            Ndef.mimeMediaRecord('application/json', data.content)
          ]);
          break;
          
        default:
          throw new Error('ประเภทข้อมูลไม่รองรับ');
      }
      
      await NfcManager.ndefHandler.writeNdefMessage(bytes);
      setSuccess(true);
      return true;
      
    } catch (ex: any) {
      if (ex.message !== 'UserCancel') {
        setError(ex.message || 'เกิดข้อผิดพลาดในการเขียน');
      }
      return false;
    } finally {
      setIsWriting(false);
      await NfcManager.cancelTechnologyRequest();
    }
  }, []);

  const formatVCard = (jsonContact: string): string => {
    try {
      const contact = JSON.parse(jsonContact);
      return [
        'BEGIN:VCARD',
        'VERSION:3.0',
        `FN:${contact.fullName || ''}`,
        `N:${contact.lastName || ''};${contact.firstName || ''};;;`,
        contact.phone ? `TEL;TYPE=CELL:${contact.phone}` : '',
        contact.email ? `EMAIL:${contact.email}` : '',
        contact.website ? `URL:${contact.website}` : '',
        contact.organization ? `ORG:${contact.organization}` : '',
        'END:VCARD',
      ].filter(Boolean).join('\n');
    } catch {
      return jsonContact;
    }
  };

  const writeProtectedTag = useCallback(async (
    data: NFCWriteData,
    password: number[]
  ): Promise<boolean> => {
    setIsWriting(true);
    
    try {
      await NfcManager.requestTechnology(NfcTech.MifareUltralight);
      
      // เปิด password protection
      // Note: ขึ้นอยู่กับ tag type
      await NfcManager.mifareUltralightHandlerAndroid.mifareUltralightWritePage(
        43, // PASSWORD page
        password
      );
      
      return true;
    } catch (ex: any) {
      setError(ex.message);
      return false;
    } finally {
      setIsWriting(false);
      await NfcManager.cancelTechnologyRequest();
    }
  }, []);

  return {
    isWriting,
    error,
    success,
    writeTag,
    writeProtectedTag,
  };
};
```

---

## 5. Workshop: NFC Tag Reader/Writer App

### screens/NFCScreen.tsx

```typescript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  TextInput,
  Alert,
  Modal,
  Platform,
  Animated,
} from 'react-native';
import NfcManager from 'react-native-nfc-manager';
import { useNFCReader } from '../hooks/useNFCReader';
import { useNFCWriter } from '../hooks/useNFCWriter';
import NFCManagerUtil from '../utils/nfcManager';

type TabType = 'read' | 'write';
type WriteType = 'text' | 'url' | 'contact' | 'wifi';

const NFCScreen: React.FC = () => {
  const [activeTab, setActiveTab] = useState<TabType>('read');
  const [isNFCEnabled, setIsNFCEnabled] = useState(false);
  const [writeType, setWriteType] = useState<WriteType>('text');
  const [writeContent, setWriteContent] = useState('');
  const [showWriteModal, setShowWriteModal] = useState(false);
  const scanAnimation = new Animated.Value(0);

  const { isReading, lastTag, error: readError, readTag } = useNFCReader();
  const { isWriting, error: writeError, success: writeSuccess, writeTag } = useNFCWriter();

  useEffect(() => {
    initNFC();
    return () => {
      NfcManager.cancelTechnologyRequest().catch(() => {});
    };
  }, []);

  const initNFC = async () => {
    const manager = NFCManagerUtil.getInstance();
    const supported = await manager.isSupported();
    
    if (!supported) {
      Alert.alert('ไม่รองรับ', 'อุปกรณ์ของคุณไม่รองรับ NFC');
      return;
    }
    
    await manager.initialize();
    const enabled = await manager.isEnabled();
    setIsNFCEnabled(enabled);
  };

  // Animation สำหรับ scanning indicator
  useEffect(() => {
    if (isReading || isWriting) {
      Animated.loop(
        Animated.sequence([
          Animated.timing(scanAnimation, {
            toValue: 1,
            duration: 1000,
            useNativeDriver: true,
          }),
          Animated.timing(scanAnimation, {
            toValue: 0,
            duration: 1000,
            useNativeDriver: true,
          }),
        ])
      ).start();
    } else {
      scanAnimation.stopAnimation();
    }
  }, [isReading, isWriting]);

  const handleRead = async () => {
    if (!isNFCEnabled) {
      Alert.alert('NFC ปิดอยู่', 'กรุณาเปิด NFC ในการตั้งค่า');
      return;
    }
    await readTag();
  };

  const handleWrite = async () => {
    if (!writeContent.trim()) {
      Alert.alert('ข้อผิดพลาด', 'กรุณากรอกข้อมูล');
      return;
    }
    
    if (!isNFCEnabled) {
      Alert.alert('NFC ปิดอยู่', 'กรุณาเปิด NFC ในการตั้งค่า');
      return;
    }

    const result = await writeTag({
      type: writeType,
      content: writeContent,
    });

    if (result) {
      Alert.alert('สำเร็จ', 'เขียนข้อมูลลง NFC tag เรียบร้อย');
      setWriteContent('');
    }
  };

  const renderTagInfo = () => {
    if (!lastTag) return null;
    
    return (
      <View style={styles.tagInfoCard}>
        <Text style={styles.tagInfoTitle}>ข้อมูล NFC Tag</Text>
        
        <View style={styles.tagField}>
          <Text style={styles.fieldLabel}>Tag ID:</Text>
          <Text style={styles.fieldValue}>{lastTag.id}</Text>
        </View>
        
        <View style={styles.tagField}>
          <Text style={styles.fieldLabel}>ประเภท:</Text>
          <Text style={styles.fieldValue}>{lastTag.type}</Text>
        </View>

        <View style={styles.tagField}>
          <Text style={styles.fieldLabel}>เทคโนโลยี:</Text>
          <Text style={styles.fieldValue}>
            {lastTag.techTypes.join(', ')}
          </Text>
        </View>
        
        {lastTag.ndefRecords && lastTag.ndefRecords.length > 0 && (
          <View>
            <Text style={styles.recordsTitle}>
              NDEF Records ({lastTag.ndefRecords.length})
            </Text>
            {lastTag.ndefRecords.map((record, index) => (
              <View key={index} style={styles.recordItem}>
                <View style={styles.recordHeader}>
                  <Text style={styles.recordType}>
                    {record.type.toUpperCase()}
                  </Text>
                  {record.language && (
                    <Text style={styles.recordLang}>lang: {record.language}</Text>
                  )}
                </View>
                <Text style={styles.recordPayload}>{record.payload}</Text>
              </View>
            ))}
          </View>
        )}
      </View>
    );
  };

  const renderWriteForm = () => {
    return (
      <View style={styles.writeForm}>
        <Text style={styles.formTitle}>เลือกประเภทข้อมูล</Text>
        
        <View style={styles.writeTypeRow}>
          {(['text', 'url', 'contact', 'wifi'] as WriteType[]).map(type => (
            <TouchableOpacity
              key={type}
              style={[
                styles.typeButton,
                writeType === type && styles.typeButtonActive
              ]}
              onPress={() => setWriteType(type)}
            >
              <Text style={[
                styles.typeButtonText,
                writeType === type && styles.typeButtonTextActive
              ]}>
                {getTypeIcon(type)} {getTypeName(type)}
              </Text>
            </TouchableOpacity>
          ))}
        </View>

        {writeType === 'contact' ? (
          <ContactForm onSubmit={setWriteContent} />
        ) : (
          <TextInput
            style={styles.input}
            value={writeContent}
            onChangeText={setWriteContent}
            placeholder={getPlaceholder(writeType)}
            multiline={writeType === 'text'}
            numberOfLines={writeType === 'text' ? 4 : 1}
            autoCapitalize={writeType === 'url' ? 'none' : 'sentences'}
            keyboardType={writeType === 'url' ? 'url' : 'default'}
          />
        )}

        <TouchableOpacity
          style={[styles.writeButton, isWriting && styles.writeButtonDisabled]}
          onPress={handleWrite}
          disabled={isWriting}
        >
          <Text style={styles.writeButtonText}>
            {isWriting ? 'กำลังเขียน...' : 'เขียนลง NFC Tag'}
          </Text>
        </TouchableOpacity>
      </View>
    );
  };

  const getTypeIcon = (type: WriteType) => {
    switch (type) {
      case 'text': return '📝';
      case 'url': return '🔗';
      case 'contact': return '👤';
      case 'wifi': return '📶';
    }
  };

  const getTypeName = (type: WriteType) => {
    switch (type) {
      case 'text': return 'ข้อความ';
      case 'url': return 'URL';
      case 'contact': return 'ผู้ติดต่อ';
      case 'wifi': return 'WiFi';
    }
  };

  const getPlaceholder = (type: WriteType) => {
    switch (type) {
      case 'text': return 'พิมพ์ข้อความที่ต้องการบันทึก...';
      case 'url': return 'https://example.com';
      case 'wifi': return 'SSID:Password (เช่น MyWifi:12345678)';
      default: return '';
    }
  };

  return (
    <ScrollView style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>NFC Manager</Text>
        <View style={styles.nfcStatus}>
          <View style={[
            styles.statusIndicator, 
            { backgroundColor: isNFCEnabled ? '#4CAF50' : '#f44336' }
          ]} />
          <Text style={styles.statusText}>
            NFC {isNFCEnabled ? 'เปิดใช้งาน' : 'ปิดใช้งาน'}
          </Text>
        </View>
      </View>

      {/* Tabs */}
      <View style={styles.tabs}>
        <TouchableOpacity
          style={[styles.tab, activeTab === 'read' && styles.tabActive]}
          onPress={() => setActiveTab('read')}
        >
          <Text style={[styles.tabText, activeTab === 'read' && styles.tabTextActive]}>
            📱 อ่าน Tag
          </Text>
        </TouchableOpacity>
        <TouchableOpacity
          style={[styles.tab, activeTab === 'write' && styles.tabActive]}
          onPress={() => setActiveTab('write')}
        >
          <Text style={[styles.tabText, activeTab === 'write' && styles.tabTextActive]}>
            ✍️ เขียน Tag
          </Text>
        </TouchableOpacity>
      </View>

      {/* Content */}
      {activeTab === 'read' ? (
        <View style={styles.content}>
          {/* Scan Animation */}
          <Animated.View
            style={[
              styles.nfcIcon,
              {
                opacity: scanAnimation.interpolate({
                  inputRange: [0, 1],
                  outputRange: [1, 0.3],
                }),
              },
            ]}
          >
            <Text style={styles.nfcIconText}>📡</Text>
          </Animated.View>
          
          <TouchableOpacity
            style={[styles.scanButton, isReading && styles.scanButtonActive]}
            onPress={handleRead}
            disabled={isReading}
          >
            <Text style={styles.scanButtonText}>
              {isReading ? 'กำลังรอ NFC Tag...' : 'แตะ NFC Tag'}
            </Text>
          </TouchableOpacity>

          {isReading && (
            <Text style={styles.hint}>
              นำ NFC tag มาแตะที่ด้านหลังโทรศัพท์
            </Text>
          )}
          
          {readError && (
            <View style={styles.errorBox}>
              <Text style={styles.errorText}>{readError}</Text>
            </View>
          )}
          
          {renderTagInfo()}
        </View>
      ) : (
        <View style={styles.content}>
          {writeError && (
            <View style={styles.errorBox}>
              <Text style={styles.errorText}>{writeError}</Text>
            </View>
          )}
          
          {renderWriteForm()}
          
          {isWriting && (
            <View style={styles.writingOverlay}>
              <Text style={styles.writingText}>
                📡 นำ NFC tag มาแตะที่โทรศัพท์
              </Text>
            </View>
          )}
        </View>
      )}
    </ScrollView>
  );
};

// Contact Form Component
const ContactForm: React.FC<{ onSubmit: (json: string) => void }> = ({ onSubmit }) => {
  const [contact, setContact] = useState({
    firstName: '',
    lastName: '',
    phone: '',
    email: '',
    organization: '',
  });

  useEffect(() => {
    onSubmit(JSON.stringify(contact));
  }, [contact]);

  return (
    <View>
      <TextInput
        style={styles.input}
        value={contact.firstName}
        onChangeText={(text) => setContact(prev => ({ ...prev, firstName: text }))}
        placeholder="ชื่อ"
      />
      <TextInput
        style={styles.input}
        value={contact.lastName}
        onChangeText={(text) => setContact(prev => ({ ...prev, lastName: text }))}
        placeholder="นามสกุล"
      />
      <TextInput
        style={styles.input}
        value={contact.phone}
        onChangeText={(text) => setContact(prev => ({ ...prev, phone: text }))}
        placeholder="เบอร์โทรศัพท์"
        keyboardType="phone-pad"
      />
      <TextInput
        style={styles.input}
        value={contact.email}
        onChangeText={(text) => setContact(prev => ({ ...prev, email: text }))}
        placeholder="อีเมล"
        keyboardType="email-address"
      />
      <TextInput
        style={styles.input}
        value={contact.organization}
        onChangeText={(text) => setContact(prev => ({ ...prev, organization: text }))}
        placeholder="องค์กร/บริษัท"
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: '#1a237e',
    padding: 20,
    paddingTop: 40,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  headerTitle: {
    fontSize: 24,
    fontWeight: 'bold',
    color: 'white',
  },
  nfcStatus: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  statusIndicator: {
    width: 10,
    height: 10,
    borderRadius: 5,
    marginRight: 6,
  },
  statusText: {
    color: 'white',
    fontSize: 12,
  },
  tabs: {
    flexDirection: 'row',
    backgroundColor: 'white',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  tab: {
    flex: 1,
    padding: 16,
    alignItems: 'center',
  },
  tabActive: {
    borderBottomWidth: 3,
    borderBottomColor: '#1a237e',
  },
  tabText: {
    fontSize: 14,
    color: '#999',
    fontWeight: '500',
  },
  tabTextActive: {
    color: '#1a237e',
    fontWeight: '700',
  },
  content: {
    padding: 16,
  },
  nfcIcon: {
    alignItems: 'center',
    padding: 40,
  },
  nfcIconText: {
    fontSize: 80,
  },
  scanButton: {
    backgroundColor: '#1a237e',
    padding: 20,
    borderRadius: 50,
    alignItems: 'center',
    marginBottom: 16,
  },
  scanButtonActive: {
    backgroundColor: '#3949ab',
  },
  scanButtonText: {
    color: 'white',
    fontSize: 18,
    fontWeight: '600',
  },
  hint: {
    textAlign: 'center',
    color: '#666',
    marginBottom: 16,
  },
  errorBox: {
    backgroundColor: '#ffebee',
    padding: 12,
    borderRadius: 8,
    marginBottom: 16,
  },
  errorText: {
    color: '#c62828',
  },
  tagInfoCard: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  tagInfoTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#1a237e',
    marginBottom: 12,
  },
  tagField: {
    flexDirection: 'row',
    paddingVertical: 6,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  fieldLabel: {
    width: 80,
    fontSize: 14,
    color: '#999',
    fontWeight: '500',
  },
  fieldValue: {
    flex: 1,
    fontSize: 14,
    color: '#333',
  },
  recordsTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginTop: 12,
    marginBottom: 8,
  },
  recordItem: {
    backgroundColor: '#f8f9fa',
    borderRadius: 8,
    padding: 12,
    marginBottom: 8,
  },
  recordHeader: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 6,
  },
  recordType: {
    backgroundColor: '#1a237e',
    color: 'white',
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 4,
    fontSize: 12,
    fontWeight: '600',
  },
  recordLang: {
    fontSize: 12,
    color: '#999',
    marginLeft: 8,
  },
  recordPayload: {
    fontSize: 14,
    color: '#333',
  },
  writeForm: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  formTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginBottom: 12,
  },
  writeTypeRow: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,
    marginBottom: 16,
  },
  typeButton: {
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: '#ddd',
    backgroundColor: 'white',
  },
  typeButtonActive: {
    backgroundColor: '#1a237e',
    borderColor: '#1a237e',
  },
  typeButtonText: {
    fontSize: 13,
    color: '#666',
  },
  typeButtonTextActive: {
    color: 'white',
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    marginBottom: 12,
    fontSize: 14,
    color: '#333',
    backgroundColor: '#fafafa',
  },
  writeButton: {
    backgroundColor: '#1a237e',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 8,
  },
  writeButtonDisabled: {
    backgroundColor: '#9e9e9e',
  },
  writeButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  writingOverlay: {
    marginTop: 16,
    backgroundColor: '#e8eaf6',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  writingText: {
    fontSize: 16,
    color: '#1a237e',
    fontWeight: '500',
  },
});

export default NFCScreen;
```

---

## 6. NFC Payments Integration

### NFC Payment ผ่าน HCE (Host Card Emulation)

```typescript
import NfcManager, { NfcTech, IsoDep } from 'react-native-nfc-manager';

class NFCPayment {
  // APDU Commands สำหรับ payment
  private static SELECT_AID = [
    0x00, 0xA4, 0x04, 0x00, 0x07,
    0xA0, 0x00, 0x00, 0x00, 0x04, 0x10, 0x10,
    0x00
  ];

  static async readPaymentCard(): Promise<any> {
    try {
      await NfcManager.requestTechnology(NfcTech.IsoDep);
      
      // Select AID (Application Identifier)
      const selectResponse = await NfcManager.isoDepHandler.transceive(
        NFCPayment.SELECT_AID
      );
      
      if (!NFCPayment.isSuccess(selectResponse)) {
        throw new Error('ไม่สามารถเลือก application ได้');
      }
      
      // Get Processing Options (GPO)
      const gpoResponse = await NfcManager.isoDepHandler.transceive([
        0x80, 0xA8, 0x00, 0x00, 0x02, 0x83, 0x00, 0x00
      ]);
      
      // Read card data
      const cardData = NFCPayment.parseGPOResponse(gpoResponse);
      
      return cardData;
      
    } catch (error) {
      console.error('NFC payment error:', error);
      throw error;
    } finally {
      await NfcManager.cancelTechnologyRequest();
    }
  }

  private static isSuccess(response: number[]): boolean {
    const len = response.length;
    return len >= 2 && response[len - 2] === 0x90 && response[len - 1] === 0x00;
  }

  private static parseGPOResponse(response: number[]): any {
    // Parse TLV (Tag-Length-Value) format
    const result: Record<string, string> = {};
    let i = 0;
    
    while (i < response.length - 2) {
      const tag = response[i].toString(16).padStart(2, '0');
      const length = response[i + 1];
      const value = response.slice(i + 2, i + 2 + length);
      result[tag] = value.map(b => b.toString(16).padStart(2, '0')).join('');
      i += 2 + length;
    }
    
    return result;
  }
}
```

---

## Tips and Best Practices

### 1. การจัดการ Lifecycle

```typescript
useEffect(() => {
  NfcManager.start().catch(console.error);
  
  return () => {
    // Cleanup เสมอ
    NfcManager.cancelTechnologyRequest().catch(console.error);
    NfcManager.unregisterTagEvent().catch(console.error);
  };
}, []);
```

### 2. Error Handling

```typescript
const readNFCTag = async () => {
  try {
    await NfcManager.requestTechnology(NfcTech.Ndef);
    const tag = await NfcManager.getTag();
    // process tag
  } catch (error: any) {
    switch (error.message) {
      case 'UserCancel':
        // ผู้ใช้ยกเลิก - ไม่แสดง error
        break;
      case 'NfcNotEnabled':
        Alert.alert('กรุณาเปิด NFC');
        break;
      default:
        Alert.alert('Error', error.message);
    }
  } finally {
    await NfcManager.cancelTechnologyRequest();
  }
};
```

### 3. Foreground Dispatch (Android)

```typescript
// ให้ app รับ NFC intent เมื่อ app อยู่ foreground
NfcManager.registerTagEvent(callback);

// ตรวจสอบ intent ที่ส่งมาเมื่อ app start
useEffect(() => {
  const handleIntent = async () => {
    const tag = await NfcManager.getLaunchTagEvent();
    if (tag) {
      processTag(tag);
    }
  };
  handleIntent();
}, []);
```

---

## Workshop Exercises

### แบบฝึกหัดที่ 1: NFC Business Card
สร้าง app ที่ให้ผู้ใช้บันทึกข้อมูลนามบัตรของตัวเอง และเขียนลง NFC tag เพื่อแชร์กับผู้อื่น

### แบบฝึกหัดที่ 2: NFC Attendance System
สร้างระบบลงเวลาเข้างาน โดยให้พนักงานแตะ NFC badge ที่โทรศัพท์ของ admin

### แบบฝึกหัดที่ 3: NFC Smart Home
สร้าง NFC tags สำหรับบ้าน เช่น แตะ tag ที่หน้าประตูแล้วปิด WiFi, เปิด alarm

### แบบฝึกหัดที่ 4: NFC Product Authentication
สร้าง app ตรวจสอบของแท้ โดยอ่าน NFC tag จากสินค้าและ verify กับ server

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **การติดตั้ง** react-native-nfc-manager
2. **การอ่าน NFC tags** - รับข้อมูลประเภทต่างๆ
3. **การเขียน NFC tags** - text, URL, contact, WiFi
4. **NFC Payments** - การทำงานกับ payment cards
5. **Workshop** - สร้าง NFC Reader/Writer app

> **Tips:** NFC ทำงานได้เฉพาะบนอุปกรณ์จริง ไม่สามารถทดสอบบน Simulator/Emulator ได้
