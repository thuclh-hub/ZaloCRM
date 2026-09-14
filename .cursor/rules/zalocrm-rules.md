# ZaloCRM Cursor Rules

> Hướng dẫn phát triển cho ZaloCRM v3.4  
> Phiên bản: 1.0  
> Cập nhật: 2026-09-14

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Cấu trúc dự án](#2-cấu-trúc-dự-án)
3. [Backend Development](#3-backend-development)
4. [Frontend Development](#4-frontend-development)
5. [Database](#5-database)
6. [API Design](#6-api-design)
7. [Security](#7-security)
8. [Testing](#8-testing)
9. [Deployment](#9-deployment)

---

## 1. Tổng quan

### 1.1 Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Node.js 20, Fastify 5, Prisma 7 |
| Frontend | Vue 3, Vuetify 4, Pinia 3, Vite 8 |
| Database | PostgreSQL 16 |
| Cache/Queue | Redis 7 |
| Realtime | Socket.IO 4 |
| Storage | MinIO / S3 / R2 |
| Zalo Integration | zca-js 2.x |

### 1.2 Key Files

| Purpose | File Path |
|---------|-----------|
| Backend entry | `backend/src/app.ts` |
| Frontend entry | `frontend/src/main.ts` |
| Database schema | `backend/prisma/schema.prisma` |
| Router config | `frontend/src/router/index.ts` |
| API client | `frontend/src/api/index.ts` |
| Auth store | `frontend/src/stores/auth.ts` |
| Docker compose | `docker-compose.yml` |

---

## 2. Cấu trúc dự án

```
ZaloCRM/
├── backend/
│   ├── src/
│   │   ├── app.ts                 # Main entry point
│   │   ├── config/               # Configuration
│   │   ├── modules/              # Feature modules (20+)
│   │   │   ├── auth/             # Authentication
│   │   │   ├── zalo/             # Zalo integration
│   │   │   ├── chat/             # Chat module
│   │   │   ├── contacts/         # Contact management
│   │   │   ├── scoring/          # Lead scoring
│   │   │   ├── ai/               # AI features
│   │   │   ├── dashboard/        # Dashboard & reports
│   │   │   └── ...
│   │   └── shared/               # Shared utilities
│   │       ├── database/         # Prisma client
│   │       ├── storage/          # Storage drivers
│   │       ├── realtime/         # Socket.IO utilities
│   │       └── security/         # Security utilities
│   └── prisma/
│       ├── schema.prisma         # Database schema
│       └── migrations/           # Migrations
│
├── frontend/
│   ├── src/
│   │   ├── main.ts               # Entry point
│   │   ├── App.vue               # Root component
│   │   ├── api/                  # API client
│   │   ├── components/           # Vue components
│   │   ├── composables/          # Vue composables
│   │   ├── router/               # Vue Router config
│   │   ├── stores/               # Pinia stores
│   │   └── views/                # Page components
│   └── package.json
│
├── docs/
│   ├── PROJECT-DOCUMENTATION.md  # Full documentation
│   └── architecture/             # Architecture diagrams
│
└── docker-compose.yml
```

---

## 3. Backend Development

### 3.1 Module Structure

Mỗi module backend nên có cấu trúc:

```
modules/feature/
├── feature-routes.ts      # Fastify route handlers
├── feature-service.ts     # Business logic
├── feature-types.ts       # TypeScript types
└── feature-utils.ts       # Helper functions
```

### 3.2 Route Handler Pattern

```typescript
// ✅ CORRECT - Route handler pattern
import { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify';

// Type for request body
interface CreateItemBody {
  name: string;
  description?: string;
}

// Route registration
export async function featureRoutes(app: FastifyInstance) {
  // GET list
  app.get('/items', async (request: FastifyRequest, reply: FastifyReply) => {
    const { orgId, userId } = request.authContext;
    const items = await prisma.item.findMany({
      where: { orgId },
      orderBy: { createdAt: 'desc' },
    });
    return reply.send({ data: items });
  });

  // GET by ID
  app.get('/items/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const item = await prisma.item.findFirst({
      where: { id, orgId: request.authContext.orgId },
    });
    if (!item) {
      return reply.status(404).send({ error: 'Item not found' });
    }
    return reply.send({ data: item });
  });

  // POST create
  app.post('/items', async (request: FastifyRequest<{ Body: CreateItemBody }>, reply) => {
    const { name, description } = request.body;
    const { orgId, userId } = request.authContext;

    if (!name?.trim()) {
      return reply.status(400).send({ error: 'Name is required' });
    }

    const item = await prisma.item.create({
      data: {
        name: name.trim(),
        description,
        orgId,
        createdBy: userId,
      },
    });

    return reply.status(201).send({ data: item });
  });

  // PUT update
  app.put('/items/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const { orgId, userId } = request.authContext;
    const data = request.body as Partial<CreateItemBody>;

    const existing = await prisma.item.findFirst({
      where: { id, orgId },
    });
    if (!existing) {
      return reply.status(404).send({ error: 'Item not found' });
    }

    const updated = await prisma.item.update({
      where: { id },
      data: {
        ...(data.name && { name: data.name.trim() }),
        ...(data.description !== undefined && { description: data.description }),
      },
    });

    return reply.send({ data: updated });
  });

  // DELETE
  app.delete('/items/:id', async (request, reply) => {
    const { id } = request.params as { id: string };
    const { orgId } = request.authContext;

    const existing = await prisma.item.findFirst({
      where: { id, orgId },
    });
    if (!existing) {
      return reply.status(404).send({ error: 'Item not found' });
    }

    await prisma.item.delete({ where: { id } });
    return reply.status(204).send();
  });
}
```

### 3.3 Authentication Context

Luôn extract `orgId` và `userId` từ request context:

```typescript
// request.authContext is set by auth middleware
const { orgId, userId } = request.authContext;

// ❌ WRONG - Don't do this
const item = await prisma.item.findFirst({
  where: { id: request.params.id }, // Missing orgId filter!
});

// ✅ CORRECT - Always filter by orgId
const item = await prisma.item.findFirst({
  where: { id, orgId },
});
```

### 3.4 Register Routes in app.ts

```typescript
// backend/src/app.ts
import { featureRoutes } from './modules/feature/feature-routes';

// In bootstrap() async function:
await app.register(featureRoutes);

// Or with prefix:
await app.register(featureRoutes, { prefix: '/api/v1/feature' });
```

### 3.5 Error Handling

```typescript
// Use try-catch for async operations
app.post('/items', async (request, reply) => {
  try {
    const result = await someAsyncOperation(request.body);
    return reply.send({ data: result });
  } catch (error: any) {
    if (error.code === 'P2002') {
      return reply.status(409).send({ error: 'Duplicate entry' });
    }
    if (error.statusCode) {
      return reply.status(error.statusCode).send({ error: error.message });
    }
    request.log.error(error);
    return reply.status(500).send({ error: 'Internal server error' });
  }
});
```

### 3.6 Logging

```typescript
import { logger } from './shared/utils/logger.js';

logger.info('Processing item creation', { orgId, userId });
logger.error('Failed to create item', { error: error.message, orgId });
logger.debug('Detailed debug info', { data });
```

---

## 4. Frontend Development

### 4.1 View Component Pattern

```vue
<!-- views/feature/FeatureView.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useToast } from '@/composables/use-toast';

// Props
const props = defineProps<{
  id?: string;
}>();

// Emits
const emit = defineEmits<{
  (e: 'updated'): void;
  (e: 'deleted'): void;
}>();

// Router & composables
const router = useRouter();
const toast = useToast();

// State
const loading = ref(false);
const saving = ref(false);
const items = ref<Item[]>([]);
const selectedItem = ref<Item | null>(null);

// Computed
const hasItems = computed(() => items.value.length > 0);

// Methods
async function fetchItems() {
  loading.value = true;
  try {
    const res = await api.get('/items');
    items.value = res.data.data;
  } catch (error: any) {
    toast.error(error.response?.data?.error || 'Failed to fetch items');
  } finally {
    loading.value = false;
  }
}

async function createItem(data: CreateItemDto) {
  saving.value = true;
  try {
    await api.post('/items', data);
    toast.success('Item created successfully');
    emit('updated');
    await fetchItems();
  } catch (error: any) {
    toast.error(error.response?.data?.error || 'Failed to create item');
  } finally {
    saving.value = false;
  }
}

async function deleteItem(id: string) {
  try {
    await api.delete(`/items/${id}`);
    toast.success('Item deleted');
    emit('deleted');
    await fetchItems();
  } catch (error: any) {
    toast.error(error.response?.data?.error || 'Failed to delete');
  }
}

// Lifecycle
onMounted(fetchItems);
</script>

<template>
  <div class="feature-view">
    <!-- Loading state -->
    <div v-if="loading" class="d-flex justify-center align-center pa-4">
      <v-progress-circular indeterminate color="primary" />
    </div>

    <!-- Content -->
    <template v-else>
      <!-- Header -->
      <div class="d-flex align-center mb-4">
        <h1 class="text-h5">Feature Items</h1>
        <v-spacer />
        <v-btn color="primary" @click="showCreateDialog = true">
          <v-icon start>mdi-plus</v-icon>
          New Item
        </v-btn>
      </div>

      <!-- Empty state -->
      <v-empty-state
        v-if="!hasItems"
        headline="No items yet"
        title="Get started"
        description="Create your first item to see it here."
      />

      <!-- List -->
      <v-list v-else>
        <v-list-item
          v-for="item in items"
          :key="item.id"
          :title="item.name"
          :subtitle="item.description"
        >
          <template #actions>
            <v-btn-icon variant="text" @click="editItem(item)">
              <v-icon>mdi-pencil</v-icon>
            </v-btn-icon>
            <v-btn-icon variant="text" color="error" @click="confirmDelete(item)">
              <v-icon>mdi-delete</v-icon>
            </v-btn-icon>
          </template>
        </v-list-item>
      </v-list>
    </template>
  </div>
</template>
```

### 4.2 Composable Pattern

```typescript
// composables/use-items.ts
import { ref, computed } from 'vue';
import { api } from '@/api/index';

export interface Item {
  id: string;
  name: string;
  description?: string;
  createdAt: string;
}

export interface CreateItemDto {
  name: string;
  description?: string;
}

export function useItems() {
  // State
  const items = ref<Item[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);

  // Computed
  const hasItems = computed(() => items.value.length > 0);

  // Methods
  async function fetchItems(params?: Record<string, any>) {
    loading.value = true;
    error.value = null;
    try {
      const res = await api.get('/items', { params });
      items.value = res.data.data;
    } catch (e: any) {
      error.value = e.response?.data?.error || 'Failed to fetch items';
      throw e;
    } finally {
      loading.value = false;
    }
  }

  async function createItem(data: CreateItemDto) {
    const res = await api.post('/items', data);
    const newItem = res.data.data;
    items.value.unshift(newItem);
    return newItem;
  }

  async function updateItem(id: string, data: Partial<CreateItemDto>) {
    const res = await api.put(`/items/${id}`, data);
    const updated = res.data.data;
    const index = items.value.findIndex((i) => i.id === id);
    if (index !== -1) {
      items.value[index] = updated;
    }
    return updated;
  }

  async function deleteItem(id: string) {
    await api.delete(`/items/${id}`);
    items.value = items.value.filter((i) => i.id !== id);
  }

  return {
    items,
    loading,
    error,
    hasItems,
    fetchItems,
    createItem,
    updateItem,
    deleteItem,
  };
}
```

### 4.3 Router Registration

```typescript
// frontend/src/router/index.ts
{
  path: '/feature',
  name: 'Feature',
  component: () => import('@/views/feature/FeatureView.vue'),
  meta: {
    requiresAuth: true,
    resource: 'feature_resource', // For RBAC
  },
},
```

### 4.4 RBAC Permission Check

```typescript
// In auth store
const authStore = useAuthStore();

// Check permission
if (authStore.canAccess('contacts', 'create')) {
  // Show create button
}

// Check role
if (authStore.isAdmin) {
  // Show admin features
}

// Check manager
if (authStore.isManager) {
  // Show manager features
}
```

### 4.5 Toast Notifications

```typescript
import { useToast } from '@/composables/use-toast';

const toast = useToast();

// Success
toast.success('Operation completed successfully');

// Error
toast.error('Failed to complete operation');

// Warning
toast.warning('Please review before proceeding');

// Info
toast.info('Operation is in progress');
```

---

## 5. Database

### 5.1 Prisma Schema Pattern

```prisma
// ✅ CORRECT - Multi-tenant model
model Item {
  id        String   @id @default(uuid())
  orgId     String   @map("org_id")  // Required for multi-tenant
  name      String
  // ... other fields

  org       Organization @relation(fields: [orgId], references: [id], onDelete: Cascade)

  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  // Indexes for common queries
  @@index([orgId, name])
  @@index([orgId, createdAt(sort: Desc)])

  @@map("items")
}
```

### 5.2 Migration Commands

```bash
# Create migration
cd backend
npx prisma migrate dev --name add_new_model

# Apply to production
docker exec zalo-crm-app npx prisma migrate deploy

# Reset (DANGER - deletes data)
npx prisma migrate reset

# Generate client
npx prisma generate

# Open studio
npx prisma studio
```

### 5.3 Query Patterns

```typescript
// ✅ Always filter by orgId
const items = await prisma.item.findMany({
  where: { orgId },
});

// ✅ With pagination
const items = await prisma.item.findMany({
  where: { orgId },
  orderBy: { createdAt: 'desc' },
  take: 20,
  skip: (page - 1) * 20,
});

// ✅ With relations
const item = await prisma.item.findFirst({
  where: { id, orgId },
  include: {
    creator: { select: { id: true, fullName: true } },
    tags: true,
  },
});

// ✅ Transaction for multi-step operations
await prisma.$transaction([
  prisma.item.update({
    where: { id },
    data: { status: 'completed' },
  }),
  prisma.activityLog.create({
    data: {
      orgId,
      userId,
      action: 'item_completed',
      targetId: id,
    },
  }),
]);
```

---

## 6. API Design

### 6.1 REST Conventions

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource` | List resources |
| GET | `/resource/:id` | Get single resource |
| POST | `/resource` | Create resource |
| PUT | `/resource/:id` | Full update |
| PATCH | `/resource/:id` | Partial update |
| DELETE | `/resource/:id` | Delete resource |

### 6.2 Response Format

```typescript
// Success
{
  "data": { /* resource */ }
}

// List with pagination
{
  "data": [ /* resources */ ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5
  }
}

// Error
{
  "error": "Error message"
}
```

### 6.3 Request Validation

```typescript
app.post('/items', async (request, reply) => {
  const body = request.body as any;

  // Validate required fields
  if (!body.name?.trim()) {
    return reply.status(400).send({ error: 'Name is required' });
  }

  // Validate length
  if (body.name.length > 255) {
    return reply.status(400).send({ error: 'Name too long' });
  }

  // Validate enum
  const validStatuses = ['pending', 'active', 'completed'];
  if (body.status && !validStatuses.includes(body.status)) {
    return reply.status(400).send({ error: 'Invalid status' });
  }

  // Continue...
});
```

---

## 7. Security

### 7.1 Multi-tenant Safety

```typescript
// ❌ NEVER do this - Security vulnerability
app.get('/items/:id', async (request, reply) => {
  const item = await prisma.item.findUnique({
    where: { id: request.params.id },
    // Missing orgId filter!
  });
});

// ✅ ALWAYS do this
app.get('/items/:id', async (request, reply) => {
  const item = await prisma.item.findFirst({
    where: {
      id: request.params.id,
      orgId: request.authContext.orgId,
    },
  });
  if (!item) {
    return reply.status(404).send({ error: 'Not found' });
  }
});
```

### 7.2 Input Sanitization

```typescript
// Sanitize string inputs
const name = String(body.name || '').trim().slice(0, 255);

// Validate phone numbers
const phoneRegex = /^(0|84|\+84)?[1-9]\d{8,9}$/;

// Validate emails
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
```

### 7.3 Rate Limiting

The API already has rate limiting (1200 requests/minute per user). For new sensitive endpoints:

```typescript
// Additional rate limiting if needed
app.post('/sensitive-action', {
  config: {
    rateLimit: {
      max: 10,
      timeWindow: '1 minute',
    },
  },
}, async (request, reply) => {
  // Handler
});
```

---

## 8. Testing

### 8.1 Backend Tests

```typescript
// tests/unit/my-feature.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { app } from '../test-helpers';

describe('My Feature', () => {
  let token: string;
  let orgId: string;

  beforeEach(async () => {
    // Setup test data
    const auth = await app.inject({
      method: 'POST',
      url: '/auth/login',
      payload: { email: 'test@example.com', password: 'password' },
    });
    token = auth.json().token;
    orgId = auth.json().user.orgId;
  });

  it('should create item', async () => {
    const res = await app.inject({
      method: 'POST',
      url: '/items',
      headers: { Authorization: `Bearer ${token}` },
      payload: { name: 'Test Item' },
    });

    expect(res.statusCode).toBe(201);
    expect(res.json().data.name).toBe('Test Item');
  });

  it('should reject without auth', async () => {
    const res = await app.inject({
      method: 'GET',
      url: '/items',
    });

    expect(res.statusCode).toBe(401);
  });

  it('should only show org items', async () => {
    const res = await app.inject({
      method: 'GET',
      url: '/items',
      headers: { Authorization: `Bearer ${token}` },
    });

    const items = res.json().data;
    expect(items.every((i: any) => i.orgId === orgId)).toBe(true);
  });
});
```

### 8.2 Frontend Tests

```typescript
// composables/use-items.spec.ts
import { describe, it, expect } from 'vitest';
import { useItems } from './use-items';

describe('useItems', () => {
  it('should initialize with empty items', () => {
    const { items, loading } = useItems();
    expect(items.value).toEqual([]);
    expect(loading.value).toBe(false);
  });

  it('should compute hasItems correctly', () => {
    const { items, hasItems } = useItems();
    expect(hasItems.value).toBe(false);
    items.value = [{ id: '1', name: 'Test' }];
    expect(hasItems.value).toBe(true);
  });
});
```

---

## 9. Deployment

### 9.1 Docker Commands

```bash
# Build and start
docker compose up -d --build

# View logs
docker compose logs -f app

# Restart services
docker compose restart app

# Run migrations
docker exec zalo-crm-app npx prisma migrate deploy

# Backup database
docker exec zalo-crm-db pg_dump -U crmuser zalocrm > backup.sql

# Restore database
cat backup.sql | docker exec -i zalo-crm-db psql -U crmuser zalocrm
```

### 9.2 Environment Variables

Required in `.env`:
```bash
# Security (REQUIRED)
JWT_SECRET=<32+ chars hex>
ENCRYPTION_KEY=<64 hex chars>
DB_PASSWORD=<password>

# Optional
REDIS_URL=redis://redis:6379
STORAGE_DRIVER=local
TELEGRAM_BRIDGE_BOT_TOKEN=
```

### 9.3 Health Check

```bash
# Check app health
curl http://localhost:3080/health

# Check DB connection
docker exec zalo-crm-app curl -s http://localhost:3000/health
```

---

## Quick Reference

### Common Tasks

| Task | Command |
|------|---------|
| Add backend route | Create `*-routes.ts`, register in `app.ts` |
| Add frontend view | Create `.vue` in `views/`, add route |
| Add database model | Edit `schema.prisma`, run migration |
| Add API endpoint | Add route, return `{ data: result }` |
| Add composable | Create `use-*.ts` in `composables/` |

### Key Files Reference

| Purpose | Path |
|---------|------|
| Backend entry | `backend/src/app.ts` |
| Frontend entry | `frontend/src/main.ts` |
| Router | `frontend/src/router/index.ts` |
| Auth store | `frontend/src/stores/auth.ts` |
| API client | `frontend/src/api/index.ts` |
| Database schema | `backend/prisma/schema.prisma` |

### Important Patterns

1. **Multi-tenant**: Always filter by `orgId`
2. **Auth context**: Use `request.authContext`
3. **Response format**: `{ data: ... }` for success, `{ error: ... }` for errors
4. **Frontend state**: Use composables with `ref`/`computed`
5. **Toast notifications**: Use `useToast()` composable
