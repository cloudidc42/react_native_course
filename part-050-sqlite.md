# Part 050: SQLite และ Local Database ใน React Native

## สารบัญ
1. [แนะนำ Local Database](#introduction)
2. [react-native-sqlite-storage](#sqlite-storage)
3. [WatermelonDB](#watermelondb)
4. [Table Creation](#table-creation)
5. [CRUD Operations](#crud)
6. [Migrations](#migrations)
7. [Workshop: Offline-capable Todo App](#workshop)

---

## 1. แนะนำ Local Database {#introduction}

Local Database ช่วยให้แอป React Native สามารถจัดเก็บข้อมูลจำนวนมากได้อย่างมีประสิทธิภาพ และทำงานได้แม้ไม่มีอินเทอร์เน็ต

### เปรียบเทียบ Local Storage Options

| ตัวเลือก | ข้อดี | ข้อเสีย | เหมาะกับ |
|---------|-------|---------|---------|
| AsyncStorage | ง่าย | ช้าสำหรับข้อมูลมาก | Key-value pairs |
| MMKV | เร็วมาก | ไม่รองรับ queries | Settings, Cache |
| SQLite | Relational, Query | ต้องเรียนรู้ SQL | Structured data |
| WatermelonDB | Performance สูง, Reactive | Setup ซับซ้อน | Large data sets |
| Realm | เร็ว, Schema | มี learning curve | Complex relations |

### ติดตั้ง SQLite

```bash
# react-native-sqlite-storage
npm install react-native-sqlite-storage
cd ios && pod install

# สำหรับ Expo
npx expo install expo-sqlite

# WatermelonDB
npm install @nozbe/watermelondb
npm install @babel/plugin-proposal-decorators
npm install @babel/plugin-transform-class-static-block
```

---

## 2. react-native-sqlite-storage {#sqlite-storage}

### การตั้งค่าและเชื่อมต่อ

```typescript
// src/database/SQLiteDatabase.ts
import SQLite, { SQLiteDatabase, ResultSet } from 'react-native-sqlite-storage';

SQLite.enablePromise(true);
SQLite.DEBUG(false); // ปิดใน production

class Database {
  private static instance: Database;
  private db: SQLiteDatabase | null = null;
  private readonly DB_NAME = 'myapp.db';
  private readonly DB_VERSION = '1.0';

  static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }

  async open(): Promise<SQLiteDatabase> {
    if (this.db) return this.db;

    this.db = await SQLite.openDatabase({
      name: this.DB_NAME,
      location: 'default',
      createFromLocation: undefined,
    });

    await this.initialize();
    return this.db;
  }

  async close(): Promise<void> {
    if (this.db) {
      await this.db.close();
      this.db = null;
    }
  }

  private async initialize(): Promise<void> {
    await this.createTables();
    await this.runMigrations();
  }

  private async createTables(): Promise<void> {
    if (!this.db) throw new Error('Database not open');

    // สร้าง tables พื้นฐาน
    await this.db.executeSql(`
      CREATE TABLE IF NOT EXISTS app_meta (
        key TEXT PRIMARY KEY,
        value TEXT NOT NULL
      )
    `);

    // ตรวจสอบ version
    const version = await this.getMetaValue('schema_version');
    if (!version) {
      await this.setMetaValue('schema_version', '1');
    }
  }

  async executeSql(sql: string, params: any[] = []): Promise<ResultSet> {
    if (!this.db) await this.open();
    const [result] = await this.db!.executeSql(sql, params);
    return result;
  }

  async transaction(operations: (tx: any) => Promise<void>): Promise<void> {
    if (!this.db) await this.open();
    await this.db!.transaction(operations);
  }

  private async getMetaValue(key: string): Promise<string | null> {
    const result = await this.executeSql(
      'SELECT value FROM app_meta WHERE key = ?',
      [key]
    );
    return result.rows.length > 0 ? result.rows.item(0).value : null;
  }

  private async setMetaValue(key: string, value: string): Promise<void> {
    await this.executeSql(
      'INSERT OR REPLACE INTO app_meta (key, value) VALUES (?, ?)',
      [key, value]
    );
  }

  private async runMigrations(): Promise<void> {
    const currentVersion = parseInt(
      (await this.getMetaValue('schema_version')) || '0'
    );

    // Migrations ตามลำดับ version
    const migrations = [
      this.migration_1, // Version 1
      this.migration_2, // Version 2
    ];

    for (let i = currentVersion; i < migrations.length; i++) {
      console.log(`Running migration ${i + 1}`);
      await migrations[i](this);
      await this.setMetaValue('schema_version', String(i + 1));
    }
  }

  private async migration_1(db: Database): Promise<void> {
    await db.executeSql(`
      CREATE TABLE IF NOT EXISTS todos (
        id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        description TEXT DEFAULT '',
        completed INTEGER DEFAULT 0,
        priority INTEGER DEFAULT 0,
        due_date TEXT,
        created_at TEXT NOT NULL,
        updated_at TEXT NOT NULL,
        synced INTEGER DEFAULT 0
      )
    `);

    await db.executeSql(`
      CREATE TABLE IF NOT EXISTS categories (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        color TEXT NOT NULL,
        icon TEXT NOT NULL,
        created_at TEXT NOT NULL
      )
    `);

    await db.executeSql(`
      CREATE TABLE IF NOT EXISTS todo_categories (
        todo_id TEXT NOT NULL,
        category_id TEXT NOT NULL,
        PRIMARY KEY (todo_id, category_id),
        FOREIGN KEY (todo_id) REFERENCES todos(id) ON DELETE CASCADE,
        FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
      )
    `);

    // Indexes
    await db.executeSql('CREATE INDEX IF NOT EXISTS idx_todos_completed ON todos(completed)');
    await db.executeSql('CREATE INDEX IF NOT EXISTS idx_todos_due_date ON todos(due_date)');
    await db.executeSql('CREATE INDEX IF NOT EXISTS idx_todos_synced ON todos(synced)');
  }

  private async migration_2(db: Database): Promise<void> {
    // เพิ่ม column tags
    await db.executeSql('ALTER TABLE todos ADD COLUMN tags TEXT DEFAULT ""');
    await db.executeSql('ALTER TABLE todos ADD COLUMN notes TEXT DEFAULT ""');
  }
}

export default Database.getInstance();
```

---

## 3. WatermelonDB {#watermelondb}

### การตั้งค่า WatermelonDB

```typescript
// src/database/watermelon/schema.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'todos',
      columns: [
        { name: 'title', type: 'string' },
        { name: 'description', type: 'string', isOptional: true },
        { name: 'is_completed', type: 'boolean' },
        { name: 'priority', type: 'number' },
        { name: 'due_date', type: 'number', isOptional: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
        { name: 'is_synced', type: 'boolean' },
        { name: 'category_id', type: 'string', isOptional: true },
        { name: 'tags', type: 'string' }, // JSON array
      ],
    }),
    tableSchema({
      name: 'categories',
      columns: [
        { name: 'name', type: 'string' },
        { name: 'color', type: 'string' },
        { name: 'icon', type: 'string' },
        { name: 'created_at', type: 'number' },
      ],
    }),
  ],
});
```

```typescript
// src/database/watermelon/models/Todo.ts
import {
  Model,
  field,
  date,
  readonly,
  relation,
  children,
  json,
} from '@nozbe/watermelondb';
import { Associations } from '@nozbe/watermelondb/Model';

export default class Todo extends Model {
  static table = 'todos';
  static associations: Associations = {
    categories: { type: 'belongs_to', key: 'category_id' },
  };

  @field('title') title!: string;
  @field('description') description!: string;
  @field('is_completed') isCompleted!: boolean;
  @field('priority') priority!: number;
  @date('due_date') dueDate!: Date | null;
  @readonly @date('created_at') createdAt!: Date;
  @readonly @date('updated_at') updatedAt!: Date;
  @field('is_synced') isSynced!: boolean;
  @field('category_id') categoryId!: string;
  @field('tags') tagsRaw!: string;

  get tags(): string[] {
    try {
      return JSON.parse(this.tagsRaw || '[]');
    } catch {
      return [];
    }
  }
}
```

```typescript
// src/database/watermelon/index.ts
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';
import { schema } from './schema';
import migrations from './migrations';
import Todo from './models/Todo';
import Category from './models/Category';

const adapter = new SQLiteAdapter({
  schema,
  migrations,
  jsi: true, // ใช้ JSI สำหรับ performance ที่ดีขึ้น
  onSetUpError: (error) => {
    console.error('WatermelonDB setup error:', error);
  },
});

const database = new Database({
  adapter,
  modelClasses: [Todo, Category],
});

export default database;
```

---

## 4. Table Creation {#table-creation}

### SQL Schema Design

```typescript
// src/database/migrations/v1.ts
export const createTables = async (db: any): Promise<void> => {
  // Users table
  await db.executeSql(`
    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY,
      email TEXT UNIQUE NOT NULL,
      username TEXT NOT NULL,
      avatar_url TEXT,
      created_at INTEGER NOT NULL,
      updated_at INTEGER NOT NULL
    )
  `);

  // Todos table
  await db.executeSql(`
    CREATE TABLE IF NOT EXISTS todos (
      id TEXT PRIMARY KEY,
      user_id TEXT NOT NULL,
      title TEXT NOT NULL,
      description TEXT DEFAULT '',
      completed INTEGER DEFAULT 0 CHECK(completed IN (0, 1)),
      priority INTEGER DEFAULT 0 CHECK(priority BETWEEN 0 AND 3),
      due_date INTEGER,
      reminder_time INTEGER,
      repeat_type TEXT CHECK(repeat_type IN ('none', 'daily', 'weekly', 'monthly')),
      tags TEXT DEFAULT '[]',
      notes TEXT DEFAULT '',
      created_at INTEGER NOT NULL,
      updated_at INTEGER NOT NULL,
      deleted_at INTEGER,
      synced INTEGER DEFAULT 0,
      FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
    )
  `);

  // Indexes
  await db.executeSql(`
    CREATE INDEX IF NOT EXISTS idx_todos_user ON todos(user_id)
  `);
  await db.executeSql(`
    CREATE INDEX IF NOT EXISTS idx_todos_completed ON todos(completed)
  `);
  await db.executeSql(`
    CREATE INDEX IF NOT EXISTS idx_todos_due ON todos(due_date)
  `);
  await db.executeSql(`
    CREATE INDEX IF NOT EXISTS idx_todos_deleted ON todos(deleted_at)
  `);

  // Full-text search
  await db.executeSql(`
    CREATE VIRTUAL TABLE IF NOT EXISTS todos_fts
    USING fts5(title, description, notes, content=todos, content_rowid=rowid)
  `);
};
```

---

## 5. CRUD Operations {#crud}

### Todo Repository

```typescript
// src/repositories/TodoRepository.ts
import { v4 as uuidv4 } from 'uuid';
import Database from '../database/SQLiteDatabase';

interface Todo {
  id: string;
  userId: string;
  title: string;
  description: string;
  completed: boolean;
  priority: number;
  dueDate?: Date;
  tags: string[];
  notes: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
  synced: boolean;
}

interface TodoFilter {
  completed?: boolean;
  priority?: number;
  search?: string;
  tags?: string[];
  dueBefore?: Date;
  dueAfter?: Date;
  includeDeleted?: boolean;
  limit?: number;
  offset?: number;
}

class TodoRepository {
  // CREATE
  static async create(data: Omit<Todo, 'id' | 'createdAt' | 'updatedAt' | 'synced'>): Promise<Todo> {
    const now = new Date();
    const todo: Todo = {
      ...data,
      id: uuidv4(),
      createdAt: now,
      updatedAt: now,
      synced: false,
    };

    await Database.executeSql(
      `INSERT INTO todos (
        id, user_id, title, description, completed, priority,
        due_date, tags, notes, created_at, updated_at, synced
      ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
      [
        todo.id,
        todo.userId,
        todo.title,
        todo.description,
        todo.completed ? 1 : 0,
        todo.priority,
        todo.dueDate?.getTime() || null,
        JSON.stringify(todo.tags),
        todo.notes,
        todo.createdAt.getTime(),
        todo.updatedAt.getTime(),
        0,
      ]
    );

    return todo;
  }

  // READ
  static async findById(id: string): Promise<Todo | null> {
    const result = await Database.executeSql(
      'SELECT * FROM todos WHERE id = ? AND deleted_at IS NULL',
      [id]
    );

    if (result.rows.length === 0) return null;
    return this.mapRowToTodo(result.rows.item(0));
  }

  static async findAll(filter: TodoFilter = {}): Promise<{ todos: Todo[]; total: number }> {
    let where = ['deleted_at IS NULL'];
    const params: any[] = [];

    if (filter.includeDeleted !== true) {
      // already included above
    }

    if (filter.completed !== undefined) {
      where.push('completed = ?');
      params.push(filter.completed ? 1 : 0);
    }

    if (filter.priority !== undefined) {
      where.push('priority = ?');
      params.push(filter.priority);
    }

    if (filter.dueBefore) {
      where.push('due_date <= ?');
      params.push(filter.dueBefore.getTime());
    }

    if (filter.dueAfter) {
      where.push('due_date >= ?');
      params.push(filter.dueAfter.getTime());
    }

    if (filter.search) {
      where.push('(title LIKE ? OR description LIKE ? OR notes LIKE ?)');
      const searchParam = `%${filter.search}%`;
      params.push(searchParam, searchParam, searchParam);
    }

    const whereClause = where.length > 0 ? `WHERE ${where.join(' AND ')}` : '';

    // Count total
    const countResult = await Database.executeSql(
      `SELECT COUNT(*) as total FROM todos ${whereClause}`,
      params
    );
    const total = countResult.rows.item(0).total;

    // Get paginated results
    const limit = filter.limit || 20;
    const offset = filter.offset || 0;

    const result = await Database.executeSql(
      `SELECT * FROM todos ${whereClause}
       ORDER BY completed ASC, priority DESC, due_date ASC, created_at DESC
       LIMIT ? OFFSET ?`,
      [...params, limit, offset]
    );

    const todos = Array.from({ length: result.rows.length }, (_, i) =>
      this.mapRowToTodo(result.rows.item(i))
    );

    return { todos, total };
  }

  // UPDATE
  static async update(id: string, data: Partial<Omit<Todo, 'id' | 'createdAt'>>): Promise<Todo | null> {
    const existing = await this.findById(id);
    if (!existing) return null;

    const updatedAt = new Date();
    const updates: string[] = ['updated_at = ?', 'synced = 0'];
    const params: any[] = [updatedAt.getTime()];

    if (data.title !== undefined) { updates.push('title = ?'); params.push(data.title); }
    if (data.description !== undefined) { updates.push('description = ?'); params.push(data.description); }
    if (data.completed !== undefined) { updates.push('completed = ?'); params.push(data.completed ? 1 : 0); }
    if (data.priority !== undefined) { updates.push('priority = ?'); params.push(data.priority); }
    if (data.dueDate !== undefined) { updates.push('due_date = ?'); params.push(data.dueDate?.getTime() || null); }
    if (data.tags !== undefined) { updates.push('tags = ?'); params.push(JSON.stringify(data.tags)); }
    if (data.notes !== undefined) { updates.push('notes = ?'); params.push(data.notes); }

    params.push(id);

    await Database.executeSql(
      `UPDATE todos SET ${updates.join(', ')} WHERE id = ?`,
      params
    );

    return { ...existing, ...data, updatedAt, synced: false };
  }

  // DELETE (Soft delete)
  static async delete(id: string): Promise<boolean> {
    const result = await Database.executeSql(
      'UPDATE todos SET deleted_at = ?, synced = 0 WHERE id = ? AND deleted_at IS NULL',
      [new Date().getTime(), id]
    );
    return result.rowsAffected > 0;
  }

  // Hard delete
  static async hardDelete(id: string): Promise<boolean> {
    const result = await Database.executeSql(
      'DELETE FROM todos WHERE id = ?',
      [id]
    );
    return result.rowsAffected > 0;
  }

  // Batch operations
  static async batchCreate(todos: Omit<Todo, 'id' | 'createdAt' | 'updatedAt' | 'synced'>[]): Promise<void> {
    await Database.transaction(async (tx) => {
      for (const data of todos) {
        const now = Date.now();
        const id = uuidv4();

        tx.executeSql(
          `INSERT INTO todos (id, user_id, title, description, completed, priority, due_date, tags, notes, created_at, updated_at, synced)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
          [id, data.userId, data.title, data.description, data.completed ? 1 : 0, data.priority, data.dueDate?.getTime() || null, JSON.stringify(data.tags), data.notes, now, now, 0]
        );
      }
    });
  }

  // Statistics
  static async getStats(userId: string) {
    const result = await Database.executeSql(
      `SELECT
        COUNT(*) as total,
        SUM(CASE WHEN completed = 1 THEN 1 ELSE 0 END) as completed,
        SUM(CASE WHEN completed = 0 AND due_date < ? THEN 1 ELSE 0 END) as overdue,
        SUM(CASE WHEN completed = 0 AND due_date BETWEEN ? AND ? THEN 1 ELSE 0 END) as due_today
       FROM todos
       WHERE user_id = ? AND deleted_at IS NULL`,
      [
        Date.now(),
        new Date().setHours(0, 0, 0, 0),
        new Date().setHours(23, 59, 59, 999),
        userId,
      ]
    );

    const row = result.rows.item(0);
    return {
      total: row.total || 0,
      completed: row.completed || 0,
      pending: (row.total || 0) - (row.completed || 0),
      overdue: row.overdue || 0,
      dueToday: row.due_today || 0,
    };
  }

  private static mapRowToTodo(row: any): Todo {
    return {
      id: row.id,
      userId: row.user_id,
      title: row.title,
      description: row.description || '',
      completed: row.completed === 1,
      priority: row.priority || 0,
      dueDate: row.due_date ? new Date(row.due_date) : undefined,
      tags: JSON.parse(row.tags || '[]'),
      notes: row.notes || '',
      createdAt: new Date(row.created_at),
      updatedAt: new Date(row.updated_at),
      deletedAt: row.deleted_at ? new Date(row.deleted_at) : undefined,
      synced: row.synced === 1,
    };
  }
}

export default TodoRepository;
```

---

## 6. Migrations {#migrations}

```typescript
// src/database/migrations/index.ts
import Database from '../SQLiteDatabase';

interface Migration {
  version: number;
  up: () => Promise<void>;
  down?: () => Promise<void>;
}

const migrations: Migration[] = [
  {
    version: 1,
    up: async () => {
      await Database.executeSql(`
        CREATE TABLE IF NOT EXISTS todos (
          id TEXT PRIMARY KEY,
          title TEXT NOT NULL,
          completed INTEGER DEFAULT 0,
          created_at INTEGER NOT NULL
        )
      `);
    },
  },
  {
    version: 2,
    up: async () => {
      // เพิ่ม columns ใน v2
      await Database.executeSql('ALTER TABLE todos ADD COLUMN description TEXT DEFAULT ""');
      await Database.executeSql('ALTER TABLE todos ADD COLUMN priority INTEGER DEFAULT 0');
      await Database.executeSql('ALTER TABLE todos ADD COLUMN updated_at INTEGER');

      // อัพเดต existing rows
      await Database.executeSql('UPDATE todos SET updated_at = created_at WHERE updated_at IS NULL');
    },
  },
  {
    version: 3,
    up: async () => {
      // สร้าง table ใหม่
      await Database.executeSql(`
        CREATE TABLE IF NOT EXISTS categories (
          id TEXT PRIMARY KEY,
          name TEXT NOT NULL,
          color TEXT NOT NULL,
          icon TEXT DEFAULT 'folder'
        )
      `);

      // เพิ่ม category reference
      await Database.executeSql('ALTER TABLE todos ADD COLUMN category_id TEXT REFERENCES categories(id)');
    },
  },
];

export async function runMigrations(): Promise<void> {
  await Database.open();

  // สร้าง migrations table
  await Database.executeSql(`
    CREATE TABLE IF NOT EXISTS migrations (
      version INTEGER PRIMARY KEY,
      applied_at INTEGER NOT NULL
    )
  `);

  // ดึง version ล่าสุดที่ run แล้ว
  const result = await Database.executeSql(
    'SELECT MAX(version) as latest FROM migrations'
  );
  const currentVersion = result.rows.item(0).latest || 0;

  // Run migrations ที่ยังไม่ได้ run
  for (const migration of migrations) {
    if (migration.version > currentVersion) {
      console.log(`Running migration v${migration.version}...`);
      
      try {
        await migration.up();
        await Database.executeSql(
          'INSERT INTO migrations (version, applied_at) VALUES (?, ?)',
          [migration.version, Date.now()]
        );
        console.log(`Migration v${migration.version} completed`);
      } catch (error) {
        console.error(`Migration v${migration.version} failed:`, error);
        throw error;
      }
    }
  }
}
```

---

## 7. Workshop: Offline-capable Todo App {#workshop}

```typescript
// src/screens/TodoScreen.tsx
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Switch,
  Modal,
  KeyboardAvoidingView,
  Platform,
  ActivityIndicator,
} from 'react-native';
import Icon from 'react-native-vector-icons/MaterialIcons';
import NetInfo from '@react-native-community/netinfo';
import TodoRepository from '../repositories/TodoRepository';
import { runMigrations } from '../database/migrations';
import { useAuth } from '../hooks/useAuth';

interface TodoItem {
  id: string;
  title: string;
  description: string;
  completed: boolean;
  priority: number;
  dueDate?: Date;
  tags: string[];
  synced: boolean;
}

const PRIORITIES = [
  { label: 'ต่ำ', value: 0, color: '#9E9E9E', icon: 'arrow-downward' },
  { label: 'ปานกลาง', value: 1, color: '#FF9800', icon: 'remove' },
  { label: 'สูง', value: 2, color: '#F44336', icon: 'arrow-upward' },
  { label: 'เร่งด่วน', value: 3, color: '#B71C1C', icon: 'priority-high' },
];

const TodoScreen: React.FC = () => {
  const { user } = useAuth();
  const [todos, setTodos] = useState<TodoItem[]>([]);
  const [loading, setLoading] = useState(true);
  const [isOnline, setIsOnline] = useState(true);
  const [showAddModal, setShowAddModal] = useState(false);
  const [editingTodo, setEditingTodo] = useState<TodoItem | null>(null);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  const [searchQuery, setSearchQuery] = useState('');
  const [stats, setStats] = useState({ total: 0, completed: 0, pending: 0, overdue: 0 });
  const [syncStatus, setSyncStatus] = useState<'synced' | 'pending' | 'syncing'>('synced');

  // New todo form
  const [newTitle, setNewTitle] = useState('');
  const [newDescription, setNewDescription] = useState('');
  const [newPriority, setNewPriority] = useState(0);
  const [newTags, setNewTags] = useState('');

  useEffect(() => {
    initDB();
    checkNetworkStatus();
  }, []);

  const initDB = async () => {
    try {
      await runMigrations();
      await loadTodos();
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถเปิดฐานข้อมูลได้');
    }
  };

  const checkNetworkStatus = () => {
    const unsubscribe = NetInfo.addEventListener(state => {
      const online = state.isConnected ?? false;
      setIsOnline(online);
      if (online) {
        syncWithServer();
      }
    });

    return () => unsubscribe();
  };

  const loadTodos = async () => {
    setLoading(true);
    try {
      const filterOptions: any = {
        search: searchQuery || undefined,
      };

      if (filter === 'active') filterOptions.completed = false;
      if (filter === 'completed') filterOptions.completed = true;

      const { todos: loadedTodos } = await TodoRepository.findAll(filterOptions);
      setTodos(loadedTodos);

      const newStats = await TodoRepository.getStats(user.id);
      setStats(newStats);

      const unsyncedCount = loadedTodos.filter(t => !t.synced).length;
      setSyncStatus(unsyncedCount > 0 ? 'pending' : 'synced');
    } catch (error) {
      console.error('Load todos error:', error);
    }
    setLoading(false);
  };

  useEffect(() => {
    loadTodos();
  }, [filter, searchQuery]);

  const addTodo = async () => {
    if (!newTitle.trim()) return;

    const tags = newTags
      .split(',')
      .map(t => t.trim())
      .filter(Boolean);

    try {
      const todo = await TodoRepository.create({
        userId: user.id,
        title: newTitle.trim(),
        description: newDescription.trim(),
        completed: false,
        priority: newPriority,
        tags,
        notes: '',
      });

      setTodos(prev => [todo, ...prev]);
      setStats(prev => ({ ...prev, total: prev.total + 1, pending: prev.pending + 1 }));

      setShowAddModal(false);
      resetForm();

      if (isOnline) {
        syncWithServer();
      }
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถเพิ่ม Todo ได้');
    }
  };

  const toggleTodo = async (id: string, completed: boolean) => {
    try {
      await TodoRepository.update(id, { completed });
      setTodos(prev =>
        prev.map(t => t.id === id ? { ...t, completed, synced: false } : t)
      );
      setStats(prev => ({
        ...prev,
        completed: completed ? prev.completed + 1 : prev.completed - 1,
        pending: completed ? prev.pending - 1 : prev.pending + 1,
      }));
      setSyncStatus('pending');

      if (isOnline) syncWithServer();
    } catch (error) {
      console.error('Toggle error:', error);
    }
  };

  const deleteTodo = async (id: string) => {
    Alert.alert(
      'ลบ Todo',
      'ต้องการลบรายการนี้หรือไม่?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ลบ',
          style: 'destructive',
          onPress: async () => {
            await TodoRepository.delete(id);
            setTodos(prev => prev.filter(t => t.id !== id));
            await loadTodos();
          },
        },
      ]
    );
  };

  const syncWithServer = async () => {
    if (!isOnline) return;
    setSyncStatus('syncing');

    try {
      // ดึง todos ที่ยังไม่ sync
      const { todos: unsyncedTodos } = await TodoRepository.findAll();
      const pendingSync = unsyncedTodos.filter(t => !t.synced);

      if (pendingSync.length === 0) {
        setSyncStatus('synced');
        return;
      }

      // ส่งไป server
      const response = await fetch('https://api.example.com/todos/sync', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${user.token}`,
        },
        body: JSON.stringify({ todos: pendingSync }),
      });

      if (response.ok) {
        // Mark as synced
        await Promise.all(
          pendingSync.map(t => TodoRepository.update(t.id, { synced: true } as any))
        );
        setTodos(prev => prev.map(t =>
          pendingSync.find(s => s.id === t.id) ? { ...t, synced: true } : t
        ));
        setSyncStatus('synced');
      }
    } catch (error) {
      console.error('Sync error:', error);
      setSyncStatus('pending');
    }
  };

  const resetForm = () => {
    setNewTitle('');
    setNewDescription('');
    setNewPriority(0);
    setNewTags('');
  };

  const getPriorityInfo = (priority: number) => {
    return PRIORITIES[priority] || PRIORITIES[0];
  };

  const renderTodo = ({ item }: { item: TodoItem }) => {
    const priorityInfo = getPriorityInfo(item.priority);

    return (
      <TouchableOpacity
        style={[
          styles.todoItem,
          item.completed && styles.completedItem,
        ]}
        onLongPress={() => {
          Alert.alert(item.title, '', [
            { text: 'ยกเลิก', style: 'cancel' },
            {
              text: 'ลบ',
              style: 'destructive',
              onPress: () => deleteTodo(item.id),
            },
          ]);
        }}
      >
        <TouchableOpacity
          style={[styles.checkbox, item.completed && styles.checkboxChecked]}
          onPress={() => toggleTodo(item.id, !item.completed)}
        >
          {item.completed && <Icon name="check" size={16} color="white" />}
        </TouchableOpacity>

        <View style={styles.todoContent}>
          <View style={styles.todoHeader}>
            <Text style={[
              styles.todoTitle,
              item.completed && styles.completedTitle,
            ]}>
              {item.title}
            </Text>
            <View style={[styles.priorityBadge, { backgroundColor: priorityInfo.color + '20' }]}>
              <Icon name={priorityInfo.icon} size={12} color={priorityInfo.color} />
            </View>
          </View>

          {item.description ? (
            <Text style={styles.todoDesc} numberOfLines={1}>{item.description}</Text>
          ) : null}

          {item.tags.length > 0 && (
            <View style={styles.tagRow}>
              {item.tags.slice(0, 3).map(tag => (
                <View key={tag} style={styles.tag}>
                  <Text style={styles.tagText}>#{tag}</Text>
                </View>
              ))}
            </View>
          )}
        </View>

        {!item.synced && (
          <Icon name="cloud-off" size={16} color="#9E9E9E" style={styles.syncIcon} />
        )}
      </TouchableOpacity>
    );
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      {/* Offline/Sync Banner */}
      {!isOnline && (
        <View style={styles.offlineBanner}>
          <Icon name="wifi-off" size={16} color="white" />
          <Text style={styles.offlineText}>ออฟไลน์ - การเปลี่ยนแปลงจะ sync เมื่อมีอินเทอร์เน็ต</Text>
        </View>
      )}

      {syncStatus === 'syncing' && isOnline && (
        <View style={styles.syncingBanner}>
          <ActivityIndicator size="small" color="white" />
          <Text style={styles.syncingText}>กำลัง sync...</Text>
        </View>
      )}

      {/* Stats */}
      <View style={styles.statsContainer}>
        <View style={styles.statCard}>
          <Text style={styles.statNumber}>{stats.total}</Text>
          <Text style={styles.statLabel}>ทั้งหมด</Text>
        </View>
        <View style={styles.statCard}>
          <Text style={[styles.statNumber, { color: '#FF9800' }]}>{stats.pending}</Text>
          <Text style={styles.statLabel}>รอดำเนินการ</Text>
        </View>
        <View style={styles.statCard}>
          <Text style={[styles.statNumber, { color: '#4CAF50' }]}>{stats.completed}</Text>
          <Text style={styles.statLabel}>เสร็จแล้ว</Text>
        </View>
        {stats.overdue > 0 && (
          <View style={styles.statCard}>
            <Text style={[styles.statNumber, { color: '#F44336' }]}>{stats.overdue}</Text>
            <Text style={styles.statLabel}>เกินกำหนด</Text>
          </View>
        )}
      </View>

      {/* Filter Tabs */}
      <View style={styles.filterTabs}>
        {(['all', 'active', 'completed'] as const).map(f => (
          <TouchableOpacity
            key={f}
            style={[styles.filterTab, filter === f && styles.activeFilterTab]}
            onPress={() => setFilter(f)}
          >
            <Text style={[styles.filterTabText, filter === f && styles.activeFilterTabText]}>
              {f === 'all' ? 'ทั้งหมด' : f === 'active' ? 'ที่รอ' : 'เสร็จแล้ว'}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Search */}
      <View style={styles.searchContainer}>
        <Icon name="search" size={20} color="#9E9E9E" />
        <TextInput
          style={styles.searchInput}
          placeholder="ค้นหา..."
          value={searchQuery}
          onChangeText={setSearchQuery}
        />
      </View>

      {/* Todo List */}
      {loading ? (
        <ActivityIndicator style={{ flex: 1 }} />
      ) : (
        <FlatList
          data={todos}
          renderItem={renderTodo}
          keyExtractor={item => item.id}
          contentContainerStyle={styles.list}
          ListEmptyComponent={
            <View style={styles.emptyState}>
              <Text style={styles.emptyIcon}>✅</Text>
              <Text style={styles.emptyText}>
                {filter === 'completed' ? 'ยังไม่มีรายการที่เสร็จ' :
                 filter === 'active' ? 'ไม่มีรายการที่รอดำเนินการ' :
                 'ยังไม่มีรายการ กดปุ่ม + เพื่อเพิ่ม'}
              </Text>
            </View>
          }
        />
      )}

      {/* FAB */}
      <TouchableOpacity
        style={styles.fab}
        onPress={() => setShowAddModal(true)}
      >
        <Icon name="add" size={28} color="white" />
      </TouchableOpacity>

      {/* Add Modal */}
      <Modal
        visible={showAddModal}
        animationType="slide"
        presentationStyle="pageSheet"
        onRequestClose={() => {
          setShowAddModal(false);
          resetForm();
        }}
      >
        <View style={styles.modal}>
          <View style={styles.modalHeader}>
            <TouchableOpacity onPress={() => { setShowAddModal(false); resetForm(); }}>
              <Text style={styles.cancelText}>ยกเลิก</Text>
            </TouchableOpacity>
            <Text style={styles.modalTitle}>เพิ่ม Todo</Text>
            <TouchableOpacity onPress={addTodo}>
              <Text style={[styles.saveText, !newTitle.trim() && styles.disabledText]}>
                บันทึก
              </Text>
            </TouchableOpacity>
          </View>

          <View style={styles.form}>
            <TextInput
              style={styles.titleInput}
              placeholder="ชื่อรายการ *"
              value={newTitle}
              onChangeText={setNewTitle}
              autoFocus
            />

            <TextInput
              style={styles.descInput}
              placeholder="รายละเอียด (ไม่บังคับ)"
              value={newDescription}
              onChangeText={setNewDescription}
              multiline
            />

            <Text style={styles.formLabel}>ระดับความสำคัญ:</Text>
            <View style={styles.priorityRow}>
              {PRIORITIES.map(p => (
                <TouchableOpacity
                  key={p.value}
                  style={[
                    styles.priorityOption,
                    { borderColor: p.color },
                    newPriority === p.value && { backgroundColor: p.color },
                  ]}
                  onPress={() => setNewPriority(p.value)}
                >
                  <Text style={[
                    styles.priorityLabel,
                    { color: newPriority === p.value ? 'white' : p.color },
                  ]}>
                    {p.label}
                  </Text>
                </TouchableOpacity>
              ))}
            </View>

            <TextInput
              style={styles.tagsInput}
              placeholder="แท็ก (คั่นด้วยจุลภาค เช่น งาน, ด่วน)"
              value={newTags}
              onChangeText={setNewTags}
            />
          </View>
        </View>
      </Modal>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  offlineBanner: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#FF5722',
    padding: 8,
    paddingHorizontal: 16,
    gap: 8,
  },
  offlineText: { color: 'white', flex: 1, fontSize: 12 },
  syncingBanner: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#2196F3',
    padding: 6,
    paddingHorizontal: 16,
    gap: 8,
  },
  syncingText: { color: 'white', fontSize: 12 },
  statsContainer: {
    flexDirection: 'row',
    backgroundColor: 'white',
    padding: 16,
    gap: 8,
  },
  statCard: { flex: 1, alignItems: 'center' },
  statNumber: { fontSize: 22, fontWeight: 'bold', color: '#333' },
  statLabel: { fontSize: 11, color: '#9E9E9E', marginTop: 2 },
  filterTabs: {
    flexDirection: 'row',
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
  },
  filterTab: { flex: 1, padding: 12, alignItems: 'center' },
  activeFilterTab: { borderBottomWidth: 2, borderBottomColor: '#2196F3' },
  filterTabText: { color: '#9E9E9E', fontWeight: '500' },
  activeFilterTabText: { color: '#2196F3' },
  searchContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    margin: 12,
    padding: 10,
    borderRadius: 10,
    gap: 8,
    elevation: 1,
  },
  searchInput: { flex: 1 },
  list: { padding: 12, paddingBottom: 80, gap: 8 },
  todoItem: {
    flexDirection: 'row',
    alignItems: 'flex-start',
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 12,
    gap: 12,
    elevation: 1,
  },
  completedItem: { opacity: 0.6 },
  checkbox: {
    width: 24,
    height: 24,
    borderRadius: 12,
    borderWidth: 2,
    borderColor: '#9E9E9E',
    justifyContent: 'center',
    alignItems: 'center',
    marginTop: 2,
  },
  checkboxChecked: { backgroundColor: '#4CAF50', borderColor: '#4CAF50' },
  todoContent: { flex: 1 },
  todoHeader: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  todoTitle: { flex: 1, fontSize: 15, fontWeight: '500', color: '#333' },
  completedTitle: { textDecorationLine: 'line-through', color: '#9E9E9E' },
  priorityBadge: { padding: 3, borderRadius: 6 },
  todoDesc: { fontSize: 13, color: '#666', marginTop: 3 },
  tagRow: { flexDirection: 'row', gap: 4, marginTop: 6 },
  tag: { backgroundColor: '#E3F2FD', paddingHorizontal: 6, paddingVertical: 2, borderRadius: 8 },
  tagText: { color: '#1565C0', fontSize: 11 },
  syncIcon: { alignSelf: 'flex-start', marginTop: 4 },
  emptyState: { alignItems: 'center', padding: 48, gap: 12 },
  emptyIcon: { fontSize: 48 },
  emptyText: { color: '#9E9E9E', textAlign: 'center' },
  fab: {
    position: 'absolute',
    right: 20,
    bottom: 24,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 8,
    shadowColor: '#2196F3',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.4,
    shadowRadius: 8,
  },
  modal: { flex: 1, backgroundColor: 'white' },
  modalHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
  },
  modalTitle: { fontSize: 17, fontWeight: 'bold' },
  cancelText: { color: '#9E9E9E', fontSize: 16 },
  saveText: { color: '#2196F3', fontWeight: 'bold', fontSize: 16 },
  disabledText: { color: '#BDBDBD' },
  form: { padding: 16, gap: 16 },
  titleInput: { fontSize: 18, borderBottomWidth: 1, borderBottomColor: '#E0E0E0', paddingVertical: 8 },
  descInput: {
    fontSize: 15,
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
    paddingVertical: 8,
    minHeight: 60,
    textAlignVertical: 'top',
  },
  formLabel: { fontSize: 14, color: '#666', fontWeight: '500' },
  priorityRow: { flexDirection: 'row', gap: 8 },
  priorityOption: {
    flex: 1,
    borderWidth: 1.5,
    borderRadius: 8,
    padding: 8,
    alignItems: 'center',
  },
  priorityLabel: { fontSize: 12, fontWeight: 'bold' },
  tagsInput: { borderBottomWidth: 1, borderBottomColor: '#E0E0E0', paddingVertical: 8 },
});

export default TodoScreen;
```

---

## Tips และ Best Practices

### 1. WAL Mode สำหรับ Performance

```typescript
// เปิด WAL mode เมื่อเปิด database
await Database.executeSql('PRAGMA journal_mode=WAL');
await Database.executeSql('PRAGMA synchronous=NORMAL');
await Database.executeSql('PRAGMA cache_size=10000');
await Database.executeSql('PRAGMA temp_store=MEMORY');
```

### 2. Sync Strategy

```typescript
// Last-Write-Wins Sync
async function syncData(localData: any[], serverData: any[]): Promise<any[]> {
  const merged = new Map<string, any>();

  // ใส่ local data ก่อน
  localData.forEach(item => merged.set(item.id, item));

  // server data ชนะถ้า updated_at ใหม่กว่า
  serverData.forEach(item => {
    const existing = merged.get(item.id);
    if (!existing || new Date(item.updated_at) > new Date(existing.updated_at)) {
      merged.set(item.id, item);
    }
  });

  return Array.from(merged.values());
}
```

### สรุป

- SQLite เหมาะกับข้อมูล structured ที่ต้องการ query ซับซ้อน
- ใช้ migrations เพื่อจัดการ schema changes
- WAL mode ช่วยให้ read/write performance ดีขึ้น
- Soft delete ช่วยให้ sync ง่ายกว่า hard delete
- Queue changes เมื่อ offline แล้ว sync เมื่อ online
- WatermelonDB เหมาะกับแอปขนาดใหญ่ที่ต้องการ reactive queries
