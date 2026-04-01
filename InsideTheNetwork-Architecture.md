# InsideTheNetwork — Full-Stack Architecture & Developer Guide

## Project Overview

InsideTheNetwork is a production-grade, tech-focused community forum for cybersecurity, programming, networking, and ethical hacking professionals. This document covers the complete system architecture, database schema, API design, code implementations, and deployment guide.

---

## 1. Project Structure

```
InsideTheNetwork/
├── apps/
│   ├── web/                          # Next.js 14 Frontend
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── (main)/
│   │   │   │   ├── layout.tsx        # Dashboard shell
│   │   │   │   ├── feed/page.tsx
│   │   │   │   ├── categories/page.tsx
│   │   │   │   ├── threads/[id]/page.tsx
│   │   │   │   ├── profile/[username]/page.tsx
│   │   │   │   └── admin/page.tsx
│   │   │   ├── api/                  # Next.js API routes (BFF layer)
│   │   │   └── layout.tsx
│   │   ├── components/
│   │   │   ├── ui/                   # Shadcn/ui + custom
│   │   │   ├── forum/
│   │   │   │   ├── ThreadCard.tsx
│   │   │   │   ├── ThreadEditor.tsx  # Markdown editor
│   │   │   │   ├── CommentTree.tsx   # Nested comments
│   │   │   │   └── VoteButton.tsx
│   │   │   ├── admin/
│   │   │   └── shared/
│   │   ├── lib/
│   │   │   ├── api-client.ts         # Axios instance + interceptors
│   │   │   ├── auth.ts               # NextAuth.js config
│   │   │   ├── socket.ts             # Socket.io client
│   │   │   └── markdown.ts           # Marked + DOMPurify config
│   │   ├── hooks/
│   │   │   ├── useThreads.ts
│   │   │   ├── useSocket.ts
│   │   │   └── useAuth.ts
│   │   └── tailwind.config.ts
│   │
│   └── api/                          # Node.js + Express Backend
│       ├── src/
│       │   ├── app.ts                # Express app setup
│       │   ├── server.ts             # HTTP + WS server
│       │   ├── config/
│       │   │   ├── database.ts       # Prisma client
│       │   │   ├── redis.ts          # Rate limiting + caching
│       │   │   └── environment.ts    # Zod-validated env vars
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   │   ├── auth.controller.ts
│       │   │   │   ├── auth.service.ts
│       │   │   │   ├── auth.middleware.ts
│       │   │   │   └── auth.routes.ts
│       │   │   ├── users/
│       │   │   ├── threads/
│       │   │   ├── comments/
│       │   │   ├── categories/
│       │   │   ├── votes/
│       │   │   ├── notifications/
│       │   │   ├── search/
│       │   │   ├── bookmarks/
│       │   │   ├── admin/
│       │   │   └── moderation/       # AI moderation
│       │   ├── shared/
│       │   │   ├── middleware/
│       │   │   │   ├── rateLimit.ts
│       │   │   │   ├── helmet.ts
│       │   │   │   ├── validate.ts   # Zod validation
│       │   │   │   └── errorHandler.ts
│       │   │   ├── guards/
│       │   │   │   ├── jwt.guard.ts
│       │   │   │   └── roles.guard.ts
│       │   │   └── utils/
│       │   └── sockets/
│       │       ├── gateway.ts        # Socket.io setup
│       │       └── events/
│       │           ├── notifications.ts
│       │           └── presence.ts
│       └── prisma/
│           └── schema.prisma
│
├── packages/
│   ├── shared-types/                 # Shared TypeScript types
│   └── ui-kit/                       # Shared component library
│
├── infrastructure/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   ├── nginx/
│   │   └── nginx.conf
│   └── k8s/                          # Kubernetes manifests
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
└── turbo.json                        # Turborepo monorepo config
```

---

## 2. Database Schema (PostgreSQL via Prisma)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ===== USERS =====
model User {
  id            String    @id @default(cuid())
  username      String    @unique
  email         String    @unique
  passwordHash  String?   // null for OAuth users
  avatarUrl     String?
  bio           String?
  skills        String[]  // Array of skill tags
  role          Role      @default(USER)
  reputation    Int       @default(0)
  isBanned      Boolean   @default(false)
  banReason     String?
  emailVerified Boolean   @default(false)
  oauthProvider String?   // "github" | "google"
  oauthId       String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  // Relations
  threads       Thread[]
  comments      Comment[]
  votes         Vote[]
  notifications Notification[]  @relation("NotifRecipient")
  bookmarks     Bookmark[]
  sessions      Session[]
  auditLogs     AuditLog[]

  @@index([email])
  @@index([username])
}

enum Role {
  USER
  MODERATOR
  ADMIN
}

// ===== SESSIONS =====
model Session {
  id        String   @id @default(cuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  ipAddress String?
  userAgent String?
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([token])
  @@index([userId])
}

// ===== CATEGORIES =====
model Category {
  id          String   @id @default(cuid())
  name        String   @unique
  slug        String   @unique
  description String
  icon        String?
  color       String?  // hex color for UI
  threadCount Int      @default(0)
  order       Int      @default(0)
  createdAt   DateTime @default(now())

  threads Thread[]

  @@index([slug])
}

// ===== THREADS =====
model Thread {
  id           String      @id @default(cuid())
  title        String
  body         String      // Markdown content
  bodyHtml     String      // Sanitized HTML (cached)
  slug         String      @unique
  isPinned     Boolean     @default(false)
  isLocked     Boolean     @default(false)
  isDeleted    Boolean     @default(false)
  deletedAt    DateTime?
  viewCount    Int         @default(0)
  voteScore    Int         @default(0)  // Cached score
  commentCount Int         @default(0)  // Cached count
  isFlagged    Boolean     @default(false)
  flagReason   String?
  aiScore      Float?      // Toxicity score from AI moderation
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt

  // Relations
  authorId    String
  categoryId  String
  author      User        @relation(fields: [authorId], references: [id])
  category    Category    @relation(fields: [categoryId], references: [id])
  comments    Comment[]
  votes       Vote[]
  bookmarks   Bookmark[]
  tags        Tag[]       @relation("ThreadTags")

  @@index([authorId])
  @@index([categoryId])
  @@index([createdAt])
  @@index([voteScore])
  // Full-text search index
  @@index([title, body], map: "thread_search_idx", type: GIN)
}

// ===== COMMENTS =====
model Comment {
  id        String    @id @default(cuid())
  body      String    // Markdown
  bodyHtml  String    // Sanitized HTML
  isDeleted Boolean   @default(false)
  isFlagged Boolean   @default(false)
  voteScore Int       @default(0)
  depth     Int       @default(0)  // Nesting level (max 3)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt

  // Relations
  authorId  String
  threadId  String
  parentId  String?   // null = top-level comment

  author    User      @relation(fields: [authorId], references: [id])
  thread    Thread    @relation(fields: [threadId], references: [id], onDelete: Cascade)
  parent    Comment?  @relation("Replies", fields: [parentId], references: [id])
  replies   Comment[] @relation("Replies")
  votes     Vote[]

  @@index([threadId])
  @@index([authorId])
  @@index([parentId])
}

// ===== VOTES =====
model Vote {
  id        String    @id @default(cuid())
  value     Int       // +1 or -1
  createdAt DateTime  @default(now())

  userId    String
  threadId  String?
  commentId String?

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  thread    Thread?   @relation(fields: [threadId], references: [id], onDelete: Cascade)
  comment   Comment?  @relation(fields: [commentId], references: [id], onDelete: Cascade)

  // A user can only vote once per thread/comment
  @@unique([userId, threadId])
  @@unique([userId, commentId])
}

// ===== TAGS =====
model Tag {
  id        String   @id @default(cuid())
  name      String   @unique
  slug      String   @unique
  useCount  Int      @default(0)

  threads   Thread[] @relation("ThreadTags")

  @@index([slug])
}

// ===== NOTIFICATIONS =====
model Notification {
  id        String           @id @default(cuid())
  type      NotificationType
  title     String
  body      String
  isRead    Boolean          @default(false)
  linkUrl   String?
  createdAt DateTime         @default(now())

  recipientId String
  recipient   User   @relation("NotifRecipient", fields: [recipientId], references: [id], onDelete: Cascade)

  @@index([recipientId, isRead])
  @@index([createdAt])
}

enum NotificationType {
  REPLY
  MENTION
  UPVOTE
  BADGE_EARNED
  MOD_ACTION
  SYSTEM
}

// ===== BOOKMARKS =====
model Bookmark {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())

  userId   String
  threadId String

  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  thread Thread @relation(fields: [threadId], references: [id], onDelete: Cascade)

  @@unique([userId, threadId])
}

// ===== AUDIT LOG =====
model AuditLog {
  id        String   @id @default(cuid())
  action    String
  targetId  String?
  targetType String?
  details   Json?
  ipAddress String?
  createdAt DateTime @default(now())

  userId String
  user   User   @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([createdAt])
}
```

---

## 3. Authentication System

### JWT + OAuth Implementation

```typescript
// apps/api/src/modules/auth/auth.service.ts

import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { prisma } from '../../config/database';
import { redis } from '../../config/redis';
import { AppError } from '../../shared/utils/AppError';

const JWT_SECRET = process.env.JWT_SECRET!;
const JWT_EXPIRES_IN = '15m';       // Short-lived access token
const REFRESH_EXPIRES_IN = '7d';    // Long-lived refresh token

export interface JWTPayload {
  userId: string;
  role: string;
  sessionId: string;
}

export class AuthService {
  
  /**
   * Register a new user with email/password
   * Validates uniqueness, hashes password, sends verification email
   */
  async register(email: string, username: string, password: string) {
    // Check uniqueness
    const existing = await prisma.user.findFirst({
      where: { OR: [{ email }, { username }] }
    });
    if (existing) throw new AppError('Email or username already taken', 409);

    // Hash password with bcrypt (cost factor 12)
    const passwordHash = await bcrypt.hash(password, 12);

    // Create user
    const user = await prisma.user.create({
      data: { email, username, passwordHash },
      select: { id: true, username: true, email: true, role: true }
    });

    // Issue tokens
    return this.issueTokens(user.id, user.role);
  }

  /**
   * Login with email/password
   */
  async login(email: string, password: string) {
    const user = await prisma.user.findUnique({
      where: { email },
      select: { id: true, passwordHash: true, role: true, isBanned: true }
    });

    if (!user || !user.passwordHash) throw new AppError('Invalid credentials', 401);
    if (user.isBanned) throw new AppError('Account suspended', 403);

    // Constant-time password comparison (prevents timing attacks)
    const valid = await bcrypt.compare(password, user.passwordHash);
    if (!valid) throw new AppError('Invalid credentials', 401);

    return this.issueTokens(user.id, user.role);
  }

  /**
   * OAuth callback handler (GitHub/Google)
   */
  async handleOAuthCallback(
    provider: 'github' | 'google',
    oauthId: string,
    email: string,
    username: string,
    avatarUrl?: string
  ) {
    // Upsert user — create on first OAuth login, update on subsequent
    const user = await prisma.user.upsert({
      where: { email },
      create: {
        email,
        username: await this.ensureUniqueUsername(username),
        avatarUrl,
        oauthProvider: provider,
        oauthId,
        emailVerified: true, // OAuth emails are pre-verified
      },
      update: { avatarUrl, oauthId, oauthProvider: provider },
      select: { id: true, role: true }
    });

    return this.issueTokens(user.id, user.role);
  }

  /**
   * Issue access + refresh token pair
   * Session stored in DB for revocation support
   */
  private async issueTokens(userId: string, role: string) {
    // Create session record
    const session = await prisma.session.create({
      data: {
        userId,
        token: crypto.randomUUID(),
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      }
    });

    const payload: JWTPayload = { userId, role, sessionId: session.id };

    const accessToken = jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRES_IN });
    const refreshToken = jwt.sign(
      { sessionId: session.id },
      JWT_SECRET,
      { expiresIn: REFRESH_EXPIRES_IN }
    );

    return { accessToken, refreshToken, expiresIn: 900 };
  }

  /**
   * Refresh access token using refresh token
   */
  async refreshToken(refreshToken: string) {
    const decoded = jwt.verify(refreshToken, JWT_SECRET) as { sessionId: string };

    const session = await prisma.session.findUnique({
      where: { id: decoded.sessionId },
      include: { user: { select: { id: true, role: true, isBanned: true } } }
    });

    if (!session || session.expiresAt < new Date()) throw new AppError('Session expired', 401);
    if (session.user.isBanned) throw new AppError('Account suspended', 403);

    return this.issueTokens(session.user.id, session.user.role);
  }

  /**
   * Logout — invalidate session and blacklist token in Redis
   */
  async logout(sessionId: string, accessToken: string) {
    await prisma.session.delete({ where: { id: sessionId } });
    // Blacklist the access token until it expires (15 min TTL)
    await redis.setEx(`blacklist:${accessToken}`, 900, '1');
  }

  private async ensureUniqueUsername(base: string): Promise<string> {
    let username = base.replace(/[^a-zA-Z0-9_]/g, '_').slice(0, 20);
    let count = 0;
    while (await prisma.user.findUnique({ where: { username } })) {
      username = `${base}_${++count}`;
    }
    return username;
  }
}
```

### JWT Middleware

```typescript
// apps/api/src/shared/guards/jwt.guard.ts

import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { redis } from '../../config/redis';
import { JWTPayload } from '../../modules/auth/auth.service';
import { AppError } from '../utils/AppError';

declare global {
  namespace Express {
    interface Request {
      user?: JWTPayload;
    }
  }
}

export async function jwtGuard(req: Request, _res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) throw new AppError('Authentication required', 401);

  // Check token blacklist (for logout invalidation)
  const blacklisted = await redis.get(`blacklist:${token}`);
  if (blacklisted) throw new AppError('Token revoked', 401);

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;
    req.user = payload;
    next();
  } catch {
    throw new AppError('Invalid or expired token', 401);
  }
}

export function requireRole(...roles: string[]) {
  return (req: Request, _res: Response, next: NextFunction) => {
    if (!req.user) throw new AppError('Unauthenticated', 401);
    if (!roles.includes(req.user.role)) throw new AppError('Insufficient permissions', 403);
    next();
  };
}
```

---

## 4. Thread System

### Thread Service

```typescript
// apps/api/src/modules/threads/threads.service.ts

import { prisma } from '../../config/database';
import { AppError } from '../../shared/utils/AppError';
import { marked } from 'marked';
import DOMPurify from 'isomorphic-dompurify';
import slugify from 'slugify';
import { ModerationService } from '../moderation/moderation.service';
import { NotificationService } from '../notifications/notifications.service';
import { SearchService } from '../search/search.service';

export class ThreadsService {
  constructor(
    private moderation: ModerationService,
    private notifications: NotificationService,
    private search: SearchService,
  ) {}

  /**
   * Create a new thread with AI moderation, markdown processing, and search indexing
   */
  async createThread(
    authorId: string,
    data: {
      title: string;
      body: string;
      categoryId: string;
      tags: string[];
    }
  ) {
    // 1. AI Moderation — flag toxic content before saving
    const moderationResult = await this.moderation.analyzeContent(
      `${data.title}\n\n${data.body}`
    );

    // 2. Process markdown → sanitized HTML
    const bodyHtml = DOMPurify.sanitize(marked.parse(data.body) as string, {
      ALLOWED_TAGS: ['p','strong','em','code','pre','blockquote','ul','ol','li','a','h2','h3','h4'],
      ALLOWED_ATTR: ['href', 'rel'],
    });

    // 3. Generate unique slug
    const slug = await this.generateUniqueSlug(data.title);

    // 4. Upsert tags
    const tagRecords = await Promise.all(
      data.tags.slice(0, 5).map(name =>
        prisma.tag.upsert({
          where: { slug: slugify(name, { lower: true }) },
          create: { name, slug: slugify(name, { lower: true }), useCount: 1 },
          update: { useCount: { increment: 1 } }
        })
      )
    );

    // 5. Create thread
    const thread = await prisma.thread.create({
      data: {
        title: data.title,
        body: data.body,
        bodyHtml,
        slug,
        authorId,
        categoryId: data.categoryId,
        isFlagged: moderationResult.isToxic,
        flagReason: moderationResult.reason,
        aiScore: moderationResult.score,
        tags: { connect: tagRecords.map(t => ({ id: t.id })) }
      },
      include: {
        author: { select: { id: true, username: true, avatarUrl: true, role: true } },
        category: true,
        tags: true,
      }
    });

    // 6. Update category thread count
    await prisma.category.update({
      where: { id: data.categoryId },
      data: { threadCount: { increment: 1 } }
    });

    // 7. Index for full-text search
    await this.search.indexThread(thread);

    // 8. Reputation reward
    await this.awardReputation(authorId, 5, 'Created thread');

    return thread;
  }

  /**
   * Get paginated thread feed with filtering and sorting
   */
  async getThreads(options: {
    categoryId?: string;
    tags?: string[];
    sort: 'latest' | 'hot' | 'top' | 'unanswered';
    page: number;
    limit: number;
  }) {
    const { sort, page, limit, categoryId, tags } = options;
    const skip = (page - 1) * limit;

    const where = {
      isDeleted: false,
      isFlagged: false, // Flagged content hidden from public
      ...(categoryId && { categoryId }),
      ...(tags?.length && { tags: { some: { slug: { in: tags } } } }),
    };

    const orderBy = {
      latest: { createdAt: 'desc' as const },
      top:    { voteScore: 'desc' as const },
      hot:    { viewCount: 'desc' as const },      // Could use Wilson score here
      unanswered: { createdAt: 'desc' as const },
    }[sort];

    const whereUnanswered = sort === 'unanswered'
      ? { ...where, commentCount: 0 }
      : where;

    const [threads, total] = await Promise.all([
      prisma.thread.findMany({
        where: whereUnanswered,
        orderBy,
        skip,
        take: limit,
        include: {
          author: { select: { id: true, username: true, avatarUrl: true, reputation: true } },
          category: { select: { id: true, name: true, slug: true, color: true } },
          tags: { select: { id: true, name: true, slug: true } },
          _count: { select: { comments: true, votes: true } },
        }
      }),
      prisma.thread.count({ where: whereUnanswered }),
    ]);

    return {
      threads,
      pagination: {
        total,
        page,
        limit,
        pages: Math.ceil(total / limit),
        hasNext: page * limit < total,
      }
    };
  }

  /**
   * Get single thread with comment tree
   */
  async getThread(slug: string, viewerUserId?: string) {
    const thread = await prisma.thread.findUnique({
      where: { slug, isDeleted: false },
      include: {
        author: { select: { id: true, username: true, avatarUrl: true, reputation: true, role: true } },
        category: true,
        tags: true,
        comments: {
          where: { isDeleted: false, parentId: null, depth: 0 }, // Top-level only
          orderBy: { voteScore: 'desc' },
          take: 50,
          include: {
            author: { select: { id: true, username: true, avatarUrl: true } },
            replies: {
              where: { isDeleted: false },
              orderBy: { createdAt: 'asc' },
              take: 20,
              include: {
                author: { select: { id: true, username: true, avatarUrl: true } },
                replies: { // Level 3 nesting
                  where: { isDeleted: false },
                  take: 10,
                  include: { author: { select: { id: true, username: true } } }
                }
              }
            }
          }
        }
      }
    });

    if (!thread) throw new AppError('Thread not found', 404);

    // Increment view count asynchronously (non-blocking)
    prisma.thread.update({
      where: { id: thread.id },
      data: { viewCount: { increment: 1 } }
    }).catch(() => {}); // Fire and forget

    // Attach viewer's vote if authenticated
    if (viewerUserId) {
      const vote = await prisma.vote.findUnique({
        where: { userId_threadId: { userId: viewerUserId, threadId: thread.id } }
      });
      return { ...thread, viewerVote: vote?.value ?? 0 };
    }

    return thread;
  }

  private async generateUniqueSlug(title: string): Promise<string> {
    const base = slugify(title, { lower: true, strict: true }).slice(0, 60);
    let slug = base;
    let i = 1;
    while (await prisma.thread.findUnique({ where: { slug } })) {
      slug = `${base}-${i++}`;
    }
    return slug;
  }

  private async awardReputation(userId: string, points: number, reason: string) {
    await prisma.user.update({
      where: { id: userId },
      data: { reputation: { increment: points } }
    });
  }
}
```

---

## 5. Comment System

```typescript
// apps/api/src/modules/comments/comments.service.ts

export class CommentsService {
  /**
   * Add comment or reply to a thread
   * Max depth: 3 levels of nesting
   */
  async createComment(
    authorId: string,
    threadId: string,
    body: string,
    parentId?: string
  ) {
    const thread = await prisma.thread.findUnique({
      where: { id: threadId, isDeleted: false },
      select: { id: true, isLocked: true, authorId: true }
    });

    if (!thread) throw new AppError('Thread not found', 404);
    if (thread.isLocked) throw new AppError('Thread is locked', 403);

    // Validate parent comment and depth
    let depth = 0;
    if (parentId) {
      const parent = await prisma.comment.findUnique({
        where: { id: parentId },
        select: { depth: true, threadId: true }
      });
      if (!parent || parent.threadId !== threadId)
        throw new AppError('Invalid parent comment', 400);
      if (parent.depth >= 3)
        throw new AppError('Maximum nesting depth reached', 400);
      depth = parent.depth + 1;
    }

    // AI moderation
    const moderation = await this.moderation.analyzeContent(body);

    const bodyHtml = DOMPurify.sanitize(marked.parse(body) as string);

    const [comment] = await prisma.$transaction([
      prisma.comment.create({
        data: {
          body, bodyHtml, authorId, threadId, parentId, depth,
          isFlagged: moderation.isToxic,
        },
        include: {
          author: { select: { id: true, username: true, avatarUrl: true } }
        }
      }),
      // Update cached count
      prisma.thread.update({
        where: { id: threadId },
        data: { commentCount: { increment: 1 } }
      })
    ]);

    // Notify thread author if different user
    if (thread.authorId !== authorId) {
      await this.notifications.create({
        recipientId: thread.authorId,
        type: 'REPLY',
        title: 'New reply on your thread',
        body: `${comment.author.username} replied to your thread`,
        linkUrl: `/threads/${threadId}#comment-${comment.id}`
      });
    }

    // Emit real-time event via Socket.io
    await this.events.emit('thread:new-comment', { threadId, comment });

    // Reputation award
    await this.awardReputation(authorId, 2, 'Posted comment');

    return comment;
  }
}
```

---

## 6. Vote System with Reputation

```typescript
// apps/api/src/modules/votes/votes.service.ts

export class VotesService {
  /**
   * Toggle vote on thread or comment (idempotent)
   * Implements Reddit-style voting with reputation cascade
   */
  async vote(
    userId: string,
    targetType: 'thread' | 'comment',
    targetId: string,
    value: 1 | -1
  ) {
    const key = targetType === 'thread'
      ? { userId_threadId: { userId, threadId: targetId } }
      : { userId_commentId: { userId, commentId: targetId } };

    const existing = await prisma.vote.findUnique({ where: key });

    // Get content author for reputation
    let contentAuthorId: string;
    if (targetType === 'thread') {
      const t = await prisma.thread.findUnique({ where: { id: targetId }, select: { authorId: true } });
      if (!t) throw new AppError('Thread not found', 404);
      if (t.authorId === userId) throw new AppError('Cannot vote on own content', 400);
      contentAuthorId = t.authorId;
    } else {
      const c = await prisma.comment.findUnique({ where: { id: targetId }, select: { authorId: true } });
      if (!c) throw new AppError('Comment not found', 404);
      if (c.authorId === userId) throw new AppError('Cannot vote on own content', 400);
      contentAuthorId = c.authorId;
    }

    let scoreDelta = 0;
    let reputationDelta = 0;

    if (!existing) {
      // New vote
      await prisma.vote.create({
        data: {
          userId,
          value,
          ...(targetType === 'thread' ? { threadId: targetId } : { commentId: targetId })
        }
      });
      scoreDelta = value;
      reputationDelta = value * (targetType === 'thread' ? 10 : 2);

    } else if (existing.value === value) {
      // Same vote = unvote (toggle off)
      await prisma.vote.delete({ where: { id: existing.id } });
      scoreDelta = -value;
      reputationDelta = -(value * (targetType === 'thread' ? 10 : 2));

    } else {
      // Flip vote (+1 → -1 or vice versa)
      await prisma.vote.update({ where: { id: existing.id }, data: { value } });
      scoreDelta = value * 2; // e.g., was -1, now +1 = +2 delta
      reputationDelta = value * 2 * (targetType === 'thread' ? 10 : 2);
    }

    // Update cached score on target
    const updateData = { voteScore: { increment: scoreDelta } };
    if (targetType === 'thread') {
      await prisma.thread.update({ where: { id: targetId }, data: updateData });
    } else {
      await prisma.comment.update({ where: { id: targetId }, data: updateData });
    }

    // Award/deduct reputation from content author
    await prisma.user.update({
      where: { id: contentAuthorId },
      data: { reputation: { increment: reputationDelta } }
    });

    return { scoreDelta };
  }
}
```

---

## 7. AI Moderation

```typescript
// apps/api/src/modules/moderation/moderation.service.ts

import OpenAI from 'openai';
import { redis } from '../../config/redis';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export interface ModerationResult {
  isToxic: boolean;
  score: number;      // 0-1 toxicity confidence
  reason?: string;
  categories: string[];
}

export class ModerationService {
  /**
   * Dual-layer moderation:
   * 1. OpenAI Moderation API (fast, free)
   * 2. Custom GPT-4o prompt for nuanced tech content
   */
  async analyzeContent(content: string): Promise<ModerationResult> {
    // Cache results to avoid re-processing identical content
    const cacheKey = `mod:${Buffer.from(content).toString('base64').slice(0, 32)}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    try {
      // Layer 1: OpenAI Moderation (fast, cheap)
      const modResponse = await openai.moderations.create({ input: content });
      const result = modResponse.results[0];

      const flaggedCategories = Object.entries(result.categories)
        .filter(([_, flagged]) => flagged)
        .map(([cat]) => cat);

      let isToxic = result.flagged;
      let score = Math.max(...Object.values(result.category_scores));
      let reason = flaggedCategories.join(', ') || undefined;

      // Layer 2: GPT-4o for context-aware analysis
      // (only for borderline cases 0.3 < score < 0.7)
      if (score > 0.3 && score < 0.7) {
        const chat = await openai.chat.completions.create({
          model: 'gpt-4o-mini',
          messages: [{
            role: 'system',
            content: `You are a content moderator for a cybersecurity forum. 
            Technical security discussions are allowed (CVEs, pentesting, CTFs).
            Flag: doxxing, selling illegal access, personal attacks, actual malware code.
            Do NOT flag: security research, ethical hacking discussion, vulnerability analysis.
            Respond ONLY with JSON: {"toxic": boolean, "reason": "string or null"}`
          }, {
            role: 'user',
            content: content.slice(0, 2000) // Limit token usage
          }],
          response_format: { type: 'json_object' },
          max_tokens: 100,
        });

        const gptResult = JSON.parse(chat.choices[0].message.content || '{}');
        isToxic = gptResult.toxic ?? isToxic;
        reason = gptResult.reason || reason;
      }

      const finalResult: ModerationResult = {
        isToxic,
        score,
        reason,
        categories: flaggedCategories,
      };

      // Cache for 1 hour
      await redis.setEx(cacheKey, 3600, JSON.stringify(finalResult));

      return finalResult;

    } catch (error) {
      // Fail open — don't block posts if AI moderation is down
      console.error('Moderation service error:', error);
      return { isToxic: false, score: 0, categories: [] };
    }
  }
}
```

---

## 8. WebSocket Real-Time Gateway

```typescript
// apps/api/src/sockets/gateway.ts

import { Server } from 'socket.io';
import { createServer } from 'http';
import jwt from 'jsonwebtoken';
import { JWTPayload } from '../modules/auth/auth.service';

export function setupSocketGateway(httpServer: ReturnType<typeof createServer>) {
  const io = new Server(httpServer, {
    cors: { origin: process.env.FRONTEND_URL, credentials: true },
    transports: ['websocket', 'polling'],
  });

  // JWT authentication middleware for Socket.io
  io.use(async (socket, next) => {
    const token = socket.handshake.auth.token;
    if (!token) return next(new Error('Authentication required'));

    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;
      socket.data.user = payload;
      next();
    } catch {
      next(new Error('Invalid token'));
    }
  });

  io.on('connection', (socket) => {
    const user = socket.data.user as JWTPayload;
    console.log(`[WS] ${user.userId} connected`);

    // Join personal notification room
    socket.join(`user:${user.userId}`);

    // Join thread room for live updates
    socket.on('thread:join', (threadId: string) => {
      socket.join(`thread:${threadId}`);
    });

    socket.on('thread:leave', (threadId: string) => {
      socket.leave(`thread:${threadId}`);
    });

    // Handle live typing indicators
    socket.on('thread:typing', (threadId: string) => {
      socket.to(`thread:${threadId}`).emit('thread:user-typing', {
        userId: user.userId,
      });
    });

    socket.on('disconnect', () => {
      io.emit('user:offline', { userId: user.userId });
    });
  });

  return io;
}

// Usage in services: inject io to emit events
// io.to(`user:${recipientId}`).emit('notification:new', notification);
// io.to(`thread:${threadId}`).emit('comment:new', comment);
```

---

## 9. API Routes

```typescript
// apps/api/src/app.ts — Route registration

import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';
import { createClient } from 'redis';
import { RedisStore } from 'rate-limit-redis';
import { authRoutes } from './modules/auth/auth.routes';
import { threadRoutes } from './modules/threads/threads.routes';
import { commentRoutes } from './modules/comments/comments.routes';
import { voteRoutes } from './modules/votes/votes.routes';
import { searchRoutes } from './modules/search/search.routes';
import { adminRoutes } from './modules/admin/admin.routes';
import { errorHandler } from './shared/middleware/errorHandler';

const app = express();

// ===== SECURITY MIDDLEWARE =====
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    }
  },
  hsts: { maxAge: 31536000, includeSubDomains: true },
}));

app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
}));

// Global rate limiter (Redis-backed)
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 300,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
});
app.use(globalLimiter);

// Strict limiter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,    // Max 10 login attempts per 15 min
  message: { error: 'Too many authentication attempts' },
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
});

app.use(express.json({ limit: '64kb' })); // Prevent oversized payloads
app.use(express.urlencoded({ extended: false }));

// ===== ROUTES =====
app.use('/api/v1/auth',          authLimiter, authRoutes);
app.use('/api/v1/threads',       threadRoutes);
app.use('/api/v1/comments',      commentRoutes);
app.use('/api/v1/votes',         voteRoutes);
app.use('/api/v1/search',        searchRoutes);
app.use('/api/v1/admin',         adminRoutes);
app.use('/api/v1/notifications', notificationRoutes);
app.use('/api/v1/users',         userRoutes);
app.use('/api/v1/bookmarks',     bookmarkRoutes);

// Error handler (last middleware)
app.use(errorHandler);

export { app };
```

### API Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/v1/auth/register` | - | Register user |
| POST | `/api/v1/auth/login` | - | Login (returns JWT pair) |
| POST | `/api/v1/auth/refresh` | - | Refresh access token |
| POST | `/api/v1/auth/logout` | JWT | Logout + invalidate session |
| GET  | `/api/v1/auth/oauth/github` | - | GitHub OAuth redirect |
| GET  | `/api/v1/auth/oauth/google` | - | Google OAuth redirect |
| GET  | `/api/v1/threads` | - | List threads (paginated) |
| POST | `/api/v1/threads` | JWT | Create thread |
| GET  | `/api/v1/threads/:slug` | optional | Get thread + comments |
| PUT  | `/api/v1/threads/:id` | JWT+Owner | Edit thread |
| DELETE | `/api/v1/threads/:id` | JWT+Owner/Mod | Delete thread |
| POST | `/api/v1/threads/:id/comments` | JWT | Add comment |
| POST | `/api/v1/votes/thread/:id` | JWT | Vote on thread |
| POST | `/api/v1/votes/comment/:id` | JWT | Vote on comment |
| GET  | `/api/v1/search?q=...` | - | Full-text search |
| GET  | `/api/v1/users/:username` | - | Public profile |
| PUT  | `/api/v1/users/me` | JWT | Update own profile |
| GET  | `/api/v1/bookmarks` | JWT | Get user bookmarks |
| POST | `/api/v1/bookmarks/:threadId` | JWT | Toggle bookmark |
| GET  | `/api/v1/notifications` | JWT | Get notifications |
| PUT  | `/api/v1/notifications/read-all` | JWT | Mark all read |
| GET  | `/api/v1/admin/users` | JWT+Admin | List all users |
| PATCH | `/api/v1/admin/users/:id/ban` | JWT+Admin | Ban user |
| PATCH | `/api/v1/admin/users/:id/role` | JWT+Admin | Change role |
| GET  | `/api/v1/admin/moderation/queue` | JWT+Mod | Flagged content queue |
| GET  | `/api/v1/categories` | - | All categories |

---

## 10. Full-Text Search

```typescript
// apps/api/src/modules/search/search.service.ts

export class SearchService {
  /**
   * PostgreSQL full-text search using tsvector/tsquery
   * Falls back to ILIKE for simple matches
   */
  async search(query: string, options: { type?: 'all' | 'threads' | 'users'; limit: number }) {
    const sanitized = query.trim().replace(/[^a-zA-Z0-9 _-]/g, '').slice(0, 100);
    if (!sanitized) return { threads: [], users: [] };

    const threads = await prisma.$queryRaw<any[]>`
      SELECT
        t.id, t.title, t.slug, t.vote_score, t.view_count, t.created_at,
        ts_headline('english', t.body, plainto_tsquery('english', ${sanitized}),
          'MaxWords=20, MinWords=10') as excerpt,
        ts_rank(
          to_tsvector('english', t.title || ' ' || t.body),
          plainto_tsquery('english', ${sanitized})
        ) as rank
      FROM threads t
      WHERE
        t.is_deleted = false AND t.is_flagged = false AND
        to_tsvector('english', t.title || ' ' || t.body) @@ plainto_tsquery('english', ${sanitized})
      ORDER BY rank DESC, t.vote_score DESC
      LIMIT ${options.limit}
    `;

    return { threads };
  }

  async indexThread(thread: any) {
    // The tsvector index is automatically updated by PostgreSQL triggers
    // This method is a hook for external search engines (Elasticsearch) if needed
  }
}
```

---

## 11. Docker & Deployment

### docker-compose.yml (Development)

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: insidethenetwork
      POSTGRES_USER: itn_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U itn_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  api:
    build:
      context: ./apps/api
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://itn_user:${POSTGRES_PASSWORD}@postgres:5432/insidethenetwork
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      JWT_SECRET: ${JWT_SECRET}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      FRONTEND_URL: http://localhost:3000
    ports:
      - "4000:4000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./apps/api:/app
      - /app/node_modules
    command: npm run dev

  web:
    build:
      context: ./apps/web
      dockerfile: Dockerfile
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:4000/api/v1
      NEXT_PUBLIC_WS_URL: ws://localhost:4000
      NEXTAUTH_URL: http://localhost:3000
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      GITHUB_ID: ${GITHUB_CLIENT_ID}
      GITHUB_SECRET: ${GITHUB_CLIENT_SECRET}
      GOOGLE_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_SECRET: ${GOOGLE_CLIENT_SECRET}
    ports:
      - "3000:3000"
    depends_on:
      - api
    volumes:
      - ./apps/web:/app
      - /app/node_modules
    command: npm run dev

volumes:
  postgres_data:
  redis_data:
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: itn_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run type-check
      - run: npm run lint
      - run: npm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/itn_test

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push API
        uses: docker/build-push-action@v5
        with:
          context: ./apps/api
          push: true
          tags: ghcr.io/${{ github.repository }}/api:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Build and push Web
        uses: docker/build-push-action@v5
        with:
          context: ./apps/web
          push: true
          tags: ghcr.io/${{ github.repository }}/web:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/insidethenetwork
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d --no-build
            docker compose -f docker-compose.prod.yml exec api npx prisma migrate deploy
```

---

## 12. Environment Variables

```bash
# .env (copy to .env.local, never commit)

# Database
DATABASE_URL="postgresql://itn_user:PASSWORD@localhost:5432/insidethenetwork"

# Cache
REDIS_URL="redis://:PASSWORD@localhost:6379"
REDIS_PASSWORD="your-redis-password"

# Auth
JWT_SECRET="your-256-bit-secret-here-use-openssl-rand-base64-32"
NEXTAUTH_SECRET="your-nextauth-secret"
NEXTAUTH_URL="https://yourdomain.com"

# OAuth
GITHUB_CLIENT_ID="your-github-app-client-id"
GITHUB_CLIENT_SECRET="your-github-app-client-secret"
GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-client-secret"

# AI Moderation
OPENAI_API_KEY="sk-..."

# Email (Resend or SendGrid)
EMAIL_API_KEY="re_..."
EMAIL_FROM="noreply@insidethenetwork.io"

# App
NODE_ENV="development"
PORT=4000
FRONTEND_URL="http://localhost:3000"
```

---

## 13. Quick Start

```bash
# 1. Clone and install dependencies
git clone https://github.com/your-org/InsideTheNetwork.git
cd InsideTheNetwork
npm install  # Turborepo installs all workspaces

# 2. Configure environment
cp .env.example .env
# Edit .env with your values

# 3. Start infrastructure
docker compose up postgres redis -d

# 4. Database setup
cd apps/api
npx prisma migrate dev --name init
npx prisma db seed  # Seeds categories, sample users

# 5. Start development servers (runs both web + api)
cd ../../
npm run dev

# App: http://localhost:3000
# API: http://localhost:4000
# API Docs: http://localhost:4000/api-docs (Swagger)

# Production
docker compose -f docker-compose.prod.yml up -d
```

---

## Security Checklist

- [x] JWT with short expiry (15min) + refresh tokens stored in httpOnly cookies
- [x] Token blacklisting via Redis for instant logout
- [x] bcrypt with cost factor 12 for password hashing
- [x] Rate limiting on all endpoints, strict on auth routes
- [x] Helmet.js with strict CSP headers
- [x] Input validation with Zod on all API inputs
- [x] SQL injection prevention via Prisma parameterized queries
- [x] XSS prevention via DOMPurify on markdown rendering
- [x] CORS configured to specific origin only
- [x] Payload size limits (64kb) to prevent DoS
- [x] Audit logging for all admin actions
- [x] AI content moderation with dual-layer analysis
- [x] OAuth PKCE flow for GitHub/Google
- [x] Non-verbose error messages in production (stack traces hidden)
