# Part 010: Lists: FlatList, SectionList, ScrollView

## สารบัญ
1. [เปรียบเทียบ List Components](#เปรียบเทียบ)
2. [ScrollView สำหรับ Lists](#scrollview)
3. [FlatList พื้นฐาน](#flatlist-พื้นฐาน)
4. [FlatList Props ที่สำคัญ](#flatlist-props)
5. [FlatList Optimization](#flatlist-optimization)
6. [SectionList](#sectionlist)
7. [Pull to Refresh](#pull-to-refresh)
8. [Pagination / Load More](#pagination)
9. [Empty State](#empty-state)
10. [Workshop: Contact List App](#workshop)

---

## เปรียบเทียบ List Components

| Component | ใช้เมื่อ | Performance | Features |
|-----------|---------|-------------|----------|
| ScrollView | ข้อมูลน้อย (< 50 items) | ปานกลาง | ง่าย |
| FlatList | ข้อมูลมาก, เรียงเป็นรายการ | สูง | virtualized |
| SectionList | ข้อมูลแบ่งกลุ่ม | สูง | headers per section |
| FlashList | ข้อมูลมากมาก | สูงมาก | Shopify's alternative |

---

## ScrollView สำหรับ Lists

ScrollView โหลดทุก items พร้อมกัน - **ไม่แนะนำ** สำหรับ list ขนาดใหญ่

```tsx
import React from 'react';
import { ScrollView, View, Text, StyleSheet } from 'react-native';

// ✅ ใช้ได้กับข้อมูลน้อย (< 50 items)
const SmallList = () => {
  const items = Array.from({ length: 20 }, (_, i) => ({
    id: String(i + 1),
    name: `รายการ ${i + 1}`,
  }));

  return (
    <ScrollView style={styles.container}>
      {items.map(item => (
        <View key={item.id} style={styles.item}>
          <Text style={styles.itemText}>{item.name}</Text>
        </View>
      ))}
    </ScrollView>
  );
};

// ❌ ไม่แนะนำกับข้อมูลมาก - โหลดทั้งหมดพร้อมกัน
const HugeList = () => {
  const items = Array.from({ length: 1000 }, ...);  // ช้ามาก!
  return <ScrollView>{items.map(...)}</ScrollView>;
};
```

### ScrollView ขั้นสูง

```tsx
const AdvancedScrollView = () => {
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = async () => {
    setRefreshing(true);
    await fetchData();
    setRefreshing(false);
  };

  return (
    <ScrollView
      // Direction
      horizontal={false}
      showsVerticalScrollIndicator={false}
      showsHorizontalScrollIndicator={false}
      
      // Bounce (iOS)
      bounces={true}
      alwaysBounceVertical={false}
      
      // Content
      contentContainerStyle={styles.contentContainer}
      contentInset={{ top: 0, bottom: 20 }}  // iOS only
      
      // Keyboard
      keyboardDismissMode="on-drag"
      keyboardShouldPersistTaps="handled"
      
      // Pull to Refresh
      refreshControl={
        <RefreshControl
          refreshing={refreshing}
          onRefresh={onRefresh}
          tintColor="#007AFF"
          colors={['#007AFF']}
          title="ดึงเพื่อรีเฟรช"
          titleColor="#999"
        />
      }
      
      // Scroll Events
      scrollEventThrottle={16}
      onScroll={({ nativeEvent }) => {
        const { contentOffset, layoutMeasurement, contentSize } = nativeEvent;
        const isAtBottom =
          layoutMeasurement.height + contentOffset.y >= contentSize.height - 20;
        if (isAtBottom) loadMore();
      }}
      
      // Snap
      pagingEnabled={false}
      snapToInterval={200}
      snapToAlignment="start"
      decelerationRate="fast"
    >
      {/* content */}
    </ScrollView>
  );
};
```

---

## FlatList พื้นฐาน

FlatList ใช้ Virtualization - render เฉพาะ items ที่อยู่ใน viewport

```tsx
import React from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

type Item = { id: string; name: string; price: number };

const data: Item[] = [
  { id: '1', name: 'สินค้า 1', price: 100 },
  { id: '2', name: 'สินค้า 2', price: 200 },
  // ...
];

// รูปแบบพื้นฐาน
const BasicFlatList = () => (
  <FlatList
    data={data}
    keyExtractor={(item) => item.id}
    renderItem={({ item }) => (
      <View style={styles.item}>
        <Text>{item.name}</Text>
        <Text>฿{item.price}</Text>
      </View>
    )}
  />
);

// renderItem props
const renderItem = ({ item, index, separators }) => {
  // item: ข้อมูลของ item นั้น
  // index: ลำดับ (0-based)
  // separators: { highlight, unhighlight, updateProps }
  
  return (
    <View key={item.id} style={styles.item}>
      <Text>{index + 1}. {item.name}</Text>
    </View>
  );
};
```

---

## FlatList Props ที่สำคัญ

### Props พื้นฐาน

```tsx
<FlatList
  // จำเป็น
  data={data}                           // Array ของข้อมูล
  keyExtractor={(item) => item.id}      // Key สำหรับแต่ละ item
  renderItem={({ item }) => <ItemCard item={item} />}  // render function
  
  // Layout
  horizontal={false}                    // แนวนอน/ตั้ง
  numColumns={2}                        // Grid layout
  columnWrapperStyle={{ gap: 8 }}       // Style สำหรับ row ใน grid
  inverted={false}                      // กลับด้าน (messages UI)
  
  // Spacing
  contentContainerStyle={{ padding: 16, gap: 12 }}
  ItemSeparatorComponent={() => <View style={{ height: 8 }} />}
  ListHeaderComponent={<Header />}      // Component ด้านบน
  ListFooterComponent={<Footer />}      // Component ด้านล่าง
  ListEmptyComponent={<EmptyState />}   // แสดงเมื่อ data ว่าง
  
  // Performance (อธิบายด้านล่าง)
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  initialNumToRender={10}
  maxToRenderPerBatch={5}
  windowSize={5}
  removeClippedSubviews={true}
  
  // Refresh
  refreshing={refreshing}
  onRefresh={onRefresh}
  
  // Pagination
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}           // 50% จากล่าง
  
  // Scroll
  showsVerticalScrollIndicator={false}
  showsHorizontalScrollIndicator={false}
  scrollEventThrottle={16}
  onScroll={handleScroll}
  
  // Keyboard
  keyboardDismissMode="on-drag"
  keyboardShouldPersistTaps="handled"
  
  // Ref
  ref={flatListRef}
  
  // Debug
  debug={false}
/>
```

### ListHeaderComponent และ ListFooterComponent

```tsx
// Header และ Footer เป็นส่วนของ FlatList (scroll พร้อมกัน)
<FlatList
  data={products}
  ListHeaderComponent={
    <View style={styles.header}>
      <Text style={styles.headerTitle}>สินค้าแนะนำ</Text>
      <TouchableOpacity>
        <Text style={styles.viewAll}>ดูทั้งหมด</Text>
      </TouchableOpacity>
    </View>
  }
  ListFooterComponent={
    isLoadingMore ? (
      <ActivityIndicator style={{ padding: 20 }} />
    ) : (
      <View style={{ height: 40 }} />  // Bottom spacing
    )
  }
  renderItem={renderProduct}
  keyExtractor={item => item.id}
/>
```

### ItemSeparatorComponent

```tsx
// Separator ระหว่าง items (ไม่แสดงหลัง item สุดท้าย)
<FlatList
  ItemSeparatorComponent={() => (
    <View style={styles.separator} />
  )}
  // ...
/>

// หรือใช้แบบมี custom logic
<FlatList
  ItemSeparatorComponent={({ highlighted }) => (
    <View
      style={[
        styles.separator,
        highlighted && { backgroundColor: '#007AFF' },
      ]}
    />
  )}
/>
```

### Grid Layout

```tsx
// 2-column grid
<FlatList
  data={products}
  numColumns={2}
  columnWrapperStyle={{ gap: 12, paddingHorizontal: 16 }}
  contentContainerStyle={{ paddingTop: 16, paddingBottom: 20 }}
  renderItem={({ item }) => (
    <View style={{ flex: 1 }}>
      <ProductCard product={item} />
    </View>
  )}
  keyExtractor={item => item.id}
/>

// Responsive grid
const { width } = useWindowDimensions();
const numColumns = width >= 768 ? 3 : 2;
const itemWidth = (width - 16 * (numColumns + 1)) / numColumns;

<FlatList
  key={numColumns}  // ต้อง re-mount เมื่อ numColumns เปลี่ยน
  numColumns={numColumns}
  data={items}
  renderItem={({ item }) => (
    <View style={{ width: itemWidth }}>
      <ItemCard item={item} />
    </View>
  )}
/>
```

### Horizontal FlatList

```tsx
// Horizontal scrolling list
<FlatList
  horizontal
  data={categories}
  showsHorizontalScrollIndicator={false}
  contentContainerStyle={{ paddingHorizontal: 16, gap: 12 }}
  renderItem={({ item }) => (
    <TouchableOpacity style={styles.categoryChip}>
      <Text>{item.name}</Text>
    </TouchableOpacity>
  )}
  keyExtractor={item => item.id}
/>

// Snap horizontal
<FlatList
  horizontal
  pagingEnabled     // Snap ทีละ item (ขนาดเท่า screen)
  // หรือ
  snapToInterval={CARD_WIDTH + CARD_GAP}
  snapToAlignment="start"
  decelerationRate="fast"
  showsHorizontalScrollIndicator={false}
  data={banners}
  renderItem={...}
/>
```

---

## FlatList Optimization

### 1. getItemLayout - สำคัญมาก!

```tsx
const ITEM_HEIGHT = 80;  // ต้องรู้ความสูงล่วงหน้า

<FlatList
  // ✅ บอก FlatList ขนาด item ล่วงหน้า
  // ช่วยให้ scroll to index ทำงานได้ และ render เร็วขึ้น
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,           // ความสูง item
    offset: ITEM_HEIGHT * index,   // ตำแหน่ง Y ของ item
    index,
  })}
  
  // กับ separator
  // offset = (ITEM_HEIGHT + SEPARATOR_HEIGHT) * index
/>

// ✅ สำหรับ dynamic heights - ใช้ onLayout
const itemHeights = useRef({});

<FlatList
  renderItem={({ item, index }) => (
    <View
      onLayout={(event) => {
        itemHeights.current[index] = event.nativeEvent.layout.height;
      }}
    >
      <DynamicItem item={item} />
    </View>
  )}
/>
```

### 2. initialNumToRender

```tsx
<FlatList
  // จำนวน items ที่ render ตั้งแต่ต้น
  // มากขึ้น = ช้าลงในตอนแรก แต่ scroll ไม่กระตุก
  initialNumToRender={10}     // default: 10
  initialScrollIndex={0}      // เริ่มที่ index นี้
/>
```

### 3. windowSize

```tsx
<FlatList
  // จำนวน "viewport heights" ที่ render ไว้
  // windowSize = 5 → render 5 viewport (2 above + current + 2 below)
  // น้อยกว่า = memory น้อย แต่ blank ขณะ scroll เร็ว
  windowSize={5}     // default: 21
/>
```

### 4. maxToRenderPerBatch

```tsx
<FlatList
  // จำนวน items ที่ render ต่อ batch
  // น้อยกว่า = scroll ราบเรียบกว่า, มากกว่า = blank น้อยกว่า
  maxToRenderPerBatch={5}     // default: 10
/>
```

### 5. removeClippedSubviews

```tsx
<FlatList
  // ✅ ลบ subviews ที่อยู่นอก viewport (Android เท่านั้น)
  // ลด memory แต่อาจ flicker ใน some cases
  removeClippedSubviews={true}  // default: false (true ใน Android)
/>
```

### 6. React.memo สำหรับ renderItem

```tsx
// ✅ ป้องกัน re-render ของ items ที่ไม่เปลี่ยน
const ProductItem = React.memo(({ item, onPress }) => (
  <TouchableOpacity onPress={() => onPress(item.id)}>
    <View style={styles.item}>
      <Text>{item.name}</Text>
      <Text>฿{item.price}</Text>
    </View>
  </TouchableOpacity>
));

// useCallback สำหรับ handler
const handlePress = useCallback((id: string) => {
  navigateToProduct(id);
}, []);

<FlatList
  data={products}
  renderItem={({ item }) => (
    <ProductItem item={item} onPress={handlePress} />
  )}
/>
```

### 7. FlatList Methods (Ref)

```tsx
const flatListRef = useRef<FlatList>(null);

// Scroll to index
flatListRef.current?.scrollToIndex({ index: 5, animated: true });

// Scroll to item
flatListRef.current?.scrollToItem({ item: specificItem, animated: true });

// Scroll to offset
flatListRef.current?.scrollToOffset({ offset: 500, animated: true });

// Scroll to end
flatListRef.current?.scrollToEnd({ animated: true });

// Record Interaction (สำหรับ virtualization)
flatListRef.current?.recordInteraction();
```

---

## SectionList

SectionList สำหรับข้อมูลที่แบ่งกลุ่ม มี section headers

### โครงสร้างข้อมูล

```tsx
type Contact = {
  id: string;
  name: string;
  phone: string;
};

type Section = {
  title: string;        // Section header text
  data: Contact[];      // Items ใน section นี้
};

const sections: Section[] = [
  {
    title: 'ก',
    data: [
      { id: '1', name: 'กนกวรรณ', phone: '081-234-5678' },
      { id: '2', name: 'กานต์', phone: '082-345-6789' },
    ],
  },
  {
    title: 'ข',
    data: [
      { id: '3', name: 'ขวัญ', phone: '083-456-7890' },
    ],
  },
  // ...
];
```

### การใช้งาน SectionList

```tsx
import { SectionList } from 'react-native';

<SectionList
  sections={sections}
  
  // Key สำหรับ items
  keyExtractor={(item) => item.id}
  
  // Render แต่ละ item
  renderItem={({ item, index, section, separators }) => (
    <View style={styles.contactItem}>
      <View style={styles.avatar}>
        <Text style={styles.avatarText}>
          {item.name.charAt(0).toUpperCase()}
        </Text>
      </View>
      <View style={styles.contactInfo}>
        <Text style={styles.contactName}>{item.name}</Text>
        <Text style={styles.contactPhone}>{item.phone}</Text>
      </View>
    </View>
  )}
  
  // Render Section Header
  renderSectionHeader={({ section: { title } }) => (
    <View style={styles.sectionHeader}>
      <Text style={styles.sectionHeaderText}>{title}</Text>
    </View>
  )}
  
  // Render Section Footer (optional)
  renderSectionFooter={({ section }) => (
    <View style={styles.sectionFooter}>
      <Text style={styles.sectionFooterText}>
        {section.data.length} รายชื่อ
      </Text>
    </View>
  )}
  
  // Separator ระหว่าง items
  ItemSeparatorComponent={() => <View style={styles.separator} />}
  
  // Section header sticky
  stickySectionHeadersEnabled={true}  // default: true on iOS
  
  // List header/footer
  ListHeaderComponent={<SearchBar />}
  ListEmptyComponent={<EmptyContactList />}
  
  // Refresh
  refreshing={refreshing}
  onRefresh={onRefresh}
  
  // Performance
  initialNumToRender={20}
  maxToRenderPerBatch={10}
/>
```

### Alphabet Index (Contact-style)

```tsx
import React, { useRef, useState } from 'react';
import {
  View, Text, SectionList, TouchableOpacity, StyleSheet,
} from 'react-native';

const AlphabetContactList = ({ contacts }) => {
  const sectionListRef = useRef(null);

  // จัดกลุ่มผู้ติดต่อตามตัวอักษร
  const sections = useMemo(() => {
    const grouped = contacts.reduce((acc, contact) => {
      const firstChar = contact.name.charAt(0).toUpperCase();
      if (!acc[firstChar]) acc[firstChar] = [];
      acc[firstChar].push(contact);
      return acc;
    }, {} as Record<string, Contact[]>);

    return Object.entries(grouped)
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([title, data]) => ({ title, data }));
  }, [contacts]);

  const alphabet = sections.map(s => s.title);

  const scrollToSection = (letter: string) => {
    const sectionIndex = sections.findIndex(s => s.title === letter);
    if (sectionIndex !== -1) {
      sectionListRef.current?.scrollToLocation({
        sectionIndex,
        itemIndex: 0,
        animated: true,
      });
    }
  };

  return (
    <View style={{ flex: 1 }}>
      <SectionList
        ref={sectionListRef}
        sections={sections}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <View style={styles.item}>
            <Text>{item.name}</Text>
            <Text style={styles.phone}>{item.phone}</Text>
          </View>
        )}
        renderSectionHeader={({ section: { title } }) => (
          <View style={styles.sectionHeader}>
            <Text style={styles.sectionTitle}>{title}</Text>
          </View>
        )}
        stickySectionHeadersEnabled
      />

      {/* Alphabet Index */}
      <View style={styles.alphabetIndex}>
        {alphabet.map(letter => (
          <TouchableOpacity
            key={letter}
            onPress={() => scrollToSection(letter)}
            style={styles.letterButton}
          >
            <Text style={styles.letterText}>{letter}</Text>
          </TouchableOpacity>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  item: { padding: 16, backgroundColor: '#fff', flexDirection: 'row', justifyContent: 'space-between' },
  phone: { color: '#999' },
  sectionHeader: {
    paddingHorizontal: 16,
    paddingVertical: 6,
    backgroundColor: '#f5f5f5',
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0',
  },
  sectionTitle: { fontSize: 14, fontWeight: '700', color: '#007AFF' },
  alphabetIndex: {
    position: 'absolute',
    right: 4,
    top: 0,
    bottom: 0,
    justifyContent: 'center',
    paddingVertical: 8,
  },
  letterButton: { padding: 2, alignItems: 'center' },
  letterText: { fontSize: 11, color: '#007AFF', fontWeight: '600' },
});
```

---

## Pull to Refresh

```tsx
import React, { useState, useCallback } from 'react';
import { FlatList, RefreshControl, View, Text, StyleSheet } from 'react-native';

const RefreshableList = () => {
  const [data, setData] = useState(initialData);
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = useCallback(async () => {
    setRefreshing(true);
    try {
      const newData = await fetchLatestData();
      setData(newData);
    } catch (error) {
      console.error('Refresh failed:', error);
    } finally {
      setRefreshing(false);
    }
  }, []);

  return (
    <FlatList
      data={data}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <ItemCard item={item} />}
      
      refreshControl={
        <RefreshControl
          refreshing={refreshing}
          onRefresh={onRefresh}
          
          // Styling
          tintColor="#007AFF"          // iOS spinner color
          colors={['#007AFF', '#FF3B30', '#34C759']}  // Android spinner colors
          
          // Text (iOS only)
          title="ดึงเพื่อรีเฟรช"
          titleColor="#999"
          
          // Offset
          progressViewOffset={20}       // Android: ระยะจากบน
        />
      }
    />
  );
};
```

### Custom Pull to Refresh Animation

```tsx
import Animated, {
  useAnimatedScrollHandler,
  useSharedValue,
  useAnimatedStyle,
  interpolate,
  withSpring,
} from 'react-native-reanimated';

const CustomRefresh = () => {
  const translateY = useSharedValue(0);
  const isRefreshing = useSharedValue(false);
  const REFRESH_THRESHOLD = 80;

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      if (!isRefreshing.value) {
        translateY.value = Math.max(0, -event.contentOffset.y);
      }
    },
  });

  const refreshIndicatorStyle = useAnimatedStyle(() => ({
    opacity: interpolate(translateY.value, [0, REFRESH_THRESHOLD], [0, 1]),
    transform: [
      { translateY: translateY.value - REFRESH_THRESHOLD },
      { rotate: `${interpolate(translateY.value, [0, REFRESH_THRESHOLD], [0, 360])}deg` },
    ],
  }));

  return (
    <View style={{ flex: 1 }}>
      {/* Custom Refresh Indicator */}
      <Animated.View style={[styles.refreshIndicator, refreshIndicatorStyle]}>
        <Text style={styles.refreshIcon}>↻</Text>
      </Animated.View>

      <Animated.FlatList
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        data={data}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
      />
    </View>
  );
};
```

---

## Pagination / Load More

### แบบ Basic

```tsx
const PaginatedList = () => {
  const [items, setItems] = useState<Item[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [initialLoading, setInitialLoading] = useState(true);

  const PAGE_SIZE = 20;

  const fetchPage = async (pageNum: number) => {
    if (loading) return;
    
    setLoading(true);
    try {
      const response = await api.getItems({ page: pageNum, limit: PAGE_SIZE });
      
      if (pageNum === 1) {
        setItems(response.data);
      } else {
        setItems(prev => [...prev, ...response.data]);
      }
      
      setHasMore(response.data.length === PAGE_SIZE);
      setPage(pageNum + 1);
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
      setInitialLoading(false);
    }
  };

  useEffect(() => {
    fetchPage(1);
  }, []);

  const handleLoadMore = () => {
    if (hasMore && !loading) {
      fetchPage(page);
    }
  };

  const handleRefresh = async () => {
    setHasMore(true);
    await fetchPage(1);
  };

  if (initialLoading) {
    return (
      <View style={styles.loadingContainer}>
        <ActivityIndicator size="large" color="#007AFF" />
      </View>
    );
  }

  return (
    <FlatList
      data={items}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <ItemCard item={item} />}
      
      // Pull to refresh
      refreshing={loading && page === 2}
      onRefresh={handleRefresh}
      
      // Load more
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.3}
      
      // Loading indicator ด้านล่าง
      ListFooterComponent={
        loading && hasMore ? (
          <View style={styles.loadingMore}>
            <ActivityIndicator color="#007AFF" />
            <Text style={styles.loadingText}>กำลังโหลด...</Text>
          </View>
        ) : !hasMore ? (
          <Text style={styles.endText}>แสดงครบทั้งหมดแล้ว</Text>
        ) : null
      }
    />
  );
};
```

### Pagination ด้วย Custom Hook

```tsx
// hooks/usePagination.ts
import { useState, useCallback, useEffect } from 'react';

interface UsePaginationOptions<T> {
  fetchFn: (page: number, pageSize: number) => Promise<T[]>;
  pageSize?: number;
  autoFetch?: boolean;
}

function usePagination<T>({
  fetchFn,
  pageSize = 20,
  autoFetch = true,
}: UsePaginationOptions<T>) {
  const [data, setData] = useState<T[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [refreshing, setRefreshing] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async (pageNum: number, isRefresh = false) => {
    if (loading && !isRefresh) return;
    
    try {
      if (isRefresh) setRefreshing(true);
      else setLoading(true);
      
      setError(null);
      const newData = await fetchFn(pageNum, pageSize);
      
      if (isRefresh || pageNum === 1) {
        setData(newData);
      } else {
        setData(prev => [...prev, ...newData]);
      }
      
      setHasMore(newData.length === pageSize);
      setPage(pageNum + 1);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
      setRefreshing(false);
    }
  }, [fetchFn, pageSize]);

  useEffect(() => {
    if (autoFetch) fetchData(1);
  }, []);

  const loadMore = useCallback(() => {
    if (hasMore && !loading) fetchData(page);
  }, [hasMore, loading, page, fetchData]);

  const refresh = useCallback(() => {
    setPage(1);
    setHasMore(true);
    fetchData(1, true);
  }, [fetchData]);

  return { data, loading, refreshing, hasMore, error, loadMore, refresh };
}

// ใช้งาน
const ProductList = () => {
  const {
    data: products,
    loading,
    refreshing,
    hasMore,
    loadMore,
    refresh,
  } = usePagination({
    fetchFn: (page, size) => api.getProducts({ page, limit: size }),
    pageSize: 20,
  });

  return (
    <FlatList
      data={products}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <ProductCard product={item} />}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      refreshing={refreshing}
      onRefresh={refresh}
      ListFooterComponent={
        loading && hasMore ? <ActivityIndicator /> : null
      }
    />
  );
};
```

---

## Empty State

```tsx
// Empty State Component ที่ดี
const EmptyState = ({ message, icon, onAction, actionLabel }) => (
  <View style={styles.emptyContainer}>
    <Text style={styles.emptyIcon}>{icon || '📭'}</Text>
    <Text style={styles.emptyTitle}>ไม่มีข้อมูล</Text>
    <Text style={styles.emptyMessage}>{message || 'ยังไม่มีรายการในขณะนี้'}</Text>
    {onAction && (
      <TouchableOpacity style={styles.emptyAction} onPress={onAction}>
        <Text style={styles.emptyActionText}>{actionLabel || 'ลองใหม่'}</Text>
      </TouchableOpacity>
    )}
  </View>
);

// ใช้ใน FlatList
<FlatList
  data={filteredData}
  ListEmptyComponent={
    loading ? (
      <LoadingState />
    ) : error ? (
      <ErrorState error={error} onRetry={refetch} />
    ) : (
      <EmptyState
        icon="🔍"
        message="ไม่พบสินค้าที่ค้นหา"
        onAction={() => setSearchText('')}
        actionLabel="ล้างการค้นหา"
      />
    )
  }
/>
```

---

## Workshop: Contact List App

```tsx
import React, { useState, useMemo, useCallback, useRef } from 'react';
import {
  View, Text, TextInput, FlatList, SectionList, TouchableOpacity,
  Image, StyleSheet, SafeAreaView, StatusBar, Modal, Alert,
  Animated, Platform,
} from 'react-native';

// Types
type Contact = {
  id: string;
  firstName: string;
  lastName: string;
  phone: string;
  email?: string;
  avatar?: string;
  isFavorite: boolean;
};

type Section = {
  title: string;
  data: Contact[];
};

// Sample Data
const generateContacts = (): Contact[] => {
  const names = [
    ['กนกวรรณ', 'สุขใจ'], ['กิตติ', 'มงคล'], ['ขวัญชัย', 'ทองดี'],
    ['คมสัน', 'วีรชัย'], ['จิรา', 'พรชัย'], ['จุฑามาศ', 'รัตนา'],
    ['ชัชชาติ', 'สิทธิพันธุ์'], ['ณัฐพล', 'ใจดี'], ['ดวงใจ', 'สมบูรณ์'],
    ['ตรีทิพย์', 'แสงทอง'], ['ธนากร', 'วชิรพงศ์'], ['นัดดา', 'รักษ์'],
    ['บุณยาพร', 'เจริญ'], ['ปรีชา', 'วิทยา'], ['พิมพ์ใจ', 'สกุล'],
    ['ภูมิพัฒน์', 'ชัยมงคล'], ['มณีรัตน์', 'สุวรรณ'], ['ยุภา', 'ประสิทธิ์'],
    ['รัตนา', 'พงษ์ไทย'], ['ลลิตา', 'เนตร'], ['วัชรพล', 'โชติ'],
    ['ศิริพร', 'ทิพย์'], ['สมชาย', 'ใจดี'], ['หฤษฎ์', 'วิวัฒน์'],
    ['อนุชา', 'เพ็ชร'], ['อรทัย', 'มานะ'],
  ];

  return names.map(([first, last], index) => ({
    id: String(index + 1),
    firstName: first,
    lastName: last,
    phone: `0${Math.floor(Math.random() * 9) + 1}${Math.random().toString().slice(2, 10)}`,
    email: `${first.toLowerCase()}@example.com`,
    isFavorite: index % 5 === 0,
  }));
};

// Group contacts by first letter
const groupContacts = (contacts: Contact[]): Section[] => {
  const grouped: Record<string, Contact[]> = {};

  contacts
    .sort((a, b) => a.firstName.localeCompare(b.firstName, 'th'))
    .forEach(contact => {
      const letter = contact.firstName.charAt(0);
      if (!grouped[letter]) grouped[letter] = [];
      grouped[letter].push(contact);
    });

  return Object.entries(grouped)
    .sort(([a], [b]) => a.localeCompare(b, 'th'))
    .map(([title, data]) => ({ title, data }));
};

// Avatar Component
const ContactAvatar = ({ contact, size = 44 }) => {
  const initials = `${contact.firstName.charAt(0)}${contact.lastName.charAt(0)}`;
  
  const getColor = (name: string) => {
    const colors = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7', '#DDA0DD'];
    let hash = 0;
    for (let i = 0; i < name.length; i++) hash += name.charCodeAt(i);
    return colors[hash % colors.length];
  };

  return (
    <View style={[
      styles.avatar,
      {
        width: size, height: size, borderRadius: size / 2,
        backgroundColor: getColor(contact.firstName),
      },
    ]}>
      <Text style={[styles.avatarText, { fontSize: size * 0.35 }]}>
        {initials}
      </Text>
    </View>
  );
};

// Contact Item
const ContactItem = React.memo(({ contact, onPress, onFavoriteToggle }) => (
  <TouchableOpacity style={styles.contactItem} onPress={() => onPress(contact)}>
    <ContactAvatar contact={contact} />
    <View style={styles.contactInfo}>
      <Text style={styles.contactName}>
        {contact.firstName} {contact.lastName}
      </Text>
      <Text style={styles.contactPhone}>{contact.phone}</Text>
    </View>
    <TouchableOpacity
      style={styles.favoriteButton}
      onPress={() => onFavoriteToggle(contact.id)}
      hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
    >
      <Text style={{ fontSize: 20 }}>
        {contact.isFavorite ? '⭐' : '☆'}
      </Text>
    </TouchableOpacity>
  </TouchableOpacity>
));

// Contact Detail Modal
const ContactDetailModal = ({ contact, visible, onClose, onCall, onEdit }) => {
  if (!contact) return null;

  return (
    <Modal visible={visible} animationType="slide" presentationStyle="pageSheet">
      <SafeAreaView style={styles.modalContainer}>
        {/* Header */}
        <View style={styles.modalHeader}>
          <TouchableOpacity onPress={onClose} style={styles.closeButton}>
            <Text style={styles.closeText}>✕</Text>
          </TouchableOpacity>
          <TouchableOpacity onPress={() => onEdit(contact)}>
            <Text style={styles.editText}>แก้ไข</Text>
          </TouchableOpacity>
        </View>

        {/* Contact Info */}
        <View style={styles.modalContent}>
          <ContactAvatar contact={contact} size={80} />
          <Text style={styles.modalName}>
            {contact.firstName} {contact.lastName}
          </Text>

          {/* Actions */}
          <View style={styles.actionRow}>
            <TouchableOpacity
              style={styles.actionBtn}
              onPress={() => onCall(contact.phone)}
            >
              <Text style={styles.actionIcon}>📞</Text>
              <Text style={styles.actionLabel}>โทร</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.actionBtn}>
              <Text style={styles.actionIcon}>💬</Text>
              <Text style={styles.actionLabel}>ข้อความ</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.actionBtn}>
              <Text style={styles.actionIcon}>📹</Text>
              <Text style={styles.actionLabel}>วิดีโอ</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.actionBtn}>
              <Text style={styles.actionIcon}>✉️</Text>
              <Text style={styles.actionLabel}>อีเมล</Text>
            </TouchableOpacity>
          </View>

          {/* Details */}
          <View style={styles.detailSection}>
            <View style={styles.detailRow}>
              <Text style={styles.detailLabel}>📱 โทรศัพท์</Text>
              <Text style={styles.detailValue}>{contact.phone}</Text>
            </View>
            {contact.email && (
              <View style={styles.detailRow}>
                <Text style={styles.detailLabel}>📧 อีเมล</Text>
                <Text style={styles.detailValue}>{contact.email}</Text>
              </View>
            )}
          </View>
        </View>
      </SafeAreaView>
    </Modal>
  );
};

// Main Contact List App
const ContactListApp = () => {
  const [allContacts, setAllContacts] = useState<Contact[]>(generateContacts);
  const [searchText, setSearchText] = useState('');
  const [selectedContact, setSelectedContact] = useState<Contact | null>(null);
  const [activeTab, setActiveTab] = useState<'all' | 'favorites'>('all');
  const sectionListRef = useRef<SectionList>(null);

  // Filter contacts
  const filteredContacts = useMemo(() => {
    let contacts = allContacts;

    if (activeTab === 'favorites') {
      contacts = contacts.filter(c => c.isFavorite);
    }

    if (searchText.trim()) {
      const query = searchText.toLowerCase().trim();
      contacts = contacts.filter(c =>
        c.firstName.toLowerCase().includes(query) ||
        c.lastName.toLowerCase().includes(query) ||
        c.phone.includes(query)
      );
    }

    return contacts;
  }, [allContacts, searchText, activeTab]);

  // Group into sections
  const sections = useMemo(
    () => groupContacts(filteredContacts),
    [filteredContacts]
  );

  // Alphabet index
  const alphabetIndex = useMemo(
    () => sections.map(s => s.title),
    [sections]
  );

  const handleFavoriteToggle = useCallback((contactId: string) => {
    setAllContacts(prev =>
      prev.map(c =>
        c.id === contactId ? { ...c, isFavorite: !c.isFavorite } : c
      )
    );
  }, []);

  const handleCall = (phone: string) => {
    Alert.alert('โทรออก', `กำลังโทรไปที่ ${phone}`, [
      { text: 'ยกเลิก', style: 'cancel' },
      { text: 'โทร', onPress: () => console.log('Calling:', phone) },
    ]);
  };

  const scrollToSection = (letter: string) => {
    const sectionIndex = sections.findIndex(s => s.title === letter);
    if (sectionIndex !== -1) {
      sectionListRef.current?.scrollToLocation({
        sectionIndex,
        itemIndex: 0,
        animated: true,
        viewOffset: 0,
      });
    }
  };

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />

      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>ผู้ติดต่อ</Text>
        <TouchableOpacity style={styles.addButton}>
          <Text style={styles.addButtonText}>+ เพิ่ม</Text>
        </TouchableOpacity>
      </View>

      {/* Search */}
      <View style={styles.searchContainer}>
        <Text style={styles.searchIcon}>🔍</Text>
        <TextInput
          value={searchText}
          onChangeText={setSearchText}
          placeholder="ค้นหาผู้ติดต่อ..."
          style={styles.searchInput}
          returnKeyType="search"
          clearButtonMode="while-editing"
          placeholderTextColor="#999"
        />
        {searchText.length > 0 && (
          <TouchableOpacity onPress={() => setSearchText('')}>
            <Text style={styles.clearSearch}>✕</Text>
          </TouchableOpacity>
        )}
      </View>

      {/* Tabs */}
      <View style={styles.tabs}>
        {['all', 'favorites'].map(tab => (
          <TouchableOpacity
            key={tab}
            style={[styles.tab, activeTab === tab && styles.tabActive]}
            onPress={() => setActiveTab(tab as 'all' | 'favorites')}
          >
            <Text style={[styles.tabText, activeTab === tab && styles.tabTextActive]}>
              {tab === 'all' ? `ทั้งหมด (${allContacts.length})` : `⭐ โปรด (${allContacts.filter(c => c.isFavorite).length})`}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Contact List */}
      <View style={{ flex: 1, flexDirection: 'row' }}>
        <SectionList
          ref={sectionListRef}
          sections={sections}
          keyExtractor={item => item.id}
          style={{ flex: 1 }}
          renderItem={({ item }) => (
            <ContactItem
              contact={item}
              onPress={setSelectedContact}
              onFavoriteToggle={handleFavoriteToggle}
            />
          )}
          renderSectionHeader={({ section: { title } }) => (
            <View style={styles.sectionHeader}>
              <Text style={styles.sectionTitle}>{title}</Text>
            </View>
          )}
          ItemSeparatorComponent={() => (
            <View style={styles.separator} />
          )}
          stickySectionHeadersEnabled
          ListEmptyComponent={
            <View style={styles.emptyState}>
              <Text style={styles.emptyIcon}>
                {activeTab === 'favorites' ? '⭐' : '👥'}
              </Text>
              <Text style={styles.emptyTitle}>
                {activeTab === 'favorites'
                  ? 'ยังไม่มีผู้ติดต่อโปรด'
                  : searchText ? 'ไม่พบผู้ติดต่อ' : 'ยังไม่มีผู้ติดต่อ'}
              </Text>
              {searchText ? (
                <TouchableOpacity onPress={() => setSearchText('')}>
                  <Text style={styles.emptyAction}>ล้างการค้นหา</Text>
                </TouchableOpacity>
              ) : null}
            </View>
          }
          contentContainerStyle={{ paddingBottom: 20 }}
        />

        {/* Alphabet Index */}
        {!searchText && (
          <View style={styles.alphabetContainer}>
            {alphabetIndex.map(letter => (
              <TouchableOpacity
                key={letter}
                onPress={() => scrollToSection(letter)}
                style={styles.letterBtn}
              >
                <Text style={styles.letterChar}>{letter}</Text>
              </TouchableOpacity>
            ))}
          </View>
        )}
      </View>

      {/* Contact Detail Modal */}
      <ContactDetailModal
        contact={selectedContact}
        visible={!!selectedContact}
        onClose={() => setSelectedContact(null)}
        onCall={handleCall}
        onEdit={(contact) => {
          console.log('Edit:', contact.id);
          setSelectedContact(null);
        }}
      />
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  header: {
    flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center',
    paddingHorizontal: 16, paddingVertical: 12,
  },
  headerTitle: { fontSize: 28, fontWeight: 'bold', color: '#111' },
  addButton: { backgroundColor: '#007AFF', paddingHorizontal: 14, paddingVertical: 6, borderRadius: 20 },
  addButtonText: { color: '#fff', fontWeight: '600', fontSize: 14 },
  searchContainer: {
    flexDirection: 'row', alignItems: 'center', marginHorizontal: 16,
    backgroundColor: '#f2f2f7', borderRadius: 10, paddingHorizontal: 12, marginBottom: 8,
  },
  searchIcon: { fontSize: 16, marginRight: 6 },
  searchInput: { flex: 1, paddingVertical: 10, fontSize: 15, color: '#111' },
  clearSearch: { fontSize: 14, color: '#999', padding: 4 },
  tabs: { flexDirection: 'row', marginHorizontal: 16, marginBottom: 4, gap: 8 },
  tab: { paddingHorizontal: 14, paddingVertical: 6, borderRadius: 20, backgroundColor: '#f2f2f7' },
  tabActive: { backgroundColor: '#007AFF' },
  tabText: { fontSize: 13, color: '#666', fontWeight: '500' },
  tabTextActive: { color: '#fff' },
  contactItem: {
    flexDirection: 'row', alignItems: 'center', paddingHorizontal: 16, paddingVertical: 10,
  },
  avatar: { justifyContent: 'center', alignItems: 'center', marginRight: 12 },
  avatarText: { color: '#fff', fontWeight: '700' },
  contactInfo: { flex: 1 },
  contactName: { fontSize: 16, fontWeight: '600', color: '#111' },
  contactPhone: { fontSize: 13, color: '#666', marginTop: 1 },
  favoriteButton: { padding: 4 },
  sectionHeader: {
    paddingHorizontal: 16, paddingVertical: 5,
    backgroundColor: '#f2f2f7',
  },
  sectionTitle: { fontSize: 13, fontWeight: '700', color: '#007AFF' },
  separator: { height: 0.5, backgroundColor: '#e0e0e0', marginLeft: 76 },
  alphabetContainer: {
    width: 24, justifyContent: 'center', paddingVertical: 8,
  },
  letterBtn: { padding: 1, alignItems: 'center' },
  letterChar: { fontSize: 10, color: '#007AFF', fontWeight: '600' },
  emptyState: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 40 },
  emptyIcon: { fontSize: 48, marginBottom: 12 },
  emptyTitle: { fontSize: 17, color: '#666', marginBottom: 12, textAlign: 'center' },
  emptyAction: { color: '#007AFF', fontWeight: '600', fontSize: 15 },
  
  // Modal
  modalContainer: { flex: 1, backgroundColor: '#fff' },
  modalHeader: {
    flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center',
    paddingHorizontal: 16, paddingVertical: 12,
    borderBottomWidth: 1, borderBottomColor: '#f0f0f0',
  },
  closeButton: { padding: 4 },
  closeText: { fontSize: 18, color: '#333' },
  editText: { color: '#007AFF', fontWeight: '600', fontSize: 16 },
  modalContent: { alignItems: 'center', padding: 24 },
  modalName: { fontSize: 24, fontWeight: 'bold', color: '#111', marginTop: 12, marginBottom: 24 },
  actionRow: { flexDirection: 'row', gap: 20, marginBottom: 32 },
  actionBtn: { alignItems: 'center', gap: 4 },
  actionIcon: { fontSize: 28 },
  actionLabel: { fontSize: 11, color: '#666' },
  detailSection: { width: '100%', backgroundColor: '#f9f9f9', borderRadius: 12, padding: 16 },
  detailRow: {
    flexDirection: 'row', justifyContent: 'space-between', paddingVertical: 8,
    borderBottomWidth: 1, borderBottomColor: '#f0f0f0',
  },
  detailLabel: { fontSize: 14, color: '#666' },
  detailValue: { fontSize: 14, color: '#111', fontWeight: '500', maxWidth: '60%', textAlign: 'right' },
});

export default ContactListApp;
```

---

## Tips และ Best Practices

### 1. เลือก List Component ให้ถูก

```
< 50 items → ScrollView (ง่ายสุด)
> 50 items → FlatList (virtualized)
มี sections → SectionList
Performance critical → FlashList (Shopify)
```

### 2. keyExtractor ที่ดี

```tsx
// ✅ ใช้ unique, stable ID
keyExtractor={(item) => item.id}
keyExtractor={(item) => String(item.userId)}

// ❌ index (unstable เมื่อ list เปลี่ยน)
keyExtractor={(_, index) => String(index)}
```

### 3. Memoize renderItem

```tsx
// ✅ ป้องกัน unnecessary re-renders
const renderItem = useCallback(
  ({ item }: { item: Product }) => (
    <ProductCard product={item} onPress={handlePress} />
  ),
  [handlePress]
);
```

### 4. getItemLayout เสมอเมื่อทำได้

```tsx
// ✅ เพิ่ม scroll performance และ scrollToIndex
getItemLayout={(_, index) => ({
  length: ITEM_HEIGHT,
  offset: ITEM_HEIGHT * index,
  index,
})}
```

---

## สรุป Part 010

### ได้เรียนรู้

1. **ScrollView** - ง่าย แต่โหลดทุก items
2. **FlatList** - Virtualized, เหมาะกับ list ใหญ่
3. **FlatList Optimization** - getItemLayout, React.memo, windowSize
4. **SectionList** - Grouped data พร้อม section headers
5. **Pull to Refresh** - RefreshControl
6. **Pagination** - onEndReached, custom hook
7. **Empty State** - UX ที่ดีเมื่อไม่มีข้อมูล
8. **Contact List App** - Workshop ครบวงจร

---

## จบ Parts 001-010!

คุณได้เรียนรู้พื้นฐานที่สำคัญของ React Native แล้ว ได้แก่:

| Part | หัวข้อ |
|------|-------|
| 001 | แนะนำ React Native |
| 002 | Setup Environment |
| 003 | สร้าง Project แรก |
| 004 | โครงสร้าง Project |
| 005 | JSX และ Components |
| 006 | Core Components |
| 007 | StyleSheet และ Flexbox |
| 008 | Props และ State |
| 009 | Event Handling |
| 010 | Lists |

**ขั้นตอนถัดไปที่แนะนำ:**
- Part 011: React Navigation
- Part 012: API Integration (Axios/Fetch)
- Part 013: AsyncStorage และ State Persistence
- Part 014: React Native Animations
- Part 015: Camera และ Media
