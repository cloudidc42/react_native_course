# Part 021: Modal, Alert, ActionSheet

## บทนำ

ในการพัฒนา React Native application การแสดงข้อมูลหรือการรับ input จากผู้ใช้ในรูปแบบ popup เป็นสิ่งที่จำเป็นมาก ในบทนี้เราจะเรียนรู้เกี่ยวกับ Modal, Alert, และ ActionSheet ซึ่งเป็น component และ API ที่ใช้สำหรับการแสดงข้อมูลในลักษณะ overlay หรือ dialog

## สารบัญ

1. Modal Component พื้นฐาน
2. Animated Modal
3. Alert.alert()
4. Alert with Input
5. ActionSheet (iOS/Android)
6. Bottom Sheet
7. Workshop: Shopping Cart Modal

---

## 1. Modal Component พื้นฐาน

`Modal` เป็น component ใน React Native ที่ใช้แสดงเนื้อหาบน layer ด้านบนของ screen หลัก

### Props หลักของ Modal

| Prop | Type | ความหมาย |
|------|------|-----------|
| visible | boolean | แสดง/ซ่อน modal |
| animationType | 'none' \| 'slide' \| 'fade' | รูปแบบ animation |
| transparent | boolean | พื้นหลังโปร่งใส |
| onRequestClose | function | callback เมื่อกด back (Android) |
| onShow | function | callback เมื่อ modal แสดง |
| onDismiss | function | callback เมื่อ modal ปิด (iOS) |
| presentationStyle | string | รูปแบบการแสดงผล (iOS) |

### ตัวอย่างพื้นฐาน

```jsx
import React, { useState } from 'react';
import {
  View,
  Text,
  Modal,
  TouchableOpacity,
  StyleSheet,
} from 'react-native';

const BasicModal = () => {
  const [modalVisible, setModalVisible] = useState(false);

  return (
    <View style={styles.container}>
      {/* ปุ่มเปิด Modal */}
      <TouchableOpacity
        style={styles.button}
        onPress={() => setModalVisible(true)}
      >
        <Text style={styles.buttonText}>เปิด Modal</Text>
      </TouchableOpacity>

      {/* Modal Component */}
      <Modal
        visible={modalVisible}
        animationType="slide"
        transparent={true}
        onRequestClose={() => setModalVisible(false)}
      >
        <View style={styles.overlay}>
          <View style={styles.modalContainer}>
            <Text style={styles.title}>Modal Title</Text>
            <Text style={styles.body}>
              นี่คือเนื้อหาใน Modal ที่แสดงอยู่บน screen หลัก
            </Text>
            
            <TouchableOpacity
              style={styles.closeButton}
              onPress={() => setModalVisible(false)}
            >
              <Text style={styles.closeText}>ปิด</Text>
            </TouchableOpacity>
          </View>
        </View>
      </Modal>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  modalContainer: {
    backgroundColor: '#fff',
    borderRadius: 16,
    padding: 24,
    width: '80%',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 10,
  },
  title: {
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 12,
    color: '#1a1a1a',
  },
  body: {
    fontSize: 16,
    color: '#666',
    lineHeight: 24,
    marginBottom: 20,
  },
  closeButton: {
    backgroundColor: '#007AFF',
    paddingVertical: 10,
    borderRadius: 8,
    alignItems: 'center',
  },
  closeText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default BasicModal;
```

### Modal แบบ Full Screen

```jsx
const FullScreenModal = () => {
  const [visible, setVisible] = useState(false);

  return (
    <View style={{ flex: 1 }}>
      <TouchableOpacity onPress={() => setVisible(true)}>
        <Text>เปิด Full Screen Modal</Text>
      </TouchableOpacity>

      <Modal
        visible={visible}
        animationType="slide"
        presentationStyle="fullScreen" // iOS only
        onRequestClose={() => setVisible(false)}
      >
        <View style={{ flex: 1, backgroundColor: '#fff' }}>
          {/* Header */}
          <View style={{
            flexDirection: 'row',
            alignItems: 'center',
            padding: 16,
            borderBottomWidth: 1,
            borderBottomColor: '#e0e0e0',
          }}>
            <TouchableOpacity onPress={() => setVisible(false)}>
              <Text style={{ color: '#007AFF', fontSize: 16 }}>ปิด</Text>
            </TouchableOpacity>
            <Text style={{ flex: 1, textAlign: 'center', fontSize: 18, fontWeight: 'bold' }}>
              หน้าเต็ม
            </Text>
          </View>

          {/* Content */}
          <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
            <Text>เนื้อหาในหน้าเต็ม</Text>
          </View>
        </View>
      </Modal>
    </View>
  );
};
```

---

## 2. Animated Modal

การใช้ Animation กับ Modal ทำให้ประสบการณ์การใช้งานดีขึ้น

### Modal พร้อม Custom Animation

```jsx
import React, { useState, useRef, useEffect } from 'react';
import {
  View,
  Text,
  Modal,
  TouchableOpacity,
  Animated,
  StyleSheet,
  Dimensions,
} from 'react-native';

const { height: SCREEN_HEIGHT } = Dimensions.get('window');

const AnimatedModal = () => {
  const [visible, setVisible] = useState(false);
  const slideAnim = useRef(new Animated.Value(SCREEN_HEIGHT)).current;
  const fadeAnim = useRef(new Animated.Value(0)).current;

  const openModal = () => {
    setVisible(true);
    // เริ่ม animation พร้อมกัน
    Animated.parallel([
      Animated.timing(slideAnim, {
        toValue: 0,
        duration: 300,
        useNativeDriver: true,
      }),
      Animated.timing(fadeAnim, {
        toValue: 1,
        duration: 300,
        useNativeDriver: true,
      }),
    ]).start();
  };

  const closeModal = () => {
    // Animation ก่อนปิด
    Animated.parallel([
      Animated.timing(slideAnim, {
        toValue: SCREEN_HEIGHT,
        duration: 250,
        useNativeDriver: true,
      }),
      Animated.timing(fadeAnim, {
        toValue: 0,
        duration: 250,
        useNativeDriver: true,
      }),
    ]).start(() => {
      setVisible(false);
      slideAnim.setValue(SCREEN_HEIGHT);
    });
  };

  return (
    <View style={styles.container}>
      <TouchableOpacity style={styles.button} onPress={openModal}>
        <Text style={styles.buttonText}>เปิด Animated Modal</Text>
      </TouchableOpacity>

      <Modal visible={visible} transparent animationType="none">
        {/* Overlay พร้อม fade animation */}
        <Animated.View
          style={[styles.overlay, { opacity: fadeAnim }]}
        >
          <TouchableOpacity
            style={{ flex: 1 }}
            activeOpacity={1}
            onPress={closeModal}
          />
        </Animated.View>

        {/* Modal content พร้อม slide animation */}
        <Animated.View
          style={[
            styles.bottomSheet,
            { transform: [{ translateY: slideAnim }] },
          ]}
        >
          {/* Handle bar */}
          <View style={styles.handle} />
          
          <Text style={styles.title}>เลือกตัวเลือก</Text>
          
          {['ตัวเลือก 1', 'ตัวเลือก 2', 'ตัวเลือก 3'].map((item, index) => (
            <TouchableOpacity
              key={index}
              style={styles.option}
              onPress={() => {
                console.log(`เลือก: ${item}`);
                closeModal();
              }}
            >
              <Text style={styles.optionText}>{item}</Text>
            </TouchableOpacity>
          ))}

          <TouchableOpacity
            style={[styles.option, styles.cancelOption]}
            onPress={closeModal}
          >
            <Text style={[styles.optionText, { color: '#FF3B30' }]}>ยกเลิก</Text>
          </TouchableOpacity>
        </Animated.View>
      </Modal>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  overlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
  },
  bottomSheet: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    backgroundColor: '#fff',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingBottom: 34, // สำหรับ iPhone X+
    paddingTop: 12,
  },
  handle: {
    width: 40,
    height: 4,
    backgroundColor: '#e0e0e0',
    borderRadius: 2,
    alignSelf: 'center',
    marginBottom: 16,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  option: {
    paddingVertical: 16,
    paddingHorizontal: 24,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  cancelOption: {
    marginTop: 8,
    borderTopWidth: 8,
    borderTopColor: '#f0f0f0',
    borderBottomWidth: 0,
  },
  optionText: {
    fontSize: 16,
    color: '#007AFF',
  },
});

export default AnimatedModal;
```

### Scale Animation Modal

```jsx
const ScaleModal = () => {
  const [visible, setVisible] = useState(false);
  const scaleAnim = useRef(new Animated.Value(0)).current;
  const opacityAnim = useRef(new Animated.Value(0)).current;

  const show = () => {
    setVisible(true);
    Animated.spring(scaleAnim, {
      toValue: 1,
      tension: 65,
      friction: 8,
      useNativeDriver: true,
    }).start();
    Animated.timing(opacityAnim, {
      toValue: 1,
      duration: 200,
      useNativeDriver: true,
    }).start();
  };

  const hide = () => {
    Animated.parallel([
      Animated.timing(scaleAnim, {
        toValue: 0,
        duration: 200,
        useNativeDriver: true,
      }),
      Animated.timing(opacityAnim, {
        toValue: 0,
        duration: 200,
        useNativeDriver: true,
      }),
    ]).start(() => {
      setVisible(false);
      scaleAnim.setValue(0);
    });
  };

  return (
    <>
      <TouchableOpacity onPress={show}>
        <Text>แสดง Scale Modal</Text>
      </TouchableOpacity>

      <Modal visible={visible} transparent animationType="none">
        <Animated.View
          style={{
            flex: 1,
            backgroundColor: 'rgba(0,0,0,0.5)',
            justifyContent: 'center',
            alignItems: 'center',
            opacity: opacityAnim,
          }}
        >
          <Animated.View
            style={{
              backgroundColor: '#fff',
              borderRadius: 16,
              padding: 24,
              width: '80%',
              transform: [{ scale: scaleAnim }],
            }}
          >
            <Text style={{ fontSize: 20, fontWeight: 'bold', marginBottom: 12 }}>
              ยืนยันการลบ
            </Text>
            <Text style={{ color: '#666', marginBottom: 20 }}>
              คุณแน่ใจหรือไม่ว่าต้องการลบรายการนี้?
            </Text>
            <View style={{ flexDirection: 'row', gap: 12 }}>
              <TouchableOpacity
                onPress={hide}
                style={{
                  flex: 1,
                  padding: 12,
                  borderRadius: 8,
                  borderWidth: 1,
                  borderColor: '#e0e0e0',
                  alignItems: 'center',
                }}
              >
                <Text>ยกเลิก</Text>
              </TouchableOpacity>
              <TouchableOpacity
                onPress={() => { /* ลบ */ hide(); }}
                style={{
                  flex: 1,
                  padding: 12,
                  borderRadius: 8,
                  backgroundColor: '#FF3B30',
                  alignItems: 'center',
                }}
              >
                <Text style={{ color: '#fff' }}>ลบ</Text>
              </TouchableOpacity>
            </View>
          </Animated.View>
        </Animated.View>
      </Modal>
    </>
  );
};
```

---

## 3. Alert.alert()

`Alert` เป็น API ใน React Native สำหรับแสดง native dialog บนทั้ง iOS และ Android

### รูปแบบการใช้งาน

```jsx
Alert.alert(
  'Title',           // หัวข้อ
  'Message',         // ข้อความ
  [                  // array ของ buttons
    {
      text: 'ยกเลิก',
      style: 'cancel',    // 'default' | 'cancel' | 'destructive'
      onPress: () => {},
    },
    {
      text: 'ตกลง',
      onPress: () => {},
    },
  ],
  {
    cancelable: true,  // Android: ปิดได้โดยการกดนอก dialog
  }
);
```

### ตัวอย่างการใช้งาน Alert แบบต่างๆ

```jsx
import React from 'react';
import { View, TouchableOpacity, Text, Alert, StyleSheet } from 'react-native';

const AlertExamples = () => {
  // 1. Alert พื้นฐาน (1 ปุ่ม)
  const showSimpleAlert = () => {
    Alert.alert('แจ้งเตือน', 'ดำเนินการสำเร็จแล้ว!');
  };

  // 2. Alert ยืนยัน (2 ปุ่ม)
  const showConfirmAlert = () => {
    Alert.alert(
      'ยืนยันการลบ',
      'คุณต้องการลบรายการนี้หรือไม่?',
      [
        {
          text: 'ยกเลิก',
          style: 'cancel',
          onPress: () => console.log('ยกเลิก'),
        },
        {
          text: 'ลบ',
          style: 'destructive',
          onPress: () => console.log('ลบแล้ว'),
        },
      ]
    );
  };

  // 3. Alert หลายปุ่ม (3 ปุ่ม)
  const showMultiButtonAlert = () => {
    Alert.alert(
      'บันทึกไฟล์',
      'ต้องการบันทึกการเปลี่ยนแปลงก่อนออกหรือไม่?',
      [
        {
          text: 'ไม่บันทึก',
          style: 'destructive',
          onPress: () => console.log('ไม่บันทึก'),
        },
        {
          text: 'ยกเลิก',
          style: 'cancel',
        },
        {
          text: 'บันทึก',
          onPress: () => console.log('บันทึกแล้ว'),
        },
      ]
    );
  };

  // 4. Alert ด้วย error
  const showErrorAlert = () => {
    Alert.alert(
      '❌ เกิดข้อผิดพลาด',
      'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้ กรุณาตรวจสอบการเชื่อมต่ออินเทอร์เน็ต',
      [
        { text: 'ลองใหม่', onPress: () => console.log('retry') },
        { text: 'ตกลง', style: 'cancel' },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <TouchableOpacity style={styles.btn} onPress={showSimpleAlert}>
        <Text style={styles.btnText}>Simple Alert</Text>
      </TouchableOpacity>

      <TouchableOpacity style={[styles.btn, styles.danger]} onPress={showConfirmAlert}>
        <Text style={styles.btnText}>Confirm Alert</Text>
      </TouchableOpacity>

      <TouchableOpacity style={[styles.btn, styles.warning]} onPress={showMultiButtonAlert}>
        <Text style={styles.btnText}>Multi Button Alert</Text>
      </TouchableOpacity>

      <TouchableOpacity style={[styles.btn, styles.error]} onPress={showErrorAlert}>
        <Text style={styles.btnText}>Error Alert</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, gap: 12 },
  btn: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  btnText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  danger: { backgroundColor: '#FF3B30' },
  warning: { backgroundColor: '#FF9500' },
  error: { backgroundColor: '#FF3B30' },
});
```

---

## 4. Alert with Input (iOS)

บน iOS สามารถใช้ `Alert.prompt()` เพื่อรับ input จากผู้ใช้

```jsx
import { Alert, Platform } from 'react-native';

// iOS only - Alert with text input
const showInputAlert = () => {
  if (Platform.OS === 'ios') {
    Alert.prompt(
      'ตั้งชื่อ',
      'กรุณากรอกชื่อของคุณ',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ตกลง',
          onPress: (text) => console.log(`ชื่อ: ${text}`),
        },
      ],
      'plain-text',      // 'plain-text' | 'secure-text' | 'login-password'
      'ชื่อเริ่มต้น',    // default value
      'default'          // keyboard type
    );
  }
};

// สำหรับ Android ต้องใช้ custom Modal แทน
const CustomInputModal = () => {
  const [visible, setVisible] = useState(false);
  const [inputValue, setInputValue] = useState('');

  const handleSubmit = () => {
    console.log('Input:', inputValue);
    setVisible(false);
    setInputValue('');
  };

  return (
    <>
      <TouchableOpacity onPress={() => setVisible(true)}>
        <Text>เปิด Input Dialog</Text>
      </TouchableOpacity>

      <Modal visible={visible} transparent animationType="fade">
        <View style={{
          flex: 1,
          backgroundColor: 'rgba(0,0,0,0.5)',
          justifyContent: 'center',
          alignItems: 'center',
        }}>
          <View style={{
            backgroundColor: '#fff',
            borderRadius: 12,
            padding: 20,
            width: '85%',
          }}>
            <Text style={{ fontSize: 18, fontWeight: 'bold', marginBottom: 8 }}>
              ตั้งชื่อ
            </Text>
            <Text style={{ color: '#666', marginBottom: 16 }}>
              กรุณากรอกชื่อของคุณ
            </Text>
            <TextInput
              style={{
                borderWidth: 1,
                borderColor: '#ddd',
                borderRadius: 8,
                padding: 12,
                fontSize: 16,
                marginBottom: 16,
              }}
              placeholder="ชื่อของคุณ"
              value={inputValue}
              onChangeText={setInputValue}
              autoFocus
            />
            <View style={{ flexDirection: 'row', gap: 12 }}>
              <TouchableOpacity
                onPress={() => setVisible(false)}
                style={{
                  flex: 1,
                  padding: 12,
                  borderRadius: 8,
                  borderWidth: 1,
                  borderColor: '#ddd',
                  alignItems: 'center',
                }}
              >
                <Text>ยกเลิก</Text>
              </TouchableOpacity>
              <TouchableOpacity
                onPress={handleSubmit}
                style={{
                  flex: 1,
                  padding: 12,
                  borderRadius: 8,
                  backgroundColor: '#007AFF',
                  alignItems: 'center',
                }}
              >
                <Text style={{ color: '#fff' }}>ตกลง</Text>
              </TouchableOpacity>
            </View>
          </View>
        </View>
      </Modal>
    </>
  );
};
```

---

## 5. ActionSheet (iOS/Android)

ActionSheet เป็น component ที่แสดง list ของ actions ที่ผู้ใช้สามารถเลือกได้

### ติดตั้ง @expo/react-native-action-sheet

```bash
npx expo install @expo/react-native-action-sheet
```

### การใช้งาน ActionSheet

```jsx
import React from 'react';
import { View, TouchableOpacity, Text } from 'react-native';
import { useActionSheet } from '@expo/react-native-action-sheet';

// ต้องครอบ App ด้วย ActionSheetProvider
// ใน App.js:
// import { ActionSheetProvider } from '@expo/react-native-action-sheet';
// <ActionSheetProvider>
//   <App />
// </ActionSheetProvider>

const ActionSheetExample = () => {
  const { showActionSheetWithOptions } = useActionSheet();

  const handlePress = () => {
    const options = ['ถ่ายรูป', 'เลือกจาก Gallery', 'ลบรูปปัจจุบัน', 'ยกเลิก'];
    const destructiveButtonIndex = 2;
    const cancelButtonIndex = 3;

    showActionSheetWithOptions(
      {
        options,
        cancelButtonIndex,
        destructiveButtonIndex,
        title: 'เปลี่ยนรูปโปรไฟล์',
        message: 'เลือกวิธีการอัปโหลดรูปภาพ',
      },
      (selectedIndex) => {
        switch (selectedIndex) {
          case 0:
            console.log('ถ่ายรูป');
            break;
          case 1:
            console.log('เลือกจาก Gallery');
            break;
          case destructiveButtonIndex:
            console.log('ลบรูปปัจจุบัน');
            break;
          case cancelButtonIndex:
            // ยกเลิก
            break;
        }
      }
    );
  };

  return (
    <TouchableOpacity onPress={handlePress}>
      <Text>แสดง ActionSheet</Text>
    </TouchableOpacity>
  );
};
```

### Custom ActionSheet Component (ใช้ได้ทั้ง iOS และ Android)

```jsx
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Modal,
  Animated,
  useRef,
  StyleSheet,
} from 'react-native';

const CustomActionSheet = ({ visible, onClose, options, title, message }) => {
  const slideAnim = useRef(new Animated.Value(300)).current;

  useEffect(() => {
    if (visible) {
      Animated.spring(slideAnim, {
        toValue: 0,
        useNativeDriver: true,
        tension: 50,
        friction: 7,
      }).start();
    } else {
      slideAnim.setValue(300);
    }
  }, [visible]);

  const handleClose = () => {
    Animated.timing(slideAnim, {
      toValue: 300,
      duration: 200,
      useNativeDriver: true,
    }).start(() => onClose());
  };

  return (
    <Modal visible={visible} transparent animationType="none">
      <View style={styles.overlay}>
        <TouchableOpacity style={{ flex: 1 }} onPress={handleClose} />
        
        <Animated.View
          style={[styles.sheet, { transform: [{ translateY: slideAnim }] }]}
        >
          {title && (
            <View style={styles.header}>
              <Text style={styles.title}>{title}</Text>
              {message && <Text style={styles.message}>{message}</Text>}
            </View>
          )}

          {options.map((option, index) => (
            <TouchableOpacity
              key={index}
              style={[
                styles.option,
                option.destructive && styles.destructiveOption,
                option.cancel && styles.cancelOption,
              ]}
              onPress={() => {
                option.onPress?.();
                handleClose();
              }}
            >
              {option.icon && <Text style={styles.optionIcon}>{option.icon}</Text>}
              <Text
                style={[
                  styles.optionText,
                  option.destructive && styles.destructiveText,
                  option.cancel && styles.cancelText,
                ]}
              >
                {option.text}
              </Text>
            </TouchableOpacity>
          ))}
        </Animated.View>
      </View>
    </Modal>
  );
};

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.4)',
    justifyContent: 'flex-end',
  },
  sheet: {
    backgroundColor: '#fff',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingBottom: 34,
  },
  header: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
    alignItems: 'center',
  },
  title: {
    fontSize: 16,
    fontWeight: '600',
    color: '#1a1a1a',
  },
  message: {
    fontSize: 13,
    color: '#666',
    marginTop: 4,
    textAlign: 'center',
  },
  option: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 16,
    paddingHorizontal: 24,
    borderBottomWidth: 1,
    borderBottomColor: '#f5f5f5',
  },
  optionIcon: {
    fontSize: 20,
    marginRight: 12,
  },
  optionText: {
    fontSize: 16,
    color: '#007AFF',
  },
  destructiveOption: {
    // ไม่มี style พิเศษ
  },
  destructiveText: {
    color: '#FF3B30',
  },
  cancelOption: {
    marginTop: 8,
    borderTopWidth: 8,
    borderTopColor: '#f0f0f0',
    borderBottomWidth: 0,
    justifyContent: 'center',
  },
  cancelText: {
    color: '#333',
    fontWeight: '600',
  },
});

// การใช้งาน
const App = () => {
  const [sheetVisible, setSheetVisible] = useState(false);

  const options = [
    {
      text: 'ถ่ายรูป',
      icon: '📷',
      onPress: () => console.log('ถ่ายรูป'),
    },
    {
      text: 'เลือกจาก Gallery',
      icon: '🖼️',
      onPress: () => console.log('gallery'),
    },
    {
      text: 'ลบรูปปัจจุบัน',
      icon: '🗑️',
      destructive: true,
      onPress: () => console.log('ลบ'),
    },
    {
      text: 'ยกเลิก',
      cancel: true,
    },
  ];

  return (
    <View style={{ flex: 1 }}>
      <TouchableOpacity onPress={() => setSheetVisible(true)}>
        <Text>แสดง Custom ActionSheet</Text>
      </TouchableOpacity>

      <CustomActionSheet
        visible={sheetVisible}
        onClose={() => setSheetVisible(false)}
        options={options}
        title="เปลี่ยนรูปโปรไฟล์"
        message="เลือกวิธีการอัปโหลดรูปภาพ"
      />
    </View>
  );
};
```

---

## 6. Bottom Sheet

Bottom Sheet เป็น UI pattern ที่นิยมมากสำหรับ mobile apps

### ติดตั้ง @gorhom/bottom-sheet

```bash
npx expo install @gorhom/bottom-sheet react-native-reanimated react-native-gesture-handler
```

### การใช้งาน Bottom Sheet

```jsx
import React, { useCallback, useRef } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import BottomSheet, { BottomSheetView, BottomSheetScrollView } from '@gorhom/bottom-sheet';
import { GestureHandlerRootView } from 'react-native-gesture-handler';

const BottomSheetExample = () => {
  const bottomSheetRef = useRef(null);
  
  // กำหนด snap points
  const snapPoints = ['25%', '50%', '90%'];

  const handleOpen = useCallback(() => {
    bottomSheetRef.current?.expand();
  }, []);

  const handleClose = useCallback(() => {
    bottomSheetRef.current?.close();
  }, []);

  const handleSheetChanges = useCallback((index) => {
    console.log('handleSheetChanges', index);
  }, []);

  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <View style={styles.container}>
        <TouchableOpacity style={styles.button} onPress={handleOpen}>
          <Text style={styles.buttonText}>เปิด Bottom Sheet</Text>
        </TouchableOpacity>

        <BottomSheet
          ref={bottomSheetRef}
          index={-1}
          snapPoints={snapPoints}
          onChange={handleSheetChanges}
          enablePanDownToClose={true}
          backgroundStyle={styles.sheetBackground}
          handleIndicatorStyle={styles.indicator}
        >
          <BottomSheetScrollView contentContainerStyle={styles.contentContainer}>
            <Text style={styles.sheetTitle}>ตัวกรองสินค้า</Text>
            
            {/* Categories */}
            <Text style={styles.sectionTitle}>หมวดหมู่</Text>
            {['อิเล็กทรอนิกส์', 'เสื้อผ้า', 'อาหาร', 'เครื่องสำอาง'].map((cat) => (
              <TouchableOpacity key={cat} style={styles.filterItem}>
                <Text>{cat}</Text>
              </TouchableOpacity>
            ))}

            {/* Price Range */}
            <Text style={styles.sectionTitle}>ช่วงราคา</Text>
            {['ต่ำกว่า 500 บาท', '500-1,000 บาท', '1,000-5,000 บาท', 'มากกว่า 5,000 บาท'].map((price) => (
              <TouchableOpacity key={price} style={styles.filterItem}>
                <Text>{price}</Text>
              </TouchableOpacity>
            ))}

            <TouchableOpacity style={styles.applyButton} onPress={handleClose}>
              <Text style={styles.applyText}>ใช้ตัวกรอง</Text>
            </TouchableOpacity>
          </BottomSheetScrollView>
        </BottomSheet>
      </View>
    </GestureHandlerRootView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20 },
  button: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  sheetBackground: { backgroundColor: '#fff' },
  indicator: { backgroundColor: '#ddd' },
  contentContainer: { padding: 20 },
  sheetTitle: { fontSize: 20, fontWeight: 'bold', marginBottom: 16 },
  sectionTitle: {
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
    marginTop: 16,
    marginBottom: 8,
  },
  filterItem: {
    paddingVertical: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  applyButton: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 24,
  },
  applyText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

## 7. Workshop: Shopping Cart Modal

ในส่วนนี้เราจะสร้าง Shopping Cart Modal ที่สมจริง

```jsx
import React, { useState, useRef } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Modal,
  FlatList,
  Image,
  Animated,
  StyleSheet,
  SafeAreaView,
} from 'react-native';

// ข้อมูลสินค้า
const PRODUCTS = [
  { id: '1', name: 'iPhone 15 Pro', price: 45900, image: 'https://via.placeholder.com/80' },
  { id: '2', name: 'AirPods Pro', price: 8990, image: 'https://via.placeholder.com/80' },
  { id: '3', name: 'Apple Watch', price: 15900, image: 'https://via.placeholder.com/80' },
  { id: '4', name: 'MacBook Air', price: 39900, image: 'https://via.placeholder.com/80' },
];

// Cart Item Component
const CartItem = ({ item, onUpdateQuantity, onRemove }) => {
  return (
    <View style={cartStyles.item}>
      <View style={cartStyles.itemImage}>
        <Text style={{ fontSize: 30 }}>📦</Text>
      </View>
      
      <View style={{ flex: 1, marginLeft: 12 }}>
        <Text style={cartStyles.itemName}>{item.name}</Text>
        <Text style={cartStyles.itemPrice}>
          ฿{(item.price * item.quantity).toLocaleString()}
        </Text>
        
        <View style={cartStyles.quantityRow}>
          <TouchableOpacity
            style={cartStyles.qtyBtn}
            onPress={() => onUpdateQuantity(item.id, item.quantity - 1)}
          >
            <Text style={cartStyles.qtyBtnText}>-</Text>
          </TouchableOpacity>
          
          <Text style={cartStyles.quantity}>{item.quantity}</Text>
          
          <TouchableOpacity
            style={cartStyles.qtyBtn}
            onPress={() => onUpdateQuantity(item.id, item.quantity + 1)}
          >
            <Text style={cartStyles.qtyBtnText}>+</Text>
          </TouchableOpacity>
        </View>
      </View>
      
      <TouchableOpacity onPress={() => onRemove(item.id)}>
        <Text style={{ color: '#FF3B30', fontSize: 20 }}>✕</Text>
      </TouchableOpacity>
    </View>
  );
};

// Shopping Cart Modal
const ShoppingCartModal = ({ visible, onClose, cartItems, onUpdateQuantity, onRemove, onCheckout }) => {
  const slideAnim = useRef(new Animated.Value(600)).current;

  useEffect(() => {
    if (visible) {
      Animated.spring(slideAnim, {
        toValue: 0,
        tension: 60,
        friction: 10,
        useNativeDriver: true,
      }).start();
    } else {
      slideAnim.setValue(600);
    }
  }, [visible]);

  const total = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const itemCount = cartItems.reduce((sum, item) => sum + item.quantity, 0);

  const handleCheckout = () => {
    Alert.alert(
      '✅ สั่งซื้อสำเร็จ',
      `สั่งซื้อรายการ ${itemCount} ชิ้น รวม ฿${total.toLocaleString()} เรียบร้อยแล้ว`,
      [
        {
          text: 'ตกลง',
          onPress: () => {
            onCheckout();
            onClose();
          },
        },
      ]
    );
  };

  return (
    <Modal visible={visible} transparent animationType="none">
      <View style={cartStyles.overlay}>
        <TouchableOpacity style={{ flex: 1 }} onPress={onClose} />
        
        <Animated.View
          style={[
            cartStyles.container,
            { transform: [{ translateY: slideAnim }] },
          ]}
        >
          <SafeAreaView>
            {/* Header */}
            <View style={cartStyles.header}>
              <View style={cartStyles.handle} />
              <Text style={cartStyles.headerTitle}>
                🛒 ตะกร้าสินค้า ({itemCount} ชิ้น)
              </Text>
              <TouchableOpacity onPress={onClose}>
                <Text style={{ color: '#007AFF', fontSize: 16 }}>ปิด</Text>
              </TouchableOpacity>
            </View>

            {/* Empty State */}
            {cartItems.length === 0 ? (
              <View style={cartStyles.emptyState}>
                <Text style={{ fontSize: 60 }}>🛒</Text>
                <Text style={cartStyles.emptyText}>ตะกร้าว่างเปล่า</Text>
                <Text style={cartStyles.emptySubtext}>
                  เพิ่มสินค้าเพื่อเริ่มต้นช้อปปิ้ง
                </Text>
              </View>
            ) : (
              <>
                {/* Cart Items */}
                <FlatList
                  data={cartItems}
                  keyExtractor={(item) => item.id}
                  renderItem={({ item }) => (
                    <CartItem
                      item={item}
                      onUpdateQuantity={onUpdateQuantity}
                      onRemove={onRemove}
                    />
                  )}
                  style={{ maxHeight: 350 }}
                  showsVerticalScrollIndicator={false}
                />

                {/* Summary */}
                <View style={cartStyles.summary}>
                  <View style={cartStyles.summaryRow}>
                    <Text style={cartStyles.summaryLabel}>ราคาสินค้า</Text>
                    <Text>฿{total.toLocaleString()}</Text>
                  </View>
                  <View style={cartStyles.summaryRow}>
                    <Text style={cartStyles.summaryLabel}>ค่าจัดส่ง</Text>
                    <Text style={{ color: '#34C759' }}>ฟรี</Text>
                  </View>
                  <View style={[cartStyles.summaryRow, { marginTop: 8 }]}>
                    <Text style={cartStyles.totalLabel}>ยอดรวม</Text>
                    <Text style={cartStyles.totalAmount}>฿{total.toLocaleString()}</Text>
                  </View>
                </View>

                {/* Checkout Button */}
                <TouchableOpacity
                  style={cartStyles.checkoutBtn}
                  onPress={handleCheckout}
                >
                  <Text style={cartStyles.checkoutText}>
                    สั่งซื้อ - ฿{total.toLocaleString()}
                  </Text>
                </TouchableOpacity>
              </>
            )}
          </SafeAreaView>
        </Animated.View>
      </View>
    </Modal>
  );
};

// Main App
const ShoppingApp = () => {
  const [cartVisible, setCartVisible] = useState(false);
  const [cartItems, setCartItems] = useState([]);

  const addToCart = (product) => {
    setCartItems((prev) => {
      const existing = prev.find((item) => item.id === product.id);
      if (existing) {
        return prev.map((item) =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...prev, { ...product, quantity: 1 }];
    });
  };

  const updateQuantity = (id, quantity) => {
    if (quantity <= 0) {
      removeItem(id);
      return;
    }
    setCartItems((prev) =>
      prev.map((item) => (item.id === id ? { ...item, quantity } : item))
    );
  };

  const removeItem = (id) => {
    Alert.alert('ลบสินค้า', 'ต้องการลบสินค้านี้ออกจากตะกร้า?', [
      { text: 'ยกเลิก', style: 'cancel' },
      {
        text: 'ลบ',
        style: 'destructive',
        onPress: () => setCartItems((prev) => prev.filter((item) => item.id !== id)),
      },
    ]);
  };

  const checkout = () => {
    setCartItems([]);
  };

  const totalItems = cartItems.reduce((sum, item) => sum + item.quantity, 0);

  return (
    <View style={{ flex: 1, backgroundColor: '#f5f5f5' }}>
      {/* Header */}
      <SafeAreaView style={appStyles.header}>
        <Text style={appStyles.headerTitle}>🛍️ Shop</Text>
        <TouchableOpacity
          style={appStyles.cartButton}
          onPress={() => setCartVisible(true)}
        >
          <Text style={appStyles.cartText}>🛒</Text>
          {totalItems > 0 && (
            <View style={appStyles.badge}>
              <Text style={appStyles.badgeText}>{totalItems}</Text>
            </View>
          )}
        </TouchableOpacity>
      </SafeAreaView>

      {/* Product List */}
      <FlatList
        data={PRODUCTS}
        keyExtractor={(item) => item.id}
        contentContainerStyle={{ padding: 16, gap: 12 }}
        renderItem={({ item }) => (
          <View style={appStyles.productCard}>
            <View style={appStyles.productImage}>
              <Text style={{ fontSize: 40 }}>📱</Text>
            </View>
            <View style={{ flex: 1, marginLeft: 12 }}>
              <Text style={appStyles.productName}>{item.name}</Text>
              <Text style={appStyles.productPrice}>฿{item.price.toLocaleString()}</Text>
            </View>
            <TouchableOpacity
              style={appStyles.addButton}
              onPress={() => addToCart(item)}
            >
              <Text style={appStyles.addText}>+ เพิ่ม</Text>
            </TouchableOpacity>
          </View>
        )}
      />

      <ShoppingCartModal
        visible={cartVisible}
        onClose={() => setCartVisible(false)}
        cartItems={cartItems}
        onUpdateQuantity={updateQuantity}
        onRemove={removeItem}
        onCheckout={checkout}
      />
    </View>
  );
};

const cartStyles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'flex-end',
  },
  container: {
    backgroundColor: '#fff',
    borderTopLeftRadius: 24,
    borderTopRightRadius: 24,
    maxHeight: '90%',
  },
  header: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  handle: {
    width: 40,
    height: 4,
    backgroundColor: '#ddd',
    borderRadius: 2,
    alignSelf: 'center',
    marginBottom: 12,
  },
  headerTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 4,
  },
  item: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f5f5f5',
  },
  itemImage: {
    width: 60,
    height: 60,
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  itemName: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  itemPrice: { color: '#007AFF', fontWeight: '700', marginBottom: 8 },
  quantityRow: { flexDirection: 'row', alignItems: 'center', gap: 12 },
  qtyBtn: {
    width: 28,
    height: 28,
    borderRadius: 14,
    backgroundColor: '#f0f0f0',
    justifyContent: 'center',
    alignItems: 'center',
  },
  qtyBtnText: { fontSize: 18, fontWeight: '600' },
  quantity: { fontSize: 16, fontWeight: '600', minWidth: 20, textAlign: 'center' },
  emptyState: {
    padding: 40,
    alignItems: 'center',
    gap: 8,
  },
  emptyText: { fontSize: 20, fontWeight: 'bold', color: '#333' },
  emptySubtext: { color: '#888', textAlign: 'center' },
  summary: {
    padding: 16,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    gap: 8,
  },
  summaryRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
  },
  summaryLabel: { color: '#666' },
  totalLabel: { fontSize: 18, fontWeight: 'bold' },
  totalAmount: { fontSize: 18, fontWeight: 'bold', color: '#007AFF' },
  checkoutBtn: {
    margin: 16,
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  checkoutText: { color: '#fff', fontSize: 18, fontWeight: '700' },
});

const appStyles = StyleSheet.create({
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  headerTitle: { fontSize: 24, fontWeight: 'bold' },
  cartButton: { position: 'relative', padding: 8 },
  cartText: { fontSize: 24 },
  badge: {
    position: 'absolute',
    top: 0,
    right: 0,
    backgroundColor: '#FF3B30',
    width: 18,
    height: 18,
    borderRadius: 9,
    justifyContent: 'center',
    alignItems: 'center',
  },
  badgeText: { color: '#fff', fontSize: 11, fontWeight: 'bold' },
  productCard: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#fff',
    padding: 16,
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  productImage: {
    width: 70,
    height: 70,
    backgroundColor: '#f5f5f5',
    borderRadius: 10,
    justifyContent: 'center',
    alignItems: 'center',
  },
  productName: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  productPrice: { color: '#666', fontSize: 14 },
  addButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 12,
    paddingVertical: 8,
    borderRadius: 8,
  },
  addText: { color: '#fff', fontWeight: '600' },
});

export default ShoppingApp;
```

---

## Tips และ Best Practices

### 1. Performance
- ใช้ `useCallback` และ `useMemo` กับ Modal handlers
- หลีกเลี่ยงการ render เนื้อหาซับซ้อนเมื่อ Modal ยังไม่แสดง
- ใช้ `removeClippedSubviews` กับ FlatList ใน Modal

### 2. UX
- ให้ผู้ใช้ปิด Modal ได้โดยการกด overlay
- เพิ่ม keyboard handling เมื่อมี input
- ใช้ animation ที่เป็นธรรมชาติ

### 3. Accessibility
```jsx
<Modal
  accessible={true}
  accessibilityViewIsModal={true}
>
  <View accessibilityRole="dialog">
    {/* content */}
  </View>
</Modal>
```

### 4. สิ่งที่ควรระวัง
- `Alert` บน Android ไม่รองรับ `prompt()` - ใช้ custom Modal แทน
- `Modal` ใน `Modal` อาจเกิดปัญหา z-index
- ควรจัดการ keyboard events ด้วย `KeyboardAvoidingView`

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การใช้ `Modal` component พร้อม animations ต่างๆ
- การใช้ `Alert.alert()` สำหรับ native dialogs
- การสร้าง custom ActionSheet
- การสร้าง Bottom Sheet
- Workshop: Shopping Cart Modal ที่สมจริง

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ ScrollView, KeyboardAvoidingView, และ SafeAreaView
