---
name: fullstacknextjs
description: Collection of skills for full-stack Next.js development with Clean Architecture and AWS serverless. Backend uses Entities, Use Cases, Repository pattern, and DI Container. Frontend uses thin server action adapters.
---

# Full-Stack Next.js Skills (Clean Architecture)

Umbrella skill containing specialized skills for Next.js + AWS serverless with Clean Architecture patterns.

## Available Skills

| Skill | Use When |
|-------|----------|
| **gs-feature-architecture** | Planning features, understanding Clean Architecture layers |
| **gs-create-domain-module** | Scaffolding new domain modules (Entity, Use Case, Repository) |
| **gs-dynamodb-onetable** | Repository implementations with DynamoDB OneTable |
| **gs-nextjs-server-actions** | Creating thin server action adapters |
| **gs-zod-validation** | Input validation schemas (presentation layer only) |
| **gs-nextjs-web-client** | Building UI components with DTOs |
| **gs-tanstack-react-query** | Data fetching with React Query |
| **gs-create-e2e-tests** | Testing by Clean Architecture layer |
| **gs-sst-infra** | AWS deployment with SST v3 |
| **gs-bun-aws-lambda** | Lambda handlers with Clean Architecture |
| **gs-evaluate-domain-module** | Assessing module compliance |
| **gs-modularity-maturity-assessor** | Codebase maturity scoring |
| **gs-santry-observability** | Sentry integration per layer |

## Quick Start (New Feature)

Follow this order when implementing new features:

1. **gs-create-domain-module** - Scaffold Entity, Repository interface, Use Case, DTO
2. **gs-dynamodb-onetable** - Implement Repository with OneTable
3. **gs-zod-validation** - Define input schemas for actions
4. **gs-nextjs-server-actions** - Create thin action adapters
5. **gs-nextjs-web-client** - Build UI components
6. **gs-create-e2e-tests** - Write tests by layer
7. **gs-sst-infra** - Deploy changes

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
