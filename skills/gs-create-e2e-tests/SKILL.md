---
name: gs-create-e2e-tests
description: Creates tests following Clean Architecture test layers. Entity tests (pure unit), Use Case tests (mock repositories), Repository tests (DynamoDB Local), E2E tests (full flow). Uses Vitest and DI Container mocking.
---

# Testing with Clean Architecture

## Test Layer Hierarchy

```
┌─────────────────────────────────────────┐
│ E2E Tests (Action → Use Case → DB)      │ Full flow
├─────────────────────────────────────────┤
│ Repository Tests (DynamoDB Local)       │ Infrastructure
├─────────────────────────────────────────┤
│ Use Case Tests (Mock Repositories)      │ Application
├─────────────────────────────────────────┤
│ Entity Tests (Pure Unit)                │ Domain
└─────────────────────────────────────────┘
```

## File Location Pattern

```
src/backend/domain/<feature>/
├── entities/
│   ├── <entity>.ts
│   └── <entity>.test.ts         # Entity unit tests

src/backend/application/<feature>/
├── use-cases/
│   ├── <use-case>.ts
│   └── <use-case>.test.ts       # Use Case tests (mock repos)

src/backend/infrastructure/<feature>/
├── repositories/
│   ├── <repo>-impl.ts
│   └── <repo>-impl.test.ts      # Repository integration tests

src/features/<feature>/
├── actions/
│   ├── <action>.ts
│   └── <action>.test.ts         # E2E tests (full flow)
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Test Runner | Vitest |
| Database | DynamoDB Local |
| Mocking | Vitest mocks + DI Container |
| Test Data | @faker-js/faker |

## 1. Entity Tests (Domain Layer)

Pure unit tests - no mocks, no I/O.

```typescript
// src/backend/domain/category/entities/category.test.ts
import { describe, it, expect } from 'vitest'
import { Category } from './category'
import { DomainException } from '@/backend/domain/shared/exceptions'

describe('Category Entity', () => {
  describe('create', () => {
    it('creates valid category', () => {
      const category = Category.create({
        id: 'cat_123',
        name: 'Electronics',
        userId: 'user_456',
      })

      expect(category.id).toBe('cat_123')
      expect(category.name).toBe('Electronics')
      expect(category.status).toBe('active')
    })

    it('throws on empty name', () => {
      expect(() =>
        Category.create({ id: 'cat_123', name: '', userId: 'user_456' })
      ).toThrow(DomainException)
    })

    it('throws on name exceeding max length', () => {
      expect(() =>
        Category.create({ id: 'cat_123', name: 'x'.repeat(256), userId: 'user_456' })
      ).toThrow(DomainException)
    })
  })

  describe('updateName', () => {
    it('updates name on valid category', () => {
      const category = Category.create({
        id: 'cat_123',
        name: 'Old Name',
        userId: 'user_456',
      })

      category.updateName('New Name')

      expect(category.name).toBe('New Name')
    })
  })

  describe('deactivate', () => {
    it('changes status to inactive', () => {
      const category = Category.create({
        id: 'cat_123',
        name: 'Electronics',
        userId: 'user_456',
      })

      category.deactivate()

      expect(category.status).toBe('inactive')
    })
  })
})
```

## 2. Use Case Tests (Application Layer)

Mock repository interfaces - test business logic only.

```typescript
// src/backend/application/category/use-cases/create-category.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { faker } from '@faker-js/faker'
import { CreateCategoryUseCase } from './create-category'
import type { ICategoryRepository } from '@/backend/domain/category/repositories'

describe('CreateCategoryUseCase', () => {
  let useCase: CreateCategoryUseCase
  let mockRepository: ICategoryRepository

  beforeEach(() => {
    mockRepository = {
      save: vi.fn(),
      findById: vi.fn(),
      findByUser: vi.fn(),
      delete: vi.fn(),
    }
    useCase = new CreateCategoryUseCase(mockRepository)
  })

  it('creates category and returns DTO', async () => {
    vi.mocked(mockRepository.save).mockResolvedValue(undefined)

    const result = await useCase.execute({
      name: 'Electronics',
      userId: 'user_456',
    })

    expect(result.name).toBe('Electronics')
    expect(result.id).toBeDefined()
    expect(mockRepository.save).toHaveBeenCalledTimes(1)
  })

  it('propagates domain exceptions', async () => {
    await expect(
      useCase.execute({ name: '', userId: 'user_456' })
    ).rejects.toThrow('Name is required')
  })

  it('propagates repository exceptions', async () => {
    vi.mocked(mockRepository.save).mockRejectedValue(
      new Error('Database error')
    )

    await expect(
      useCase.execute({ name: 'Valid', userId: 'user_456' })
    ).rejects.toThrow('Database error')
  })
})
```

## 3. Repository Tests (Infrastructure Layer)

Integration tests with DynamoDB Local.

### Test Database Helpers

```typescript
// src/test/db-helpers.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { Table } from 'dynamodb-onetable'
import { schema } from '@/backend/infrastructure/database/schema'

const testClient = new DynamoDBClient({
  endpoint: process.env.DYNAMODB_LOCAL_ENDPOINT || 'http://localhost:8000',
  region: 'local',
  credentials: { accessKeyId: 'local', secretAccessKey: 'local' },
})

let testTable: Table

export const setupTestDb = async () => {
  testTable = new Table({
    client: testClient,
    name: process.env.TEST_TABLE_NAME || 'test-table',
    schema,
    partial: true,
  })

  try {
    await testTable.createTable()
  } catch (error: any) {
    if (error.name !== 'ResourceInUseException') throw error
  }
  return testTable
}

export const clearTestData = async (modelName: string) => {
  const Model = testTable.getModel(modelName)
  const items = await Model.scan({})
  for (const item of items) await Model.remove(item)
}

export const getTestTable = () => testTable
```

### Repository Test

```typescript
// src/backend/infrastructure/category/repositories/category-repository-impl.test.ts
import { describe, it, expect, beforeAll, beforeEach } from 'vitest'
import { faker } from '@faker-js/faker'
import { setupTestDb, clearTestData, getTestTable } from '@/test/db-helpers'
import { CategoryRepositoryImpl } from './category-repository-impl'
import { Category } from '@/backend/domain/category/entities'

describe('CategoryRepositoryImpl', () => {
  let repository: CategoryRepositoryImpl

  beforeAll(async () => {
    await setupTestDb()
    repository = new CategoryRepositoryImpl(getTestTable())
  })

  beforeEach(async () => {
    await clearTestData('Category')
  })

  describe('save', () => {
    it('persists category to database', async () => {
      const category = Category.create({
        id: faker.string.ulid(),
        name: 'Electronics',
        userId: 'user_456',
      })

      await repository.save(category)

      const found = await repository.findById(category.id, category.userId)
      expect(found?.name).toBe('Electronics')
    })
  })

  describe('findById', () => {
    it('returns null for non-existent category', async () => {
      const result = await repository.findById('non_existent', 'user_456')
      expect(result).toBeNull()
    })
  })

  describe('findByUser', () => {
    it('returns paginated results', async () => {
      const userId = faker.string.ulid()

      for (let i = 0; i < 5; i++) {
        const category = Category.create({
          id: faker.string.ulid(),
          name: `Category ${i}`,
          userId,
        })
        await repository.save(category)
      }

      const page1 = await repository.findByUser(userId, { limit: 2 })
      expect(page1.items).toHaveLength(2)
      expect(page1.nextCursor).toBeDefined()

      const page2 = await repository.findByUser(userId, {
        limit: 2,
        cursor: page1.nextCursor,
      })
      expect(page2.items).toHaveLength(2)
    })
  })
})
```

## 4. E2E Tests (Full Flow)

Test action → use case → repository → database.

```typescript
// src/features/category/actions/create-category.test.ts
import { describe, it, expect, beforeAll, beforeEach, vi } from 'vitest'
import { faker } from '@faker-js/faker'
import { setupTestDb, clearTestData } from '@/test/db-helpers'
import { createCategoryAction } from './create-category'

// Mock auth context
vi.mock('@saas4dev/auth', () => ({
  authServer: {
    api: {
      getSession: vi.fn().mockResolvedValue({ user: { id: 'user_123' } }),
    },
  },
}))

describe('createCategoryAction E2E', () => {
  beforeAll(async () => {
    await setupTestDb()
  })

  beforeEach(async () => {
    await clearTestData('Category')
  })

  it('creates category through full stack', async () => {
    const [result, err] = await createCategoryAction({ name: 'Electronics' })

    expect(err).toBeNull()
    expect(result?.name).toBe('Electronics')
    expect(result?.id).toBeDefined()
  })

  it('returns validation error for empty name', async () => {
    const [result, err] = await createCategoryAction({ name: '' })

    expect(result).toBeNull()
    expect(err).toBeDefined()
  })
})
```

## DI Container Mocking

For testing with DI Container, override registrations.

```typescript
// src/test/di-helpers.ts
import { DIContainer, TOKENS } from '@/backend/di'

export const mockRepository = <T>(token: symbol, mock: Partial<T>) => {
  const original = DIContainer.resolve(token)
  DIContainer.register(token, { useValue: mock as T })
  return () => DIContainer.register(token, { useValue: original })
}
```

```typescript
// Usage in tests
import { mockRepository } from '@/test/di-helpers'
import { TOKENS } from '@/backend/di'

describe('UseCase with mocked repo', () => {
  let restore: () => void

  beforeEach(() => {
    restore = mockRepository(TOKENS.CategoryRepository, {
      save: vi.fn(),
      findById: vi.fn().mockResolvedValue(null),
    })
  })

  afterEach(() => restore())

  it('uses mocked repository', async () => {
    // Test with mocked DI
  })
})
```

## Test Data Factory

```typescript
// src/test/factories/category-factory.ts
import { faker } from '@faker-js/faker'
import type { CategoryDTO } from '@/backend/application/category/dtos'

export const categoryFactory = {
  dto(overrides: Partial<CategoryDTO> = {}): CategoryDTO {
    return {
      id: faker.string.ulid(),
      name: faker.commerce.department(),
      status: 'active',
      userId: faker.string.ulid(),
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      ...overrides,
    }
  },

  createInput(overrides = {}) {
    return {
      name: faker.commerce.department(),
      ...overrides,
    }
  },
}
```

## Coverage by Layer

| Layer | Priority | Focus |
|-------|----------|-------|
| Entity | High | Business rules, validation |
| Use Case | High | Orchestration logic |
| Repository | Medium | Data persistence |
| Action | Low | Integration verification |

## CI/CD Integration

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      dynamodb-local:
        image: amazon/dynamodb-local:latest
        ports:
          - 8000:8000

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
        env:
          DYNAMODB_LOCAL_ENDPOINT: http://localhost:8000
```

## Running Tests

```bash
# All tests
npx vitest

# By layer
npx vitest src/backend/domain           # Entity tests
npx vitest src/backend/application      # Use Case tests
npx vitest src/backend/infrastructure   # Repository tests
npx vitest src/features                 # E2E tests

# With coverage
npx vitest --coverage
```

## Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|-----------------|
| Testing Entity via database | Pure unit tests, no I/O |
| Mocking Entity internals | Mock Repository interface |
| Direct `new UseCase()` in tests | Use DI Container or explicit injection |
| Cross-feature imports in tests | Mock via DI Container |

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- DynamoDB OneTable: `skills/dynamodb-onetable/SKILL.md`
