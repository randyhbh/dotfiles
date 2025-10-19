---
name: database-migration-safety
description: Use when creating database migrations to ensure zero-downtime deployments and backward compatibility
---

# Database Migration Safety

## Overview

Database migrations must never break running instances. Schema changes must be deployed incrementally with backward compatibility.

**Core principle:** ALWAYS ensure backward compatibility - running instances must not break.

## When to Use

**Mandatory:**
- Creating any Flyway migration
- Modifying database schema
- Before committing migration files
- During code review of migrations

**Use especially when:**
- Adding NOT NULL constraints
- Dropping columns
- Changing column types
- Adding foreign keys
- Renaming tables/columns

## The Iron Law

```
NO BREAKING CHANGES IN A SINGLE MIGRATION
```

If old code would break with new schema, split into multiple migrations.

## Migration Checklist

**Before creating migration, use TodoWrite for these items:**

- [ ] Migration 1: Add column with default value (safe to deploy)
- [ ] Verify old code works with new column
- [ ] Deploy Migration 1 to all instances
- [ ] Migration 2: Populate data where NULL (if needed)
- [ ] Migration 3: Add NOT NULL constraint (if needed)
- [ ] Test migration on copy of production data
- [ ] Verify migration naming follows convention

## Safe Migration Pattern

### Adding NOT NULL Column

**WRONG (breaks running instances):**
```sql
-- ❌ BAD: This will fail if old code tries to insert without new column
ALTER TABLE orders ADD COLUMN status TEXT NOT NULL;
```

**CORRECT (three-step deployment):**

**Migration 1** - Deploy first:
```sql
-- V2024011510300000__add_order_status_column.sql
-- Safe: Old code can still insert (default provides value)
ALTER TABLE orders ADD COLUMN status TEXT DEFAULT 'PENDING';
```

**Wait for all instances to update**, then:

**Migration 2** - Populate data:
```sql
-- V2024011610300000__populate_order_status.sql
-- Backfill existing data
UPDATE orders
SET status = 'COMPLETED'
WHERE completed_at IS NOT NULL;

UPDATE orders
SET status = 'CANCELLED'
WHERE cancelled_at IS NOT NULL;
```

**Migration 3** - Add constraint:
```sql
-- V2024011710300000__add_order_status_constraint.sql
-- Now safe: All rows have values, new code deployed
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

### Dropping Columns

**Never drop in one step.** Follow this sequence:

1. **Remove code references** to column
2. **Deploy new code** (column still exists but unused)
3. **Migration to drop column** (safe, no code references it)

### Changing Column Types

**Expand then contract:**

1. **Add new column** with new type
2. **Populate new column** from old column
3. **Update code** to use new column
4. **Deploy code**
5. **Drop old column** (separate migration)

### Renaming Tables/Columns

**Use views for zero-downtime:**

1. **Rename physical table/column**
2. **Create view** with old name
3. **Update code** to use new name
4. **Deploy code**
5. **Drop view** (separate migration)

## Flyway Conventions (Spring Boot Project)

### Migration Naming
- **Format**: `VYYYYMMDDHHMMSSNN__description.sql`
- **Example**: `V2024011510300001__add_customer_settings.sql`
- 14-digit timestamp after V
- NN is sequential number for same timestamp

### Configuration
```yaml
spring:
  flyway:
    enabled: true
    out-of-order: true              # Allow non-sequential for parallel dev
    validate-migration-naming: true  # Enforce naming convention
    default-schema: public
```

### Database Conventions (PostgreSQL)
- **Table/Column Names**: snake_case
- **Text Columns**: Use TEXT (not VARCHAR)
- **Primary Keys**: UUID with `@GeneratedValue(strategy = AUTO)`
- **Indexes**: Create for foreign keys and frequently queried columns
- **JSONB**: Use for flexible/dynamic data
- **Timestamps**: Use `TIMESTAMP` (not `TIMESTAMP WITH TIME ZONE`)

## Testing Migrations

**Before committing:**

1. **Test on production copy:**
```bash
# Dump production schema
pg_dump -s production_db > schema.sql

# Restore locally
psql test_db < schema.sql

# Run migration
./gradlew flywayMigrate
```

2. **Verify backward compatibility:**
- Old code can still run?
- No constraint violations?
- Default values work?

3. **Check migration history:**
```bash
./gradlew flywayInfo
```

## Red Flags - STOP and Split Migration

If migration contains:
- Adding NOT NULL without default
- Dropping columns still referenced in code
- Changing column types in one step
- Adding foreign keys without validating data first
- Renaming without views

**ALL of these mean: STOP. Split into multiple migrations.**

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "It's just dev environment" | Habits form in dev. Practice safety everywhere. |
| "Quick fix, will improve later" | Later never comes. Fix it right now. |
| "Production deploys are always down" | Zero-downtime should be default, not exception. |
| "This column is never NULL anyway" | Data assumptions change. Default + NOT NULL is safer. |
| "One migration is simpler" | One breaking migration = production outage. |

## Integration with Other Skills

This skill works with:
- `debugging/verification-before-completion` - Verify migration before claiming done
- `collaboration/requesting-code-review` - Have migrations reviewed
- `architecture/preserving-productive-tensions` - Balance speed vs safety

## Quick Reference

| Change | Safe Pattern |
|--------|--------------|
| **Add NOT NULL column** | 1. Add with default → 2. Populate → 3. Add constraint |
| **Drop column** | 1. Remove code → 2. Deploy → 3. Drop column |
| **Rename** | 1. Rename + create view → 2. Update code → 3. Drop view |
| **Change type** | 1. Add new column → 2. Populate → 3. Switch code → 4. Drop old |

## When to Get Help

**Ask Randy before:**
- First migration on a new table
- Uncertain about backward compatibility
- Complex data transformation required
- Migration affects critical production tables
- Need to backfill large amounts of data

Remember: Production outages from bad migrations are expensive. Take time to do it right.
