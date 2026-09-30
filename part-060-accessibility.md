# Part 060: Accessibility (a11y) ใน React Native

## ความสำคัญของ Accessibility

Accessibility (a11y) คือการออกแบบแอปให้ผู้ใช้ทุกคนสามารถใช้งานได้ รวมถึงผู้ที่มีความพิการทางสายตา การได้ยิน หรือการเคลื่อนไหว

### สถิติที่ควรรู้

```
- 15% ของประชากรโลกมีความพิการบางรูปแบบ
- 2.2 พันล้านคนมีปัญหาด้านการมองเห็น
- ผู้ใช้ screen reader ส่วนใหญ่ใช้ VoiceOver (iOS) หรือ TalkBack (Android)
- กฎหมายในหลายประเทศบังคับให้แอปต้องเข้าถึงได้
```

### Screen Readers

```
iOS: VoiceOver
Android: TalkBack
```

---

## accessibilityLabel

`accessibilityLabel` บอก screen reader ว่าควรอ่านอะไรให้ผู้ใช้ฟัง

```typescript
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Image,
  TextInput,
  StyleSheet,
} from 'react-native';

// ตัวอย่างการใช้ accessibilityLabel
const AccessibilityLabelExamples: React.FC = () => {
  return (
    <View style={a11yStyles.container}>
      
      {/* Button */}
      <TouchableOpacity
        style={a11yStyles.button}
        // ✅ accessibilityLabel อธิบายว่าปุ่มนี้ทำอะไร
        accessibilityLabel="เพิ่มสินค้าลงตะกร้า"
        // ❌ แทนที่จะให้ VoiceOver อ่าน "เพิ่ม" จาก icon
      >
        <Text style={a11yStyles.buttonText}>🛒 เพิ่ม</Text>
      </TouchableOpacity>

      {/* Image */}
      <Image
        source={{ uri: 'https://example.com/product.jpg' }}
        style={a11yStyles.productImage}
        // ✅ อธิบายรูปภาพให้ผู้ใช้ screen reader เข้าใจ
        accessibilityLabel="รูปรองเท้า Nike Air Max สีขาว"
        // ถ้าเป็น decorative image ให้ใช้ accessible={false}
      />

      {/* Decorative Image */}
      <Image
        source={require('./decorative-divider.png')}
        style={a11yStyles.divider}
        accessible={false} // ✅ Skip โดย screen reader
      />

      {/* Icon Button */}
      <TouchableOpacity
        style={a11yStyles.iconButton}
        accessibilityLabel="ค้นหาสินค้า"
      >
        <Text style={a11yStyles.icon}>🔍</Text>
      </TouchableOpacity>

      {/* Combined Text */}
      <TouchableOpacity
        style={a11yStyles.productCard}
        // รวม label ให้ screen reader อ่านในครั้งเดียว
        accessibilityLabel="iPhone 15 Pro ราคา 45,900 บาท มีสินค้า"
      >
        <Text style={a11yStyles.productName}>iPhone 15 Pro</Text>
        <Text style={a11yStyles.productPrice}>฿45,900</Text>
        <Text style={a11yStyles.inStockBadge}>มีสินค้า</Text>
      </TouchableOpacity>

      {/* Text Input */}
      <TextInput
        style={a11yStyles.input}
        placeholder="ค้นหา..."
        // ✅ Label สำหรับ input ที่ไม่มี visible label
        accessibilityLabel="ช่องค้นหาสินค้า"
      />
    </View>
  );
};

const a11yStyles = StyleSheet.create({
  container: { padding: 20 },
  button: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 15,
  },
  buttonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  productImage: { width: '100%', height: 200, borderRadius: 10, marginBottom: 15 },
  divider: { width: '100%', height: 2 },
  iconButton: {
    padding: 12,
    backgroundColor: '#f0f0f0',
    borderRadius: 8,
    alignSelf: 'flex-start',
    marginBottom: 15,
  },
  icon: { fontSize: 24 },
  productCard: {
    backgroundColor: 'white',
    padding: 15,
    borderRadius: 10,
    marginBottom: 15,
    elevation: 2,
  },
  productName: { fontSize: 16, fontWeight: 'bold', color: '#333' },
  productPrice: { fontSize: 18, color: '#2196F3', fontWeight: 'bold' },
  inStockBadge: { fontSize: 12, color: '#4CAF50' },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 15,
  },
});

export default AccessibilityLabelExamples;
```

---

## accessibilityRole

`accessibilityRole` บอก screen reader ว่า element นี้เป็นประเภทอะไร

```typescript
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Switch } from 'react-native';

const AccessibilityRoleExamples: React.FC = () => {
  return (
    <View style={roleStyles.container}>
      
      {/* Button */}
      <TouchableOpacity
        style={roleStyles.button}
        accessibilityRole="button"  // บอก screen reader ว่านี่คือปุ่ม
        accessibilityLabel="ยืนยันการซื้อ"
      >
        <Text style={roleStyles.buttonText}>ยืนยัน</Text>
      </TouchableOpacity>

      {/* Link */}
      <TouchableOpacity
        accessibilityRole="link"    // VoiceOver จะบอกว่า "ลิงก์"
        accessibilityLabel="นโยบายความเป็นส่วนตัว"
      >
        <Text style={roleStyles.link}>นโยบายความเป็นส่วนตัว</Text>
      </TouchableOpacity>

      {/* Header */}
      <Text
        style={roleStyles.heading}
        accessibilityRole="header"  // บอกว่าเป็น heading
      >
        รายการสินค้า
      </Text>

      {/* Image */}
      <View accessibilityRole="image" accessibilityLabel="แผนที่แสดงที่ตั้งร้าน">
        {/* Map component */}
        <Text>แผนที่</Text>
      </View>

      {/* Checkbox */}
      <View style={roleStyles.checkboxRow}>
        <TouchableOpacity
          accessibilityRole="checkbox"
          accessibilityState={{ checked: true }}
          accessibilityLabel="ยอมรับข้อตกลง"
          style={roleStyles.checkbox}
        >
          <Text>✓</Text>
        </TouchableOpacity>
        <Text style={roleStyles.checkboxLabel}>ยอมรับข้อตกลงและเงื่อนไข</Text>
      </View>

      {/* Tab */}
      <View style={roleStyles.tabs}>
        {['ทั้งหมด', 'กำลังดำเนินการ', 'เสร็จสิ้น'].map((tab, index) => (
          <TouchableOpacity
            key={tab}
            style={[roleStyles.tab, index === 0 && roleStyles.activeTab]}
            accessibilityRole="tab"
            accessibilityState={{ selected: index === 0 }}
            accessibilityLabel={`${tab} แท็บ`}
          >
            <Text style={[roleStyles.tabText, index === 0 && roleStyles.activeTabText]}>
              {tab}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* RadioButton */}
      {['ชำระด้วยบัตรเครดิต', 'ชำระด้วยบัตรเดบิต', 'พร้อมเพย์'].map((option, index) => (
        <TouchableOpacity
          key={option}
          style={roleStyles.radioOption}
          accessibilityRole="radio"
          accessibilityState={{ checked: index === 0 }}
          accessibilityLabel={option}
        >
          <View style={[roleStyles.radioCircle, index === 0 && roleStyles.radioSelected]} />
          <Text style={roleStyles.radioLabel}>{option}</Text>
        </TouchableOpacity>
      ))}
    </View>
  );
};

const roleStyles = StyleSheet.create({
  container: { padding: 20 },
  button: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 15,
  },
  buttonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  link: { color: '#2196F3', textDecorationLine: 'underline', marginBottom: 15 },
  heading: { fontSize: 20, fontWeight: 'bold', color: '#333', marginBottom: 15 },
  checkboxRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 15 },
  checkbox: {
    width: 24,
    height: 24,
    borderWidth: 2,
    borderColor: '#2196F3',
    borderRadius: 4,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#2196F3',
    marginRight: 10,
  },
  checkboxLabel: { fontSize: 15, color: '#333' },
  tabs: { flexDirection: 'row', backgroundColor: '#f0f0f0', borderRadius: 8, marginBottom: 15 },
  tab: { flex: 1, padding: 10, alignItems: 'center' },
  activeTab: { backgroundColor: '#2196F3', borderRadius: 8 },
  tabText: { fontSize: 13, color: '#666' },
  activeTabText: { color: 'white', fontWeight: 'bold' },
  radioOption: { flexDirection: 'row', alignItems: 'center', marginBottom: 12 },
  radioCircle: {
    width: 20,
    height: 20,
    borderRadius: 10,
    borderWidth: 2,
    borderColor: '#2196F3',
    marginRight: 10,
  },
  radioSelected: { backgroundColor: '#2196F3' },
  radioLabel: { fontSize: 15, color: '#333' },
});

export default AccessibilityRoleExamples;
```

---

## accessibilityHint

`accessibilityHint` ให้คำอธิบายเพิ่มเติมว่าจะเกิดอะไรขึ้นเมื่อ interact กับ element

```typescript
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, FlatList } from 'react-native';

const AccessibilityHintExamples: React.FC = () => {
  const [liked, setLiked] = useState(false);
  const [expanded, setExpanded] = useState(false);

  return (
    <View style={hintStyles.container}>
      
      {/* Like Button */}
      <TouchableOpacity
        style={hintStyles.likeButton}
        onPress={() => setLiked(l => !l)}
        accessibilityLabel={liked ? 'ถูกใจแล้ว' : 'ถูกใจ'}
        accessibilityHint={liked
          ? 'กดเพื่อยกเลิกการถูกใจ'
          : 'กดเพื่อถูกใจโพสต์นี้'
        }
        accessibilityState={{ checked: liked }}
      >
        <Text style={hintStyles.likeIcon}>{liked ? '❤️' : '🤍'}</Text>
        <Text style={hintStyles.likeCount}>1,234</Text>
      </TouchableOpacity>

      {/* Delete Button */}
      <TouchableOpacity
        style={hintStyles.deleteButton}
        accessibilityLabel="ลบรายการ"
        accessibilityHint="กดค้างเพื่อยืนยันการลบ หรือกดสั้นๆ เพื่อดูตัวเลือก"
      >
        <Text style={hintStyles.deleteText}>🗑️ ลบ</Text>
      </TouchableOpacity>

      {/* Expandable Section */}
      <TouchableOpacity
        style={hintStyles.expandButton}
        onPress={() => setExpanded(e => !e)}
        accessibilityLabel="รายละเอียดสินค้า"
        accessibilityHint={expanded
          ? 'กดเพื่อย่อรายละเอียด'
          : 'กดเพื่อดูรายละเอียดเพิ่มเติม'
        }
        accessibilityState={{ expanded }}
      >
        <Text style={hintStyles.expandTitle}>รายละเอียดสินค้า</Text>
        <Text style={hintStyles.expandArrow}>{expanded ? '▲' : '▼'}</Text>
      </TouchableOpacity>
      
      {expanded && (
        <View style={hintStyles.expandContent}>
          <Text>น้ำหนัก: 200g</Text>
          <Text>ขนาด: 15 x 10 x 5 cm</Text>
          <Text>วัสดุ: พลาสติก ABS</Text>
        </View>
      )}

      {/* Slider */}
      <View
        accessibilityRole="adjustable"
        accessibilityLabel="ระดับเสียง"
        accessibilityHint="ปัดซ้ายขวาเพื่อปรับระดับเสียง"
        accessibilityValue={{ min: 0, max: 100, now: 50 }}
        style={hintStyles.sliderPlaceholder}
      >
        <Text style={hintStyles.sliderText}>Slider: 50%</Text>
      </View>
    </View>
  );
};

const hintStyles = StyleSheet.create({
  container: { padding: 20 },
  likeButton: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 15,
    padding: 10,
    backgroundColor: '#f9f9f9',
    borderRadius: 8,
  },
  likeIcon: { fontSize: 24, marginRight: 8 },
  likeCount: { fontSize: 14, color: '#666' },
  deleteButton: {
    backgroundColor: '#FFEBEE',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 15,
  },
  deleteText: { color: '#F44336', fontWeight: '500' },
  expandButton: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#ddd',
  },
  expandTitle: { fontSize: 15, fontWeight: '500', color: '#333' },
  expandArrow: { fontSize: 14, color: '#888' },
  expandContent: {
    backgroundColor: '#f9f9f9',
    padding: 12,
    borderRadius: 8,
    marginTop: 5,
    marginBottom: 15,
  },
  sliderPlaceholder: {
    backgroundColor: '#E3F2FD',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 10,
  },
  sliderText: { color: '#1565C0', fontSize: 14 },
});

export default AccessibilityHintExamples;
```

---

## accessibilityState

```typescript
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

const AccessibilityStateExamples: React.FC = () => {
  const [isLoading, setIsLoading] = useState(false);
  const [isChecked, setIsChecked] = useState(false);
  const [isDisabled, setIsDisabled] = useState(false);
  const [isSelected, setIsSelected] = useState<number>(0);

  return (
    <View style={stateStyles.container}>
      
      {/* Busy/Loading state */}
      <TouchableOpacity
        style={[stateStyles.button, isLoading && stateStyles.loadingButton]}
        onPress={() => {
          setIsLoading(true);
          setTimeout(() => setIsLoading(false), 2000);
        }}
        accessibilityState={{ busy: isLoading }}  // VoiceOver: "กำลังโหลด"
        accessibilityLabel={isLoading ? 'กำลังบันทึก...' : 'บันทึก'}
        disabled={isLoading}
      >
        <Text style={stateStyles.buttonText}>
          {isLoading ? 'กำลังบันทึก...' : 'บันทึก'}
        </Text>
      </TouchableOpacity>

      {/* Disabled state */}
      <TouchableOpacity
        style={[stateStyles.button, isDisabled && stateStyles.disabledButton]}
        accessibilityState={{ disabled: isDisabled }}
        accessibilityLabel="ชำระเงิน"
        accessibilityHint={isDisabled ? 'กรุณาเลือกที่อยู่ก่อน' : undefined}
        disabled={isDisabled}
        onPress={() => {}}
      >
        <Text style={stateStyles.buttonText}>ชำระเงิน</Text>
      </TouchableOpacity>

      <TouchableOpacity onPress={() => setIsDisabled(d => !d)}>
        <Text style={stateStyles.toggleText}>
          {isDisabled ? 'เปิด' : 'ปิด'} ปุ่มชำระเงิน
        </Text>
      </TouchableOpacity>

      {/* Checked/Selected state */}
      <TouchableOpacity
        style={stateStyles.checkRow}
        onPress={() => setIsChecked(c => !c)}
        accessibilityRole="checkbox"
        accessibilityState={{ checked: isChecked }}
        accessibilityLabel="รับข่าวสารทางอีเมล"
      >
        <View style={[stateStyles.checkbox, isChecked && stateStyles.checkboxChecked]}>
          {isChecked && <Text style={stateStyles.checkMark}>✓</Text>}
        </View>
        <Text style={stateStyles.checkLabel}>รับข่าวสารทางอีเมล</Text>
      </TouchableOpacity>

      {/* Selected in list */}
      {['เมนู A', 'เมนู B', 'เมนู C'].map((menu, index) => (
        <TouchableOpacity
          key={menu}
          style={[stateStyles.menuItem, isSelected === index && stateStyles.selectedMenuItem]}
          onPress={() => setIsSelected(index)}
          accessibilityRole="menuitem"
          accessibilityState={{ selected: isSelected === index }}
          accessibilityLabel={menu}
        >
          <Text style={[stateStyles.menuText, isSelected === index && stateStyles.selectedMenuText]}>
            {menu}
          </Text>
        </TouchableOpacity>
      ))}
    </View>
  );
};

const stateStyles = StyleSheet.create({
  container: { padding: 20 },
  button: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 12,
  },
  loadingButton: { backgroundColor: '#90CAF9' },
  disabledButton: { backgroundColor: '#E0E0E0' },
  buttonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  toggleText: { color: '#2196F3', textAlign: 'center', marginBottom: 20, textDecorationLine: 'underline' },
  checkRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 20 },
  checkbox: {
    width: 24,
    height: 24,
    borderWidth: 2,
    borderColor: '#2196F3',
    borderRadius: 4,
    marginRight: 12,
    justifyContent: 'center',
    alignItems: 'center',
  },
  checkboxChecked: { backgroundColor: '#2196F3' },
  checkMark: { color: 'white', fontWeight: 'bold', fontSize: 14 },
  checkLabel: { fontSize: 15, color: '#333' },
  menuItem: {
    padding: 12,
    borderRadius: 8,
    marginBottom: 5,
    backgroundColor: '#f5f5f5',
  },
  selectedMenuItem: { backgroundColor: '#E3F2FD', borderLeftWidth: 4, borderLeftColor: '#2196F3' },
  menuText: { fontSize: 15, color: '#555' },
  selectedMenuText: { color: '#1565C0', fontWeight: 'bold' },
});
```

---

## Focus Management

```typescript
import React, { useRef, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  AccessibilityInfo,
  findNodeHandle,
} from 'react-native';

const FocusManagementExamples: React.FC = () => {
  const errorMessageRef = useRef<Text>(null);
  const successMessageRef = useRef<View>(null);
  const firstInputRef = useRef<TextInput>(null);

  // Focus ไปยัง element เมื่อ modal เปิด
  const focusOnError = () => {
    if (errorMessageRef.current) {
      const node = findNodeHandle(errorMessageRef.current);
      if (node) {
        AccessibilityInfo.setAccessibilityFocus(node);
      }
    }
  };

  // ตัวอย่าง: Focus ไปยัง element แรกเมื่อ form โหลด
  useEffect(() => {
    // รอให้ screen reader พร้อม
    const timer = setTimeout(() => {
      AccessibilityInfo.isScreenReaderEnabled().then(enabled => {
        if (enabled && firstInputRef.current) {
          const node = findNodeHandle(firstInputRef.current);
          if (node) {
            AccessibilityInfo.setAccessibilityFocus(node);
          }
        }
      });
    }, 500);

    return () => clearTimeout(timer);
  }, []);

  return (
    <View style={focusStyles.container}>
      
      {/* Error ที่ควร focus อัตโนมัติ */}
      <Text
        ref={errorMessageRef}
        style={focusStyles.errorMessage}
        accessibilityRole="alert"  // สำคัญ: VoiceOver จะ announce ทันที
        accessibilityLiveRegion="assertive"  // Announce ทันที
      >
        ❌ รหัสผ่านไม่ถูกต้อง
      </Text>

      <TouchableOpacity
        style={focusStyles.button}
        onPress={focusOnError}
        accessibilityLabel="ทดสอบ focus ไปยัง error"
      >
        <Text style={focusStyles.buttonText}>Focus ไปยัง Error</Text>
      </TouchableOpacity>

      {/* Live Region สำหรับ dynamic content */}
      <View
        ref={successMessageRef}
        accessibilityLiveRegion="polite"  // Announce เมื่อ content เปลี่ยน
        style={focusStyles.liveRegion}
      >
        <Text style={focusStyles.liveText}>
          ข้อความนี้จะถูก announce โดย screen reader เมื่อเปลี่ยน
        </Text>
      </View>

      <TextInput
        ref={firstInputRef}
        style={focusStyles.input}
        placeholder="First input (focused on mount)"
        accessibilityLabel="ชื่อผู้ใช้"
      />
    </View>
  );
};

const focusStyles = StyleSheet.create({
  container: { padding: 20 },
  errorMessage: {
    color: '#C62828',
    fontSize: 15,
    backgroundColor: '#FFEBEE',
    padding: 12,
    borderRadius: 8,
    marginBottom: 15,
  },
  button: {
    backgroundColor: '#2196F3',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 15,
  },
  buttonText: { color: 'white', fontWeight: 'bold' },
  liveRegion: {
    backgroundColor: '#E8F5E9',
    padding: 12,
    borderRadius: 8,
    marginBottom: 15,
  },
  liveText: { color: '#2E7D32', fontSize: 14 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 15,
  },
});
```

---

## Workshop: Accessible App

```typescript
import React, { useState, useRef, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  TextInput,
  FlatList,
  StyleSheet,
  AccessibilityInfo,
  findNodeHandle,
  Switch,
  ScrollView,
  Platform,
} from 'react-native';

interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  inStock: boolean;
  rating: number;
  imageAlt: string;
}

const products: Product[] = [
  {
    id: '1',
    name: 'iPhone 15 Pro',
    price: 45900,
    description: 'สมาร์ทโฟน Apple รุ่นล่าสุด',
    inStock: true,
    rating: 4.8,
    imageAlt: 'iPhone 15 Pro สีไทเทเนียม พร้อม Dynamic Island',
  },
  {
    id: '2',
    name: 'Samsung Galaxy S24',
    price: 29900,
    description: 'สมาร์ทโฟน Android ประมวลผล AI',
    inStock: true,
    rating: 4.6,
    imageAlt: 'Samsung Galaxy S24 สีม่วง หน้าจอโค้ง',
  },
  {
    id: '3',
    name: 'Google Pixel 8',
    price: 25900,
    description: 'Android บริสุทธิ์จาก Google',
    inStock: false,
    rating: 4.5,
    imageAlt: 'Google Pixel 8 สีเขียว',
  },
];

const AccessibleShopApp: React.FC = () => {
  const [cart, setCart] = useState<string[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [filterInStock, setFilterInStock] = useState(false);
  const [announcementText, setAnnouncementText] = useState('');
  const [selectedTab, setSelectedTab] = useState<'products' | 'cart'>('products');
  
  const cartCountRef = useRef<Text>(null);

  // Announce ให้ screen reader
  const announce = useCallback((message: string) => {
    setAnnouncementText(message);
    // Clear หลัง announce
    setTimeout(() => setAnnouncementText(''), 1000);
  }, []);

  const addToCart = useCallback((product: Product) => {
    if (!product.inStock) {
      announce(`${product.name} ไม่มีสินค้าในสต็อก`);
      return;
    }
    
    setCart(prev => [...prev, product.id]);
    announce(`เพิ่ม ${product.name} ลงตะกร้าแล้ว ตะกร้ามี ${cart.length + 1} ชิ้น`);
  }, [cart.length, announce]);

  const removeFromCart = useCallback((productId: string) => {
    const product = products.find(p => p.id === productId);
    setCart(prev => {
      const index = prev.indexOf(productId);
      if (index !== -1) {
        const newCart = [...prev];
        newCart.splice(index, 1);
        return newCart;
      }
      return prev;
    });
    if (product) {
      announce(`ลบ ${product.name} ออกจากตะกร้าแล้ว`);
    }
  }, [announce]);

  const filteredProducts = products.filter(p => {
    if (filterInStock && !p.inStock) return false;
    if (searchQuery && !p.name.toLowerCase().includes(searchQuery.toLowerCase())) return false;
    return true;
  });

  const cartCount = cart.length;
  const cartTotal = cart.reduce((sum, id) => {
    const product = products.find(p => p.id === id);
    return sum + (product?.price || 0);
  }, 0);

  const renderProduct = ({ item }: { item: Product }) => {
    const inCart = cart.includes(item.id);
    
    return (
      <View
        style={shopStyles.productCard}
        accessible={true}
        accessibilityLabel={`${item.name}, ราคา ${item.price.toLocaleString()} บาท, ${item.inStock ? 'มีสินค้า' : 'สินค้าหมด'}, คะแนน ${item.rating} จาก 5`}
      >
        {/* Product Image Placeholder */}
        <View
          style={shopStyles.productImage}
          accessible={true}
          accessibilityLabel={item.imageAlt}
          accessibilityRole="image"
        >
          <Text style={shopStyles.productEmoji}>📱</Text>
        </View>
        
        <View style={shopStyles.productInfo}>
          <Text
            style={shopStyles.productName}
            accessibilityRole="header"
          >
            {item.name}
          </Text>
          
          <Text style={shopStyles.productDesc}>{item.description}</Text>
          
          {/* Rating */}
          <View
            style={shopStyles.ratingRow}
            accessible={true}
            accessibilityLabel={`คะแนน ${item.rating} จาก 5 ดาว`}
          >
            <Text style={shopStyles.stars}>
              {'⭐'.repeat(Math.floor(item.rating))}
            </Text>
            <Text style={shopStyles.ratingText}>{item.rating}</Text>
          </View>
          
          <View style={shopStyles.priceRow}>
            <Text
              style={shopStyles.price}
              accessibilityLabel={`ราคา ${item.price.toLocaleString()} บาท`}
            >
              ฿{item.price.toLocaleString()}
            </Text>
            
            {/* Stock Status */}
            <Text
              style={[shopStyles.stockStatus, { color: item.inStock ? '#4CAF50' : '#F44336' }]}
              accessibilityRole="text"
              accessibilityLabel={item.inStock ? 'มีสินค้าในสต็อก' : 'สินค้าหมด'}
            >
              {item.inStock ? '✅ มีสินค้า' : '❌ สินค้าหมด'}
            </Text>
          </View>
          
          <TouchableOpacity
            style={[
              shopStyles.addButton,
              (!item.inStock || inCart) && shopStyles.addButtonDisabled,
            ]}
            onPress={() => addToCart(item)}
            disabled={!item.inStock}
            accessibilityRole="button"
            accessibilityLabel={
              !item.inStock
                ? `${item.name} สินค้าหมด ไม่สามารถเพิ่มได้`
                : inCart
                ? `${item.name} อยู่ในตะกร้าแล้ว กดเพื่อเพิ่มอีก`
                : `เพิ่ม ${item.name} ลงตะกร้า`
            }
            accessibilityState={{ disabled: !item.inStock }}
          >
            <Text style={shopStyles.addButtonText}>
              {!item.inStock ? 'สินค้าหมด' : inCart ? '+ เพิ่มอีก' : '+ เพิ่มลงตะกร้า'}
            </Text>
          </TouchableOpacity>
        </View>
      </View>
    );
  };

  return (
    <View style={shopStyles.container}>
      
      {/* Accessibility Announcement (invisible) */}
      {announcementText !== '' && (
        <View
          style={shopStyles.srOnly}
          accessible={true}
          accessibilityLiveRegion="assertive"
          accessibilityLabel={announcementText}
        />
      )}

      {/* Tab Bar */}
      <View style={shopStyles.tabBar} accessibilityRole="tablist">
        {[
          { id: 'products' as const, label: 'สินค้า', count: filteredProducts.length },
          { id: 'cart' as const, label: 'ตะกร้า', count: cartCount },
        ].map(tab => (
          <TouchableOpacity
            key={tab.id}
            style={[shopStyles.tab, selectedTab === tab.id && shopStyles.activeTab]}
            onPress={() => setSelectedTab(tab.id)}
            accessibilityRole="tab"
            accessibilityState={{ selected: selectedTab === tab.id }}
            accessibilityLabel={`${tab.label}${tab.count > 0 ? ` ${tab.count} รายการ` : ''}`}
          >
            <Text style={[shopStyles.tabText, selectedTab === tab.id && shopStyles.activeTabText]}>
              {tab.label}
            </Text>
            {tab.count > 0 && (
              <View style={shopStyles.badge} accessible={false}>
                <Text style={shopStyles.badgeText}>{tab.count}</Text>
              </View>
            )}
          </TouchableOpacity>
        ))}
      </View>

      {selectedTab === 'products' ? (
        <View style={{ flex: 1 }}>
          {/* Search */}
          <View style={shopStyles.searchContainer}>
            <TextInput
              style={shopStyles.searchInput}
              value={searchQuery}
              onChangeText={setSearchQuery}
              placeholder="ค้นหาสินค้า..."
              accessibilityLabel="ช่องค้นหาสินค้า"
              accessibilityHint="พิมพ์ชื่อสินค้าที่ต้องการค้นหา"
              returnKeyType="search"
            />
          </View>

          {/* Filter */}
          <View style={shopStyles.filterRow}>
            <Text style={shopStyles.filterLabel}>แสดงเฉพาะสินค้าที่มีในสต็อก</Text>
            <Switch
              value={filterInStock}
              onValueChange={setFilterInStock}
              accessibilityLabel="กรองสินค้าที่มีในสต็อก"
              accessibilityHint={filterInStock ? 'กดเพื่อแสดงสินค้าทั้งหมด' : 'กดเพื่อแสดงเฉพาะสินค้าที่มีในสต็อก'}
              trackColor={{ false: '#ddd', true: '#2196F3' }}
            />
          </View>

          <Text
            style={shopStyles.resultCount}
            accessibilityLiveRegion="polite"
            accessibilityLabel={`พบสินค้า ${filteredProducts.length} รายการ`}
          >
            พบ {filteredProducts.length} รายการ
          </Text>

          <FlatList
            data={filteredProducts}
            keyExtractor={item => item.id}
            renderItem={renderProduct}
            contentContainerStyle={shopStyles.productList}
          />
        </View>
      ) : (
        <ScrollView style={{ flex: 1 }}>
          {cartCount === 0 ? (
            <View style={shopStyles.emptyCart}>
              <Text
                style={shopStyles.emptyCartText}
                accessibilityRole="text"
              >
                🛒 ตะกร้าว่างเปล่า
              </Text>
            </View>
          ) : (
            <View>
              {/* Cart Items */}
              {cart.map((productId, index) => {
                const product = products.find(p => p.id === productId);
                if (!product) return null;
                return (
                  <View
                    key={`${productId}-${index}`}
                    style={shopStyles.cartItem}
                    accessible={true}
                    accessibilityLabel={`${product.name} ราคา ${product.price.toLocaleString()} บาท`}
                  >
                    <Text style={shopStyles.cartItemName}>{product.name}</Text>
                    <Text style={shopStyles.cartItemPrice}>฿{product.price.toLocaleString()}</Text>
                    <TouchableOpacity
                      onPress={() => removeFromCart(productId)}
                      accessibilityLabel={`ลบ ${product.name} ออกจากตะกร้า`}
                      accessibilityRole="button"
                      hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
                    >
                      <Text style={shopStyles.removeText}>✕</Text>
                    </TouchableOpacity>
                  </View>
                );
              })}

              {/* Total */}
              <View
                style={shopStyles.totalRow}
                accessible={true}
                accessibilityLabel={`ราคารวมทั้งหมด ${cartTotal.toLocaleString()} บาท`}
              >
                <Text style={shopStyles.totalLabel}>ราคารวม</Text>
                <Text style={shopStyles.totalPrice}>฿{cartTotal.toLocaleString()}</Text>
              </View>

              <TouchableOpacity
                style={shopStyles.checkoutButton}
                accessibilityRole="button"
                accessibilityLabel={`ชำระเงิน ราคารวม ${cartTotal.toLocaleString()} บาท`}
                accessibilityHint="กดเพื่อดำเนินการชำระเงิน"
              >
                <Text style={shopStyles.checkoutText}>ชำระเงิน</Text>
              </TouchableOpacity>
            </View>
          )}
        </ScrollView>
      )}
    </View>
  );
};

const shopStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  srOnly: {
    position: 'absolute',
    width: 1,
    height: 1,
    overflow: 'hidden',
    opacity: 0,
  },
  tabBar: {
    flexDirection: 'row',
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  tab: {
    flex: 1,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    paddingVertical: 14,
    gap: 6,
  },
  activeTab: { borderBottomWidth: 3, borderBottomColor: '#2196F3' },
  tabText: { fontSize: 15, color: '#888' },
  activeTabText: { color: '#2196F3', fontWeight: 'bold' },
  badge: {
    backgroundColor: '#F44336',
    borderRadius: 10,
    paddingHorizontal: 6,
    paddingVertical: 2,
    minWidth: 20,
    alignItems: 'center',
  },
  badgeText: { color: 'white', fontSize: 11, fontWeight: 'bold' },
  searchContainer: { padding: 12, backgroundColor: 'white' },
  searchInput: {
    backgroundColor: '#f5f5f5',
    borderRadius: 10,
    padding: 10,
    fontSize: 15,
  },
  filterRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  filterLabel: { fontSize: 14, color: '#555' },
  resultCount: { padding: 12, fontSize: 13, color: '#888', backgroundColor: 'white' },
  productList: { padding: 10 },
  productCard: {
    flexDirection: 'row',
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 14,
    marginBottom: 12,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 3,
  },
  productImage: {
    width: 80,
    height: 80,
    backgroundColor: '#E3F2FD',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 14,
  },
  productEmoji: { fontSize: 36 },
  productInfo: { flex: 1 },
  productName: { fontSize: 16, fontWeight: 'bold', color: '#333', marginBottom: 4 },
  productDesc: { fontSize: 12, color: '#888', marginBottom: 6 },
  ratingRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 6 },
  stars: { fontSize: 12, marginRight: 4 },
  ratingText: { fontSize: 12, color: '#666' },
  priceRow: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 8 },
  price: { fontSize: 18, fontWeight: 'bold', color: '#2196F3' },
  stockStatus: { fontSize: 12 },
  addButton: {
    backgroundColor: '#2196F3',
    padding: 8,
    borderRadius: 8,
    alignItems: 'center',
  },
  addButtonDisabled: { backgroundColor: '#BDBDBD' },
  addButtonText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
  emptyCart: { padding: 40, alignItems: 'center' },
  emptyCartText: { fontSize: 18, color: '#999' },
  cartItem: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 14,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  cartItemName: { flex: 1, fontSize: 15, color: '#333' },
  cartItemPrice: { fontSize: 15, fontWeight: 'bold', color: '#2196F3', marginRight: 12 },
  removeText: { color: '#F44336', fontSize: 18, fontWeight: 'bold' },
  totalRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#F5F5F5',
    padding: 16,
  },
  totalLabel: { fontSize: 16, fontWeight: 'bold', color: '#333' },
  totalPrice: { fontSize: 20, fontWeight: 'bold', color: '#2196F3' },
  checkoutButton: {
    margin: 15,
    backgroundColor: '#4CAF50',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  checkoutText: { color: 'white', fontWeight: 'bold', fontSize: 18 },
});

export default AccessibleShopApp;
```

---

## Tips การทดสอบ Accessibility

### 1. เปิดใช้ Screen Reader
```
iOS VoiceOver: Settings → Accessibility → VoiceOver
Android TalkBack: Settings → Accessibility → TalkBack
```

### 2. Checklist การตรวจสอบ

```typescript
// ✅ ทุก interactive element มี accessibilityLabel
// ✅ ทุก image มี accessibilityLabel หรือ accessible={false}
// ✅ ใช้ accessibilityRole ที่ถูกต้อง
// ✅ accessibilityState สะท้อน UI state จริงๆ
// ✅ Touch targets ขนาดอย่างน้อย 44x44 points
// ✅ Color contrast ผ่านมาตรฐาน WCAG AA (4.5:1)
// ✅ ไม่ใช้สีเป็นสิ่งเดียวที่บ่งบอก state
// ✅ Error messages ชัดเจน
// ✅ Focus order สมเหตุสมผล
```

### 3. Minimum Touch Target
```typescript
// ✅ ปุ่มต้องมีขนาดอย่างน้อย 44x44 points
<TouchableOpacity
  style={{ minWidth: 44, minHeight: 44, justifyContent: 'center', alignItems: 'center' }}
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
>
```

### 4. Color Contrast
```typescript
// Text บน Background ต้องมี contrast ratio อย่างน้อย 4.5:1
// ใช้ tool: https://webaim.org/resources/contrastchecker/

// ❌ ไม่ดี - contrast ต่ำ
<Text style={{ color: '#BBBBBB', backgroundColor: 'white' }}>ข้อความ</Text>

// ✅ ดี - contrast สูง
<Text style={{ color: '#333333', backgroundColor: 'white' }}>ข้อความ</Text>
```

---

## สรุป

Accessibility ทำให้แอปใช้งานได้กับทุกคน:
- **accessibilityLabel**: อธิบาย element ให้ screen reader
- **accessibilityRole**: บอกประเภทของ element
- **accessibilityHint**: อธิบาย action ที่จะเกิดขึ้น
- **accessibilityState**: สะท้อน state ปัจจุบัน
- **Focus Management**: จัดการ focus อย่างเหมาะสม
- **Live Regions**: announce dynamic content
- **Touch Targets**: ขนาดอย่างน้อย 44x44 points
- **Color Contrast**: ผ่านมาตรฐาน WCAG
- **Testing**: ทดสอบด้วย VoiceOver/TalkBack จริงๆ
