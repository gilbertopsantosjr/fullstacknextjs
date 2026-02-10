---
name: gs-modularity-maturity-assessor
description: Assesses Clean Architecture compliance and maturity levels. Evaluates Dependency Rule, Entity patterns, Use Cases, Repository pattern, DI Container, and thin actions.
---

# Architecture Maturity Assessor (Clean Architecture)

Evaluates codebase maturity against Clean Architecture principles with weighted scoring.

## Critical Principles (Highest Weight)

| Rank | Principle | Weight | Why Critical |
|------|-----------|--------|--------------|
| 1 | **Dependency Rule** | 1.5x | Foundation - inner layers must not import outer |
| 2 | **Entity Pattern** | 1.2x | Business rules must live in Domain |
| 3 | **Repository Pattern** | 1.2x | Interface in Domain, impl in Infrastructure |
| 4 | **DI Container** | 1.0x | Enables testability and decoupling |

## Run Assessment Script

```bash
echo "========================================"
echo "Clean Architecture Compliance Check"
echo "========================================"

FAILED=0

# P0: Domain imports outer layers
echo -e "\n[P0] Domain Layer Violations..."
VIOLATIONS=$(grep -rn "from '@/backend/application\|from '@/backend/infrastructure\|from '@/features/" src/backend/domain/ 2>/dev/null)
if [ ! -z "$VIOLATIONS" ]; then
  echo "FAIL: Domain importing outer layers:"
  echo "$VIOLATIONS"
  FAILED=1
else
  echo "PASS"
fi

# P0: Application imports Infrastructure
echo -e "\n[P0] Application Layer Violations..."
VIOLATIONS=$(grep -rn "from '@/backend/infrastructure" src/backend/application/ 2>/dev/null)
if [ ! -z "$VIOLATIONS" ]; then
  echo "FAIL: Application importing Infrastructure:"
  echo "$VIOLATIONS"
  FAILED=1
else
  echo "PASS"
fi

# P0: Backend imports Next.js
echo -e "\n[P0] Backend Framework Coupling..."
VIOLATIONS=$(grep -rn "from 'next/\|from 'react\|'use server'" src/backend/ 2>/dev/null)
if [ ! -z "$VIOLATIONS" ]; then
  echo "FAIL: Backend importing Next.js:"
  echo "$VIOLATIONS"
  FAILED=1
else
  echo "PASS"
fi

# P0: Direct instantiation
echo -e "\n[P0] Direct Instantiation..."
VIOLATIONS=$(grep -rn "new.*UseCase(\|new.*Repository(" src/features/ 2>/dev/null)
if [ ! -z "$VIOLATIONS" ]; then
  echo "FAIL: Direct instantiation in features:"
  echo "$VIOLATIONS"
  FAILED=1
else
  echo "PASS"
fi

# P1: Entity pattern
echo -e "\n[P1] Entity Pattern..."
for file in src/backend/domain/*/entities/*.ts; do
  if [ -f "$file" ] && [[ ! "$file" =~ index ]]; then
    if ! grep -q "private constructor" "$file"; then
      echo "WARNING: $file missing private constructor"
    fi
    if ! grep -q "validate()" "$file"; then
      echo "WARNING: $file missing validate()"
    fi
  fi
done
echo "DONE"

# P1: Fat actions
echo -e "\n[P1] Action Sizes..."
find src/features/*/actions -name "*.ts" ! -name "index.ts" -exec wc -l {} \; 2>/dev/null | \
  awk '$1 > 30 {print "WARNING: " $2 " has " $1 " lines (>30)"}'
echo "DONE"

# Summary
echo -e "\n========================================"
if [ $FAILED -eq 1 ]; then
  echo "RESULT: FAILED - Critical violations found"
  exit 1
else
  echo "RESULT: PASSED - No critical violations"
  exit 0
fi
```

## Maturity Scoring

| Principle | Score | Weight | Notes |
|-----------|-------|--------|-------|
| Dependency Rule | X/10 | 1.5x | Domain must not import outer layers |
| Entity Pattern | X/10 | 1.2x | Private constructor + validate() |
| Repository Pattern | X/10 | 1.2x | Interface in Domain |
| Use Case Pattern | X/10 | 1.0x | Classes with execute() |
| DI Container | X/10 | 1.0x | All dependencies registered |
| Thin Actions | X/10 | 1.0x | <15 lines per handler |
| DTOs | X/10 | 0.8x | fromEntity() methods |
| Testing | X/10 | 0.7x | Coverage per layer |
| **TOTAL** | | 9.4x | **X/100** |

## Auto-Fail Conditions

| Violation | Max Score | Reason |
|-----------|-----------|--------|
| Domain imports outer layers | 40% | Dependency Rule broken |
| Backend imports Next.js | 50% | Framework coupling |
| Direct instantiation | 60% | DI not used |
| Entity without private constructor | 70% | Factory pattern missing |

## Maturity Levels

### Immature (0-40%)
- Dependency Rule violations
- No Entity classes
- Direct DB access in actions
- No DI Container

### Developing (41-65%)
- Some Entities properly structured
- Partial Use Case coverage
- Some DI registration
- Fat actions remain

### Mature (66-85%)
- Dependency Rule compliant
- Entities with validate()
- Repository pattern throughout
- DI Container used
- Thin actions

### Advanced (86-100%)
- Zero violations
- All patterns implemented
- Full test coverage
- Excellent separation

## Output Format

```markdown
# Architecture Maturity Assessment

**Score**: {X}/100 ({Level})
**Critical Violations**: {count}

## P0 - Critical
- [ ] {violation + file}

## P1 - High Priority
- [ ] {issue}

## P2 - Medium
- [ ] {improvement}

## Recommendations
1. {Fix highest priority first}
```

## CI/CD Integration

```yaml
name: Architecture Compliance
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dependency Rule
        run: |
          if grep -rn "from '@/backend/application\|from '@/backend/infrastructure" src/backend/domain/; then
            echo "Domain layer violation" && exit 1
          fi
      - name: Backend Framework Coupling
        run: |
          if grep -rn "from 'next/\|from 'react" src/backend/; then
            echo "Backend imports framework" && exit 1
          fi
```

## References

- Feature Architecture: `skills/feature-architecture/SKILL.md`
- Evaluate Module: `skills/evaluate-domain-module/SKILL.md`
