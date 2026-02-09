---
description: Evaluates feature modules for Clean Architecture compliance including Dependency Rule violations, layer separation, Entity patterns, Use Case classes, and DI Container usage.
name: evaluate-domain-module
---

# Feature Module Evaluator (Clean Architecture)

Evaluates features against Clean Architecture principles: Dependency Rule, layer separation, Entity patterns, Use Cases, Repository pattern, DI Container.

## Critical Checks (Must Pass)

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | Domain imports outer layers | `grep -rn "from '@/backend/application\|from '@/backend/infrastructure" src/backend/domain/` | Empty |
| 2 | Application imports Infrastructure | `grep -rn "from '@/backend/infrastructure" src/backend/application/` | Empty |
| 3 | Backend imports Next.js | `grep -rn "from 'next/\|from 'react\|'use server'" src/backend/` | Empty |
| 4 | Direct instantiation | `grep -rn "new.*UseCase(\|new.*Repository(" src/features/` | Empty |
| 5 | Fat server actions | Action files >30 lines | <30 lines each |

## Quick Evaluation Script

```bash
FEATURE="{feature-name}"

echo "=== P0: Dependency Rule Violations ==="
grep -rn "from '@/backend/application\|from '@/backend/infrastructure\|from '@/features/" src/backend/domain/$FEATURE/
grep -rn "from '@/backend/infrastructure" src/backend/application/$FEATURE/
grep -rn "from 'next/\|from 'react\|'use server'" src/backend/

echo "=== P0: Direct Instantiation ==="
grep -rn "new.*UseCase(\|new.*Repository(" src/features/$FEATURE/

echo "=== P1: Entity Pattern ==="
grep -n "private constructor" src/backend/domain/$FEATURE/entities/*.ts
grep -n "validate()" src/backend/domain/$FEATURE/entities/*.ts

echo "=== P1: Repository Pattern ==="
ls src/backend/domain/$FEATURE/repositories/I*.ts 2>/dev/null
ls src/backend/infrastructure/$FEATURE/repositories/*.ts 2>/dev/null

echo "=== P1: Action Sizes ==="
find src/features/$FEATURE/actions -name "*.ts" ! -name "index.ts" -exec wc -l {} \;

echo "=== P2: DI Container ==="
grep -n "$FEATURE" src/backend/infrastructure/di/tokens.ts
```

## Scoring

| Criterion | Weight | Good | Poor |
|-----------|--------|------|------|
| Dependency Rule | 30% | No violations | Any violations |
| Entity Pattern | 20% | Private constructor + validate() | Missing |
| Repository Pattern | 20% | Interface + Implementation | Direct access |
| DI Container | 15% | All registered | None |
| Thin Actions | 15% | <15 lines | >30 lines |

## Auto-Fail Conditions

| Violation | Max Score |
|-----------|-----------|
| Domain imports outer layers | 40% |
| Backend imports Next.js | 50% |
| Direct instantiation in actions | 60% |
| Entity without private constructor | 70% |

## Layer Checklist

### Domain (`src/backend/domain/<feature>/`)
- [ ] Entity with private constructor
- [ ] `static create()` and `fromPersistence()`
- [ ] `validate()` method
- [ ] Repository interface (I*Repository.ts)
- [ ] NO imports from outer layers

### Application (`src/backend/application/<feature>/`)
- [ ] Use Case classes with `execute()`
- [ ] Constructor injection
- [ ] DTOs with `static fromEntity()`
- [ ] NO imports from Infrastructure

### Infrastructure (`src/backend/infrastructure/<feature>/`)
- [ ] Repository implementation
- [ ] DI Container registration

### Presentation (`src/features/<feature>/`)
- [ ] Thin server actions (3-5 lines)
- [ ] Uses DI Container
- [ ] Zod for input shape only

## Output Format

```markdown
# Evaluation: {feature}

## Status: [PASS / FAIL]

### P0 Violations
- [ ] {violation + file}

### P1 Issues
- [ ] {issue}

### Score: {X}/100

## Fix Priority
1. {Highest priority}
2. {Second priority}
```

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- Create Domain Module: `skills/create-domain-module/SKILL.md`
