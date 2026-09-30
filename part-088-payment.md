# Part 088: Payment Integration ใน React Native

## บทนำ

การรับชำระเงินเป็นส่วนสำคัญของ e-commerce app เราจะเรียนรู้การ integrate กับ payment gateways หลัก ได้แก่ Stripe, Omise (popular ในไทย), PayPal, และ Apple Pay/Google Pay

## สิ่งที่จะเรียนรู้

1. Stripe Payment
2. Omise สำหรับตลาดไทย
3. PayPal Integration
4. Apple Pay และ Google Pay
5. Workshop: E-commerce Payment Flow

---

## 1. Stripe Integration

### การติดตั้ง

```bash
npm install @stripe/stripe-react-native
cd ios && pod install
```

### ตั้งค่า iOS (AppDelegate.m)

```objc
#import <Stripe/Stripe.h>

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
  didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  [StripeAPI setDefaultPublishableKey:@"pk_test_YOUR_KEY"];
  // ...
}
```

### ตั้งค่า Android

```kotlin
// android/app/src/main/AndroidManifest.xml
<activity android:name=".MainActivity" ...>
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="stripesdk" android:host="3ds.stripeintent" />
  </intent-filter>
</activity>
```

### App.tsx - Stripe Provider

```typescript
import { StripeProvider } from '@stripe/stripe-react-native';

export default function App() {
  return (
    <StripeProvider
      publishableKey="pk_test_YOUR_STRIPE_PUBLISHABLE_KEY"
      merchantIdentifier="merchant.com.yourapp" // สำหรับ Apple Pay
      urlScheme="yourapp" // สำหรับ 3D Secure
    >
      <NavigationContainer>
        {/* ... */}
      </NavigationContainer>
    </StripeProvider>
  );
}
```

### hooks/useStripePayment.ts

```typescript
import { useState } from 'react';
import {
  useStripe,
  usePaymentSheet,
  PaymentSheetError,
} from '@stripe/stripe-react-native';

interface PaymentParams {
  amount: number; // ใน satang (สตางค์)
  currency: string;
  description: string;
  customerId?: string;
}

export const useStripePayment = () => {
  const { initPaymentSheet, presentPaymentSheet } = usePaymentSheet();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // ดึง payment intent จาก backend
  const fetchPaymentSheetParams = async (params: PaymentParams) => {
    const response = await fetch('https://api.yourapp.com/payment/stripe/create-intent', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${await getAuthToken()}`,
      },
      body: JSON.stringify({
        amount: params.amount,
        currency: params.currency,
        description: params.description,
      }),
    });
    
    const { paymentIntent, ephemeralKey, customer } = await response.json();
    return { paymentIntent, ephemeralKey, customer };
  };

  const initializePayment = async (params: PaymentParams) => {
    setLoading(true);
    setError(null);
    
    try {
      const { paymentIntent, ephemeralKey, customer } = 
        await fetchPaymentSheetParams(params);
      
      const { error } = await initPaymentSheet({
        merchantDisplayName: 'ร้านค้าของเรา',
        customerId: customer,
        customerEphemeralKeySecret: ephemeralKey,
        paymentIntentClientSecret: paymentIntent,
        allowsDelayedPaymentMethods: true,
        defaultBillingDetails: {
          name: 'ชื่อลูกค้า',
        },
        applePay: {
          merchantCountryCode: 'TH',
        },
        googlePay: {
          merchantCountryCode: 'TH',
          testEnv: true, // เปลี่ยนเป็น false ใน production
        },
        style: 'automatic',
        returnURL: 'yourapp://stripe-redirect',
      });
      
      if (error) {
        throw new Error(error.message);
      }
      
      return true;
    } catch (err: any) {
      setError(err.message);
      return false;
    } finally {
      setLoading(false);
    }
  };

  const presentPayment = async (): Promise<boolean> => {
    const { error } = await presentPaymentSheet();
    
    if (error) {
      if (error.code === PaymentSheetError.Canceled) {
        return false; // ผู้ใช้ยกเลิก
      }
      setError(error.message);
      return false;
    }
    
    return true;
  };

  const pay = async (params: PaymentParams): Promise<boolean> => {
    const initialized = await initializePayment(params);
    if (!initialized) return false;
    
    return await presentPayment();
  };

  return {
    loading,
    error,
    initializePayment,
    presentPayment,
    pay,
  };
};
```

### screens/CheckoutScreen.tsx (Stripe)

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Alert,
  ActivityIndicator,
} from 'react-native';
import { useStripePayment } from '../hooks/useStripePayment';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
  image: string;
}

interface CheckoutScreenProps {
  cartItems: CartItem[];
  onSuccess: (paymentId: string) => void;
}

const CheckoutScreen: React.FC<CheckoutScreenProps> = ({ cartItems, onSuccess }) => {
  const { loading, error, pay } = useStripePayment();
  const [isProcessing, setIsProcessing] = useState(false);

  const subtotal = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const shipping = 50;
  const total = subtotal + shipping;

  const handlePayment = async () => {
    setIsProcessing(true);
    
    try {
      const success = await pay({
        amount: total * 100, // แปลงเป็น satang
        currency: 'thb',
        description: `สั่งซื้อสินค้า ${cartItems.length} รายการ`,
      });
      
      if (success) {
        Alert.alert(
          'ชำระเงินสำเร็จ! 🎉',
          'ขอบคุณที่ซื้อสินค้า',
          [{ text: 'ตกลง', onPress: () => onSuccess('payment_id') }]
        );
      }
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>ตรวจสอบคำสั่งซื้อ</Text>
      
      {/* Cart Items */}
      <View style={styles.card}>
        <Text style={styles.sectionTitle}>รายการสินค้า</Text>
        {cartItems.map(item => (
          <View key={item.id} style={styles.cartItem}>
            <Text style={styles.itemName}>{item.name}</Text>
            <Text style={styles.itemQty}>x{item.quantity}</Text>
            <Text style={styles.itemPrice}>
              ฿{(item.price * item.quantity).toLocaleString()}
            </Text>
          </View>
        ))}
      </View>
      
      {/* Price Summary */}
      <View style={styles.card}>
        <Text style={styles.sectionTitle}>สรุปราคา</Text>
        <View style={styles.priceRow}>
          <Text style={styles.priceLabel}>ราคาสินค้า</Text>
          <Text style={styles.priceValue}>฿{subtotal.toLocaleString()}</Text>
        </View>
        <View style={styles.priceRow}>
          <Text style={styles.priceLabel}>ค่าจัดส่ง</Text>
          <Text style={styles.priceValue}>฿{shipping.toLocaleString()}</Text>
        </View>
        <View style={[styles.priceRow, styles.totalRow]}>
          <Text style={styles.totalLabel}>ยอดรวม</Text>
          <Text style={styles.totalValue}>฿{total.toLocaleString()}</Text>
        </View>
      </View>
      
      {/* Error */}
      {error && (
        <View style={styles.errorBox}>
          <Text style={styles.errorText}>{error}</Text>
        </View>
      )}
      
      {/* Payment Methods */}
      <TouchableOpacity
        style={styles.payButton}
        onPress={handlePayment}
        disabled={isProcessing || loading}
      >
        {isProcessing || loading ? (
          <ActivityIndicator color="white" />
        ) : (
          <>
            <Text style={styles.payButtonText}>ชำระเงิน</Text>
            <Text style={styles.payButtonAmount}>฿{total.toLocaleString()}</Text>
          </>
        )}
      </TouchableOpacity>
      
      <Text style={styles.secureText}>🔒 การชำระเงินปลอดภัยด้วย SSL</Text>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5', padding: 16 },
  title: { fontSize: 24, fontWeight: 'bold', color: '#333', marginBottom: 16 },
  card: {
    backgroundColor: 'white', borderRadius: 12, padding: 16, marginBottom: 16,
    shadowColor: '#000', shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1, shadowRadius: 4, elevation: 2,
  },
  sectionTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 12 },
  cartItem: { flexDirection: 'row', alignItems: 'center', paddingVertical: 8,
    borderBottomWidth: 1, borderBottomColor: '#f0f0f0' },
  itemName: { flex: 1, fontSize: 14, color: '#333' },
  itemQty: { fontSize: 14, color: '#999', marginHorizontal: 8 },
  itemPrice: { fontSize: 14, fontWeight: '500', color: '#333' },
  priceRow: { flexDirection: 'row', justifyContent: 'space-between', paddingVertical: 6 },
  priceLabel: { fontSize: 14, color: '#666' },
  priceValue: { fontSize: 14, color: '#333' },
  totalRow: { borderTopWidth: 1, borderTopColor: '#f0f0f0', paddingTop: 12, marginTop: 6 },
  totalLabel: { fontSize: 16, fontWeight: 'bold', color: '#333' },
  totalValue: { fontSize: 18, fontWeight: 'bold', color: '#2196F3' },
  errorBox: { backgroundColor: '#ffebee', padding: 12, borderRadius: 8, marginBottom: 16 },
  errorText: { color: '#c62828' },
  payButton: {
    backgroundColor: '#2196F3', padding: 20, borderRadius: 12,
    flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center',
  },
  payButtonText: { color: 'white', fontSize: 18, fontWeight: '600' },
  payButtonAmount: { color: 'white', fontSize: 18, fontWeight: '600' },
  secureText: { textAlign: 'center', color: '#999', marginTop: 12, marginBottom: 32 },
});

export default CheckoutScreen;
```

---

## 2. Omise Payment (ไทย)

Omise เป็น payment gateway ที่ได้รับความนิยมในไทย รองรับบัตรเครดิต, PromptPay, และ TrueMoney

### การติดตั้ง

```bash
npm install omise-react-native
```

### hooks/useOmisePayment.ts

```typescript
import { useState } from 'react';
import { Platform } from 'react-native';
import OmiseManager from 'omise-react-native';

const OMISE_PUBLIC_KEY = 'pkey_test_YOUR_KEY';

export const useOmisePayment = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // สร้าง token จากบัตรเครดิต
  const createCardToken = async (cardDetails: {
    name: string;
    number: string;
    expirationMonth: string;
    expirationYear: string;
    securityCode: string;
  }): Promise<string | null> => {
    setLoading(true);
    setError(null);
    
    try {
      // ใช้ OmiseJS SDK
      const response = await fetch('https://vault.omise.co/tokens', {
        method: 'POST',
        headers: {
          'Authorization': `Basic ${btoa(OMISE_PUBLIC_KEY + ':')}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          card: {
            name: cardDetails.name,
            number: cardDetails.number,
            expiration_month: parseInt(cardDetails.expirationMonth),
            expiration_year: parseInt(cardDetails.expirationYear),
            security_code: cardDetails.securityCode,
          }
        }),
      });
      
      const data = await response.json();
      
      if (data.object === 'error') {
        throw new Error(data.message);
      }
      
      return data.id; // Token ID
    } catch (err: any) {
      setError(err.message);
      return null;
    } finally {
      setLoading(false);
    }
  };

  // ชำระเงินด้วย PromptPay
  const payWithPromptPay = async (amount: number): Promise<string | null> => {
    setLoading(true);
    
    try {
      // สร้าง source สำหรับ PromptPay
      const response = await fetch('https://api.your-backend.com/payment/omise/promptpay', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`,
        },
        body: JSON.stringify({ amount }),
      });
      
      const data = await response.json();
      return data.qrCodeUrl; // URL ของ QR Code
    } catch (err: any) {
      setError(err.message);
      return null;
    } finally {
      setLoading(false);
    }
  };

  // ชำระเงินด้วยบัตรเครดิต
  const chargeCard = async (
    tokenId: string, 
    amount: number,
    description: string
  ): Promise<boolean> => {
    setLoading(true);
    
    try {
      const response = await fetch('https://api.your-backend.com/payment/omise/charge', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`,
        },
        body: JSON.stringify({
          tokenId,
          amount,
          description,
          currency: 'thb',
        }),
      });
      
      const data = await response.json();
      return data.status === 'successful';
    } catch (err: any) {
      setError(err.message);
      return false;
    } finally {
      setLoading(false);
    }
  };

  return {
    loading,
    error,
    createCardToken,
    payWithPromptPay,
    chargeCard,
  };
};
```

### screens/OmiseCheckoutScreen.tsx

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Image,
  Alert,
} from 'react-native';
import { useOmisePayment } from '../hooks/useOmisePayment';

type PaymentMethod = 'card' | 'promptpay' | 'truemoney';

const OmiseCheckoutScreen: React.FC<{ amount: number; onSuccess: () => void }> = ({
  amount, onSuccess
}) => {
  const { loading, error, createCardToken, chargeCard, payWithPromptPay } = useOmisePayment();
  const [paymentMethod, setPaymentMethod] = useState<PaymentMethod>('card');
  const [cardDetails, setCardDetails] = useState({
    name: '',
    number: '',
    expiryMonth: '',
    expiryYear: '',
    cvv: '',
  });
  const [qrCodeUrl, setQrCodeUrl] = useState<string | null>(null);

  // Format card number: XXXX XXXX XXXX XXXX
  const formatCardNumber = (text: string) => {
    const cleaned = text.replace(/\s/g, '');
    const groups = cleaned.match(/.{1,4}/g);
    return groups ? groups.join(' ') : cleaned;
  };

  const handleCardPayment = async () => {
    const token = await createCardToken({
      name: cardDetails.name,
      number: cardDetails.number.replace(/\s/g, ''),
      expirationMonth: cardDetails.expiryMonth,
      expirationYear: `20${cardDetails.expiryYear}`,
      securityCode: cardDetails.cvv,
    });
    
    if (!token) return;
    
    const success = await chargeCard(token, amount * 100, 'ซื้อสินค้า');
    
    if (success) {
      Alert.alert('สำเร็จ! 🎉', 'ชำระเงินเรียบร้อยแล้ว', [
        { text: 'ตกลง', onPress: onSuccess }
      ]);
    }
  };

  const handlePromptPay = async () => {
    const url = await payWithPromptPay(amount * 100);
    if (url) {
      setQrCodeUrl(url);
    }
  };

  return (
    <ScrollView style={styles.container}>
      <View style={styles.amountCard}>
        <Text style={styles.amountLabel}>ยอดที่ต้องชำระ</Text>
        <Text style={styles.amountValue}>฿{amount.toLocaleString()}</Text>
      </View>

      {/* Payment Method Selection */}
      <View style={styles.methodCard}>
        <Text style={styles.sectionTitle}>วิธีการชำระเงิน</Text>
        <View style={styles.methodList}>
          {[
            { type: 'card' as PaymentMethod, icon: '💳', label: 'บัตรเครดิต/เดบิต' },
            { type: 'promptpay' as PaymentMethod, icon: '📱', label: 'PromptPay QR' },
            { type: 'truemoney' as PaymentMethod, icon: '🔴', label: 'TrueMoney Wallet' },
          ].map(method => (
            <TouchableOpacity
              key={method.type}
              style={[
                styles.methodItem,
                paymentMethod === method.type && styles.methodItemActive
              ]}
              onPress={() => setPaymentMethod(method.type)}
            >
              <Text style={styles.methodIcon}>{method.icon}</Text>
              <Text style={[
                styles.methodLabel,
                paymentMethod === method.type && styles.methodLabelActive
              ]}>
                {method.label}
              </Text>
              {paymentMethod === method.type && (
                <Text style={styles.checkmark}>✓</Text>
              )}
            </TouchableOpacity>
          ))}
        </View>
      </View>

      {/* Card Form */}
      {paymentMethod === 'card' && (
        <View style={styles.formCard}>
          <Text style={styles.sectionTitle}>ข้อมูลบัตร</Text>
          
          <Text style={styles.label}>ชื่อบนบัตร</Text>
          <TextInput
            style={styles.input}
            value={cardDetails.name}
            onChangeText={(text) => setCardDetails(prev => ({ ...prev, name: text }))}
            placeholder="JOHN DOE"
            autoCapitalize="characters"
          />
          
          <Text style={styles.label}>หมายเลขบัตร</Text>
          <TextInput
            style={styles.input}
            value={cardDetails.number}
            onChangeText={(text) => setCardDetails(prev => ({
              ...prev,
              number: formatCardNumber(text)
            }))}
            placeholder="0000 0000 0000 0000"
            keyboardType="numeric"
            maxLength={19}
          />
          
          <View style={styles.row}>
            <View style={styles.halfInput}>
              <Text style={styles.label}>วันหมดอายุ</Text>
              <TextInput
                style={styles.input}
                value={cardDetails.expiryMonth}
                onChangeText={(text) => setCardDetails(prev => ({ ...prev, expiryMonth: text }))}
                placeholder="MM"
                keyboardType="numeric"
                maxLength={2}
              />
            </View>
            <View style={styles.halfInput}>
              <Text style={styles.label}>ปี</Text>
              <TextInput
                style={styles.input}
                value={cardDetails.expiryYear}
                onChangeText={(text) => setCardDetails(prev => ({ ...prev, expiryYear: text }))}
                placeholder="YY"
                keyboardType="numeric"
                maxLength={2}
              />
            </View>
            <View style={styles.halfInput}>
              <Text style={styles.label}>CVV</Text>
              <TextInput
                style={styles.input}
                value={cardDetails.cvv}
                onChangeText={(text) => setCardDetails(prev => ({ ...prev, cvv: text }))}
                placeholder="***"
                keyboardType="numeric"
                maxLength={4}
                secureTextEntry
              />
            </View>
          </View>
          
          <View style={styles.acceptedCards}>
            <Text style={styles.acceptedText}>รับบัตร: Visa, Mastercard, JCB, Amex</Text>
          </View>
          
          <TouchableOpacity
            style={styles.payButton}
            onPress={handleCardPayment}
            disabled={loading}
          >
            <Text style={styles.payButtonText}>
              {loading ? 'กำลังดำเนินการ...' : `ชำระ ฿${amount.toLocaleString()}`}
            </Text>
          </TouchableOpacity>
        </View>
      )}

      {/* PromptPay QR */}
      {paymentMethod === 'promptpay' && (
        <View style={styles.formCard}>
          {qrCodeUrl ? (
            <View style={styles.qrContainer}>
              <Text style={styles.qrTitle}>สแกน QR Code เพื่อชำระเงิน</Text>
              <Image source={{ uri: qrCodeUrl }} style={styles.qrCode} />
              <Text style={styles.qrAmount}>฿{amount.toLocaleString()}</Text>
              <Text style={styles.qrExpiry}>QR Code หมดอายุใน 30 นาที</Text>
            </View>
          ) : (
            <TouchableOpacity
              style={styles.payButton}
              onPress={handlePromptPay}
              disabled={loading}
            >
              <Text style={styles.payButtonText}>
                {loading ? 'กำลังสร้าง QR...' : 'สร้าง QR Code PromptPay'}
              </Text>
            </TouchableOpacity>
          )}
        </View>
      )}

      {error && (
        <View style={styles.errorBox}>
          <Text style={styles.errorText}>❌ {error}</Text>
        </View>
      )}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5', padding: 16 },
  amountCard: {
    backgroundColor: '#1565C0', borderRadius: 16, padding: 24, marginBottom: 16,
    alignItems: 'center',
  },
  amountLabel: { color: 'rgba(255,255,255,0.8)', fontSize: 14 },
  amountValue: { color: 'white', fontSize: 36, fontWeight: 'bold', marginTop: 4 },
  methodCard: {
    backgroundColor: 'white', borderRadius: 12, padding: 16, marginBottom: 16,
    shadowColor: '#000', shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1, shadowRadius: 4, elevation: 2,
  },
  sectionTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 12 },
  methodList: { gap: 8 },
  methodItem: {
    flexDirection: 'row', alignItems: 'center', padding: 12,
    borderRadius: 8, borderWidth: 1, borderColor: '#e0e0e0',
  },
  methodItemActive: { borderColor: '#1565C0', backgroundColor: '#e3f2fd' },
  methodIcon: { fontSize: 24, marginRight: 12 },
  methodLabel: { flex: 1, fontSize: 14, color: '#333' },
  methodLabelActive: { color: '#1565C0', fontWeight: '600' },
  checkmark: { color: '#1565C0', fontWeight: 'bold', fontSize: 16 },
  formCard: {
    backgroundColor: 'white', borderRadius: 12, padding: 16, marginBottom: 16,
    shadowColor: '#000', shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1, shadowRadius: 4, elevation: 2,
  },
  label: { fontSize: 12, color: '#666', marginBottom: 4, fontWeight: '500' },
  input: {
    borderWidth: 1, borderColor: '#ddd', borderRadius: 8, padding: 12,
    fontSize: 14, marginBottom: 12, backgroundColor: '#fafafa',
  },
  row: { flexDirection: 'row', gap: 8 },
  halfInput: { flex: 1 },
  acceptedCards: { padding: 8, backgroundColor: '#f5f5f5', borderRadius: 8, marginBottom: 12 },
  acceptedText: { fontSize: 12, color: '#666', textAlign: 'center' },
  payButton: {
    backgroundColor: '#1565C0', padding: 16, borderRadius: 12, alignItems: 'center',
  },
  payButtonText: { color: 'white', fontSize: 16, fontWeight: '600' },
  qrContainer: { alignItems: 'center', padding: 16 },
  qrTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 16 },
  qrCode: { width: 250, height: 250, resizeMode: 'contain' },
  qrAmount: { fontSize: 24, fontWeight: 'bold', color: '#1565C0', marginTop: 12 },
  qrExpiry: { fontSize: 12, color: '#999', marginTop: 4 },
  errorBox: { backgroundColor: '#ffebee', padding: 12, borderRadius: 8, marginBottom: 16 },
  errorText: { color: '#c62828' },
});

export default OmiseCheckoutScreen;
```

---

## 3. Apple Pay / Google Pay

### hooks/useNativePay.ts

```typescript
import { Platform } from 'react-native';
import {
  useApplePay,
  useGooglePay,
  ApplePayResult,
  GooglePayResult,
} from '@stripe/stripe-react-native';

export const useNativePay = () => {
  // Apple Pay (iOS only)
  const { isApplePaySupported, presentApplePay, confirmApplePayPayment } = useApplePay();
  
  // Google Pay (Android only)
  const { isGooglePaySupported, initGooglePay, presentGooglePay } = useGooglePay();

  const payWithApplePay = async (amount: number): Promise<boolean> => {
    if (!isApplePaySupported) {
      console.log('Apple Pay ไม่รองรับ');
      return false;
    }
    
    const { error, paymentMethod } = await presentApplePay({
      cartItems: [
        {
          label: 'สินค้า',
          amount: (amount / 100).toFixed(2),
          type: 'final',
        },
      ],
      country: 'TH',
      currency: 'THB',
      shippingMethods: [
        {
          id: 'standard',
          label: 'ส่งมาตรฐาน',
          detail: 'ถึงใน 3-5 วัน',
          amount: '50.00',
        },
      ],
      requiredShippingAddressFields: ['postalAddress', 'phoneNumber'],
      requiredBillingContactFields: ['phoneNumber', 'name'],
    });
    
    if (error) {
      console.error('Apple Pay error:', error);
      return false;
    }
    
    // ยืนยันการชำระเงินกับ backend
    const clientSecret = await getClientSecretFromBackend(amount);
    const { error: confirmError } = await confirmApplePayPayment(clientSecret);
    
    return !confirmError;
  };

  const payWithGooglePay = async (amount: number): Promise<boolean> => {
    const supported = await isGooglePaySupported({ testEnv: true });
    if (!supported) {
      console.log('Google Pay ไม่รองรับ');
      return false;
    }
    
    const { error: initError } = await initGooglePay({
      testEnv: true,
      merchantName: 'ร้านค้าของเรา',
      countryCode: 'TH',
      billingAddressConfig: {
        format: 'FULL',
        isPhoneNumberRequired: true,
        isRequired: false,
      },
      existingPaymentMethodRequired: false,
      isEmailRequired: true,
    });
    
    if (initError) {
      console.error('Google Pay init error:', initError);
      return false;
    }
    
    const clientSecret = await getClientSecretFromBackend(amount);
    
    const { error } = await presentGooglePay({
      clientSecret,
      forSetupIntent: false,
    });
    
    return !error;
  };

  const pay = async (amount: number): Promise<boolean> => {
    if (Platform.OS === 'ios') {
      return payWithApplePay(amount);
    } else {
      return payWithGooglePay(amount);
    }
  };

  const getClientSecretFromBackend = async (amount: number): Promise<string> => {
    const response = await fetch('https://api.yourapp.com/payment/create-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount }),
    });
    const { clientSecret } = await response.json();
    return clientSecret;
  };

  return {
    isApplePaySupported,
    payWithApplePay,
    payWithGooglePay,
    pay,
  };
};
```

---

## 4. Backend API (Node.js)

### server/payment.js

```javascript
const express = require('express');
const router = express.Router();
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const Omise = require('omise')({
  publicKey: process.env.OMISE_PUBLIC_KEY,
  secretKey: process.env.OMISE_SECRET_KEY,
});

// Stripe - Create Payment Intent
router.post('/stripe/create-intent', async (req, res) => {
  try {
    const { amount, currency, description, customerId } = req.body;
    
    const paymentIntent = await stripe.paymentIntents.create({
      amount,
      currency,
      description,
      customer: customerId,
      automatic_payment_methods: {
        enabled: true,
      },
    });
    
    // Create ephemeral key
    const ephemeralKey = await stripe.ephemeralKeys.create(
      { customer: customerId },
      { apiVersion: '2022-11-15' }
    );
    
    res.json({
      paymentIntent: paymentIntent.client_secret,
      ephemeralKey: ephemeralKey.secret,
      customer: customerId,
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Omise - Create Charge
router.post('/omise/charge', async (req, res) => {
  try {
    const { tokenId, amount, description, currency } = req.body;
    
    const charge = await Omise.charges.create({
      amount, // ใน satang
      currency,
      description,
      card: tokenId,
    });
    
    res.json({
      id: charge.id,
      status: charge.status,
      amount: charge.amount,
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Omise - PromptPay
router.post('/omise/promptpay', async (req, res) => {
  try {
    const { amount } = req.body;
    
    // สร้าง source
    const source = await Omise.sources.create({
      amount,
      currency: 'thb',
      type: 'promptpay',
    });
    
    // สร้าง charge
    const charge = await Omise.charges.create({
      amount,
      currency: 'thb',
      source: source.id,
    });
    
    res.json({
      chargeId: charge.id,
      qrCodeUrl: charge.source.scannable_code?.image?.download_uri,
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

---

## Workshop: Complete E-commerce Payment

### แบบฝึกหัดที่ 1: Multi-Payment Gateway
สร้างหน้า checkout ที่รองรับหลาย payment methods และให้ผู้ใช้เลือก

### แบบฝึกหัดที่ 2: Saved Cards
ใช้ Stripe Customer เพื่อบันทึกบัตรไว้ใช้ครั้งต่อไป

### แบบฝึกหัดที่ 3: Subscription
สร้างระบบ subscription ด้วย Stripe Subscriptions

### แบบฝึกหัดที่ 4: Refunds
สร้างระบบ refund เมื่อผู้ใช้ยกเลิกคำสั่งซื้อ

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Stripe** - Payment Sheet, Cards, Apple/Google Pay
2. **Omise** - บัตรเครดิต, PromptPay
3. **Native Pay** - Apple Pay และ Google Pay
4. **Backend** - การสร้าง payment server

> **Tips:** ทดสอบด้วย test cards เสมอก่อน go live และตรวจสอบ PCI DSS compliance
