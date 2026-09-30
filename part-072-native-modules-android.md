# Part 072: Native Modules - Android

## Native Modules สำหรับ Android

Android Native Modules ใช้ Java หรือ Kotlin ในการสร้าง bridge ระหว่าง JavaScript และ Android platform ช่วยให้เราเข้าถึง Android API, ใช้ native libraries, หรือทำงานที่ต้องการ performance สูง

---

## 1. การสร้าง Native Module ด้วย Java

### 1.1 สร้าง Module Class

```java
// CalendarModule.java
package com.myapp;

import android.Manifest;
import android.content.ContentValues;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.provider.CalendarContract;

import androidx.annotation.NonNull;
import androidx.core.content.ContextCompat;

import com.facebook.react.bridge.Arguments;
import com.facebook.react.bridge.Promise;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.bridge.WritableArray;
import com.facebook.react.bridge.WritableMap;
import com.facebook.react.module.annotations.ReactModule;

import java.util.TimeZone;

@ReactModule(name = CalendarModule.NAME)
public class CalendarModule extends ReactContextBaseJavaModule {
  
  public static final String NAME = "CalendarModule";
  
  public CalendarModule(ReactApplicationContext reactContext) {
    super(reactContext);
  }
  
  @NonNull
  @Override
  public String getName() {
    return NAME;
  }
  
  // Export method ให้ JavaScript เรียกได้
  @ReactMethod
  public void createCalendarEvent(
    String title,
    String location,
    double startTime,
    Promise promise
  ) {
    // ตรวจสอบ permission
    if (ContextCompat.checkSelfPermission(
      getReactApplicationContext(),
      Manifest.permission.WRITE_CALENDAR
    ) != PackageManager.PERMISSION_GRANTED) {
      promise.reject("PERMISSION_DENIED", "Calendar write permission not granted");
      return;
    }
    
    try {
      ContentValues values = new ContentValues();
      values.put(CalendarContract.Events.TITLE, title);
      values.put(CalendarContract.Events.EVENT_LOCATION, location);
      values.put(CalendarContract.Events.DTSTART, (long) startTime);
      values.put(CalendarContract.Events.DTEND, (long) startTime + 3600000); // +1 hour
      values.put(CalendarContract.Events.EVENT_TIMEZONE, TimeZone.getDefault().getID());
      values.put(CalendarContract.Events.CALENDAR_ID, 1);
      
      Uri uri = getReactApplicationContext()
        .getContentResolver()
        .insert(CalendarContract.Events.CONTENT_URI, values);
      
      if (uri != null) {
        String eventId = uri.getLastPathSegment();
        promise.resolve(eventId);
      } else {
        promise.reject("SAVE_FAILED", "Failed to create calendar event");
      }
    } catch (Exception e) {
      promise.reject("ERROR", e.getMessage(), e);
    }
  }
  
  @ReactMethod
  public void getCalendars(Promise promise) {
    WritableArray calendars = Arguments.createArray();
    
    // Query for available calendars
    String[] projection = {
      CalendarContract.Calendars._ID,
      CalendarContract.Calendars.NAME,
      CalendarContract.Calendars.CALENDAR_DISPLAY_NAME
    };
    
    try (android.database.Cursor cursor = getReactApplicationContext()
      .getContentResolver()
      .query(
        CalendarContract.Calendars.CONTENT_URI,
        projection,
        null, null, null
      )) {
      
      if (cursor != null) {
        while (cursor.moveToNext()) {
          WritableMap calendar = Arguments.createMap();
          calendar.putString("id", cursor.getString(0));
          calendar.putString("name", cursor.getString(1));
          calendar.putString("displayName", cursor.getString(2));
          calendars.pushMap(calendar);
        }
      }
      
      promise.resolve(calendars);
    } catch (Exception e) {
      promise.reject("ERROR", e.getMessage(), e);
    }
  }
  
  // Constants ที่ส่งให้ JavaScript
  @Override
  public java.util.Map<String, Object> getConstants() {
    final java.util.Map<String, Object> constants = new java.util.HashMap<>();
    constants.put("PERMISSION_WRITE", Manifest.permission.WRITE_CALENDAR);
    constants.put("PERMISSION_READ", Manifest.permission.READ_CALENDAR);
    return constants;
  }
}
```

### 1.2 สร้าง Package

```java
// CalendarPackage.java
package com.myapp;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class CalendarPackage implements ReactPackage {
  
  @Override
  public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();
    modules.add(new CalendarModule(reactContext));
    return modules;
  }
  
  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }
}
```

### 1.3 ลงทะเบียนใน MainApplication

```java
// MainApplication.java
import com.myapp.CalendarPackage;

public class MainApplication extends Application implements ReactApplication {
  
  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    @Override
    protected List<ReactPackage> getPackages() {
      List<ReactPackage> packages = new PackageList(this).getPackages();
      packages.add(new CalendarPackage()); // เพิ่ม Package ที่นี่
      return packages;
    }
    
    // ... other config
  };
}
```

---

## 2. การสร้าง Native Module ด้วย Kotlin

### 2.1 Kotlin Module

```kotlin
// BiometricModule.kt
package com.myapp

import android.os.Build
import android.util.Log
import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity
import com.facebook.react.bridge.*
import com.facebook.react.module.annotations.ReactModule
import java.util.concurrent.Executor

@ReactModule(name = BiometricModule.NAME)
class BiometricModule(private val reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext) {
  
  companion object {
    const val NAME = "BiometricModule"
  }
  
  override fun getName() = NAME
  
  @ReactMethod
  fun checkBiometricAvailability(promise: Promise) {
    val biometricManager = BiometricManager.from(reactContext)
    
    val result = Arguments.createMap()
    
    when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_WEAK)) {
      BiometricManager.BIOMETRIC_SUCCESS -> {
        result.putBoolean("available", true)
        result.putString("type", "biometric")
        promise.resolve(result)
      }
      BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> {
        result.putBoolean("available", false)
        result.putString("errorCode", "NO_HARDWARE")
        promise.resolve(result)
      }
      BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> {
        result.putBoolean("available", false)
        result.putString("errorCode", "HARDWARE_UNAVAILABLE")
        promise.resolve(result)
      }
      BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> {
        result.putBoolean("available", false)
        result.putString("errorCode", "NOT_ENROLLED")
        promise.resolve(result)
      }
      else -> {
        result.putBoolean("available", false)
        result.putString("errorCode", "UNKNOWN")
        promise.resolve(result)
      }
    }
  }
  
  @ReactMethod
  fun authenticate(
    title: String,
    subtitle: String,
    description: String,
    cancelLabel: String,
    promise: Promise
  ) {
    val activity = currentActivity as? FragmentActivity
    if (activity == null) {
      promise.reject("ACTIVITY_NOT_FOUND", "Fragment activity not found")
      return
    }
    
    val executor: Executor = ContextCompat.getMainExecutor(reactContext)
    
    val callback = object : BiometricPrompt.AuthenticationCallback() {
      override fun onAuthenticationSucceeded(
        result: BiometricPrompt.AuthenticationResult
      ) {
        val response = Arguments.createMap()
        response.putBoolean("success", true)
        promise.resolve(response)
      }
      
      override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
        val response = Arguments.createMap()
        response.putBoolean("success", false)
        response.putString("errorCode", getErrorCode(errorCode))
        response.putString("errorMessage", errString.toString())
        promise.resolve(response)
      }
      
      override fun onAuthenticationFailed() {
        Log.d(NAME, "Authentication failed")
      }
    }
    
    activity.runOnUiThread {
      val biometricPrompt = BiometricPrompt(activity, executor, callback)
      
      val promptInfo = BiometricPrompt.PromptInfo.Builder()
        .setTitle(title)
        .setSubtitle(subtitle)
        .setDescription(description)
        .setNegativeButtonText(cancelLabel)
        .build()
      
      biometricPrompt.authenticate(promptInfo)
    }
  }
  
  private fun getErrorCode(errorCode: Int): String {
    return when (errorCode) {
      BiometricPrompt.ERROR_USER_CANCELED -> "USER_CANCEL"
      BiometricPrompt.ERROR_NEGATIVE_BUTTON -> "USER_FALLBACK"
      BiometricPrompt.ERROR_LOCKOUT -> "LOCKOUT"
      BiometricPrompt.ERROR_LOCKOUT_PERMANENT -> "LOCKOUT_PERMANENT"
      BiometricPrompt.ERROR_HW_UNAVAILABLE -> "HW_UNAVAILABLE"
      else -> "UNKNOWN"
    }
  }
}
```

### 2.2 Kotlin Package

```kotlin
// BiometricPackage.kt
package com.myapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager

class BiometricPackage : ReactPackage {
  
  override fun createNativeModules(
    reactContext: ReactApplicationContext
  ): List<NativeModule> {
    return listOf(BiometricModule(reactContext))
  }
  
  override fun createViewManagers(
    reactContext: ReactApplicationContext
  ): List<ViewManager<*, *>> {
    return emptyList()
  }
}
```

---

## 3. Events จาก Native ไปยัง JavaScript

### 3.1 Java Event Emitter

```java
// NetworkModule.java
package com.myapp;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.net.ConnectivityManager;
import android.net.NetworkInfo;

import com.facebook.react.bridge.Arguments;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.bridge.WritableMap;
import com.facebook.react.modules.core.DeviceEventManagerModule;

import javax.annotation.Nullable;

public class NetworkModule extends ReactContextBaseJavaModule {
  
  public static final String NAME = "NetworkModule";
  private static final String EVENT_NETWORK_CHANGE = "onNetworkChange";
  
  private BroadcastReceiver networkReceiver;
  private int listenerCount = 0;
  
  public NetworkModule(ReactApplicationContext reactContext) {
    super(reactContext);
  }
  
  @Override
  public String getName() {
    return NAME;
  }
  
  // เพิ่ม listener count
  @ReactMethod
  public void addListener(String eventName) {
    listenerCount++;
    if (listenerCount == 1) {
      startNetworkMonitoring();
    }
  }
  
  // ลด listener count
  @ReactMethod
  public void removeListeners(Integer count) {
    listenerCount -= count;
    if (listenerCount == 0) {
      stopNetworkMonitoring();
    }
  }
  
  private void startNetworkMonitoring() {
    networkReceiver = new BroadcastReceiver() {
      @Override
      public void onReceive(Context context, Intent intent) {
        sendNetworkStatus();
      }
    };
    
    IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
    getReactApplicationContext().registerReceiver(networkReceiver, filter);
  }
  
  private void stopNetworkMonitoring() {
    if (networkReceiver != null) {
      try {
        getReactApplicationContext().unregisterReceiver(networkReceiver);
      } catch (Exception ignored) {}
      networkReceiver = null;
    }
  }
  
  private void sendNetworkStatus() {
    ConnectivityManager cm = (ConnectivityManager) getReactApplicationContext()
      .getSystemService(Context.CONNECTIVITY_SERVICE);
    
    NetworkInfo networkInfo = cm.getActiveNetworkInfo();
    
    WritableMap params = Arguments.createMap();
    params.putBoolean("isConnected", networkInfo != null && networkInfo.isConnected());
    
    if (networkInfo != null) {
      params.putString("type", getNetworkType(networkInfo.getType()));
    } else {
      params.putString("type", "none");
    }
    
    sendEvent(EVENT_NETWORK_CHANGE, params);
  }
  
  private void sendEvent(String eventName, @Nullable WritableMap params) {
    getReactApplicationContext()
      .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
      .emit(eventName, params);
  }
  
  private String getNetworkType(int type) {
    switch (type) {
      case ConnectivityManager.TYPE_WIFI:
        return "wifi";
      case ConnectivityManager.TYPE_MOBILE:
        return "cellular";
      case ConnectivityManager.TYPE_ETHERNET:
        return "ethernet";
      default:
        return "unknown";
    }
  }
}
```

### 3.2 Kotlin Event Emitter

```kotlin
// SensorModule.kt
package com.myapp

import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.content.Context
import com.facebook.react.bridge.*
import com.facebook.react.modules.core.DeviceEventManagerModule

class SensorModule(private val reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext), SensorEventListener {
  
  companion object {
    const val NAME = "SensorModule"
    const val EVENT_ACCELEROMETER = "onAccelerometerUpdate"
    const val EVENT_GYROSCOPE = "onGyroscopeUpdate"
  }
  
  private var sensorManager: SensorManager? = null
  private var listenerCount = 0
  
  override fun getName() = NAME
  
  @ReactMethod
  fun addListener(eventName: String) {
    listenerCount++
    if (listenerCount == 1) {
      initializeSensors()
    }
  }
  
  @ReactMethod
  fun removeListeners(count: Int) {
    listenerCount -= count
    if (listenerCount == 0) {
      stopSensors()
    }
  }
  
  @ReactMethod
  fun startAccelerometer(promise: Promise) {
    sensorManager = reactContext.getSystemService(Context.SENSOR_SERVICE) as SensorManager
    val sensor = sensorManager?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
    
    if (sensor != null) {
      sensorManager?.registerListener(
        this,
        sensor,
        SensorManager.SENSOR_DELAY_NORMAL
      )
      promise.resolve("Accelerometer started")
    } else {
      promise.reject("NO_SENSOR", "Accelerometer not available")
    }
  }
  
  @ReactMethod
  fun stopAccelerometer() {
    sensorManager?.unregisterListener(this)
  }
  
  override fun onSensorChanged(event: SensorEvent?) {
    event ?: return
    
    val data = Arguments.createMap().apply {
      putDouble("x", event.values[0].toDouble())
      putDouble("y", event.values[1].toDouble())
      putDouble("z", event.values[2].toDouble())
      putDouble("timestamp", System.currentTimeMillis().toDouble())
    }
    
    val eventName = when (event.sensor.type) {
      Sensor.TYPE_ACCELEROMETER -> EVENT_ACCELEROMETER
      Sensor.TYPE_GYROSCOPE -> EVENT_GYROSCOPE
      else -> return
    }
    
    reactContext
      .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
      .emit(eventName, data)
  }
  
  override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
  
  private fun initializeSensors() {
    // Initialize sensor listeners
  }
  
  private fun stopSensors() {
    sensorManager?.unregisterListener(this)
  }
  
  override fun onCatalystInstanceDestroy() {
    super.onCatalystInstanceDestroy()
    stopSensors()
  }
}
```

---

## 4. การจัดการ Permissions

```kotlin
// PermissionModule.kt
package com.myapp

import android.Manifest
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import com.facebook.react.bridge.*
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.modules.core.PermissionAwareActivity
import com.facebook.react.modules.core.PermissionListener

@ReactModule(name = PermissionModule.NAME)
class PermissionModule(reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext), PermissionListener {
  
  companion object {
    const val NAME = "PermissionModule"
    const val PERMISSION_GRANTED = "granted"
    const val PERMISSION_DENIED = "denied"
    const val PERMISSION_NEVER_ASK_AGAIN = "never_ask_again"
  }
  
  private var permissionPromises = mutableMapOf<Int, Promise>()
  private var requestCode = 0
  
  override fun getName() = NAME
  
  @ReactMethod
  fun checkPermission(permission: String, promise: Promise) {
    val result = ContextCompat.checkSelfPermission(
      reactApplicationContext,
      permission
    )
    
    promise.resolve(
      if (result == PackageManager.PERMISSION_GRANTED) PERMISSION_GRANTED
      else PERMISSION_DENIED
    )
  }
  
  @ReactMethod
  fun requestPermission(permission: String, promise: Promise) {
    val activity = currentActivity as? PermissionAwareActivity
    
    if (activity == null) {
      promise.reject("ACTIVITY_NOT_FOUND", "Activity not found")
      return
    }
    
    val currentStatus = ContextCompat.checkSelfPermission(
      reactApplicationContext,
      permission
    )
    
    if (currentStatus == PackageManager.PERMISSION_GRANTED) {
      promise.resolve(PERMISSION_GRANTED)
      return
    }
    
    val code = requestCode++
    permissionPromises[code] = promise
    activity.requestPermissions(arrayOf(permission), code, this)
  }
  
  @ReactMethod
  fun requestMultiplePermissions(permissions: ReadableArray, promise: Promise) {
    val permList = (0 until permissions.size()).map { permissions.getString(it) }
    val activity = currentActivity as? PermissionAwareActivity
    
    if (activity == null) {
      promise.reject("ACTIVITY_NOT_FOUND", "Activity not found")
      return
    }
    
    val notGranted = permList.filter { perm ->
      ContextCompat.checkSelfPermission(
        reactApplicationContext, perm
      ) != PackageManager.PERMISSION_GRANTED
    }
    
    if (notGranted.isEmpty()) {
      val results = Arguments.createMap()
      permList.forEach { results.putString(it, PERMISSION_GRANTED) }
      promise.resolve(results)
      return
    }
    
    val code = requestCode++
    permissionPromises[code] = promise
    activity.requestPermissions(notGranted.toTypedArray(), code, this)
  }
  
  override fun onRequestPermissionsResult(
    requestCode: Int,
    permissions: Array<String>,
    grantResults: IntArray
  ): Boolean {
    val promise = permissionPromises.remove(requestCode) ?: return false
    
    val results = Arguments.createMap()
    permissions.forEachIndexed { index, permission ->
      results.putString(
        permission,
        if (grantResults[index] == PackageManager.PERMISSION_GRANTED)
          PERMISSION_GRANTED
        else
          PERMISSION_DENIED
      )
    }
    
    promise.resolve(results)
    return true
  }
}
```

---

## 5. การใช้งานใน JavaScript/TypeScript

```typescript
// NativeModuleService.ts
import { NativeModules, NativeEventEmitter, Platform } from 'react-native';
import { useEffect, useState } from 'react';

const {
  CalendarModule,
  BiometricModule,
  NetworkModule,
  SensorModule
} = NativeModules;

// Calendar Service
export const CalendarService = {
  async createEvent(title: string, location: string, startTime: number): Promise<string> {
    if (Platform.OS !== 'android') throw new Error('Android only');
    return CalendarModule.createCalendarEvent(title, location, startTime);
  },
  
  async getCalendars() {
    return CalendarModule.getCalendars();
  }
};

// Biometric Service
export const BiometricService = {
  async checkAvailability() {
    return BiometricModule.checkBiometricAvailability();
  },
  
  async authenticate(title = 'ยืนยันตัวตน', subtitle = '', description = '') {
    return BiometricModule.authenticate(
      title,
      subtitle,
      description,
      'ยกเลิก'
    );
  }
};

// Network Monitor Hook
export function useNetworkStatus() {
  const [isConnected, setIsConnected] = useState(true);
  const [networkType, setNetworkType] = useState('unknown');
  
  useEffect(() => {
    if (Platform.OS !== 'android') return;
    
    const emitter = new NativeEventEmitter();
    
    NetworkModule.addListener('onNetworkChange');
    
    const subscription = emitter.addListener('onNetworkChange', (event: any) => {
      setIsConnected(event.isConnected);
      setNetworkType(event.type);
    });
    
    return () => {
      subscription.remove();
      NetworkModule.removeListeners(1);
    };
  }, []);
  
  return { isConnected, networkType };
}

// Accelerometer Hook
export function useAccelerometer() {
  const [data, setData] = useState({ x: 0, y: 0, z: 0 });
  
  useEffect(() => {
    if (Platform.OS !== 'android') return;
    
    const emitter = new NativeEventEmitter();
    
    SensorModule.startAccelerometer()
      .catch(console.error);
    
    SensorModule.addListener('onAccelerometerUpdate');
    
    const subscription = emitter.addListener(
      'onAccelerometerUpdate',
      (event: any) => {
        setData({ x: event.x, y: event.y, z: event.z });
      }
    );
    
    return () => {
      subscription.remove();
      SensorModule.stopAccelerometer();
      SensorModule.removeListeners(1);
    };
  }, []);
  
  return data;
}
```

---

## Workshop: สร้าง Android Notification Module

### Step 1: Native Implementation

```kotlin
// NotificationModule.kt
package com.myapp

import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.os.Build
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import com.facebook.react.bridge.*
import com.facebook.react.module.annotations.ReactModule

@ReactModule(name = NotificationModule.NAME)
class NotificationModule(private val reactContext: ReactApplicationContext) :
  ReactContextBaseJavaModule(reactContext) {
  
  companion object {
    const val NAME = "NotificationModule"
    const val CHANNEL_ID = "default_channel"
    const val CHANNEL_NAME = "Default Notifications"
  }
  
  override fun getName() = NAME
  
  override fun initialize() {
    super.initialize()
    createNotificationChannel()
  }
  
  private fun createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      val channel = NotificationChannel(
        CHANNEL_ID,
        CHANNEL_NAME,
        NotificationManager.IMPORTANCE_DEFAULT
      ).apply {
        description = "Default notification channel"
        enableVibration(true)
      }
      
      val manager = reactContext.getSystemService(NotificationManager::class.java)
      manager.createNotificationChannel(channel)
    }
  }
  
  @ReactMethod
  fun showNotification(options: ReadableMap, promise: Promise) {
    val title = options.getString("title") ?: "Notification"
    val body = options.getString("body") ?: ""
    val notificationId = options.getInt("id").takeIf { options.hasKey("id") } ?: 1
    
    try {
      val intent = reactContext.packageManager
        .getLaunchIntentForPackage(reactContext.packageName)
        ?.apply { flags = Intent.FLAG_ACTIVITY_SINGLE_TOP }
      
      val pendingIntent = PendingIntent.getActivity(
        reactContext,
        0,
        intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
      )
      
      val notification = NotificationCompat.Builder(reactContext, CHANNEL_ID)
        .setSmallIcon(android.R.drawable.ic_dialog_info)
        .setContentTitle(title)
        .setContentText(body)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()
      
      NotificationManagerCompat.from(reactContext)
        .notify(notificationId, notification)
      
      promise.resolve("Notification shown")
    } catch (e: Exception) {
      promise.reject("ERROR", e.message, e)
    }
  }
  
  @ReactMethod
  fun cancelNotification(id: Int, promise: Promise) {
    NotificationManagerCompat.from(reactContext).cancel(id)
    promise.resolve("Notification cancelled")
  }
  
  @ReactMethod
  fun cancelAllNotifications(promise: Promise) {
    NotificationManagerCompat.from(reactContext).cancelAll()
    promise.resolve("All notifications cancelled")
  }
  
  @ReactMethod
  fun checkNotificationPermission(promise: Promise) {
    val result = Arguments.createMap()
    val notificationManager = NotificationManagerCompat.from(reactContext)
    result.putBoolean("granted", notificationManager.areNotificationsEnabled())
    promise.resolve(result)
  }
  
  @ReactMethod
  fun scheduleNotification(options: ReadableMap, promise: Promise) {
    val title = options.getString("title") ?: "Notification"
    val body = options.getString("body") ?: ""
    val delayMs = options.getDouble("delay").toLong()
    val notificationId = options.getInt("id").takeIf { options.hasKey("id") } ?: 1
    
    // ใช้ Handler สำหรับ delay ง่ายๆ (production ควรใช้ WorkManager)
    android.os.Handler(android.os.Looper.getMainLooper()).postDelayed({
      val notification = NotificationCompat.Builder(reactContext, CHANNEL_ID)
        .setSmallIcon(android.R.drawable.ic_dialog_info)
        .setContentTitle(title)
        .setContentText(body)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setAutoCancel(true)
        .build()
      
      NotificationManagerCompat.from(reactContext)
        .notify(notificationId, notification)
    }, delayMs)
    
    promise.resolve("Notification scheduled")
  }
}
```

### Step 2: Package Registration

```kotlin
// NotificationPackage.kt
package com.myapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager

class NotificationPackage : ReactPackage {
  override fun createNativeModules(
    reactContext: ReactApplicationContext
  ): List<NativeModule> = listOf(NotificationModule(reactContext))
  
  override fun createViewManagers(
    reactContext: ReactApplicationContext
  ): List<ViewManager<*, *>> = emptyList()
}
```

### Step 3: JavaScript Service

```typescript
// NotificationService.ts
import { NativeModules, Platform } from 'react-native';

const { NotificationModule } = NativeModules;

interface NotificationOptions {
  id?: number;
  title: string;
  body: string;
  delay?: number;
}

export const NotificationService = {
  async show(options: NotificationOptions): Promise<void> {
    if (Platform.OS !== 'android') return;
    
    const hasPermission = await this.checkPermission();
    if (!hasPermission) {
      throw new Error('Notification permission not granted');
    }
    
    return NotificationModule.showNotification(options);
  },
  
  async schedule(options: NotificationOptions & { delay: number }): Promise<void> {
    if (Platform.OS !== 'android') return;
    return NotificationModule.scheduleNotification(options);
  },
  
  async cancel(id: number): Promise<void> {
    if (Platform.OS !== 'android') return;
    return NotificationModule.cancelNotification(id);
  },
  
  async cancelAll(): Promise<void> {
    if (Platform.OS !== 'android') return;
    return NotificationModule.cancelAllNotifications();
  },
  
  async checkPermission(): Promise<boolean> {
    if (Platform.OS !== 'android') return true;
    const result = await NotificationModule.checkNotificationPermission();
    return result.granted;
  }
};
```

### Step 4: React Component

```tsx
// NotificationDemo.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  TextInput,
  StyleSheet,
  Alert,
  ScrollView
} from 'react-native';
import { NotificationService } from './NotificationService';

const NotificationDemo: React.FC = () => {
  const [title, setTitle] = useState('');
  const [body, setBody] = useState('');
  const [delay, setDelay] = useState('');
  
  const handleShow = async () => {
    if (!title || !body) {
      Alert.alert('Error', 'กรุณากรอก title และ body');
      return;
    }
    
    try {
      await NotificationService.show({
        id: Date.now(),
        title,
        body
      });
      Alert.alert('สำเร็จ', 'แสดง notification แล้ว');
    } catch (error: any) {
      Alert.alert('Error', error.message);
    }
  };
  
  const handleSchedule = async () => {
    if (!title || !body || !delay) {
      Alert.alert('Error', 'กรุณากรอกข้อมูลให้ครบ');
      return;
    }
    
    try {
      const delayMs = parseInt(delay) * 1000;
      await NotificationService.schedule({
        id: Date.now(),
        title,
        body,
        delay: delayMs
      });
      Alert.alert('สำเร็จ', `จะแสดง notification ใน ${delay} วินาที`);
    } catch (error: any) {
      Alert.alert('Error', error.message);
    }
  };
  
  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>Notification Demo</Text>
      
      <TextInput
        style={styles.input}
        placeholder="หัวข้อ notification"
        value={title}
        onChangeText={setTitle}
      />
      
      <TextInput
        style={styles.input}
        placeholder="เนื้อหา notification"
        value={body}
        onChangeText={setBody}
        multiline
      />
      
      <TextInput
        style={styles.input}
        placeholder="delay (วินาที)"
        value={delay}
        onChangeText={setDelay}
        keyboardType="numeric"
      />
      
      <TouchableOpacity style={styles.button} onPress={handleShow}>
        <Text style={styles.buttonText}>แสดงทันที</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        style={[styles.button, styles.scheduleButton]}
        onPress={handleSchedule}
      >
        <Text style={styles.buttonText}>ตั้งเวลา</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        style={[styles.button, styles.cancelButton]}
        onPress={() => NotificationService.cancelAll()}
      >
        <Text style={styles.buttonText}>ยกเลิกทั้งหมด</Text>
      </TouchableOpacity>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#f5f5f5'
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center'
  },
  input: {
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    marginBottom: 12,
    fontSize: 16
  },
  button: {
    backgroundColor: '#007AFF',
    padding: 15,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 10
  },
  scheduleButton: {
    backgroundColor: '#34C759'
  },
  cancelButton: {
    backgroundColor: '#FF3B30'
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600'
  }
});

export default NotificationDemo;
```

---

## Tips และ Best Practices สำหรับ Android

### 1. Lifecycle Management
```kotlin
// ล้าง resources เมื่อ module ถูก destroy
override fun onCatalystInstanceDestroy() {
  super.onCatalystInstanceDestroy()
  // Unregister receivers, sensors, etc.
  cleanup()
}
```

### 2. Main Thread Operations
```kotlin
// Run on UI thread เมื่อจำเป็น
@ReactMethod
fun updateUI(message: String, promise: Promise) {
  currentActivity?.runOnUiThread {
    // UI operations here
    promise.resolve("UI updated")
  } ?: promise.reject("NO_ACTIVITY", "Activity not found")
}
```

### 3. Android API Level Checks
```kotlin
@ReactMethod
fun useNewAPI(promise: Promise) {
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    // Use Android 10+ API
    promise.resolve("Using new API")
  } else {
    // Fallback for older versions
    promise.resolve("Using legacy API")
  }
}
```

### 4. Proper Error Handling
```kotlin
@ReactMethod
fun riskyOperation(promise: Promise) {
  try {
    // operation
    promise.resolve("success")
  } catch (e: SecurityException) {
    promise.reject("SECURITY_ERROR", "Permission denied: ${e.message}", e)
  } catch (e: IllegalStateException) {
    promise.reject("STATE_ERROR", "Invalid state: ${e.message}", e)
  } catch (e: Exception) {
    promise.reject("UNKNOWN_ERROR", e.message ?: "Unknown error", e)
  }
}
```

---

## สรุป

Android Native Modules มีความสำคัญสำหรับ:
1. เข้าถึง Android-specific APIs
2. ใช้งาน Android libraries ที่มีอยู่
3. ปรับปรุง performance ของแอป
4. รับ system events

ควรระวัง:
- Thread management ระหว่าง JS thread และ UI thread
- Permission handling ที่ถูกต้อง
- Memory leaks จาก unregistered receivers/listeners
- API level compatibility
