# Part 067: Analytics ใน React Native

## บทนำ

Analytics เป็นสิ่งสำคัญในการเข้าใจพฤติกรรมผู้ใช้ เพื่อปรับปรุงแอปและตัดสินใจทางธุรกิจ ในบทนี้เราจะเรียนรู้การ implement analytics ด้วย Firebase Analytics, Mixpanel และ Amplitude

---

## 1. Analytics Strategy

### ข้อมูลที่ควรติดตาม

```
User Acquisition:
- แหล่งที่มาของผู้ใช้ (organic, paid, referral)
- แพลตฟอร์ม (iOS, Android)
- Region/Country

User Engagement:
- DAU/MAU (Daily/Monthly Active Users)
- Session duration
- Screen views
- Feature usage

User Retention:
- Day 1, Day 7, Day 30 retention
- Churn rate
- Re-engagement

Conversion:
- Registration rate
- Purchase conversion
- Funnel completion
```

### Analytics Architecture

```typescript
// src/analytics/index.ts
export interface AnalyticsEvent {
  name: string;
  params?: Record<string, unknown>;
}

export interface UserProperties {
  userId?: string;
  userType?: string;
  plan?: string;
  country?: string;
}

export interface AnalyticsProvider {
  initialize(): Promise<void>;
  setUserId(userId: string): void;
  setUserProperties(properties: UserProperties): void;
  logEvent(event: AnalyticsEvent): void;
  logScreenView(screenName: string, screenClass?: string): void;
  reset(): void;
}
```

---

## 2. Firebase Analytics

### ติดตั้ง Firebase Analytics

```bash
npm install @react-native-firebase/app @react-native-firebase/analytics
cd ios && pod install
```

### ตั้งค่า Firebase ใน React Native

```typescript
// src/analytics/firebase.ts
import analytics, { FirebaseAnalyticsTypes } from '@react-native-firebase/analytics';
import { AnalyticsProvider, AnalyticsEvent, UserProperties } from './index';

class FirebaseAnalyticsProvider implements AnalyticsProvider {
  async initialize(): Promise<void> {
    // Analytics enabled by default
    await analytics().setAnalyticsCollectionEnabled(true);
  }

  setUserId(userId: string): void {
    analytics().setUserId(userId);
  }

  setUserProperties(properties: UserProperties): void {
    const firebaseProps: Record<string, string> = {};
    
    Object.entries(properties).forEach(([key, value]) => {
      if (value !== undefined) {
        firebaseProps[key] = String(value);
      }
    });
    
    analytics().setUserProperties(firebaseProps);
  }

  logEvent({ name, params }: AnalyticsEvent): void {
    analytics().logEvent(name, params as Record<string, FirebaseAnalyticsTypes.EventParams>);
  }

  logScreenView(screenName: string, screenClass?: string): void {
    analytics().logScreenView({
      screen_name: screenName,
      screen_class: screenClass || screenName,
    });
  }

  reset(): void {
    analytics().resetAnalyticsData();
  }
}

export default new FirebaseAnalyticsProvider();
```

### Predefined Events

```typescript
// src/analytics/events.ts
import analytics from '@react-native-firebase/analytics';

// E-commerce Events
export const trackPurchase = async (params: {
  transactionId: string;
  currency: string;
  value: number;
  items: Array<{
    itemId: string;
    itemName: string;
    price: number;
    quantity: number;
  }>;
}) => {
  await analytics().logPurchase({
    transaction_id: params.transactionId,
    currency: params.currency,
    value: params.value,
    items: params.items.map(item => ({
      item_id: item.itemId,
      item_name: item.itemName,
      price: item.price,
      quantity: item.quantity,
    })),
  });
};

export const trackAddToCart = async (params: {
  itemId: string;
  itemName: string;
  price: number;
  currency: string;
}) => {
  await analytics().logAddToCart({
    currency: params.currency,
    value: params.price,
    items: [{
      item_id: params.itemId,
      item_name: params.itemName,
      price: params.price,
      quantity: 1,
    }],
  });
};

export const trackViewItem = async (params: {
  itemId: string;
  itemName: string;
  category: string;
  price: number;
}) => {
  await analytics().logViewItem({
    currency: 'THB',
    value: params.price,
    items: [{
      item_id: params.itemId,
      item_name: params.itemName,
      item_category: params.category,
      price: params.price,
    }],
  });
};

// Auth Events
export const trackLogin = async (method: string) => {
  await analytics().logLogin({ method });
};

export const trackSignUp = async (method: string) => {
  await analytics().logSignUp({ method });
};

// Search Events
export const trackSearch = async (searchTerm: string) => {
  await analytics().logSearch({ search_term: searchTerm });
};

// Content Events
export const trackShare = async (params: {
  contentType: string;
  contentId: string;
  method: string;
}) => {
  await analytics().logShare({
    content_type: params.contentType,
    item_id: params.contentId,
    method: params.method,
  });
};
```

### Custom Events

```typescript
// src/analytics/customEvents.ts
import analytics from '@react-native-firebase/analytics';

// กำหนด event names เป็น constants
export const EVENTS = {
  // Onboarding
  ONBOARDING_STARTED: 'onboarding_started',
  ONBOARDING_COMPLETED: 'onboarding_completed',
  ONBOARDING_SKIPPED: 'onboarding_skipped',
  
  // Profile
  PROFILE_UPDATED: 'profile_updated',
  PROFILE_PICTURE_CHANGED: 'profile_picture_changed',
  
  // Features
  FEATURE_USED: 'feature_used',
  FILTER_APPLIED: 'filter_applied',
  
  // Errors
  ERROR_OCCURRED: 'error_occurred',
  
  // Performance
  APP_LOADED: 'app_loaded',
} as const;

export const trackEvent = async (
  eventName: keyof typeof EVENTS | string,
  params?: Record<string, string | number | boolean>
) => {
  const name = EVENTS[eventName as keyof typeof EVENTS] || eventName;
  
  try {
    await analytics().logEvent(name, {
      ...params,
      timestamp: Date.now(),
    });
  } catch (error) {
    console.error('Analytics error:', error);
  }
};

// ตัวอย่างการใช้งาน
export const trackOnboardingCompleted = async (duration: number) => {
  await trackEvent('ONBOARDING_COMPLETED', {
    duration_seconds: duration,
  });
};

export const trackFeatureUsed = async (featureName: string, duration?: number) => {
  await trackEvent('FEATURE_USED', {
    feature_name: featureName,
    duration_seconds: duration,
  });
};

export const trackError = async (errorCode: string, errorMessage: string, screen: string) => {
  await trackEvent('ERROR_OCCURRED', {
    error_code: errorCode,
    error_message: errorMessage.substring(0, 100),
    screen,
  });
};
```

---

## 3. Screen Tracking

### Navigation Analytics Hook

```typescript
// src/hooks/useAnalyticsNavigationTracking.ts
import { useEffect, useRef } from 'react';
import { NavigationContainerRef, NavigationState } from '@react-navigation/native';
import analytics from '@react-native-firebase/analytics';

export const useAnalyticsNavigationTracking = (
  navigationRef: React.RefObject<NavigationContainerRef<any>>
) => {
  const routeNameRef = useRef<string>('');

  const getActiveRouteName = (state: NavigationState): string => {
    const route = state.routes[state.index];
    
    if (route.state) {
      return getActiveRouteName(route.state as NavigationState);
    }
    
    return route.name;
  };

  const onStateChange = async (state?: NavigationState) => {
    if (!state) return;
    
    const currentRouteName = getActiveRouteName(state);
    const previousRouteName = routeNameRef.current;

    if (currentRouteName !== previousRouteName) {
      // Log screen view
      await analytics().logScreenView({
        screen_name: currentRouteName,
        screen_class: currentRouteName,
      });
      
      // Track screen view with custom event for more detail
      await analytics().logEvent('screen_view_custom', {
        screen_name: currentRouteName,
        previous_screen: previousRouteName,
        timestamp: Date.now(),
      });
    }
    
    routeNameRef.current = currentRouteName;
  };

  return { onStateChange };
};
```

### ใช้งานใน App.tsx

```typescript
// App.tsx
import React, { useRef } from 'react';
import { NavigationContainer, NavigationContainerRef } from '@react-navigation/native';
import { useAnalyticsNavigationTracking } from './src/hooks/useAnalyticsNavigationTracking';
import AppNavigator from './src/navigation/AppNavigator';

const App: React.FC = () => {
  const navigationRef = useRef<NavigationContainerRef<any>>(null);
  const { onStateChange } = useAnalyticsNavigationTracking(navigationRef);

  return (
    <NavigationContainer
      ref={navigationRef}
      onStateChange={onStateChange}
    >
      <AppNavigator />
    </NavigationContainer>
  );
};

export default App;
```

---

## 4. Mixpanel

### ติดตั้ง Mixpanel

```bash
npm install mixpanel-react-native
cd ios && pod install
```

### ตั้งค่า Mixpanel

```typescript
// src/analytics/mixpanel.ts
import { Mixpanel, MixpanelProperties } from 'mixpanel-react-native';
import { AnalyticsProvider, AnalyticsEvent, UserProperties } from './index';

const TOKEN = process.env.MIXPANEL_TOKEN || '';

class MixpanelProvider implements AnalyticsProvider {
  private client: Mixpanel;

  constructor() {
    this.client = new Mixpanel(TOKEN, true); // true = track automatic events
  }

  async initialize(): Promise<void> {
    await this.client.init();
    
    // ตั้งค่า super properties (ส่งไปทุก event)
    this.client.registerSuperProperties({
      platform: Platform.OS,
      app_version: DeviceInfo.getVersion(),
    });
  }

  setUserId(userId: string): void {
    this.client.identify(userId);
  }

  setUserProperties(properties: UserProperties): void {
    const peopleProps: MixpanelProperties = {};
    
    Object.entries(properties).forEach(([key, value]) => {
      if (value !== undefined) {
        peopleProps[`$${key}`] = value; // Mixpanel uses $ prefix for reserved props
      }
    });
    
    // Update People profile
    this.client.getPeople().set(peopleProps);
  }

  logEvent({ name, params }: AnalyticsEvent): void {
    this.client.track(name, params as MixpanelProperties);
  }

  logScreenView(screenName: string): void {
    this.client.track('Page View', {
      page: screenName,
    });
  }

  reset(): void {
    this.client.reset();
  }
}

export default new MixpanelProvider();
```

### Mixpanel Features ขั้นสูง

```typescript
import { Mixpanel } from 'mixpanel-react-native';

const mixpanel = new Mixpanel(TOKEN, true);

// People Analytics
const trackUserAction = async (userId: string) => {
  // Set user properties
  mixpanel.getPeople().set({
    $name: 'John Doe',
    $email: 'john@example.com',
    plan: 'premium',
    signupDate: new Date().toISOString(),
  });
  
  // Increment numeric property
  mixpanel.getPeople().increment('purchases', 1);
  mixpanel.getPeople().increment('total_spend', 299.99);
  
  // Append to list property
  mixpanel.getPeople().append('visited_screens', 'ProductDetail');
  
  // Track time between events
  mixpanel.timeEvent('checkout_completed');
  // ... user checks out ...
  mixpanel.track('checkout_completed'); // records duration automatically
};

// Funnels
const trackOnboardingFunnel = async (step: number) => {
  const stepNames = [
    'profile_setup',
    'preferences_selected', 
    'tutorial_completed',
    'first_action',
  ];
  
  if (step < stepNames.length) {
    mixpanel.track('onboarding_step_completed', {
      step_number: step,
      step_name: stepNames[step],
    });
  }
};

// Groups (organization-level tracking)
const trackOrganizationEvent = (orgId: string, eventName: string) => {
  mixpanel.setGroup('organization_id', orgId);
  mixpanel.track(eventName, { organization_id: orgId });
};
```

---

## 5. Amplitude

### ติดตั้ง Amplitude

```bash
npm install @amplitude/analytics-react-native
cd ios && pod install
```

### ตั้งค่า Amplitude

```typescript
// src/analytics/amplitude.ts
import * as amplitude from '@amplitude/analytics-react-native';
import { AnalyticsProvider, AnalyticsEvent, UserProperties } from './index';

const API_KEY = process.env.AMPLITUDE_API_KEY || '';

class AmplitudeProvider implements AnalyticsProvider {
  async initialize(): Promise<void> {
    await amplitude.init(API_KEY, undefined, {
      trackingSessionEvents: true,
      useBatch: true,
      serverUrl: 'https://api.amplitude.com/2/httpapi',
      logLevel: amplitude.Types.LogLevel.Warn,
    });
  }

  setUserId(userId: string): void {
    amplitude.setUserId(userId);
  }

  setUserProperties(properties: UserProperties): void {
    const identify = new amplitude.Identify();
    
    Object.entries(properties).forEach(([key, value]) => {
      if (value !== undefined) {
        identify.set(key, value);
      }
    });
    
    amplitude.identify(identify);
  }

  logEvent({ name, params }: AnalyticsEvent): void {
    amplitude.track(name, params);
  }

  logScreenView(screenName: string): void {
    amplitude.track('[Amplitude] Page Viewed', {
      page_title: screenName,
      page_url: screenName,
    });
  }

  reset(): void {
    amplitude.reset();
  }
}

// Revenue tracking
export const trackRevenue = (amount: number, productId: string, quantity: number = 1) => {
  const revenue = new amplitude.Revenue()
    .setProductId(productId)
    .setQuantity(quantity)
    .setPrice(amount);
    
  amplitude.revenue(revenue);
};

export default new AmplitudeProvider();
```

---

## 6. Analytics Manager (Multi-Provider)

### สร้าง Analytics Manager

```typescript
// src/analytics/AnalyticsManager.ts
import { AnalyticsProvider, AnalyticsEvent, UserProperties } from './index';
import FirebaseAnalytics from './firebase';
import MixpanelAnalytics from './mixpanel';
import AmplitudeAnalytics from './amplitude';

class AnalyticsManager {
  private providers: AnalyticsProvider[] = [];
  private isEnabled: boolean = true;

  constructor() {
    // เพิ่ม providers ตาม environment
    if (process.env.FIREBASE_ENABLED === 'true') {
      this.providers.push(FirebaseAnalytics);
    }
    if (process.env.MIXPANEL_ENABLED === 'true') {
      this.providers.push(MixpanelAnalytics);
    }
    if (process.env.AMPLITUDE_ENABLED === 'true') {
      this.providers.push(AmplitudeAnalytics);
    }
  }

  async initialize(): Promise<void> {
    await Promise.all(
      this.providers.map(provider => provider.initialize())
    );
  }

  setEnabled(enabled: boolean): void {
    this.isEnabled = enabled;
  }

  setUserId(userId: string): void {
    if (!this.isEnabled) return;
    this.providers.forEach(p => p.setUserId(userId));
  }

  setUserProperties(properties: UserProperties): void {
    if (!this.isEnabled) return;
    this.providers.forEach(p => p.setUserProperties(properties));
  }

  logEvent(event: AnalyticsEvent): void {
    if (!this.isEnabled) return;
    
    if (__DEV__) {
      console.log('[Analytics]', event.name, event.params);
    }
    
    this.providers.forEach(p => p.logEvent(event));
  }

  logScreenView(screenName: string, screenClass?: string): void {
    if (!this.isEnabled) return;
    this.providers.forEach(p => p.logScreenView(screenName, screenClass));
  }

  track(eventName: string, params?: Record<string, unknown>): void {
    this.logEvent({ name: eventName, params });
  }

  reset(): void {
    this.providers.forEach(p => p.reset());
  }
}

const Analytics = new AnalyticsManager();
export default Analytics;
```

### Custom Hook สำหรับ Analytics

```typescript
// src/hooks/useAnalytics.ts
import { useCallback } from 'react';
import Analytics from '../analytics/AnalyticsManager';

interface UseAnalyticsReturn {
  track: (eventName: string, params?: Record<string, unknown>) => void;
  trackScreenView: (screenName: string) => void;
  setUser: (userId: string, properties?: Record<string, unknown>) => void;
  resetUser: () => void;
}

export const useAnalytics = (): UseAnalyticsReturn => {
  const track = useCallback((eventName: string, params?: Record<string, unknown>) => {
    Analytics.track(eventName, params);
  }, []);

  const trackScreenView = useCallback((screenName: string) => {
    Analytics.logScreenView(screenName);
  }, []);

  const setUser = useCallback((userId: string, properties?: Record<string, unknown>) => {
    Analytics.setUserId(userId);
    if (properties) {
      Analytics.setUserProperties(properties as any);
    }
  }, []);

  const resetUser = useCallback(() => {
    Analytics.reset();
  }, []);

  return { track, trackScreenView, setUser, resetUser };
};
```

---

## 7. Event Tracking ขั้นสูง

### Funnel Tracking

```typescript
// src/analytics/funnels.ts
import Analytics from './AnalyticsManager';

export const FUNNELS = {
  REGISTRATION: 'registration',
  ONBOARDING: 'onboarding',
  PURCHASE: 'purchase',
  FEATURE_ADOPTION: 'feature_adoption',
} as const;

class FunnelTracker {
  private stepTimestamps: Map<string, number> = new Map();

  startFunnel(funnelName: string, metadata?: Record<string, unknown>): void {
    const key = `${funnelName}_start`;
    this.stepTimestamps.set(key, Date.now());
    
    Analytics.track(`${funnelName}_funnel_started`, {
      funnel: funnelName,
      ...metadata,
    });
  }

  trackStep(
    funnelName: string,
    step: number,
    stepName: string,
    metadata?: Record<string, unknown>
  ): void {
    const startKey = `${funnelName}_start`;
    const startTime = this.stepTimestamps.get(startKey);
    const elapsedTime = startTime ? Date.now() - startTime : 0;
    
    Analytics.track(`${funnelName}_step_${step}`, {
      funnel: funnelName,
      step,
      step_name: stepName,
      elapsed_time_ms: elapsedTime,
      ...metadata,
    });
  }

  completeFunnel(funnelName: string, metadata?: Record<string, unknown>): void {
    const startKey = `${funnelName}_start`;
    const startTime = this.stepTimestamps.get(startKey);
    const totalTime = startTime ? Date.now() - startTime : 0;
    
    Analytics.track(`${funnelName}_funnel_completed`, {
      funnel: funnelName,
      total_time_ms: totalTime,
      ...metadata,
    });
    
    this.stepTimestamps.delete(startKey);
  }

  abandonFunnel(funnelName: string, atStep: number, reason?: string): void {
    Analytics.track(`${funnelName}_funnel_abandoned`, {
      funnel: funnelName,
      at_step: atStep,
      reason,
    });
    
    this.stepTimestamps.delete(`${funnelName}_start`);
  }
}

export const funnelTracker = new FunnelTracker();
```

### User Session Tracking

```typescript
// src/analytics/session.ts
import Analytics from './AnalyticsManager';
import AsyncStorage from '@react-native-async-storage/async-storage';

const SESSION_KEY = '@analytics_session';

interface Session {
  id: string;
  startTime: number;
  screenViews: number;
  eventsTracked: number;
}

class SessionTracker {
  private currentSession: Session | null = null;

  async startSession(): Promise<void> {
    const sessionId = `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    
    this.currentSession = {
      id: sessionId,
      startTime: Date.now(),
      screenViews: 0,
      eventsTracked: 0,
    };
    
    await AsyncStorage.setItem(SESSION_KEY, JSON.stringify(this.currentSession));
    
    Analytics.track('session_started', {
      session_id: sessionId,
    });
  }

  async endSession(): Promise<void> {
    if (!this.currentSession) return;
    
    const duration = Date.now() - this.currentSession.startTime;
    
    Analytics.track('session_ended', {
      session_id: this.currentSession.id,
      duration_seconds: Math.floor(duration / 1000),
      screen_views: this.currentSession.screenViews,
      events_tracked: this.currentSession.eventsTracked,
    });
    
    await AsyncStorage.removeItem(SESSION_KEY);
    this.currentSession = null;
  }

  trackScreenView(): void {
    if (this.currentSession) {
      this.currentSession.screenViews++;
    }
  }

  trackEvent(): void {
    if (this.currentSession) {
      this.currentSession.eventsTracked++;
    }
  }

  getSessionId(): string | undefined {
    return this.currentSession?.id;
  }
}

export const sessionTracker = new SessionTracker();
```

---

## Workshop: Analytics Implementation

### Analytics Dashboard Data Collection

```typescript
// screens/HomeScreen.tsx
import React, { useEffect } from 'react';
import { View, Text, ScrollView } from 'react-native';
import { useAnalytics } from '../hooks/useAnalytics';
import { sessionTracker } from '../analytics/session';

const HomeScreen: React.FC = () => {
  const { track, trackScreenView } = useAnalytics();

  useEffect(() => {
    // Track screen view
    trackScreenView('HomeScreen');
    sessionTracker.trackScreenView();
    
    // Track time on screen
    const startTime = Date.now();
    
    return () => {
      const timeOnScreen = Date.now() - startTime;
      track('screen_time', {
        screen: 'HomeScreen',
        duration_ms: timeOnScreen,
      });
    };
  }, []);

  const handleFeatureTap = (featureName: string) => {
    track('feature_tapped', {
      feature: featureName,
      screen: 'HomeScreen',
    });
  };

  return (
    <ScrollView>
      <Text>Home Screen</Text>
    </ScrollView>
  );
};
```

### Analytics Privacy

```typescript
// src/analytics/privacyManager.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import Analytics from './AnalyticsManager';

const ANALYTICS_CONSENT_KEY = '@analytics_consent';

export const setAnalyticsConsent = async (hasConsent: boolean): Promise<void> => {
  await AsyncStorage.setItem(ANALYTICS_CONSENT_KEY, JSON.stringify(hasConsent));
  Analytics.setEnabled(hasConsent);
};

export const getAnalyticsConsent = async (): Promise<boolean> => {
  const stored = await AsyncStorage.getItem(ANALYTICS_CONSENT_KEY);
  return stored ? JSON.parse(stored) : false;
};

export const initializeAnalyticsWithConsent = async (): Promise<void> => {
  const hasConsent = await getAnalyticsConsent();
  
  if (hasConsent) {
    await Analytics.initialize();
  }
};
```

---

## Tips และ Best Practices

### 1. Event Naming Convention

```
# format: noun_verb หรือ object_action
user_registered       ✓
button_clicked        ✗ (too generic)
purchase_completed    ✓
signup_button_tapped  ✓

# ใช้ snake_case
product_viewed         ✓
productViewed          ✗
ProductViewed          ✗
```

### 2. Sanitize Data

```typescript
// ลบ PII ออกจาก analytics
const sanitizeParams = (params: Record<string, unknown>) => {
  const sensitiveKeys = ['email', 'phone', 'creditCard', 'ssn', 'password'];
  
  return Object.entries(params)
    .filter(([key]) => !sensitiveKeys.some(k => key.toLowerCase().includes(k)))
    .reduce((acc, [key, value]) => ({ ...acc, [key]: value }), {});
};
```

### 3. Test Analytics ใน Development

```typescript
// ใน development mode, log แทนส่ง
if (__DEV__) {
  console.group(`[Analytics] ${eventName}`);
  console.log('Params:', params);
  console.groupEnd();
  return; // ไม่ส่งจริงใน dev
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Firebase Analytics**: Setup และ custom events
2. **Mixpanel**: Advanced people analytics
3. **Amplitude**: Behavioral analytics
4. **Analytics Manager**: Multi-provider architecture
5. **Funnel Tracking**: ติดตาม user journeys
6. **Privacy**: จัดการ consent และ data protection
