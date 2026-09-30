# Part 100: Full Project Capstone - Social Media App (SocialRN)

## บทนำ

ยินดีต้อนรับสู่บทสุดท้ายของ React Native Course! ในบทนี้เราจะสร้าง Social Media App แบบ Full-Stack ที่ครบครันตั้งแต่ต้นจนจบ โดยใช้ความรู้จากทุกบทที่ผ่านมา

**SocialRN** - แอปโซเชียลมีเดียที่มี:
- React Native Frontend (iOS + Android)
- Node.js + Express Backend
- MongoDB Database
- Real-time features (Socket.io)
- JWT Authentication
- Push Notifications (FCM/APNs)
- File Uploads (S3)
- Complete Deployment (CI/CD)

---

## สถาปัตยกรรมโดยรวม

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                          │
│              React Native App (iOS/Android)              │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / WebSocket
┌────────────────────────▼────────────────────────────────┐
│                    API GATEWAY                           │
│           (Rate Limiting, Auth, Logging)                 │
└──┬─────────────┬──────────────┬──────────────┬──────────┘
   │             │              │              │
┌──▼──┐    ┌────▼───┐    ┌─────▼──┐    ┌──────▼──┐
│Auth │    │ Posts  │    │ Users  │    │  Files  │
│Svc  │    │  Svc   │    │  Svc   │    │   Svc   │
└──┬──┘    └────┬───┘    └─────┬──┘    └──────┬──┘
   │             │              │              │
┌──▼─────────────▼──────────────▼──────────────▼──┐
│                   MongoDB Atlas                  │
│         users / posts / comments / likes        │
└──────────────────────────────────────────────────┘
```

---

## Part 1: Project Setup

### Backend Structure

```
backend/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   └── env.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── upload.ts
│   │   ├── rateLimit.ts
│   │   └── errorHandler.ts
│   ├── models/
│   │   ├── User.ts
│   │   ├── Post.ts
│   │   ├── Comment.ts
│   │   └── Notification.ts
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── users.ts
│   │   ├── posts.ts
│   │   ├── comments.ts
│   │   └── notifications.ts
│   ├── services/
│   │   ├── authService.ts
│   │   ├── postService.ts
│   │   ├── notificationService.ts
│   │   └── storageService.ts
│   ├── socket/
│   │   └── socketHandlers.ts
│   └── app.ts
├── package.json
└── tsconfig.json
```

### Frontend Structure

```
mobile/
├── src/
│   ├── api/
│   │   ├── client.ts
│   │   ├── auth.ts
│   │   ├── posts.ts
│   │   └── users.ts
│   ├── components/
│   │   ├── common/
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Avatar.tsx
│   │   │   └── Loading.tsx
│   │   ├── post/
│   │   │   ├── PostCard.tsx
│   │   │   ├── PostForm.tsx
│   │   │   └── CommentList.tsx
│   │   └── user/
│   │       ├── UserCard.tsx
│   │       └── FollowButton.tsx
│   ├── screens/
│   │   ├── auth/
│   │   │   ├── LoginScreen.tsx
│   │   │   └── RegisterScreen.tsx
│   │   ├── home/
│   │   │   └── HomeScreen.tsx
│   │   ├── profile/
│   │   │   └── ProfileScreen.tsx
│   │   ├── search/
│   │   │   └── SearchScreen.tsx
│   │   └── notifications/
│   │       └── NotificationsScreen.tsx
│   ├── store/
│   │   ├── authStore.ts
│   │   ├── postsStore.ts
│   │   └── notificationsStore.ts
│   ├── navigation/
│   │   ├── AppNavigator.tsx
│   │   ├── AuthNavigator.tsx
│   │   └── MainNavigator.tsx
│   ├── hooks/
│   │   ├── useSocket.ts
│   │   ├── usePushNotifications.ts
│   │   └── useInfiniteScroll.ts
│   └── utils/
│       ├── formatters.ts
│       └── validators.ts
├── package.json
└── tsconfig.json
```

---

## Part 2: Backend - Database Models

### User Model

```typescript
// backend/src/models/User.ts
import mongoose, { Document, Schema } from 'mongoose';
import bcrypt from 'bcryptjs';

export interface IUser extends Document {
  username: string;
  email: string;
  password: string;
  displayName: string;
  bio?: string;
  avatar?: string;
  coverImage?: string;
  followers: mongoose.Types.ObjectId[];
  following: mongoose.Types.ObjectId[];
  isVerified: boolean;
  fcmToken?: string;
  createdAt: Date;
  updatedAt: Date;
  comparePassword(password: string): Promise<boolean>;
  getPublicProfile(): Partial<IUser>;
}

const UserSchema = new Schema<IUser>(
  {
    username: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
      minlength: 3,
      maxlength: 30,
      match: /^[a-z0-9_.]+$/,
    },
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    password: {
      type: String,
      required: true,
      minlength: 8,
      select: false,
    },
    displayName: {
      type: String,
      required: true,
      trim: true,
      maxlength: 50,
    },
    bio: { type: String, maxlength: 200 },
    avatar: { type: String },
    coverImage: { type: String },
    followers: [{ type: Schema.Types.ObjectId, ref: 'User' }],
    following: [{ type: Schema.Types.ObjectId, ref: 'User' }],
    isVerified: { type: Boolean, default: false },
    fcmToken: { type: String },
  },
  { timestamps: true }
);

// Hash password before save
UserSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  const salt = await bcrypt.genSalt(12);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

UserSchema.methods.comparePassword = async function (password: string): Promise<boolean> {
  return bcrypt.compare(password, this.password);
};

UserSchema.methods.getPublicProfile = function (): Partial<IUser> {
  const { password, fcmToken, ...profile } = this.toObject();
  return profile;
};

// Index
UserSchema.index({ username: 'text', displayName: 'text' });

export const User = mongoose.model<IUser>('User', UserSchema);
```

### Post Model

```typescript
// backend/src/models/Post.ts
import mongoose, { Document, Schema } from 'mongoose';

export interface IPost extends Document {
  author: mongoose.Types.ObjectId;
  content: string;
  media: Array<{
    url: string;
    type: 'image' | 'video';
    thumbnail?: string;
    width?: number;
    height?: number;
  }>;
  likes: mongoose.Types.ObjectId[];
  comments: mongoose.Types.ObjectId[];
  shares: number;
  hashtags: string[];
  mentions: mongoose.Types.ObjectId[];
  location?: {
    name: string;
    coordinates: [number, number];
  };
  visibility: 'public' | 'followers' | 'private';
  isArchived: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const PostSchema = new Schema<IPost>(
  {
    author: {
      type: Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    content: {
      type: String,
      required: true,
      maxlength: 2000,
    },
    media: [
      {
        url: { type: String, required: true },
        type: { type: String, enum: ['image', 'video'], required: true },
        thumbnail: String,
        width: Number,
        height: Number,
      },
    ],
    likes: [{ type: Schema.Types.ObjectId, ref: 'User' }],
    comments: [{ type: Schema.Types.ObjectId, ref: 'Comment' }],
    shares: { type: Number, default: 0 },
    hashtags: [{ type: String, lowercase: true }],
    mentions: [{ type: Schema.Types.ObjectId, ref: 'User' }],
    location: {
      name: String,
      coordinates: { type: [Number], index: '2dsphere' },
    },
    visibility: {
      type: String,
      enum: ['public', 'followers', 'private'],
      default: 'public',
    },
    isArchived: { type: Boolean, default: false },
  },
  { timestamps: true }
);

PostSchema.index({ author: 1, createdAt: -1 });
PostSchema.index({ hashtags: 1 });
PostSchema.index({ content: 'text' });

export const Post = mongoose.model<IPost>('Post', PostSchema);
```

### Comment Model

```typescript
// backend/src/models/Comment.ts
import mongoose, { Document, Schema } from 'mongoose';

export interface IComment extends Document {
  post: mongoose.Types.ObjectId;
  author: mongoose.Types.ObjectId;
  content: string;
  likes: mongoose.Types.ObjectId[];
  replies: mongoose.Types.ObjectId[];
  parentComment?: mongoose.Types.ObjectId;
  createdAt: Date;
}

const CommentSchema = new Schema<IComment>(
  {
    post: { type: Schema.Types.ObjectId, ref: 'Post', required: true },
    author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    content: { type: String, required: true, maxlength: 500 },
    likes: [{ type: Schema.Types.ObjectId, ref: 'User' }],
    replies: [{ type: Schema.Types.ObjectId, ref: 'Comment' }],
    parentComment: { type: Schema.Types.ObjectId, ref: 'Comment' },
  },
  { timestamps: true }
);

export const Comment = mongoose.model<IComment>('Comment', CommentSchema);
```

### Notification Model

```typescript
// backend/src/models/Notification.ts
import mongoose, { Document, Schema } from 'mongoose';

export type NotificationType = 
  | 'like' | 'comment' | 'follow' | 'mention' | 'share' | 'reply';

export interface INotification extends Document {
  recipient: mongoose.Types.ObjectId;
  sender: mongoose.Types.ObjectId;
  type: NotificationType;
  post?: mongoose.Types.ObjectId;
  comment?: mongoose.Types.ObjectId;
  isRead: boolean;
  createdAt: Date;
}

const NotificationSchema = new Schema<INotification>(
  {
    recipient: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    sender: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    type: {
      type: String,
      enum: ['like', 'comment', 'follow', 'mention', 'share', 'reply'],
      required: true,
    },
    post: { type: Schema.Types.ObjectId, ref: 'Post' },
    comment: { type: Schema.Types.ObjectId, ref: 'Comment' },
    isRead: { type: Boolean, default: false },
  },
  { timestamps: true }
);

NotificationSchema.index({ recipient: 1, isRead: 1, createdAt: -1 });

export const Notification = mongoose.model<INotification>('Notification', NotificationSchema);
```

---

## Part 3: Backend - Routes และ Services

### Authentication Service

```typescript
// backend/src/services/authService.ts
import jwt from 'jsonwebtoken';
import { User, IUser } from '../models/User';
import { redisClient } from '../config/redis';

interface TokenPayload {
  userId: string;
  username: string;
}

interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET!;
  private readonly REFRESH_SECRET = process.env.REFRESH_SECRET!;
  private readonly ACCESS_EXPIRY = '15m';
  private readonly REFRESH_EXPIRY = '7d';

  generateTokens(payload: TokenPayload): AuthTokens {
    const accessToken = jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: this.ACCESS_EXPIRY,
    });
    const refreshToken = jwt.sign(payload, this.REFRESH_SECRET, {
      expiresIn: this.REFRESH_EXPIRY,
    });
    return { accessToken, refreshToken };
  }

  async register(data: {
    username: string;
    email: string;
    password: string;
    displayName: string;
  }): Promise<{ user: IUser; tokens: AuthTokens }> {
    // Check existing
    const existing = await User.findOne({
      $or: [{ email: data.email }, { username: data.username }],
    });
    if (existing) {
      throw new Error(
        existing.email === data.email
          ? 'Email นี้ถูกใช้แล้ว'
          : 'Username นี้ถูกใช้แล้ว'
      );
    }

    const user = await User.create(data);
    const tokens = this.generateTokens({
      userId: user._id.toString(),
      username: user.username,
    });

    // Store refresh token
    await redisClient.setEx(
      `refresh:${user._id}`,
      7 * 24 * 60 * 60,
      tokens.refreshToken
    );

    return { user, tokens };
  }

  async login(
    identifier: string,
    password: string
  ): Promise<{ user: IUser; tokens: AuthTokens }> {
    const user = await User.findOne({
      $or: [{ email: identifier }, { username: identifier }],
    }).select('+password');

    if (!user || !(await user.comparePassword(password))) {
      throw new Error('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง');
    }

    const tokens = this.generateTokens({
      userId: user._id.toString(),
      username: user.username,
    });

    await redisClient.setEx(
      `refresh:${user._id}`,
      7 * 24 * 60 * 60,
      tokens.refreshToken
    );

    return { user, tokens };
  }

  async refreshTokens(refreshToken: string): Promise<AuthTokens> {
    const decoded = jwt.verify(refreshToken, this.REFRESH_SECRET) as TokenPayload;
    
    const storedToken = await redisClient.get(`refresh:${decoded.userId}`);
    if (storedToken !== refreshToken) {
      throw new Error('Refresh token ไม่ถูกต้อง');
    }

    const tokens = this.generateTokens({
      userId: decoded.userId,
      username: decoded.username,
    });

    await redisClient.setEx(
      `refresh:${decoded.userId}`,
      7 * 24 * 60 * 60,
      tokens.refreshToken
    );

    return tokens;
  }

  async logout(userId: string): Promise<void> {
    await redisClient.del(`refresh:${userId}`);
  }

  verifyAccessToken(token: string): TokenPayload {
    return jwt.verify(token, this.JWT_SECRET) as TokenPayload;
  }
}

export const authService = new AuthService();
```

### Post Service

```typescript
// backend/src/services/postService.ts
import mongoose from 'mongoose';
import { Post, IPost } from '../models/Post';
import { User } from '../models/User';
import { Notification } from '../models/Notification';
import { NotificationService } from './notificationService';

export class PostService {
  private notificationService = new NotificationService();

  async createPost(
    authorId: string,
    data: {
      content: string;
      media?: Array<{ url: string; type: 'image' | 'video' }>;
      visibility?: 'public' | 'followers' | 'private';
    }
  ): Promise<IPost> {
    // Extract hashtags and mentions
    const hashtags = this.extractHashtags(data.content);
    const mentionUsernames = this.extractMentions(data.content);

    // Resolve mention usernames to IDs
    const mentionedUsers = await User.find({
      username: { $in: mentionUsernames },
    }).select('_id username');

    const post = await Post.create({
      author: authorId,
      content: data.content,
      media: data.media || [],
      hashtags,
      mentions: mentionedUsers.map(u => u._id),
      visibility: data.visibility || 'public',
    });

    // Send mention notifications
    for (const user of mentionedUsers) {
      if (user._id.toString() !== authorId) {
        await this.notificationService.createNotification({
          recipientId: user._id.toString(),
          senderId: authorId,
          type: 'mention',
          postId: post._id.toString(),
        });
      }
    }

    return post.populate('author', 'username displayName avatar');
  }

  async getFeed(
    userId: string,
    options: { page?: number; limit?: number; cursor?: string }
  ): Promise<{ posts: IPost[]; nextCursor?: string; hasMore: boolean }> {
    const { page = 1, limit = 20, cursor } = options;

    // Get following list
    const user = await User.findById(userId).select('following');
    const followingIds = user?.following || [];
    const authorIds = [...followingIds, new mongoose.Types.ObjectId(userId)];

    let query: any = {
      author: { $in: authorIds },
      visibility: { $in: ['public', 'followers'] },
      isArchived: false,
    };

    if (cursor) {
      query._id = { $lt: new mongoose.Types.ObjectId(cursor) };
    }

    const posts = await Post.find(query)
      .sort({ _id: -1 })
      .limit(limit + 1)
      .populate('author', 'username displayName avatar isVerified')
      .lean();

    const hasMore = posts.length > limit;
    const resultPosts = hasMore ? posts.slice(0, limit) : posts;
    const nextCursor = hasMore ? resultPosts[resultPosts.length - 1]._id.toString() : undefined;

    // Add isLiked field
    const postsWithLiked = resultPosts.map(post => ({
      ...post,
      isLiked: (post.likes as any[]).some(
        id => id.toString() === userId
      ),
      likesCount: (post.likes as any[]).length,
      commentsCount: (post.comments as any[]).length,
    }));

    return { posts: postsWithLiked as any, nextCursor, hasMore };
  }

  async likePost(userId: string, postId: string): Promise<boolean> {
    const post = await Post.findById(postId);
    if (!post) throw new Error('ไม่พบ Post');

    const isLiked = post.likes.some(id => id.toString() === userId);

    if (isLiked) {
      post.likes = post.likes.filter(id => id.toString() !== userId);
    } else {
      post.likes.push(new mongoose.Types.ObjectId(userId));
      
      // Notify post author
      if (post.author.toString() !== userId) {
        await this.notificationService.createNotification({
          recipientId: post.author.toString(),
          senderId: userId,
          type: 'like',
          postId: post._id.toString(),
        });
      }
    }

    await post.save();
    return !isLiked; // Return new like state
  }

  async deletePost(userId: string, postId: string): Promise<void> {
    const post = await Post.findById(postId);
    if (!post) throw new Error('ไม่พบ Post');
    if (post.author.toString() !== userId) {
      throw new Error('ไม่มีสิทธิ์ลบ Post นี้');
    }
    await post.deleteOne();
  }

  private extractHashtags(content: string): string[] {
    const matches = content.match(/#[\w฀-๿]+/g) || [];
    return [...new Set(matches.map(h => h.slice(1).toLowerCase()))];
  }

  private extractMentions(content: string): string[] {
    const matches = content.match(/@[\w.]+/g) || [];
    return [...new Set(matches.map(m => m.slice(1).toLowerCase()))];
  }
}
```

### Notification Service

```typescript
// backend/src/services/notificationService.ts
import { Notification, INotification, NotificationType } from '../models/Notification';
import { User } from '../models/User';
import admin from 'firebase-admin';

export class NotificationService {
  async createNotification(data: {
    recipientId: string;
    senderId: string;
    type: NotificationType;
    postId?: string;
    commentId?: string;
  }): Promise<INotification> {
    const notification = await Notification.create({
      recipient: data.recipientId,
      sender: data.senderId,
      type: data.type,
      post: data.postId,
      comment: data.commentId,
    });

    // Send push notification
    const recipient = await User.findById(data.recipientId).select('fcmToken');
    if (recipient?.fcmToken) {
      await this.sendPushNotification(
        recipient.fcmToken,
        notification._id.toString(),
        data
      );
    }

    return notification;
  }

  private async sendPushNotification(
    fcmToken: string,
    notificationId: string,
    data: any
  ): Promise<void> {
    const sender = await User.findById(data.senderId).select('displayName username');
    
    const messages: Record<string, { title: string; body: string }> = {
      like: { title: 'ถูกใจโพสต์', body: `${sender?.displayName} ถูกใจโพสต์ของคุณ` },
      comment: { title: 'ความคิดเห็นใหม่', body: `${sender?.displayName} แสดงความคิดเห็น` },
      follow: { title: 'ผู้ติดตามใหม่', body: `${sender?.displayName} เริ่มติดตามคุณ` },
      mention: { title: 'ถูกกล่าวถึง', body: `${sender?.displayName} กล่าวถึงคุณในโพสต์` },
      share: { title: 'แชร์โพสต์', body: `${sender?.displayName} แชร์โพสต์ของคุณ` },
      reply: { title: 'ตอบกลับความคิดเห็น', body: `${sender?.displayName} ตอบกลับความคิดเห็นของคุณ` },
    };

    const msg = messages[data.type];
    
    try {
      await admin.messaging().send({
        token: fcmToken,
        notification: {
          title: msg.title,
          body: msg.body,
        },
        data: {
          notificationId,
          type: data.type,
          postId: data.postId || '',
        },
        android: {
          priority: 'high',
          notification: { channelId: 'social_notifications' },
        },
        apns: {
          payload: {
            aps: {
              badge: 1,
              sound: 'default',
            },
          },
        },
      });
    } catch (error) {
      console.error('Push notification failed:', error);
    }
  }

  async getUserNotifications(
    userId: string,
    options: { page?: number; limit?: number }
  ) {
    const { page = 1, limit = 20 } = options;
    const skip = (page - 1) * limit;

    const [notifications, unreadCount] = await Promise.all([
      Notification.find({ recipient: userId })
        .sort({ createdAt: -1 })
        .skip(skip)
        .limit(limit)
        .populate('sender', 'username displayName avatar')
        .populate('post', 'content media'),
      Notification.countDocuments({ recipient: userId, isRead: false }),
    ]);

    return { notifications, unreadCount };
  }

  async markAsRead(userId: string, notificationId?: string): Promise<void> {
    const query: any = { recipient: userId };
    if (notificationId) query._id = notificationId;
    await Notification.updateMany(query, { isRead: true });
  }
}
```

---

## Part 4: Backend - Auth Middleware และ Routes

### Auth Middleware

```typescript
// backend/src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { authService } from '../services/authService';

export interface AuthRequest extends Request {
  userId?: string;
  username?: string;
}

export const authenticate = (
  req: AuthRequest,
  res: Response,
  next: NextFunction
): void => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    res.status(401).json({ success: false, message: 'ต้องเข้าสู่ระบบก่อน' });
    return;
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = authService.verifyAccessToken(token);
    req.userId = payload.userId;
    req.username = payload.username;
    next();
  } catch {
    res.status(401).json({ success: false, message: 'Token ไม่ถูกต้องหรือหมดอายุ' });
  }
};

export const optionalAuth = (
  req: AuthRequest,
  _res: Response,
  next: NextFunction
): void => {
  const authHeader = req.headers.authorization;
  if (authHeader?.startsWith('Bearer ')) {
    try {
      const token = authHeader.split(' ')[1];
      const payload = authService.verifyAccessToken(token);
      req.userId = payload.userId;
      req.username = payload.username;
    } catch {
      // Ignore error - optional auth
    }
  }
  next();
};
```

### Auth Routes

```typescript
// backend/src/routes/auth.ts
import { Router, Request, Response } from 'express';
import { body, validationResult } from 'express-validator';
import { authService } from '../services/authService';
import { authenticate, AuthRequest } from '../middleware/auth';

const router = Router();

router.post(
  '/register',
  [
    body('username').isLength({ min: 3, max: 30 }).matches(/^[a-z0-9_.]+$/i),
    body('email').isEmail(),
    body('password').isLength({ min: 8 }),
    body('displayName').isLength({ min: 1, max: 50 }),
  ],
  async (req: Request, res: Response) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      res.status(400).json({ success: false, errors: errors.array() });
      return;
    }

    try {
      const { user, tokens } = await authService.register(req.body);
      res.status(201).json({
        success: true,
        data: { user: user.getPublicProfile(), tokens },
      });
    } catch (error: any) {
      res.status(400).json({ success: false, message: error.message });
    }
  }
);

router.post(
  '/login',
  [
    body('identifier').notEmpty(),
    body('password').notEmpty(),
  ],
  async (req: Request, res: Response) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      res.status(400).json({ success: false, errors: errors.array() });
      return;
    }

    try {
      const { identifier, password } = req.body;
      const { user, tokens } = await authService.login(identifier, password);
      res.json({
        success: true,
        data: { user: user.getPublicProfile(), tokens },
      });
    } catch (error: any) {
      res.status(401).json({ success: false, message: error.message });
    }
  }
);

router.post('/refresh', async (req: Request, res: Response) => {
  try {
    const { refreshToken } = req.body;
    if (!refreshToken) {
      res.status(400).json({ success: false, message: 'ต้องการ refresh token' });
      return;
    }
    const tokens = await authService.refreshTokens(refreshToken);
    res.json({ success: true, data: { tokens } });
  } catch (error: any) {
    res.status(401).json({ success: false, message: error.message });
  }
});

router.post('/logout', authenticate, async (req: AuthRequest, res: Response) => {
  try {
    await authService.logout(req.userId!);
    res.json({ success: true, message: 'ออกจากระบบสำเร็จ' });
  } catch (error: any) {
    res.status(500).json({ success: false, message: error.message });
  }
});

export default router;
```

### Posts Routes

```typescript
// backend/src/routes/posts.ts
import { Router, Response } from 'express';
import { authenticate, AuthRequest } from '../middleware/auth';
import { PostService } from '../services/postService';
import { upload } from '../middleware/upload';

const router = Router();
const postService = new PostService();

// Get feed
router.get('/feed', authenticate, async (req: AuthRequest, res: Response) => {
  try {
    const { cursor, limit } = req.query;
    const result = await postService.getFeed(req.userId!, {
      cursor: cursor as string,
      limit: limit ? parseInt(limit as string) : 20,
    });
    res.json({ success: true, data: result });
  } catch (error: any) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Create post
router.post(
  '/',
  authenticate,
  upload.array('media', 10),
  async (req: AuthRequest, res: Response) => {
    try {
      const { content, visibility } = req.body;
      const files = req.files as Express.Multer.File[];
      
      // Upload to S3 (handled in storageService)
      const media = files?.map(file => ({
        url: (file as any).location, // S3 URL
        type: file.mimetype.startsWith('image/') ? 'image' : 'video' as 'image' | 'video',
      }));

      const post = await postService.createPost(req.userId!, {
        content,
        media,
        visibility,
      });

      // Emit to socket for real-time feed
      const io = req.app.get('io');
      io?.emit(`feed:${req.userId}`, { type: 'new_post', post });

      res.status(201).json({ success: true, data: { post } });
    } catch (error: any) {
      res.status(400).json({ success: false, message: error.message });
    }
  }
);

// Like/Unlike post
router.post('/:postId/like', authenticate, async (req: AuthRequest, res: Response) => {
  try {
    const isLiked = await postService.likePost(req.userId!, req.params.postId);
    
    // Real-time update
    const io = req.app.get('io');
    io?.to(`post:${req.params.postId}`).emit('like_update', {
      postId: req.params.postId,
      userId: req.userId,
      isLiked,
    });

    res.json({ success: true, data: { isLiked } });
  } catch (error: any) {
    res.status(400).json({ success: false, message: error.message });
  }
});

// Delete post
router.delete('/:postId', authenticate, async (req: AuthRequest, res: Response) => {
  try {
    await postService.deletePost(req.userId!, req.params.postId);
    res.json({ success: true, message: 'ลบโพสต์สำเร็จ' });
  } catch (error: any) {
    res.status(400).json({ success: false, message: error.message });
  }
});

export default router;
```

### Socket.io Handlers

```typescript
// backend/src/socket/socketHandlers.ts
import { Server, Socket } from 'socket.io';
import { authService } from '../services/authService';
import { Comment } from '../models/Comment';
import { Post } from '../models/Post';

export const setupSocketHandlers = (io: Server): void => {
  // Auth middleware for socket
  io.use((socket: Socket, next) => {
    const token = socket.handshake.auth.token;
    if (!token) {
      next(new Error('Authentication required'));
      return;
    }
    try {
      const payload = authService.verifyAccessToken(token);
      (socket as any).userId = payload.userId;
      (socket as any).username = payload.username;
      next();
    } catch {
      next(new Error('Invalid token'));
    }
  });

  io.on('connection', (socket: Socket) => {
    const userId = (socket as any).userId;
    console.log(`User connected: ${userId}`);

    // Join personal room
    socket.join(`user:${userId}`);

    // Join post rooms for real-time updates
    socket.on('subscribe_post', (postId: string) => {
      socket.join(`post:${postId}`);
    });

    socket.on('unsubscribe_post', (postId: string) => {
      socket.leave(`post:${postId}`);
    });

    // Real-time typing indicator
    socket.on('typing', ({ postId }: { postId: string }) => {
      socket.to(`post:${postId}`).emit('user_typing', {
        userId,
        username: (socket as any).username,
      });
    });

    // Real-time comment
    socket.on('add_comment', async (data: {
      postId: string;
      content: string;
      parentCommentId?: string;
    }) => {
      try {
        const post = await Post.findById(data.postId);
        if (!post) return;

        const comment = await Comment.create({
          post: data.postId,
          author: userId,
          content: data.content,
          parentComment: data.parentCommentId,
        });

        await Post.findByIdAndUpdate(data.postId, {
          $push: { comments: comment._id },
        });

        const populatedComment = await comment.populate(
          'author',
          'username displayName avatar'
        );

        // Emit to post room
        io.to(`post:${data.postId}`).emit('new_comment', {
          comment: populatedComment,
        });
      } catch (error) {
        socket.emit('error', { message: 'เพิ่มความคิดเห็นไม่สำเร็จ' });
      }
    });

    socket.on('disconnect', () => {
      console.log(`User disconnected: ${userId}`);
    });
  });
};
```

---

## Part 5: Frontend - API Client

### API Client

```typescript
// mobile/src/api/client.ts
import axios, { AxiosInstance, AxiosError, InternalAxiosRequestConfig } from 'axios';
import { useAuthStore } from '../store/authStore';

const BASE_URL = __DEV__
  ? 'http://localhost:3000/api'
  : 'https://api.socialrn.com/api';

export const createAPIClient = (): AxiosInstance => {
  const client = axios.create({
    baseURL: BASE_URL,
    timeout: 15000,
    headers: {
      'Content-Type': 'application/json',
    },
  });

  // Request interceptor - add auth token
  client.interceptors.request.use(
    (config: InternalAxiosRequestConfig) => {
      const { accessToken } = useAuthStore.getState();
      if (accessToken) {
        config.headers.Authorization = `Bearer ${accessToken}`;
      }
      return config;
    },
    (error) => Promise.reject(error)
  );

  // Response interceptor - handle token refresh
  client.interceptors.response.use(
    (response) => response,
    async (error: AxiosError) => {
      const original = error.config as InternalAxiosRequestConfig & { _retry?: boolean };
      
      if (error.response?.status === 401 && !original._retry) {
        original._retry = true;
        
        try {
          const { refreshToken, setTokens, logout } = useAuthStore.getState();
          if (!refreshToken) {
            logout();
            return Promise.reject(error);
          }

          const response = await axios.post(`${BASE_URL}/auth/refresh`, {
            refreshToken,
          });

          const { tokens } = response.data.data;
          setTokens(tokens.accessToken, tokens.refreshToken);
          
          original.headers.Authorization = `Bearer ${tokens.accessToken}`;
          return client(original);
        } catch {
          useAuthStore.getState().logout();
          return Promise.reject(error);
        }
      }
      
      return Promise.reject(error);
    }
  );

  return client;
};

export const apiClient = createAPIClient();
```

---

## Part 6: Frontend - State Management

### Auth Store

```typescript
// mobile/src/store/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { apiClient } from '../api/client';

interface User {
  _id: string;
  username: string;
  email: string;
  displayName: string;
  avatar?: string;
  bio?: string;
  followers: string[];
  following: string[];
  isVerified: boolean;
}

interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (identifier: string, password: string) => Promise<void>;
  register: (data: RegisterData) => Promise<void>;
  logout: () => void;
  setTokens: (access: string, refresh: string) => void;
  updateUser: (data: Partial<User>) => void;
}

interface RegisterData {
  username: string;
  email: string;
  password: string;
  displayName: string;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      refreshToken: null,
      isAuthenticated: false,
      isLoading: false,

      login: async (identifier, password) => {
        set({ isLoading: true });
        try {
          const { data } = await apiClient.post('/auth/login', {
            identifier,
            password,
          });
          const { user, tokens } = data.data;
          set({
            user,
            accessToken: tokens.accessToken,
            refreshToken: tokens.refreshToken,
            isAuthenticated: true,
          });
        } finally {
          set({ isLoading: false });
        }
      },

      register: async (registerData) => {
        set({ isLoading: true });
        try {
          const { data } = await apiClient.post('/auth/register', registerData);
          const { user, tokens } = data.data;
          set({
            user,
            accessToken: tokens.accessToken,
            refreshToken: tokens.refreshToken,
            isAuthenticated: true,
          });
        } finally {
          set({ isLoading: false });
        }
      },

      logout: () => {
        apiClient.post('/auth/logout').catch(() => {});
        set({
          user: null,
          accessToken: null,
          refreshToken: null,
          isAuthenticated: false,
        });
      },

      setTokens: (access, refresh) => {
        set({ accessToken: access, refreshToken: refresh });
      },

      updateUser: (data) => {
        const { user } = get();
        if (user) {
          set({ user: { ...user, ...data } });
        }
      },
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({
        user: state.user,
        accessToken: state.accessToken,
        refreshToken: state.refreshToken,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);
```

### Posts Store

```typescript
// mobile/src/store/postsStore.ts
import { create } from 'zustand';
import { apiClient } from '../api/client';

interface Media {
  url: string;
  type: 'image' | 'video';
}

interface Author {
  _id: string;
  username: string;
  displayName: string;
  avatar?: string;
  isVerified: boolean;
}

interface Post {
  _id: string;
  author: Author;
  content: string;
  media: Media[];
  likesCount: number;
  commentsCount: number;
  isLiked: boolean;
  hashtags: string[];
  createdAt: string;
}

interface PostsState {
  feed: Post[];
  isLoading: boolean;
  isRefreshing: boolean;
  hasMore: boolean;
  cursor?: string;
  fetchFeed: (refresh?: boolean) => Promise<void>;
  likePost: (postId: string) => void;
  addPost: (post: Post) => void;
  removePost: (postId: string) => void;
}

export const usePostsStore = create<PostsState>((set, get) => ({
  feed: [],
  isLoading: false,
  isRefreshing: false,
  hasMore: true,
  cursor: undefined,

  fetchFeed: async (refresh = false) => {
    const { isLoading, cursor, hasMore } = get();
    if (isLoading) return;
    if (!refresh && !hasMore) return;

    set({ isLoading: true, isRefreshing: refresh });

    try {
      const params: any = { limit: 20 };
      if (!refresh && cursor) {
        params.cursor = cursor;
      }

      const { data } = await apiClient.get('/posts/feed', { params });
      const { posts, nextCursor, hasMore: more } = data.data;

      set((state) => ({
        feed: refresh ? posts : [...state.feed, ...posts],
        cursor: nextCursor,
        hasMore: more,
      }));
    } finally {
      set({ isLoading: false, isRefreshing: false });
    }
  },

  likePost: (postId) => {
    // Optimistic update
    set((state) => ({
      feed: state.feed.map((post) =>
        post._id === postId
          ? {
              ...post,
              isLiked: !post.isLiked,
              likesCount: post.isLiked
                ? post.likesCount - 1
                : post.likesCount + 1,
            }
          : post
      ),
    }));

    // API call
    apiClient.post(`/posts/${postId}/like`).catch(() => {
      // Revert on error
      set((state) => ({
        feed: state.feed.map((post) =>
          post._id === postId
            ? {
                ...post,
                isLiked: !post.isLiked,
                likesCount: post.isLiked
                  ? post.likesCount - 1
                  : post.likesCount + 1,
              }
            : post
        ),
      }));
    });
  },

  addPost: (post) => {
    set((state) => ({ feed: [post, ...state.feed] }));
  },

  removePost: (postId) => {
    set((state) => ({
      feed: state.feed.filter((p) => p._id !== postId),
    }));
  },
}));
```

---

## Part 7: Frontend - Screens

### Login Screen

```typescript
// mobile/src/screens/auth/LoginScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  ActivityIndicator,
  Alert,
} from 'react-native';
import { useAuthStore } from '../../store/authStore';
import type { NavigationProp } from '@react-navigation/native';

interface Props {
  navigation: NavigationProp<any>;
}

const LoginScreen: React.FC<Props> = ({ navigation }) => {
  const [identifier, setIdentifier] = useState('');
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  
  const { login, isLoading } = useAuthStore();

  const handleLogin = async () => {
    if (!identifier.trim() || !password) {
      Alert.alert('ข้อผิดพลาด', 'กรุณากรอกข้อมูลให้ครบถ้วน');
      return;
    }

    try {
      await login(identifier.trim(), password);
    } catch (error: any) {
      Alert.alert(
        'เข้าสู่ระบบไม่สำเร็จ',
        error?.response?.data?.message || 'กรุณาลองใหม่อีกครั้ง'
      );
    }
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <View style={styles.content}>
        <Text style={styles.logo}>SocialRN</Text>
        <Text style={styles.tagline}>เชื่อมต่อกับคนที่คุณรัก</Text>

        <View style={styles.form}>
          <TextInput
            style={styles.input}
            placeholder="อีเมล หรือ ชื่อผู้ใช้"
            value={identifier}
            onChangeText={setIdentifier}
            autoCapitalize="none"
            keyboardType="email-address"
          />

          <View style={styles.passwordContainer}>
            <TextInput
              style={styles.passwordInput}
              placeholder="รหัสผ่าน"
              value={password}
              onChangeText={setPassword}
              secureTextEntry={!showPassword}
            />
            <TouchableOpacity
              onPress={() => setShowPassword(!showPassword)}
              style={styles.eyeButton}
            >
              <Text>{showPassword ? '🙈' : '👁'}</Text>
            </TouchableOpacity>
          </View>

          <TouchableOpacity
            style={[styles.loginButton, isLoading && styles.buttonDisabled]}
            onPress={handleLogin}
            disabled={isLoading}
          >
            {isLoading ? (
              <ActivityIndicator color="#fff" />
            ) : (
              <Text style={styles.loginButtonText}>เข้าสู่ระบบ</Text>
            )}
          </TouchableOpacity>

          <TouchableOpacity onPress={() => navigation.navigate('ForgotPassword')}>
            <Text style={styles.forgotPassword}>ลืมรหัสผ่าน?</Text>
          </TouchableOpacity>
        </View>

        <View style={styles.registerContainer}>
          <Text style={styles.registerText}>ยังไม่มีบัญชี? </Text>
          <TouchableOpacity onPress={() => navigation.navigate('Register')}>
            <Text style={styles.registerLink}>สมัครสมาชิก</Text>
          </TouchableOpacity>
        </View>
      </View>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  content: {
    flex: 1,
    justifyContent: 'center',
    paddingHorizontal: 32,
  },
  logo: {
    fontSize: 42,
    fontWeight: 'bold',
    color: '#1DA1F2',
    textAlign: 'center',
    marginBottom: 8,
  },
  tagline: {
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
    marginBottom: 40,
  },
  form: { gap: 16 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 12,
    paddingHorizontal: 16,
    paddingVertical: 14,
    fontSize: 16,
    backgroundColor: '#f9f9f9',
  },
  passwordContainer: {
    flexDirection: 'row',
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 12,
    backgroundColor: '#f9f9f9',
    alignItems: 'center',
  },
  passwordInput: {
    flex: 1,
    paddingHorizontal: 16,
    paddingVertical: 14,
    fontSize: 16,
  },
  eyeButton: { paddingHorizontal: 16 },
  loginButton: {
    backgroundColor: '#1DA1F2',
    borderRadius: 12,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonDisabled: { opacity: 0.7 },
  loginButtonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  forgotPassword: {
    color: '#1DA1F2',
    textAlign: 'center',
    fontSize: 14,
  },
  registerContainer: {
    flexDirection: 'row',
    justifyContent: 'center',
    marginTop: 32,
  },
  registerText: { color: '#666', fontSize: 14 },
  registerLink: { color: '#1DA1F2', fontSize: 14, fontWeight: '600' },
});

export default LoginScreen;
```

### Home Screen

```typescript
// mobile/src/screens/home/HomeScreen.tsx
import React, { useEffect, useCallback } from 'react';
import {
  FlatList,
  RefreshControl,
  StyleSheet,
  View,
  Text,
  ActivityIndicator,
  ListRenderItemInfo,
} from 'react-native';
import { usePostsStore } from '../../store/postsStore';
import PostCard from '../../components/post/PostCard';
import { useSocket } from '../../hooks/useSocket';

const HomeScreen: React.FC = () => {
  const { feed, isLoading, isRefreshing, hasMore, fetchFeed } = usePostsStore();
  const { socket } = useSocket();

  useEffect(() => {
    fetchFeed(true);
  }, []);

  useEffect(() => {
    if (!socket) return;

    socket.on('new_post', ({ post }: any) => {
      usePostsStore.getState().addPost(post);
    });

    return () => {
      socket.off('new_post');
    };
  }, [socket]);

  const handleRefresh = useCallback(() => {
    fetchFeed(true);
  }, []);

  const handleLoadMore = useCallback(() => {
    if (!isLoading && hasMore) {
      fetchFeed(false);
    }
  }, [isLoading, hasMore]);

  const renderItem = useCallback(
    ({ item }: ListRenderItemInfo<any>) => <PostCard post={item} />,
    []
  );

  const renderFooter = () => {
    if (!isLoading) return null;
    return (
      <View style={styles.loadingFooter}>
        <ActivityIndicator color="#1DA1F2" />
      </View>
    );
  };

  const renderEmpty = () => {
    if (isLoading) return null;
    return (
      <View style={styles.emptyContainer}>
        <Text style={styles.emptyText}>ยังไม่มีโพสต์</Text>
        <Text style={styles.emptySubtext}>ติดตามคนอื่นเพื่อดูโพสต์ของพวกเขา</Text>
      </View>
    );
  };

  return (
    <View style={styles.container}>
      <FlatList
        data={feed}
        renderItem={renderItem}
        keyExtractor={(item) => item._id}
        onEndReached={handleLoadMore}
        onEndReachedThreshold={0.5}
        ListFooterComponent={renderFooter}
        ListEmptyComponent={renderEmpty}
        refreshControl={
          <RefreshControl
            refreshing={isRefreshing}
            onRefresh={handleRefresh}
            tintColor="#1DA1F2"
          />
        }
        showsVerticalScrollIndicator={false}
        contentContainerStyle={styles.content}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f0f0f0' },
  content: { paddingVertical: 8 },
  loadingFooter: { paddingVertical: 16, alignItems: 'center' },
  emptyContainer: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingTop: 80,
  },
  emptyText: { fontSize: 18, fontWeight: '600', color: '#333' },
  emptySubtext: { fontSize: 14, color: '#888', marginTop: 8 },
});

export default HomeScreen;
```

### Post Card Component

```typescript
// mobile/src/components/post/PostCard.tsx
import React, { memo } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  Dimensions,
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { usePostsStore } from '../../store/postsStore';
import { formatRelativeTime } from '../../utils/formatters';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface Props {
  post: {
    _id: string;
    author: {
      _id: string;
      username: string;
      displayName: string;
      avatar?: string;
      isVerified: boolean;
    };
    content: string;
    media: Array<{ url: string; type: string }>;
    likesCount: number;
    commentsCount: number;
    isLiked: boolean;
    createdAt: string;
  };
}

const PostCard: React.FC<Props> = memo(({ post }) => {
  const navigation = useNavigation<any>();
  const { likePost } = usePostsStore();

  return (
    <View style={styles.card}>
      {/* Header */}
      <TouchableOpacity
        style={styles.header}
        onPress={() => navigation.navigate('Profile', { userId: post.author._id })}
      >
        <Image
          source={
            post.author.avatar
              ? { uri: post.author.avatar }
              : require('../../assets/default-avatar.png')
          }
          style={styles.avatar}
        />
        <View style={styles.authorInfo}>
          <View style={styles.nameRow}>
            <Text style={styles.displayName}>{post.author.displayName}</Text>
            {post.author.isVerified && <Text style={styles.verifiedBadge}>✓</Text>}
          </View>
          <Text style={styles.username}>@{post.author.username}</Text>
        </View>
        <Text style={styles.time}>{formatRelativeTime(post.createdAt)}</Text>
      </TouchableOpacity>

      {/* Content */}
      <TouchableOpacity
        onPress={() => navigation.navigate('PostDetail', { postId: post._id })}
      >
        <Text style={styles.content}>{post.content}</Text>
      </TouchableOpacity>

      {/* Media */}
      {post.media.length > 0 && (
        <View style={styles.mediaContainer}>
          {post.media.slice(0, 4).map((item, index) => (
            <Image
              key={index}
              source={{ uri: item.url }}
              style={[
                styles.mediaItem,
                post.media.length === 1 && styles.singleMedia,
                post.media.length > 1 && styles.gridMedia,
              ]}
              resizeMode="cover"
            />
          ))}
        </View>
      )}

      {/* Actions */}
      <View style={styles.actions}>
        <TouchableOpacity
          style={styles.actionButton}
          onPress={() => likePost(post._id)}
        >
          <Text style={[styles.actionIcon, post.isLiked && styles.likedIcon]}>
            {post.isLiked ? '❤️' : '🤍'}
          </Text>
          <Text style={styles.actionCount}>{post.likesCount}</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={styles.actionButton}
          onPress={() => navigation.navigate('PostDetail', { postId: post._id })}
        >
          <Text style={styles.actionIcon}>💬</Text>
          <Text style={styles.actionCount}>{post.commentsCount}</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.actionButton}>
          <Text style={styles.actionIcon}>🔁</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.actionButton}>
          <Text style={styles.actionIcon}>📤</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
});

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    marginHorizontal: 0,
    marginBottom: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
  },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 10,
  },
  avatar: { width: 44, height: 44, borderRadius: 22 },
  authorInfo: { flex: 1, marginLeft: 10 },
  nameRow: { flexDirection: 'row', alignItems: 'center', gap: 4 },
  displayName: { fontWeight: '600', fontSize: 15, color: '#000' },
  verifiedBadge: { color: '#1DA1F2', fontSize: 14 },
  username: { color: '#888', fontSize: 13 },
  time: { color: '#aaa', fontSize: 13 },
  content: { fontSize: 15, color: '#333', lineHeight: 22, marginBottom: 10 },
  mediaContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 2,
    marginBottom: 10,
    borderRadius: 12,
    overflow: 'hidden',
  },
  mediaItem: { borderRadius: 8 },
  singleMedia: {
    width: SCREEN_WIDTH - 32,
    height: 240,
  },
  gridMedia: {
    width: (SCREEN_WIDTH - 36) / 2,
    height: 160,
  },
  actions: {
    flexDirection: 'row',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    paddingTop: 10,
    gap: 24,
  },
  actionButton: { flexDirection: 'row', alignItems: 'center', gap: 6 },
  actionIcon: { fontSize: 20 },
  likedIcon: { fontSize: 20 },
  actionCount: { color: '#888', fontSize: 14 },
});

export default PostCard;
```

---

## Part 8: Real-time และ Push Notifications

### Socket Hook

```typescript
// mobile/src/hooks/useSocket.ts
import { useEffect, useRef, useState } from 'react';
import { io, Socket } from 'socket.io-client';
import { useAuthStore } from '../store/authStore';

const SOCKET_URL = __DEV__
  ? 'http://localhost:3000'
  : 'https://api.socialrn.com';

export const useSocket = () => {
  const socketRef = useRef<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const { accessToken, isAuthenticated } = useAuthStore();

  useEffect(() => {
    if (!isAuthenticated || !accessToken) {
      socketRef.current?.disconnect();
      return;
    }

    const socket = io(SOCKET_URL, {
      auth: { token: accessToken },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
    });

    socket.on('connect', () => {
      setIsConnected(true);
      console.log('Socket connected');
    });

    socket.on('disconnect', () => {
      setIsConnected(false);
    });

    socket.on('connect_error', (error) => {
      console.error('Socket error:', error.message);
    });

    socketRef.current = socket;

    return () => {
      socket.disconnect();
      socketRef.current = null;
    };
  }, [isAuthenticated, accessToken]);

  return { socket: socketRef.current, isConnected };
};
```

### Push Notifications Hook

```typescript
// mobile/src/hooks/usePushNotifications.ts
import { useEffect } from 'react';
import { Platform, Alert } from 'react-native';
import messaging from '@react-native-firebase/messaging';
import { useAuthStore } from '../store/authStore';
import { apiClient } from '../api/client';
import { useNavigation } from '@react-navigation/native';

export const usePushNotifications = () => {
  const { isAuthenticated } = useAuthStore();
  const navigation = useNavigation<any>();

  useEffect(() => {
    if (!isAuthenticated) return;

    const setupNotifications = async () => {
      // Request permission
      const authStatus = await messaging().requestPermission();
      const enabled =
        authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
        authStatus === messaging.AuthorizationStatus.PROVISIONAL;

      if (!enabled) return;

      // Get FCM token
      const fcmToken = await messaging().getToken();
      if (fcmToken) {
        await apiClient.patch('/users/me/fcm-token', { fcmToken });
      }

      // Listen for token refresh
      const unsubscribeToken = messaging().onTokenRefresh(async (token) => {
        await apiClient.patch('/users/me/fcm-token', { fcmToken: token });
      });

      return unsubscribeToken;
    };

    const unsubscribeSetup = setupNotifications();

    // Foreground notification
    const unsubscribeForeground = messaging().onMessage(async (remoteMessage) => {
      Alert.alert(
        remoteMessage.notification?.title || 'การแจ้งเตือน',
        remoteMessage.notification?.body,
        [
          { text: 'ปิด' },
          {
            text: 'ดู',
            onPress: () => handleNotificationTap(remoteMessage.data),
          },
        ]
      );
    });

    // Background/Quit notification tap
    const unsubscribeOpened = messaging().onNotificationOpenedApp(
      (remoteMessage) => {
        if (remoteMessage.data) {
          handleNotificationTap(remoteMessage.data);
        }
      }
    );

    // Check initial notification (app opened from quit state)
    messaging()
      .getInitialNotification()
      .then((remoteMessage) => {
        if (remoteMessage?.data) {
          handleNotificationTap(remoteMessage.data);
        }
      });

    return () => {
      unsubscribeSetup.then((fn) => fn?.());
      unsubscribeForeground();
      unsubscribeOpened();
    };
  }, [isAuthenticated]);

  const handleNotificationTap = (data: any) => {
    if (!data) return;
    switch (data.type) {
      case 'like':
      case 'comment':
      case 'mention':
        if (data.postId) {
          navigation.navigate('PostDetail', { postId: data.postId });
        }
        break;
      case 'follow':
        if (data.senderId) {
          navigation.navigate('Profile', { userId: data.senderId });
        }
        break;
      default:
        navigation.navigate('Notifications');
    }
  };
};
```

---

## Part 9: Navigation

```typescript
// mobile/src/navigation/AppNavigator.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useAuthStore } from '../store/authStore';
import AuthNavigator from './AuthNavigator';
import MainNavigator from './MainNavigator';
import SplashScreen from '../screens/SplashScreen';

const Stack = createNativeStackNavigator();

const AppNavigator: React.FC = () => {
  const { isAuthenticated } = useAuthStore();

  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="Main" component={MainNavigator} />
        ) : (
          <Stack.Screen name="Auth" component={AuthNavigator} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default AppNavigator;
```

```typescript
// mobile/src/navigation/MainNavigator.tsx
import React from 'react';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import HomeScreen from '../screens/home/HomeScreen';
import SearchScreen from '../screens/search/SearchScreen';
import CreatePostScreen from '../screens/post/CreatePostScreen';
import NotificationsScreen from '../screens/notifications/NotificationsScreen';
import ProfileScreen from '../screens/profile/ProfileScreen';
import PostDetailScreen from '../screens/post/PostDetailScreen';
import { Text, TouchableOpacity, StyleSheet } from 'react-native';
import { usePostsStore } from '../store/postsStore';

const Tab = createBottomTabNavigator();
const Stack = createNativeStackNavigator();

const HomeStack: React.FC = () => (
  <Stack.Navigator>
    <Stack.Screen
      name="Home"
      component={HomeScreen}
      options={{ title: 'หน้าหลัก' }}
    />
    <Stack.Screen
      name="PostDetail"
      component={PostDetailScreen}
      options={{ title: 'โพสต์' }}
    />
    <Stack.Screen
      name="Profile"
      component={ProfileScreen}
      options={{ title: 'โปรไฟล์' }}
    />
  </Stack.Navigator>
);

const MainNavigator: React.FC = () => {
  return (
    <Tab.Navigator
      screenOptions={{
        tabBarActiveTintColor: '#1DA1F2',
        tabBarInactiveTintColor: '#888',
        tabBarShowLabel: false,
        headerShown: false,
      }}
    >
      <Tab.Screen
        name="HomeTab"
        component={HomeStack}
        options={{ tabBarIcon: ({ color }) => <Text style={{ fontSize: 24, color }}>🏠</Text> }}
      />
      <Tab.Screen
        name="SearchTab"
        component={SearchScreen}
        options={{ tabBarIcon: ({ color }) => <Text style={{ fontSize: 24, color }}>🔍</Text> }}
      />
      <Tab.Screen
        name="CreateTab"
        component={CreatePostScreen}
        options={{
          tabBarIcon: () => (
            <TouchableOpacity style={styles.createButton}>
              <Text style={styles.createIcon}>+</Text>
            </TouchableOpacity>
          ),
        }}
      />
      <Tab.Screen
        name="NotificationsTab"
        component={NotificationsScreen}
        options={{ tabBarIcon: ({ color }) => <Text style={{ fontSize: 24, color }}>🔔</Text> }}
      />
      <Tab.Screen
        name="ProfileTab"
        component={ProfileScreen}
        options={{ tabBarIcon: ({ color }) => <Text style={{ fontSize: 24, color }}>👤</Text> }}
      />
    </Tab.Navigator>
  );
};

const styles = StyleSheet.create({
  createButton: {
    width: 50,
    height: 50,
    backgroundColor: '#1DA1F2',
    borderRadius: 25,
    alignItems: 'center',
    justifyContent: 'center',
    marginBottom: 4,
  },
  createIcon: { color: '#fff', fontSize: 28, fontWeight: '300' },
});

export default MainNavigator;
```

---

## Part 10: Deployment

### Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGODB_URI=${MONGODB_URI}
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - REFRESH_SECRET=${REFRESH_SECRET}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - S3_BUCKET=${S3_BUCKET}
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - /etc/letsencrypt:/etc/letsencrypt
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  redis_data:
```

### Dockerfile (Backend)

```dockerfile
# backend/Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/app.js"]
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy Backend

on:
  push:
    branches: [main]
    paths:
      - 'backend/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Run tests
        working-directory: backend
        run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd backend
          docker build -t $ECR_REGISTRY/socialrn-backend:$IMAGE_TAG .
          docker push $ECR_REGISTRY/socialrn-backend:$IMAGE_TAG

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster socialrn-prod \
            --service backend \
            --force-new-deployment
```

---

## Workshop: ขยาย App

### Exercise 1: Stories Feature
สร้าง Stories ที่หายไปใน 24 ชั่วโมง เหมือน Instagram Stories

```typescript
// ความคิดพื้นฐาน
interface Story {
  _id: string;
  author: Author;
  media: { url: string; type: 'image' | 'video' };
  duration: number; // seconds
  viewers: string[];
  expiresAt: Date; // +24h from createdAt
}
```

### Exercise 2: Direct Messages
สร้าง DM ด้วย Socket.io

```typescript
// Socket events ที่ต้องเพิ่ม
socket.on('send_dm', ({ recipientId, content }) => { ... });
socket.on('dm_received', ({ message }) => { ... });
```

### Exercise 3: Trending Hashtags
```typescript
// Aggregate trending hashtags
const trending = await Post.aggregate([
  { $match: { createdAt: { $gte: new Date(Date.now() - 24*60*60*1000) } } },
  { $unwind: '$hashtags' },
  { $group: { _id: '$hashtags', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 },
]);
```

---

## สรุปสิ่งที่เรียนทั้งคอร์ส

ตลอดทั้ง 100 Parts เราได้เรียนรู้:

| Part | หัวข้อ | ความสำคัญ |
|------|--------|----------|
| 001-010 | React Native พื้นฐาน | Foundation |
| 011-020 | Navigation, State | Core Skills |
| 021-040 | Networking, Storage | Real Apps |
| 041-060 | Native Modules, Performance | Advanced |
| 061-080 | Architecture, Testing | Professional |
| 081-085 | Camera, Maps, AR | Modern Features |
| 086-090 | BLE, NFC, Security | Enterprise |
| 091-095 | Animation, Design System | Polish |
| 096-099 | Microservices, DevOps, ASO | Production |
| 100 | Full Project | Mastery |

> **ยินดีด้วย! คุณสำเร็จ React Native Course แล้ว 🎉**
> ตอนนี้คุณพร้อมสร้าง App ระดับ Production และส่ง Store ได้แล้ว!
