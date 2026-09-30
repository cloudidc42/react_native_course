# Part 096: Micro Services Backend for Mobile

## บทนำ

Microservices architecture แบ่ง backend เป็น services เล็กๆ แต่ละ service ทำหน้าที่เฉพาะ เราจะเรียนรู้การออกแบบ microservices backend สำหรับ mobile app

## หัวข้อที่จะเรียน

1. API Gateway
2. Service Mesh
3. gRPC
4. Event-driven Architecture
5. Workshop: Microservices Backend

---

## 1. Architecture Overview

```
Mobile App
    │
    ▼
API Gateway (port 3000)
    │
    ├── Auth Service (port 3001)
    ├── User Service (port 3002)
    ├── Product Service (port 3003)
    ├── Order Service (port 3004)
    ├── Payment Service (port 3005)
    ├── Notification Service (port 3006)
    └── File Service (port 3007)
    
Message Broker (RabbitMQ/Kafka)
    │
    ├── order.created → Payment Service
    ├── payment.completed → Order Service
    └── order.shipped → Notification Service
```

---

## 2. API Gateway

### gateway/src/index.ts

```typescript
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';
import rateLimit from 'express-rate-limit';
import helmet from 'helmet';
import cors from 'cors';
import jwt from 'jsonwebtoken';
import Redis from 'ioredis';

const app = express();
const redis = new Redis(process.env.REDIS_URL!);

// Security
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
  credentials: true,
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: { error: 'Too many requests, please try again later.' },
  standardHeaders: true,
  legacyHeaders: false,
  skip: (req) => req.path === '/health',
});

app.use(limiter);

// JWT Authentication Middleware
const authMiddleware = async (
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    // Check token blacklist (logout)
    const isBlacklisted = await redis.get(`blacklist:${token}`);
    if (isBlacklisted) {
      return res.status(401).json({ error: 'Token has been revoked' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      role: string;
    };
    
    req.headers['x-user-id'] = decoded.userId;
    req.headers['x-user-role'] = decoded.role;
    
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Service Routes
const services = {
  auth: process.env.AUTH_SERVICE_URL || 'http://auth-service:3001',
  users: process.env.USER_SERVICE_URL || 'http://user-service:3002',
  products: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3003',
  orders: process.env.ORDER_SERVICE_URL || 'http://order-service:3004',
  payments: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3005',
  notifications: process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3006',
  files: process.env.FILE_SERVICE_URL || 'http://file-service:3007',
};

// Public routes (no auth required)
app.use('/api/auth', createProxyMiddleware({
  target: services.auth,
  changeOrigin: true,
  pathRewrite: { '^/api/auth': '' },
  on: {
    error: (err, req, res) => {
      (res as express.Response).status(503).json({ error: 'Auth service unavailable' });
    },
  },
}));

// Protected routes
app.use('/api/users', authMiddleware, createProxyMiddleware({
  target: services.users,
  changeOrigin: true,
  pathRewrite: { '^/api/users': '' },
}));

app.use('/api/products', createProxyMiddleware({
  target: services.products,
  changeOrigin: true,
  pathRewrite: { '^/api/products': '' },
}));

app.use('/api/orders', authMiddleware, createProxyMiddleware({
  target: services.orders,
  changeOrigin: true,
  pathRewrite: { '^/api/orders': '' },
}));

app.use('/api/payments', authMiddleware, createProxyMiddleware({
  target: services.payments,
  changeOrigin: true,
  pathRewrite: { '^/api/payments': '' },
}));

app.use('/api/files', authMiddleware, createProxyMiddleware({
  target: services.files,
  changeOrigin: true,
  pathRewrite: { '^/api/files': '' },
}));

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Request logging
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} ${res.statusCode} ${duration}ms`);
  });
  next();
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`API Gateway running on port ${PORT}`);
});
```

---

## 3. Auth Service

### auth-service/src/index.ts

```typescript
import express from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { MongoClient } from 'mongodb';
import Redis from 'ioredis';

const app = express();
app.use(express.json());

const redis = new Redis(process.env.REDIS_URL!);

// Register
app.post('/register', async (req, res) => {
  try {
    const { email, password, displayName } = req.body;
    
    // Validate
    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }

    const db = await getDb();
    
    // Check existing user
    const existing = await db.collection('users').findOne({ email });
    if (existing) {
      return res.status(409).json({ error: 'Email already registered' });
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 12);
    
    // Create user
    const user = {
      email,
      passwordHash,
      displayName: displayName || email.split('@')[0],
      role: 'user',
      isActive: true,
      createdAt: new Date(),
    };
    
    const result = await db.collection('users').insertOne(user);
    
    // Generate tokens
    const tokens = generateTokens(result.insertedId.toString(), 'user');
    
    // Store refresh token
    await redis.setex(
      `refresh:${result.insertedId}`,
      60 * 60 * 24 * 7, // 7 days
      tokens.refreshToken
    );
    
    res.status(201).json({
      data: {
        user: { id: result.insertedId, email, displayName: user.displayName },
        ...tokens,
      },
    });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Login
app.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const db = await getDb();
    const user = await db.collection('users').findOne({ email });
    
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    if (!user.isActive) {
      return res.status(403).json({ error: 'Account deactivated' });
    }
    
    const tokens = generateTokens(user._id.toString(), user.role);
    
    // Store refresh token
    await redis.setex(
      `refresh:${user._id}`,
      60 * 60 * 24 * 7,
      tokens.refreshToken
    );
    
    // Update last login
    await db.collection('users').updateOne(
      { _id: user._id },
      { $set: { lastLoginAt: new Date() } }
    );
    
    res.json({
      data: {
        user: {
          id: user._id,
          email: user.email,
          displayName: user.displayName,
          role: user.role,
        },
        ...tokens,
      },
    });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Refresh Token
app.post('/refresh', async (req, res) => {
  try {
    const { refreshToken } = req.body;
    
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET!) as {
      userId: string;
      type: string;
    };
    
    if (decoded.type !== 'refresh') {
      return res.status(401).json({ error: 'Invalid token type' });
    }
    
    // Check stored refresh token
    const stored = await redis.get(`refresh:${decoded.userId}`);
    if (stored !== refreshToken) {
      return res.status(401).json({ error: 'Refresh token revoked' });
    }
    
    const db = await getDb();
    const user = await db.collection('users').findOne(
      { _id: new (require('mongodb').ObjectId)(decoded.userId) }
    );
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    const tokens = generateTokens(decoded.userId, user.role);
    
    // Update stored refresh token
    await redis.setex(
      `refresh:${decoded.userId}`,
      60 * 60 * 24 * 7,
      tokens.refreshToken
    );
    
    res.json({ data: tokens });
  } catch (error) {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
});

// Logout
app.post('/logout', async (req, res) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    const { refreshToken } = req.body;
    
    if (token) {
      // Blacklist access token
      const decoded = jwt.decode(token) as { exp: number; userId: string };
      if (decoded?.exp) {
        const ttl = decoded.exp - Math.floor(Date.now() / 1000);
        if (ttl > 0) {
          await redis.setex(`blacklist:${token}`, ttl, '1');
        }
      }
      
      // Remove refresh token
      if (decoded?.userId) {
        await redis.del(`refresh:${decoded.userId}`);
      }
    }
    
    res.json({ message: 'Logged out successfully' });
  } catch (error) {
    res.status(500).json({ error: 'Logout failed' });
  }
});

function generateTokens(userId: string, role: string) {
  const accessToken = jwt.sign(
    { userId, role, type: 'access' },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );
  
  const refreshToken = jwt.sign(
    { userId, type: 'refresh' },
    process.env.REFRESH_TOKEN_SECRET!,
    { expiresIn: '7d' }
  );
  
  return { accessToken, refreshToken, expiresIn: 900 };
}

let dbClient: MongoClient | null = null;
async function getDb() {
  if (!dbClient) {
    dbClient = await MongoClient.connect(process.env.MONGODB_URL!);
  }
  return dbClient.db('auth');
}

app.listen(3001, () => console.log('Auth service running on port 3001'));
```

---

## 4. Event-Driven Architecture

### Message Broker Setup (RabbitMQ)

```typescript
// shared/messagebroker.ts
import amqp from 'amqplib';

type Handler<T> = (message: T) => Promise<void>;

export class MessageBroker {
  private connection: amqp.Connection | null = null;
  private channel: amqp.Channel | null = null;
  private static instance: MessageBroker;

  static getInstance(): MessageBroker {
    if (!MessageBroker.instance) {
      MessageBroker.instance = new MessageBroker();
    }
    return MessageBroker.instance;
  }

  async connect(url: string): Promise<void> {
    this.connection = await amqp.connect(url);
    this.channel = await this.connection.createChannel();
    
    // Setup dead letter queue
    await this.channel.assertExchange('dead-letter', 'direct', { durable: true });
    
    console.log('Connected to RabbitMQ');
  }

  async publish<T>(exchange: string, routingKey: string, message: T): Promise<void> {
    if (!this.channel) throw new Error('Not connected');
    
    await this.channel.assertExchange(exchange, 'topic', { durable: true });
    
    const content = Buffer.from(JSON.stringify({
      data: message,
      timestamp: new Date().toISOString(),
      id: Math.random().toString(36).substr(2, 9),
    }));
    
    this.channel.publish(exchange, routingKey, content, {
      persistent: true,
      contentType: 'application/json',
    });
    
    console.log(`Published: ${exchange}.${routingKey}`);
  }

  async subscribe<T>(
    exchange: string,
    routingKey: string,
    queueName: string,
    handler: Handler<T>
  ): Promise<void> {
    if (!this.channel) throw new Error('Not connected');
    
    await this.channel.assertExchange(exchange, 'topic', { durable: true });
    
    const queue = await this.channel.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'dead-letter',
        'x-dead-letter-routing-key': `${queueName}.failed`,
      },
    });
    
    await this.channel.bindQueue(queue.queue, exchange, routingKey);
    await this.channel.prefetch(1); // Process one at a time
    
    this.channel.consume(queue.queue, async (msg) => {
      if (!msg) return;
      
      try {
        const { data } = JSON.parse(msg.content.toString());
        await handler(data);
        this.channel!.ack(msg);
      } catch (error) {
        console.error(`Error processing message:`, error);
        // Retry 3 times, then send to dead letter
        const retryCount = (msg.properties.headers?.retryCount || 0) + 1;
        if (retryCount < 3) {
          this.channel!.nack(msg, false, true);
        } else {
          this.channel!.nack(msg, false, false);
        }
      }
    });
    
    console.log(`Subscribed: ${exchange}.${routingKey} → ${queueName}`);
  }
}
```

### Events Definition

```typescript
// shared/events.ts

// Order Events
export interface OrderCreatedEvent {
  orderId: string;
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
    price: number;
  }>;
  total: number;
  currency: string;
}

export interface OrderStatusChangedEvent {
  orderId: string;
  userId: string;
  previousStatus: string;
  newStatus: string;
  reason?: string;
}

// Payment Events
export interface PaymentCompletedEvent {
  paymentId: string;
  orderId: string;
  userId: string;
  amount: number;
  method: string;
}

export interface PaymentFailedEvent {
  orderId: string;
  userId: string;
  reason: string;
}

// Event Topics
export const EVENTS = {
  // Orders
  ORDER_CREATED: 'orders.created',
  ORDER_CONFIRMED: 'orders.confirmed',
  ORDER_SHIPPED: 'orders.shipped',
  ORDER_DELIVERED: 'orders.delivered',
  ORDER_CANCELLED: 'orders.cancelled',
  
  // Payments
  PAYMENT_INITIATED: 'payments.initiated',
  PAYMENT_COMPLETED: 'payments.completed',
  PAYMENT_FAILED: 'payments.failed',
  
  // Notifications
  PUSH_NOTIFICATION: 'notifications.push',
  EMAIL_SEND: 'notifications.email',
  
  // Users
  USER_REGISTERED: 'users.registered',
  USER_UPDATED: 'users.updated',
};
```

### Order Service

```typescript
// order-service/src/index.ts
import express from 'express';
import { MessageBroker } from '../../shared/messagebroker';
import { EVENTS } from '../../shared/events';

const app = express();
app.use(express.json());

const broker = MessageBroker.getInstance();

// Subscribe to payment events
async function setupSubscriptions() {
  await broker.subscribe(
    'payments',
    EVENTS.PAYMENT_COMPLETED,
    'orders.payment-completed',
    async (event: { orderId: string; paymentId: string }) => {
      // Update order status
      await db.collection('orders').updateOne(
        { _id: event.orderId },
        { 
          $set: { 
            status: 'confirmed',
            paymentId: event.paymentId,
            confirmedAt: new Date(),
          }
        }
      );
      
      // Publish order confirmed event
      await broker.publish('orders', EVENTS.ORDER_CONFIRMED, {
        orderId: event.orderId,
        newStatus: 'confirmed',
      });
    }
  );

  await broker.subscribe(
    'payments',
    EVENTS.PAYMENT_FAILED,
    'orders.payment-failed',
    async (event: { orderId: string; reason: string }) => {
      await db.collection('orders').updateOne(
        { _id: event.orderId },
        { $set: { status: 'payment_failed', failReason: event.reason } }
      );
    }
  );
}

// Create Order
app.post('/', async (req, res) => {
  try {
    const userId = req.headers['x-user-id'] as string;
    const { items, shippingAddress, paymentMethod } = req.body;
    
    // Calculate total
    const total = items.reduce((sum: number, item: any) => 
      sum + item.price * item.quantity, 0
    );
    
    const order = {
      userId,
      items,
      shippingAddress,
      paymentMethod,
      total,
      status: 'pending',
      createdAt: new Date(),
    };
    
    const result = await db.collection('orders').insertOne(order);
    const orderId = result.insertedId.toString();
    
    // Publish order created event
    await broker.publish('orders', EVENTS.ORDER_CREATED, {
      orderId,
      userId,
      items,
      total,
      currency: 'THB',
    });
    
    res.status(201).json({ data: { orderId, status: 'pending' } });
  } catch (error) {
    res.status(500).json({ error: 'Failed to create order' });
  }
});

let db: any;
async function start() {
  await broker.connect(process.env.RABBITMQ_URL!);
  await setupSubscriptions();
  
  const { MongoClient } = require('mongodb');
  const client = await MongoClient.connect(process.env.MONGODB_URL!);
  db = client.db('orders');
  
  app.listen(3004, () => console.log('Order service on port 3004'));
}

start();
```

---

## 5. Docker Compose Setup

### docker-compose.yml

```yaml
version: '3.9'

services:
  # API Gateway
  api-gateway:
    build: ./gateway
    ports:
      - "3000:3000"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_URL=redis://redis:6379
      - AUTH_SERVICE_URL=http://auth-service:3001
      - USER_SERVICE_URL=http://user-service:3002
      - PRODUCT_SERVICE_URL=http://product-service:3003
      - ORDER_SERVICE_URL=http://order-service:3004
      - PAYMENT_SERVICE_URL=http://payment-service:3005
    depends_on:
      - redis
      - auth-service

  # Auth Service
  auth-service:
    build: ./auth-service
    environment:
      - MONGODB_URL=mongodb://mongo:27017/auth
      - JWT_SECRET=${JWT_SECRET}
      - REFRESH_TOKEN_SECRET=${REFRESH_TOKEN_SECRET}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - mongo
      - redis

  # Order Service
  order-service:
    build: ./order-service
    environment:
      - MONGODB_URL=mongodb://mongo:27017/orders
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - mongo
      - rabbitmq

  # Payment Service
  payment-service:
    build: ./payment-service
    environment:
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - rabbitmq

  # MongoDB
  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

volumes:
  mongo-data:
  redis-data:
  rabbitmq-data:
```

---

## Workshop Exercises

1. **Service Discovery** - ใช้ Consul สำหรับ service discovery
2. **Circuit Breaker** - implement circuit breaker pattern
3. **Distributed Tracing** - ใช้ Jaeger หรือ Zipkin
4. **GraphQL Federation** - รวม multiple services ด้วย GraphQL

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **API Gateway** - single entry point สำหรับ mobile
2. **Auth Service** - JWT authentication, token refresh
3. **Event-driven** - asynchronous communication ด้วย RabbitMQ
4. **Order Service** - business logic ที่ใช้ events
5. **Docker** - container setup สำหรับทุก services
