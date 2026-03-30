# Prisma 数据库配置说明

> **完成时间**: 2026-03-30
> **对应阶段**: 第一阶段 - 任务 2
> **目标**: 完成 Prisma ORM 配置与数据库模型定义

---

## 一、安装 Prisma 依赖

### 1.1 安装命令

```bash
npm install prisma @prisma/client
npm install -D prisma
```

### 1.2 初始化 Prisma

```bash
npx prisma init
```

**生成的文件**：

- `prisma/schema.prisma` - 数据库模型定义文件
- `.env` - 环境变量文件（包含数据库连接字符串）

---

## 二、配置数据库连接

### 2.1 编辑 `.env` 文件

在项目根目录的 `.env` 文件中配置数据库连接字符串：

```env
DATABASE_URL="postgresql://postgres:password@localhost:5432/ai_writing?schema=public"
```

**连接字符串说明**：

- 用户名：`postgres`
- 密码：`password`
- 主机：`localhost`
- 端口：`5432`
- 数据库名：`ai_writing`
- Schema：`public`

---

## 三、定义 Prisma Schema

### 3.1 版本问题处理

**遇到的问题**：初始安装的 Prisma 7.6.0 版本改变了配置方式，`datasource` 中的 `url` 属性不再支持。

**解决方案**：降级到稳定的 Prisma 5.22.0 版本

```bash
npm uninstall prisma @prisma/client
npm install prisma@5.22.0 @prisma/client@5.22.0
```

### 3.2 Schema 文件结构

完整的 `prisma/schema.prisma` 文件包含以下部分：

#### 3.2.1 Generator 配置

```prisma
generator client {
  provider = "prisma-client-js"
}
```

#### 3.2.2 Datasource 配置

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

#### 3.2.3 枚举类型定义（4个）

```prisma
enum Role {
  OWNER
  EDITOR
  VIEWER
}

enum CollaboratorStatus {
  ACTIVE
  REMOVED
}

enum InviteStatus {
  PENDING
  ACCEPTED
  EXPIRED
  REVOKED
}

enum GenerationStatus {
  PENDING
  GENERATING
  COMPLETED
  FAILED
}
```

#### 3.2.4 数据模型定义（8个）

##### 1. User（用户表）

```prisma
model User {
  id             String         @id @default(cuid())
  email          String         @unique
  name           String?
  avatarUrl      String?
  password       String?
  documents      Document[]
  collaborations Collaborator[]
  sentInvites    Invite[]
  createdAt      DateTime       @default(now())
  updatedAt      DateTime       @updatedAt
}
```

**字段说明**：

- `id`: 主键，使用 cuid 生成
- `email`: 邮箱，唯一索引
- `name`: 用户名（可选）
- `avatarUrl`: 头像 URL（可选）
- `password`: 密码哈希（可选，OAuth 用户可能没有）
- 关联关系：拥有的文档、参与的协作、发送的邀请

##### 2. Document（文档表）

```prisma
model Document {
  id             String            @id @default(cuid())
  title          String
  ydoc           Bytes?
  ownerId        String
  owner          User              @relation(fields: [ownerId], references: [id], onDelete: Cascade)

  // 房间权限配置:允许编辑的角色列表
  editableRoles  Role[]            @default([OWNER, EDITOR])

  collaborators  Collaborator[]
  versions       DocumentVersion[]
  shareLinks     ShareLink[]
  aiGenerations  AIGeneration[]
  invites        Invite[]
  inviteLinks    InviteLink[]

  createdAt      DateTime          @default(now())
  updatedAt      DateTime          @updatedAt

  @@index([ownerId])
}
```

**关键字段**：

- `ydoc`: Yjs 文档的二进制数据（Bytes 类型）
- `editableRoles`: 数组字段，存储允许编辑的角色列表
- `onDelete: Cascade`: 级联删除，删除用户时自动删除其文档

##### 3. Collaborator（协作者表）

```prisma
model Collaborator {
  id         String              @id @default(cuid())
  documentId String
  document   Document            @relation(fields: [documentId], references: [id], onDelete: Cascade)
  userId     String
  user       User                @relation(fields: [userId], references: [id], onDelete: Cascade)
  role       Role
  status     CollaboratorStatus  @default(ACTIVE)
  removedBy  String?
  removedAt  DateTime?
  createdAt  DateTime            @default(now())

  @@unique([documentId, userId])
  @@index([documentId])
  @@index([userId])
}
```

**关键设计**：

- `@@unique([documentId, userId])`: 确保同一用户在同一文档中只有一条记录
- `status`: 协作者状态（ACTIVE/REMOVED）
- `removedBy`: 记录是谁踢出的该成员
- 多个索引优化查询性能

##### 4. Invite（邮箱邀请表）

```prisma
model Invite {
  id         String       @id @default(cuid())
  documentId String
  document   Document     @relation(fields: [documentId], references: [id], onDelete: Cascade)
  inviterId  String
  inviter    User         @relation(fields: [inviterId], references: [id], onDelete: Cascade)
  email      String
  role       Role
  token      String       @unique
  status     InviteStatus @default(PENDING)
  expiresAt  DateTime?
  createdAt  DateTime     @default(now())

  @@index([documentId])
  @@index([token])
}
```

**用途**：通过邮箱邀请用户加入文档协作

##### 5. InviteLink（链接邀请表）

```prisma
model InviteLink {
  id         String    @id @default(cuid())
  documentId String
  document   Document  @relation(fields: [documentId], references: [id], onDelete: Cascade)
  token      String    @unique
  role       Role
  maxUses    Int?
  usedCount  Int       @default(0)
  expiresAt  DateTime?
  createdAt  DateTime  @default(now())

  @@index([documentId])
  @@index([token])
}
```

**用途**：生成可分享的邀请链接，支持设置最大使用次数和过期时间

##### 6. ShareLink（分享链接表）

```prisma
model ShareLink {
  id         String    @id @default(cuid())
  documentId String
  document   Document  @relation(fields: [documentId], references: [id], onDelete: Cascade)
  token      String    @unique
  role       Role
  expiresAt  DateTime?
  createdAt  DateTime  @default(now())

  @@index([documentId])
  @@index([token])
}
```

**用途**：生成文档分享链接（只读或可编辑）

##### 7. DocumentVersion（文档版本表）

```prisma
model DocumentVersion {
  id          String   @id @default(cuid())
  documentId  String
  document    Document @relation(fields: [documentId], references: [id], onDelete: Cascade)
  snapshot    Bytes
  stateVector Bytes
  label       String?
  createdBy   String?
  createdAt   DateTime @default(now())

  @@index([documentId])
  @@index([createdAt])
}
```

**关键字段**：

- `snapshot`: Yjs 文档快照（二进制）
- `stateVector`: Yjs 状态向量（用于增量同步）
- `label`: 版本标签（可选）

##### 8. AIGeneration（AI生成记录表）

```prisma
model AIGeneration {
  id          String           @id @default(cuid())
  documentId  String
  document    Document         @relation(fields: [documentId], references: [id], onDelete: Cascade)
  prompt      String           @db.Text
  outline     Json?
  status      GenerationStatus
  result      String?          @db.Text
  createdAt   DateTime         @default(now())
  completedAt DateTime?

  @@index([documentId])
  @@index([status])
}
```

**关键字段**：

- `prompt`: 用户输入的提示词（大文本）
- `outline`: 生成的大纲（JSON 格式）
- `result`: 生成的内容（大文本）
- `@db.Text`: 指定使用 PostgreSQL 的 TEXT 类型（无长度限制）

---

## 四、验证与生成

### 4.1 格式化并验证 Schema

```bash
npx prisma format
```

**输出**：

```
Prisma schema loaded from prisma\schema.prisma
Formatted prisma\schema.prisma in 20ms 🚀
```

### 4.2 生成 Prisma Client

```bash
npx prisma generate
```

**输出**：

```
✔ Generated Prisma Client (v5.22.0) to .\node_modules\@prisma\client in 64ms
```

---

## 五、创建 Prisma Client 单例

### 5.1 创建 lib 目录

```bash
mkdir -p lib
```

### 5.2 创建 `lib/prisma.ts` 文件

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

**设计说明**：

- 在开发环境中使用全局变量缓存 Prisma Client 实例
- 避免热重载时创建多个数据库连接
- 生产环境每次都创建新实例

---

## 六、数据库设计亮点

### 6.1 权限系统设计

- **三级角色**：OWNER（所有者）、EDITOR（编辑者）、VIEWER（观察者）
- **动态权限配置**：`Document.editableRoles` 数组字段允许 Owner 动态调整哪些角色可以编辑
- **协作者状态管理**：支持踢出成员并记录操作人

### 6.2 邀请系统设计

- **双重邀请方式**：
  - 邮箱邀请（Invite）：精确邀请指定用户
  - 链接邀请（InviteLink）：生成可分享链接，支持使用次数限制

### 6.3 版本控制设计

- 基于 Yjs 的快照机制
- 存储 `snapshot`（完整快照）和 `stateVector`（状态向量）
- 支持版本对比和回滚

### 6.4 性能优化

- **索引策略**：
  - 外键字段添加索引（如 `documentId`、`userId`）
  - 查询频繁的字段添加索引（如 `token`、`status`、`createdAt`）
- **级联删除**：使用 `onDelete: Cascade` 保证数据一致性
- **唯一约束**：防止重复数据（如 `email`、`token`、`[documentId, userId]`）

---

## 七、后续步骤

1. ✅ Prisma Schema 定义完成
2. ✅ Prisma Client 生成完成
3. ⏳ 等待 Docker Compose 启动 PostgreSQL 后执行数据库迁移
4. ⏳ 配置 NextAuth.js 认证系统

---

## 八、注意事项

### 8.1 数据库迁移

当前只是定义了 Schema，还没有创建实际的数据库表。需要在 PostgreSQL 启动后执行：

```bash
npx prisma migrate dev --name init
```

### 8.2 Prisma Studio

可以使用 Prisma Studio 可视化管理数据库：

```bash
npx prisma studio
```

### 8.3 类型安全

Prisma Client 提供完整的 TypeScript 类型支持，所有数据库操作都是类型安全的。

---

## 九、前后端数据交互说明（重要概念）

### 9.1 核心问题：前端如何获取数据库内容？

**关键概念**：在 Next.js 架构中，**前端（浏览器）不能直接连接数据库**。

#### ❌ 错误理解
- 浏览器中的 JavaScript 代码无法直接连接 PostgreSQL
- 数据库凭证（用户名、密码）不能暴露给浏览器
- `lib/prisma.ts` 只能在**服务端**运行

#### ✅ 正确的数据流向

```
浏览器（前端）
    ↓ HTTP 请求
Next.js 服务器（后端）
    ↓ 使用 lib/prisma.ts
PostgreSQL 数据库
```

---

### 9.2 Next.js 中的三种数据获取方式

#### 方式 1：API Routes（传统方式）

**后端**：创建 API 端点

```typescript
// app/api/documents/route.ts
import { prisma } from '@/lib/prisma'
import { NextResponse } from 'next/server'

export async function GET() {
  // 这段代码在服务器上运行
  const documents = await prisma.document.findMany()
  return NextResponse.json(documents)
}
```

**前端**：通过 fetch 调用

```typescript
// app/documents/page.tsx (客户端组件)
'use client'

export default function DocumentsPage() {
  const [documents, setDocuments] = useState([])

  useEffect(() => {
    // 浏览器发送 HTTP 请求到服务器
    fetch('/api/documents')
      .then(res => res.json())
      .then(data => setDocuments(data))
  }, [])

  return <div>{/* 渲染文档列表 */}</div>
}
```

---

#### 方式 2：Server Components（推荐，Next.js 15 新特性）

**直接在服务端组件中查询数据库**：

```typescript
// app/documents/page.tsx (服务端组件，默认)
import { prisma } from '@/lib/prisma'

// 这个组件在服务器上渲染
export default async function DocumentsPage() {
  // 直接查询数据库，不需要 API 路由
  const documents = await prisma.document.findMany()

  // 返回的 HTML 已经包含数据
  return (
    <div>
      {documents.map(doc => (
        <div key={doc.id}>{doc.title}</div>
      ))}
    </div>
  )
}
```

**优势**：
- 不需要额外的 API 端点
- 更快的首屏加载（服务端渲染）
- 代码更简洁

---

#### 方式 3：Server Actions（推荐，用于表单提交和数据修改）

**定义服务端操作**：

```typescript
// app/actions/documents.ts
'use server'

import { prisma } from '@/lib/prisma'

export async function createDocument(title: string) {
  // 这段代码在服务器上运行
  const document = await prisma.document.create({
    data: { title, ownerId: 'user-id' }
  })
  return document
}
```

**前端调用**：

```typescript
// app/documents/new/page.tsx
'use client'

import { createDocument } from '@/app/actions/documents'

export default function NewDocumentPage() {
  async function handleSubmit(formData: FormData) {
    const title = formData.get('title') as string
    // 直接调用服务端函数，Next.js 自动处理 RPC
    await createDocument(title)
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      <button type="submit">创建</button>
    </form>
  )
}
```

---

### 9.3 本项目推荐的混合方式

根据项目需求，建议采用以下策略：

#### 1. 数据读取：使用 Server Components

```typescript
// app/dashboard/page.tsx
import { prisma } from '@/lib/prisma'
import { auth } from '@/lib/auth' // NextAuth

export default async function Dashboard() {
  const session = await auth()

  // 服务端直接查询
  const documents = await prisma.document.findMany({
    where: { ownerId: session.user.id },
    include: { collaborators: true }
  })

  return <DocumentList documents={documents} />
}
```

#### 2. 数据修改：使用 Server Actions

```typescript
// app/actions/documents.ts
'use server'

import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function updateDocument(id: string, title: string) {
  await prisma.document.update({
    where: { id },
    data: { title }
  })

  // 刷新页面缓存
  revalidatePath('/dashboard')
}
```

#### 3. 实时功能：使用 API Routes（SSE、WebSocket）

```typescript
// app/api/documents/[id]/generate/route.ts
import { prisma } from '@/lib/prisma'

export async function POST(req: Request) {
  const { prompt } = await req.json()

  // 保存生成记录到数据库
  const generation = await prisma.aIGeneration.create({
    data: {
      documentId: 'xxx',
      prompt,
      status: 'PENDING'
    }
  })

  // 转发到 AI 服务...
}
```

---

### 9.4 关键要点总结

| 问题 | 答案 |
|------|------|
| 前端如何获取数据库内容？ | 前端**不直接**连接数据库，通过 Next.js 服务器作为中间层 |
| lib/prisma.ts 的作用？ | 只在**服务端代码**中使用（Server Components、API Routes、Server Actions） |
| 前端如何使用？ | Server Components：直接 `import { prisma }`<br>Client Components：通过 API 或 Server Actions 间接访问 |
| 数据库连接配置？ | 从 `.env` 文件读取 `DATABASE_URL`，只在服务器端可见，浏览器永远看不到 |

---

## 十、相关文件清单

```
e:\project\my-ai-writing\
├── prisma/
│   └── schema.prisma          # 数据库模型定义（完整）
├── lib/
│   └── prisma.ts              # Prisma Client 单例
├── .env                       # 环境变量（包含 DATABASE_URL）
└── package.json               # 依赖配置（已添加 prisma 和 @prisma/client）
```

---

**文档版本**: v1.1
**最后更新**: 2026-03-30（新增第九章：前后端数据交互说明）
