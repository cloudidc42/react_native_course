# Part 054: In-App Purchase ใน React Native

## ความเข้าใจ IAP (In-App Purchase)

In-App Purchase (IAP) คือการซื้อสินค้าหรือบริการดิจิทัลภายในแอปผ่าน App Store (iOS) หรือ Google Play Store (Android) ช่วยให้ developer สามารถสร้างรายได้จากแอปได้หลายรูปแบบ

### ประเภทของ IAP

```
1. Consumable (สิ่งที่ใช้แล้วหมด)
   - เหรียญเกม, คะแนน, Boost items
   - ซื้อได้หลายครั้ง
   - หมดแล้วต้องซื้อใหม่

2. Non-Consumable (ถาวร)
   - ปลดล็อคฟีเจอร์, Remove Ads, Themes
   - ซื้อครั้งเดียว ใช้ได้ตลอด
   - Restore ได้เมื่อเปลี่ยน device

3. Auto-Renewable Subscription
   - Premium membership, Netflix-style
   - ต่ออายุอัตโนมัติ
   - ยกเลิกได้จาก settings

4. Non-Renewing Subscription
   - สมาชิกรายเดือน/ปี (ไม่ต่อเอง)
   - ต้องซื้อใหม่เมื่อหมดอายุ
```

---

## การติดตั้ง react-native-iap

```bash
npm install react-native-iap
cd ios && pod install
```

### ตั้งค่า iOS - Xcode Capabilities

เปิด Xcode → Signing & Capabilities → เพิ่ม "In-App Purchase"

### ตั้งค่า Android

เพิ่มใน `android/app/build.gradle`:
```gradle
dependencies {
    implementation 'com.android.billingclient:billing:6.0.1'
}
```

---

## IAP Manager

```typescript
import * as RNIap from 'react-native-iap';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Platform, Alert } from 'react-native';

// Product IDs - ต้องตรงกับที่ตั้งใน App Store / Play Store
const PRODUCT_IDS = {
  // Consumables
  COINS_100: 'com.yourapp.coins.100',
  COINS_500: 'com.yourapp.coins.500',
  COINS_1000: 'com.yourapp.coins.1000',
  
  // Non-consumables
  REMOVE_ADS: 'com.yourapp.removeads',
  PREMIUM_THEME: 'com.yourapp.theme.premium',
  UNLOCK_ALL: 'com.yourapp.unlock.all',
  
  // Subscriptions
  PREMIUM_MONTHLY: 'com.yourapp.premium.monthly',
  PREMIUM_YEARLY: 'com.yourapp.premium.yearly',
};

interface Purchase {
  productId: string;
  transactionId: string;
  transactionDate: number;
  isAcknowledged: boolean;
}

interface IAPProduct {
  productId: string;
  title: string;
  description: string;
  price: string;
  currency: string;
  type: 'consumable' | 'non-consumable' | 'subscription';
}

class IAPManager {
  private static instance: IAPManager;
  private isConnected = false;
  private purchaseUpdateSubscription: any = null;
  private purchaseErrorSubscription: any = null;

  static getInstance(): IAPManager {
    if (!IAPManager.instance) {
      IAPManager.instance = new IAPManager();
    }
    return IAPManager.instance;
  }

  async connect(): Promise<void> {
    try {
      await RNIap.initConnection();
      this.isConnected = true;
      console.log('[IAP] Connected to store');
    } catch (error: any) {
      console.error('[IAP] Failed to connect:', error);
      throw error;
    }
  }

  async disconnect(): Promise<void> {
    this.purchaseUpdateSubscription?.remove();
    this.purchaseErrorSubscription?.remove();
    await RNIap.endConnection();
    this.isConnected = false;
  }

  setupPurchaseListeners(
    onSuccess: (purchase: RNIap.Purchase) => Promise<void>,
    onError: (error: RNIap.PurchaseError) => void
  ): void {
    this.purchaseUpdateSubscription = RNIap.purchaseUpdatedListener(
      async (purchase: RNIap.SubscriptionPurchase | RNIap.ProductPurchase) => {
        console.log('[IAP] Purchase received:', purchase);
        
        const receipt = purchase.transactionReceipt;
        if (receipt) {
          try {
            // Verify receipt กับ server ก่อน
            await this.verifyReceipt(receipt, purchase.productId);
            
            // ทำ finish transaction
            await RNIap.finishTransaction({ purchase, isConsumable: false });
            
            await onSuccess(purchase);
          } catch (error) {
            console.error('[IAP] Receipt verification failed:', error);
          }
        }
      }
    );

    this.purchaseErrorSubscription = RNIap.purchaseErrorListener(
      (error: RNIap.PurchaseError) => {
        console.error('[IAP] Purchase error:', error);
        onError(error);
      }
    );
  }

  async getProducts(productIds: string[]): Promise<RNIap.Product[]> {
    if (!this.isConnected) await this.connect();
    
    try {
      const products = await RNIap.getProducts({ skus: productIds });
      return products;
    } catch (error) {
      console.error('[IAP] Failed to fetch products:', error);
      return [];
    }
  }

  async getSubscriptions(subscriptionIds: string[]): Promise<RNIap.Subscription[]> {
    if (!this.isConnected) await this.connect();
    
    try {
      const subscriptions = await RNIap.getSubscriptions({ skus: subscriptionIds });
      return subscriptions;
    } catch (error) {
      console.error('[IAP] Failed to fetch subscriptions:', error);
      return [];
    }
  }

  async purchaseProduct(productId: string): Promise<void> {
    try {
      await RNIap.requestPurchase({ sku: productId });
    } catch (error: any) {
      if (error.code !== 'E_USER_CANCELLED') {
        throw error;
      }
    }
  }

  async purchaseSubscription(productId: string): Promise<void> {
    try {
      await RNIap.requestSubscription({ sku: productId });
    } catch (error: any) {
      if (error.code !== 'E_USER_CANCELLED') {
        throw error;
      }
    }
  }

  async restorePurchases(): Promise<RNIap.Purchase[]> {
    if (!this.isConnected) await this.connect();
    
    try {
      const purchases = await RNIap.getAvailablePurchases();
      console.log('[IAP] Restored purchases:', purchases.length);
      return purchases;
    } catch (error) {
      console.error('[IAP] Restore failed:', error);
      return [];
    }
  }

  private async verifyReceipt(receipt: string, productId: string): Promise<boolean> {
    // ส่ง receipt ไปยัง backend ของคุณเพื่อ verify
    try {
      const response = await fetch('https://api.yourapp.com/iap/verify', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await this.getAuthToken()}`,
        },
        body: JSON.stringify({
          receipt,
          productId,
          platform: Platform.OS,
        }),
      });

      const data = await response.json();
      return data.valid;
    } catch (error) {
      // ถ้าไม่สามารถ verify ได้ อย่าปฏิเสธการซื้อ
      // แต่ควร log เพื่อตรวจสอบภายหลัง
      console.error('[IAP] Verify error:', error);
      return true; // Optimistic approach
    }
  }

  private async getAuthToken(): Promise<string> {
    return await AsyncStorage.getItem('@auth_token') || '';
  }
}

export { IAPManager, PRODUCT_IDS, IAPProduct };
```

---

## IAP Store Component

```typescript
import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator,
  Modal,
  ScrollView,
} from 'react-native';
import * as RNIap from 'react-native-iap';
import { IAPManager, PRODUCT_IDS } from './IAPManager';

interface StoreItem {
  id: string;
  title: string;
  description: string;
  price: string;
  type: 'consumable' | 'non-consumable' | 'subscription';
  badge?: string;
  icon: string;
}

const IAPStoreScreen: React.FC = () => {
  const [products, setProducts] = useState<RNIap.Product[]>([]);
  const [subscriptions, setSubscriptions] = useState<RNIap.Subscription[]>([]);
  const [loading, setLoading] = useState(true);
  const [purchasing, setPurchasing] = useState<string | null>(null);
  const [userCoins, setUserCoins] = useState(0);
  const [isPremium, setIsPremium] = useState(false);
  const [showReceipt, setShowReceipt] = useState<string | null>(null);
  
  const iapManager = IAPManager.getInstance();

  useEffect(() => {
    initIAP();
    
    return () => {
      iapManager.disconnect();
    };
  }, []);

  const initIAP = async () => {
    setLoading(true);
    try {
      await iapManager.connect();
      
      // Setup listeners
      iapManager.setupPurchaseListeners(
        handlePurchaseSuccess,
        handlePurchaseError
      );

      // โหลด products
      const [fetchedProducts, fetchedSubs] = await Promise.all([
        iapManager.getProducts([
          PRODUCT_IDS.COINS_100,
          PRODUCT_IDS.COINS_500,
          PRODUCT_IDS.COINS_1000,
          PRODUCT_IDS.REMOVE_ADS,
          PRODUCT_IDS.UNLOCK_ALL,
        ]),
        iapManager.getSubscriptions([
          PRODUCT_IDS.PREMIUM_MONTHLY,
          PRODUCT_IDS.PREMIUM_YEARLY,
        ]),
      ]);

      setProducts(fetchedProducts);
      setSubscriptions(fetchedSubs);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถโหลด store ได้');
    } finally {
      setLoading(false);
    }
  };

  const handlePurchaseSuccess = async (purchase: RNIap.Purchase) => {
    const { productId } = purchase;
    
    // จัดการตาม product type
    switch (productId) {
      case PRODUCT_IDS.COINS_100:
        setUserCoins(prev => prev + 100);
        Alert.alert('สำเร็จ!', 'คุณได้รับ 100 เหรียญ 🎉');
        break;
        
      case PRODUCT_IDS.COINS_500:
        setUserCoins(prev => prev + 500);
        Alert.alert('สำเร็จ!', 'คุณได้รับ 500 เหรียญ 🎉');
        break;
        
      case PRODUCT_IDS.COINS_1000:
        setUserCoins(prev => prev + 1000);
        Alert.alert('สำเร็จ!', 'คุณได้รับ 1,000 เหรียญ 🎉');
        break;
        
      case PRODUCT_IDS.REMOVE_ADS:
      case PRODUCT_IDS.UNLOCK_ALL:
      case PRODUCT_IDS.PREMIUM_MONTHLY:
      case PRODUCT_IDS.PREMIUM_YEARLY:
        setIsPremium(true);
        Alert.alert('ยินดีต้อนรับสู่ Premium!', 'คุณได้ปลดล็อคฟีเจอร์ทั้งหมดแล้ว ✨');
        break;
    }
    
    setShowReceipt(purchase.transactionId || null);
    setPurchasing(null);
  };

  const handlePurchaseError = (error: RNIap.PurchaseError) => {
    setPurchasing(null);
    
    if (error.code === 'E_USER_CANCELLED') return;
    
    Alert.alert(
      'ไม่สามารถซื้อได้',
      error.message || 'เกิดข้อผิดพลาด กรุณาลองใหม่'
    );
  };

  const handlePurchase = async (productId: string, isSubscription = false) => {
    if (purchasing) return;
    
    setPurchasing(productId);
    try {
      if (isSubscription) {
        await iapManager.purchaseSubscription(productId);
      } else {
        await iapManager.purchaseProduct(productId);
      }
    } catch (error: any) {
      setPurchasing(null);
      if (error.code !== 'E_USER_CANCELLED') {
        Alert.alert('ข้อผิดพลาด', error.message);
      }
    }
  };

  const handleRestore = async () => {
    setLoading(true);
    try {
      const restoredPurchases = await iapManager.restorePurchases();
      
      if (restoredPurchases.length === 0) {
        Alert.alert('ไม่พบการซื้อ', 'ไม่พบการซื้อก่อนหน้าที่สามารถ restore ได้');
        return;
      }
      
      // ตรวจสอบ purchases ที่ restore มา
      let restoredPremium = false;
      for (const purchase of restoredPurchases) {
        if ([PRODUCT_IDS.REMOVE_ADS, PRODUCT_IDS.UNLOCK_ALL].includes(purchase.productId)) {
          restoredPremium = true;
        }
      }
      
      if (restoredPremium) {
        setIsPremium(true);
        Alert.alert('Restore สำเร็จ', 'คืนค่าการซื้อทั้งหมดแล้ว');
      }
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถ restore ได้');
    } finally {
      setLoading(false);
    }
  };

  // Mock store items เมื่อ store ไม่พร้อม
  const storeItems: StoreItem[] = [
    {
      id: PRODUCT_IDS.COINS_100,
      title: '100 เหรียญ',
      description: 'แพ็คเริ่มต้น',
      price: '฿35',
      type: 'consumable',
      icon: '🪙',
    },
    {
      id: PRODUCT_IDS.COINS_500,
      title: '500 เหรียญ',
      description: 'แพ็คคุ้มค่า',
      price: '฿149',
      type: 'consumable',
      icon: '💰',
      badge: 'ยอดนิยม',
    },
    {
      id: PRODUCT_IDS.COINS_1000,
      title: '1,000 เหรียญ',
      description: 'แพ็คสุดคุ้ม',
      price: '฿249',
      type: 'consumable',
      icon: '💎',
      badge: 'สุดคุ้ม',
    },
    {
      id: PRODUCT_IDS.REMOVE_ADS,
      title: 'ปิดโฆษณา',
      description: 'ใช้แอปไม่มีโฆษณาตลอดไป',
      price: '฿79',
      type: 'non-consumable',
      icon: '🚫',
    },
    {
      id: PRODUCT_IDS.UNLOCK_ALL,
      title: 'ปลดล็อคทั้งหมด',
      description: 'เข้าถึงฟีเจอร์ทั้งหมดตลอดไป',
      price: '฿199',
      type: 'non-consumable',
      icon: '🔓',
      badge: 'ดีที่สุด',
    },
    {
      id: PRODUCT_IDS.PREMIUM_MONTHLY,
      title: 'Premium รายเดือน',
      description: 'สมาชิก Premium 1 เดือน',
      price: '฿99/เดือน',
      type: 'subscription',
      icon: '⭐',
    },
    {
      id: PRODUCT_IDS.PREMIUM_YEARLY,
      title: 'Premium รายปี',
      description: 'สมาชิก Premium 1 ปี ประหยัด 50%',
      price: '฿599/ปี',
      type: 'subscription',
      icon: '👑',
      badge: 'ประหยัดสุด',
    },
  ];

  const renderStoreItem = ({ item }: { item: StoreItem }) => {
    const isPurchased = item.type === 'non-consumable' && isPremium;
    const isPurchasing = purchasing === item.id;
    
    return (
      <View style={storeStyles.itemCard}>
        {item.badge && (
          <View style={storeStyles.badge}>
            <Text style={storeStyles.badgeText}>{item.badge}</Text>
          </View>
        )}
        
        <View style={storeStyles.itemHeader}>
          <Text style={storeStyles.itemIcon}>{item.icon}</Text>
          <View style={storeStyles.itemInfo}>
            <Text style={storeStyles.itemTitle}>{item.title}</Text>
            <Text style={storeStyles.itemDesc}>{item.description}</Text>
            <Text style={storeStyles.itemType}>
              {item.type === 'consumable' ? 'สินค้า' :
               item.type === 'non-consumable' ? 'ถาวร' : 'สมาชิก'}
            </Text>
          </View>
          <View style={storeStyles.priceSection}>
            <Text style={storeStyles.itemPrice}>{item.price}</Text>
            <TouchableOpacity
              style={[
                storeStyles.buyButton,
                isPurchased && storeStyles.buyButtonOwned,
                isPurchasing && storeStyles.buyButtonLoading,
              ]}
              onPress={() => !isPurchased && handlePurchase(
                item.id,
                item.type === 'subscription'
              )}
              disabled={isPurchased || isPurchasing}
            >
              {isPurchasing ? (
                <ActivityIndicator size="small" color="white" />
              ) : (
                <Text style={storeStyles.buyButtonText}>
                  {isPurchased ? '✓ มีแล้ว' : 'ซื้อ'}
                </Text>
              )}
            </TouchableOpacity>
          </View>
        </View>
      </View>
    );
  };

  if (loading) {
    return (
      <View style={storeStyles.loadingContainer}>
        <ActivityIndicator size="large" color="#FF6B35" />
        <Text style={storeStyles.loadingText}>กำลังโหลด Store...</Text>
      </View>
    );
  }

  return (
    <View style={storeStyles.container}>
      {/* Header */}
      <View style={storeStyles.header}>
        <View style={storeStyles.coinsDisplay}>
          <Text style={storeStyles.coinsIcon}>🪙</Text>
          <Text style={storeStyles.coinsText}>{userCoins.toLocaleString()}</Text>
        </View>
        {isPremium && (
          <View style={storeStyles.premiumBadge}>
            <Text style={storeStyles.premiumText}>👑 Premium</Text>
          </View>
        )}
      </View>

      <FlatList
        data={storeItems}
        keyExtractor={item => item.id}
        renderItem={renderStoreItem}
        contentContainerStyle={storeStyles.listContent}
        ListHeaderComponent={
          <Text style={storeStyles.sectionTitle}>ร้านค้าในแอป</Text>
        }
        ListFooterComponent={
          <TouchableOpacity
            style={storeStyles.restoreButton}
            onPress={handleRestore}
          >
            <Text style={storeStyles.restoreText}>คืนค่าการซื้อ (Restore Purchases)</Text>
          </TouchableOpacity>
        }
      />

      {/* Receipt Modal */}
      <Modal
        visible={!!showReceipt}
        transparent
        animationType="fade"
        onRequestClose={() => setShowReceipt(null)}
      >
        <View style={storeStyles.receiptOverlay}>
          <View style={storeStyles.receiptModal}>
            <Text style={storeStyles.receiptSuccess}>✅</Text>
            <Text style={storeStyles.receiptTitle}>ซื้อสำเร็จ!</Text>
            <Text style={storeStyles.receiptId}>
              Transaction ID: {showReceipt}
            </Text>
            <TouchableOpacity
              style={storeStyles.receiptClose}
              onPress={() => setShowReceipt(null)}
            >
              <Text style={storeStyles.receiptCloseText}>ปิด</Text>
            </TouchableOpacity>
          </View>
        </View>
      </Modal>
    </View>
  );
};

const storeStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#1a1a2e' },
  loadingContainer: { flex: 1, justifyContent: 'center', alignItems: 'center', backgroundColor: '#1a1a2e' },
  loadingText: { color: 'white', marginTop: 15, fontSize: 16 },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 20,
    paddingBottom: 10,
  },
  coinsDisplay: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'rgba(255,255,255,0.1)',
    paddingHorizontal: 14,
    paddingVertical: 8,
    borderRadius: 20,
  },
  coinsIcon: { fontSize: 18, marginRight: 6 },
  coinsText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  premiumBadge: {
    backgroundColor: '#FFD700',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 15,
  },
  premiumText: { color: '#1a1a2e', fontWeight: 'bold', fontSize: 13 },
  listContent: { padding: 15, paddingTop: 5 },
  sectionTitle: { fontSize: 22, fontWeight: 'bold', color: 'white', marginBottom: 15 },
  itemCard: {
    backgroundColor: 'rgba(255,255,255,0.1)',
    borderRadius: 12,
    padding: 15,
    marginBottom: 12,
    position: 'relative',
  },
  badge: {
    position: 'absolute',
    top: -8,
    right: 15,
    backgroundColor: '#FF6B35',
    paddingHorizontal: 10,
    paddingVertical: 3,
    borderRadius: 10,
    zIndex: 1,
  },
  badgeText: { color: 'white', fontSize: 11, fontWeight: 'bold' },
  itemHeader: { flexDirection: 'row', alignItems: 'center' },
  itemIcon: { fontSize: 36, marginRight: 12 },
  itemInfo: { flex: 1 },
  itemTitle: { fontSize: 16, fontWeight: 'bold', color: 'white' },
  itemDesc: { fontSize: 13, color: '#aaa', marginTop: 2 },
  itemType: {
    fontSize: 11,
    color: '#666',
    marginTop: 4,
    textTransform: 'uppercase',
  },
  priceSection: { alignItems: 'flex-end' },
  itemPrice: { color: '#FFD700', fontWeight: 'bold', fontSize: 15, marginBottom: 6 },
  buyButton: {
    backgroundColor: '#FF6B35',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    minWidth: 60,
    alignItems: 'center',
  },
  buyButtonOwned: { backgroundColor: '#4CAF50' },
  buyButtonLoading: { opacity: 0.7 },
  buyButtonText: { color: 'white', fontWeight: 'bold', fontSize: 14 },
  restoreButton: {
    padding: 15,
    alignItems: 'center',
    marginTop: 10,
  },
  restoreText: { color: '#888', fontSize: 14, textDecorationLine: 'underline' },
  receiptOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.7)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  receiptModal: {
    backgroundColor: 'white',
    borderRadius: 20,
    padding: 30,
    alignItems: 'center',
    width: 280,
  },
  receiptSuccess: { fontSize: 60, marginBottom: 10 },
  receiptTitle: { fontSize: 24, fontWeight: 'bold', color: '#333', marginBottom: 10 },
  receiptId: { fontSize: 12, color: '#888', textAlign: 'center', marginBottom: 20 },
  receiptClose: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 30,
    paddingVertical: 12,
    borderRadius: 25,
  },
  receiptCloseText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
});

export default IAPStoreScreen;
```

---

## Subscription Management

```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Alert, Linking } from 'react-native';
import * as RNIap from 'react-native-iap';
import { Platform } from 'react-native';

interface SubscriptionInfo {
  productId: string;
  expiresDate: Date | null;
  isActive: boolean;
  autoRenewing: boolean;
}

const SubscriptionScreen: React.FC = () => {
  const [subscription, setSubscription] = useState<SubscriptionInfo | null>(null);
  const [loading, setLoading] = useState(true);

  const checkSubscription = async () => {
    try {
      const purchases = await RNIap.getAvailablePurchases();
      
      const subPurchase = purchases.find(p =>
        p.productId === 'com.yourapp.premium.monthly' ||
        p.productId === 'com.yourapp.premium.yearly'
      );
      
      if (subPurchase) {
        // ตรวจสอบ expiry date
        let expiresDate: Date | null = null;
        
        if (Platform.OS === 'ios' && subPurchase.transactionReceipt) {
          // iOS: ต้อง decode receipt เพื่อดู expiry
          // ในระบบจริงต้องส่ง receipt ไปยัง backend
          expiresDate = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // Mock
        } else if (Platform.OS === 'android') {
          // Android: ใช้ purchaseStateAndroid
          const isSubscribed = subPurchase.purchaseStateAndroid === RNIap.PurchaseStateAndroid.PURCHASED;
          if (isSubscribed) {
            expiresDate = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // Mock
          }
        }
        
        setSubscription({
          productId: subPurchase.productId,
          expiresDate,
          isActive: expiresDate ? expiresDate > new Date() : false,
          autoRenewing: true,
        });
      }
    } catch (error) {
      console.error('Failed to check subscription:', error);
    } finally {
      setLoading(false);
    }
  };

  const openSubscriptionManagement = () => {
    if (Platform.OS === 'ios') {
      Linking.openURL('https://apps.apple.com/account/subscriptions');
    } else {
      Linking.openURL('https://play.google.com/store/account/subscriptions');
    }
  };

  useEffect(() => {
    checkSubscription();
  }, []);

  if (loading) {
    return (
      <View style={subStyles.container}>
        <Text>กำลังตรวจสอบ subscription...</Text>
      </View>
    );
  }

  return (
    <View style={subStyles.container}>
      <Text style={subStyles.title}>จัดการ Subscription</Text>

      {subscription?.isActive ? (
        <View style={subStyles.activeCard}>
          <Text style={subStyles.activeTitle}>✅ Premium Active</Text>
          <Text style={subStyles.activeProduct}>{subscription.productId}</Text>
          {subscription.expiresDate && (
            <Text style={subStyles.expiresText}>
              หมดอายุ: {subscription.expiresDate.toLocaleDateString('th-TH')}
            </Text>
          )}
          <Text style={subStyles.autoRenew}>
            {subscription.autoRenewing ? '🔄 ต่ออายุอัตโนมัติ' : '⚠️ ไม่ต่ออายุอัตโนมัติ'}
          </Text>
          
          <TouchableOpacity
            style={subStyles.manageButton}
            onPress={openSubscriptionManagement}
          >
            <Text style={subStyles.manageButtonText}>จัดการ Subscription</Text>
          </TouchableOpacity>
        </View>
      ) : (
        <View style={subStyles.inactiveCard}>
          <Text style={subStyles.inactiveTitle}>ไม่มี Active Subscription</Text>
          <Text style={subStyles.inactiveDesc}>
            สมัครสมาชิก Premium เพื่อเข้าถึงฟีเจอร์ทั้งหมด
          </Text>
        </View>
      )}
    </View>
  );
};

const subStyles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#f5f5f5' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20, color: '#333' },
  activeCard: {
    backgroundColor: '#E8F5E9',
    borderRadius: 15,
    padding: 20,
    borderWidth: 2,
    borderColor: '#4CAF50',
  },
  activeTitle: { fontSize: 20, fontWeight: 'bold', color: '#2E7D32', marginBottom: 8 },
  activeProduct: { fontSize: 14, color: '#555', marginBottom: 5 },
  expiresText: { fontSize: 14, color: '#666', marginBottom: 5 },
  autoRenew: { fontSize: 14, color: '#555', marginBottom: 15 },
  manageButton: {
    backgroundColor: '#4CAF50',
    padding: 12,
    borderRadius: 10,
    alignItems: 'center',
  },
  manageButtonText: { color: 'white', fontWeight: 'bold', fontSize: 15 },
  inactiveCard: {
    backgroundColor: '#FFF3E0',
    borderRadius: 15,
    padding: 20,
    borderWidth: 2,
    borderColor: '#FF9800',
    alignItems: 'center',
  },
  inactiveTitle: { fontSize: 18, fontWeight: 'bold', color: '#E65100', marginBottom: 8 },
  inactiveDesc: { fontSize: 14, color: '#666', textAlign: 'center' },
});

export default SubscriptionScreen;
```

---

## Tips และ Best Practices

### 1. Server-side Receipt Validation
```typescript
// ส่งเสมอ receipt ไปยัง server เพื่อ validate
// อย่า trust client-side validation เพียงอย่างเดียว
const validatePurchase = async (receipt: string, platform: string) => {
  const endpoint = platform === 'ios'
    ? 'https://buy.itunes.apple.com/verifyReceipt'  // Production
    : 'https://your-server.com/validate-android-purchase';
    
  // ส่ง receipt ไป backend ของคุณ แล้วให้ backend ติดต่อ Apple/Google
};
```

### 2. Handle Network Errors
```typescript
// รอก่อนยกเลิก purchase ถ้าไม่มี network
const purchaseWithRetry = async (productId: string, maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await RNIap.requestPurchase({ sku: productId });
      return;
    } catch (error: any) {
      if (error.code === 'E_USER_CANCELLED') throw error;
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * (i + 1)));
    }
  }
};
```

### 3. Testing IAP
- iOS: ใช้ Sandbox accounts ที่สร้างใน App Store Connect
- Android: ใช้ License Testing accounts ใน Google Play Console
- ทดสอบ edge cases: network failure, cancel, duplicate purchase

---

## สรุป

IAP เป็นส่วนสำคัญของ revenue model สำหรับแอป:
- **Consumables**: สำหรับ items ที่ใช้ซ้ำๆ ได้
- **Non-consumables**: สำหรับ features ถาวร
- **Subscriptions**: สำหรับ ongoing service
- **Server Validation**: สำคัญมากสำหรับความปลอดภัย
- **UX**: ทำให้กระบวนการซื้อง่ายและโปร่งใส
