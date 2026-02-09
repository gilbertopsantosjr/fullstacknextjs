---
name: fullstacknextjs
description: Collection of skills for full-stack Next.js development with Clean Architecture and AWS serverless. Backend uses Entities, Use Cases, Repository pattern, and DI Container. Frontend uses thin server action adapters.
---

# Full-Stack Next.js Skills (Clean Architecture)

Umbrella skill containing specialized skills for Next.js + AWS serverless with Clean Architecture patterns.

## Available Skills

| Skill | Use When |
|-------|----------|
| **feature-architecture** | Planning features, understanding Clean Architecture layers |
| **create-domain-module** | Scaffolding new domain modules (Entity, Use Case, Repository) |
| **dynamodb-onetable** | Repository implementations with DynamoDB OneTable |
| **nextjs-server-actions** | Creating thin server action adapters |
| **zod-validation** | Input validation schemas (presentation layer only) |
| **nextjs-web-client** | Building UI components with DTOs |
| **tanstack-react-query** | Data fetching with React Query |
| **create-e2e-tests** | Testing by Clean Architecture layer |
| **sst-infra** | AWS deployment with SST v3 |
| **bun-aws-lambda** | Lambda handlers with Clean Architecture |
| **evaluate-domain-module** | Assessing module compliance |
| **modularity-maturity-assessor** | Codebase maturity scoring |
| **santry-observability** | Sentry integration per layer |

## Quick Start (New Feature)

Follow this order when implementing new features:

1. **create-domain-module** - Scaffold Entity, Repository interface, Use Case, DTO
2. **dynamodb-onetable** - Implement Repository with OneTable
3. **zod-validation** - Define input schemas for actions
4. **nextjs-server-actions** - Create thin action adapters
5. **nextjs-web-client** - Build UI components
6. **create-e2e-tests** - Write tests by layer
7. **sst-infra** - Deploy changes

## Architecture Layers

```
┌─────────────────────────────────────────┐
│ Presentation (Next.js, Actions, UI)     │
├─────────────────────────────────────────┤
│ Application (Use Cases, DTOs)           │
├─────────────────────────────────────────┤
│ Domain (Entities, Repository Interfaces)│
├─────────────────────────────────────────┤
│ Infrastructure (DynamoDB, External APIs)│
└─────────────────────────────────────────┘
```

**Dependency Rule**: Inner layers NEVER import from outer layers.

## Tech Stack

- **Frontend**: Next.js 15+, React 19+, Shadcn UI, Tailwind CSS
- **Backend**: Clean Architecture (Entities, Use Cases, Repositories)
- **Database**: DynamoDB (single-table), OneTable ORM
- **Actions**: ZSA (thin adapters to Use Cases)
- **DI**: Custom DI Container with token registration
- **Auth**: Better Auth
- **Infrastructure**: SST v3, AWS Lambda, CloudFront
- **Validation**: Zod (presentation) + Entity.validate() (domain)
