# ZaloCRM - Tài liệu Dự án

> **Phiên bản:** 3.4.0  
> **Cập nhật:** 2026-09-14  
> **Giấy phép:** AGPL-3.0 (dual-license thương mại)

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Kiến trúc hệ thống](#2-kiến-trúc-hệ-thống)
3. [Công nghệ sử dụng](#3-công-nghệ-sử-dụng)
4. [Cấu trúc dự án](#4-cấu-trúc-dự-án)
5. [Database Schema](#5-database-schema)
6. [API Endpoints](#6-api-endpoints)
7. [Frontend Structure](#7-frontend-structure)
8. [Module Backend](#8-module-backend)
9. [Quy ước code](#9-quy-ước-code)
10. [Hướng dẫn phát triển](#10-hướng-dẫn-phát-triển)

---

## 1. Tổng quan

### 1.1 Giới thiệu

**ZaloCRM** là hệ thống CRM mã nguồn mở quản lý nhiều tài khoản Zalo cá nhân trên một giao diện web duy nhất. Dự án được phát triển bởi Nguyễn Tiến Lộc và phát hành theo giấy phép AGPL-3.0.

### 1.2 Tính năng chính

| Module | Mô tả |
|--------|--------|
| **Quản lý Zalo** | Kết nối nhiều nick Zalo, QR login, auto-reconnect |
| **Chat Real-time** | Nhắn tin 2 chiều với khách hàng qua Socket.IO |
| **CRM** | Quản lý Contact, Pipeline, Lead Scoring |
| **Lịch hẹn** | Tạo, theo dõi, nhắc nhở cuộc hẹn |
| **AI Assistant** | Gợi ý trả lời, tóm tắt, phân tích cảm xúc |
| **Báo cáo** | Dashboard, Analytics, Export Excel |
| **Automation** | Workflow tự động, Sequence, Broadcast |
| **Integrations** | Telegram Bridge, Facebook Lead Ads, Google Sheets, Zapier |
| **Media** | Kho ảnh/video/file với MinIO/S3/R2 |
| **RBAC** | Phân quyền Owner/Admin/Member, Department, Permission Groups |

### 1.3 Docker Services

```
┌─────────────────────────────────────────────────────────────┐
│                     ZaloCRM Stack                           │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│     App     │     DB      │    Redis    │      MinIO      │
│   (Node.js) │ (PostgreSQL)│  (Cache/Queue) │   (Storage)   │
├─────────────┴─────────────┴─────────────┴─────────────────┤
│                    ClamAV (Antivirus - Optional)           │
│                    Backup (Daily PostgreSQL)               │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Kiến trúc hệ thống

### 2.1 Sơ đồ tổng quan

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (Browser)                         │
│   ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐  │
│   │  Vue 3 + Vuetify │  │  Pinia (State)   │  │  Socket.IO │  │
│   │     Frontend     │  │                  │  │   Client   │  │
│   └────────┬─────────┘  └──────────────────┘  └─────┬──────┘  │
└────────────┼────────────────────────────────────────────┼───────┘
             │ REST API                                      │ WebSocket
             ▼                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Backend (Node.js + Fastify)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Auth    │  │  Zalo    │  │  Chat    │  │   Realtime    │  │
│  │  Module  │  │  Module  │  │  Module  │  │  (Socket.IO)  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Contacts │  │ Scoring  │  │   AI     │  │  Integrations │  │
│  │  Module  │  │  Module  │  │  Module  │  │    Module     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │Dashboard │  │Analytics │  │  Media   │  │    RBAC      │  │
│  │  Module  │  │  Module  │  │  Module  │  │   Module     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
             │                         │
             ▼                         ▼
┌───────────────────┐   ┌───────────────────────────────────────┐
│   PostgreSQL 16   │   │              Redis 7                   │
│   (Primary DB)    │   │   (Cache / Event Buffer / BullMQ)     │
└───────────────────┘   └───────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        MinIO / S3 / R2                           │
│                    (Object Storage - Media)                       │
└─────────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Zalo API (External)                            │
│               via zca-js library                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Luồng dữ liệu

1. **Tin nhắn đến:**
   ```
   Zalo User → Zalo Server → zca-js → ZaloCRM Backend → PostgreSQL → Socket.IO → Frontend
   ```

2. **Tin nhắn gửi:**
   ```
   Frontend → REST API → Backend → zca-js → Zalo Server → Zalo User
   ```

3. **Real-time:**
   ```
   Backend Event → Event Buffer (Redis) → Socket.IO → Frontend Update
   ```

---

## 3. Công nghệ sử dụng

### 3.1 Backend

| Thành phần | Công nghệ | Phiên bản |
|------------|------------|------------|
| Runtime | Node.js | 20.x |
| Framework | Fastify | 5.x |
| ORM | Prisma | 7.x |
| Database | PostgreSQL | 16 |
| Cache/Queue | Redis | 7 |
| Realtime | Socket.IO | 4.x |
| Auth | @fastify/jwt | - |
| Validation | - | - |
| Object Storage | MinIO / S3 / R2 | - |
| Zalo Integration | zca-js | 2.x |

### 3.2 Frontend

| Thành phần | Công nghệ | Phiên bản |
|------------|------------|------------|
| Framework | Vue 3 | 3.5.x |
| Build Tool | Vite | 8.x |
| UI Library | Vuetify | 4.x |
| State Management | Pinia | 3.x |
| Router | Vue Router | 4.x |
| HTTP Client | Axios | 1.x |
| Rich Text Editor | TipTap | 3.x |
| Charts | Chart.js + vue-chartjs | 4.x / 5.x |
| i18n | vue-i18n | 11.x |
| PWA | vite-plugin-pwa | 1.x |

### 3.3 Infrastructure

| Thành phần | Mục đích |
|------------|----------|
| Docker Compose | Container orchestration |
| PostgreSQL 16-alpine | Database |
| Redis 7-alpine | Cache, Queue |
| MinIO latest | S3-compatible storage |
| ClamAV 1.4 | Antivirus (optional) |

---

## 4. Cấu trúc dự án

```
ZaloCRM/
├── backend/                      # Node.js + Fastify backend
│   ├── src/
│   │   ├── app.ts              # Main entry point
│   │   ├── config/             # Configuration
│   │   ├── modules/            # Feature modules
│   │   │   ├── auth/          # Authentication
│   │   │   ├── zalo/          # Zalo integration
│   │   │   ├── chat/          # Chat module
│   │   │   ├── contacts/      # Contact management
│   │   │   ├── scoring/       # Lead scoring
│   │   │   ├── ai/            # AI features
│   │   │   ├── dashboard/     # Dashboard & reports
│   │   │   ├── analytics/     # Analytics
│   │   │   ├── media/        # Media management
│   │   │   ├── rbac/         # Role-based access
│   │   │   ├── privacy/      # Privacy features
│   │   │   ├── integrations/  # Third-party integrations
│   │   │   └── ...
│   │   ├── shared/            # Shared utilities
│   │   │   ├── database/     # Prisma client
│   │   │   ├── storage/      # Storage drivers
│   │   │   ├── realtime/     # Socket.IO utilities
│   │   │   ├── security/     # Security utilities
│   │   │   └── ...
│   │   └── _ee/              # Enterprise features (stubs)
│   ├── prisma/
│   │   ├── schema.prisma     # Database schema
│   │   ├── migrations/       # Database migrations
│   │   └── seeds/            # Seed data
│   ├── tests/                # Backend tests
│   └── scripts/              # Utility scripts
│
├── frontend/                    # Vue 3 frontend
│   ├── src/
│   │   ├── main.ts           # Entry point
│   │   ├── App.vue           # Root component
│   │   ├── api/              # API client
│   │   ├── assets/           # Static assets & styles
│   │   ├── components/       # Vue components
│   │   │   ├── chat/        # Chat components
│   │   │   ├── contacts/    # Contact components
│   │   │   ├── dashboard/   # Dashboard components
│   │   │   ├── zalo-accounts/ # Zalo account components
│   │   │   └── ...
│   │   ├── composables/     # Vue composables
│   │   ├── constants/       # Constants
│   │   ├── layouts/         # Layout components
│   │   ├── router/          # Vue Router config
│   │   ├── stores/          # Pinia stores
│   │   ├── views/           # Page components
│   │   │   ├── reports/    # Report pages
│   │   │   ├── settings/   # Settings pages
│   │   │   ├── rbac/       # RBAC pages
│   │   │   ├── marketing/   # Marketing pages
│   │   │   └── ...
│   │   ├── plugins/         # Vue plugins
│   │   └── utils/          # Utilities
│   ├── public/              # Public assets
│   ├── package.json
│   └── vite.config.ts
│
├── docs/                       # Documentation
│   ├── architecture/          # Architecture diagrams
│   ├── zalocrm-api/          # API documentation
│   └── ...
│
├── docker/                    # Docker files
├── scripts/                   # Deployment scripts
├── docker-compose.yml          # Production compose
├── docker-compose.dev.yml      # Dev compose
├── .env.example               # Environment template
└── README.md
```

---

## 5. Database Schema

### 5.1 Core Models

```
┌─────────────────┐
│ Organization    │  ← Tenant (mỗi tổ chức là 1 tenant)
├─────────────────┤
│ id (UUID)       │
│ name            │
│ timezone        │
│ logo_url        │
│ ...             │
└────────┬────────┘
         │
         ├──┬──────────────────┬─────────────────┬──────────────┐
         │  │                  │                 │              │
         ▼  ▼                  ▼                 ▼              ▼
    ┌──────────┐     ┌──────────────┐    ┌─────────────┐   ┌──────────┐
    │   Team   │     │ ZaloAccount │    │   Contact   │   │ User     │
    ├──────────┤     ├──────────────┤    ├─────────────┤   ├──────────┤
    │ id       │     │ id          │    │ id          │   │ id       │
    │ orgId    │     │ orgId       │    │ orgId       │   │ orgId    │
    │ name     │     │ ownerUserId │    │ zaloUid     │   │ email    │
    └──────────┘     │ zaloUid     │    │ fullName    │   │ password │
                     │ status      │    │ phone       │   │ role     │
                     └──────┬──────┘    │ statusId    │   └──────────┘
                            │           └──────┬──────┘
                            │                  │
                            ▼                  ▼
                     ┌──────────────┐   ┌─────────────┐
                     │  Conversation│   │   Friend   │
                     ├──────────────┤   ├─────────────┤
                     │ id           │   │ id          │
                     │ orgId       │   │ orgId       │
                     │ zaloAccountId│   │ contactId   │
                     │ contactId   │   │ zaloAccountId│
                     │ threadType  │   │ zaloUid     │
                     └──────┬──────┘   └─────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   Message    │
                     ├──────────────┤
                     │ id           │
                     │ convId      │
                     │ zaloMsgId   │
                     │ senderType  │
                     │ content     │
                     │ contentType │
                     └─────────────┘
```

### 5.2 Key Models

| Model | Mô tả | Key Fields |
|-------|--------|------------|
| **Organization** | Tenant/công ty | id, name, timezone |
| **User** | Người dùng CRM | id, email, phone, role, permissionGroupId |
| **Team** | Nhóm trong tổ chức | id, orgId, name |
| **ZaloAccount** | Nick Zalo | id, ownerUserId, zaloUid, status, sessionData, privacyMode |
| **Contact** | Khách hàng (Cha) | id, orgId, zaloGlobalId, fullName, phone, leadScore |
| **Friend** | Phiếu chăm sóc (Con) | id, contactId, zaloAccountId, relationshipKind |
| **Conversation** | Hội thoại | id, zaloAccountId, contactId, threadType |
| **Message** | Tin nhắn | id, convId, zaloMsgId, content, contentType, senderType |
| **Appointment** | Lịch hẹn | id, contactId, appointmentDate, status |
| **Status** | Trạng thái KH | id, orgId, name, order, color |
| **CrmTag** | Tag CRM | id, orgId, name, color |
| **ZaloLabel** | Nhãn Zalo | id, orgId, zaloAccountId, name |
| **AiConfig** | Cấu hình AI | id, orgId, provider, model |
| **ScoringConfig** | Cấu hình scoring | id, orgId, weights |
| **AutomationRule** | Quy tắc automation | id, orgId, name, trigger |
| **RefreshToken** | Refresh token | id, userId, tokenHash, familyId |

### 5.3 Important Indexes

```sql
-- Contact lookups
CREATE INDEX contacts_org_phone ON contacts(org_id, phone);
CREATE INDEX contacts_org_zalo_uid ON contacts(org_id, zalo_uid);
CREATE INDEX contacts_org_last_activity ON contacts(org_id, last_activity);

-- Conversation lookups
CREATE INDEX conv_org_acc_reply ON conversations(org_id, zalo_account_id, is_replied, last_message_at);
CREATE INDEX conv_org_tab ON conversations(org_id, tab, last_message_at);

-- Message lookups
CREATE INDEX msg_conv_zalo_msg_id ON messages(conversation_id, zalo_msg_id);
CREATE INDEX msg_conv_time ON messages(conversation_id, sent_at, sender_type);
```

---

## 6. API Endpoints

### 6.1 Authentication

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| POST | `/api/v1/auth/login` | Đăng nhập |
| POST | `/api/v1/auth/refresh` | Refresh token |
| POST | `/api/v1/auth/logout` | Đăng xuất |
| GET | `/api/v1/profile` | Lấy thông tin user |

### 6.2 Zalo Accounts

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/zalo-accounts` | Danh sách nick Zalo |
| POST | `/api/v1/zalo-accounts` | Thêm nick Zalo |
| GET | `/api/v1/zalo-accounts/:id` | Chi tiết nick |
| PUT | `/api/v1/zalo-accounts/:id` | Cập nhật nick |
| DELETE | `/api/v1/zalo-accounts/:id` | Xóa nick |
| POST | `/api/v1/zalo-accounts/:id/proxy` | Cập nhật proxy |
| POST | `/api/v1/zalo-accounts/:id/labels/*` | Quản lý nhãn |

### 6.3 Chat

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/conversations` | Danh sách hội thoại |
| GET | `/api/v1/conversations/:id` | Chi tiết hội thoại |
| GET | `/api/v1/conversations/:id/messages` | Tin nhắn hội thoại |
| POST | `/api/v1/conversations/:id/messages` | Gửi tin nhắn |
| POST | `/api/v1/conversations/:id/attachments` | Gửi file đính kèm |

### 6.4 Contacts

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/contacts` | Danh sách khách hàng |
| POST | `/api/v1/contacts` | Tạo khách hàng |
| GET | `/api/v1/contacts/:id` | Chi tiết khách hàng |
| PUT | `/api/v1/contacts/:id` | Cập nhật khách hàng |
| DELETE | `/api/v1/contacts/:id` | Xóa khách hàng |
| POST | `/api/v1/contacts/:id/merge` | Gộp khách hàng |

### 6.5 Appointments

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/appointments` | Danh sách lịch hẹn |
| POST | `/api/v1/appointments` | Tạo lịch hẹn |
| PUT | `/api/v1/appointments/:id` | Cập nhật lịch hẹn |
| DELETE | `/api/v1/appointments/:id` | Xóa lịch hẹn |

### 6.6 Dashboard & Reports

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/dashboard/stats` | Thống kê dashboard |
| GET | `/api/v1/reports/*` | Các báo cáo khác nhau |
| GET | `/api/v1/analytics/*` | Analytics nâng cao |

### 6.7 AI

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/v1/ai/config` | Lấy cấu hình AI |
| PUT | `/api/v1/ai/config` | Cập nhật cấu hình AI |
| POST | `/api/v1/ai/suggest-reply` | Gợi ý trả lời |
| POST | `/api/v1/ai/summarize` | Tóm tắt hội thoại |
| POST | `/api/v1/ai/sentiment` | Phân tích cảm xúc |

### 6.8 Public API

| Method | Endpoint | Mô tả |
|--------|----------|--------|
| GET | `/api/public/contacts` | Danh sách KH (API Key) |
| POST | `/api/public/contacts` | Tạo KH (API Key) |
| POST | `/api/public/messages/send` | Gửi tin nhắn (API Key) |

---

## 7. Frontend Structure

### 7.1 Router Structure

```typescript
// main routes
/                  → DashboardView
/chat/:convId?     → ChatView
/contacts          → ContactsView
/contacts/:id      → ContactProfileView
/friends           → FriendsView
/appointments      → AppointmentsView
/media             → MediaView
/groups            → GroupsView
/analytics         → AnalyticsView
/reports/*         → ReportsShell (nested routes)
  /reports/tong-quan
  /reports/nick
  /reports/sale
  /reports/pipeline
  /reports/engagement
  /reports/audit
/settings/*        → SettingsLayout (nested routes)
  /settings/personal/profile
  /settings/org/profile
  /settings/rbac/*
  /settings/crm/*
  /settings/channels/*
  /settings/dev/*
/marketing/*       → CommunityMarketingShell
  /marketing/group-scan
  /marketing/lists
```

### 7.2 Pinia Stores

| Store | Mục đích |
|-------|----------|
| `auth` | Authentication state, user info, grants |
| `rbac` | RBAC permissions and access control |
| `privacy` | Privacy mode settings |

### 7.3 Key Composables

| Composable | Mô tả |
|------------|--------|
| `useChat` | Chat state and operations |
| `useContacts` | Contact CRUD operations |
| `useZaloAccounts` | Zalo account management |
| `useDashboard` | Dashboard data fetching |
| `useToast` | Toast notifications |
| `useFriends` | Friend list operations |
| `useAppointments` | Appointment management |
| `useGroups` | Group management |
| `useAnalytics` | Analytics data |
| `useAutomationRules` | Automation rules |
| `useOrgTimezone` | Organization timezone |
| `useZaloPresence` | Zalo online status |

### 7.4 API Layer

```typescript
// frontend/src/api/index.ts
const api = axios.create({
  baseURL: '/api/v1',
  timeout: 30000,
});

// Features:
// - JWT interceptor (auto-add Bearer token)
// - Token refresh (single-flight)
// - 401 → refresh → retry
// - 403 → toast error
// - 5xx → throttle toast
```

---

## 8. Module Backend

### 8.1 Module Map

| Module | Đường dẫn | Mô tả |
|--------|-----------|--------|
| **auth** | `modules/auth/` | Authentication, login, token refresh |
| **rbac** | `modules/rbac/` | Role-based access control |
| **zalo** | `modules/zalo/` | Zalo integration, pool, listener |
| **chat** | `modules/chat/` | Chat routes, message handling |
| **contacts** | `modules/contacts/` | Contact CRUD, merge, intelligence |
| **scoring** | `modules/scoring/` | Lead scoring, signal detection |
| **ai** | `modules/ai/` | AI features, providers |
| **dashboard** | `modules/dashboard/` | Dashboard, reports |
| **analytics** | `modules/analytics/` | Analytics queries |
| **media** | `modules/media/` | Media upload, storage |
| **privacy** | `modules/privacy/` | Privacy features, OTP |
| **integrations** | `modules/integrations/` | Telegram, Facebook, etc. |
| **branding** | `modules/branding/` | Org branding |
| **activity** | `modules/activity/` | Activity logging |
| **notifications** | `modules/notifications/` | Push notifications |
| **search** | `modules/search/` | Global search |
| **campaign** | `modules/campaign/` | Campaign management |
| **tags** | `modules/tags/` | Tag management |
| **lists** | `modules/lists/` | Customer lists |
| **engagement** | `modules/engagement/` | Engagement tracking |
| **system-notifications** | `modules/system-notifications/` | System notifications |

### 8.2 Zalo Module Structure

```
modules/zalo/
├── zalo-pool.ts           # Zalo account pool management
├── zalo-socket.ts        # Socket.IO handlers
├── zalo-routes.ts        # Zalo API routes
├── zalo-sync-routes.ts   # Sync endpoints
├── zalo-labels-routes.ts # Label management
├── zalo-access-middleware.ts # Zalo access control
├── friend-routes.ts      # Friend endpoints
├── friend-sync-service.ts # Friend sync logic
├── friend-event-handler.ts # Friend event processing
├── profile-routes.ts     # Profile endpoints
├── profile-operations.ts  # Profile operations
├── group-routes.ts       # Group endpoints
├── group-scan-*          # Group scanning
├── presence-service.ts    # Online presence
├── status-log-*         # Status logging
├── sdk-limit-*           # SDK rate limiting
└── ...
```

### 8.3 Entry Point (app.ts)

```typescript
// Key startup sequence:
// 1. Fastify setup with plugins (CORS, JWT, rate-limit, multipart)
// 2. Security headers registration
// 3. Socket.IO setup with auth
// 4. Route registration (20+ modules)
// 5. Cron jobs start
// 6. Zalo account reconnect
// 7. Graceful shutdown handling
```

---

## 9. Quy ước code

### 9.1 Backend

#### File Naming
- Routes: `*-routes.ts`
- Services: `*-service.ts`
- Helpers: `*-helpers.ts` hoặc `*.ts` (utilities)
- Types: `types.ts`
- Constants: `constants.ts`

#### Code Patterns

```typescript
// Route handler pattern
export async function routeHandler(
  request: FastifyRequest,
  reply: FastifyReply
) {
  const { orgId, userId } = request.authContext;
  const { param } = request.body as { param: string };
  
  // Validation
  if (!param) {
    return reply.status(400).send({ error: 'param required' });
  }
  
  // Business logic
  const result = await prisma.model.findFirst({
    where: { orgId, id: param }
  });
  
  // Response
  return reply.send({ data: result });
}

// Middleware pattern
export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
) {
  try {
    await request.jwtVerify();
    request.authContext = {
      orgId: request.user.orgId,
      userId: request.user.id,
    };
  } catch (err) {
    return reply.status(401).send({ error: 'Unauthorized' });
  }
}
```

#### Database Patterns

```typescript
// Always filter by orgId for multi-tenant safety
const results = await prisma.contact.findMany({
  where: { orgId },
  include: { conversations: true },
  orderBy: { lastActivity: 'desc' },
  take: limit,
  skip: offset,
});

// Use transactions for multi-step operations
await prisma.$transaction([
  prisma.contact.update({ where: { id }, data: { ... } }),
  prisma.activityLog.create({ data: { ... } }),
]);
```

### 9.2 Frontend

#### File Naming
- Components: `PascalCase.vue`
- Composables: `use-*.ts`
- Utils: `camelCase.ts`
- Stores: `*.ts`

#### Vue Component Structure

```vue
<script setup lang="ts">
// 1. Imports
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useToast } from '@/composables/use-toast';

// 2. Props/Emits
const props = defineProps<{ id: string }>();
const emit = defineEmits<{ (e: 'updated'): void }>();

// 3. State
const loading = ref(false);
const data = ref(null);

// 4. Computed
const isValid = computed(() => !!data.value);

// 5. Methods
async function fetchData() {
  loading.value = true;
  try {
    data.value = await api.get(`/items/${props.id}`);
  } finally {
    loading.value = false;
  }
}

// 6. Lifecycle
onMounted(fetchData);
</script>

<template>
  <v-card v-if="!loading">
    <!-- Content -->
  </v-card>
  <v-skeleton-loader v-else />
</template>
```

#### API Composable Pattern

```typescript
// composables/use-items.ts
export function useItems() {
  const items = ref<Item[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);

  async function fetchItems(params?: QueryParams) {
    loading.value = true;
    error.value = null;
    try {
      const res = await api.get('/items', { params });
      items.value = res.data;
    } catch (e) {
      error.value = e.message;
    } finally {
      loading.value = false;
    }
  }

  return { items, loading, error, fetchItems };
}
```

### 9.3 Git Conventions

```
feat: add new feature
fix: fix bug
refactor: code refactoring
docs: documentation
style: formatting
test: tests
chore: maintenance
perf: performance
security: security fixes
```

### 9.4 Environment Variables

```bash
# Server
PORT=3000
APP_PORT=3080
NODE_ENV=production

# Security (REQUIRED)
JWT_SECRET=<32+ chars hex>
ENCRYPTION_KEY=<64 hex chars>

# Database
DB_USER=crmuser
DB_PASSWORD=<password>
DB_NAME=zalocrm
DATABASE_URL=postgresql://...

# Redis
REDIS_URL=redis://redis:6379

# Storage
STORAGE_DRIVER=local|r2
UPLOAD_DIR=/var/lib/zalo-crm/files
S3_ENDPOINT=
S3_BUCKET=
S3_ACCESS_KEY=
S3_SECRET_KEY=

# AI Providers
AI_DEFAULT_PROVIDER=anthropic
ANTHROPIC_AUTH_TOKEN=
GEMINI_AUTH_TOKEN=
OPENAI_AUTH_TOKEN=

# Integrations
TELEGRAM_BRIDGE_BOT_TOKEN=
FB_APP_ID=
```

---

## 10. Hướng dẫn phát triển

### 10.1 Local Development Setup

```bash
# 1. Clone repo
git clone https://github.com/locphamnguyen/ZaloCRM.git
cd ZaloCRM

# 2. Copy env
cp .env.example .env
# Edit .env with required values

# 3. Start dev environment
docker compose -f docker-compose.dev.yml up -d

# 4. Backend dev (with hot reload)
cd backend
npm install
npm run dev

# 5. Frontend dev (separate terminal)
cd frontend
npm install
npm run dev
```

### 10.2 Database Operations

```bash
# Generate Prisma client
cd backend
npx prisma generate

# Run migrations
npx prisma migrate dev

# Apply migrations to production
docker exec zalo-crm-app npx prisma migrate deploy

# Open Prisma Studio
npx prisma studio
```

### 10.3 Adding New Features

#### Backend

1. **Create route file:**
   ```typescript
   // backend/src/modules/new-feature/new-feature-routes.ts
   import { FastifyInstance } from 'fastify';
   
   export async function newFeatureRoutes(app: FastifyInstance) {
     app.get('/new-feature', async (request, reply) => {
       // Implementation
     });
   }
   ```

2. **Register in app.ts:**
   ```typescript
   import { newFeatureRoutes } from './modules/new-feature/new-feature-routes';
   await app.register(newFeatureRoutes);
   ```

3. **Add to Prisma schema if needed:**
   ```prisma
   // backend/prisma/schema.prisma
   model NewModel {
     id    String @id @default(uuid())
     // fields...
   }
   ```

4. **Run migration:**
   ```bash
   npx prisma migrate dev --name add_new_model
   ```

#### Frontend

1. **Create view:**
   ```vue
   // frontend/src/views/new-feature/NewFeatureView.vue
   <script setup lang="ts">
   // Implementation
   </script>
   
   <template>
     <div>New Feature</div>
   </template>
   ```

2. **Add route:**
   ```typescript
   // frontend/src/router/index.ts
   {
     path: '/new-feature',
     name: 'NewFeature',
     component: () => import('@/views/new-feature/NewFeatureView.vue'),
     meta: { requiresAuth: true },
   }
   ```

3. **Create composable:**
   ```typescript
   // frontend/src/composables/use-new-feature.ts
   export function useNewFeature() {
     // Implementation
   }
   ```

### 10.4 Testing

```bash
# Backend tests
cd backend
npm test

# Frontend tests
cd frontend
npm test

# E2E (if available)
npm run test:e2e
```

### 10.5 Common Issues

| Issue | Solution |
|-------|----------|
| Zalo disconnected | Check sessionData, try reconnect |
| Rate limit hit | Wait or increase SDK limits |
| Token expired | Implement refresh flow |
| DB migration fails | Check Prisma schema syntax |

---

## Phụ lục

### A. Socket.IO Events

```typescript
// Client → Server
'subscribe:conversation'  // Join conversation room
'unsubscribe:conversation' // Leave conversation room
'send:message'            // Send message

// Server → Client
'message:new'             // New message received
'message:update'          // Message updated
'message:delete'          // Message deleted
'presence:update'         // Zalo presence changed
'zalo:status'             // Zalo account status
```

### B. Cron Jobs

| Job | Schedule | Description |
|-----|----------|-------------|
| appointment-reminder | Every 30s | Send appointment reminders |
| zalo-health-check | Every 5min | Check Zalo connections |
| contact-intelligence | Daily 3am | Enrich contact profiles |
| engagement-classify | Daily 2:30am | Classify engagement patterns |
| friend-sync | Every 15min | Sync friend data |
| group-info-refresh | Every 6h | Refresh group info |
| stuck-detection | Daily 6am | Detect stuck leads |
| scoring-decay | Hourly | Apply score decay |

### C. Security Considerations

- Always filter queries by `orgId` (multi-tenant)
- Use parameterized queries (Prisma handles this)
- Validate all user input
- Rate limit sensitive endpoints
- Use HTTPS in production
- Keep JWT secrets secure
- Implement CSRF protection
- Sanitize file uploads
- Log security events

---

*Document version 1.0 - Last updated: 2026-09-14*
