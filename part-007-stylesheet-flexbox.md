# Part 007: StyleSheet และ Flexbox Layout

## สารบัญ
1. [StyleSheet.create()](#stylesheet-create)
2. [Units ใน React Native](#units)
3. [Flexbox พื้นฐาน](#flexbox-พื้นฐาน)
4. [flex](#flex-property)
5. [flexDirection](#flexdirection)
6. [justifyContent](#justifycontent)
7. [alignItems](#alignitems)
8. [alignSelf](#alignself)
9. [flexWrap](#flexwrap)
10. [Spacing: margin และ padding](#spacing)
11. [Width และ Height](#width-height)
12. [Position: absolute และ relative](#position)
13. [zIndex](#zindex)
14. [Workshop: สร้าง Responsive Layout](#workshop)

---

## StyleSheet.create()

### ทำไมต้องใช้ StyleSheet.create()

```tsx
// ✅ แนะนำ - StyleSheet.create()
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
  },
});

// ใช้งาน
<View style={styles.container}>
  <Text style={styles.title}>Hello</Text>
</View>
```

**ข้อดีของ StyleSheet.create():**
1. **Performance**: Styles ถูก validate และ cache ครั้งเดียว (ไม่สร้าง object ใหม่ทุก render)
2. **Validation**: ตรวจสอบ property ที่ไม่ถูกต้อง (dev mode)
3. **Auto-completion**: IDE แนะนำ properties ได้
4. **Optimization**: ใน production ส่ง style ID แทน object

```tsx
// ❌ ไม่แนะนำ - Inline style สร้าง object ใหม่ทุก render
<View style={{ flex: 1, backgroundColor: '#fff' }}>
  <Text style={{ fontSize: 24, color: '#333' }}>Hello</Text>
</View>
```

### Style Arrays

```tsx
// รวม styles หลายอัน
<View style={[styles.base, styles.primary, isActive && styles.active, customStyle]}>
  <Text>Hello</Text>
</View>

// Style ที่อยู่หลังจะ override ค่าก่อนหน้า
const styles = StyleSheet.create({
  base: { padding: 16, backgroundColor: '#fff' },
  primary: { backgroundColor: '#007AFF' },  // Override base
  active: { borderWidth: 2, borderColor: '#FF3B30' },
});
```

### Platform-specific Styles

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 4,
      },
      android: {
        elevation: 4,
      },
    }),
  },
  
  text: {
    fontFamily: Platform.OS === 'ios' ? 'Helvetica Neue' : 'Roboto',
    fontSize: Platform.OS === 'ios' ? 17 : 16,
  },
});

// หรือ
const isIOS = Platform.OS === 'ios';
const isAndroid = Platform.OS === 'android';

// Platform version
const isOldAndroid = Platform.OS === 'android' && Platform.Version < 25;
```

### Flatten Styles

```tsx
import { StyleSheet } from 'react-native';

// flatten รวม style array เป็น object เดียว
const flatStyle = StyleSheet.flatten([styles.base, styles.primary]);
// { flex: 1, backgroundColor: '#007AFF', padding: 16 }

// ใช้เมื่อต้องการ read style value
const fontSize = StyleSheet.flatten(style)?.fontSize ?? 16;
```

---

## Units ใน React Native

### ไม่มี px, em, rem

React Native ใช้ **density-independent pixels (dp)** โดยอัตโนมัติ

```tsx
// ตัวเลขล้วนๆ = dp (density-independent pixels)
const styles = StyleSheet.create({
  box: {
    width: 100,      // 100dp = ขนาดเดียวกันบนทุก device density
    height: 100,
    padding: 16,
    margin: 8,
    fontSize: 16,
    borderRadius: 8,
  },
});
```

### Dimensions API

```tsx
import { Dimensions } from 'react-native';

// ดู screen size
const { width: screenWidth, height: screenHeight } = Dimensions.get('window');
// 'window' = app window, 'screen' = total screen (รวม status bar)

// Responsive sizing
const cardWidth = screenWidth * 0.9;  // 90% ของ screen width
const halfWidth = screenWidth / 2 - 24;  // Half screen with margin

// Listen สำหรับ orientation change
const subscription = Dimensions.addEventListener('change', ({ window }) => {
  console.log('New size:', window.width, window.height);
});

// Cleanup
subscription.remove();
```

### useWindowDimensions Hook (แนะนำ)

```tsx
import { useWindowDimensions } from 'react-native';

const ResponsiveComponent = () => {
  const { width, height, scale, fontScale } = useWindowDimensions();
  
  const isLandscape = width > height;
  const isTablet = width >= 768;
  
  return (
    <View style={{ width: width * 0.9 }}>
      <Text style={{ fontSize: 16 * fontScale }}>
        {isLandscape ? 'Landscape' : 'Portrait'}
      </Text>
    </View>
  );
};
```

### Percentage Values

```tsx
// % ทำงานบน React Native
const styles = StyleSheet.create({
  halfWidth: {
    width: '50%',     // 50% ของ parent
  },
  fullWidth: {
    width: '100%',
  },
  relativeHeight: {
    height: '80%',    // 80% ของ parent height
  },
});
```

---

## Flexbox พื้นฐาน

React Native ใช้ Flexbox เป็น layout system หลัก

### ความแตกต่างจาก Web CSS

| | Web CSS | React Native |
|--|---------|-------------|
| Default flexDirection | row | **column** |
| Default alignContent | flex-start | flex-start |
| Default flexShrink | 1 | **0** |
| flex ใช้ได้กับ | flex items | ทุก View |

```
Flex Container (ค่าเริ่มต้น React Native):
┌─────────────────────────────┐
│  Item 1  ← flexDirection: column
│  Item 2     (จาก top to bottom)
│  Item 3
└─────────────────────────────┘
```

---

## flex Property

```tsx
// flex: กำหนดสัดส่วนการกินพื้นที่

// Example 1: Three equal columns
<View style={{ flexDirection: 'row', flex: 1 }}>
  <View style={{ flex: 1, backgroundColor: '#FF6B6B' }} />
  <View style={{ flex: 1, backgroundColor: '#4ECDC4' }} />
  <View style={{ flex: 1, backgroundColor: '#45B7D1' }} />
</View>

// Example 2: 1:2:1 ratio
<View style={{ flexDirection: 'row', flex: 1 }}>
  <View style={{ flex: 1, backgroundColor: '#FF6B6B' }} />  {/* 25% */}
  <View style={{ flex: 2, backgroundColor: '#4ECDC4' }} />  {/* 50% */}
  <View style={{ flex: 1, backgroundColor: '#45B7D1' }} />  {/* 25% */}
</View>

// Example 3: Fixed + Flexible
<View style={{ flex: 1 }}>
  <View style={{ height: 60 }} />            {/* Fixed header */}
  <View style={{ flex: 1 }} />               {/* Flexible content */}
  <View style={{ height: 80 }} />            {/* Fixed footer */}
</View>
```

### flexGrow, flexShrink, flexBasis

```tsx
// flexBasis: ขนาดเริ่มต้นก่อน distribute space
// flexGrow: โตเท่าไหร่เมื่อมี extra space
// flexShrink: หดเท่าไหร่เมื่อพื้นที่ไม่พอ

<View style={{ flexDirection: 'row' }}>
  <View style={{ flexBasis: 100, flexGrow: 0 }} />  {/* Fixed 100dp */}
  <View style={{ flexBasis: 100, flexGrow: 1 }} />  {/* 100dp + กิน remaining space */}
  <View style={{ flexBasis: 100, flexShrink: 2 }} /> {/* หดเร็วกว่า */}
</View>
```

---

## flexDirection

```tsx
// flexDirection กำหนดทิศทาง main axis

// column (default) - บนลงล่าง
<View style={{ flexDirection: 'column' }}>
  <View style={styles.box1} />
  <View style={styles.box2} />
  <View style={styles.box3} />
</View>

// column-reverse - ล่างขึ้นบน
<View style={{ flexDirection: 'column-reverse' }}>
  <View style={styles.box1} />  {/* อยู่ล่างสุด */}
  <View style={styles.box3} />  {/* อยู่บนสุด */}
</View>

// row - ซ้ายไปขวา
<View style={{ flexDirection: 'row' }}>
  <View style={styles.box1} />
  <View style={styles.box2} />
  <View style={styles.box3} />
</View>

// row-reverse - ขวาไปซ้าย
<View style={{ flexDirection: 'row-reverse' }}>
  <View style={styles.box1} />  {/* อยู่ขวาสุด */}
  <View style={styles.box3} />  {/* อยู่ซ้ายสุด */}
</View>
```

---

## justifyContent

justifyContent จัด items บน **main axis**

```tsx
// ถ้า flexDirection: 'column' → main axis = แนวตั้ง
// ถ้า flexDirection: 'row' → main axis = แนวนอน

// flex-start (default) - ชิดต้น
<View style={{ flex: 1, justifyContent: 'flex-start' }}>
  {/* Items อยู่บนสุด (column) / ซ้ายสุด (row) */}
</View>

// flex-end - ชิดปลาย
<View style={{ flex: 1, justifyContent: 'flex-end' }}>
  {/* Items อยู่ล่างสุด (column) / ขวาสุด (row) */}
</View>

// center - ตรงกลาง
<View style={{ flex: 1, justifyContent: 'center' }}>
  {/* Items อยู่กลาง */}
</View>

// space-between - กระจายเต็มที่ (ชิดขอบ)
<View style={{ flex: 1, justifyContent: 'space-between' }}>
  {/* Item1 ------- Item2 ------- Item3 */}
</View>

// space-around - มี space รอบๆ item
<View style={{ flex: 1, justifyContent: 'space-around' }}>
  {/* _Item1_ _Item2_ _Item3_ */}
</View>

// space-evenly - space เท่ากันทั้งหมด
<View style={{ flex: 1, justifyContent: 'space-evenly' }}>
  {/* _ Item1 _ Item2 _ Item3 _ */}
</View>
```

---

## alignItems

alignItems จัด items บน **cross axis**

```tsx
// ถ้า flexDirection: 'column' → cross axis = แนวนอน
// ถ้า flexDirection: 'row' → cross axis = แนวตั้ง

// stretch (default) - ยืดเต็มความกว้าง/สูง
<View style={{ flex: 1, alignItems: 'stretch' }}>
  {/* Items กว้างเต็ม parent (ถ้าไม่ได้กำหนด width) */}
</View>

// flex-start - ชิดต้น
<View style={{ flex: 1, alignItems: 'flex-start' }}>
  {/* Items ชิดซ้าย (column) / ชิดบน (row) */}
</View>

// flex-end - ชิดปลาย
<View style={{ flex: 1, alignItems: 'flex-end' }}>
  {/* Items ชิดขวา (column) / ชิดล่าง (row) */}
</View>

// center - กลาง
<View style={{ flex: 1, alignItems: 'center' }}>
  {/* Items อยู่กลาง cross axis */}
</View>

// baseline - จัดตาม text baseline
<View style={{ flexDirection: 'row', alignItems: 'baseline' }}>
  <Text style={{ fontSize: 28 }}>Large</Text>
  <Text style={{ fontSize: 14 }}>Small</Text>
  {/* Text จัดตาม baseline */}
</View>
```

---

## alignSelf

alignSelf override alignItems สำหรับ item นั้นๆ

```tsx
<View style={{ flexDirection: 'row', alignItems: 'flex-start' }}>
  <View style={{ width: 50, height: 50 }} />   {/* ชิดบน (default) */}
  <View
    style={{
      width: 50, height: 50,
      alignSelf: 'center',    // Override: อยู่ตรงกลาง
    }}
  />
  <View
    style={{
      width: 50, height: 50,
      alignSelf: 'flex-end',  // Override: ชิดล่าง
    }}
  />
  <View
    style={{
      width: 50, height: 50,
      alignSelf: 'stretch',   // ยืดเต็มความสูง
    }}
  />
</View>
```

---

## flexWrap

```tsx
// nowrap (default) - ไม่ขึ้นบรรทัดใหม่
<View style={{ flexDirection: 'row', flexWrap: 'nowrap' }}>
  {/* items อยู่บรรทัดเดียว อาจล้น */}
</View>

// wrap - ขึ้นบรรทัดใหม่
<View style={{ flexDirection: 'row', flexWrap: 'wrap' }}>
  {/* items ขึ้นบรรทัดใหม่เมื่อเต็ม */}
</View>

// wrap-reverse - ขึ้นบรรทัดใหม่แบบย้อนกลับ
<View style={{ flexDirection: 'row', flexWrap: 'wrap-reverse' }}>
  {/* items ขึ้นบรรทัดจากล่างขึ้นบน */}
</View>
```

### Tag/Chip Layout ด้วย flexWrap

```tsx
const tags = ['React', 'JavaScript', 'TypeScript', 'Mobile', 'iOS', 'Android', 'Expo'];

const TagList = () => (
  <View style={styles.tagContainer}>
    {tags.map(tag => (
      <View key={tag} style={styles.tag}>
        <Text style={styles.tagText}>{tag}</Text>
      </View>
    ))}
  </View>
);

const styles = StyleSheet.create({
  tagContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,             // React Native 0.71+ รองรับ gap
    // หรือใช้ margin แทน gap
  },
  tag: {
    backgroundColor: '#EBF5FB',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 16,
  },
  tagText: {
    color: '#1A5276',
    fontSize: 13,
    fontWeight: '500',
  },
});
```

---

## Spacing: margin และ padding

```tsx
const styles = StyleSheet.create({
  box: {
    // Padding
    padding: 16,                // ทุกด้าน
    paddingHorizontal: 16,      // ซ้ายขวา
    paddingVertical: 12,        // บนล่าง
    paddingTop: 8,              // บน
    paddingBottom: 8,           // ล่าง
    paddingLeft: 16,            // ซ้าย
    paddingRight: 16,           // ขวา
    paddingStart: 16,           // ซ้าย (LTR) / ขวา (RTL) - แนะนำ
    paddingEnd: 16,             // ขวา (LTR) / ซ้าย (RTL) - แนะนำ

    // Margin
    margin: 16,                 // ทุกด้าน
    marginHorizontal: 16,       // ซ้ายขวา
    marginVertical: 12,         // บนล่าง
    marginTop: 8,
    marginBottom: 8,
    marginLeft: 16,
    marginRight: 16,
    marginStart: 16,            // RTL-aware
    marginEnd: 16,              // RTL-aware
  },
});
```

### gap (React Native 0.71+)

```tsx
// gap แทน margin ใน flex containers
<View style={{ flexDirection: 'row', gap: 12 }}>
  <Button />
  <Button />
  <Button />
</View>

// rowGap และ columnGap
<View style={{ flexWrap: 'wrap', rowGap: 8, columnGap: 12 }}>
  {items.map(item => <Item key={item.id} />)}
</View>
```

---

## Width และ Height

### Fixed size

```tsx
<View style={{ width: 200, height: 100 }}>
  {/* Fixed 200x100 dp */}
</View>
```

### Percentage

```tsx
<View style={{ width: '100%', height: '50%' }}>
  {/* 100% of parent width, 50% of parent height */}
</View>
```

### flex: 1

```tsx
<View style={{ flex: 1 }}>
  {/* กิน space ที่เหลือทั้งหมด */}
</View>
```

### minWidth/maxWidth

```tsx
<View style={{ minWidth: 100, maxWidth: 300 }}>
  {/* อย่างน้อย 100, อย่างมาก 300 */}
</View>

<Text style={{ maxWidth: '80%' }} numberOfLines={1}>
  {/* Text ไม่เกิน 80% ของ parent */}
</Text>
```

### aspectRatio

```tsx
// กำหนด width:height ratio
<Image
  source={{ uri: imageUrl }}
  style={{
    width: '100%',
    aspectRatio: 16 / 9,   // 16:9 ratio
  }}
  resizeMode="cover"
/>

// Square image ที่ responsive
<View
  style={{
    width: '48%',
    aspectRatio: 1,           // Square
  }}
>
  <Image style={{ flex: 1 }} source={...} />
</View>
```

---

## Position: absolute และ relative

### position: 'relative' (default)

```tsx
// relative: อยู่ใน normal flow
<View style={{ position: 'relative' }}>
  <View style={{ top: 10, left: 10 }}>
    {/* Offset จาก ตำแหน่ง normal */}
  </View>
</View>
```

### position: 'absolute'

```tsx
// absolute: ออกจาก flow, วางตาม parent
<View style={{ position: 'relative', height: 200 }}>
  {/* Parent ต้องเป็น relative (default ก็ได้) */}
  
  <View
    style={{
      position: 'absolute',
      top: 0,
      right: 0,
      // bottom, left
    }}
  >
    {/* อยู่มุมขวาบนของ parent */}
  </View>
  
  {/* Overlay ทับทั้งหมด */}
  <View
    style={{
      position: 'absolute',
      top: 0,
      left: 0,
      right: 0,
      bottom: 0,
      backgroundColor: 'rgba(0,0,0,0.5)',
    }}
  />
</View>
```

### ตัวอย่าง: Floating Action Button (FAB)

```tsx
const FABExample = () => (
  <View style={{ flex: 1 }}>
    {/* Main content */}
    <ScrollView>
      {/* ... */}
    </ScrollView>
    
    {/* FAB - absolute positioned */}
    <TouchableOpacity
      style={{
        position: 'absolute',
        right: 16,
        bottom: 16,
        width: 56,
        height: 56,
        borderRadius: 28,
        backgroundColor: '#007AFF',
        justifyContent: 'center',
        alignItems: 'center',
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 4 },
        shadowOpacity: 0.3,
        shadowRadius: 8,
        elevation: 8,
      }}
    >
      <Text style={{ color: '#fff', fontSize: 24 }}>+</Text>
    </TouchableOpacity>
  </View>
);
```

### ตัวอย่าง: Badge บน Icon

```tsx
const NotificationIcon = ({ count = 5 }) => (
  <View style={{ position: 'relative', width: 40, height: 40 }}>
    {/* Icon */}
    <View
      style={{
        width: 40, height: 40,
        backgroundColor: '#f0f0f0',
        borderRadius: 20,
        justifyContent: 'center',
        alignItems: 'center',
      }}
    >
      <Text>🔔</Text>
    </View>
    
    {/* Badge */}
    {count > 0 && (
      <View
        style={{
          position: 'absolute',
          top: -2,
          right: -2,
          backgroundColor: '#FF3B30',
          minWidth: 18,
          height: 18,
          borderRadius: 9,
          justifyContent: 'center',
          alignItems: 'center',
          paddingHorizontal: 4,
          borderWidth: 1.5,
          borderColor: '#fff',
        }}
      >
        <Text style={{ color: '#fff', fontSize: 10, fontWeight: '700' }}>
          {count > 99 ? '99+' : count}
        </Text>
      </View>
    )}
  </View>
);
```

---

## zIndex

```tsx
// zIndex ควบคุมลำดับการวาง (ค่ามากขึ้น = อยู่บน)
<View style={{ position: 'relative', height: 200 }}>
  <View style={{ position: 'absolute', top: 0, left: 0, zIndex: 1, backgroundColor: 'red' }} />
  <View style={{ position: 'absolute', top: 20, left: 20, zIndex: 2, backgroundColor: 'blue' }} />
  {/* Blue อยู่บน Red */}
</View>
```

---

## Workshop: สร้าง Responsive Layout

### Workshop 7.1: Basic Layouts

```tsx
import React from 'react';
import { View, Text, StyleSheet, SafeAreaView, useWindowDimensions } from 'react-native';

// Layout Examples
const LayoutExamples = () => {
  const { width } = useWindowDimensions();

  return (
    <SafeAreaView style={styles.container}>
      {/* Example 1: Stack Layout */}
      <Text style={styles.exampleTitle}>1. Stack (Column)</Text>
      <View style={styles.stackContainer}>
        <View style={[styles.box, { backgroundColor: '#FF6B6B' }]}>
          <Text style={styles.boxText}>Top</Text>
        </View>
        <View style={[styles.box, { backgroundColor: '#4ECDC4' }]}>
          <Text style={styles.boxText}>Middle</Text>
        </View>
        <View style={[styles.box, { backgroundColor: '#45B7D1' }]}>
          <Text style={styles.boxText}>Bottom</Text>
        </View>
      </View>

      {/* Example 2: Row Layout */}
      <Text style={styles.exampleTitle}>2. Row</Text>
      <View style={styles.rowContainer}>
        <View style={[styles.box, { flex: 1, backgroundColor: '#FF6B6B' }]}>
          <Text style={styles.boxText}>1</Text>
        </View>
        <View style={[styles.box, { flex: 2, backgroundColor: '#4ECDC4' }]}>
          <Text style={styles.boxText}>2 (ใหญ่กว่า)</Text>
        </View>
        <View style={[styles.box, { flex: 1, backgroundColor: '#45B7D1' }]}>
          <Text style={styles.boxText}>3</Text>
        </View>
      </View>

      {/* Example 3: Center */}
      <Text style={styles.exampleTitle}>3. Center</Text>
      <View style={styles.centerContainer}>
        <View style={[styles.box, { backgroundColor: '#96CEB4' }]}>
          <Text style={styles.boxText}>Centered</Text>
        </View>
      </View>

      {/* Example 4: Responsive Grid */}
      <Text style={styles.exampleTitle}>4. Responsive Grid</Text>
      <View style={styles.gridContainer}>
        {[1,2,3,4,5,6].map(n => (
          <View
            key={n}
            style={[styles.gridItem, { width: (width - 48) / 2 }]}
          >
            <Text style={styles.gridText}>Item {n}</Text>
          </View>
        ))}
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5', padding: 16 },
  exampleTitle: {
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
    marginTop: 16,
    marginBottom: 8,
  },
  stackContainer: { gap: 8 },
  rowContainer: { flexDirection: 'row', gap: 8 },
  centerContainer: {
    height: 100,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#fff',
    borderRadius: 8,
  },
  box: {
    height: 50,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  boxText: { color: '#fff', fontWeight: '600' },
  gridContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 16,
  },
  gridItem: {
    height: 80,
    backgroundColor: '#fff',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 4,
    elevation: 2,
  },
  gridText: { color: '#333', fontWeight: '500' },
});

export default LayoutExamples;
```

### Workshop 7.2: App Layout Structure

```tsx
import React, { useState } from 'react';
import {
  View, Text, TouchableOpacity, ScrollView,
  StyleSheet, SafeAreaView, StatusBar,
} from 'react-native';

// App Shell Layout
const AppLayout = () => {
  const [activeTab, setActiveTab] = useState('home');

  const tabs = [
    { id: 'home', label: 'หน้าแรก', icon: '🏠' },
    { id: 'search', label: 'ค้นหา', icon: '🔍' },
    { id: 'cart', label: 'ตะกร้า', icon: '🛒' },
    { id: 'profile', label: 'โปรไฟล์', icon: '👤' },
  ];

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />
      
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>My App</Text>
        <View style={styles.headerRight}>
          <NotificationIcon count={3} />
        </View>
      </View>

      {/* Content */}
      <ScrollView
        style={styles.content}
        contentContainerStyle={styles.contentContainer}
      >
        <HomeContent />
      </ScrollView>

      {/* Tab Bar */}
      <View style={styles.tabBar}>
        {tabs.map(tab => (
          <TouchableOpacity
            key={tab.id}
            style={styles.tab}
            onPress={() => setActiveTab(tab.id)}
          >
            <Text style={styles.tabIcon}>{tab.icon}</Text>
            <Text
              style={[
                styles.tabLabel,
                activeTab === tab.id && styles.tabLabelActive,
              ]}
            >
              {tab.label}
            </Text>
            {activeTab === tab.id && <View style={styles.tabIndicator} />}
          </TouchableOpacity>
        ))}
      </View>
    </SafeAreaView>
  );
};

// Sub-components
const NotificationIcon = ({ count }) => (
  <View style={{ position: 'relative' }}>
    <Text style={{ fontSize: 24 }}>🔔</Text>
    {count > 0 && (
      <View style={{
        position: 'absolute', top: -4, right: -4,
        backgroundColor: '#FF3B30',
        width: 16, height: 16, borderRadius: 8,
        justifyContent: 'center', alignItems: 'center',
      }}>
        <Text style={{ color: '#fff', fontSize: 9, fontWeight: '700' }}>
          {count}
        </Text>
      </View>
    )}
  </View>
);

const HomeContent = () => (
  <View>
    {/* Banner */}
    <View style={styles.banner}>
      <Text style={styles.bannerTitle}>ยินดีต้อนรับ!</Text>
      <Text style={styles.bannerSubtitle}>ค้นพบสินค้าใหม่ๆ</Text>
    </View>

    {/* Categories */}
    <Text style={styles.sectionTitle}>หมวดหมู่</Text>
    <ScrollView horizontal showsHorizontalScrollIndicator={false}>
      {['อาหาร', 'เสื้อผ้า', 'อิเล็กทรอนิกส์', 'หนังสือ', 'กีฬา'].map(cat => (
        <View key={cat} style={styles.category}>
          <Text style={styles.categoryText}>{cat}</Text>
        </View>
      ))}
    </ScrollView>

    {/* Products Grid */}
    <Text style={styles.sectionTitle}>สินค้าแนะนำ</Text>
    <View style={styles.grid}>
      {[1,2,3,4,5,6].map(n => (
        <View key={n} style={styles.productCard}>
          <View style={styles.productImage} />
          <View style={styles.productInfo}>
            <Text style={styles.productName}>สินค้า {n}</Text>
            <Text style={styles.productPrice}>฿{n * 100}</Text>
          </View>
        </View>
      ))}
    </View>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },

  // Header
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#fff',
    paddingHorizontal: 16,
    paddingVertical: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  headerTitle: { fontSize: 20, fontWeight: 'bold', color: '#111' },
  headerRight: { flexDirection: 'row', alignItems: 'center', gap: 12 },

  // Content
  content: { flex: 1 },
  contentContainer: { padding: 16, gap: 16 },

  // Banner
  banner: {
    backgroundColor: '#007AFF',
    borderRadius: 16,
    padding: 24,
    marginBottom: 8,
  },
  bannerTitle: { fontSize: 22, fontWeight: 'bold', color: '#fff' },
  bannerSubtitle: { fontSize: 14, color: 'rgba(255,255,255,0.8)', marginTop: 4 },

  // Section
  sectionTitle: { fontSize: 18, fontWeight: '700', color: '#111', marginBottom: 12, marginTop: 4 },

  // Categories
  category: {
    backgroundColor: '#fff',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    marginRight: 10,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 2,
    elevation: 1,
  },
  categoryText: { fontSize: 13, color: '#333', fontWeight: '500' },

  // Grid
  grid: { flexDirection: 'row', flexWrap: 'wrap', gap: 12 },
  productCard: {
    width: '48%',
    backgroundColor: '#fff',
    borderRadius: 12,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.06,
    shadowRadius: 4,
    elevation: 2,
  },
  productImage: {
    width: '100%',
    aspectRatio: 1,
    backgroundColor: '#e0e0e0',
  },
  productInfo: { padding: 10 },
  productName: { fontSize: 13, color: '#333', fontWeight: '500' },
  productPrice: { fontSize: 15, color: '#007AFF', fontWeight: '700', marginTop: 2 },

  // Tab Bar
  tabBar: {
    flexDirection: 'row',
    backgroundColor: '#fff',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    paddingBottom: 4,
  },
  tab: {
    flex: 1,
    alignItems: 'center',
    paddingVertical: 8,
    position: 'relative',
  },
  tabIcon: { fontSize: 22 },
  tabLabel: { fontSize: 10, color: '#999', marginTop: 2 },
  tabLabelActive: { color: '#007AFF', fontWeight: '600' },
  tabIndicator: {
    position: 'absolute',
    top: 0,
    left: '25%',
    right: '25%',
    height: 3,
    backgroundColor: '#007AFF',
    borderBottomLeftRadius: 2,
    borderBottomRightRadius: 2,
  },
});

export default AppLayout;
```

### Workshop 7.3: แบบฝึกหัด

**แบบฝึกหัด 1:** สร้าง Dashboard Layout:
- Header พร้อม username และ avatar
- 2x2 Grid ของ stat cards
- Section "กิจกรรมล่าสุด" ที่มีรายการ
- Bottom Tab Bar (4 ปุ่ม)

**แบบฝึกหัด 2:** Responsive Layout:
- บน mobile (width < 768): แสดงแบบ 1 column
- บน tablet (width >= 768): แสดงแบบ 2 column
- ใช้ `useWindowDimensions` hook

**แบบฝึกหัด 3:** ลอก Layout จากแอปที่คุณชอบ:
- เลือก 1 หน้าจากแอปที่คุณชอบ
- วาด wireframe
- implement ด้วย React Native

---

## Tips และ Best Practices

### 1. ใช้ gap แทน margin

```tsx
// ✅ ใหม่ - gap (0.71+)
<View style={{ flexDirection: 'row', gap: 12 }}>
  <Button />
  <Button />
</View>

// ❌ เก่า - margin ทุกตัว
<View style={{ flexDirection: 'row' }}>
  <Button style={{ marginRight: 12 }} />
  <Button />
</View>
```

### 2. Avoid Fixed Heights

```tsx
// ❌ ไม่ดี - height fixed อาจตัดเนื้อหา
<View style={{ height: 100 }}>
  <Text>อาจถูกตัดถ้า text ยาว</Text>
</View>

// ✅ ดี - ใช้ minHeight
<View style={{ minHeight: 100 }}>
  <Text>ขยายได้ตามเนื้อหา</Text>
</View>

// ✅ ดี - padding แทน height
<View style={{ paddingVertical: 16 }}>
  <Text>ปรับตามเนื้อหา</Text>
</View>
```

### 3. SafeArea ทุก Screen

```tsx
// ✅ ทุก screen ควรมี SafeAreaView
const Screen = () => (
  <SafeAreaView style={{ flex: 1 }}>
    {/* content */}
  </SafeAreaView>
);
```

### 4. Platform-specific Shadows

```tsx
// ✅ Shadow บน iOS + Elevation บน Android
const cardStyle = {
  backgroundColor: '#fff',
  ...Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.1,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
  }),
};
```

---

## สรุป Part 007

### ได้เรียนรู้

1. **StyleSheet.create()** - performance, validation, caching
2. **Units** - dp, percentage, Dimensions API
3. **flex** - สัดส่วนการกินพื้นที่
4. **flexDirection** - column (default), row, reverse variants
5. **justifyContent** - จัด items บน main axis
6. **alignItems** - จัด items บน cross axis
7. **alignSelf** - override alignItems สำหรับ item เดียว
8. **flexWrap** - ขึ้นบรรทัดใหม่เมื่อเต็ม
9. **Position** - absolute, relative
10. **zIndex** - ลำดับการวาง

---

**ต่อไป → Part 008: Props และ State**
