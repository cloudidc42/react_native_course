# Part 020: Forms และ TextInput

## สารบัญ
1. TextInput Properties
2. Form Validation
3. React Hook Form
4. Formik
5. Keyboard Handling
6. Workshop: Login/Register Form

---

## 1. TextInput Properties

`TextInput` คือ component หลักสำหรับรับ input จากผู้ใช้ใน React Native

### Properties พื้นฐาน

```javascript
import React, { useState } from 'react';
import { TextInput, StyleSheet } from 'react-native';

function BasicInput() {
  const [text, setText] = useState('');
  
  return (
    <TextInput
      // ===== Value & Change =====
      value={text}
      onChangeText={setText}
      defaultValue="ค่าเริ่มต้น"  // uncontrolled component
      
      // ===== Placeholder =====
      placeholder="พิมพ์ข้อความที่นี่..."
      placeholderTextColor="#AAAAAA"
      
      // ===== Keyboard Type =====
      keyboardType="default"
      // Options: 'default', 'numeric', 'email-address', 'phone-pad',
      //          'decimal-pad', 'url', 'number-pad', 'visible-password'
      
      // ===== Auto Capitalize =====
      autoCapitalize="none"
      // Options: 'none', 'sentences', 'words', 'characters'
      
      // ===== Auto Correct =====
      autoCorrect={false}
      spellCheck={false}
      
      // ===== Secure Text (Password) =====
      secureTextEntry={false}
      
      // ===== Return Key =====
      returnKeyType="done"
      // Options: 'done', 'go', 'next', 'search', 'send', 'none', 'previous'
      
      // ===== Multiline =====
      multiline={false}
      numberOfLines={4}        // Android
      maxLength={100}
      
      // ===== Clear =====
      clearButtonMode="while-editing"  // iOS only
      // Options: 'never', 'while-editing', 'unless-editing', 'always'
      
      // ===== Events =====
      onFocus={() => console.log('focused')}
      onBlur={() => console.log('blurred')}
      onSubmitEditing={() => console.log('submitted')}
      onKeyPress={({ nativeEvent }) => {
        if (nativeEvent.key === 'Enter') console.log('Enter pressed');
      }}
      
      // ===== Style =====
      style={styles.input}
      
      // ===== Accessibility =====
      accessibilityLabel="ช่องกรอกข้อความ"
      testID="text-input"
    />
  );
}

const styles = StyleSheet.create({
  input: {
    height: 50,
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 10,
    paddingHorizontal: 14,
    fontSize: 16,
    backgroundColor: '#FAFAFA',
    color: '#212121',
  },
});
```

### TextInput Types ที่ใช้บ่อย

```javascript
// 1. Email
<TextInput
  keyboardType="email-address"
  autoCapitalize="none"
  autoCorrect={false}
  placeholder="อีเมล"
  textContentType="emailAddress"  // iOS autofill
/>

// 2. Password
<TextInput
  secureTextEntry
  autoCapitalize="none"
  autoCorrect={false}
  placeholder="รหัสผ่าน"
  textContentType="password"  // iOS autofill
/>

// 3. Phone
<TextInput
  keyboardType="phone-pad"
  placeholder="เบอร์โทรศัพท์"
  textContentType="telephoneNumber"  // iOS
/>

// 4. Number
<TextInput
  keyboardType="numeric"
  placeholder="จำนวน"
/>

// 5. Multiline (Textarea)
<TextInput
  multiline
  numberOfLines={4}  // Android
  textAlignVertical="top"  // Android
  style={{ height: 100, paddingTop: 12 }}
  placeholder="รายละเอียด..."
/>

// 6. Search
<TextInput
  returnKeyType="search"
  clearButtonMode="while-editing"  // iOS
  placeholder="ค้นหา..."
  onSubmitEditing={({ nativeEvent }) => handleSearch(nativeEvent.text)}
/>
```

### Controlled vs Uncontrolled

```javascript
// ✅ Controlled (แนะนำ)
function ControlledInput() {
  const [value, setValue] = useState('');
  return <TextInput value={value} onChangeText={setValue} />;
}

// Uncontrolled (ใช้ ref)
function UncontrolledInput() {
  const inputRef = useRef(null);
  
  const handleSubmit = () => {
    const value = inputRef.current.props.value;
    console.log(value);
  };
  
  return (
    <View>
      <TextInput ref={inputRef} defaultValue="" />
      <Button title="Submit" onPress={handleSubmit} />
    </View>
  );
}
```

---

## 2. Form Validation

### Validation แบบ Manual

```javascript
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet } from 'react-native';

const validators = {
  required: (value) => !value?.trim() ? 'จำเป็นต้องกรอก' : null,
  email: (value) => !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) ? 'อีเมลไม่ถูกต้อง' : null,
  minLength: (min) => (value) => value?.length < min ? `ต้องมีอย่างน้อย ${min} ตัวอักษร` : null,
  maxLength: (max) => (value) => value?.length > max ? `ต้องไม่เกิน ${max} ตัวอักษร` : null,
  pattern: (regex, message) => (value) => !regex.test(value) ? message : null,
  match: (fieldName, fieldValue) => (value) => value !== fieldValue ? `ต้องตรงกับ${fieldName}` : null,
};

function validate(value, rules) {
  for (const rule of rules) {
    const error = rule(value);
    if (error) return error;
  }
  return null;
}

function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const validateForm = (formValues) => {
    const newErrors = {};
    
    const emailError = validate(formValues.email, [
      validators.required,
      validators.email,
    ]);
    if (emailError) newErrors.email = emailError;
    
    const passwordError = validate(formValues.password, [
      validators.required,
      validators.minLength(8),
    ]);
    if (passwordError) newErrors.password = passwordError;
    
    return newErrors;
  };

  const handleBlur = (field) => {
    setTouched(prev => ({ ...prev, [field]: true }));
    const formErrors = validateForm({ email, password });
    setErrors(formErrors);
  };

  const handleSubmit = () => {
    setTouched({ email: true, password: true });
    const formErrors = validateForm({ email, password });
    setErrors(formErrors);
    
    if (Object.keys(formErrors).length === 0) {
      console.log('Form valid! Submit:', { email, password });
    }
  };

  return (
    <View style={styles.form}>
      <View style={styles.field}>
        <Text style={styles.label}>อีเมล</Text>
        <TextInput
          style={[styles.input, touched.email && errors.email && styles.inputError]}
          value={email}
          onChangeText={setEmail}
          onBlur={() => handleBlur('email')}
          placeholder="example@email.com"
          keyboardType="email-address"
          autoCapitalize="none"
        />
        {touched.email && errors.email && (
          <Text style={styles.errorText}>{errors.email}</Text>
        )}
      </View>

      <View style={styles.field}>
        <Text style={styles.label}>รหัสผ่าน</Text>
        <TextInput
          style={[styles.input, touched.password && errors.password && styles.inputError]}
          value={password}
          onChangeText={setPassword}
          onBlur={() => handleBlur('password')}
          placeholder="อย่างน้อย 8 ตัวอักษร"
          secureTextEntry
        />
        {touched.password && errors.password && (
          <Text style={styles.errorText}>{errors.password}</Text>
        )}
      </View>

      <TouchableOpacity style={styles.button} onPress={handleSubmit}>
        <Text style={styles.buttonText}>เข้าสู่ระบบ</Text>
      </TouchableOpacity>
    </View>
  );
}
```

---

## 3. React Hook Form

React Hook Form เป็น library สำหรับจัดการ form ที่ performance ดีที่สุด

### ติดตั้ง

```bash
npm install react-hook-form
```

### การใช้งานพื้นฐาน

```javascript
import { useForm, Controller } from 'react-hook-form';

function LoginFormRHF() {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting, isValid },
  } = useForm({
    defaultValues: {
      email: '',
      password: '',
    },
    mode: 'onBlur',  // 'onBlur' | 'onChange' | 'onSubmit' | 'onTouched' | 'all'
  });

  const onSubmit = async (data) => {
    console.log('Form data:', data);
    await loginUser(data);
  };

  return (
    <View>
      {/* Email */}
      <Controller
        control={control}
        name="email"
        rules={{
          required: 'กรุณากรอกอีเมล',
          pattern: {
            value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'รูปแบบอีเมลไม่ถูกต้อง',
          },
        }}
        render={({ field: { onChange, onBlur, value } }) => (
          <View>
            <TextInput
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              placeholder="อีเมล"
              keyboardType="email-address"
              autoCapitalize="none"
              style={[styles.input, errors.email && styles.inputError]}
            />
            {errors.email && (
              <Text style={styles.errorText}>{errors.email.message}</Text>
            )}
          </View>
        )}
      />

      {/* Password */}
      <Controller
        control={control}
        name="password"
        rules={{
          required: 'กรุณากรอกรหัสผ่าน',
          minLength: {
            value: 8,
            message: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
          },
        }}
        render={({ field: { onChange, onBlur, value } }) => (
          <View>
            <TextInput
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              placeholder="รหัสผ่าน"
              secureTextEntry
              style={[styles.input, errors.password && styles.inputError]}
            />
            {errors.password && (
              <Text style={styles.errorText}>{errors.password.message}</Text>
            )}
          </View>
        )}
      />

      <TouchableOpacity
        style={[styles.button, (!isValid || isSubmitting) && styles.disabled]}
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
      >
        <Text style={styles.buttonText}>
          {isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}
```

### RHF สำหรับ Register Form ที่ซับซ้อน

```javascript
import { useForm, Controller, useWatch } from 'react-hook-form';

function RegisterFormRHF() {
  const {
    control,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting },
  } = useForm({
    defaultValues: {
      fullName: '',
      email: '',
      phone: '',
      password: '',
      confirmPassword: '',
      agreeToTerms: false,
    },
  });

  const password = watch('password');

  const onSubmit = async (data) => {
    const { confirmPassword, agreeToTerms, ...userData } = data;
    await registerUser(userData);
  };

  return (
    <ScrollView>
      {/* Full Name */}
      <Controller
        control={control}
        name="fullName"
        rules={{
          required: 'กรุณากรอกชื่อ-นามสกุล',
          minLength: { value: 3, message: 'ต้องมีอย่างน้อย 3 ตัวอักษร' },
          pattern: {
            value: /^[฀-๿a-zA-Z\s]+$/,
            message: 'ชื่อต้องเป็นภาษาไทยหรืออังกฤษเท่านั้น',
          },
        }}
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.fieldGroup}>
            <Text style={styles.label}>ชื่อ-นามสกุล *</Text>
            <TextInput
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              placeholder="สมชาย ใจดี"
              style={[styles.input, errors.fullName && styles.inputError]}
            />
            {errors.fullName && (
              <Text style={styles.errorText}>⚠️ {errors.fullName.message}</Text>
            )}
          </View>
        )}
      />

      {/* Password with Confirm */}
      <Controller
        control={control}
        name="password"
        rules={{
          required: 'กรุณากรอกรหัสผ่าน',
          minLength: { value: 8, message: 'ต้องมีอย่างน้อย 8 ตัวอักษร' },
          pattern: {
            value: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
            message: 'ต้องมีตัวพิมพ์เล็ก ใหญ่ และตัวเลข',
          },
        }}
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.fieldGroup}>
            <Text style={styles.label}>รหัสผ่าน *</Text>
            <TextInput
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              placeholder="อย่างน้อย 8 ตัวอักษร"
              secureTextEntry
              style={[styles.input, errors.password && styles.inputError]}
            />
            {errors.password && (
              <Text style={styles.errorText}>⚠️ {errors.password.message}</Text>
            )}
          </View>
        )}
      />

      <Controller
        control={control}
        name="confirmPassword"
        rules={{
          required: 'กรุณายืนยันรหัสผ่าน',
          validate: (value) =>
            value === password || 'รหัสผ่านไม่ตรงกัน',
        }}
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.fieldGroup}>
            <Text style={styles.label}>ยืนยันรหัสผ่าน *</Text>
            <TextInput
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              placeholder="พิมพ์รหัสผ่านอีกครั้ง"
              secureTextEntry
              style={[styles.input, errors.confirmPassword && styles.inputError]}
            />
            {errors.confirmPassword && (
              <Text style={styles.errorText}>⚠️ {errors.confirmPassword.message}</Text>
            )}
          </View>
        )}
      />

      <TouchableOpacity
        style={[styles.button, isSubmitting && styles.disabled]}
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
      >
        <Text style={styles.buttonText}>
          {isSubmitting ? 'กำลังสมัคร...' : 'สมัครสมาชิก'}
        </Text>
      </TouchableOpacity>
    </ScrollView>
  );
}
```

---

## 4. Formik

Formik เป็นอีก library ที่นิยมสำหรับ form management

### ติดตั้ง

```bash
npm install formik yup
```

### การใช้งาน Formik + Yup

```javascript
import { Formik } from 'formik';
import * as Yup from 'yup';

const loginSchema = Yup.object({
  email: Yup.string()
    .email('อีเมลไม่ถูกต้อง')
    .required('กรุณากรอกอีเมล'),
  password: Yup.string()
    .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    .required('กรุณากรอกรหัสผ่าน'),
});

function LoginFormFormik() {
  return (
    <Formik
      initialValues={{ email: '', password: '' }}
      validationSchema={loginSchema}
      onSubmit={async (values, { setSubmitting }) => {
        try {
          await loginUser(values);
        } finally {
          setSubmitting(false);
        }
      }}
    >
      {({
        handleChange,
        handleBlur,
        handleSubmit,
        values,
        errors,
        touched,
        isSubmitting,
      }) => (
        <View>
          <TextInput
            placeholder="อีเมล"
            value={values.email}
            onChangeText={handleChange('email')}
            onBlur={handleBlur('email')}
            style={[styles.input, touched.email && errors.email && styles.inputError]}
            keyboardType="email-address"
            autoCapitalize="none"
          />
          {touched.email && errors.email && (
            <Text style={styles.errorText}>{errors.email}</Text>
          )}

          <TextInput
            placeholder="รหัสผ่าน"
            value={values.password}
            onChangeText={handleChange('password')}
            onBlur={handleBlur('password')}
            secureTextEntry
            style={[styles.input, touched.password && errors.password && styles.inputError]}
          />
          {touched.password && errors.password && (
            <Text style={styles.errorText}>{errors.password}</Text>
          )}

          <TouchableOpacity
            style={styles.button}
            onPress={handleSubmit}
            disabled={isSubmitting}
          >
            <Text style={styles.buttonText}>
              {isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
            </Text>
          </TouchableOpacity>
        </View>
      )}
    </Formik>
  );
}
```

---

## 5. Keyboard Handling

```javascript
import { 
  KeyboardAvoidingView, 
  Platform, 
  ScrollView,
  Keyboard,
  TouchableWithoutFeedback 
} from 'react-native';

// KeyboardAvoidingView - เลื่อน content ขึ้นเมื่อ keyboard เปิด
function FormWithKeyboard() {
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 64 : 0}
    >
      <TouchableWithoutFeedback onPress={Keyboard.dismiss}>
        <ScrollView>
          <TextInput placeholder="ชื่อ" style={styles.input} />
          <TextInput placeholder="อีเมล" style={styles.input} />
          <TextInput placeholder="ข้อความ" multiline style={styles.textarea} />
          <TouchableOpacity style={styles.button}>
            <Text style={styles.buttonText}>ส่ง</Text>
          </TouchableOpacity>
        </ScrollView>
      </TouchableWithoutFeedback>
    </KeyboardAvoidingView>
  );
}
```

---

## Workshop: Login/Register Form

### สร้าง Auth Form สมบูรณ์

```javascript
// screens/AuthScreen.js
import React, { useState, useRef } from 'react';
import {
  View, Text, TextInput, TouchableOpacity,
  StyleSheet, ScrollView, KeyboardAvoidingView,
  Platform, Alert, ActivityIndicator, Animated
} from 'react-native';
import { useForm, Controller } from 'react-hook-form';

// Input Component แบบ Reusable
function FormInput({
  label,
  error,
  touched,
  secureEntry = false,
  rightElement,
  ...inputProps
}) {
  const [isFocused, setIsFocused] = useState(false);

  return (
    <View style={styles.fieldGroup}>
      {label && <Text style={styles.label}>{label}</Text>}
      <View style={[
        styles.inputWrapper,
        isFocused && styles.inputFocused,
        touched && error && styles.inputErrorWrapper,
      ]}>
        <TextInput
          style={styles.input}
          secureTextEntry={secureEntry}
          onFocus={() => setIsFocused(true)}
          onBlur={() => setIsFocused(false)}
          placeholderTextColor="#AAAAAA"
          {...inputProps}
        />
        {rightElement}
      </View>
      {touched && error && (
        <Text style={styles.errorText}>⚠️ {error}</Text>
      )}
    </View>
  );
}

// Password Input with Toggle
function PasswordInput({ control, name, label, rules, ...props }) {
  const [showPassword, setShowPassword] = useState(false);

  return (
    <Controller
      control={control}
      name={name}
      rules={rules}
      render={({ field: { onChange, onBlur, value }, fieldState: { error, isTouched } }) => (
        <FormInput
          label={label}
          value={value}
          onChangeText={onChange}
          onBlur={onBlur}
          secureEntry={!showPassword}
          error={error?.message}
          touched={isTouched}
          rightElement={
            <TouchableOpacity
              onPress={() => setShowPassword(!showPassword)}
              style={styles.eyeBtn}
            >
              <Text>{showPassword ? '🙈' : '👁️'}</Text>
            </TouchableOpacity>
          }
          {...props}
        />
      )}
    />
  );
}

// Tab: Login / Register
function AuthScreen({ navigation }) {
  const [activeTab, setActiveTab] = useState('login');
  const tabIndicator = useRef(new Animated.Value(0)).current;

  const switchTab = (tab) => {
    setActiveTab(tab);
    Animated.timing(tabIndicator, {
      toValue: tab === 'login' ? 0 : 1,
      duration: 200,
      useNativeDriver: false,
    }).start();
  };

  const indicatorLeft = tabIndicator.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '50%'],
  });

  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView
        contentContainerStyle={styles.container}
        keyboardShouldPersistTaps="handled"
        showsVerticalScrollIndicator={false}
      >
        {/* Logo */}
        <View style={styles.logoSection}>
          <Text style={styles.logo}>🚀</Text>
          <Text style={styles.appName}>MyApp</Text>
          <Text style={styles.tagline}>ยินดีต้อนรับ!</Text>
        </View>

        {/* Tabs */}
        <View style={styles.tabContainer}>
          <View style={styles.tabs}>
            <TouchableOpacity
              style={styles.tab}
              onPress={() => switchTab('login')}
            >
              <Text style={[styles.tabText, activeTab === 'login' && styles.activeTabText]}>
                เข้าสู่ระบบ
              </Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={styles.tab}
              onPress={() => switchTab('register')}
            >
              <Text style={[styles.tabText, activeTab === 'register' && styles.activeTabText]}>
                สมัครสมาชิก
              </Text>
            </TouchableOpacity>
          </View>
          <View style={styles.tabIndicatorContainer}>
            <Animated.View style={[styles.tabIndicator, { left: indicatorLeft }]} />
          </View>
        </View>

        {/* Form */}
        {activeTab === 'login'
          ? <LoginForm navigation={navigation} />
          : <RegisterForm navigation={navigation} />
        }
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

// Login Form
function LoginForm({ navigation }) {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    defaultValues: { email: '', password: '' },
    mode: 'onBlur',
  });

  const passwordRef = useRef(null);

  const onSubmit = async ({ email, password }) => {
    try {
      await new Promise(r => setTimeout(r, 1500));  // Simulate API call
      Alert.alert('สำเร็จ', 'เข้าสู่ระบบแล้ว!', [
        { text: 'ตกลง', onPress: () => navigation.replace('Home') }
      ]);
    } catch {
      Alert.alert('ผิดพลาด', 'อีเมลหรือรหัสผ่านไม่ถูกต้อง');
    }
  };

  return (
    <View style={styles.form}>
      <Controller
        control={control}
        name="email"
        rules={{
          required: 'กรุณากรอกอีเมล',
          pattern: {
            value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'รูปแบบอีเมลไม่ถูกต้อง',
          },
        }}
        render={({ field: { onChange, onBlur, value }, fieldState: { error, isTouched } }) => (
          <FormInput
            label="อีเมล"
            placeholder="example@email.com"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={error?.message}
            touched={isTouched}
            keyboardType="email-address"
            autoCapitalize="none"
            returnKeyType="next"
            onSubmitEditing={() => passwordRef.current?.focus()}
          />
        )}
      />

      <PasswordInput
        control={control}
        name="password"
        label="รหัสผ่าน"
        placeholder="กรอกรหัสผ่าน"
        rules={{
          required: 'กรุณากรอกรหัสผ่าน',
          minLength: { value: 6, message: 'ต้องมีอย่างน้อย 6 ตัวอักษร' },
        }}
        returnKeyType="done"
        onSubmitEditing={handleSubmit(onSubmit)}
      />

      <TouchableOpacity style={styles.forgotPassword}>
        <Text style={styles.forgotPasswordText}>ลืมรหัสผ่าน?</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={[styles.primaryButton, isSubmitting && styles.disabledButton]}
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
      >
        {isSubmitting
          ? <ActivityIndicator color="#FFF" />
          : <Text style={styles.primaryButtonText}>เข้าสู่ระบบ</Text>
        }
      </TouchableOpacity>

      <View style={styles.divider}>
        <View style={styles.dividerLine} />
        <Text style={styles.dividerText}>หรือ</Text>
        <View style={styles.dividerLine} />
      </View>

      <View style={styles.socialButtons}>
        <TouchableOpacity style={styles.socialBtn}>
          <Text style={styles.socialBtnText}>🌐 Google</Text>
        </TouchableOpacity>
        <TouchableOpacity style={[styles.socialBtn, styles.facebookBtn]}>
          <Text style={[styles.socialBtnText, { color: '#FFF' }]}>📘 Facebook</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

// Register Form
function RegisterForm({ navigation }) {
  const {
    control,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting },
  } = useForm({
    defaultValues: {
      fullName: '',
      email: '',
      phone: '',
      password: '',
      confirmPassword: '',
    },
    mode: 'onBlur',
  });

  const password = watch('password');

  const onSubmit = async (data) => {
    try {
      await new Promise(r => setTimeout(r, 2000));
      Alert.alert('สำเร็จ', 'สมัครสมาชิกแล้ว! กรุณาตรวจสอบอีเมลเพื่อยืนยัน', [
        { text: 'ตกลง' }
      ]);
    } catch {
      Alert.alert('ผิดพลาด', 'เกิดข้อผิดพลาด กรุณาลองใหม่');
    }
  };

  return (
    <View style={styles.form}>
      <Controller
        control={control}
        name="fullName"
        rules={{
          required: 'กรุณากรอกชื่อ-นามสกุล',
          minLength: { value: 3, message: 'ต้องมีอย่างน้อย 3 ตัวอักษร' },
        }}
        render={({ field: { onChange, onBlur, value }, fieldState: { error, isTouched } }) => (
          <FormInput
            label="ชื่อ-นามสกุล"
            placeholder="สมชาย ใจดี"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={error?.message}
            touched={isTouched}
          />
        )}
      />

      <Controller
        control={control}
        name="email"
        rules={{
          required: 'กรุณากรอกอีเมล',
          pattern: {
            value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
            message: 'รูปแบบอีเมลไม่ถูกต้อง',
          },
        }}
        render={({ field: { onChange, onBlur, value }, fieldState: { error, isTouched } }) => (
          <FormInput
            label="อีเมล"
            placeholder="example@email.com"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={error?.message}
            touched={isTouched}
            keyboardType="email-address"
            autoCapitalize="none"
          />
        )}
      />

      <Controller
        control={control}
        name="phone"
        rules={{
          required: 'กรุณากรอกเบอร์โทร',
          pattern: {
            value: /^0[0-9]{9}$/,
            message: 'เบอร์โทรต้องเป็น 10 หลัก เริ่มต้นด้วย 0',
          },
        }}
        render={({ field: { onChange, onBlur, value }, fieldState: { error, isTouched } }) => (
          <FormInput
            label="เบอร์โทรศัพท์"
            placeholder="0812345678"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            error={error?.message}
            touched={isTouched}
            keyboardType="phone-pad"
          />
        )}
      />

      <PasswordInput
        control={control}
        name="password"
        label="รหัสผ่าน"
        placeholder="อย่างน้อย 8 ตัวอักษร"
        rules={{
          required: 'กรุณากรอกรหัสผ่าน',
          minLength: { value: 8, message: 'ต้องมีอย่างน้อย 8 ตัวอักษร' },
          pattern: {
            value: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
            message: 'ต้องมีตัวพิมพ์เล็ก ใหญ่ และตัวเลข',
          },
        }}
      />

      <PasswordInput
        control={control}
        name="confirmPassword"
        label="ยืนยันรหัสผ่าน"
        placeholder="พิมพ์รหัสผ่านอีกครั้ง"
        rules={{
          required: 'กรุณายืนยันรหัสผ่าน',
          validate: (value) => value === password || 'รหัสผ่านไม่ตรงกัน',
        }}
      />

      <TouchableOpacity
        style={[styles.primaryButton, isSubmitting && styles.disabledButton]}
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
      >
        {isSubmitting
          ? <ActivityIndicator color="#FFF" />
          : <Text style={styles.primaryButtonText}>สมัครสมาชิก</Text>
        }
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flexGrow: 1, padding: 20, backgroundColor: '#F5F5F5' },
  logoSection: { alignItems: 'center', paddingVertical: 32 },
  logo: { fontSize: 60 },
  appName: { fontSize: 28, fontWeight: 'bold', color: '#6200EE', marginTop: 8 },
  tagline: { fontSize: 16, color: '#888', marginTop: 4 },
  tabContainer: { backgroundColor: '#FFF', borderRadius: 14, overflow: 'hidden', marginBottom: 20 },
  tabs: { flexDirection: 'row' },
  tab: { flex: 1, paddingVertical: 14, alignItems: 'center' },
  tabText: { fontSize: 15, color: '#888', fontWeight: '500' },
  activeTabText: { color: '#6200EE', fontWeight: '700' },
  tabIndicatorContainer: { height: 3, backgroundColor: '#EEE' },
  tabIndicator: {
    position: 'absolute',
    width: '50%',
    height: 3,
    backgroundColor: '#6200EE',
    borderRadius: 2,
  },
  form: { backgroundColor: '#FFF', borderRadius: 14, padding: 20 },
  fieldGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', color: '#444', marginBottom: 6 },
  inputWrapper: {
    flexDirection: 'row',
    alignItems: 'center',
    borderWidth: 1.5,
    borderColor: '#E0E0E0',
    borderRadius: 10,
    backgroundColor: '#FAFAFA',
  },
  inputFocused: { borderColor: '#6200EE', backgroundColor: '#FFF' },
  inputErrorWrapper: { borderColor: '#E53935' },
  input: {
    flex: 1,
    paddingHorizontal: 14,
    paddingVertical: 12,
    fontSize: 15,
    color: '#212121',
  },
  eyeBtn: { padding: 12 },
  errorText: { fontSize: 12, color: '#E53935', marginTop: 4 },
  forgotPassword: { alignSelf: 'flex-end', marginBottom: 20 },
  forgotPasswordText: { color: '#6200EE', fontSize: 14 },
  primaryButton: {
    backgroundColor: '#6200EE',
    borderRadius: 12,
    paddingVertical: 15,
    alignItems: 'center',
    shadowColor: '#6200EE',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 5,
  },
  disabledButton: { backgroundColor: '#B39DDB', elevation: 0, shadowOpacity: 0 },
  primaryButtonText: { color: '#FFF', fontSize: 16, fontWeight: '700' },
  divider: { flexDirection: 'row', alignItems: 'center', marginVertical: 20 },
  dividerLine: { flex: 1, height: 1, backgroundColor: '#EEE' },
  dividerText: { marginHorizontal: 12, color: '#BBB', fontSize: 13 },
  socialButtons: { flexDirection: 'row', gap: 12 },
  socialBtn: {
    flex: 1,
    borderWidth: 1.5,
    borderColor: '#E0E0E0',
    borderRadius: 10,
    paddingVertical: 12,
    alignItems: 'center',
  },
  facebookBtn: { backgroundColor: '#1877F2', borderColor: '#1877F2' },
  socialBtnText: { fontSize: 14, fontWeight: '600', color: '#212121' },
});

export default AuthScreen;
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ React Hook Form สำหรับ form ขนาดใหญ่
- ใช้ KeyboardAvoidingView ทุกครั้งที่มี input
- Validate ทั้ง client-side และ server-side
- แสดง error เฉพาะ field ที่ touched แล้ว
- ใช้ ref เพื่อ focus field ถัดไปอัตโนมัติ

❌ DON'T:
- ไม่ validate เฉพาะ onSubmit (ผู้ใช้รอนาน)
- ไม่ show error ก่อนที่ user จะ touch field
- ไม่ทำ validation ซับซ้อนใน render
- ไม่ลืม disable submit button ระหว่าง submitting

📱 UX Best Practices:
- ใช้ returnKeyType เพื่อ flow ที่ดี
- เพิ่ม clearButtonMode="while-editing" สำหรับ iOS
- ใช้ autoFocus สำหรับ field แรก
- เพิ่ม maxLength เพื่อ prevent overflow
```

---

## สรุป

Forms เป็นส่วนสำคัญของทุก app ที่เราได้เรียนรู้:

1. **TextInput Properties** - ทุก property ที่ใช้บ่อย
2. **Manual Validation** - สร้าง validation เอง
3. **React Hook Form** - library ที่ performance ดี
4. **Formik + Yup** - อีกทางเลือกที่นิยม
5. **Keyboard Handling** - จัดการ keyboard อย่างถูกต้อง
6. **Workshop** - Login/Register form สมบูรณ์แบบ
