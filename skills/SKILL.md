---
name: fullstacknextjs
description: Collection of skills for full-stack Next.js development with AWS serverless architecture. Use this skill to get an overview of the architecture and find the right specialized skill for your task.
---

# Full-Stack Next.js Skills Collection

This is an umbrella skill containing specialized skills for different concerns in a Next.js + AWS serverless stack.

## Available Skills

| Skill | Use When |
|-------|----------|
| **feature-architecture** | Planning features, understanding layer structure |
| **nextjs-web-client** | Building UI components, forms, pages |
| **nextjs-server-actions** | Creating API endpoints, mutations |
| **dynamodb-onetable** | Database schemas, queries, DAL functions |
| **zod-validation** | Validation schemas, type definitions |
| **sst-infra** | Deployment, infrastructure, CI/CD |

## Quick Start

For new features, use skills in this order:

1. **feature-architecture** - Plan the feature structure
2. **zod-validation** - Define schemas and types
3. **dynamodb-onetable** - Implement DAL functions
4. **nextjs-server-actions** - Create server actions
5. **nextjs-web-client** - Build UI components
6. **sst-infra** - Deploy changes

## Tech Stack

- **Frontend**: Next.js 15+, React 19+, Shadcn UI, Tailwind CSS
- **Backend**: Server Actions (ZSA), Services
- **Database**: DynamoDB (single-table), OneTable ORM
- **Auth**: Better Auth
- **Infrastructure**: SST v3, AWS Lambda, CloudFront
- **Validation**: Zod
