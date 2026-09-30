# Part 099: App Store Optimization (ASO)

## บทนำ

ASO (App Store Optimization) คือกระบวนการเพิ่ม visibility ของ app ใน App Store และ Google Play Store เพื่อให้มีผู้ดาวน์โหลดมากขึ้น เหมือน SEO แต่สำหรับ App Store

## หัวข้อที่จะเรียน

1. App Metadata Optimization
2. Screenshots และ Videos
3. Keyword Research และ Optimization
4. Ratings และ Reviews Management
5. A/B Testing สำหรับ Stores
6. Workshop: ASO Strategy

---

## 1. App Metadata Optimization

### App Name (ชื่อ App)

ชื่อ app มีผลต่อ search ranking อย่างมาก

**หลักการ:**
- ชื่อหลัก + Keyword สำคัญ
- ไม่เกิน 30 ตัวอักษร (App Store) / 50 ตัวอักษร (Play Store)
- Unique และจำง่าย
- ตรงกับ brand

**ตัวอย่างที่ดี:**
```
Grab - Ride, Food & Delivery
LINE: Calls & Messages
Lazada - ช้อปปิ้งออนไลน์
```

**เครื่องมือวิเคราะห์:**
```typescript
// tools/aso-analyzer.ts
interface AppMetadata {
  name: string;
  subtitle?: string; // iOS only
  description: string;
  keywords: string; // iOS only (100 chars limit)
  whatsnew?: string;
}

const analyzeMetadata = (metadata: AppMetadata) => {
  const issues: string[] = [];
  const suggestions: string[] = [];

  // ตรวจสอบชื่อ
  if (metadata.name.length > 30) {
    issues.push(`ชื่อยาวเกินไป: ${metadata.name.length}/30 ตัวอักษร`);
  }
  if (!hasKeyword(metadata.name)) {
    suggestions.push('พิจารณาเพิ่ม keyword สำคัญในชื่อ');
  }

  // ตรวจสอบ description
  if (metadata.description.length < 500) {
    suggestions.push('Description สั้นเกินไป ควรมีอย่างน้อย 500 ตัวอักษร');
  }
  if (!hasCallToAction(metadata.description)) {
    suggestions.push('เพิ่ม Call-to-Action ใน description');
  }

  // iOS Keywords
  if (metadata.keywords && metadata.keywords.length > 100) {
    issues.push(`Keywords ยาวเกิน: ${metadata.keywords.length}/100 ตัวอักษร`);
  }

  return { issues, suggestions };
};

const hasKeyword = (text: string): boolean => {
  const keywords = ['app', 'แอป', 'ดาวน์โหลด', 'ฟรี', 'ออนไลน์'];
  return keywords.some(k => text.toLowerCase().includes(k));
};

const hasCallToAction = (text: string): boolean => {
  const ctas = ['ดาวน์โหลด', 'ลงทะเบียน', 'เริ่มต้น', 'ทดลองใช้', 'download'];
  return ctas.some(cta => text.toLowerCase().includes(cta));
};
```

### App Description Best Practices

```markdown
## App Description Template

### บรรทัดแรก (สำคัญที่สุด!)
[Headline ที่น่าสนใจ ภายใน 3 บรรทัดแรก - ส่วนนี้จะแสดงก่อน "more"]

### Features Section
✨ คุณสมบัติหลัก:
• Feature 1 - อธิบายสั้นๆ
• Feature 2 - อธิบายสั้นๆ
• Feature 3 - อธิบายสั้นๆ

### Social Proof
⭐ ผู้ใช้มากกว่า 1 ล้านคนทั่วไทยเลือกใช้

### Keywords (แทรกอย่างเป็นธรรมชาติ)
[ใส่ keywords ในประโยค ไม่ใช่ list ธรรมดา]

### Call to Action
ดาวน์โหลดฟรีวันนี้ แล้วสัมผัสความแตกต่าง!

### Contact
📧 support@yourapp.com
🌐 www.yourapp.com
```

---

## 2. Screenshots และ Videos

### Screenshot Strategy

Screenshots เป็นสิ่งแรกที่ผู้ใช้เห็น ส่งผลต่อ conversion rate อย่างมาก

```typescript
// tools/screenshot-generator.ts
interface ScreenshotSpec {
  device: string;
  size: { width: number; height: number };
  required: boolean;
}

const REQUIRED_SCREENSHOTS: Record<string, ScreenshotSpec[]> = {
  ios: [
    // iPhone 15 Pro Max / 14 Plus
    { device: 'iPhone 6.7"', size: { width: 1290, height: 2796 }, required: true },
    // iPhone 15 Pro / 14
    { device: 'iPhone 6.1"', size: { width: 1179, height: 2556 }, required: true },
    // iPhone SE
    { device: 'iPhone 4.7"', size: { width: 750, height: 1334 }, required: false },
    // iPad Pro 12.9"
    { device: 'iPad 12.9"', size: { width: 2048, height: 2732 }, required: true },
  ],
  android: [
    { device: 'Phone', size: { width: 1080, height: 1920 }, required: true },
    { device: '7" Tablet', size: { width: 1200, height: 1920 }, required: false },
    { device: '10" Tablet', size: { width: 1600, height: 2560 }, required: false },
  ],
};

// Screenshot frame template
const SCREENSHOT_TEMPLATE = `
Screenshot ควรมี:
1. Background สีสวย / Gradient
2. Device mockup (โทรศัพท์จริง)
3. Headline ที่ชัดเจน (ไม่เกิน 5 คำ)
4. Subtext อธิบาย (1-2 ประโยค)
5. Feature highlight สำคัญ

การจัดเรียง:
Screenshot 1: Value Proposition หลัก
Screenshot 2-4: Features สำคัญ
Screenshot 5: Social proof / Awards
Screenshot 6: Call to action
`;
```

### Fastlane Snapshot สำหรับ Auto Screenshot

```ruby
# ios/fastlane/Snapfile

devices([
  "iPhone 15 Pro Max",
  "iPhone 15 Pro",
  "iPad Pro (12.9-inch) (6th generation)",
])

languages([
  "th-TH",
  "en-US",
])

scheme "App-Screenshots"

output_directory "./screenshots"
clear_previous_screenshots true

concurrent_simulators true
```

```swift
// ios/SnapshotTests/SnapshotTests.swift
import XCTest

class SnapshotTests: XCTestCase {
  func testHomeScreen() {
    let app = XCUIApplication()
    setupSnapshot(app)
    app.launch()
    
    // ไปที่ Home screen
    snapshot("01_HomeScreen")
  }
  
  func testProductList() {
    let app = XCUIApplication()
    setupSnapshot(app)
    app.launch()
    
    app.tabBars.buttons["Products"].tap()
    snapshot("02_ProductList")
  }
  
  func testCheckout() {
    let app = XCUIApplication()
    setupSnapshot(app)
    app.launch()
    
    // Navigate to checkout
    app.tabBars.buttons["Cart"].tap()
    snapshot("03_Checkout")
  }
}
```

---

## 3. Keyword Research

### Keyword Strategy

```typescript
// tools/keyword-analyzer.ts
interface KeywordData {
  keyword: string;
  volume: number; // 1-10 scale
  difficulty: number; // 1-10 scale
  relevance: number; // 1-10 scale
  ranking?: number; // current ranking
}

const calculateKeywordScore = (kw: KeywordData): number => {
  // Score = (Volume * 0.4) + (Relevance * 0.4) - (Difficulty * 0.2)
  return (kw.volume * 0.4) + (kw.relevance * 0.4) - (kw.difficulty * 0.2);
};

const prioritizeKeywords = (keywords: KeywordData[]): KeywordData[] => {
  return keywords
    .map(kw => ({ ...kw, score: calculateKeywordScore(kw) }))
    .sort((a, b) => (b as any).score - (a as any).score);
};

// ตัวอย่างสำหรับ food delivery app
const foodDeliveryKeywords: KeywordData[] = [
  { keyword: 'สั่งอาหาร', volume: 9, difficulty: 9, relevance: 10 },
  { keyword: 'food delivery', volume: 8, difficulty: 8, relevance: 10 },
  { keyword: 'สั่งอาหารออนไลน์', volume: 8, difficulty: 7, relevance: 10 },
  { keyword: 'ส่งอาหาร', volume: 9, difficulty: 8, relevance: 9 },
  { keyword: 'อาหารใกล้ฉัน', volume: 7, difficulty: 6, relevance: 9 },
  { keyword: 'เดลิเวอรี่', volume: 8, difficulty: 8, relevance: 9 },
  { keyword: 'ร้านอาหาร', volume: 9, difficulty: 9, relevance: 8 },
];

// iOS Keywords field (100 characters limit)
const buildIOSKeywords = (keywords: string[]): string => {
  let result = '';
  for (const kw of keywords) {
    const addition = result ? `,${kw}` : kw;
    if ((result + addition).length > 100) break;
    result += addition;
  }
  return result;
};
```

### Keyword Tracking Tool

```typescript
// tools/keyword-tracker.ts
import axios from 'axios';

interface RankingResult {
  keyword: string;
  rank: number | null;
  change: number | null; // เปลี่ยนแปลงจากสัปดาห์ก่อน
  date: Date;
}

class KeywordTracker {
  private history: Map<string, RankingResult[]> = new Map();

  async trackKeywords(
    appId: string,
    keywords: string[],
    country: string = 'th'
  ): Promise<RankingResult[]> {
    const results: RankingResult[] = [];
    
    for (const keyword of keywords) {
      const rank = await this.searchRank(appId, keyword, country);
      const previousRankings = this.history.get(keyword) || [];
      const lastRanking = previousRankings[previousRankings.length - 1];
      
      const result: RankingResult = {
        keyword,
        rank,
        change: lastRanking?.rank !== undefined 
          ? (lastRanking.rank || 0) - (rank || 0) 
          : null,
        date: new Date(),
      };
      
      // บันทึก history
      this.history.set(keyword, [...previousRankings, result]);
      results.push(result);
    }
    
    return results;
  }

  private async searchRank(
    appId: string,
    keyword: string,
    country: string
  ): Promise<number | null> {
    // ใช้ App Store Search API หรือ scraping tool
    // ตัวอย่างนี้เป็น mock
    return Math.floor(Math.random() * 100) + 1;
  }

  generateReport(results: RankingResult[]): string {
    const improved = results.filter(r => r.change && r.change > 0);
    const declined = results.filter(r => r.change && r.change < 0);
    const top10 = results.filter(r => r.rank && r.rank <= 10);

    return `
# Keyword Ranking Report - ${new Date().toLocaleDateString('th-TH')}

## สรุป
- Keywords ทั้งหมด: ${results.length}
- Top 10: ${top10.length} keywords
- ขึ้น: ${improved.length} keywords
- ลง: ${declined.length} keywords

## Top Rankings
${results
  .filter(r => r.rank && r.rank <= 50)
  .sort((a, b) => (a.rank || 999) - (b.rank || 999))
  .slice(0, 10)
  .map(r => `- "${r.keyword}": #${r.rank} ${r.change ? (r.change > 0 ? `(↑${r.change})` : `(↓${Math.abs(r.change)})`) : '(new)'}`)
  .join('\n')}
    `;
  }
}
```

---

## 4. Ratings และ Reviews

### In-App Review Request Strategy

```typescript
// src/utils/reviewManager.ts
import { Alert, Linking, Platform } from 'react-native';
import InAppReview from 'react-native-in-app-review';
import AsyncStorage from '@react-native-async-storage/async-storage';

const REVIEW_KEYS = {
  LAST_REQUESTED: '@review_last_requested',
  TIMES_LAUNCHED: '@times_launched',
  NEGATIVE_FEEDBACK: '@negative_feedback',
  HAS_RATED: '@has_rated',
};

class ReviewManager {
  private static instance: ReviewManager;

  static getInstance(): ReviewManager {
    if (!ReviewManager.instance) {
      ReviewManager.instance = new ReviewManager();
    }
    return ReviewManager.instance;
  }

  async trackLaunch(): Promise<void> {
    const launches = parseInt(
      await AsyncStorage.getItem(REVIEW_KEYS.TIMES_LAUNCHED) || '0'
    );
    await AsyncStorage.setItem(
      REVIEW_KEYS.TIMES_LAUNCHED,
      String(launches + 1)
    );
  }

  async shouldRequestReview(): Promise<boolean> {
    const [hasRated, lastRequested, launches] = await Promise.all([
      AsyncStorage.getItem(REVIEW_KEYS.HAS_RATED),
      AsyncStorage.getItem(REVIEW_KEYS.LAST_REQUESTED),
      AsyncStorage.getItem(REVIEW_KEYS.TIMES_LAUNCHED),
    ]);

    // ไม่ขอถ้า rated แล้ว
    if (hasRated === 'true') return false;

    const launchCount = parseInt(launches || '0');
    const lastDate = lastRequested ? new Date(lastRequested) : null;
    const daysSinceLastRequest = lastDate
      ? (Date.now() - lastDate.getTime()) / (1000 * 60 * 60 * 24)
      : Infinity;

    // ขอหลังจาก launch 10 ครั้ง และ 30 วันนับจากครั้งล่าสุด
    return launchCount >= 10 && daysSinceLastRequest >= 30;
  }

  async requestReview(): Promise<void> {
    const should = await this.shouldRequestReview();
    if (!should) return;

    // Check if in-app review available
    const isAvailable = InAppReview.isAvailable();
    
    if (isAvailable) {
      // ใช้ native in-app review (iOS/Android)
      const hasFlowed = await InAppReview.RequestInAppReview();
      if (hasFlowed) {
        await AsyncStorage.setItem(REVIEW_KEYS.LAST_REQUESTED, new Date().toISOString());
        await AsyncStorage.setItem(REVIEW_KEYS.HAS_RATED, 'true');
      }
    } else {
      // Fallback: แสดง dialog ขอไป store
      this.showReviewDialog();
    }
  }

  private showReviewDialog(): void {
    Alert.alert(
      'คุณชอบแอปของเราไหม? 😊',
      'ช่วยให้คะแนนและรีวิวเพื่อช่วยให้เราพัฒนาต่อไปได้นะ',
      [
        {
          text: 'ให้คะแนนเลย! ⭐',
          onPress: () => {
            this.openStoreReview();
            AsyncStorage.setItem(REVIEW_KEYS.HAS_RATED, 'true');
          },
        },
        {
          text: 'ไว้ทีหลัง',
          onPress: () => {
            AsyncStorage.setItem(REVIEW_KEYS.LAST_REQUESTED, new Date().toISOString());
          },
        },
        {
          text: 'ไม่ขอบคุณ',
          style: 'cancel',
          onPress: () => {
            // บันทึกว่าไม่ต้องการ review
            AsyncStorage.setItem(REVIEW_KEYS.HAS_RATED, 'true');
          },
        },
      ]
    );
  }

  private openStoreReview(): void {
    const APP_ID = 'YOUR_APP_ID';
    
    if (Platform.OS === 'ios') {
      Linking.openURL(
        `https://apps.apple.com/app/id${APP_ID}?action=write-review`
      );
    } else {
      Linking.openURL(
        `market://details?id=com.yourcompany.app&showAllReviews=true`
      );
    }
  }

  // สำหรับ negative feedback - redirect ไป support แทน Store
  async handleUserSatisfaction(isSatisfied: boolean): Promise<void> {
    if (isSatisfied) {
      await this.requestReview();
    } else {
      // redirect ไป feedback form
      await AsyncStorage.setItem(REVIEW_KEYS.NEGATIVE_FEEDBACK, 'true');
      Alert.alert(
        'ขอโทษด้วยนะ 😔',
        'บอกเราว่าเราทำอะไรได้ดีกว่านี้',
        [
          {
            text: 'ส่ง Feedback',
            onPress: () => Linking.openURL('mailto:support@yourapp.com'),
          },
          { text: 'ปิด', style: 'cancel' },
        ]
      );
    }
  }
}

export default ReviewManager;
```

### Review Response Strategy

```typescript
// tools/review-monitor.ts
interface AppReview {
  id: string;
  rating: number;
  title: string;
  body: string;
  authorName: string;
  date: Date;
  platform: 'ios' | 'android';
  responded: boolean;
}

const RESPONSE_TEMPLATES = {
  positive5Star: `ขอบคุณมากสำหรับรีวิว 5 ดาว! 🌟 
เรายินดีมากที่คุณชอบแอปของเรา 
หากมีข้อเสนอแนะหรือต้องการความช่วยเหลือ ติดต่อเราได้ที่ support@yourapp.com ครับ/ค่ะ`,
  
  negative12Star: `ขอโทษที่ทำให้คุณไม่พอใจนะครับ/ค่ะ 😔
เราอยากแก้ไขปัญหาให้คุณ 
กรุณาติดต่อเราที่ support@yourapp.com หรือ LINE: @yourapp 
เพื่อให้เราช่วยเหลือได้โดยตรงครับ/ค่ะ`,
  
  bug34Star: `ขอบคุณที่แจ้งปัญหาครับ/ค่ะ 🙏
ทีมงานได้รับทราบและกำลังดำเนินการแก้ไข
update ใหม่จะออกเร็วๆ นี้ 
ระหว่างนี้หากต้องการความช่วยเหลือเพิ่มเติม ติดต่อ support@yourapp.com ครับ/ค่ะ`,
};

class ReviewMonitor {
  async analyzeReviews(reviews: AppReview[]) {
    const stats = {
      totalReviews: reviews.length,
      averageRating: reviews.reduce((sum, r) => sum + r.rating, 0) / reviews.length,
      distribution: { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 },
      responseRate: reviews.filter(r => r.responded).length / reviews.length,
      criticalUnresponded: reviews.filter(r => r.rating <= 2 && !r.responded),
    };

    for (const review of reviews) {
      stats.distribution[review.rating as keyof typeof stats.distribution]++;
    }

    return stats;
  }

  suggestResponse(review: AppReview): string {
    if (review.rating === 5) return RESPONSE_TEMPLATES.positive5Star;
    if (review.rating <= 2) return RESPONSE_TEMPLATES.negative12Star;
    return RESPONSE_TEMPLATES.bug34Star;
  }
}
```

---

## 5. A/B Testing สำหรับ App Store

### App Store Connect Custom Product Pages (iOS)

```typescript
// สร้าง metadata variants สำหรับ A/B testing
interface ProductPageVariant {
  id: string;
  name: string;
  title?: string;
  subtitle?: string;
  screenshots: string[];
  promotionalText?: string;
  targetAudience: string;
}

const variants: ProductPageVariant[] = [
  {
    id: 'control',
    name: 'Control - Current',
    title: 'MyApp - ช้อปปิ้งออนไลน์',
    screenshots: ['screenshot-1.png', 'screenshot-2.png'],
    targetAudience: 'General',
  },
  {
    id: 'variant-a',
    name: 'Variant A - Value Focus',
    title: 'MyApp - ราคาถูก ส่งฟรี',
    screenshots: ['screenshot-promo-1.png', 'screenshot-promo-2.png'],
    promotionalText: 'ลดสูงสุด 70%! สมัครวันนี้รับ Voucher 100 บาท',
    targetAudience: 'Price-conscious shoppers',
  },
  {
    id: 'variant-b',
    name: 'Variant B - Convenience Focus',
    title: 'MyApp - ส่งถึงบ้านใน 1 ชั่วโมง',
    screenshots: ['screenshot-delivery-1.png', 'screenshot-delivery-2.png'],
    targetAudience: 'Busy professionals',
  },
];

// Track performance ของแต่ละ variant
interface VariantMetrics {
  variantId: string;
  impressions: number;
  pageViews: number;
  installs: number;
  conversionRate: number;
  daysSinceStart: number;
}

const determineWinner = (metrics: VariantMetrics[]): string | null => {
  // ต้องการ statistical significance (p < 0.05)
  const MINIMUM_SAMPLE_SIZE = 1000;
  const MINIMUM_DAYS = 7;
  
  const validMetrics = metrics.filter(m => 
    m.impressions >= MINIMUM_SAMPLE_SIZE && m.daysSinceStart >= MINIMUM_DAYS
  );
  
  if (validMetrics.length < 2) return null; // ยังไม่พอ data
  
  const sorted = validMetrics.sort((a, b) => b.conversionRate - a.conversionRate);
  const best = sorted[0];
  const second = sorted[1];
  
  // ต้องดีกว่า 10%
  if (best.conversionRate > second.conversionRate * 1.1) {
    return best.variantId;
  }
  
  return null; // ยังไม่สรุปได้
};
```

---

## 6. Workshop: ASO Strategy

### ASO Audit Script

```typescript
// tools/aso-audit.ts
interface ASOAuditResult {
  score: number; // 0-100
  items: AuditItem[];
}

interface AuditItem {
  category: string;
  item: string;
  status: 'pass' | 'fail' | 'warning';
  impact: 'high' | 'medium' | 'low';
  recommendation: string;
}

const runASOAudit = (metadata: any): ASOAuditResult => {
  const items: AuditItem[] = [];
  let totalPoints = 0;
  let earnedPoints = 0;

  // 1. App Name (20 points)
  const nameChecks = [
    { check: metadata.name.length <= 30, points: 10, item: 'ชื่อ ≤ 30 ตัวอักษร' },
    { check: /[ก-๙]|[a-z]/i.test(metadata.name), points: 5, item: 'มี Keywords ในชื่อ' },
    { check: metadata.name.length >= 10, points: 5, item: 'ชื่อยาวพอสมควร' },
  ];
  
  for (const check of nameChecks) {
    totalPoints += check.points;
    if (check.check) {
      earnedPoints += check.points;
      items.push({ category: 'App Name', item: check.item, status: 'pass', impact: 'high', recommendation: '' });
    } else {
      items.push({ category: 'App Name', item: check.item, status: 'fail', impact: 'high', recommendation: `ปรับปรุง: ${check.item}` });
    }
  }

  // 2. Description (30 points)
  const descChecks = [
    { check: metadata.description.length >= 1000, points: 15, item: 'Description ≥ 1000 ตัวอักษร' },
    { check: hasEmoji(metadata.description), points: 5, item: 'มี Emoji ในความอธิบาย' },
    { check: hasBulletPoints(metadata.description), points: 10, item: 'มี Bullet points' },
  ];
  
  for (const check of descChecks) {
    totalPoints += check.points;
    if (check.check) {
      earnedPoints += check.points;
      items.push({ category: 'Description', item: check.item, status: 'pass', impact: 'medium', recommendation: '' });
    } else {
      items.push({ category: 'Description', item: check.item, status: 'warning', impact: 'medium', recommendation: `พิจารณาเพิ่ม: ${check.item}` });
    }
  }

  // 3. Screenshots (30 points)
  const screenshotChecks = [
    { check: metadata.screenshots?.length >= 5, points: 15, item: 'มี Screenshots ≥ 5 รูป' },
    { check: metadata.hasVideo, points: 15, item: 'มี Preview Video' },
  ];
  
  for (const check of screenshotChecks) {
    totalPoints += check.points;
    if (check.check) {
      earnedPoints += check.points;
      items.push({ category: 'Screenshots', item: check.item, status: 'pass', impact: 'high', recommendation: '' });
    } else {
      items.push({ category: 'Screenshots', item: check.item, status: 'fail', impact: 'high', recommendation: `เพิ่ม: ${check.item}` });
    }
  }

  const score = Math.round((earnedPoints / totalPoints) * 100);
  return { score, items };
};

const hasEmoji = (text: string): boolean => /\p{Emoji}/u.test(text);
const hasBulletPoints = (text: string): boolean => /[•\-\*]/.test(text);
```

### Monthly ASO Report Template

```typescript
const generateMonthlyReport = (data: any) => `
# ASO Monthly Report - ${new Date().toLocaleDateString('th-TH', { month: 'long', year: 'numeric' })}

## Executive Summary
- App Store Rank: ${data.appStoreRank} (${data.rankChange > 0 ? `↑${data.rankChange}` : `↓${Math.abs(data.rankChange)}`})
- Google Play Rank: ${data.playStoreRank}
- Average Rating: ${data.averageRating.toFixed(1)} ⭐ (${data.totalRatings} ratings)
- Organic Installs: ${data.organicInstalls.toLocaleString()} (+${data.organicGrowth}%)

## Keyword Performance
Top 5 Keywords:
${data.topKeywords.map((k: any, i: number) => 
  `${i+1}. "${k.keyword}" - #${k.rank} (${k.change > 0 ? `↑${k.change}` : `↓${Math.abs(k.change)}`})`
).join('\n')}

## Actions for Next Month
1. ${data.recommendations[0]}
2. ${data.recommendations[1]}
3. ${data.recommendations[2]}
`;
```

---

## Workshop Exercises

1. **Keyword Research** - วิจัย keywords 20 คำสำหรับ app ของคุณ
2. **Screenshot Design** - ออกแบบ 5 screenshots ที่ convert ได้ดี
3. **Review Response** - สร้าง template สำหรับตอบรีวิวทุกประเภท
4. **A/B Test Setup** - วางแผน A/B test สำหรับ icon และ screenshots

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **App Metadata** - ชื่อ, description, keywords ที่ optimize
2. **Screenshots** - design และ strategy
3. **Keyword Research** - วิธีหา และ track keywords
4. **Reviews** - วิธีขอรีวิวและตอบอย่างมืออาชีพ
5. **A/B Testing** - ทดสอบ store listing variants

> **Pro Tip:** ASO เป็น long-term game ต้องทำอย่างสม่ำเสมอและวัดผลทุกเดือน
