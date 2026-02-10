---
name: gs-feature-architecture
description: Guide for implementing features in Clean Architecture OOP with Next.js. Use when planning new features, understanding the 4-layer structure (Domain, Application, Infrastructure, Presentation), or deciding where code should live.
---

# Feature Architecture (Clean Architecture OOP)

## Layer Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Presentation Layer (src/features/, src/app/)                   │
│  Server Actions (thin adapters), Components, Pages              │
├─────────────────────────────────────────────────────────────────┤
│  Application Layer (src/backend/application/)                   │
│  Use Cases, DTOs, Application Services                          │
├─────────────────────────────────────────────────────────────────┤
│  Infrastructure Layer (src/backend/infrastructure/)             │
│  Repository Implementations, External Services, DI Container    │
├─────────────────────────────────────────────────────────────────┤
│  Domain Layer (src/backend/domain/)                             │
│  Entities, Value Objects, Repository Interfaces, Domain Events  │
└─────────────────────────────────────────────────────────────────┘
```

## The Dependency Rule

**CRITICAL**: Dependencies only point inward. Inner layers NEVER import from outer layers.

```
Domain ← Application ← Infrastructure ← Presentation
```

| Layer | Can Import From | CANNOT Import From |
|-------|-----------------|-------------------|
| Domain | Nothing | Application, Infrastructure, Presentation |
| Application | Domain | Infrastructure, Presentation |
| Infrastructure | Domain, Application | Presentation |
| Presentation | All layers | - |

## Implementation Order

**Always implement from inside-out (Domain first):**

1. **Entity** - Domain object with private constructor, validate(), factory methods
2. **Repository Interface** - Contract in Domain layer
3. **Use Case** - Business logic with constructor injection
4. **DTO** - Data transfer object with `static fromEntity()`
5. **Repository Implementation** - DynamoDB/external service in Infrastructure
6. **DI Registration** - Wire dependencies in Container
7. **Action Adapter** - Thin server action (3-5 lines)
8. **Components** - UI consuming DTOs

## Directory Structure

```
src/
├── backend/
│   ├── domain/
│   │   └── <feature>/
│   │       ├── entities/
│   │       │   └── <Entity>.ts           # Entity class with validate()
│   │       ├── repositories/
│   │       │   └── I<Entity>Repository.ts # Interface only
│   │       ├── value-objects/
│   │       │   └── <ValueObject>.ts       # Immutable value types
│   │       └── exceptions/
│   │           └── <Feature>Exceptions.ts # Domain exceptions
│   │
│   ├── application/
│   │   └── <feature>/
│   │       ├── use-cases/
│   │       │   ├── Create<Entity>UseCase.ts
│   │       │   ├── Update<Entity>UseCase.ts
│   │       │   ├── Delete<Entity>UseCase.ts
│   │       │   └── Get<Entity>UseCase.ts
│   │       └── dtos/
│   │           └── <Entity>DTO.ts         # Static fromEntity()
│   │
│   ├── infrastructure/
│   │   ├── <feature>/
│   │   │   └── repositories/
│   │   │       └── DynamoDB<Entity>Repository.ts
│   │   └── di/
│   │       ├── container.ts              # DI Container
│   │       └── tokens.ts                 # Injection tokens
│   │
│   └── shared/
│       ├── exceptions/
│       │   └── BaseException.ts
│       └── types/
│           └── index.ts
│
├── features/
│   └── <feature>/
│       ├── actions/
│       │   ├── create-<entity>-action.ts  # Thin adapter
│       │   ├── update-<entity>-action.ts
│       │   └── index.ts
│       ├── components/
│       │   ├── <Component>.tsx
│       │   └── <Component>-client.tsx
│       └── index.ts                       # Public exports (DTOs, actions)
│
└── app/
    └── (dashboard)/
        └── <feature>/
            └── page.tsx
```

## Layer Responsibilities

### Domain Layer (Innermost)
- **Entities**: Business objects with behavior, private constructor, `create()`, `fromPersistence()`, `validate()`, `toDTO()`
- **Repository Interfaces**: Contracts for data access (e.g., `ICategoryRepository`)
- **Value Objects**: Immutable objects representing concepts (e.g., `Money`, `Email`)
- **Domain Exceptions**: Business rule violations
- **NO** external dependencies, **NO** frameworks

### Application Layer
- **Use Cases**: Single-responsibility business operations with `execute()` method
- **DTOs**: Data structures for crossing boundaries, `static fromEntity()`
- **Application Services**: Orchestration when needed
- Uses **constructor injection** for dependencies
- **NO** direct database access, **NO** HTTP concerns

### Infrastructure Layer
- **Repository Implementations**: DynamoDB, external APIs
- **DI Container**: Dependency injection setup
- **External Service Adapters**: Email, payment, etc.
- **Throws exceptions** on errors (no `{success, data?, error?}` pattern)

### Presentation Layer
- **Server Actions**: Thin adapters (3-5 lines), resolve Use Case from DI, execute, return
- **Components**: React components receiving DTOs
- **Pages**: Route definitions
- **Zod Schemas**: Input shape validation only (not business rules)

## Data Flow

```
User Action
    ↓
Component (client)
    ↓
Server Action (thin adapter)
    ↓
    ├── Resolve Use Case from DI Container
    ├── Execute Use Case
    └── Return DTO
    ↓
Use Case (Application Layer)
    ↓
    ├── Create/Load Entity
    ├── Entity.validate() (business rules)
    └── Repository.save(entity)
    ↓
Repository Implementation (Infrastructure)
    ↓
DynamoDB
```

## Quick Reference

| Need to... | Go to... |
|------------|----------|
| Add business rules | `src/backend/domain/<feature>/entities/<Entity>.ts` → `validate()` |
| Add data access contract | `src/backend/domain/<feature>/repositories/I<Entity>Repository.ts` |
| Add business operation | `src/backend/application/<feature>/use-cases/` |
| Add data shape for UI | `src/backend/application/<feature>/dtos/<Entity>DTO.ts` |
| Implement repository | `src/backend/infrastructure/<feature>/repositories/` |
| Add API endpoint | `src/features/<feature>/actions/` (thin adapter) |
| Add UI element | `src/features/<feature>/components/` |
| Add page route | `src/app/` |
| Add input validation | `src/features/<feature>/schemas/` (Zod for shape only) |

## Naming Conventions

| Layer | Type | Convention | Example |
|-------|------|------------|---------|
| Domain | Entities | PascalCase | `Category.ts` |
| Domain | Interfaces | IPascalCase | `ICategoryRepository.ts` |
| Domain | Exceptions | PascalCase + Exception | `CategoryNotFoundException.ts` |
| Application | Use Cases | VerbNounUseCase | `CreateCategoryUseCase.ts` |
| Application | DTOs | PascalCase + DTO | `CategoryDTO.ts` |
| Infrastructure | Implementations | PrefixPascalCase | `DynamoDBCategoryRepository.ts` |
| Presentation | Actions | kebab-case | `create-category-action.ts` |
| Presentation | Components | PascalCase | `CategoryCard.tsx` |
| Presentation | Schemas | PascalCase + Schema | `CreateCategorySchema.ts` |

## Import Restrictions (Dependency Rule)

```typescript
// ❌ VIOLATION: Domain importing from Application
// src/backend/domain/category/entities/Category.ts
import { CategoryDTO } from '@/backend/application/category/dtos' // WRONG!

// ❌ VIOLATION: Domain importing from Infrastructure
// src/backend/domain/category/entities/Category.ts
import { getDynamoDbTable } from '@/backend/infrastructure/database' // WRONG!

// ❌ VIOLATION: Application importing from Infrastructure
// src/backend/application/category/use-cases/CreateCategoryUseCase.ts
import { DynamoDBCategoryRepository } from '@/backend/infrastructure' // WRONG!

// ✅ CORRECT: Application imports from Domain
// src/backend/application/category/use-cases/CreateCategoryUseCase.ts
import { Category } from '@/backend/domain/category/entities'
import type { ICategoryRepository } from '@/backend/domain/category/repositories'

// ✅ CORRECT: Infrastructure imports from Domain and Application
// src/backend/infrastructure/category/repositories/DynamoDBCategoryRepository.ts
import type { ICategoryRepository } from '@/backend/domain/category/repositories'
import { Category } from '@/backend/domain/category/entities'

// ✅ CORRECT: Presentation imports from all inner layers
// src/features/category/actions/create-category-action.ts
import { CreateCategoryUseCase } from '@/backend/application/category/use-cases'
import { DIContainer, TOKENS } from '@/backend/infrastructure/di'
```

## Detection Commands

```bash
# Domain importing from outer layers (CRITICAL VIOLATION)
grep -rn "from '@/backend/application\|from '@/backend/infrastructure\|from '@/features/" src/backend/domain/

# Application importing from Infrastructure (VIOLATION)
grep -rn "from '@/backend/infrastructure" src/backend/application/

# Backend importing Next.js (VIOLATION)
grep -rn "from 'next/\|from 'react\|'use server'" src/backend/

# Fat server actions (direct DB access - VIOLATION)
grep -rn "getDynamoDbTable\|getModel" src/features/*/actions/

# Direct instantiation instead of DI (VIOLATION)
grep -rn "new.*UseCase(\|new.*Repository(" src/features/

# Entity without private constructor (VIOLATION)
grep -rn "export class" src/backend/domain/*/entities/ -A5 | grep -v "private constructor"
```

## Related Skills

- **create-domain-module** - Generate complete feature with Clean Architecture
- **zod-validation** - Input shape validation in Presentation layer
- **dynamodb-onetable** - Repository implementation patterns
- **nextjs-server-actions** - Thin adapter action patterns
- **nextjs-web-client** - Component patterns with DTOs
- **evaluate-domain-module** - Assess Clean Architecture compliance
