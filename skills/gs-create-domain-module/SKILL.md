---
description: Creates complete, production-ready feature modules following Clean Architecture OOP principles with Entities, Use Cases, Repository pattern, DI Container, and thin adapter actions. Adapted for Next.js 15+, DynamoDB/OneTable, ZSA, and Vitest.
name: gs-create-domain-module
---

# Feature Module Generator (Clean Architecture OOP)

You are an expert in creating feature modules that comply with Clean Architecture principles, emphasizing the Dependency Rule, Entity-centric design, Use Case classes, Repository pattern, and Dependency Injection.

## Technology Stack

| Component | Technology |
|-----------|------------|
| Framework | Next.js 15+ / React 19+ |
| Database | DynamoDB (single-table design) |
| ORM | OneTable |
| Server Actions | ZSA (Zod Server Actions) |
| Validation | Zod (input shape) + Entity.validate() (business rules) |
| Testing | Vitest |
| ID Generation | ULID |
| Deployment | SST (Serverless Stack) |
| DI Container | Custom lightweight container |

## Core Principles Applied

| Principle | How Applied |
|-----------|-------------|
| **Dependency Rule** | Domain → Application → Infrastructure → Presentation (inward only) |
| **Entity-Centric** | Rich domain objects with behavior, private constructor, factory methods |
| **Use Case Classes** | Single-responsibility classes with `execute()` method |
| **Repository Pattern** | Interface in Domain, implementation in Infrastructure |
| **Dependency Injection** | Constructor injection, resolved via DI Container |
| **Thin Adapters** | Server actions are 3-5 lines, only orchestration |

## When to Use This Skill

Use this skill when:

- Creating a new feature module from scratch
- User asks to "create a feature", "scaffold a feature", or "generate a module"
- User mentions needing a new business domain (categories, accounts, billing, etc.)
- Starting a new bounded context that needs its own entities and logic

## Requirements Gathering

Before generating any code, ask the user these questions:

1. **Feature name** (kebab-case, e.g., "category", "account", "notification")
2. **Initial entities** (comma-separated list, e.g., "Category, Subcategory")
3. **Key business rules** (what validations belong in the Entity?)
4. **Key access patterns** (how will data be queried? e.g., "by userId", "by status")
5. **Cross-feature dependencies** (does this feature depend on others?)

## Folder Structure Generation

Generate this structure for feature modules:

```
src/
├── backend/
│   ├── domain/
│   │   └── <feature>/
│   │       ├── entities/
│   │       │   ├── <Entity>.ts
│   │       │   └── index.ts
│   │       ├── repositories/
│   │       │   ├── I<Entity>Repository.ts
│   │       │   └── index.ts
│   │       ├── value-objects/
│   │       │   └── index.ts
│   │       ├── exceptions/
│   │       │   ├── <Feature>Exceptions.ts
│   │       │   └── index.ts
│   │       └── index.ts
│   │
│   ├── application/
│   │   └── <feature>/
│   │       ├── use-cases/
│   │       │   ├── Create<Entity>UseCase.ts
│   │       │   ├── Update<Entity>UseCase.ts
│   │       │   ├── Delete<Entity>UseCase.ts
│   │       │   ├── Get<Entity>UseCase.ts
│   │       │   ├── List<Entity>sUseCase.ts
│   │       │   └── index.ts
│   │       ├── dtos/
│   │       │   ├── <Entity>DTO.ts
│   │       │   └── index.ts
│   │       └── index.ts
│   │
│   └── infrastructure/
│       └── <feature>/
│           ├── repositories/
│           │   ├── DynamoDB<Entity>Repository.ts
│           │   └── index.ts
│           └── index.ts
│
├── features/
│   └── <feature>/
│       ├── actions/
│       │   ├── create-<entity>-action.ts
│       │   ├── update-<entity>-action.ts
│       │   ├── delete-<entity>-action.ts
│       │   ├── get-<entity>-action.ts
│       │   ├── list-<entity>s-action.ts
│       │   └── index.ts
│       ├── schemas/
│       │   ├── <entity>-schemas.ts
│       │   └── index.ts
│       ├── components/
│       │   └── index.ts
│       └── index.ts
│
└── test/
    └── <feature>/
        ├── entities/
        │   └── <Entity>.test.ts
        ├── use-cases/
        │   └── Create<Entity>UseCase.test.ts
        └── repositories/
            └── DynamoDB<Entity>Repository.test.ts
```

## Component Generation Instructions

### 1. Entity Class (Domain Layer)

```typescript
// src/backend/domain/<feature>/entities/<Entity>.ts
import { ulid } from 'ulid'

export interface <Entity>Props {
  id: string
  userId: string
  name: string
  description?: string
  status: <Entity>Status
  createdAt: Date
  updatedAt: Date
}

export type <Entity>Status = 'active' | 'inactive' | 'archived'

export interface Create<Entity>Input {
  name: string
  description?: string
}

export class <Entity> {
  private constructor(private readonly props: <Entity>Props) {
    this.validate()
  }

  // Factory method for creating new entities
  static create(input: Create<Entity>Input & { userId: string }): <Entity> {
    const now = new Date()
    return new <Entity>({
      id: ulid(),
      userId: input.userId,
      name: input.name,
      description: input.description,
      status: 'active',
      createdAt: now,
      updatedAt: now,
    })
  }

  // Factory method for reconstituting from persistence
  static fromPersistence(data: Record<string, unknown>): <Entity> {
    return new <Entity>({
      id: data.id as string,
      userId: data.userId as string,
      name: data.name as string,
      description: data.description as string | undefined,
      status: data.status as <Entity>Status,
      createdAt: new Date(data.createdAt as string),
      updatedAt: new Date(data.updatedAt as string),
    })
  }

  // Business rule validation (throws DomainException on failure)
  private validate(): void {
    if (!this.props.name || this.props.name.trim().length === 0) {
      throw new <Entity>ValidationException('Name is required')
    }
    if (this.props.name.length > 255) {
      throw new <Entity>ValidationException('Name must be 255 characters or less')
    }
    // Add more business rules here
  }

  // Getters for read-only access
  get id(): string { return this.props.id }
  get userId(): string { return this.props.userId }
  get name(): string { return this.props.name }
  get description(): string | undefined { return this.props.description }
  get status(): <Entity>Status { return this.props.status }
  get createdAt(): Date { return this.props.createdAt }
  get updatedAt(): Date { return this.props.updatedAt }

  // Domain methods with behavior
  updateName(name: string): <Entity> {
    return new <Entity>({
      ...this.props,
      name,
      updatedAt: new Date(),
    })
  }

  archive(): <Entity> {
    if (this.props.status === 'archived') {
      throw new <Entity>AlreadyArchivedException(this.props.id)
    }
    return new <Entity>({
      ...this.props,
      status: 'archived',
      updatedAt: new Date(),
    })
  }

  // Conversion to persistence format
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

### 2. Domain Exceptions

```typescript
// src/backend/domain/<feature>/exceptions/<Feature>Exceptions.ts
import { DomainException } from '@/backend/shared/exceptions'

export class <Entity>ValidationException extends DomainException {
  constructor(message: string) {
    super(message, '<ENTITY>_VALIDATION_ERROR')
  }
}

export class <Entity>NotFoundException extends DomainException {
  constructor(id: string) {
    super(`<Entity> with id ${id} not found`, '<ENTITY>_NOT_FOUND')
  }
}

export class <Entity>AlreadyArchivedException extends DomainException {
  constructor(id: string) {
    super(`<Entity> with id ${id} is already archived`, '<ENTITY>_ALREADY_ARCHIVED')
  }
}

export class <Entity>UnauthorizedException extends DomainException {
  constructor(id: string, userId: string) {
    super(`User ${userId} is not authorized to access <Entity> ${id}`, '<ENTITY>_UNAUTHORIZED')
  }
}
```

### 3. Repository Interface (Domain Layer)

```typescript
// src/backend/domain/<feature>/repositories/I<Entity>Repository.ts
import type { <Entity> } from '../entities/<Entity>'

export interface I<Entity>Repository {
  save(entity: <Entity>): Promise<void>
  findById(id: string, userId: string): Promise<<Entity> | null>
  findByUserId(userId: string, options?: ListOptions): Promise<PaginatedResult<<Entity>>>
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

### 4. DTO (Application Layer)

```typescript
// src/backend/application/<feature>/dtos/<Entity>DTO.ts
import type { <Entity> } from '@/backend/domain/<feature>/entities/<Entity>'

export interface <Entity>DTO {
  id: string
  userId: string
  name: string
  description: string | null
  status: string
  createdAt: string
  updatedAt: string
}

export class <Entity>DTOMapper {
  static fromEntity(entity: <Entity>): <Entity>DTO {
    return {
      id: entity.id,
      userId: entity.userId,
      name: entity.name,
      description: entity.description ?? null,
      status: entity.status,
      createdAt: entity.createdAt.toISOString(),
      updatedAt: entity.updatedAt.toISOString(),
    }
  }

  static fromEntities(entities: <Entity>[]): <Entity>DTO[] {
    return entities.map(this.fromEntity)
  }
}
```

### 5. Use Case Classes (Application Layer)

```typescript
// src/backend/application/<feature>/use-cases/Create<Entity>UseCase.ts
import { <Entity> } from '@/backend/domain/<feature>/entities/<Entity>'
import type { I<Entity>Repository } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { <Entity>DTO, <Entity>DTOMapper } from '../dtos/<Entity>DTO'

export interface Create<Entity>Input {
  name: string
  description?: string
  userId: string
}

export class Create<Entity>UseCase {
  constructor(private readonly <entity>Repository: I<Entity>Repository) {}

  async execute(input: Create<Entity>Input): Promise<<Entity>DTO> {
    const entity = <Entity>.create({
      name: input.name,
      description: input.description,
      userId: input.userId,
    })

    await this.<entity>Repository.save(entity)

    return <Entity>DTOMapper.fromEntity(entity)
  }
}
```

```typescript
// src/backend/application/<feature>/use-cases/Get<Entity>UseCase.ts
import type { I<Entity>Repository } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { <Entity>NotFoundException, <Entity>UnauthorizedException } from '@/backend/domain/<feature>/exceptions'
import { <Entity>DTO, <Entity>DTOMapper } from '../dtos/<Entity>DTO'

export interface Get<Entity>Input {
  id: string
  userId: string
}

export class Get<Entity>UseCase {
  constructor(private readonly <entity>Repository: I<Entity>Repository) {}

  async execute(input: Get<Entity>Input): Promise<<Entity>DTO> {
    const entity = await this.<entity>Repository.findById(input.id, input.userId)

    if (!entity) {
      throw new <Entity>NotFoundException(input.id)
    }

    if (entity.userId !== input.userId) {
      throw new <Entity>UnauthorizedException(input.id, input.userId)
    }

    return <Entity>DTOMapper.fromEntity(entity)
  }
}
```

```typescript
// src/backend/application/<feature>/use-cases/List<Entity>sUseCase.ts
import type { I<Entity>Repository, ListOptions } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { <Entity>DTOMapper, type <Entity>DTO } from '../dtos/<Entity>DTO'

export interface List<Entity>sInput {
  userId: string
  limit?: number
  cursor?: string
  status?: string
}

export interface List<Entity>sOutput {
  items: <Entity>DTO[]
  nextCursor?: string
  hasMore: boolean
}

export class List<Entity>sUseCase {
  constructor(private readonly <entity>Repository: I<Entity>Repository) {}

  async execute(input: List<Entity>sInput): Promise<List<Entity>sOutput> {
    const options: ListOptions = {
      limit: input.limit ?? 20,
      cursor: input.cursor,
      status: input.status,
    }

    const result = await this.<entity>Repository.findByUserId(input.userId, options)

    return {
      items: <Entity>DTOMapper.fromEntities(result.items),
      nextCursor: result.nextCursor,
      hasMore: result.hasMore,
    }
  }
}
```

```typescript
// src/backend/application/<feature>/use-cases/Update<Entity>UseCase.ts
import type { I<Entity>Repository } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { <Entity>NotFoundException, <Entity>UnauthorizedException } from '@/backend/domain/<feature>/exceptions'
import { <Entity>DTO, <Entity>DTOMapper } from '../dtos/<Entity>DTO'

export interface Update<Entity>Input {
  id: string
  userId: string
  name?: string
  description?: string
}

export class Update<Entity>UseCase {
  constructor(private readonly <entity>Repository: I<Entity>Repository) {}

  async execute(input: Update<Entity>Input): Promise<<Entity>DTO> {
    let entity = await this.<entity>Repository.findById(input.id, input.userId)

    if (!entity) {
      throw new <Entity>NotFoundException(input.id)
    }

    if (entity.userId !== input.userId) {
      throw new <Entity>UnauthorizedException(input.id, input.userId)
    }

    if (input.name) {
      entity = entity.updateName(input.name)
    }

    await this.<entity>Repository.save(entity)

    return <Entity>DTOMapper.fromEntity(entity)
  }
}
```

```typescript
// src/backend/application/<feature>/use-cases/Delete<Entity>UseCase.ts
import type { I<Entity>Repository } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { <Entity>NotFoundException, <Entity>UnauthorizedException } from '@/backend/domain/<feature>/exceptions'

export interface Delete<Entity>Input {
  id: string
  userId: string
}

export class Delete<Entity>UseCase {
  constructor(private readonly <entity>Repository: I<Entity>Repository) {}

  async execute(input: Delete<Entity>Input): Promise<void> {
    const entity = await this.<entity>Repository.findById(input.id, input.userId)

    if (!entity) {
      throw new <Entity>NotFoundException(input.id)
    }

    if (entity.userId !== input.userId) {
      throw new <Entity>UnauthorizedException(input.id, input.userId)
    }

    await this.<entity>Repository.delete(input.id, input.userId)
  }
}
```

### 6. Repository Implementation (Infrastructure Layer)

```typescript
// src/backend/infrastructure/<feature>/repositories/DynamoDB<Entity>Repository.ts
import { getDynamoDbTable } from '@/backend/infrastructure/database/db-config'
import { <Entity> } from '@/backend/domain/<feature>/entities/<Entity>'
import type { I<Entity>Repository, ListOptions, PaginatedResult } from '@/backend/domain/<feature>/repositories/I<Entity>Repository'
import { log } from '@/lib/logger'

export class DynamoDB<Entity>Repository implements I<Entity>Repository {
  private getModel() {
    return getDynamoDbTable().getModel('<Entity>')
  }

  async save(entity: <Entity>): Promise<void> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      await Model.upsert(entity.toPersistence())

      log.debug('[<Entity>Repository.save] Success', {
        id: entity.id,
        duration: Date.now() - startTime,
      })
    } catch (error) {
      log.error('[<Entity>Repository.save] Failed', { error, id: entity.id })
      throw error
    }
  }

  async findById(id: string, userId: string): Promise<<Entity> | null> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      const data = await Model.get({
        pk: `USER#${userId}`,
        sk: `<FEATURE>#<entity>#${id}`,
      })

      log.debug('[<Entity>Repository.findById] Complete', {
        id,
        found: !!data,
        duration: Date.now() - startTime,
      })

      if (!data) return null

      return <Entity>.fromPersistence(data)
    } catch (error) {
      log.error('[<Entity>Repository.findById] Failed', { error, id })
      throw error
    }
  }

  async findByUserId(userId: string, options: ListOptions = {}): Promise<PaginatedResult<<Entity>>> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      const limit = options.limit ?? 20

      const queryOptions: any = {
        pk: `USER#${userId}`,
        sk: { begins: '<FEATURE>#<entity>#' },
        limit: limit + 1, // Fetch one extra to check for more
      }

      if (options.cursor) {
        queryOptions.start = JSON.parse(Buffer.from(options.cursor, 'base64').toString())
      }

      const results = await Model.find(queryOptions)

      const hasMore = results.length > limit
      const items = hasMore ? results.slice(0, limit) : results

      const nextCursor = hasMore && results[limit - 1]
        ? Buffer.from(JSON.stringify({ pk: results[limit - 1].pk, sk: results[limit - 1].sk })).toString('base64')
        : undefined

      log.debug('[<Entity>Repository.findByUserId] Complete', {
        userId,
        count: items.length,
        hasMore,
        duration: Date.now() - startTime,
      })

      return {
        items: items.map(data => <Entity>.fromPersistence(data)),
        nextCursor,
        hasMore,
      }
    } catch (error) {
      log.error('[<Entity>Repository.findByUserId] Failed', { error, userId })
      throw error
    }
  }

  async delete(id: string, userId: string): Promise<void> {
    const startTime = Date.now()
    try {
      const Model = this.getModel()
      await Model.remove({
        pk: `USER#${userId}`,
        sk: `<FEATURE>#<entity>#${id}`,
      })

      log.debug('[<Entity>Repository.delete] Success', {
        id,
        duration: Date.now() - startTime,
      })
    } catch (error) {
      log.error('[<Entity>Repository.delete] Failed', { error, id })
      throw error
    }
  }
}
```

### 7. DI Container Registration

```typescript
// src/backend/infrastructure/di/tokens.ts
export const TOKENS = {
  // Repositories
  <Entity>Repository: Symbol.for('<Entity>Repository'),

  // Use Cases
  Create<Entity>UseCase: Symbol.for('Create<Entity>UseCase'),
  Get<Entity>UseCase: Symbol.for('Get<Entity>UseCase'),
  List<Entity>sUseCase: Symbol.for('List<Entity>sUseCase'),
  Update<Entity>UseCase: Symbol.for('Update<Entity>UseCase'),
  Delete<Entity>UseCase: Symbol.for('Delete<Entity>UseCase'),
} as const
```

```typescript
// src/backend/infrastructure/di/container.ts
import { DynamoDB<Entity>Repository } from '../<feature>/repositories/DynamoDB<Entity>Repository'
import { Create<Entity>UseCase } from '@/backend/application/<feature>/use-cases/Create<Entity>UseCase'
import { Get<Entity>UseCase } from '@/backend/application/<feature>/use-cases/Get<Entity>UseCase'
import { List<Entity>sUseCase } from '@/backend/application/<feature>/use-cases/List<Entity>sUseCase'
import { Update<Entity>UseCase } from '@/backend/application/<feature>/use-cases/Update<Entity>UseCase'
import { Delete<Entity>UseCase } from '@/backend/application/<feature>/use-cases/Delete<Entity>UseCase'
import { TOKENS } from './tokens'

class Container {
  private instances = new Map<symbol, unknown>()
  private factories = new Map<symbol, () => unknown>()

  register<T>(token: symbol, factory: () => T): void {
    this.factories.set(token, factory)
  }

  resolve<T>(token: symbol): T {
    if (this.instances.has(token)) {
      return this.instances.get(token) as T
    }

    const factory = this.factories.get(token)
    if (!factory) {
      throw new Error(`No factory registered for token: ${String(token)}`)
    }

    const instance = factory() as T
    this.instances.set(token, instance)
    return instance
  }
}

export const DIContainer = new Container()

// Register repositories (singletons)
DIContainer.register(TOKENS.<Entity>Repository, () => new DynamoDB<Entity>Repository())

// Register use cases (with injected dependencies)
DIContainer.register(TOKENS.Create<Entity>UseCase, () =>
  new Create<Entity>UseCase(DIContainer.resolve(TOKENS.<Entity>Repository))
)
DIContainer.register(TOKENS.Get<Entity>UseCase, () =>
  new Get<Entity>UseCase(DIContainer.resolve(TOKENS.<Entity>Repository))
)
DIContainer.register(TOKENS.List<Entity>sUseCase, () =>
  new List<Entity>sUseCase(DIContainer.resolve(TOKENS.<Entity>Repository))
)
DIContainer.register(TOKENS.Update<Entity>UseCase, () =>
  new Update<Entity>UseCase(DIContainer.resolve(TOKENS.<Entity>Repository))
)
DIContainer.register(TOKENS.Delete<Entity>UseCase, () =>
  new Delete<Entity>UseCase(DIContainer.resolve(TOKENS.<Entity>Repository))
)
```

### 8. Input Schemas (Presentation Layer - Zod for shape only)

```typescript
// src/features/<feature>/schemas/<entity>-schemas.ts
import { z } from 'zod'

// Input shape validation ONLY - business rules are in Entity.validate()
export const Create<Entity>Schema = z.object({
  name: z.string().min(1, 'Name is required'),
  description: z.string().optional(),
})

export type Create<Entity>Input = z.infer<typeof Create<Entity>Schema>

export const Update<Entity>Schema = z.object({
  id: z.string().ulid(),
  name: z.string().min(1).optional(),
  description: z.string().optional(),
})

export type Update<Entity>Input = z.infer<typeof Update<Entity>Schema>

export const Get<Entity>Schema = z.object({
  id: z.string().ulid(),
})

export const List<Entity>sSchema = z.object({
  limit: z.coerce.number().min(1).max(100).optional(),
  cursor: z.string().optional(),
  status: z.enum(['active', 'inactive', 'archived']).optional(),
})

export const Delete<Entity>Schema = z.object({
  id: z.string().ulid(),
})
```

### 9. Thin Server Action Adapters (Presentation Layer)

```typescript
// src/features/<feature>/actions/create-<entity>-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { Create<Entity>Schema } from '../schemas/<entity>-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { Create<Entity>UseCase } from '@/backend/application/<feature>/use-cases'

export const create<Entity>Action = authedProcedure
  .createServerAction()
  .input(Create<Entity>Schema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<Create<Entity>UseCase>(TOKENS.Create<Entity>UseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

```typescript
// src/features/<feature>/actions/get-<entity>-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { Get<Entity>Schema } from '../schemas/<entity>-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { Get<Entity>UseCase } from '@/backend/application/<feature>/use-cases'

export const get<Entity>Action = authedProcedure
  .createServerAction()
  .input(Get<Entity>Schema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<Get<Entity>UseCase>(TOKENS.Get<Entity>UseCase)
    return useCase.execute({ id: input.id, userId: ctx.user.id })
  })
```

```typescript
// src/features/<feature>/actions/list-<entity>s-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { List<Entity>sSchema } from '../schemas/<entity>-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { List<Entity>sUseCase } from '@/backend/application/<feature>/use-cases'

export const list<Entity>sAction = authedProcedure
  .createServerAction()
  .input(List<Entity>sSchema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<List<Entity>sUseCase>(TOKENS.List<Entity>sUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

```typescript
// src/features/<feature>/actions/update-<entity>-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { Update<Entity>Schema } from '../schemas/<entity>-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { Update<Entity>UseCase } from '@/backend/application/<feature>/use-cases'

export const update<Entity>Action = authedProcedure
  .createServerAction()
  .input(Update<Entity>Schema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<Update<Entity>UseCase>(TOKENS.Update<Entity>UseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

```typescript
// src/features/<feature>/actions/delete-<entity>-action.ts
'use server'
import 'server-only'
import { authedProcedure } from '@/lib/zsa'
import { Delete<Entity>Schema } from '../schemas/<entity>-schemas'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
import type { Delete<Entity>UseCase } from '@/backend/application/<feature>/use-cases'

export const delete<Entity>Action = authedProcedure
  .createServerAction()
  .input(Delete<Entity>Schema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<Delete<Entity>UseCase>(TOKENS.Delete<Entity>UseCase)
    return useCase.execute({ id: input.id, userId: ctx.user.id })
  })
```

### 10. Feature Index (Public Exports)

```typescript
// src/features/<feature>/index.ts

// Actions (for use in components)
export {
  create<Entity>Action,
  get<Entity>Action,
  list<Entity>sAction,
  update<Entity>Action,
  delete<Entity>Action,
} from './actions'

// DTOs (for TypeScript consumers)
export type { <Entity>DTO } from '@/backend/application/<feature>/dtos'

// Input types (for forms)
export type {
  Create<Entity>Input,
  Update<Entity>Input,
} from './schemas/<entity>-schemas'

// --------------------------------------------------------
// NEVER export: Entities, Repositories, Use Cases directly
// Consumers use actions to interact with the feature
// --------------------------------------------------------
```

## Naming Conventions

| Layer | Type | Convention | Example |
|-------|------|------------|---------|
| Domain | Entities | PascalCase | `Category.ts` |
| Domain | Interfaces | I + PascalCase | `ICategoryRepository.ts` |
| Domain | Exceptions | PascalCase + Exception | `CategoryNotFoundException.ts` |
| Application | Use Cases | VerbNounUseCase | `CreateCategoryUseCase.ts` |
| Application | DTOs | PascalCase + DTO | `CategoryDTO.ts` |
| Infrastructure | Implementations | Prefix + PascalCase | `DynamoDBCategoryRepository.ts` |
| Presentation | Actions | kebab-case | `create-category-action.ts` |
| Presentation | Schemas | kebab-case | `category-schemas.ts` |

## Anti-Patterns to Avoid

### Domain Importing Outer Layers (CRITICAL)

**BAD**:
```typescript
// src/backend/domain/category/entities/Category.ts
import { getDynamoDbTable } from '@/backend/infrastructure/database' // VIOLATION!
```

**GOOD**:
```typescript
// Domain has NO external imports
export class Category {
  private constructor(private readonly props: CategoryProps) {}
  // ...
}
```

### Fat Server Actions (CRITICAL)

**BAD**:
```typescript
export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)
  .handler(async ({ input, ctx }) => {
    // Validation logic here...
    // Business rules here...
    // Database operations here...
    // 50+ lines
  })
```

**GOOD**:
```typescript
export const createCategoryAction = authedProcedure
  .createServerAction()
  .input(CreateCategorySchema)
  .handler(async ({ input, ctx }) => {
    const useCase = DIContainer.resolve<CreateCategoryUseCase>(TOKENS.CreateCategoryUseCase)
    return useCase.execute({ ...input, userId: ctx.user.id })
  })
```

### Direct Instantiation Instead of DI

**BAD**:
```typescript
// src/features/category/actions/create-category-action.ts
const repository = new DynamoDBCategoryRepository() // VIOLATION!
const useCase = new CreateCategoryUseCase(repository) // VIOLATION!
```

**GOOD**:
```typescript
const useCase = DIContainer.resolve<CreateCategoryUseCase>(TOKENS.CreateCategoryUseCase)
```

### Business Rules in Zod Schemas

**BAD**:
```typescript
// Zod doing business validation
export const CreateCategorySchema = z.object({
  name: z.string()
    .min(1)
    .max(255)
    .refine(name => !name.includes('banned'), 'Name contains banned words'), // Business rule!
})
```

**GOOD**:
```typescript
// Zod for shape only
export const CreateCategorySchema = z.object({
  name: z.string().min(1),
})

// Business rules in Entity
export class Category {
  private validate(): void {
    if (this.props.name.includes('banned')) {
      throw new CategoryValidationException('Name contains banned words')
    }
  }
}
```

## Verification Commands

After generating the feature, run these verification commands:

### Critical Checks (P0 - Must Pass)

```bash
FEATURE="{feature-name}"

# 1. Domain importing outer layers (MUST be empty)
echo "=== Domain Layer Violations ==="
grep -rn "from '@/backend/application\|from '@/backend/infrastructure\|from '@/features/" src/backend/domain/$FEATURE/

# 2. Application importing Infrastructure (MUST be empty)
echo "=== Application Layer Violations ==="
grep -rn "from '@/backend/infrastructure" src/backend/application/$FEATURE/

# 3. Backend importing Next.js (MUST be empty)
echo "=== Next.js in Backend Violations ==="
grep -rn "from 'next/\|from 'react\|'use server'" src/backend/

# 4. Direct instantiation in features (MUST be empty)
echo "=== Direct Instantiation Violations ==="
grep -rn "new.*UseCase(\|new.*Repository(" src/features/$FEATURE/

# 5. Fat actions (should be <10 lines in handler)
echo "=== Action Handler Sizes ==="
find src/features/$FEATURE/actions -name "*.ts" ! -name "index.ts" -exec wc -l {} \;
```

### High Priority Checks (P1)

```bash
# 6. Entity has private constructor
echo "=== Entity Constructor Check ==="
grep -n "constructor" src/backend/domain/$FEATURE/entities/*.ts

# 7. Entity has validate method
echo "=== Entity Validate Method ==="
grep -n "validate()" src/backend/domain/$FEATURE/entities/*.ts

# 8. Repository interface exists
echo "=== Repository Interface ==="
ls src/backend/domain/$FEATURE/repositories/I*.ts

# 9. DI tokens registered
echo "=== DI Tokens ==="
grep "$FEATURE" src/backend/infrastructure/di/tokens.ts
```

## Generation Process

Follow these steps in order:

1. **Gather requirements** using AskQuestion tool
2. **Create folder structure** with all necessary directories
3. **Generate Entity** with private constructor, factory methods, validate()
4. **Generate Repository Interface** in Domain layer
5. **Generate DTO** with static fromEntity()
6. **Generate Use Cases** with constructor injection
7. **Generate Repository Implementation** in Infrastructure
8. **Register in DI Container** (tokens + factories)
9. **Generate Zod Schemas** for input shape validation
10. **Generate Server Action Adapters** (3-5 lines each)
11. **Generate Feature Index** with public exports
12. **Run verification commands** to check compliance
13. **Report results** to user with next steps

## Success Criteria

A successfully generated feature should:

- Pass all verification commands (no violations)
- Have Entity with private constructor and validate()
- Have Repository interface in Domain, implementation in Infrastructure
- Have Use Cases with constructor injection
- Have thin server action adapters (3-5 lines)
- Use DI Container for all instantiation
- Have Zod schemas for input shape only (not business rules)
- Export only DTOs and actions from feature index

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- DynamoDB Patterns: `skills/dynamodb-onetable/SKILL.md`
- Server Actions: `skills/nextjs-server-actions/SKILL.md`
- Zod Validation: `skills/zod-validation/SKILL.md`
