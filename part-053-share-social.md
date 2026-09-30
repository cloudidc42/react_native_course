# Part 053: Share และ Social Features ใน React Native

## Share API พื้นฐาน

React Native มี built-in `Share` API สำหรับการแชร์ content ไปยัง apps อื่น

```typescript
import React from 'react';
import { View, Text, TouchableOpacity, Share, Alert, StyleSheet } from 'react-native';

const BasicShareDemo: React.FC = () => {
  const shareText = async () => {
    try {
      const result = await Share.share({
        message: 'ลองดูแอปนี้สิ! มันเจ๋งมาก 🚀',
        title: 'แชร์แอปของฉัน', // Android เท่านั้น
      });

      if (result.action === Share.sharedAction) {
        if (result.activityType) {
          // iOS: ผู้ใช้แชร์ผ่าน activity type นี้
          console.log('Shared via:', result.activityType);
        } else {
          console.log('Shared successfully');
        }
      } else if (result.action === Share.dismissedAction) {
        // iOS: ผู้ใช้ยกเลิก
        console.log('Share dismissed');
      }
    } catch (error: any) {
      Alert.alert('Error', error.message);
    }
  };

  const shareURL = async () => {
    try {
      await Share.share({
        message: 'ดูสินค้าสุดเจ๋ง https://example.com/products/123',
        url: 'https://example.com/products/123', // iOS เท่านั้น
        title: 'สินค้าแนะนำ',
      });
    } catch (error: any) {
      Alert.alert('Error', error.message);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Share API Demo</Text>
      
      <TouchableOpacity style={styles.button} onPress={shareText}>
        <Text style={styles.buttonText}>แชร์ข้อความ</Text>
      </TouchableOpacity>
      
      <TouchableOpacity style={[styles.button, styles.urlButton]} onPress={shareURL}>
        <Text style={styles.buttonText}>แชร์ URL</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#f5f5f5' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20, color: '#333' },
  button: {
    backgroundColor: '#2196F3',
    padding: 15,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 15,
  },
  urlButton: { backgroundColor: '#4CAF50' },
  buttonText: { color: 'white', fontSize: 16, fontWeight: 'bold' },
});

export default BasicShareDemo;
```

---

## react-native-share

Library นี้ให้ความสามารถ share ที่มากกว่า built-in Share API

### การติดตั้ง

```bash
npm install react-native-share
cd ios && pod install
```

### Android - AndroidManifest.xml

```xml
<application ...>
  <!-- สำหรับ Android 7+ ต้องใช้ FileProvider -->
  <provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.provider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
      android:name="android.support.FILE_PROVIDER_PATHS"
      android:resource="@xml/provider_paths" />
  </provider>
</application>
```

สร้าง `android/app/src/main/res/xml/provider_paths.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
  <external-path name="external" path="." />
  <external-files-path name="external_files" path="." />
  <cache-path name="cache" path="." />
  <external-cache-path name="external_cache" path="." />
  <files-path name="files" path="." />
</paths>
```

### ตัวอย่างการใช้งาน react-native-share

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Image,
  ScrollView,
} from 'react-native';
import Share, { ShareSingleOptions, Social } from 'react-native-share';

const AdvancedShareDemo: React.FC = () => {
  const [shareResult, setShareResult] = useState<string>('');

  // แชร์ไฟล์รูปภาพ
  const shareImage = async () => {
    try {
      const options = {
        url: 'file:///path/to/image.jpg',
        // หรือ base64
        // url: `data:image/jpeg;base64,${base64Image}`,
        type: 'image/jpeg',
        title: 'แชร์รูปภาพ',
        message: 'ดูรูปสวยๆ นี้!',
        subject: 'รูปภาพจากแอป',
      };

      const result = await Share.open(options);
      setShareResult(`แชร์สำเร็จ: ${result.app}`);
    } catch (error: any) {
      if (error.message !== 'User did not share') {
        Alert.alert('Error', error.message);
      }
      setShareResult('ยกเลิกการแชร์');
    }
  };

  // แชร์ไปยัง Facebook เท่านั้น
  const shareToFacebook = async () => {
    try {
      const options: ShareSingleOptions = {
        message: 'เช็คอินจากแอปของฉัน! 📍',
        url: 'https://example.com',
        social: Social.Facebook,
      };

      await Share.shareSingle(options);
      setShareResult('แชร์ไป Facebook สำเร็จ');
    } catch (error: any) {
      if (error.message !== 'User did not share') {
        // Facebook อาจไม่ได้ติดตั้ง
        Alert.alert('ไม่สามารถแชร์', 'Facebook ไม่ได้ติดตั้งหรือไม่รองรับ');
      }
    }
  };

  // แชร์ไปยัง Instagram Stories
  const shareToInstagramStories = async () => {
    try {
      const options: ShareSingleOptions = {
        backgroundImage: 'file:///path/to/background.jpg',
        stickerImage: 'file:///path/to/sticker.png',
        backgroundBottomColor: '#000000',
        backgroundTopColor: '#ffffff',
        attributionURL: 'https://example.com',
        social: Social.InstagramStories,
      };

      await Share.shareSingle(options);
    } catch (error: any) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถแชร์ไป Instagram Stories ได้');
    }
  };

  // แชร์ไปยัง WhatsApp
  const shareToWhatsApp = async () => {
    try {
      const options: ShareSingleOptions = {
        message: 'สวัสดีจากแอปของฉัน! 👋',
        url: 'https://example.com',
        social: Social.Whatsapp,
      };

      await Share.shareSingle(options);
    } catch (error: any) {
      Alert.alert('ข้อผิดพลาด', 'WhatsApp ไม่ได้ติดตั้ง');
    }
  };

  // แชร์หลายไฟล์
  const shareMultipleFiles = async () => {
    try {
      const options = {
        urls: [
          'file:///path/to/image1.jpg',
          'file:///path/to/image2.jpg',
          'file:///path/to/document.pdf',
        ],
        type: ['image/jpeg', 'image/jpeg', 'application/pdf'],
        message: 'ส่งไฟล์ให้คุณ',
      };

      await Share.open(options);
    } catch (error: any) {
      if (error.message !== 'User did not share') {
        Alert.alert('Error', error.message);
      }
    }
  };

  // ตรวจสอบว่า App ติดตั้งอยู่หรือไม่ (Android เท่านั้น)
  const checkAppInstalled = async (appPackage: string) => {
    try {
      const isInstalled = await Share.isPackageInstalled(appPackage);
      Alert.alert(
        'ผลการตรวจสอบ',
        `${appPackage}: ${isInstalled.isInstalled ? 'ติดตั้งแล้ว' : 'ไม่ได้ติดตั้ง'}`
      );
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <ScrollView style={advStyles.container}>
      <Text style={advStyles.title}>Advanced Share Demo</Text>
      
      {shareResult ? (
        <View style={advStyles.resultBox}>
          <Text style={advStyles.resultText}>{shareResult}</Text>
        </View>
      ) : null}

      <Text style={advStyles.sectionTitle}>แชร์เนื้อหา</Text>
      
      <TouchableOpacity style={advStyles.shareButton} onPress={shareImage}>
        <Text style={advStyles.shareIcon}>🖼️</Text>
        <Text style={advStyles.shareLabel}>แชร์รูปภาพ</Text>
      </TouchableOpacity>
      
      <TouchableOpacity style={advStyles.shareButton} onPress={shareMultipleFiles}>
        <Text style={advStyles.shareIcon}>📁</Text>
        <Text style={advStyles.shareLabel}>แชร์หลายไฟล์</Text>
      </TouchableOpacity>

      <Text style={advStyles.sectionTitle}>แชร์ไปยัง Social Media</Text>

      <TouchableOpacity
        style={[advStyles.socialButton, advStyles.facebookButton]}
        onPress={shareToFacebook}
      >
        <Text style={advStyles.socialButtonText}>📘 Facebook</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={[advStyles.socialButton, advStyles.instagramButton]}
        onPress={shareToInstagramStories}
      >
        <Text style={advStyles.socialButtonText}>📷 Instagram Stories</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={[advStyles.socialButton, advStyles.whatsappButton]}
        onPress={shareToWhatsApp}
      >
        <Text style={advStyles.socialButtonText}>💬 WhatsApp</Text>
      </TouchableOpacity>

      <Text style={advStyles.sectionTitle}>ตรวจสอบ Apps</Text>
      
      <TouchableOpacity
        style={advStyles.checkButton}
        onPress={() => checkAppInstalled('com.facebook.katana')}
      >
        <Text style={advStyles.checkButtonText}>ตรวจสอบ Facebook</Text>
      </TouchableOpacity>
    </ScrollView>
  );
};

const advStyles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#f5f5f5' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20, color: '#333' },
  resultBox: {
    backgroundColor: '#E8F5E9',
    padding: 12,
    borderRadius: 8,
    marginBottom: 15,
    borderLeftWidth: 3,
    borderLeftColor: '#4CAF50',
  },
  resultText: { color: '#2E7D32', fontSize: 14 },
  sectionTitle: { fontSize: 18, fontWeight: '600', color: '#444', marginBottom: 12, marginTop: 10 },
  shareButton: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 15,
    borderRadius: 10,
    marginBottom: 10,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  shareIcon: { fontSize: 24, marginRight: 12 },
  shareLabel: { fontSize: 16, color: '#333' },
  socialButton: {
    padding: 15,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 10,
  },
  facebookButton: { backgroundColor: '#1877F2' },
  instagramButton: { backgroundColor: '#E1306C' },
  whatsappButton: { backgroundColor: '#25D366' },
  socialButtonText: { color: 'white', fontSize: 16, fontWeight: 'bold' },
  checkButton: {
    backgroundColor: '#9E9E9E',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 10,
  },
  checkButtonText: { color: 'white', fontSize: 14 },
});

export default AdvancedShareDemo;
```

---

## Clipboard API

การคัดลอกข้อมูลไปยัง clipboard

### การติดตั้ง

```bash
npm install @react-native-clipboard/clipboard
cd ios && pod install
```

### การใช้งาน

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  FlatList,
} from 'react-native';
import Clipboard from '@react-native-clipboard/clipboard';

const ClipboardDemo: React.FC = () => {
  const [inputText, setInputText] = useState('');
  const [clipboardContent, setClipboardContent] = useState('');
  const [copyHistory, setCopyHistory] = useState<string[]>([]);

  const copyToClipboard = async (text: string) => {
    Clipboard.setString(text);
    setCopyHistory(prev => [text, ...prev.slice(0, 9)]);
    Alert.alert('คัดลอกแล้ว', `"${text}" คัดลอกไปยัง clipboard แล้ว`);
  };

  const pasteFromClipboard = async () => {
    try {
      const content = await Clipboard.getString();
      setClipboardContent(content);
      setInputText(content);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถอ่าน clipboard ได้');
    }
  };

  const checkClipboard = async () => {
    try {
      const hasContent = await Clipboard.hasString();
      const content = hasContent ? await Clipboard.getString() : 'ว่างเปล่า';
      Alert.alert('เนื้อหา Clipboard', content || 'ไม่มีข้อความ');
    } catch (error) {
      console.error(error);
    }
  };

  // ข้อมูลตัวอย่างที่คัดลอกได้บ่อย
  const quickCopyItems = [
    { label: 'เบอร์โทร', value: '0812345678' },
    { label: 'อีเมล', value: 'example@email.com' },
    { label: 'ที่อยู่', value: '123 ถ.สุขุมวิท กรุงเทพฯ 10110' },
    { label: 'บัญชีธนาคาร', value: '123-4-56789-0' },
  ];

  return (
    <View style={clipStyles.container}>
      <Text style={clipStyles.title}>Clipboard Demo</Text>

      <View style={clipStyles.inputSection}>
        <TextInput
          style={clipStyles.input}
          value={inputText}
          onChangeText={setInputText}
          placeholder="พิมพ์ข้อความหรือวางจาก clipboard..."
          multiline
        />
        <View style={clipStyles.buttonRow}>
          <TouchableOpacity
            style={clipStyles.copyBtn}
            onPress={() => copyToClipboard(inputText)}
          >
            <Text style={clipStyles.btnText}>คัดลอก</Text>
          </TouchableOpacity>
          <TouchableOpacity
            style={clipStyles.pasteBtn}
            onPress={pasteFromClipboard}
          >
            <Text style={clipStyles.btnText}>วาง</Text>
          </TouchableOpacity>
          <TouchableOpacity
            style={clipStyles.checkBtn}
            onPress={checkClipboard}
          >
            <Text style={clipStyles.btnText}>ตรวจสอบ</Text>
          </TouchableOpacity>
        </View>
      </View>

      <Text style={clipStyles.sectionTitle}>คัดลอกด่วน</Text>
      {quickCopyItems.map((item, index) => (
        <TouchableOpacity
          key={index}
          style={clipStyles.quickItem}
          onPress={() => copyToClipboard(item.value)}
        >
          <View>
            <Text style={clipStyles.quickLabel}>{item.label}</Text>
            <Text style={clipStyles.quickValue}>{item.value}</Text>
          </View>
          <Text style={clipStyles.copyIcon}>📋</Text>
        </TouchableOpacity>
      ))}

      {copyHistory.length > 0 && (
        <>
          <Text style={clipStyles.sectionTitle}>ประวัติการคัดลอก</Text>
          {copyHistory.map((item, index) => (
            <TouchableOpacity
              key={index}
              style={clipStyles.historyItem}
              onPress={() => copyToClipboard(item)}
            >
              <Text style={clipStyles.historyText} numberOfLines={1}>{item}</Text>
            </TouchableOpacity>
          ))}
        </>
      )}
    </View>
  );
};

const clipStyles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#f5f5f5' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20, color: '#333' },
  inputSection: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 15,
    marginBottom: 20,
    elevation: 2,
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 10,
    minHeight: 80,
    fontSize: 15,
    textAlignVertical: 'top',
    marginBottom: 10,
  },
  buttonRow: { flexDirection: 'row', gap: 8 },
  copyBtn: { flex: 1, backgroundColor: '#2196F3', padding: 10, borderRadius: 8, alignItems: 'center' },
  pasteBtn: { flex: 1, backgroundColor: '#4CAF50', padding: 10, borderRadius: 8, alignItems: 'center' },
  checkBtn: { flex: 1, backgroundColor: '#FF9800', padding: 10, borderRadius: 8, alignItems: 'center' },
  btnText: { color: 'white', fontWeight: 'bold', fontSize: 14 },
  sectionTitle: { fontSize: 18, fontWeight: '600', color: '#444', marginBottom: 12, marginTop: 5 },
  quickItem: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 10,
    marginBottom: 8,
    elevation: 1,
  },
  quickLabel: { fontSize: 12, color: '#999', marginBottom: 3 },
  quickValue: { fontSize: 15, color: '#333', fontWeight: '500' },
  copyIcon: { fontSize: 20 },
  historyItem: {
    backgroundColor: '#FFF8E1',
    padding: 12,
    borderRadius: 8,
    marginBottom: 5,
  },
  historyText: { fontSize: 14, color: '#555' },
});

export default ClipboardDemo;
```

---

## Contacts

การเข้าถึง Contacts ของ device

### การติดตั้ง

```bash
npm install react-native-contacts
cd ios && pod install
```

### Permission

**iOS - Info.plist:**
```xml
<key>NSContactsUsageDescription</key>
<string>แอปต้องการเข้าถึงรายชื่อเพื่อให้คุณสามารถแชร์และนำเข้าข้อมูลติดต่อ</string>
```

**Android - AndroidManifest.xml:**
```xml
<uses-permission android:name="android.permission.READ_CONTACTS" />
<uses-permission android:name="android.permission.WRITE_CONTACTS" />
```

### การใช้งาน Contacts

```typescript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  StyleSheet,
  Alert,
  PermissionsAndroid,
  Platform,
  TextInput,
  Image,
} from 'react-native';
import Contacts from 'react-native-contacts';

interface Contact {
  recordID: string;
  displayName: string;
  phoneNumbers: Array<{ label: string; number: string }>;
  emailAddresses: Array<{ label: string; email: string }>;
  thumbnailPath?: string;
}

const ContactsDemo: React.FC = () => {
  const [contacts, setContacts] = useState<Contact[]>([]);
  const [filteredContacts, setFilteredContacts] = useState<Contact[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [loading, setLoading] = useState(false);

  const requestPermission = async (): Promise<boolean> => {
    if (Platform.OS === 'ios') {
      const permission = await Contacts.requestPermission();
      return permission === 'authorized';
    }

    const granted = await PermissionsAndroid.request(
      PermissionsAndroid.PERMISSIONS.READ_CONTACTS
    );
    return granted === PermissionsAndroid.RESULTS.GRANTED;
  };

  const loadContacts = async () => {
    setLoading(true);
    
    const hasPermission = await requestPermission();
    if (!hasPermission) {
      Alert.alert('Permission Denied', 'ต้องการ permission เพื่อเข้าถึงรายชื่อ');
      setLoading(false);
      return;
    }

    try {
      const allContacts = await Contacts.getAll();
      const sorted = allContacts.sort((a, b) =>
        (a.displayName || '').localeCompare(b.displayName || '', 'th')
      );
      setContacts(sorted);
      setFilteredContacts(sorted);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถโหลดรายชื่อได้');
    } finally {
      setLoading(false);
    }
  };

  const searchContacts = (query: string) => {
    setSearchQuery(query);
    if (!query.trim()) {
      setFilteredContacts(contacts);
      return;
    }

    const filtered = contacts.filter(contact => {
      const name = contact.displayName?.toLowerCase() || '';
      const phone = contact.phoneNumbers.map(p => p.number).join('');
      return (
        name.includes(query.toLowerCase()) ||
        phone.includes(query)
      );
    });
    setFilteredContacts(filtered);
  };

  const addContact = async () => {
    const newContact = {
      displayName: 'John Doe',
      phoneNumbers: [{ label: 'mobile', number: '0812345678' }],
      emailAddresses: [{ label: 'work', email: 'john@example.com' }],
    };

    try {
      await Contacts.addContact(newContact);
      Alert.alert('สำเร็จ', 'เพิ่มรายชื่อแล้ว');
      loadContacts();
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถเพิ่มรายชื่อได้');
    }
  };

  const shareContact = async (contact: Contact) => {
    const { Share } = require('react-native');
    const phoneText = contact.phoneNumbers
      .map(p => `${p.label}: ${p.number}`)
      .join('\n');
    
    await Share.share({
      message: `ติดต่อ: ${contact.displayName}\n${phoneText}`,
      title: 'แชร์รายชื่อ',
    });
  };

  useEffect(() => {
    loadContacts();
  }, []);

  const renderContact = ({ item }: { item: Contact }) => (
    <TouchableOpacity
      style={contStyles.contactItem}
      onPress={() => shareContact(item)}
    >
      <View style={contStyles.avatar}>
        {item.thumbnailPath ? (
          <Image source={{ uri: item.thumbnailPath }} style={contStyles.avatarImage} />
        ) : (
          <Text style={contStyles.avatarText}>
            {(item.displayName || '?')[0].toUpperCase()}
          </Text>
        )}
      </View>
      <View style={contStyles.contactInfo}>
        <Text style={contStyles.contactName}>{item.displayName || 'ไม่มีชื่อ'}</Text>
        {item.phoneNumbers.length > 0 && (
          <Text style={contStyles.contactPhone}>
            {item.phoneNumbers[0].number}
          </Text>
        )}
      </View>
      <Text style={contStyles.shareIcon}>📤</Text>
    </TouchableOpacity>
  );

  return (
    <View style={contStyles.container}>
      <Text style={contStyles.title}>Contacts ({contacts.length})</Text>
      
      <TextInput
        style={contStyles.searchInput}
        value={searchQuery}
        onChangeText={searchContacts}
        placeholder="ค้นหารายชื่อ..."
      />

      <TouchableOpacity style={contStyles.addButton} onPress={addContact}>
        <Text style={contStyles.addButtonText}>+ เพิ่มรายชื่อใหม่</Text>
      </TouchableOpacity>

      {loading ? (
        <Text style={contStyles.loadingText}>กำลังโหลด...</Text>
      ) : (
        <FlatList
          data={filteredContacts}
          keyExtractor={(item) => item.recordID}
          renderItem={renderContact}
          ListEmptyComponent={
            <Text style={contStyles.emptyText}>ไม่พบรายชื่อ</Text>
          }
        />
      )}
    </View>
  );
};

const contStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: { fontSize: 22, fontWeight: 'bold', padding: 20, paddingBottom: 10, color: '#333' },
  searchInput: {
    margin: 15,
    marginTop: 5,
    backgroundColor: 'white',
    borderRadius: 10,
    padding: 12,
    fontSize: 15,
    elevation: 2,
  },
  addButton: {
    marginHorizontal: 15,
    marginBottom: 10,
    backgroundColor: '#4CAF50',
    padding: 12,
    borderRadius: 10,
    alignItems: 'center',
  },
  addButtonText: { color: 'white', fontWeight: 'bold', fontSize: 15 },
  loadingText: { textAlign: 'center', color: '#999', marginTop: 30, fontSize: 16 },
  contactItem: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    paddingVertical: 12,
    paddingHorizontal: 15,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  avatar: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  avatarImage: { width: 44, height: 44, borderRadius: 22 },
  avatarText: { color: 'white', fontSize: 18, fontWeight: 'bold' },
  contactInfo: { flex: 1 },
  contactName: { fontSize: 15, fontWeight: '500', color: '#333' },
  contactPhone: { fontSize: 13, color: '#888', marginTop: 2 },
  shareIcon: { fontSize: 18, marginLeft: 10 },
  emptyText: { textAlign: 'center', color: '#999', fontSize: 16, marginTop: 30 },
});

export default ContactsDemo;
```

---

## Workshop: Share Feature

สร้างระบบ Share ที่สมบูรณ์สำหรับ e-commerce app

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Modal,
  Share,
  Alert,
  Dimensions,
  Image,
  ScrollView,
} from 'react-native';
import Clipboard from '@react-native-clipboard/clipboard';

const { width } = Dimensions.get('window');

interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  imageUrl: string;
  shareUrl: string;
}

interface ShareOption {
  id: string;
  icon: string;
  label: string;
  color: string;
}

const shareOptions: ShareOption[] = [
  { id: 'general', icon: '📤', label: 'แชร์ทั่วไป', color: '#2196F3' },
  { id: 'copy', icon: '📋', label: 'คัดลอกลิงก์', color: '#607D8B' },
  { id: 'message', icon: '💬', label: 'ข้อความ', color: '#4CAF50' },
  { id: 'line', icon: '💚', label: 'LINE', color: '#00B900' },
  { id: 'facebook', icon: '📘', label: 'Facebook', color: '#1877F2' },
  { id: 'twitter', icon: '🐦', label: 'Twitter/X', color: '#000000' },
];

const ShareFeatureWorkshop: React.FC = () => {
  const [showShareModal, setShowShareModal] = useState(false);
  const [selectedProduct, setSelectedProduct] = useState<Product | null>(null);
  const [shareCount, setShareCount] = useState<Record<string, number>>({});

  const products: Product[] = [
    {
      id: '1',
      name: 'Nike Air Max 270',
      price: 4290,
      description: 'รองเท้าผ้าใบสไตล์สปอร์ตแบรนด์ Nike รุ่น Air Max',
      imageUrl: 'https://example.com/shoes.jpg',
      shareUrl: 'https://myshop.com/products/1',
    },
    {
      id: '2',
      name: 'Samsung Galaxy S24',
      price: 29900,
      description: 'สมาร์ทโฟน Android รุ่นใหม่ล่าสุดจาก Samsung',
      imageUrl: 'https://example.com/phone.jpg',
      shareUrl: 'https://myshop.com/products/2',
    },
  ];

  const handleShare = async (option: ShareOption, product: Product) => {
    setShowShareModal(false);
    
    // อัปเดต share count
    setShareCount(prev => ({
      ...prev,
      [product.id]: (prev[product.id] || 0) + 1,
    }));

    const shareText = `${product.name} - ฿${product.price.toLocaleString()}\n${product.description}\n\nซื้อได้ที่: ${product.shareUrl}`;

    switch (option.id) {
      case 'general':
        await Share.share({
          message: shareText,
          url: product.shareUrl,
          title: product.name,
        });
        break;

      case 'copy':
        Clipboard.setString(product.shareUrl);
        Alert.alert('คัดลอกแล้ว', 'ลิงก์ถูกคัดลอกไปยัง clipboard แล้ว');
        break;

      case 'line':
        // เปิด LINE เพื่อแชร์
        const lineUrl = `line://msg/text/${encodeURIComponent(shareText)}`;
        const { Linking } = require('react-native');
        try {
          await Linking.openURL(lineUrl);
        } catch {
          Alert.alert('LINE ไม่ได้ติดตั้ง', 'กรุณาติดตั้ง LINE ก่อน');
        }
        break;

      default:
        await Share.share({ message: shareText });
    }
  };

  const openShareModal = (product: Product) => {
    setSelectedProduct(product);
    setShowShareModal(true);
  };

  return (
    <View style={shareStyles.container}>
      <Text style={shareStyles.title}>Share Feature Workshop</Text>

      <ScrollView>
        {products.map(product => (
          <View key={product.id} style={shareStyles.productCard}>
            <View style={shareStyles.productImagePlaceholder}>
              <Text style={shareStyles.productImageText}>📦</Text>
            </View>
            
            <View style={shareStyles.productDetails}>
              <Text style={shareStyles.productName}>{product.name}</Text>
              <Text style={shareStyles.productPrice}>
                ฿{product.price.toLocaleString()}
              </Text>
              <Text style={shareStyles.productDesc} numberOfLines={2}>
                {product.description}
              </Text>
              
              <View style={shareStyles.productFooter}>
                <Text style={shareStyles.shareCountText}>
                  แชร์แล้ว {shareCount[product.id] || 0} ครั้ง
                </Text>
                <TouchableOpacity
                  style={shareStyles.shareButton}
                  onPress={() => openShareModal(product)}
                >
                  <Text style={shareStyles.shareButtonText}>📤 แชร์</Text>
                </TouchableOpacity>
              </View>
            </View>
          </View>
        ))}
      </ScrollView>

      {/* Share Modal */}
      <Modal
        visible={showShareModal}
        transparent
        animationType="slide"
        onRequestClose={() => setShowShareModal(false)}
      >
        <TouchableOpacity
          style={shareStyles.modalOverlay}
          activeOpacity={1}
          onPress={() => setShowShareModal(false)}
        >
          <View style={shareStyles.modalContent}>
            <View style={shareStyles.modalHandle} />
            
            <Text style={shareStyles.modalTitle}>แชร์สินค้า</Text>
            
            {selectedProduct && (
              <View style={shareStyles.previewCard}>
                <Text style={shareStyles.previewName}>{selectedProduct.name}</Text>
                <Text style={shareStyles.previewPrice}>
                  ฿{selectedProduct.price.toLocaleString()}
                </Text>
                <Text style={shareStyles.previewUrl} numberOfLines={1}>
                  {selectedProduct.shareUrl}
                </Text>
              </View>
            )}

            <View style={shareStyles.shareGrid}>
              {shareOptions.map(option => (
                <TouchableOpacity
                  key={option.id}
                  style={shareStyles.shareOption}
                  onPress={() => selectedProduct && handleShare(option, selectedProduct)}
                >
                  <View style={[shareStyles.shareOptionIcon, { backgroundColor: option.color }]}>
                    <Text style={shareStyles.shareOptionEmoji}>{option.icon}</Text>
                  </View>
                  <Text style={shareStyles.shareOptionLabel}>{option.label}</Text>
                </TouchableOpacity>
              ))}
            </View>

            <TouchableOpacity
              style={shareStyles.cancelButton}
              onPress={() => setShowShareModal(false)}
            >
              <Text style={shareStyles.cancelButtonText}>ยกเลิก</Text>
            </TouchableOpacity>
          </View>
        </TouchableOpacity>
      </Modal>
    </View>
  );
};

const shareStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: { fontSize: 22, fontWeight: 'bold', padding: 20, paddingBottom: 15, color: '#333' },
  productCard: {
    backgroundColor: 'white',
    margin: 15,
    marginTop: 0,
    borderRadius: 12,
    padding: 15,
    flexDirection: 'row',
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  productImagePlaceholder: {
    width: 80,
    height: 80,
    backgroundColor: '#f0f0f0',
    borderRadius: 10,
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 15,
  },
  productImageText: { fontSize: 36 },
  productDetails: { flex: 1 },
  productName: { fontSize: 16, fontWeight: 'bold', color: '#333', marginBottom: 4 },
  productPrice: { fontSize: 18, fontWeight: 'bold', color: '#2196F3', marginBottom: 6 },
  productDesc: { fontSize: 13, color: '#666', marginBottom: 10 },
  productFooter: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },
  shareCountText: { fontSize: 12, color: '#999' },
  shareButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
  },
  shareButtonText: { color: 'white', fontWeight: 'bold', fontSize: 14 },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'flex-end',
  },
  modalContent: {
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingBottom: 30,
  },
  modalHandle: {
    width: 40,
    height: 4,
    backgroundColor: '#ddd',
    borderRadius: 2,
    alignSelf: 'center',
    marginTop: 12,
    marginBottom: 16,
  },
  modalTitle: { fontSize: 18, fontWeight: 'bold', textAlign: 'center', marginBottom: 15, color: '#333' },
  previewCard: {
    backgroundColor: '#F5F5F5',
    margin: 15,
    marginTop: 0,
    padding: 12,
    borderRadius: 10,
  },
  previewName: { fontSize: 15, fontWeight: '600', color: '#333' },
  previewPrice: { fontSize: 16, fontWeight: 'bold', color: '#2196F3', marginTop: 3 },
  previewUrl: { fontSize: 12, color: '#888', marginTop: 5 },
  shareGrid: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    padding: 10,
    justifyContent: 'center',
  },
  shareOption: {
    width: (width - 60) / 3,
    alignItems: 'center',
    padding: 10,
    marginBottom: 10,
  },
  shareOptionIcon: {
    width: 54,
    height: 54,
    borderRadius: 27,
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 6,
  },
  shareOptionEmoji: { fontSize: 24 },
  shareOptionLabel: { fontSize: 12, color: '#555', textAlign: 'center' },
  cancelButton: {
    margin: 15,
    marginTop: 5,
    backgroundColor: '#f0f0f0',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  cancelButtonText: { fontSize: 16, color: '#666', fontWeight: '500' },
});

export default ShareFeatureWorkshop;
```

---

## Tips และ Best Practices

### 1. Share tracking
```typescript
// ติดตาม share events เพื่อ analytics
const trackShare = async (platform: string, contentId: string) => {
  await analytics().logEvent('share', {
    method: platform,
    content_type: 'product',
    item_id: contentId,
  });
};
```

### 2. Dynamic Share Content
```typescript
const generateShareContent = (product: Product, discount?: number) => {
  const message = discount
    ? `${product.name} ลด ${discount}%! เพียง ฿${Math.floor(product.price * (1 - discount/100)).toLocaleString()}`
    : `${product.name} ฿${product.price.toLocaleString()}`;
    
  return {
    message,
    url: product.shareUrl,
  };
};
```

---

## สรุป

Share และ Social Features ช่วยให้แอปมี:
- **Built-in Share**: ง่าย รองรับทุก platform
- **react-native-share**: ควบคุมได้มากขึ้น แชร์ไฟล์ได้
- **Clipboard**: คัดลอก/วางข้อมูล
- **Contacts**: เข้าถึงรายชื่อ device

สิ่งสำคัญ: ขอ Permission ก่อนเสมอและอธิบายให้ผู้ใช้เข้าใจว่าทำไมถึงต้องการ
