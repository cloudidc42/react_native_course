# Part 043: Location และ Maps ใน React Native

## สารบัญ
1. [แนะนำ Location Services](#introduction)
2. [Geolocation API](#geolocation)
3. [react-native-maps](#react-native-maps)
4. [Markers, Polylines, Polygons](#markers)
5. [Directions API](#directions)
6. [Geocoding](#geocoding)
7. [Workshop: Location Tracking App](#workshop)

---

## 1. แนะนำ Location Services {#introduction}

Location Services เป็นฟีเจอร์สำคัญสำหรับแอปที่ต้องการระบุตำแหน่งของผู้ใช้

### ติดตั้ง Dependencies

```bash
# react-native-maps (Maps)
npm install react-native-maps

# Geolocation
npm install @react-native-community/geolocation
# หรือ
npm install react-native-geolocation-service

# สำหรับ Expo
npx expo install expo-location
npx expo install react-native-maps
```

### การตั้งค่า Permissions

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

```xml
<!-- ios/YourApp/Info.plist -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อแสดงสถานที่ใกล้เคียง</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณตลอดเวลาสำหรับการติดตาม</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อการติดตาม</string>
```

---

## 2. Geolocation API {#geolocation}

### การใช้งาน Geolocation

```typescript
// src/hooks/useLocation.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import Geolocation from 'react-native-geolocation-service';
import { PermissionsAndroid, Platform, Alert, Linking } from 'react-native';

interface Location {
  latitude: number;
  longitude: number;
  altitude?: number;
  accuracy?: number;
  speed?: number;
  heading?: number;
  timestamp: number;
}

interface LocationError {
  code: number;
  message: string;
}

interface UseLocationOptions {
  enableHighAccuracy?: boolean;
  timeout?: number;
  maximumAge?: number;
  watchMode?: boolean;
  distanceFilter?: number;
}

export function useLocation(options: UseLocationOptions = {}) {
  const [location, setLocation] = useState<Location | null>(null);
  const [error, setError] = useState<LocationError | null>(null);
  const [loading, setLoading] = useState(false);
  const [permissionStatus, setPermissionStatus] = useState<'granted' | 'denied' | 'unknown'>('unknown');
  const watchId = useRef<number | null>(null);

  const {
    enableHighAccuracy = true,
    timeout = 15000,
    maximumAge = 10000,
    watchMode = false,
    distanceFilter = 10,
  } = options;

  // ขอ Permission
  const requestPermission = useCallback(async (): Promise<boolean> => {
    if (Platform.OS === 'android') {
      const result = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
        {
          title: 'ต้องการเข้าถึงตำแหน่ง',
          message: 'แอปต้องการตำแหน่งของคุณเพื่อแสดงสถานที่ใกล้เคียง',
          buttonPositive: 'อนุญาต',
          buttonNegative: 'ปฏิเสธ',
          buttonNeutral: 'ถามภายหลัง',
        }
      );

      if (result === PermissionsAndroid.RESULTS.GRANTED) {
        setPermissionStatus('granted');
        return true;
      } else {
        setPermissionStatus('denied');
        if (result === PermissionsAndroid.RESULTS.NEVER_ASK_AGAIN) {
          Alert.alert(
            'ต้องการสิทธิ์',
            'กรุณาเปิดสิทธิ์การเข้าถึงตำแหน่งในการตั้งค่าของอุปกรณ์',
            [
              { text: 'ยกเลิก', style: 'cancel' },
              { text: 'เปิดการตั้งค่า', onPress: () => Linking.openSettings() },
            ]
          );
        }
        return false;
      }
    } else {
      const result = await Geolocation.requestAuthorization('whenInUse');
      if (result === 'granted') {
        setPermissionStatus('granted');
        return true;
      } else {
        setPermissionStatus('denied');
        return false;
      }
    }
  }, []);

  // รับตำแหน่งครั้งเดียว
  const getCurrentLocation = useCallback(async (): Promise<Location | null> => {
    setLoading(true);
    setError(null);

    const hasPermission = await requestPermission();
    if (!hasPermission) {
      setLoading(false);
      return null;
    }

    return new Promise((resolve) => {
      Geolocation.getCurrentPosition(
        (position) => {
          const loc: Location = {
            latitude: position.coords.latitude,
            longitude: position.coords.longitude,
            altitude: position.coords.altitude ?? undefined,
            accuracy: position.coords.accuracy,
            speed: position.coords.speed ?? undefined,
            heading: position.coords.heading ?? undefined,
            timestamp: position.timestamp,
          };
          setLocation(loc);
          setLoading(false);
          resolve(loc);
        },
        (err) => {
          const locationError: LocationError = {
            code: err.code,
            message: getErrorMessage(err.code),
          };
          setError(locationError);
          setLoading(false);
          resolve(null);
        },
        {
          enableHighAccuracy,
          timeout,
          maximumAge,
        }
      );
    });
  }, [enableHighAccuracy, timeout, maximumAge, requestPermission]);

  // ติดตามตำแหน่งแบบ Real-time
  const startWatching = useCallback(async () => {
    if (watchId.current !== null) return;

    const hasPermission = await requestPermission();
    if (!hasPermission) return;

    watchId.current = Geolocation.watchPosition(
      (position) => {
        const loc: Location = {
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
          altitude: position.coords.altitude ?? undefined,
          accuracy: position.coords.accuracy,
          speed: position.coords.speed ?? undefined,
          heading: position.coords.heading ?? undefined,
          timestamp: position.timestamp,
        };
        setLocation(loc);
      },
      (err) => {
        setError({ code: err.code, message: getErrorMessage(err.code) });
      },
      {
        enableHighAccuracy,
        distanceFilter,
        interval: 5000,
        fastestInterval: 2000,
      }
    );
  }, [enableHighAccuracy, distanceFilter, requestPermission]);

  // หยุดติดตาม
  const stopWatching = useCallback(() => {
    if (watchId.current !== null) {
      Geolocation.clearWatch(watchId.current);
      watchId.current = null;
    }
  }, []);

  useEffect(() => {
    if (watchMode) {
      startWatching();
    } else {
      getCurrentLocation();
    }

    return () => {
      stopWatching();
    };
  }, [watchMode]);

  return {
    location,
    error,
    loading,
    permissionStatus,
    getCurrentLocation,
    startWatching,
    stopWatching,
  };
}

function getErrorMessage(code: number): string {
  switch (code) {
    case 1: return 'การเข้าถึงตำแหน่งถูกปฏิเสธ';
    case 2: return 'ไม่สามารถระบุตำแหน่งได้';
    case 3: return 'หมดเวลารอการระบุตำแหน่ง';
    default: return 'เกิดข้อผิดพลาดในการระบุตำแหน่ง';
  }
}
```

---

## 3. react-native-maps {#react-native-maps}

### การตั้งค่า Google Maps

```gradle
// android/app/build.gradle
android {
    defaultConfig {
        manifestPlaceholders = [
            GOOGLE_MAPS_API_KEY: "YOUR_GOOGLE_MAPS_API_KEY"
        ]
    }
}
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="${GOOGLE_MAPS_API_KEY}"/>
```

```objective-c
// ios/YourApp/AppDelegate.mm
#import <GoogleMaps/GoogleMaps.h>

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  [GMSServices provideAPIKey:@"YOUR_GOOGLE_MAPS_API_KEY"];
  // ...
}
```

### MapView พื้นฐาน

```typescript
// src/components/MapComponent.tsx
import React, { useState, useRef, useCallback } from 'react';
import {
  View,
  StyleSheet,
  TouchableOpacity,
  Text,
  Platform,
} from 'react-native';
import MapView, {
  PROVIDER_GOOGLE,
  PROVIDER_DEFAULT,
  Region,
  MapType,
  Camera,
  LatLng,
} from 'react-native-maps';
import Icon from 'react-native-vector-icons/MaterialIcons';

interface MapComponentProps {
  initialRegion?: Region;
  onRegionChange?: (region: Region) => void;
  onMapPress?: (coordinate: LatLng) => void;
  showsUserLocation?: boolean;
  children?: React.ReactNode;
}

const MapComponent: React.FC<MapComponentProps> = ({
  initialRegion = {
    latitude: 13.7563, // Bangkok
    longitude: 100.5018,
    latitudeDelta: 0.0922,
    longitudeDelta: 0.0421,
  },
  onRegionChange,
  onMapPress,
  showsUserLocation = true,
  children,
}) => {
  const mapRef = useRef<MapView>(null);
  const [mapType, setMapType] = useState<MapType>('standard');

  // ย้ายกล้องไปยังตำแหน่งที่กำหนด
  const animateToRegion = useCallback((region: Region) => {
    mapRef.current?.animateToRegion(region, 1000);
  }, []);

  // ย้ายไปตำแหน่งผู้ใช้
  const goToUserLocation = useCallback(async () => {
    const { location } = useLocation();
    if (location) {
      animateToRegion({
        latitude: location.latitude,
        longitude: location.longitude,
        latitudeDelta: 0.01,
        longitudeDelta: 0.01,
      });
    }
  }, []);

  // Zoom in/out
  const zoom = useCallback(async (inOut: 'in' | 'out') => {
    const camera = await mapRef.current?.getCamera();
    if (camera) {
      const newZoom = inOut === 'in'
        ? Math.min((camera.zoom || 15) + 1, 20)
        : Math.max((camera.zoom || 15) - 1, 1);
      
      mapRef.current?.animateCamera({ ...camera, zoom: newZoom }, { duration: 300 });
    }
  }, []);

  return (
    <View style={styles.container}>
      <MapView
        ref={mapRef}
        style={styles.map}
        provider={Platform.OS === 'android' ? PROVIDER_GOOGLE : PROVIDER_DEFAULT}
        initialRegion={initialRegion}
        mapType={mapType}
        showsUserLocation={showsUserLocation}
        showsMyLocationButton={false}
        showsCompass={true}
        showsScale={true}
        showsTraffic={false}
        showsBuildings={true}
        onRegionChangeComplete={onRegionChange}
        onPress={(event) => onMapPress?.(event.nativeEvent.coordinate)}
        loadingEnabled={true}
        loadingIndicatorColor="#2196F3"
      >
        {children}
      </MapView>

      {/* Map Controls */}
      <View style={styles.controls}>
        <TouchableOpacity style={styles.controlBtn} onPress={() => zoom('in')}>
          <Icon name="add" size={24} color="#333" />
        </TouchableOpacity>
        <TouchableOpacity style={styles.controlBtn} onPress={() => zoom('out')}>
          <Icon name="remove" size={24} color="#333" />
        </TouchableOpacity>
        <TouchableOpacity
          style={styles.controlBtn}
          onPress={() => setMapType(t =>
            t === 'standard' ? 'satellite' : t === 'satellite' ? 'hybrid' : 'standard'
          )}
        >
          <Icon name="layers" size={24} color="#333" />
        </TouchableOpacity>
        <TouchableOpacity style={styles.controlBtn} onPress={goToUserLocation}>
          <Icon name="my-location" size={24} color="#2196F3" />
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  map: { flex: 1 },
  controls: {
    position: 'absolute',
    right: 16,
    bottom: 100,
    gap: 8,
  },
  controlBtn: {
    width: 44,
    height: 44,
    borderRadius: 8,
    backgroundColor: 'white',
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
});

export default MapComponent;
```

---

## 4. Markers, Polylines, Polygons {#markers}

### Custom Markers

```typescript
// src/components/MapMarkers.tsx
import React, { memo } from 'react';
import { View, Text, Image, StyleSheet } from 'react-native';
import {
  Marker,
  Callout,
  Polyline,
  Polygon,
  Circle,
  LatLng,
} from 'react-native-maps';

interface PlaceMarker {
  id: string;
  coordinate: LatLng;
  title: string;
  description: string;
  type: 'restaurant' | 'hotel' | 'shop' | 'park';
  rating?: number;
  imageUrl?: string;
}

// Custom Marker Component
export const CustomMarker = memo(({ place, onPress }: {
  place: PlaceMarker;
  onPress: (place: PlaceMarker) => void;
}) => {
  const getMarkerColor = (type: string) => {
    switch (type) {
      case 'restaurant': return '#FF5722';
      case 'hotel': return '#2196F3';
      case 'shop': return '#4CAF50';
      case 'park': return '#8BC34A';
      default: return '#9E9E9E';
    }
  };

  const getMarkerIcon = (type: string) => {
    switch (type) {
      case 'restaurant': return '🍽️';
      case 'hotel': return '🏨';
      case 'shop': return '🏪';
      case 'park': return '🌳';
      default: return '📍';
    }
  };

  return (
    <Marker
      coordinate={place.coordinate}
      onPress={() => onPress(place)}
      anchor={{ x: 0.5, y: 1 }}
    >
      {/* Custom Marker View */}
      <View style={styles.markerContainer}>
        <View style={[styles.markerBubble, { backgroundColor: getMarkerColor(place.type) }]}>
          <Text style={styles.markerIcon}>{getMarkerIcon(place.type)}</Text>
        </View>
        <View style={[styles.markerArrow, { borderTopColor: getMarkerColor(place.type) }]} />
      </View>

      {/* Callout (popup เมื่อกด marker) */}
      <Callout tooltip>
        <View style={styles.callout}>
          {place.imageUrl && (
            <Image source={{ uri: place.imageUrl }} style={styles.calloutImage} />
          )}
          <Text style={styles.calloutTitle}>{place.title}</Text>
          <Text style={styles.calloutDesc}>{place.description}</Text>
          {place.rating && (
            <View style={styles.ratingContainer}>
              <Text style={styles.ratingText}>⭐ {place.rating.toFixed(1)}</Text>
            </View>
          )}
        </View>
      </Callout>
    </Marker>
  );
});

// Route Polyline
export const RoutePolyline = ({ coordinates, color = '#2196F3', width = 4 }: {
  coordinates: LatLng[];
  color?: string;
  width?: number;
}) => (
  <Polyline
    coordinates={coordinates}
    strokeColor={color}
    strokeWidth={width}
    lineDashPattern={[0]}
    lineJoin="round"
    lineCap="round"
  />
);

// Area Polygon
export const AreaPolygon = ({
  coordinates,
  fillColor = 'rgba(33, 150, 243, 0.2)',
  strokeColor = '#2196F3',
}: {
  coordinates: LatLng[];
  fillColor?: string;
  strokeColor?: string;
}) => (
  <Polygon
    coordinates={coordinates}
    fillColor={fillColor}
    strokeColor={strokeColor}
    strokeWidth={2}
  />
);

// Radius Circle
export const RadiusCircle = ({ center, radius, color = '#2196F3' }: {
  center: LatLng;
  radius: number; // เมตร
  color?: string;
}) => (
  <Circle
    center={center}
    radius={radius}
    fillColor={`${color}20`}
    strokeColor={color}
    strokeWidth={2}
  />
);

const styles = StyleSheet.create({
  markerContainer: {
    alignItems: 'center',
  },
  markerBubble: {
    width: 44,
    height: 44,
    borderRadius: 22,
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.3,
    shadowRadius: 3,
  },
  markerIcon: {
    fontSize: 22,
  },
  markerArrow: {
    width: 0,
    height: 0,
    borderLeftWidth: 8,
    borderRightWidth: 8,
    borderTopWidth: 12,
    borderLeftColor: 'transparent',
    borderRightColor: 'transparent',
    marginTop: -1,
  },
  callout: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 12,
    minWidth: 200,
    maxWidth: 280,
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.3,
    shadowRadius: 4,
  },
  calloutImage: {
    width: '100%',
    height: 120,
    borderRadius: 8,
    marginBottom: 8,
  },
  calloutTitle: {
    fontWeight: 'bold',
    fontSize: 15,
    marginBottom: 4,
  },
  calloutDesc: {
    fontSize: 13,
    color: '#666',
  },
  ratingContainer: {
    marginTop: 8,
  },
  ratingText: {
    fontSize: 13,
    fontWeight: 'bold',
  },
});
```

---

## 5. Directions API {#directions}

### การใช้งาน Google Directions API

```typescript
// src/services/DirectionsService.ts

interface DirectionResult {
  distance: string;
  duration: string;
  steps: DirectionStep[];
  polylinePoints: LatLng[];
  overview_polyline: string;
}

interface DirectionStep {
  instruction: string;
  distance: string;
  duration: string;
  startLocation: LatLng;
  endLocation: LatLng;
  maneuver: string;
}

class DirectionsService {
  private static API_KEY = 'YOUR_GOOGLE_MAPS_API_KEY';
  private static BASE_URL = 'https://maps.googleapis.com/maps/api/directions/json';

  static async getDirections(
    origin: LatLng,
    destination: LatLng,
    mode: 'driving' | 'walking' | 'bicycling' | 'transit' = 'driving',
    waypoints?: LatLng[]
  ): Promise<DirectionResult | null> {
    try {
      const params = new URLSearchParams({
        origin: `${origin.latitude},${origin.longitude}`,
        destination: `${destination.latitude},${destination.longitude}`,
        mode,
        key: this.API_KEY,
        language: 'th',
        units: 'metric',
      });

      if (waypoints && waypoints.length > 0) {
        const waypointStr = waypoints
          .map(wp => `${wp.latitude},${wp.longitude}`)
          .join('|');
        params.append('waypoints', `optimize:true|${waypointStr}`);
      }

      const response = await fetch(`${this.BASE_URL}?${params}`);
      const data = await response.json();

      if (data.status !== 'OK' || !data.routes[0]) {
        console.error('Directions error:', data.status);
        return null;
      }

      const route = data.routes[0];
      const leg = route.legs[0];

      return {
        distance: leg.distance.text,
        duration: leg.duration.text,
        steps: leg.steps.map((step: any) => ({
          instruction: step.html_instructions.replace(/<[^>]*>/g, ''),
          distance: step.distance.text,
          duration: step.duration.text,
          startLocation: step.start_location,
          endLocation: step.end_location,
          maneuver: step.maneuver || '',
        })),
        polylinePoints: this.decodePolyline(route.overview_polyline.points),
        overview_polyline: route.overview_polyline.points,
      };
    } catch (error) {
      console.error('Directions fetch error:', error);
      return null;
    }
  }

  // Decode Google Polyline
  private static decodePolyline(encoded: string): LatLng[] {
    const points: LatLng[] = [];
    let index = 0;
    let lat = 0;
    let lng = 0;

    while (index < encoded.length) {
      let b: number;
      let shift = 0;
      let result = 0;
      
      do {
        b = encoded.charCodeAt(index++) - 63;
        result |= (b & 0x1f) << shift;
        shift += 5;
      } while (b >= 0x20);
      
      const dlat = result & 1 ? ~(result >> 1) : result >> 1;
      lat += dlat;

      shift = 0;
      result = 0;
      
      do {
        b = encoded.charCodeAt(index++) - 63;
        result |= (b & 0x1f) << shift;
        shift += 5;
      } while (b >= 0x20);
      
      const dlng = result & 1 ? ~(result >> 1) : result >> 1;
      lng += dlng;

      points.push({
        latitude: lat / 1e5,
        longitude: lng / 1e5,
      });
    }

    return points;
  }

  // คำนวณระยะทางระหว่างสองจุด (Haversine Formula)
  static calculateDistance(point1: LatLng, point2: LatLng): number {
    const R = 6371000; // รัศมีโลกในเมตร
    const lat1 = point1.latitude * Math.PI / 180;
    const lat2 = point2.latitude * Math.PI / 180;
    const dLat = (point2.latitude - point1.latitude) * Math.PI / 180;
    const dLng = (point2.longitude - point1.longitude) * Math.PI / 180;

    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1) * Math.cos(lat2) *
              Math.sin(dLng/2) * Math.sin(dLng/2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return R * c; // ระยะทางในเมตร
  }
}

export default DirectionsService;
```

---

## 6. Geocoding {#geocoding}

### การแปลง LatLng เป็นที่อยู่ และ ที่อยู่เป็น LatLng

```typescript
// src/services/GeocodingService.ts

interface Address {
  formatted: string;
  street?: string;
  district?: string;
  city?: string;
  province?: string;
  country?: string;
  postalCode?: string;
}

class GeocodingService {
  private static API_KEY = 'YOUR_GOOGLE_MAPS_API_KEY';

  // LatLng -> Address (Reverse Geocoding)
  static async reverseGeocode(coordinate: LatLng): Promise<Address | null> {
    try {
      const response = await fetch(
        `https://maps.googleapis.com/maps/api/geocode/json?latlng=${coordinate.latitude},${coordinate.longitude}&language=th&key=${this.API_KEY}`
      );
      const data = await response.json();

      if (data.status !== 'OK' || !data.results[0]) return null;

      const result = data.results[0];
      const components = result.address_components;

      return {
        formatted: result.formatted_address,
        street: this.getComponent(components, 'route'),
        district: this.getComponent(components, 'sublocality'),
        city: this.getComponent(components, 'locality'),
        province: this.getComponent(components, 'administrative_area_level_1'),
        country: this.getComponent(components, 'country'),
        postalCode: this.getComponent(components, 'postal_code'),
      };
    } catch (error) {
      console.error('Reverse geocode error:', error);
      return null;
    }
  }

  // Address -> LatLng (Forward Geocoding)
  static async geocode(address: string): Promise<LatLng | null> {
    try {
      const response = await fetch(
        `https://maps.googleapis.com/maps/api/geocode/json?address=${encodeURIComponent(address)}&language=th&key=${this.API_KEY}`
      );
      const data = await response.json();

      if (data.status !== 'OK' || !data.results[0]) return null;

      return data.results[0].geometry.location;
    } catch (error) {
      console.error('Geocode error:', error);
      return null;
    }
  }

  // ค้นหาสถานที่
  static async searchPlaces(query: string, location?: LatLng, radius?: number): Promise<any[]> {
    try {
      const params = new URLSearchParams({
        query,
        key: this.API_KEY,
        language: 'th',
      });

      if (location) {
        params.append('location', `${location.latitude},${location.longitude}`);
        params.append('radius', String(radius || 5000));
      }

      const response = await fetch(
        `https://maps.googleapis.com/maps/api/place/textsearch/json?${params}`
      );
      const data = await response.json();

      return data.results || [];
    } catch (error) {
      console.error('Search places error:', error);
      return [];
    }
  }

  private static getComponent(components: any[], type: string): string | undefined {
    const component = components.find(c => c.types.includes(type));
    return component?.long_name;
  }
}

export default GeocodingService;
```

---

## 7. Workshop: Location Tracking App {#workshop}

```typescript
// src/screens/LocationTrackingScreen.tsx
import React, { useState, useEffect, useRef, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Alert,
  FlatList,
} from 'react-native';
import MapView, {
  Marker,
  Polyline,
  LatLng,
  Region,
  PROVIDER_GOOGLE,
} from 'react-native-maps';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { useLocation } from '../hooks/useLocation';
import DirectionsService from '../services/DirectionsService';
import GeocodingService from '../services/GeocodingService';

interface TrackingPoint {
  coordinate: LatLng;
  timestamp: number;
  speed?: number;
  accuracy?: number;
}

interface Trip {
  id: string;
  startTime: number;
  endTime?: number;
  points: TrackingPoint[];
  totalDistance: number;
  startAddress?: string;
  endAddress?: string;
}

const LocationTrackingScreen: React.FC = () => {
  const mapRef = useRef<MapView>(null);
  const [isTracking, setIsTracking] = useState(false);
  const [currentTrip, setCurrentTrip] = useState<Trip | null>(null);
  const [trips, setTrips] = useState<Trip[]>([]);
  const [selectedTrip, setSelectedTrip] = useState<Trip | null>(null);
  const [currentAddress, setCurrentAddress] = useState<string>('');

  const { location, startWatching, stopWatching } = useLocation({
    watchMode: true,
    enableHighAccuracy: true,
    distanceFilter: 5,
  });

  // อัพเดต address เมื่อตำแหน่งเปลี่ยน
  useEffect(() => {
    if (location) {
      updateCurrentAddress(location);
      if (isTracking) {
        addTrackingPoint(location);
      }
    }
  }, [location, isTracking]);

  const updateCurrentAddress = async (loc: { latitude: number; longitude: number }) => {
    const address = await GeocodingService.reverseGeocode({
      latitude: loc.latitude,
      longitude: loc.longitude,
    });
    if (address) {
      setCurrentAddress(address.formatted);
    }
  };

  const addTrackingPoint = useCallback((loc: typeof location) => {
    if (!loc || !currentTrip) return;

    const newPoint: TrackingPoint = {
      coordinate: { latitude: loc.latitude, longitude: loc.longitude },
      timestamp: loc.timestamp,
      speed: loc.speed,
      accuracy: loc.accuracy,
    };

    setCurrentTrip(prev => {
      if (!prev) return prev;

      // คำนวณระยะทางเพิ่ม
      let additionalDistance = 0;
      if (prev.points.length > 0) {
        const lastPoint = prev.points[prev.points.length - 1];
        additionalDistance = DirectionsService.calculateDistance(
          lastPoint.coordinate,
          newPoint.coordinate
        );
      }

      return {
        ...prev,
        points: [...prev.points, newPoint],
        totalDistance: prev.totalDistance + additionalDistance,
      };
    });
  }, [currentTrip]);

  const startTracking = async () => {
    if (!location) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถระบุตำแหน่งได้');
      return;
    }

    const address = await GeocodingService.reverseGeocode({
      latitude: location.latitude,
      longitude: location.longitude,
    });

    const newTrip: Trip = {
      id: `trip_${Date.now()}`,
      startTime: Date.now(),
      points: [],
      totalDistance: 0,
      startAddress: address?.formatted,
    };

    setCurrentTrip(newTrip);
    setIsTracking(true);
    startWatching();
  };

  const stopTracking = async () => {
    if (!currentTrip || !location) return;

    const address = await GeocodingService.reverseGeocode({
      latitude: location.latitude,
      longitude: location.longitude,
    });

    const finishedTrip: Trip = {
      ...currentTrip,
      endTime: Date.now(),
      endAddress: address?.formatted,
    };

    setTrips(prev => [finishedTrip, ...prev]);
    setCurrentTrip(null);
    setIsTracking(false);
    stopWatching();
    setSelectedTrip(finishedTrip);
  };

  const formatDistance = (meters: number): string => {
    if (meters < 1000) return `${Math.round(meters)} ม.`;
    return `${(meters / 1000).toFixed(1)} กม.`;
  };

  const formatDuration = (startTime: number, endTime?: number): string => {
    const duration = ((endTime || Date.now()) - startTime) / 1000;
    const minutes = Math.floor(duration / 60);
    const seconds = Math.floor(duration % 60);
    if (minutes < 60) return `${minutes}:${seconds.toString().padStart(2, '0')} นาที`;
    const hours = Math.floor(minutes / 60);
    return `${hours} ชม. ${minutes % 60} นาที`;
  };

  const getMapRegion = (trip: Trip): Region | undefined => {
    if (trip.points.length === 0) return undefined;

    const lats = trip.points.map(p => p.coordinate.latitude);
    const lngs = trip.points.map(p => p.coordinate.longitude);
    const minLat = Math.min(...lats);
    const maxLat = Math.max(...lats);
    const minLng = Math.min(...lngs);
    const maxLng = Math.max(...lngs);

    return {
      latitude: (minLat + maxLat) / 2,
      longitude: (minLng + maxLng) / 2,
      latitudeDelta: (maxLat - minLat) * 1.5 + 0.01,
      longitudeDelta: (maxLng - minLng) * 1.5 + 0.01,
    };
  };

  const displayTrip = selectedTrip || currentTrip;

  return (
    <View style={styles.container}>
      {/* Map */}
      <MapView
        ref={mapRef}
        style={styles.map}
        provider={PROVIDER_GOOGLE}
        showsUserLocation={true}
        followsUserLocation={isTracking}
        region={displayTrip ? getMapRegion(displayTrip) : undefined}
      >
        {displayTrip && displayTrip.points.length > 1 && (
          <>
            <Polyline
              coordinates={displayTrip.points.map(p => p.coordinate)}
              strokeColor={isTracking ? '#2196F3' : '#4CAF50'}
              strokeWidth={4}
            />
            <Marker
              coordinate={displayTrip.points[0].coordinate}
              title="จุดเริ่มต้น"
              pinColor="green"
            />
            {!isTracking && (
              <Marker
                coordinate={displayTrip.points[displayTrip.points.length - 1].coordinate}
                title="จุดสิ้นสุด"
                pinColor="red"
              />
            )}
          </>
        )}
      </MapView>

      {/* Current Location Info */}
      {location && (
        <View style={styles.locationInfo}>
          <Icon name="location-on" size={16} color="#666" />
          <Text style={styles.locationText} numberOfLines={1}>{currentAddress}</Text>
        </View>
      )}

      {/* Tracking Controls */}
      <View style={styles.trackingPanel}>
        {isTracking && currentTrip ? (
          <View style={styles.trackingStats}>
            <View style={styles.statItem}>
              <Text style={styles.statValue}>
                {formatDistance(currentTrip.totalDistance)}
              </Text>
              <Text style={styles.statLabel}>ระยะทาง</Text>
            </View>
            <View style={styles.statItem}>
              <Text style={styles.statValue}>
                {formatDuration(currentTrip.startTime)}
              </Text>
              <Text style={styles.statLabel}>เวลา</Text>
            </View>
            <View style={styles.statItem}>
              <Text style={styles.statValue}>
                {location?.speed
                  ? `${((location.speed * 3.6)).toFixed(1)} กม./ชม.`
                  : '0 กม./ชม.'}
              </Text>
              <Text style={styles.statLabel}>ความเร็ว</Text>
            </View>
            <TouchableOpacity style={styles.stopButton} onPress={stopTracking}>
              <Icon name="stop" size={28} color="white" />
            </TouchableOpacity>
          </View>
        ) : (
          <TouchableOpacity style={styles.startButton} onPress={startTracking}>
            <Icon name="play-arrow" size={28} color="white" />
            <Text style={styles.startButtonText}>เริ่มติดตาม</Text>
          </TouchableOpacity>
        )}
      </View>

      {/* Trip History */}
      {!isTracking && trips.length > 0 && (
        <View style={styles.tripsPanel}>
          <Text style={styles.tripsPanelTitle}>ประวัติเส้นทาง</Text>
          <FlatList
            data={trips}
            horizontal
            renderItem={({ item }) => (
              <TouchableOpacity
                style={[
                  styles.tripCard,
                  selectedTrip?.id === item.id && styles.selectedTripCard,
                ]}
                onPress={() => setSelectedTrip(item)}
              >
                <Text style={styles.tripDate}>
                  {new Date(item.startTime).toLocaleDateString('th-TH')}
                </Text>
                <Text style={styles.tripDistance}>
                  {formatDistance(item.totalDistance)}
                </Text>
                <Text style={styles.tripDuration}>
                  {formatDuration(item.startTime, item.endTime)}
                </Text>
              </TouchableOpacity>
            )}
            keyExtractor={item => item.id}
            showsHorizontalScrollIndicator={false}
          />
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  map: { flex: 1 },
  locationInfo: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 8,
    paddingHorizontal: 16,
    gap: 4,
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
  },
  locationText: { flex: 1, fontSize: 12, color: '#666' },
  trackingPanel: {
    backgroundColor: 'white',
    padding: 16,
    elevation: 8,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  trackingStats: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
  },
  statItem: { alignItems: 'center', flex: 1 },
  statValue: { fontSize: 18, fontWeight: 'bold', color: '#2196F3' },
  statLabel: { fontSize: 11, color: '#666', marginTop: 2 },
  startButton: {
    flexDirection: 'row',
    backgroundColor: '#4CAF50',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    justifyContent: 'center',
    gap: 8,
  },
  startButtonText: { color: 'white', fontSize: 16, fontWeight: 'bold' },
  stopButton: {
    width: 52,
    height: 52,
    borderRadius: 26,
    backgroundColor: '#F44336',
    justifyContent: 'center',
    alignItems: 'center',
  },
  tripsPanel: {
    backgroundColor: 'white',
    padding: 12,
    borderTopWidth: 1,
    borderTopColor: '#E0E0E0',
  },
  tripsPanelTitle: { fontWeight: 'bold', marginBottom: 8 },
  tripCard: {
    backgroundColor: '#F5F5F5',
    padding: 12,
    borderRadius: 8,
    marginRight: 8,
    minWidth: 120,
    alignItems: 'center',
  },
  selectedTripCard: { backgroundColor: '#E3F2FD', borderWidth: 2, borderColor: '#2196F3' },
  tripDate: { fontSize: 11, color: '#666', marginBottom: 4 },
  tripDistance: { fontSize: 16, fontWeight: 'bold', color: '#333' },
  tripDuration: { fontSize: 11, color: '#666', marginTop: 4 },
});

export default LocationTrackingScreen;
```

---

## Tips และ Best Practices

### 1. Battery Optimization

```typescript
// ปรับ accuracy ตามความต้องการ
const batteryEfficientOptions = {
  enableHighAccuracy: false, // ใช้ Network แทน GPS
  distanceFilter: 50, // อัพเดตทุก 50 เมตร
  interval: 30000, // อัพเดตทุก 30 วินาที
};

const highAccuracyOptions = {
  enableHighAccuracy: true, // ใช้ GPS
  distanceFilter: 5,
  interval: 5000,
};
```

### 2. Geofencing

```typescript
// สร้าง Geofence อย่างง่าย
function isInsideGeofence(
  userLocation: LatLng,
  center: LatLng,
  radiusMeters: number
): boolean {
  const distance = DirectionsService.calculateDistance(userLocation, center);
  return distance <= radiusMeters;
}
```

### สรุป

- ใช้ react-native-geolocation-service แทน Geolocation API พื้นฐาน
- ใช้ distanceFilter เพื่อประหยัด battery
- ตรวจสอบ permission ก่อนใช้งาน location เสมอ
- แสดง map type ให้ผู้ใช้เลือกได้
- Cache ผลลัพธ์ geocoding เพื่อลด API calls
- คำนวณ route ฝั่ง client ได้ด้วย Haversine formula
