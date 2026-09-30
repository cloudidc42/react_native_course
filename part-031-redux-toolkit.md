# Part 031: Redux Toolkit - State Management

## ทำไมต้องใช้ Redux?

ในการพัฒนา React Native application ขนาดใหญ่ การจัดการ state ด้วย useState และ useContext อาจไม่เพียงพอ เมื่อ application มีความซับซ้อนมากขึ้น ปัญหาที่พบบ่อย ได้แก่:

### ปัญหาของ State Management ปกติ

1. **Prop Drilling** - การส่ง props ผ่านหลายชั้น component
2. **State Synchronization** - การ sync state ระหว่าง component ที่ไม่เกี่ยวข้องกัน
3. **Debugging ยาก** - ไม่รู้ว่า state เปลี่ยนแปลงที่ไหน เมื่อไหร่
4. **Testing ซับซ้อน** - Logic กระจายอยู่ใน component

### Redux แก้ปัญหาอย่างไร?

Redux เป็น predictable state container ที่:
- เก็บ state ทั้งหมดไว้ใน **Single Store**
- State เปลี่ยนได้ผ่าน **Actions** เท่านั้น
- การเปลี่ยนแปลงเกิดจาก **Pure Functions (Reducers)**

```
User Action → Dispatch Action → Reducer → New State → UI Update
```

### Redux Toolkit คืออะไร?

Redux Toolkit (RTK) คือ official toolset สำหรับ Redux ที่ทำให้:
- เขียนโค้ดน้อยลง
- ลด boilerplate
- มี best practices built-in
- รองรับ TypeScript ดีมาก

---

## การติดตั้ง Redux Toolkit

```bash
# สำหรับ React Native
npm install @reduxjs/toolkit react-redux

# หรือ
yarn add @reduxjs/toolkit react-redux
```

### โครงสร้างโปรเจกต์แนะนำ

```
src/
├── store/
│   ├── index.ts          # Configure store
│   ├── hooks.ts          # Typed hooks
│   └── slices/
│       ├── authSlice.ts
│       ├── todosSlice.ts
│       └── uiSlice.ts
├── screens/
└── components/
```

---

## createSlice - หัวใจของ Redux Toolkit

`createSlice` รวม actions และ reducer ไว้ในที่เดียว

### ตัวอย่างพื้นฐาน - Counter Slice

```typescript
// src/store/slices/counterSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

// กำหนด type ของ state
interface CounterState {
  value: number;
  step: number;
}

// กำหนด initial state
const initialState: CounterState = {
  value: 0,
  step: 1,
};

// สร้าง slice
const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    // Action ที่ไม่รับ payload
    increment: (state) => {
      state.value += state.step; // Immer ทำให้ mutate ได้โดยตรง
    },
    decrement: (state) => {
      state.value -= state.step;
    },
    reset: (state) => {
      state.value = 0;
    },
    // Action ที่รับ payload
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
    setStep: (state, action: PayloadAction<number>) => {
      state.step = action.payload;
    },
  },
});

// Export actions
export const { increment, decrement, reset, incrementByAmount, setStep } =
  counterSlice.actions;

// Export reducer
export default counterSlice.reducer;

// Export selectors
export const selectCount = (state: { counter: CounterState }) =>
  state.counter.value;
export const selectStep = (state: { counter: CounterState }) =>
  state.counter.step;
```

### ตัวอย่าง - Todos Slice

```typescript
// src/store/slices/todosSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: number;
}

interface TodosState {
  items: Todo[];
  filter: 'all' | 'active' | 'completed';
  searchQuery: string;
}

const initialState: TodosState = {
  items: [],
  filter: 'all',
  searchQuery: '',
};

const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    addTodo: (
      state,
      action: PayloadAction<{
        text: string;
        priority: Todo['priority'];
      }>
    ) => {
      const newTodo: Todo = {
        id: Date.now().toString(),
        text: action.payload.text,
        completed: false,
        priority: action.payload.priority,
        createdAt: Date.now(),
      };
      state.items.push(newTodo);
    },

    toggleTodo: (state, action: PayloadAction<string>) => {
      const todo = state.items.find((item) => item.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed;
      }
    },

    deleteTodo: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter((item) => item.id !== action.payload);
    },

    editTodo: (
      state,
      action: PayloadAction<{ id: string; text: string }>
    ) => {
      const todo = state.items.find((item) => item.id === action.payload.id);
      if (todo) {
        todo.text = action.payload.text;
      }
    },

    setPriority: (
      state,
      action: PayloadAction<{ id: string; priority: Todo['priority'] }>
    ) => {
      const todo = state.items.find((item) => item.id === action.payload.id);
      if (todo) {
        todo.priority = action.payload.priority;
      }
    },

    setFilter: (state, action: PayloadAction<TodosState['filter']>) => {
      state.filter = action.payload;
    },

    setSearchQuery: (state, action: PayloadAction<string>) => {
      state.searchQuery = action.payload;
    },

    clearCompleted: (state) => {
      state.items = state.items.filter((item) => !item.completed);
    },

    reorderTodos: (
      state,
      action: PayloadAction<{ fromIndex: number; toIndex: number }>
    ) => {
      const { fromIndex, toIndex } = action.payload;
      const [removed] = state.items.splice(fromIndex, 1);
      state.items.splice(toIndex, 0, removed);
    },
  },
});

export const {
  addTodo,
  toggleTodo,
  deleteTodo,
  editTodo,
  setPriority,
  setFilter,
  setSearchQuery,
  clearCompleted,
  reorderTodos,
} = todosSlice.actions;

export default todosSlice.reducer;

// Selectors
export const selectAllTodos = (state: { todos: TodosState }) =>
  state.todos.items;

export const selectFilter = (state: { todos: TodosState }) =>
  state.todos.filter;

export const selectFilteredTodos = (state: { todos: TodosState }) => {
  const { items, filter, searchQuery } = state.todos;

  let filtered = items;

  // Apply filter
  if (filter === 'active') {
    filtered = filtered.filter((todo) => !todo.completed);
  } else if (filter === 'completed') {
    filtered = filtered.filter((todo) => todo.completed);
  }

  // Apply search
  if (searchQuery) {
    filtered = filtered.filter((todo) =>
      todo.text.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }

  return filtered;
};

export const selectTodoStats = (state: { todos: TodosState }) => {
  const items = state.todos.items;
  return {
    total: items.length,
    completed: items.filter((t) => t.completed).length,
    active: items.filter((t) => !t.completed).length,
  };
};
```

---

## configureStore - ตั้งค่า Redux Store

```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './slices/counterSlice';
import todosReducer from './slices/todosSlice';
import authReducer from './slices/authSlice';

export const store = configureStore({
  reducer: {
    counter: counterReducer,
    todos: todosReducer,
    auth: authReducer,
  },
  // Middleware (RTK มี redux-thunk และ serializability check built-in)
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        // Ignore ค่าที่ไม่ serialize ได้
        ignoredActions: ['auth/setUser'],
        ignoredPaths: ['auth.user.createdAt'],
      },
    }),
  // เปิด DevTools เฉพาะ dev mode
  devTools: process.env.NODE_ENV !== 'production',
});

// Infer the `RootState` and `AppDispatch` types from the store itself
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Typed Hooks

```typescript
// src/store/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

// ใช้ hooks เหล่านี้แทน plain useDispatch และ useSelector
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

---

## เชื่อม Redux กับ React Native App

```typescript
// App.tsx
import React from 'react';
import { Provider } from 'react-redux';
import { store } from './src/store';
import AppNavigator from './src/navigation/AppNavigator';

export default function App() {
  return (
    <Provider store={store}>
      <AppNavigator />
    </Provider>
  );
}
```

---

## useSelector และ useDispatch

### useSelector - อ่าน State

```typescript
// ใช้ useAppSelector ที่เราสร้างไว้
import { useAppSelector } from '../store/hooks';
import {
  selectFilteredTodos,
  selectTodoStats,
  selectFilter,
} from '../store/slices/todosSlice';

function TodoList() {
  // เลือก filtered todos
  const todos = useAppSelector(selectFilteredTodos);

  // เลือก stats
  const stats = useAppSelector(selectTodoStats);

  // เลือก filter
  const currentFilter = useAppSelector(selectFilter);

  // หรือเลือก state โดยตรง
  const allItems = useAppSelector((state) => state.todos.items);

  return (
    // ...
  );
}
```

### useDispatch - ส่ง Actions

```typescript
import { useAppDispatch } from '../store/hooks';
import {
  addTodo,
  toggleTodo,
  deleteTodo,
  setFilter,
} from '../store/slices/todosSlice';

function TodoActions() {
  const dispatch = useAppDispatch();

  const handleAdd = (text: string) => {
    dispatch(addTodo({ text, priority: 'medium' }));
  };

  const handleToggle = (id: string) => {
    dispatch(toggleTodo(id));
  };

  const handleDelete = (id: string) => {
    dispatch(deleteTodo(id));
  };

  const handleFilterChange = (filter: 'all' | 'active' | 'completed') => {
    dispatch(setFilter(filter));
  };

  return (
    // ...
  );
}
```

---

## Redux DevTools

Redux DevTools ช่วยให้ debug state changes ได้ง่าย

### ติดตั้งสำหรับ React Native

```bash
npm install --save-dev redux-devtools-extension
```

### ใช้กับ Flipper (React Native Debugger)

```bash
# ติดตั้ง Flipper plugin
npm install --save-dev react-native-flipper redux-flipper
```

```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import todosReducer from './slices/todosSlice';

// เพิ่ม redux-flipper middleware ใน development mode
const middlewareEnhancers = [];

if (__DEV__) {
  const createDebugger = require('redux-flipper').default;
  middlewareEnhancers.push(createDebugger());
}

export const store = configureStore({
  reducer: {
    todos: todosReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(middlewareEnhancers),
});
```

### Time Travel Debugging

Redux DevTools รองรับ time travel ซึ่งทำให้:
- ย้อนดู actions ที่ผ่านมา
- ย้อน state กลับไปจุดก่อนหน้า
- Import/Export state
- Skip actions บางตัว

---

## Workshop: Todo App ด้วย Redux Toolkit

ตอนนี้เราจะสร้าง Todo App แบบสมบูรณ์พร้อม Redux

### 1. Setup Project Structure

```
src/
├── store/
│   ├── index.ts
│   ├── hooks.ts
│   └── slices/
│       └── todosSlice.ts
├── screens/
│   └── TodoScreen.tsx
└── components/
    ├── TodoItem.tsx
    ├── AddTodoForm.tsx
    └── TodoFilters.tsx
```

### 2. Auth Slice (ตัวอย่างเพิ่มเติม)

```typescript
// src/store/slices/authSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  isAuthenticated: false,
  isLoading: false,
  error: null,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    loginStart: (state) => {
      state.isLoading = true;
      state.error = null;
    },
    loginSuccess: (state, action: PayloadAction<User>) => {
      state.user = action.payload;
      state.isAuthenticated = true;
      state.isLoading = false;
      state.error = null;
    },
    loginFailure: (state, action: PayloadAction<string>) => {
      state.isLoading = false;
      state.error = action.payload;
    },
    logout: (state) => {
      state.user = null;
      state.isAuthenticated = false;
      state.error = null;
    },
    clearError: (state) => {
      state.error = null;
    },
  },
});

export const { loginStart, loginSuccess, loginFailure, logout, clearError } =
  authSlice.actions;

export default authSlice.reducer;

export const selectUser = (state: { auth: AuthState }) => state.auth.user;
export const selectIsAuthenticated = (state: { auth: AuthState }) =>
  state.auth.isAuthenticated;
export const selectAuthLoading = (state: { auth: AuthState }) =>
  state.auth.isLoading;
export const selectAuthError = (state: { auth: AuthState }) => state.auth.error;
```

### 3. TodoItem Component

```typescript
// src/components/TodoItem.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  TextInput,
  StyleSheet,
  Animated,
} from 'react-native';
import { useAppDispatch } from '../store/hooks';
import {
  toggleTodo,
  deleteTodo,
  editTodo,
  setPriority,
  Todo,
} from '../store/slices/todosSlice';

interface TodoItemProps {
  todo: Todo;
}

const PRIORITY_COLORS = {
  low: '#4CAF50',
  medium: '#FF9800',
  high: '#F44336',
};

export const TodoItem: React.FC<TodoItemProps> = ({ todo }) => {
  const dispatch = useAppDispatch();
  const [isEditing, setIsEditing] = useState(false);
  const [editText, setEditText] = useState(todo.text);

  const handleToggle = () => {
    dispatch(toggleTodo(todo.id));
  };

  const handleDelete = () => {
    dispatch(deleteTodo(todo.id));
  };

  const handleEdit = () => {
    if (editText.trim()) {
      dispatch(editTodo({ id: todo.id, text: editText.trim() }));
    }
    setIsEditing(false);
  };

  const handlePriorityChange = (priority: Todo['priority']) => {
    dispatch(setPriority({ id: todo.id, priority }));
  };

  return (
    <View style={[styles.container, todo.completed && styles.completed]}>
      {/* Checkbox */}
      <TouchableOpacity onPress={handleToggle} style={styles.checkbox}>
        <View
          style={[styles.checkboxInner, todo.completed && styles.checkboxChecked]}
        >
          {todo.completed && <Text style={styles.checkmark}>✓</Text>}
        </View>
      </TouchableOpacity>

      {/* Content */}
      <View style={styles.content}>
        {isEditing ? (
          <TextInput
            value={editText}
            onChangeText={setEditText}
            onBlur={handleEdit}
            onSubmitEditing={handleEdit}
            autoFocus
            style={styles.editInput}
          />
        ) : (
          <TouchableOpacity onLongPress={() => setIsEditing(true)}>
            <Text
              style={[styles.text, todo.completed && styles.completedText]}
            >
              {todo.text}
            </Text>
          </TouchableOpacity>
        )}

        {/* Priority */}
        <View style={styles.priorityContainer}>
          {(['low', 'medium', 'high'] as Todo['priority'][]).map((p) => (
            <TouchableOpacity
              key={p}
              onPress={() => handlePriorityChange(p)}
              style={[
                styles.priorityBadge,
                { backgroundColor: PRIORITY_COLORS[p] },
                todo.priority === p && styles.activePriority,
              ]}
            >
              <Text style={styles.priorityText}>{p}</Text>
            </TouchableOpacity>
          ))}
        </View>
      </View>

      {/* Delete button */}
      <TouchableOpacity onPress={handleDelete} style={styles.deleteButton}>
        <Text style={styles.deleteText}>🗑️</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#fff',
    padding: 12,
    marginVertical: 4,
    marginHorizontal: 16,
    borderRadius: 8,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  completed: {
    opacity: 0.6,
  },
  checkbox: {
    marginRight: 12,
  },
  checkboxInner: {
    width: 24,
    height: 24,
    borderRadius: 12,
    borderWidth: 2,
    borderColor: '#6200EE',
    alignItems: 'center',
    justifyContent: 'center',
  },
  checkboxChecked: {
    backgroundColor: '#6200EE',
  },
  checkmark: {
    color: '#fff',
    fontSize: 14,
    fontWeight: 'bold',
  },
  content: {
    flex: 1,
  },
  text: {
    fontSize: 16,
    color: '#333',
    marginBottom: 4,
  },
  completedText: {
    textDecorationLine: 'line-through',
    color: '#999',
  },
  editInput: {
    fontSize: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#6200EE',
    paddingVertical: 2,
    marginBottom: 4,
  },
  priorityContainer: {
    flexDirection: 'row',
    gap: 4,
  },
  priorityBadge: {
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 4,
    opacity: 0.5,
  },
  activePriority: {
    opacity: 1,
  },
  priorityText: {
    color: '#fff',
    fontSize: 10,
    fontWeight: 'bold',
  },
  deleteButton: {
    padding: 8,
  },
  deleteText: {
    fontSize: 18,
  },
});
```

### 4. AddTodoForm Component

```typescript
// src/components/AddTodoForm.tsx
import React, { useState } from 'react';
import {
  View,
  TextInput,
  TouchableOpacity,
  Text,
  StyleSheet,
} from 'react-native';
import { useAppDispatch } from '../store/hooks';
import { addTodo, Todo } from '../store/slices/todosSlice';

export const AddTodoForm: React.FC = () => {
  const dispatch = useAppDispatch();
  const [text, setText] = useState('');
  const [priority, setPriority] = useState<Todo['priority']>('medium');

  const handleAdd = () => {
    if (text.trim()) {
      dispatch(addTodo({ text: text.trim(), priority }));
      setText('');
    }
  };

  return (
    <View style={styles.container}>
      <TextInput
        value={text}
        onChangeText={setText}
        placeholder="เพิ่ม Todo ใหม่..."
        style={styles.input}
        onSubmitEditing={handleAdd}
        returnKeyType="done"
      />

      <View style={styles.prioritySelector}>
        {(['low', 'medium', 'high'] as Todo['priority'][]).map((p) => (
          <TouchableOpacity
            key={p}
            onPress={() => setPriority(p)}
            style={[
              styles.priorityButton,
              priority === p && styles.activePriorityButton,
            ]}
          >
            <Text
              style={[
                styles.priorityButtonText,
                priority === p && styles.activePriorityText,
              ]}
            >
              {p === 'low' ? 'ต่ำ' : p === 'medium' ? 'กลาง' : 'สูง'}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      <TouchableOpacity
        onPress={handleAdd}
        style={[styles.addButton, !text.trim() && styles.disabledButton]}
        disabled={!text.trim()}
      >
        <Text style={styles.addButtonText}>เพิ่ม</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    backgroundColor: '#fff',
    padding: 16,
    margin: 16,
    borderRadius: 12,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  input: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    marginBottom: 12,
  },
  prioritySelector: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 12,
    gap: 8,
  },
  priorityButton: {
    flex: 1,
    padding: 8,
    borderRadius: 6,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    alignItems: 'center',
  },
  activePriorityButton: {
    backgroundColor: '#6200EE',
    borderColor: '#6200EE',
  },
  priorityButtonText: {
    fontSize: 12,
    color: '#666',
  },
  activePriorityText: {
    color: '#fff',
    fontWeight: 'bold',
  },
  addButton: {
    backgroundColor: '#6200EE',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  disabledButton: {
    backgroundColor: '#E0E0E0',
  },
  addButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
  },
});
```

### 5. TodoFilters Component

```typescript
// src/components/TodoFilters.tsx
import React from 'react';
import { View, Text, TouchableOpacity, TextInput, StyleSheet } from 'react-native';
import { useAppDispatch, useAppSelector } from '../store/hooks';
import {
  setFilter,
  setSearchQuery,
  clearCompleted,
  selectFilter,
  selectTodoStats,
} from '../store/slices/todosSlice';

export const TodoFilters: React.FC = () => {
  const dispatch = useAppDispatch();
  const currentFilter = useAppSelector(selectFilter);
  const stats = useAppSelector(selectTodoStats);

  const filters = [
    { value: 'all', label: 'ทั้งหมด' },
    { value: 'active', label: 'ยังทำ' },
    { value: 'completed', label: 'เสร็จแล้ว' },
  ] as const;

  return (
    <View style={styles.container}>
      {/* Stats */}
      <View style={styles.stats}>
        <Text style={styles.statsText}>
          {stats.active} รายการที่ยังไม่เสร็จ / {stats.total} ทั้งหมด
        </Text>
      </View>

      {/* Search */}
      <TextInput
        placeholder="ค้นหา Todo..."
        onChangeText={(text) => dispatch(setSearchQuery(text))}
        style={styles.searchInput}
        clearButtonMode="while-editing"
      />

      {/* Filter buttons */}
      <View style={styles.filterRow}>
        {filters.map((f) => (
          <TouchableOpacity
            key={f.value}
            onPress={() => dispatch(setFilter(f.value))}
            style={[
              styles.filterButton,
              currentFilter === f.value && styles.activeFilter,
            ]}
          >
            <Text
              style={[
                styles.filterText,
                currentFilter === f.value && styles.activeFilterText,
              ]}
            >
              {f.label}
            </Text>
          </TouchableOpacity>
        ))}

        <TouchableOpacity
          onPress={() => dispatch(clearCompleted())}
          style={styles.clearButton}
        >
          <Text style={styles.clearButtonText}>ล้างเสร็จแล้ว</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    backgroundColor: '#fff',
    padding: 16,
    marginHorizontal: 16,
    marginBottom: 8,
    borderRadius: 12,
    elevation: 2,
  },
  stats: {
    marginBottom: 8,
  },
  statsText: {
    fontSize: 14,
    color: '#666',
  },
  searchInput: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 8,
    padding: 10,
    fontSize: 14,
    marginBottom: 12,
  },
  filterRow: {
    flexDirection: 'row',
    gap: 8,
    flexWrap: 'wrap',
  },
  filterButton: {
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: '#E0E0E0',
  },
  activeFilter: {
    backgroundColor: '#6200EE',
    borderColor: '#6200EE',
  },
  filterText: {
    fontSize: 12,
    color: '#666',
  },
  activeFilterText: {
    color: '#fff',
    fontWeight: 'bold',
  },
  clearButton: {
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: '#F44336',
  },
  clearButtonText: {
    fontSize: 12,
    color: '#F44336',
  },
});
```

### 6. TodoScreen - Main Screen

```typescript
// src/screens/TodoScreen.tsx
import React from 'react';
import {
  View,
  FlatList,
  Text,
  StyleSheet,
  SafeAreaView,
  StatusBar,
} from 'react-native';
import { useAppSelector } from '../store/hooks';
import { selectFilteredTodos } from '../store/slices/todosSlice';
import { TodoItem } from '../components/TodoItem';
import { AddTodoForm } from '../components/AddTodoForm';
import { TodoFilters } from '../components/TodoFilters';
import { Todo } from '../store/slices/todosSlice';

export const TodoScreen: React.FC = () => {
  const todos = useAppSelector(selectFilteredTodos);

  const renderItem = ({ item }: { item: Todo }) => (
    <TodoItem todo={item} />
  );

  const renderEmpty = () => (
    <View style={styles.emptyContainer}>
      <Text style={styles.emptyText}>ยังไม่มี Todo</Text>
      <Text style={styles.emptySubtext}>เพิ่ม Todo แรกของคุณด้านบน</Text>
    </View>
  );

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" backgroundColor="#f5f5f5" />

      <View style={styles.header}>
        <Text style={styles.title}>📋 Todo List</Text>
        <Text style={styles.subtitle}>จัดการงานของคุณด้วย Redux</Text>
      </View>

      <FlatList
        data={todos}
        keyExtractor={(item) => item.id}
        renderItem={renderItem}
        ListHeaderComponent={() => (
          <>
            <AddTodoForm />
            <TodoFilters />
          </>
        )}
        ListEmptyComponent={renderEmpty}
        contentContainerStyle={styles.listContent}
        showsVerticalScrollIndicator={false}
      />
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: '#6200EE',
    padding: 16,
    paddingTop: 20,
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#fff',
  },
  subtitle: {
    fontSize: 14,
    color: 'rgba(255,255,255,0.8)',
    marginTop: 4,
  },
  listContent: {
    paddingBottom: 20,
  },
  emptyContainer: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingTop: 60,
  },
  emptyText: {
    fontSize: 20,
    color: '#999',
    marginBottom: 8,
  },
  emptySubtext: {
    fontSize: 14,
    color: '#bbb',
  },
});
```

---

## Best Practices

### 1. ใช้ Selectors เสมอ

```typescript
// ❌ ไม่ดี - access state โดยตรงใน component
const todos = useAppSelector((state) => state.todos.items.filter(...));

// ✅ ดีกว่า - ใช้ selector ที่ defined ไว้
const todos = useAppSelector(selectFilteredTodos);
```

### 2. Memoized Selectors ด้วย createSelector

```typescript
import { createSelector } from '@reduxjs/toolkit';

// สร้าง memoized selector
export const selectHighPriorityTodos = createSelector(
  [selectAllTodos],
  (todos) => todos.filter((todo) => todo.priority === 'high')
);

// Selector ที่ซับซ้อนขึ้น
export const selectTodosByPriority = createSelector(
  [selectAllTodos, (state: RootState, priority: Todo['priority']) => priority],
  (todos, priority) => todos.filter((todo) => todo.priority === priority)
);
```

### 3. Normalize State สำหรับข้อมูลขนาดใหญ่

```typescript
import { createEntityAdapter, createSlice } from '@reduxjs/toolkit';

const todosAdapter = createEntityAdapter<Todo>({
  selectId: (todo) => todo.id,
  sortComparer: (a, b) => b.createdAt - a.createdAt,
});

const todosSlice = createSlice({
  name: 'todos',
  initialState: todosAdapter.getInitialState({
    filter: 'all' as const,
  }),
  reducers: {
    addTodo: todosAdapter.addOne,
    updateTodo: todosAdapter.updateOne,
    removeTodo: todosAdapter.removeOne,
    setAll: todosAdapter.setAll,
  },
});

// Selectors ที่ adapter สร้างให้
const todoSelectors = todosAdapter.getSelectors(
  (state: RootState) => state.todos
);
export const { selectAll: selectAllTodos, selectById: selectTodoById } =
  todoSelectors;
```

### 4. ใช้ Prepare Callback สำหรับ Complex Actions

```typescript
const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    addTodo: {
      reducer(state, action: PayloadAction<Todo>) {
        state.items.push(action.payload);
      },
      // prepare ใช้สร้าง payload ก่อน reducer รับ
      prepare(text: string, priority: Todo['priority']) {
        return {
          payload: {
            id: Date.now().toString(),
            text,
            completed: false,
            priority,
            createdAt: Date.now(),
          } as Todo,
        };
      },
    },
  },
});

// ใช้งาน
dispatch(addTodo('ซื้อของ', 'high'));
```

---

## Tips และ Tricks

### Performance Optimization

```typescript
// ใช้ shallowEqual เพื่อลด re-renders
import { shallowEqual } from 'react-redux';

// แทนที่จะ
const { filter, searchQuery } = useAppSelector((state) => ({
  filter: state.todos.filter,
  searchQuery: state.todos.searchQuery,
}));

// ใช้ shallowEqual
const { filter, searchQuery } = useAppSelector(
  (state) => ({
    filter: state.todos.filter,
    searchQuery: state.todos.searchQuery,
  }),
  shallowEqual
);
```

### Redux Persist (บันทึก State ลง Storage)

```bash
npm install redux-persist @react-native-async-storage/async-storage
```

```typescript
import { persistStore, persistReducer } from 'redux-persist';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { PersistGate } from 'redux-persist/integration/react';

const persistConfig = {
  key: 'root',
  storage: AsyncStorage,
  whitelist: ['todos'], // เลือก reducer ที่จะ persist
};

const persistedTodosReducer = persistReducer(persistConfig, todosReducer);

export const store = configureStore({
  reducer: {
    todos: persistedTodosReducer,
  },
});

export const persistor = persistStore(store);

// ใน App.tsx
<Provider store={store}>
  <PersistGate loading={null} persistor={persistor}>
    <App />
  </PersistGate>
</Provider>
```

---

## สรุป

Redux Toolkit ช่วยให้การจัดการ state ใน React Native app ง่ายขึ้นมาก:

1. **createSlice** - รวม actions และ reducer ไว้ด้วยกัน
2. **configureStore** - ตั้งค่า store พร้อม middleware ที่ดีไว้ให้
3. **useSelector** - อ่าน state อย่างมีประสิทธิภาพด้วย selectors
4. **useDispatch** - ส่ง actions เพื่อเปลี่ยน state
5. **Immer** built-in ทำให้ mutate state ได้โดยตรง
6. **TypeScript** support ดีมากเมื่อใช้ typed hooks

ใน Part 032 เราจะเรียนเรื่อง **Redux Thunk และ Async Actions** สำหรับการ fetch ข้อมูลจาก API
