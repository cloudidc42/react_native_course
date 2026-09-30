# Part 057: Performance Optimization ใน React Native

## ปัญหา Performance ที่พบบ่อย

การทำ performance optimization ต้องเข้าใจก่อนว่าปัญหาคืออะไร และอยู่ที่ไหน

### ปัญหาหลักที่พบบ่อย

```
1. FlatList ที่ render items ช้า
2. Re-render component โดยไม่จำเป็น
3. รูปภาพขนาดใหญ่เกินไป
4. JavaScript thread blocked
5. Animation ที่ไม่ smooth (dropped frames)
6. Memory leaks ทำให้แอปช้าลงเรื่อยๆ
7. Bundle size ใหญ่ เปิดแอปช้า
8. useEffect ที่ทำงานซ้ำโดยไม่จำเป็น
```

### เครื่องมือ Profiling

```bash
# React DevTools Profiler
# Flipper
# Performance Monitor ใน developer menu
# Chrome DevTools สำหรับ JS profiling
```

---

## FlatList Optimization

FlatList เป็น component ที่ใช้บ่อยที่สุดและมักเป็นต้นเหตุของ performance ปัญหา

```typescript
import React, { useState, useCallback, memo, useMemo } from 'react';
import {
  View,
  Text,
  FlatList,
  StyleSheet,
  TouchableOpacity,
  Image,
} from 'react-native';

// ปัญหา: Item component ที่ไม่ได้ optimize
// const BadItemComponent = ({ item, onPress }) => {
//   return (
//     <TouchableOpacity onPress={() => onPress(item.id)}>
//       <Text>{item.title}</Text>
//     </TouchableOpacity>
//   );
// };
// ทุก re-render ของ parent จะ re-render item ทั้งหมด!

// วิธีแก้: ใช้ memo + useCallback
interface Item {
  id: string;
  title: string;
  subtitle: string;
  imageUrl: string;
  price: number;
  category: string;
}

// Good: ใช้ React.memo ป้องกัน unnecessary re-render
const ProductItem = memo(({ item, onPress, onFavorite }: {
  item: Item;
  onPress: (id: string) => void;
  onFavorite: (id: string) => void;
}) => {
  console.log('Rendering item:', item.id); // ดูว่า re-render กี่ครั้ง
  
  return (
    <TouchableOpacity
      style={flatListStyles.item}
      onPress={() => onPress(item.id)}
    >
      <Image
        source={{ uri: item.imageUrl }}
        style={flatListStyles.image}
        // ใช้ resizeMode ที่เหมาะสม
        resizeMode="cover"
        // Progressive loading
        progressiveRenderingEnabled={true}
      />
      <View style={flatListStyles.info}>
        <Text style={flatListStyles.title} numberOfLines={1}>
          {item.title}
        </Text>
        <Text style={flatListStyles.subtitle} numberOfLines={2}>
          {item.subtitle}
        </Text>
        <Text style={flatListStyles.price}>
          ฿{item.price.toLocaleString()}
        </Text>
      </View>
      <TouchableOpacity
        style={flatListStyles.favoriteButton}
        onPress={() => onFavorite(item.id)}
        hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
      >
        <Text>❤️</Text>
      </TouchableOpacity>
    </TouchableOpacity>
  );
});

const OptimizedFlatList: React.FC = () => {
  const [items, setItems] = useState<Item[]>(generateItems(100));
  const [refreshing, setRefreshing] = useState(false);

  // Good: ใช้ useCallback ป้องกัน function recreation
  const handlePress = useCallback((id: string) => {
    console.log('Pressed:', id);
  }, []);

  const handleFavorite = useCallback((id: string) => {
    console.log('Favorite:', id);
  }, []);

  const handleRefresh = useCallback(async () => {
    setRefreshing(true);
    await new Promise(resolve => setTimeout(resolve, 1000));
    setItems(generateItems(100));
    setRefreshing(false);
  }, []);

  // Good: ใช้ keyExtractor ที่ consistent
  const keyExtractor = useCallback((item: Item) => item.id, []);

  // Good: ใช้ getItemLayout ถ้า item มีขนาดคงที่
  const ITEM_HEIGHT = 100;
  const getItemLayout = useCallback(
    (_: any, index: number) => ({
      length: ITEM_HEIGHT,
      offset: ITEM_HEIGHT * index,
      index,
    }),
    []
  );

  // Good: renderItem เป็น useCallback
  const renderItem = useCallback(({ item }: { item: Item }) => (
    <ProductItem
      item={item}
      onPress={handlePress}
      onFavorite={handleFavorite}
    />
  ), [handlePress, handleFavorite]);

  // Good: ListEmptyComponent เป็น useMemo
  const ListEmptyComponent = useMemo(() => (
    <View style={flatListStyles.empty}>
      <Text>ไม่มีสินค้า</Text>
    </View>
  ), []);

  return (
    <FlatList
      data={items}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      
      // Performance props
      getItemLayout={getItemLayout}          // สำคัญมาก! ทำให้ scroll ไปยัง index ได้เร็ว
      initialNumToRender={10}               // render กี่ items แรก
      maxToRenderPerBatch={5}               // render กี่ items ต่อ batch
      windowSize={5}                        // จำนวน window ที่ keep ไว้ใน memory
      removeClippedSubviews={true}          // ลบ items ที่ไม่อยู่ใน viewport (Android)
      updateCellsBatchingPeriod={50}        // ms ระหว่าง batch updates
      
      // Pull to refresh
      refreshing={refreshing}
      onRefresh={handleRefresh}
      
      // ป้องกัน scroll ที่สะดุด
      showsVerticalScrollIndicator={false}
      
      // Horizontal separation
      ItemSeparatorComponent={() => <View style={flatListStyles.separator} />}
      
      ListEmptyComponent={ListEmptyComponent}
    />
  );
};

function generateItems(count: number): Item[] {
  return Array.from({ length: count }, (_, i) => ({
    id: `item-${i}`,
    title: `สินค้า ${i + 1}`,
    subtitle: `รายละเอียดสินค้า ${i + 1} ที่น่าสนใจมาก`,
    imageUrl: `https://picsum.photos/100/100?random=${i}`,
    price: Math.floor(Math.random() * 10000) + 100,
    category: ['electronics', 'clothing', 'food'][i % 3],
  }));
}

const flatListStyles = StyleSheet.create({
  item: {
    flexDirection: 'row',
    padding: 12,
    backgroundColor: 'white',
    height: 100,
  },
  image: {
    width: 76,
    height: 76,
    borderRadius: 8,
    backgroundColor: '#f0f0f0',
  },
  info: { flex: 1, paddingHorizontal: 12 },
  title: { fontSize: 15, fontWeight: '600', color: '#333' },
  subtitle: { fontSize: 13, color: '#888', marginTop: 3 },
  price: { fontSize: 16, fontWeight: 'bold', color: '#2196F3', marginTop: 4 },
  favoriteButton: { justifyContent: 'center', padding: 5 },
  separator: { height: 1, backgroundColor: '#f0f0f0' },
  empty: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 40 },
});

export default OptimizedFlatList;
```

---

## Memoization Strategies

```typescript
import React, { useState, useMemo, useCallback, memo } from 'react';
import { View, Text, TouchableOpacity, TextInput } from 'react-native';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

const products: Product[] = Array.from({ length: 1000 }, (_, i) => ({
  id: `${i}`,
  name: `Product ${i}`,
  price: Math.floor(Math.random() * 10000) + 100,
  category: ['A', 'B', 'C'][i % 3],
  inStock: i % 4 !== 0,
}));

// BAD: คำนวณทุก render
// const BadComponent = ({ category, minPrice }) => {
//   const filtered = products.filter(p => 
//     p.category === category && p.price >= minPrice
//   ); // คำนวณใหม่ทุกครั้งที่ state เปลี่ยน
//   ...
// };

// GOOD: ใช้ useMemo
const FilteredProductList: React.FC = () => {
  const [category, setCategory] = useState('A');
  const [minPrice, setMinPrice] = useState(0);
  const [searchText, setSearchText] = useState('');
  const [counter, setCounter] = useState(0); // state ที่ไม่เกี่ยวกับ filter

  // คำนวณใหม่เฉพาะเมื่อ category, minPrice, หรือ searchText เปลี่ยน
  const filteredProducts = useMemo(() => {
    console.log('Filtering products...'); // ดูว่า run กี่ครั้ง
    
    return products
      .filter(p => {
        if (p.category !== category) return false;
        if (p.price < minPrice) return false;
        if (searchText && !p.name.toLowerCase().includes(searchText.toLowerCase())) return false;
        return true;
      })
      .sort((a, b) => a.price - b.price);
  }, [category, minPrice, searchText]);

  // สถิติที่คำนวณจาก filteredProducts
  const stats = useMemo(() => ({
    count: filteredProducts.length,
    avgPrice: filteredProducts.length > 0
      ? filteredProducts.reduce((sum, p) => sum + p.price, 0) / filteredProducts.length
      : 0,
    inStockCount: filteredProducts.filter(p => p.inStock).length,
  }), [filteredProducts]);

  // callback ที่ stable
  const handleCategoryChange = useCallback((cat: string) => {
    setCategory(cat);
  }, []);

  return (
    <View style={{ flex: 1, padding: 15 }}>
      <Text style={{ fontSize: 16, color: '#888', marginBottom: 10 }}>
        Counter: {counter} (เปลี่ยนแล้ว filter ไม่ recalculate)
      </Text>
      
      <TouchableOpacity
        style={{ backgroundColor: '#2196F3', padding: 10, borderRadius: 8, marginBottom: 10 }}
        onPress={() => setCounter(c => c + 1)}
      >
        <Text style={{ color: 'white' }}>เพิ่ม Counter (ไม่กระทบ filter)</Text>
      </TouchableOpacity>

      <TextInput
        value={searchText}
        onChangeText={setSearchText}
        placeholder="ค้นหาสินค้า..."
        style={{ borderWidth: 1, borderColor: '#ddd', padding: 10, borderRadius: 8, marginBottom: 10 }}
      />

      <View style={{ flexDirection: 'row', marginBottom: 10, gap: 8 }}>
        {['A', 'B', 'C'].map(cat => (
          <TouchableOpacity
            key={cat}
            style={{
              flex: 1,
              padding: 8,
              backgroundColor: category === cat ? '#2196F3' : '#eee',
              borderRadius: 8,
              alignItems: 'center',
            }}
            onPress={() => handleCategoryChange(cat)}
          >
            <Text style={{ color: category === cat ? 'white' : '#333' }}>
              หมวด {cat}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      <View style={{ backgroundColor: '#E3F2FD', padding: 12, borderRadius: 8, marginBottom: 10 }}>
        <Text style={{ color: '#1565C0' }}>
          พบ {stats.count} รายการ | ราคาเฉลี่ย ฿{Math.round(stats.avgPrice).toLocaleString()} | มีสินค้า {stats.inStockCount} ชิ้น
        </Text>
      </View>
    </View>
  );
};
```

---

## Image Optimization

```typescript
import React, { useState } from 'react';
import {
  View,
  Image,
  FlatList,
  StyleSheet,
  Dimensions,
  Text,
} from 'react-native';

const { width } = Dimensions.get('window');

// ใช้ FastImage สำหรับ better performance
// npm install react-native-fast-image
import FastImage from 'react-native-fast-image';

const ImageOptimizationDemo: React.FC = () => {
  const imageUrls = Array.from({ length: 50 }, (_, i) =>
    `https://picsum.photos/400/300?random=${i}`
  );

  // Bad: ไม่มี image caching
  const BadImageComponent = ({ url }: { url: string }) => (
    <Image
      source={{ uri: url }}
      style={{ width: width / 2 - 10, height: 150 }}
      // ไม่มี cache control
    />
  );

  // Good: ใช้ FastImage พร้อม cache
  const GoodImageComponent = ({ url }: { url: string }) => (
    <FastImage
      source={{
        uri: url,
        priority: FastImage.priority.normal,
        cache: FastImage.cacheControl.immutable, // cache ตลอดไป
      }}
      style={{ width: width / 2 - 10, height: 150, borderRadius: 8 }}
      resizeMode={FastImage.resizeMode.cover}
      onLoadStart={() => {}} // ระหว่างโหลด
      onLoad={() => {}}       // โหลดเสร็จ
      onError={() => {}}      // ถ้าโหลดไม่ได้
    />
  );

  // Lazy loading image ด้วย BlurHash หรือ placeholder
  const LazyImage = memo(({ url, index }: { url: string; index: number }) => {
    const [loaded, setLoaded] = useState(false);
    
    return (
      <View style={imgStyles.imageContainer}>
        {!loaded && (
          <View style={[imgStyles.placeholder, { backgroundColor: `hsl(${index * 30}, 50%, 80%)` }]}>
            <Text style={imgStyles.placeholderText}>📷</Text>
          </View>
        )}
        <FastImage
          source={{ uri: url, priority: FastImage.priority.normal }}
          style={[imgStyles.image, { opacity: loaded ? 1 : 0 }]}
          resizeMode={FastImage.resizeMode.cover}
          onLoad={() => setLoaded(true)}
        />
      </View>
    );
  });

  return (
    <FlatList
      data={imageUrls}
      numColumns={2}
      keyExtractor={(_, index) => index.toString()}
      renderItem={({ item, index }) => (
        <LazyImage url={item} index={index} />
      )}
      columnWrapperStyle={imgStyles.columnWrapper}
      contentContainerStyle={imgStyles.listContent}
      // ลด re-render
      removeClippedSubviews={true}
      initialNumToRender={6}
      maxToRenderPerBatch={4}
      windowSize={3}
    />
  );
};

const imgStyles = StyleSheet.create({
  columnWrapper: { justifyContent: 'space-between', paddingHorizontal: 10 },
  listContent: { padding: 5 },
  imageContainer: {
    width: width / 2 - 15,
    height: 150,
    marginBottom: 10,
    borderRadius: 8,
    overflow: 'hidden',
  },
  placeholder: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'center',
    alignItems: 'center',
    borderRadius: 8,
  },
  placeholderText: { fontSize: 30 },
  image: {
    width: '100%',
    height: '100%',
  },
});
```

---

## Bundle Size Reduction

```bash
# วิเคราะห์ bundle size
npx react-native bundle --platform android --dev false --entry-file index.js --bundle-output /tmp/bundle.js --assets-dest /tmp/assets

# ดู bundle size
npx react-native-bundle-visualizer

# หรือใช้ source-map-explorer
npm install -g source-map-explorer
```

### เทคนิคลด Bundle Size

```typescript
// 1. Dynamic imports / Code Splitting
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

// 2. Tree shaking - import เฉพาะที่ต้องการ
// Bad:
// import _ from 'lodash'; // import ทั้ง library
// Good:
import debounce from 'lodash/debounce'; // import เฉพาะ function

// 3. ใช้ date-fns แทน moment.js (เล็กกว่ามาก)
import { format, parseISO } from 'date-fns';
import { th } from 'date-fns/locale';

const formatDate = (dateString: string) => {
  return format(parseISO(dateString), 'dd MMMM yyyy', { locale: th });
};

// 4. Image optimization ก่อน bundle
// - ใช้ WebP format (เล็กกว่า PNG/JPEG)
// - Compress ก่อน include ใน project
// - ใช้ CDN สำหรับ dynamic images

// 5. ใช้ Metro bundler config เพื่อ optimize
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true, // โหลด modules เฉพาะเมื่อต้องการ
      },
    }),
  },
};
```

---

## Workshop: Optimize a Slow App

```typescript
import React, { useState, useCallback, useMemo, memo, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  FlatList,
  StyleSheet,
  TouchableOpacity,
  Switch,
  Platform,
} from 'react-native';
import { InteractionManager } from 'react-native';

interface DataItem {
  id: number;
  name: string;
  category: string;
  value: number;
  active: boolean;
}

// สร้างข้อมูลจำนวนมาก
const generateData = (count: number): DataItem[] =>
  Array.from({ length: count }, (_, i) => ({
    id: i,
    name: `Item ${i + 1}`,
    category: `Cat ${(i % 5) + 1}`,
    value: Math.floor(Math.random() * 1000),
    active: i % 3 !== 0,
  }));

// Stats component ที่ optimize แล้ว
const StatsPanel = memo(({ data }: { data: DataItem[] }) => {
  const stats = useMemo(() => {
    const total = data.length;
    const active = data.filter(d => d.active).length;
    const avgValue = total > 0 ? data.reduce((s, d) => s + d.value, 0) / total : 0;
    const maxValue = total > 0 ? Math.max(...data.map(d => d.value)) : 0;
    
    return { total, active, avgValue: Math.round(avgValue), maxValue };
  }, [data]);

  return (
    <View style={perfStyles.statsPanel}>
      <View style={perfStyles.statItem}>
        <Text style={perfStyles.statValue}>{stats.total}</Text>
        <Text style={perfStyles.statLabel}>ทั้งหมด</Text>
      </View>
      <View style={perfStyles.statItem}>
        <Text style={perfStyles.statValue}>{stats.active}</Text>
        <Text style={perfStyles.statLabel}>Active</Text>
      </View>
      <View style={perfStyles.statItem}>
        <Text style={perfStyles.statValue}>{stats.avgValue}</Text>
        <Text style={perfStyles.statLabel}>เฉลี่ย</Text>
      </View>
      <View style={perfStyles.statItem}>
        <Text style={perfStyles.statValue}>{stats.maxValue}</Text>
        <Text style={perfStyles.statLabel}>สูงสุด</Text>
      </View>
    </View>
  );
});

// Row component ที่ optimize แล้ว
const DataRow = memo(({ item, onToggle, onSelect }: {
  item: DataItem;
  onToggle: (id: number) => void;
  onSelect: (id: number) => void;
}) => (
  <TouchableOpacity
    style={[perfStyles.row, !item.active && perfStyles.inactiveRow]}
    onPress={() => onSelect(item.id)}
  >
    <View style={perfStyles.rowLeft}>
      <Text style={perfStyles.rowName}>{item.name}</Text>
      <Text style={perfStyles.rowCategory}>{item.category}</Text>
    </View>
    <Text style={perfStyles.rowValue}>{item.value}</Text>
    <Switch
      value={item.active}
      onValueChange={() => onToggle(item.id)}
      trackColor={{ false: '#ddd', true: '#4CAF50' }}
    />
  </TouchableOpacity>
));

const PerformanceWorkshop: React.FC = () => {
  const [data, setData] = useState<DataItem[]>([]);
  const [searchText, setSearchText] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('');
  const [showActive, setShowActive] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [renderCount, setRenderCount] = useState(0);

  useEffect(() => {
    // โหลดข้อมูลหลัง interaction เสร็จ (ไม่บล็อก animation)
    const task = InteractionManager.runAfterInteractions(() => {
      const generated = generateData(1000);
      setData(generated);
      setIsLoading(false);
    });

    return () => task.cancel();
  }, []);

  // Track re-renders
  useEffect(() => {
    setRenderCount(c => c + 1);
  });

  // Filtered data ด้วย useMemo
  const filteredData = useMemo(() => {
    return data.filter(item => {
      if (showActive && !item.active) return false;
      if (selectedCategory && item.category !== selectedCategory) return false;
      if (searchText) {
        const query = searchText.toLowerCase();
        if (!item.name.toLowerCase().includes(query) &&
            !item.category.toLowerCase().includes(query)) return false;
      }
      return true;
    });
  }, [data, searchText, selectedCategory, showActive]);

  const categories = useMemo(() => {
    const cats = new Set(data.map(d => d.category));
    return ['', ...Array.from(cats)];
  }, [data]);

  const handleToggle = useCallback((id: number) => {
    setData(prev =>
      prev.map(item =>
        item.id === id ? { ...item, active: !item.active } : item
      )
    );
  }, []);

  const handleSelect = useCallback((id: number) => {
    console.log('Selected:', id);
  }, []);

  const keyExtractor = useCallback((item: DataItem) => item.id.toString(), []);

  const getItemLayout = useCallback(
    (_: any, index: number) => ({ length: 60, offset: 60 * index, index }),
    []
  );

  const renderItem = useCallback(({ item }: { item: DataItem }) => (
    <DataRow item={item} onToggle={handleToggle} onSelect={handleSelect} />
  ), [handleToggle, handleSelect]);

  if (isLoading) {
    return (
      <View style={perfStyles.loading}>
        <Text>กำลังโหลด...</Text>
      </View>
    );
  }

  return (
    <View style={perfStyles.container}>
      <View style={perfStyles.header}>
        <Text style={perfStyles.title}>Performance Demo</Text>
        <Text style={perfStyles.renderCount}>Renders: {renderCount}</Text>
      </View>

      <StatsPanel data={filteredData} />

      <TextInput
        style={perfStyles.search}
        value={searchText}
        onChangeText={setSearchText}
        placeholder="ค้นหา..."
      />

      <View style={perfStyles.filters}>
        <View style={perfStyles.activeFilter}>
          <Text style={perfStyles.filterLabel}>Active เท่านั้น</Text>
          <Switch
            value={showActive}
            onValueChange={setShowActive}
            trackColor={{ false: '#ddd', true: '#2196F3' }}
          />
        </View>
        
        <FlatList
          horizontal
          data={categories}
          keyExtractor={item => item || 'all'}
          renderItem={({ item }) => (
            <TouchableOpacity
              style={[
                perfStyles.categoryChip,
                selectedCategory === item && perfStyles.categoryChipSelected,
              ]}
              onPress={() => setSelectedCategory(item)}
            >
              <Text style={[
                perfStyles.categoryChipText,
                selectedCategory === item && perfStyles.categoryChipTextSelected,
              ]}>
                {item || 'ทั้งหมด'}
              </Text>
            </TouchableOpacity>
          )}
          showsHorizontalScrollIndicator={false}
        />
      </View>

      <Text style={perfStyles.resultCount}>
        แสดง {filteredData.length} รายการ จาก {data.length}
      </Text>

      <FlatList
        data={filteredData}
        keyExtractor={keyExtractor}
        renderItem={renderItem}
        getItemLayout={getItemLayout}
        removeClippedSubviews={Platform.OS === 'android'}
        initialNumToRender={15}
        maxToRenderPerBatch={10}
        windowSize={7}
        updateCellsBatchingPeriod={50}
        showsVerticalScrollIndicator={false}
      />
    </View>
  );
};

const perfStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  loading: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 15,
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  title: { fontSize: 20, fontWeight: 'bold', color: '#333' },
  renderCount: { fontSize: 12, color: '#999' },
  statsPanel: {
    flexDirection: 'row',
    backgroundColor: '#1565C0',
    padding: 15,
  },
  statItem: { flex: 1, alignItems: 'center' },
  statValue: { fontSize: 22, fontWeight: 'bold', color: 'white' },
  statLabel: { fontSize: 11, color: 'rgba(255,255,255,0.7)', marginTop: 2 },
  search: {
    margin: 10,
    backgroundColor: 'white',
    borderRadius: 10,
    padding: 12,
    fontSize: 15,
    elevation: 1,
  },
  filters: {
    backgroundColor: 'white',
    paddingHorizontal: 10,
    paddingBottom: 10,
  },
  activeFilter: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
  },
  filterLabel: { fontSize: 14, color: '#444' },
  categoryChip: {
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 15,
    backgroundColor: '#f0f0f0',
    marginRight: 8,
  },
  categoryChipSelected: { backgroundColor: '#2196F3' },
  categoryChipText: { fontSize: 13, color: '#555' },
  categoryChipTextSelected: { color: 'white', fontWeight: 'bold' },
  resultCount: { padding: 10, fontSize: 12, color: '#888', backgroundColor: 'white' },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    paddingHorizontal: 15,
    paddingVertical: 10,
    height: 60,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  inactiveRow: { opacity: 0.5 },
  rowLeft: { flex: 1 },
  rowName: { fontSize: 14, fontWeight: '500', color: '#333' },
  rowCategory: { fontSize: 12, color: '#888' },
  rowValue: { fontSize: 16, fontWeight: 'bold', color: '#2196F3', marginRight: 15 },
});

export default PerformanceWorkshop;
```

---

## Tips สรุป Performance

### 1. หลัก Memoization
```typescript
// useMemo - สำหรับ expensive calculations
const result = useMemo(() => expensiveCalc(data), [data]);

// useCallback - สำหรับ functions ที่ pass ไปยัง child
const handler = useCallback(() => doSomething(), [dependency]);

// React.memo - สำหรับ component ที่ re-render บ่อย
const MyComp = memo(({ prop }) => <View />);
```

### 2. หลีกเลี่ยง Anti-patterns
```typescript
// Bad: anonymous function ใน render
<Button onPress={() => handlePress(id)} />  // สร้างใหม่ทุก render!

// Good: useCallback
const handlePressWithId = useCallback(() => handlePress(id), [id]);
<Button onPress={handlePressWithId} />

// Bad: object/array literal ใน props
<Component style={{ color: 'red' }} />  // สร้าง object ใหม่ทุก render!

// Good: StyleSheet หรือ useMemo
const style = useMemo(() => ({ color: 'red' }), []);
<Component style={style} />
```

### 3. Profiling
```bash
# เปิด Systrace สำหรับ Android
$ adb shell atrace --async_start -b 16384 gfx input view sched dalvik

# ใช้ Performance Monitor
# Shake device → Performance Monitor
```

---

## สรุป

Performance Optimization สำคัญมากสำหรับ UX:
- **FlatList**: ใช้ memo, getItemLayout, windowSize ที่เหมาะสม
- **Memoization**: useMemo + useCallback + React.memo ลด re-render
- **Images**: FastImage + WebP + lazy loading
- **Bundle Size**: Tree shaking + dynamic imports + image optimization
- **Profiling**: ใช้เครื่องมือก่อน optimize เสมอ (อย่า premature optimization)
