---
name: dynamodb-onetable
description: Guide for DynamoDB single-table design with Repository pattern using OneTable ORM. Use when implementing Repository classes in the Infrastructure layer, designing database schemas, or handling data persistence for Clean Architecture.
---

# DynamoDB Repository Implementation with OneTable

## Architecture Context

In Clean Architecture, DynamoDB access is encapsulated in **Repository implementations** in the Infrastructure layer. The Repository pattern abstracts persistence from the Domain layer.

```
Domain Layer          Application Layer         Infrastructure Layer
┌─────────────┐      ┌─────────────────┐       ┌──────────────────────────┐
│   Entity    │ ←─── │    Use Case     │ ───→  │  Repository Interface    │
│             │      │                 │       │  (ICategoryRepository)   │
└─────────────┘      └─────────────────┘       └──────────┬───────────────┘
                                                          │
                                               ┌──────────▼───────────────┐
                                               │  DynamoDBCategoryRepo    │
                                               │  (implements interface)  │
                                               └──────────┬───────────────┘
                                                          │
                                               ┌──────────▼───────────────┐
                                               │       DynamoDB           │
                                               └──────────────────────────┘
```

## Key Design Patterns

| Access Pattern | pk | sk | Index |
|---------------|----|----|-------|
| User's items | `USER#${userId}` | `FEATURE#entity#${id}` | primary |
| Item by ID | `ITEM#${id}` | `USER#${userId}` | gsi1 |
| Hierarchical | `USER#${userId}` | `FEATURE#parent#${parentId}#${id}` | primary |
| By date | `USER#${userId}` | `FEATURE#date#${date}#${id}` | primary |
| By status | `STATUS#${status}` | `${createdAt}#${id}` | gsi1 |

## Schema Definition

```typescript
// src/backend/infrastructure/database/db-schema.ts
export const Schema = {
  format: 'onetable:1.1.0',
  version: '0.0.1',
  indexes: {
    primary: { hash: 'pk', sort: 'sk' },
    gsi1: { hash: 'gsi1pk', sort: 'gsi1sk', project: 'all' },
  },
  models: {
    Category: {
      pk: { type: String, value: 'USER#${userId}' },
      sk: { type: String, value: 'CATEGORY#category#${id}' },
      gsi1pk: { type: String, value: 'CATEGORY#${id}' },
      gsi1sk: { type: String, value: 'USER#${userId}' },
      id: { type: String, required: true, generate: 'ulid' },
      userId: { type: String, required: true },
      name: { type: String, required: true },
      description: { type: String },
      status: { type: String, enum: ['active', 'inactive', 'archived'], default: 'active' },
      createdAt: { type: String },
      updatedAt: { type: String },
    },
  },
}
```

## Entity Class (Domain Layer)

Entities define business behavior, NOT persistence:

```typescript
// src/backend/domain/category/entities/Category.ts
import { ulid } from 'ulid'
import { CategoryValidationException } from '../exceptions'

export interface CategoryProps {
  id: string
  userId: string
  name: string
  description?: string
  status: CategoryStatus
  createdAt: Date
  updatedAt: Date
}

export type CategoryStatus = 'active' | 'inactive' | 'archived'

export class Category {
  private constructor(private readonly props: CategoryProps) {
    this.validate()
  }

  static create(input: { name: string; description?: string; userId: string }): Category {
    const now = new Date()
    return new Category({
      id: ulid(),
      userId: input.userId,
      name: input.name,
      description: input.description,
      status: 'active',
      createdAt: now,
      updatedAt: now,
    })
  }

  static fromPersistence(data: Record<string, unknown>): Category {
    return new Category({
      id: data.id as string,
      userId: data.userId as string,
      name: data.name as string,
      description: data.description as string | undefined,
      status: data.status as CategoryStatus,
      createdAt: new Date(data.createdAt as string),
      updatedAt: new Date(data.updatedAt as string),
    })
  }

  private validate(): void {
    if (!this.props.name || this.props.name.trim().length === 0) {
      throw new CategoryValidationException('Name is required')
    }
    if (this.props.name.length > 255) {
      throw new CategoryValidationException('Name must be 255 characters or less')
    }
  }

  get id(): string { return this.props.id }
  get userId(): string { return this.props.userId }
  get name(): string { return this.props.name }
  get description(): string | undefined { return this.props.description }
  get status(): CategoryStatus { return this.props.status }
  get createdAt(): Date { return this.props.createdAt }
  get updatedAt(): Date { return this.props.updatedAt }

  toPersistence(): Record<string, unknown> {
    return {
      id: this.props.id,
      userId: this.props.userId,
      name: this.props.name,
      description: this.props.description,
      status: this.props.status,
      createdAt: this.props.createdAt.toISOString(),
      updatedAt: this.props.updatedAt.toISOString(),
    }
  }
}
```

## Repository Interface (Domain Layer)

```typescript
// src/backend/domain/category/repositories/ICategoryRepository.ts
import type { Category } from '../entities/Category'

export interface ICategoryRepository {
  save(entity: Category): Promise<void>
  findById(id: string, userId: string): Promise<Category | null>
  findByUserId(userId: string, options?: ListOptions): Promise<PaginatedResult<Category>>
  delete(id: string, userId: string): Promise<void>
}

export interface ListOptions {
  limit?: number
  cursor?: string
  status?: string
}

export interface PaginatedResult<T> {
  items: T[]
  nextCursor?: string
  hasMore: boolean
}
```

## Repository Implementation (Infrastructure Layer)

**Key rules:**
- Implements interface from Domain layer
- Throws exceptions on errors (no `{success, data?, error?}` pattern)
- Uses Entity's `toPersistence()` and `fromPersistence()` methods
- Located in `src/backend/infrastructure/<feature>/repositories/`

```typescript
// src/backend/infrastructure/category/repositories/DynamoDBCategoryRepository.ts
import { getDynamoDbTable } from '@/backend/infrastructure/database/db-config'
import { Category } from '@/backend/domain/category/entities/Category'
import type {
  ICategoryRepository,
  ListOptions,
  PaginatedResult,
} from '@/backend/domain/category/repositories/ICategoryRepository'
import { log } from '@/lib/logger'

export class DynamoDBCategoryRepository implements ICategoryRepository {
  private getModel() {
    return getDynamoDbTable().getModel('Category')
  }

  async save(entity: Category): Promise<void> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      await Model.upsert(entity.toPersistence())

      log.debug('[CategoryRepository.save] Success', {
        id: entity.id,
        duration: Date.now() - startTime,
      })
    } catch (error) {
      log.error('[CategoryRepository.save] Failed', { error, id: entity.id })
      throw error
    }
  }

  async findById(id: string, userId: string): Promise<Category | null> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      const data = await Model.get({
        pk: `USER#${userId}`,
        sk: `CATEGORY#category#${id}`,
      })

      log.debug('[CategoryRepository.findById] Complete', {
        id,
        found: !!data,
        duration: Date.now() - startTime,
      })

      if (!data) return null

      return Category.fromPersistence(data)
    } catch (error) {
      log.error('[CategoryRepository.findById] Failed', { error, id })
      throw error
    }
  }

  async findByUserId(userId: string, options: ListOptions = {}): Promise<PaginatedResult<Category>> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      const limit = options.limit ?? 20

      const queryOptions: any = {
        pk: `USER#${userId}`,
        sk: { begins: 'CATEGORY#category#' },
        limit: limit + 1,
      }

      if (options.cursor) {
        queryOptions.start = JSON.parse(Buffer.from(options.cursor, 'base64').toString())
      }

      if (options.status) {
        queryOptions.where = '${status} = {status}'
        queryOptions.substitutions = { status: options.status }
      }

      const results = await Model.find(queryOptions)

      const hasMore = results.length > limit
      const items = hasMore ? results.slice(0, limit) : results

      const nextCursor = hasMore && results[limit - 1]
        ? Buffer.from(JSON.stringify({
            pk: results[limit - 1].pk,
            sk: results[limit - 1].sk,
          })).toString('base64')
        : undefined

      log.debug('[CategoryRepository.findByUserId] Complete', {
        userId,
        count: items.length,
        hasMore,
        duration: Date.now() - startTime,
      })

      return {
        items: items.map(data => Category.fromPersistence(data)),
        nextCursor,
        hasMore,
      }
    } catch (error) {
      log.error('[CategoryRepository.findByUserId] Failed', { error, userId })
      throw error
    }
  }

  async delete(id: string, userId: string): Promise<void> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      await Model.remove({
        pk: `USER#${userId}`,
        sk: `CATEGORY#category#${id}`,
      })

      log.debug('[CategoryRepository.delete] Success', {
        id,
        duration: Date.now() - startTime,
      })
    } catch (error) {
      log.error('[CategoryRepository.delete] Failed', { error, id })
      throw error
    }
  }
}
```

## Query Patterns

### Get by GSI (Global Secondary Index)

```typescript
async findByIdOnly(id: string): Promise<Category | null> {
  const Model = this.getModel()
  const data = await Model.get(
    { gsi1pk: `CATEGORY#${id}` },
    { index: 'gsi1' }
  )

  if (!data) return null
  return Category.fromPersistence(data)
}
```

### Query with Filter

```typescript
async findActiveByUserId(userId: string): Promise<Category[]> {
  const Model = this.getModel()
  const results = await Model.find(
    { pk: `USER#${userId}`, sk: { begins: 'CATEGORY#category#' } },
    { where: '${status} = {active}', substitutions: { active: 'active' } }
  )

  return results.map(data => Category.fromPersistence(data))
}
```

### Soft Delete Pattern

```typescript
async softDelete(id: string, userId: string): Promise<void> {
  const entity = await this.findById(id, userId)
  if (!entity) return

  const archived = entity.archive() // Entity method returns new instance
  await this.save(archived)
}
```

### Batch Operations

```typescript
async saveMany(entities: Category[]): Promise<void> {
  const Model = this.getModel()
  const batch = entities.map(e => ({
    put: e.toPersistence(),
  }))

  await Model.batchWrite(batch)
}
```

## Schema Evolution

**Adding fields:** Always optional with defaults handled in Entity:

```typescript
// Schema update - field is optional
newField: { type: String }

// Entity handles missing field
static fromPersistence(data: Record<string, unknown>): Category {
  return new Category({
    // ...
    newField: (data.newField as string) ?? 'default-value',
  })
}
```

## Key Pattern Rules

| Pattern | Format | Example |
|---------|--------|---------|
| User partition | `USER#${userId}` | `USER#01HXYZ123` |
| Entity sort key | `FEATURE#entity#${id}` | `CATEGORY#category#01HXYZ456` |
| GSI by ID | `FEATURE#${id}` | `CATEGORY#01HXYZ456` |
| Hierarchical | `FEATURE#parent#${parentId}#${id}` | `CATEGORY#parent#01HX#01HY` |
| Time-based | `FEATURE#date#${date}#${id}` | `CATEGORY#date#2024-01-15#01HX` |

## Rules Summary

1. **Use ULID for IDs** (time-sortable)
2. **Entity owns persistence format** via `toPersistence()` and `fromPersistence()`
3. **Repository throws exceptions** (not `{success, data?, error?}`)
4. **Log with timing** for observability
5. **Handle missing fields** in `fromPersistence()` with defaults
6. **No business logic** in Repository (belongs in Entity/Use Case)
7. **Interface in Domain**, implementation in Infrastructure

## Anti-Patterns

### ❌ Repository returning result objects

```typescript
// BAD - Functional pattern
async save(entity: Category): Promise<{ success: boolean; error?: string }> {
  try {
    await Model.upsert(entity.toPersistence())
    return { success: true }
  } catch (error) {
    return { success: false, error: 'Failed to save' }
  }
}
```

```typescript
// GOOD - Clean Architecture pattern
async save(entity: Category): Promise<void> {
  try {
    await Model.upsert(entity.toPersistence())
  } catch (error) {
    log.error('[CategoryRepository.save] Failed', { error })
    throw error // Let use case handle it
  }
}
```

### ❌ Business logic in Repository

```typescript
// BAD - Business rule in repository
async save(entity: Category): Promise<void> {
  if (entity.name.includes('banned')) { // Business rule!
    throw new Error('Invalid name')
  }
  await Model.upsert(entity.toPersistence())
}
```

```typescript
// GOOD - Business rule in Entity
// Entity.ts
private validate(): void {
  if (this.props.name.includes('banned')) {
    throw new CategoryValidationException('Name contains banned words')
  }
}
```

### ❌ Repository in Domain layer

```typescript
// BAD - Implementation in domain
// src/backend/domain/category/DynamoDBCategoryRepository.ts
import { getDynamoDbTable } from '@/backend/infrastructure/database' // VIOLATION!
```

```typescript
// GOOD - Only interface in domain
// src/backend/domain/category/repositories/ICategoryRepository.ts
export interface ICategoryRepository {
  save(entity: Category): Promise<void>
  // ...
}

// Implementation in infrastructure
// src/backend/infrastructure/category/repositories/DynamoDBCategoryRepository.ts
export class DynamoDBCategoryRepository implements ICategoryRepository { ... }
```

## DI Container Registration

```typescript
// src/backend/infrastructure/di/container.ts
import { DynamoDBCategoryRepository } from '../category/repositories/DynamoDBCategoryRepository'
import { TOKENS } from './tokens'

// Register repository as singleton
DIContainer.register(TOKENS.CategoryRepository, () => new DynamoDBCategoryRepository())
```

## Testing Repositories

```typescript
// src/test/category/repositories/DynamoDBCategoryRepository.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { DynamoDBCategoryRepository } from '@/backend/infrastructure/category/repositories'
import { Category } from '@/backend/domain/category/entities'
import { setupTestDb, clearTestData } from '@/test/db-helpers'

describe('DynamoDBCategoryRepository', () => {
  const repository = new DynamoDBCategoryRepository()
  const testUserId = '01HXYZ123456789ABCDEFGHIJK'

  beforeEach(async () => {
    await clearTestData('Category')
  })

  describe('save', () => {
    it('persists a new category', async () => {
      const category = Category.create({
        name: 'Test Category',
        userId: testUserId,
      })

      await repository.save(category)

      const found = await repository.findById(category.id, testUserId)
      expect(found).not.toBeNull()
      expect(found?.name).toBe('Test Category')
    })
  })

  describe('findById', () => {
    it('returns null for non-existent category', async () => {
      const found = await repository.findById('nonexistent', testUserId)
      expect(found).toBeNull()
    })
  })

  describe('findByUserId', () => {
    it('returns paginated results', async () => {
      // Create test data
      await Promise.all([
        repository.save(Category.create({ name: 'Cat 1', userId: testUserId })),
        repository.save(Category.create({ name: 'Cat 2', userId: testUserId })),
        repository.save(Category.create({ name: 'Cat 3', userId: testUserId })),
      ])

      const result = await repository.findByUserId(testUserId, { limit: 2 })

      expect(result.items).toHaveLength(2)
      expect(result.hasMore).toBe(true)
      expect(result.nextCursor).toBeDefined()
    })
  })
})
```

## References

- Clean Architecture: `skills/feature-architecture/SKILL.md`
- Create Domain Module: `skills/create-domain-module/SKILL.md`
- Testing: `skills/create-e2e-tests/SKILL.md`
