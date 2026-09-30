# Part 069: A/B Testing และ Feature Flags ใน React Native

## บทนำ

A/B Testing และ Feature Flags เป็นเครื่องมือสำคัญที่ช่วยให้ทีมพัฒนาสามารถทดสอบ features ใหม่ กับผู้ใช้บางส่วนก่อน แล้ววัดผลว่าเวอร์ชันไหนดีกว่า และ deploy features ได้อย่างปลอดภัย

---

## 1. ทำความเข้าใจ A/B Testing

### A/B Testing คืออะไร?

```
A/B Testing = การทดสอบสองเวอร์ชันของฟีเจอร์หรือ UI 
กับกลุ่มผู้ใช้ที่แบ่งแยกกัน เพื่อดูว่าเวอร์ชันไหนให้ผลลัพธ์ที่ดีกว่า

Version A (Control):   ฟีเจอร์/UI เดิม
Version B (Treatment): ฟีเจอร์/UI ใหม่

ตัวอย่าง:
- ปุ่ม "Buy Now" สีน้ำเงิน vs สีแดง
- Checkout flow แบบ 3 ขั้นตอน vs 1 ขั้นตอน
- ราคาแสดงแบบ $9.99 vs $10
```

### Feature Flags คืออะไร?

```
Feature Flags = Switch ที่ให้เปิด/ปิด features 
โดยไม่ต้อง deploy code ใหม่

ประโยชน์:
- Deploy code แต่ยังไม่เปิด feature ให้ใช้
- เปิด feature ให้ user บางกลุ่มก่อน (gradual rollout)
- ปิด feature ได้ทันทีถ้ามีปัญหา (kill switch)
- ทดสอบ feature ใน production
```

### ความแตกต่าง

```
Feature Flags:          เปิด/ปิด feature
A/B Testing:           เปรียบเทียบสอง versions
Multivariate Testing:  เปรียบเทียบหลาย versions พร้อมกัน
```

---

## 2. Firebase Remote Config

### ติดตั้ง Firebase Remote Config

```bash
npm install @react-native-firebase/app @react-native-firebase/remote-config
cd ios && pod install
```

### ตั้งค่า Remote Config

```typescript
// src/featureFlags/remoteConfig.ts
import remoteConfig from '@react-native-firebase/remote-config';

// Default values (ใช้เมื่อไม่มี network หรือ first launch)
const DEFAULT_CONFIG = {
  // A/B Test values
  checkout_button_color: 'blue',
  checkout_flow_version: 'v1',
  
  // Feature Flags
  feature_dark_mode: false,
  feature_biometric_auth: false,
  feature_social_login: false,
  feature_new_profile: false,
  feature_ai_recommendations: false,
  
  // UI Configuration
  home_banner_enabled: true,
  products_per_page: 20,
  max_cart_items: 50,
  
  // Pricing
  discount_percentage: 0,
  promo_code_enabled: false,
  
  // Experiment assignment
  user_segment: 'default',
} as const;

class RemoteConfigService {
  private initialized = false;

  async initialize(): Promise<void> {
    try {
      // ตั้งค่า default values
      await remoteConfig().setDefaults(DEFAULT_CONFIG);
      
      // ตั้งค่า fetch interval
      await remoteConfig().setConfigSettings({
        minimumFetchIntervalMillis: __DEV__ 
          ? 0          // 0 seconds ใน development
          : 3600000,   // 1 hour ใน production
      });
      
      // Fetch และ activate config
      await this.fetchAndActivate();
      
      this.initialized = true;
    } catch (error) {
      console.error('Remote Config initialization failed:', error);
    }
  }

  async fetchAndActivate(): Promise<boolean> {
    try {
      return await remoteConfig().fetchAndActivate();
    } catch (error) {
      console.error('Remote Config fetch failed:', error);
      return false;
    }
  }

  getString(key: keyof typeof DEFAULT_CONFIG): string {
    return remoteConfig().getValue(key).asString();
  }

  getBoolean(key: keyof typeof DEFAULT_CONFIG): boolean {
    return remoteConfig().getValue(key).asBoolean();
  }

  getNumber(key: keyof typeof DEFAULT_CONFIG): number {
    return remoteConfig().getValue(key).asNumber();
  }

  getAll(): Record<string, unknown> {
    const all = remoteConfig().getAll();
    return Object.entries(all).reduce((acc, [key, value]) => ({
      ...acc,
      [key]: value.asString(),
    }), {});
  }
}

export default new RemoteConfigService();
```

### ใช้ Remote Config ใน Components

```typescript
// src/hooks/useRemoteConfig.ts
import { useEffect, useState, useCallback } from 'react';
import RemoteConfigService from '../featureFlags/remoteConfig';

interface RemoteConfigState {
  isLoading: boolean;
  config: Record<string, unknown>;
  refresh: () => Promise<void>;
}

export const useRemoteConfig = (): RemoteConfigState => {
  const [isLoading, setIsLoading] = useState(true);
  const [config, setConfig] = useState<Record<string, unknown>>({});

  const loadConfig = useCallback(async () => {
    setIsLoading(true);
    await RemoteConfigService.fetchAndActivate();
    setConfig(RemoteConfigService.getAll());
    setIsLoading(false);
  }, []);

  useEffect(() => {
    loadConfig();
  }, [loadConfig]);

  return { isLoading, config, refresh: loadConfig };
};

// Hook เฉพาะสำหรับ feature flags
export const useFeatureFlag = (flagName: string): boolean => {
  return RemoteConfigService.getBoolean(flagName as any);
};

// Hook สำหรับ A/B test variant
export const useExperimentVariant = (experimentName: string): string => {
  return RemoteConfigService.getString(experimentName as any);
};
```

---

## 3. Feature Flags System

### Feature Flag Manager

```typescript
// src/featureFlags/FeatureFlagManager.ts
import RemoteConfigService from './remoteConfig';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface FeatureFlag {
  name: string;
  defaultValue: boolean;
  description?: string;
  rolloutPercentage?: number; // 0-100
}

const FEATURE_FLAGS: Record<string, FeatureFlag> = {
  DARK_MODE: {
    name: 'feature_dark_mode',
    defaultValue: false,
    description: 'Enable dark mode support',
    rolloutPercentage: 100,
  },
  BIOMETRIC_AUTH: {
    name: 'feature_biometric_auth',
    defaultValue: false,
    description: 'Enable biometric authentication',
    rolloutPercentage: 50, // เปิดให้ 50% ของ users
  },
  NEW_CHECKOUT: {
    name: 'feature_new_checkout',
    defaultValue: false,
    description: 'New checkout flow with fewer steps',
    rolloutPercentage: 20, // เปิดให้ 20% ก่อน
  },
  AI_RECOMMENDATIONS: {
    name: 'feature_ai_recommendations',
    defaultValue: false,
    description: 'AI-powered product recommendations',
    rolloutPercentage: 10,
  },
} as const;

const OVERRIDE_KEY = '@feature_flag_overrides';

class FeatureFlagManager {
  private overrides: Record<string, boolean> = {};
  private userId: string = '';

  async initialize(userId: string): Promise<void> {
    this.userId = userId;
    
    // โหลด overrides จาก AsyncStorage (สำหรับ dev testing)
    if (__DEV__) {
      const stored = await AsyncStorage.getItem(OVERRIDE_KEY);
      if (stored) {
        this.overrides = JSON.parse(stored);
      }
    }
  }

  isEnabled(flagKey: keyof typeof FEATURE_FLAGS): boolean {
    const flag = FEATURE_FLAGS[flagKey];
    if (!flag) return false;

    // Check manual override (dev only)
    if (__DEV__ && this.overrides[flag.name] !== undefined) {
      return this.overrides[flag.name];
    }

    // Check Remote Config
    const remoteValue = RemoteConfigService.getBoolean(flag.name as any);
    
    // Apply rollout percentage
    if (flag.rolloutPercentage !== undefined && flag.rolloutPercentage < 100) {
      const userHash = this.getUserHash(this.userId, flag.name);
      return remoteValue && userHash < flag.rolloutPercentage;
    }
    
    return remoteValue;
  }

  // Hash function เพื่อ consistent assignment
  private getUserHash(userId: string, flagName: string): number {
    const str = `${userId}_${flagName}`;
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash) % 100;
  }

  // Override สำหรับ development testing
  async setOverride(flagName: string, value: boolean): Promise<void> {
    if (!__DEV__) return;
    
    this.overrides[flagName] = value;
    await AsyncStorage.setItem(OVERRIDE_KEY, JSON.stringify(this.overrides));
  }

  async clearOverrides(): Promise<void> {
    if (!__DEV__) return;
    
    this.overrides = {};
    await AsyncStorage.removeItem(OVERRIDE_KEY);
  }

  getAll(): Record<string, boolean> {
    return Object.entries(FEATURE_FLAGS).reduce((acc, [key]) => ({
      ...acc,
      [key]: this.isEnabled(key as keyof typeof FEATURE_FLAGS),
    }), {});
  }
}

export const featureFlags = new FeatureFlagManager();
export { FEATURE_FLAGS };
```

### Feature Flag Component

```typescript
// src/components/FeatureFlag.tsx
import React from 'react';
import { featureFlags, FEATURE_FLAGS } from '../featureFlags/FeatureFlagManager';

interface FeatureFlagProps {
  flag: keyof typeof FEATURE_FLAGS;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

const FeatureFlag: React.FC<FeatureFlagProps> = ({ flag, children, fallback }) => {
  const isEnabled = featureFlags.isEnabled(flag);
  
  if (isEnabled) {
    return <>{children}</>;
  }
  
  return fallback ? <>{fallback}</> : null;
};

export default FeatureFlag;

// Higher-Order Component
export const withFeatureFlag = <P extends object>(
  WrappedComponent: React.ComponentType<P>,
  flag: keyof typeof FEATURE_FLAGS,
  FallbackComponent?: React.ComponentType<P>
) => {
  return (props: P) => {
    const isEnabled = featureFlags.isEnabled(flag);
    
    if (isEnabled) {
      return <WrappedComponent {...props} />;
    }
    
    if (FallbackComponent) {
      return <FallbackComponent {...props} />;
    }
    
    return null;
  };
};
```

### การใช้งาน Feature Flags

```typescript
// ใน component
import FeatureFlag from '../components/FeatureFlag';
import { featureFlags } from '../featureFlags/FeatureFlagManager';

const ProfileScreen: React.FC = () => {
  const hasBiometric = featureFlags.isEnabled('BIOMETRIC_AUTH');
  
  return (
    <View>
      {/* ใช้ component */}
      <FeatureFlag flag="DARK_MODE">
        <DarkModeToggle />
      </FeatureFlag>
      
      {/* ใช้ hook */}
      {hasBiometric && (
        <TouchableOpacity onPress={handleBiometricSetup}>
          <Text>Enable Face ID / Touch ID</Text>
        </TouchableOpacity>
      )}
      
      {/* Feature ที่มี fallback */}
      <FeatureFlag 
        flag="AI_RECOMMENDATIONS"
        fallback={<ManualRecommendations />}
      >
        <AIRecommendations />
      </FeatureFlag>
    </View>
  );
};
```

---

## 4. A/B Testing Strategy

### A/B Test Manager

```typescript
// src/experiments/ExperimentManager.ts
import RemoteConfigService from '../featureFlags/remoteConfig';
import Analytics from '../analytics/AnalyticsManager';

interface Experiment {
  id: string;
  name: string;
  variants: string[];
  defaultVariant: string;
}

const EXPERIMENTS: Record<string, Experiment> = {
  CHECKOUT_BUTTON_COLOR: {
    id: 'checkout_button_color',
    name: 'Checkout Button Color Test',
    variants: ['blue', 'red', 'green'],
    defaultVariant: 'blue',
  },
  CHECKOUT_FLOW: {
    id: 'checkout_flow_version',
    name: 'Checkout Flow Version Test',
    variants: ['v1', 'v2', 'v3'],
    defaultVariant: 'v1',
  },
  PRICING_DISPLAY: {
    id: 'pricing_display',
    name: 'Pricing Display Format',
    variants: ['decimal', 'rounded', 'discount_badge'],
    defaultVariant: 'decimal',
  },
  ONBOARDING: {
    id: 'onboarding_version',
    name: 'Onboarding Flow Test',
    variants: ['standard', 'simplified', 'video'],
    defaultVariant: 'standard',
  },
} as const;

class ExperimentManager {
  private impressions: Set<string> = new Set();

  getVariant(experimentKey: keyof typeof EXPERIMENTS): string {
    const experiment = EXPERIMENTS[experimentKey];
    if (!experiment) return '';
    
    const variant = RemoteConfigService.getString(experiment.id as any);
    
    // Validate variant
    if (experiment.variants.includes(variant)) {
      return variant;
    }
    
    return experiment.defaultVariant;
  }

  // Track experiment impression (เรียกครั้งเดียวต่อ session)
  trackImpression(experimentKey: keyof typeof EXPERIMENTS): void {
    const experiment = EXPERIMENTS[experimentKey];
    if (!experiment) return;
    
    const variant = this.getVariant(experimentKey);
    const impressionKey = `${experiment.id}_${variant}`;
    
    if (!this.impressions.has(impressionKey)) {
      this.impressions.add(impressionKey);
      
      Analytics.track('experiment_impression', {
        experiment_id: experiment.id,
        experiment_name: experiment.name,
        variant,
      });
    }
  }

  // Track conversion
  trackConversion(
    experimentKey: keyof typeof EXPERIMENTS,
    conversionEvent: string,
    value?: number
  ): void {
    const experiment = EXPERIMENTS[experimentKey];
    if (!experiment) return;
    
    const variant = this.getVariant(experimentKey);
    
    Analytics.track('experiment_conversion', {
      experiment_id: experiment.id,
      experiment_name: experiment.name,
      variant,
      conversion_event: conversionEvent,
      value,
    });
  }

  getAllVariants(): Record<string, string> {
    return Object.entries(EXPERIMENTS).reduce((acc, [key]) => ({
      ...acc,
      [key]: this.getVariant(key as keyof typeof EXPERIMENTS),
    }), {});
  }
}

export const experimentManager = new ExperimentManager();
export { EXPERIMENTS };
```

### ใช้ A/B Test ใน Components

```typescript
// src/screens/CheckoutScreen.tsx
import React, { useEffect } from 'react';
import { View, TouchableOpacity, Text, StyleSheet } from 'react-native';
import { experimentManager } from '../experiments/ExperimentManager';

const CheckoutScreen: React.FC = () => {
  const buttonColor = experimentManager.getVariant('CHECKOUT_BUTTON_COLOR');
  const checkoutFlow = experimentManager.getVariant('CHECKOUT_FLOW');

  useEffect(() => {
    // Track impression เมื่อ user เห็น checkout screen
    experimentManager.trackImpression('CHECKOUT_BUTTON_COLOR');
    experimentManager.trackImpression('CHECKOUT_FLOW');
  }, []);

  const handleCheckout = async () => {
    // ... process checkout
    
    // Track conversion เมื่อ user กด checkout สำเร็จ
    experimentManager.trackConversion('CHECKOUT_BUTTON_COLOR', 'checkout_clicked');
    experimentManager.trackConversion('CHECKOUT_FLOW', 'checkout_completed', totalAmount);
  };

  const getButtonStyle = () => {
    const colors: Record<string, string> = {
      blue: '#2196F3',
      red: '#f44336',
      green: '#4CAF50',
    };
    return { backgroundColor: colors[buttonColor] || colors.blue };
  };

  return (
    <View style={styles.container}>
      {checkoutFlow === 'v2' ? (
        <CheckoutFlowV2 />
      ) : checkoutFlow === 'v3' ? (
        <CheckoutFlowV3 />
      ) : (
        <CheckoutFlowV1 />
      )}
      
      <TouchableOpacity
        style={[styles.checkoutButton, getButtonStyle()]}
        onPress={handleCheckout}
      >
        <Text style={styles.buttonText}>Complete Purchase</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20 },
  checkoutButton: {
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 20,
  },
  buttonText: {
    color: '#fff',
    fontSize: 18,
    fontWeight: 'bold',
  },
});
```

---

## 5. Measuring Results

### Analytics Events สำหรับ A/B Tests

```typescript
// src/analytics/experimentTracking.ts
import Analytics from '../analytics/AnalyticsManager';

interface ConversionGoal {
  experimentId: string;
  variant: string;
  goalName: string;
  value?: number;
  metadata?: Record<string, unknown>;
}

export const trackConversionGoal = (goal: ConversionGoal): void => {
  Analytics.track('conversion_goal', {
    experiment_id: goal.experimentId,
    variant: goal.variant,
    goal_name: goal.goalName,
    value: goal.value,
    timestamp: Date.now(),
    ...goal.metadata,
  });
};

// Predefined goal tracking helpers
export const trackCheckoutStarted = (
  experimentId: string,
  variant: string,
  cartValue: number
) => {
  trackConversionGoal({
    experimentId,
    variant,
    goalName: 'checkout_started',
    value: cartValue,
  });
};

export const trackPurchaseCompleted = (
  experimentId: string,
  variant: string,
  orderValue: number
) => {
  trackConversionGoal({
    experimentId,
    variant,
    goalName: 'purchase_completed',
    value: orderValue,
  });
};
```

### Statistical Significance Calculator

```typescript
// src/utils/statisticsCalculator.ts
interface ExperimentResults {
  controlConversions: number;
  controlVisitors: number;
  variantConversions: number;
  variantVisitors: number;
}

export const calculateStatisticalSignificance = (
  results: ExperimentResults
): {
  isSignificant: boolean;
  confidence: number;
  improvement: number;
  pValue: number;
} => {
  const { controlConversions, controlVisitors, variantConversions, variantVisitors } = results;
  
  const controlRate = controlConversions / controlVisitors;
  const variantRate = variantConversions / variantVisitors;
  
  // Calculate improvement
  const improvement = ((variantRate - controlRate) / controlRate) * 100;
  
  // Chi-squared test (simplified)
  const expected_control = controlVisitors * controlRate;
  const expected_variant = variantVisitors * variantRate;
  
  // Z-score calculation
  const pooledRate = (controlConversions + variantConversions) / 
                     (controlVisitors + variantVisitors);
  const se = Math.sqrt(pooledRate * (1 - pooledRate) * 
                       (1/controlVisitors + 1/variantVisitors));
  const zScore = (variantRate - controlRate) / se;
  
  // P-value (approximation)
  const pValue = 2 * (1 - normalCDF(Math.abs(zScore)));
  const confidence = (1 - pValue) * 100;
  const isSignificant = pValue < 0.05; // 95% confidence
  
  return { isSignificant, confidence, improvement, pValue };
};

// Normal CDF approximation
const normalCDF = (z: number): number => {
  const t = 1 / (1 + 0.2316419 * Math.abs(z));
  const d = 0.3989423 * Math.exp(-z * z / 2);
  const prob = d * t * (0.3193815 + t * (-0.3565638 + 
    t * (1.781478 + t * (-1.821256 + t * 1.330274))));
  return z > 0 ? 1 - prob : prob;
};
```

---

## 6. Debug Panel (Development)

### Feature Flag Debug Screen

```typescript
// src/screens/dev/FeatureFlagDebugScreen.tsx
import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  Switch,
  ScrollView,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import { featureFlags, FEATURE_FLAGS } from '../../featureFlags/FeatureFlagManager';
import { experimentManager, EXPERIMENTS } from '../../experiments/ExperimentManager';

// เฉพาะ development mode
if (!__DEV__) {
  throw new Error('FeatureFlagDebugScreen should only be used in development');
}

const FeatureFlagDebugScreen: React.FC = () => {
  const [flags, setFlags] = useState<Record<string, boolean>>({});
  const [variants, setVariants] = useState<Record<string, string>>({});

  useEffect(() => {
    refreshAll();
  }, []);

  const refreshAll = () => {
    setFlags(featureFlags.getAll());
    setVariants(experimentManager.getAllVariants());
  };

  const toggleFlag = async (flagKey: string, value: boolean) => {
    const flag = FEATURE_FLAGS[flagKey as keyof typeof FEATURE_FLAGS];
    if (flag) {
      await featureFlags.setOverride(flag.name, value);
      refreshAll();
    }
  };

  const resetAll = async () => {
    Alert.alert(
      'Reset Overrides',
      'This will clear all manual overrides. Continue?',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Reset',
          style: 'destructive',
          onPress: async () => {
            await featureFlags.clearOverrides();
            refreshAll();
          },
        },
      ]
    );
  };

  return (
    <ScrollView style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>Feature Flags Debug</Text>
        <TouchableOpacity onPress={resetAll} style={styles.resetButton}>
          <Text style={styles.resetText}>Reset All</Text>
        </TouchableOpacity>
      </View>

      <Text style={styles.sectionTitle}>Feature Flags</Text>
      {Object.entries(FEATURE_FLAGS).map(([key, flag]) => (
        <View key={key} style={styles.flagItem}>
          <View style={styles.flagInfo}>
            <Text style={styles.flagName}>{key}</Text>
            <Text style={styles.flagDescription}>{flag.description}</Text>
          </View>
          <Switch
            value={flags[key] || false}
            onValueChange={(value) => toggleFlag(key, value)}
          />
        </View>
      ))}

      <Text style={styles.sectionTitle}>Experiments</Text>
      {Object.entries(EXPERIMENTS).map(([key, exp]) => (
        <View key={key} style={styles.experimentItem}>
          <Text style={styles.expName}>{key}</Text>
          <View style={styles.variantButtons}>
            {exp.variants.map(variant => (
              <TouchableOpacity
                key={variant}
                style={[
                  styles.variantButton,
                  variants[key] === variant && styles.activeVariant,
                ]}
                onPress={() => {/* set variant */}}
              >
                <Text style={[
                  styles.variantText,
                  variants[key] === variant && styles.activeVariantText,
                ]}>
                  {variant}
                </Text>
              </TouchableOpacity>
            ))}
          </View>
          <Text style={styles.currentVariant}>Current: {variants[key]}</Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  title: { fontSize: 20, fontWeight: 'bold' },
  resetButton: { padding: 8 },
  resetText: { color: '#f44336', fontWeight: '600' },
  sectionTitle: {
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
    padding: 16,
    paddingBottom: 8,
    textTransform: 'uppercase',
  },
  flagItem: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#fff',
    padding: 16,
    marginBottom: 1,
  },
  flagInfo: { flex: 1 },
  flagName: { fontSize: 16, fontWeight: '600', color: '#333' },
  flagDescription: { fontSize: 12, color: '#666', marginTop: 2 },
  experimentItem: {
    backgroundColor: '#fff',
    padding: 16,
    marginBottom: 8,
  },
  expName: { fontSize: 16, fontWeight: '600', marginBottom: 8 },
  variantButtons: { flexDirection: 'row', flexWrap: 'wrap', gap: 8 },
  variantButton: {
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: '#ddd',
    backgroundColor: '#f5f5f5',
  },
  activeVariant: {
    backgroundColor: '#2196F3',
    borderColor: '#2196F3',
  },
  variantText: { fontSize: 14, color: '#333' },
  activeVariantText: { color: '#fff' },
  currentVariant: { fontSize: 12, color: '#666', marginTop: 8 },
});

export default FeatureFlagDebugScreen;
```

---

## Workshop: Feature Flag System

### Complete Implementation

```typescript
// src/featureFlags/index.ts - Entry point

export { featureFlags, FEATURE_FLAGS } from './FeatureFlagManager';
export { experimentManager, EXPERIMENTS } from '../experiments/ExperimentManager';
export { default as FeatureFlag, withFeatureFlag } from '../components/FeatureFlag';
export { useFeatureFlag, useExperimentVariant } from '../hooks/useRemoteConfig';
```

### Context Provider

```typescript
// src/contexts/FeatureFlagContext.tsx
import React, { createContext, useContext, useEffect, useState } from 'react';
import { featureFlags } from '../featureFlags/FeatureFlagManager';
import { experimentManager } from '../experiments/ExperimentManager';
import RemoteConfigService from '../featureFlags/remoteConfig';

interface FeatureFlagContextValue {
  flags: Record<string, boolean>;
  variants: Record<string, string>;
  isLoading: boolean;
  refresh: () => Promise<void>;
}

const FeatureFlagContext = createContext<FeatureFlagContextValue>({
  flags: {},
  variants: {},
  isLoading: true,
  refresh: async () => {},
});

export const FeatureFlagProvider: React.FC<{
  userId: string;
  children: React.ReactNode;
}> = ({ userId, children }) => {
  const [isLoading, setIsLoading] = useState(true);
  const [flags, setFlags] = useState<Record<string, boolean>>({});
  const [variants, setVariants] = useState<Record<string, string>>({});

  const refresh = async () => {
    await RemoteConfigService.fetchAndActivate();
    setFlags(featureFlags.getAll());
    setVariants(experimentManager.getAllVariants());
  };

  useEffect(() => {
    const initialize = async () => {
      await RemoteConfigService.initialize();
      await featureFlags.initialize(userId);
      setFlags(featureFlags.getAll());
      setVariants(experimentManager.getAllVariants());
      setIsLoading(false);
    };
    
    initialize();
  }, [userId]);

  return (
    <FeatureFlagContext.Provider value={{ flags, variants, isLoading, refresh }}>
      {children}
    </FeatureFlagContext.Provider>
  );
};

export const useFeatureFlagContext = () => useContext(FeatureFlagContext);
```

---

## Tips และ Best Practices

### 1. Test ทั้งสอง Variants

```typescript
// ใน unit tests ตรวจสอบทั้งสอง path
describe('CheckoutButton', () => {
  it('renders blue button for variant A', () => {
    jest.spyOn(experimentManager, 'getVariant').mockReturnValue('blue');
    render(<CheckoutButton />);
    // assert blue button
  });

  it('renders red button for variant B', () => {
    jest.spyOn(experimentManager, 'getVariant').mockReturnValue('red');
    render(<CheckoutButton />);
    // assert red button
  });
});
```

### 2. Experiment Lifecycle

```
Phase 1: Design   - กำหนด hypothesis และ metrics
Phase 2: Implement - code ทั้งสอง variants  
Phase 3: Launch   - เปิด experiment สำหรับ % ของ users
Phase 4: Monitor  - ดูผลทุกวัน
Phase 5: Analyze  - รอ statistical significance
Phase 6: Decide   - เลือก winner หรือ iterate
Phase 7: Cleanup  - ลบ losing variant code
```

### 3. Sample Size Calculator

```
Minimum sample per variant:
- Conversion rate: 2%
- Minimum detectable effect: 20%
- Statistical power: 80%
- Significance level: 5%
→ ต้องการ ~17,000 users per variant
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Feature Flags**: การ implement ระบบ feature flags
2. **Firebase Remote Config**: Server-side configuration
3. **A/B Testing**: การออกแบบและ implement experiments
4. **Measuring Results**: การวัดผลและ statistical significance
5. **Debug Tools**: Developer tools สำหรับทดสอบ
6. **Best Practices**: Experiment lifecycle และ cleanup
