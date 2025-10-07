# **Complete Production-Ready Guide to Flyway with Spring Boot**

---

## **Table of Contents**
1. [Introduction to Flyway](#1-introduction-to-flyway)
2. [Why Use Flyway? Real-World Impact](#2-why-use-flyway-real-world-impact)
3. [Core Concepts Deep Dive](#3-core-concepts-deep-dive)
4. [Setting Up Flyway in Spring Boot](#4-setting-up-flyway-in-spring-boot)
5. [Writing Migrations: From Simple to Complex](#5-writing-migrations-from-simple-to-complex)
6. [Flyway Lifecycle & Execution Model](#6-flyway-lifecycle--execution-model)
7. [Configuration Guide: Development to Production](#7-configuration-guide-development-to-production)
8. [Advanced Topics & Real-World Patterns](#8-advanced-topics--real-world-patterns)
9. [Team Workflows & Collaboration](#9-team-workflows--collaboration)
10. [Production Deployment Strategies](#10-production-deployment-strategies)
11. [Troubleshooting Guide](#11-troubleshooting-guide)
12. [Performance Optimization](#12-performance-optimization)
13. [Security Best Practices](#13-security-best-practices)
14. [Testing Migrations](#14-testing-migrations)
15. [Flyway vs Liquibase: Detailed Comparison](#15-flyway-vs-liquibase-detailed-comparison)
16. [Migration Patterns & Anti-Patterns](#16-migration-patterns--anti-patterns)
17. [Flyway CLI & Automation](#17-flyway-cli--automation)
18. [FAQs & Common Scenarios](#18-faqs--common-scenarios)
19. [Resources & Tools](#19-resources--tools)

---

## **1. Introduction to Flyway**

### **What is Flyway?**
Flyway is a database migration tool that enables version control for your database. Think of it as "Git for your database schema" - it tracks changes, ensures consistency, and automates deployment across all environments.

### **Core Philosophy**
- **Migrations are immutable**: Once applied, never modify them
- **Forward-only by default**: Focus on moving forward, not backward
- **SQL-first approach**: Write plain SQL (also supports Java-based migrations)
- **Convention over configuration**: Minimal setup, maximum productivity

### **Key Features**
- ✅ Version control for database changes
- ✅ Automatic migration execution
- ✅ Multi-environment consistency
- ✅ Migration validation and verification
- ✅ Rollback support (with proper planning)
- ✅ Support for 20+ database systems
- ✅ Zero external dependencies

---

## **2. Why Use Flyway? Real-World Impact**

### **Problem: The Traditional Approach**

**Scenario**: Your team has 5 developers, 3 environments (dev, staging, prod)

```
Developer A: Runs script_v1.sql locally ✓
Developer B: Forgets to run script_v1.sql ✗
Staging: script_v1.sql applied ✓
Production: script_v1.sql partially applied due to error ✗
Developer C: Creates script_v2.sql depending on script_v1.sql ✗
```

**Result**: Production crashes, emergency hotfix needed, 3 hours of downtime, angry customers.

### **Solution: With Flyway**

```
Developer A: Creates V1__add_users_table.sql, commits to Git
Developer B: Pulls latest code, Flyway auto-applies V1
Staging: Flyway auto-applies V1 on deployment
Production: Flyway auto-applies V1 on deployment
Developer C: Creates V2__add_user_profile.sql, builds on V1 safely
```

**Result**: Zero manual intervention, zero human error, zero downtime.

### **Real Cost Savings**

| Metric | Traditional | With Flyway | Savings |
|--------|------------|-------------|---------|
| Migration time per environment | 30 minutes | 0 minutes (automatic) | 30 min |
| Human errors per month | 2-3 | 0 | Priceless |
| Debugging schema issues | 2 hours | 10 minutes | 1h 50min |
| Rollback preparation | 1 hour | Pre-planned | 1 hour |
| **Monthly time savings** | - | - | **~15-20 hours** |

---

## **3. Core Concepts Deep Dive**

### **3.1 Migration Files**

#### **Versioned Migrations**
```
Format: V<VERSION>__<DESCRIPTION>.sql
Example: V1.0.1__create_users_table.sql
```

**Version numbering strategies:**
```
V1__initial_schema.sql           (Simple sequential)
V1.1__add_email_column.sql       (Semantic versioning)
V20241007_1430__add_index.sql    (Timestamp-based)
V2024.10.07.001__create_orders.sql (Date + counter)
```

**Best Practice**: Choose one strategy and stick with it across your team.

#### **Repeatable Migrations**
```
Format: R__<DESCRIPTION>.sql
Example: R__create_materialized_views.sql
```

Use for:
- Views
- Stored procedures
- Functions
- Triggers (use with caution)

### **3.2 The flyway_schema_history Table**

After running migrations, Flyway creates this tracking table:

```sql
SELECT * FROM flyway_schema_history ORDER BY installed_rank;
```

| installed_rank | version | description | type | script | checksum | installed_by | installed_on | execution_time | success |
|----------------|---------|-------------|------|--------|----------|--------------|--------------|----------------|---------|
| 1 | 1 | create users table | SQL | V1__create_users_table.sql | 123456789 | admin | 2024-10-07 10:30:00 | 45 | true |
| 2 | 2 | add email index | SQL | V2__add_email_index.sql | 987654321 | admin | 2024-10-07 10:30:01 | 12 | true |

**Critical Fields:**
- **checksum**: Detects if migration file was modified (BAD practice)
- **success**: `false` means migration failed - must be fixed before proceeding
- **installed_rank**: Order of execution (critical for debugging)

### **3.3 Migration States**

```
Pending → Running → Success
                  ↘ Failed → Requires Manual Fix
```

**Handling Failed Migrations:**
```sql
-- Check failed migrations
SELECT * FROM flyway_schema_history WHERE success = false;

-- After fixing the SQL file, use repair:
-- flyway repair (removes failed entry, allowing retry)
```

---

## **4. Setting Up Flyway in Spring Boot**

### **4.1 Basic Setup**

#### **Step 1: Add Dependency**

**Maven (pom.xml):**
```xml
<dependencies>
    <!-- Flyway Core -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    
    <!-- Database Driver (example: PostgreSQL) -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Gradle (build.gradle):**
```gradle
dependencies {
    implementation 'org.flywaydb:flyway-core'
    runtimeOnly 'org.postgresql:postgresql'
}
```

#### **Step 2: Project Structure**

```
src/
├── main/
│   ├── java/
│   │   └── com/example/demo/
│   │       └── DemoApplication.java
│   └── resources/
│       ├── application.yml
│       └── db/
│           └── migration/
│               ├── V1__create_users_table.sql
│               ├── V2__add_email_index.sql
│               ├── V3__create_orders_table.sql
│               └── R__refresh_user_stats_view.sql
```

#### **Step 3: Configuration**

**application.yml (Development):**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: dev_user
    password: dev_password
    
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    validate-on-migrate: true
    clean-disabled: true
    out-of-order: false
```

**application-prod.yml (Production):**
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false  # Don't baseline in prod
    validate-on-migrate: true   # Always validate
    clean-disabled: true        # CRITICAL: Never allow clean in prod
    out-of-order: false         # Strict ordering in prod
    placeholder-replacement: true
    placeholders:
      environment: production
```

### **4.2 Multiple Database Setup**

**Configuration for Multiple Datasources:**

```java
@Configuration
public class FlywayConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public Flyway primaryFlyway(@Qualifier("primaryDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/primary")
            .load();
    }
    
    @Bean
    public Flyway secondaryFlyway(@Qualifier("secondaryDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/secondary")
            .load();
    }
    
    @Bean
    public FlywayMigrationInitializer primaryFlywayInitializer(Flyway primaryFlyway) {
        return new FlywayMigrationInitializer(primaryFlyway);
    }
    
    @Bean
    public FlywayMigrationInitializer secondaryFlywayInitializer(Flyway secondaryFlyway) {
        return new FlywayMigrationInitializer(secondaryFlyway);
    }
}
```

---

## **5. Writing Migrations: From Simple to Complex**

### **5.1 Basic Table Creation**

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Add index for common queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Add comment for documentation
COMMENT ON TABLE users IS 'Core user authentication and profile table';
```

### **5.2 Altering Existing Tables**

```sql
-- V2__add_user_profile_fields.sql
ALTER TABLE users 
    ADD COLUMN first_name VARCHAR(100),
    ADD COLUMN last_name VARCHAR(100),
    ADD COLUMN phone VARCHAR(20),
    ADD COLUMN date_of_birth DATE;

-- Add constraint
ALTER TABLE users 
    ADD CONSTRAINT check_age CHECK (
        date_of_birth IS NULL OR 
        date_of_birth < CURRENT_DATE - INTERVAL '13 years'
    );
```

### **5.3 Complex Data Migration**

```sql
-- V3__migrate_user_preferences.sql

-- Step 1: Create new table
CREATE TABLE user_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    preference_key VARCHAR(100) NOT NULL,
    preference_value TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, preference_key)
);

-- Step 2: Migrate existing data from old format
INSERT INTO user_preferences (user_id, preference_key, preference_value)
SELECT 
    id as user_id,
    'theme' as preference_key,
    COALESCE(theme_preference, 'light') as preference_value
FROM users
WHERE theme_preference IS NOT NULL;

-- Step 3: Drop old column (only after verifying migration)
-- Consider doing this in a separate migration after verification
-- ALTER TABLE users DROP COLUMN theme_preference;
```

### **5.4 Handling Large Data Migrations**

```sql
-- V4__backfill_user_stats.sql

-- For large tables, use batching to avoid locks
DO $$
DECLARE
    batch_size INTEGER := 1000;
    processed INTEGER := 0;
    total_rows INTEGER;
BEGIN
    SELECT COUNT(*) INTO total_rows FROM users WHERE stats_calculated = false;
    
    WHILE processed < total_rows LOOP
        -- Update in batches
        WITH batch AS (
            SELECT id FROM users 
            WHERE stats_calculated = false 
            LIMIT batch_size
        )
        UPDATE users 
        SET 
            total_orders = (SELECT COUNT(*) FROM orders WHERE user_id = users.id),
            total_spent = (SELECT COALESCE(SUM(amount), 0) FROM orders WHERE user_id = users.id),
            stats_calculated = true
        WHERE id IN (SELECT id FROM batch);
        
        processed := processed + batch_size;
        
        -- Commit between batches to release locks
        COMMIT;
        
        -- Optional: Add delay to reduce load
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

### **5.5 Safe Column Drops (Zero Downtime)**

```sql
-- V5__prepare_to_drop_old_column.sql
-- Step 1: Stop writing to the column (code deployment)
-- Then run this migration

-- Make column nullable first
ALTER TABLE users ALTER COLUMN old_address DROP NOT NULL;

-- V6__actually_drop_old_column.sql (run after verifying Step 1)
-- Step 2: Drop the column
ALTER TABLE users DROP COLUMN old_address;
```

### **5.6 Creating Indexes Concurrently**

```sql
-- V7__add_order_indexes.sql

-- Standard index creation locks the table
-- CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Better: Use CONCURRENTLY (PostgreSQL)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user_id ON orders(user_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_status ON orders(status);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_created_at ON orders(created_at DESC);

-- Composite index for common query patterns
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user_status 
    ON orders(user_id, status, created_at DESC);
```

### **5.7 Repeatable Migrations (Views & Functions)**

```sql
-- R__user_statistics_view.sql
-- This will be rerun whenever the file changes

DROP VIEW IF EXISTS user_statistics;

CREATE VIEW user_statistics AS
SELECT 
    u.id,
    u.username,
    u.email,
    COUNT(DISTINCT o.id) as total_orders,
    COALESCE(SUM(o.amount), 0) as total_spent,
    MAX(o.created_at) as last_order_date,
    CASE 
        WHEN COUNT(o.id) > 10 THEN 'VIP'
        WHEN COUNT(o.id) > 5 THEN 'Regular'
        ELSE 'New'
    END as customer_tier
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username, u.email;

-- Add comment
COMMENT ON VIEW user_statistics IS 'Aggregated user statistics for reporting';
```

---

## **6. Flyway Lifecycle & Execution Model**

### **6.1 Complete Lifecycle**

```
Application Startup
    ↓
Flyway Initialization
    ↓
Load Configuration (application.yml)
    ↓
Connect to Database
    ↓
Check/Create flyway_schema_history table
    ↓
Scan Migration Files (db/migration/)
    ↓
Calculate Checksums
    ↓
Compare with flyway_schema_history
    ↓
Detect Pending Migrations
    ↓
Validate Existing Migrations
    ↓
┌─────────────────────────────┐
│  Execute Pending Migrations │
│  (in version order)         │
└─────────────────────────────┘
    ↓
    ├─ Begin Transaction
    ├─ Execute SQL
    ├─ Record in flyway_schema_history
    └─ Commit Transaction
    ↓
Application Starts Normally
```

### **6.2 Transaction Handling**

**Single Transaction (Default):**
```sql
-- V1__multiple_operations.sql
-- All succeed or all fail together

BEGIN; -- Automatic

CREATE TABLE users (...);
CREATE TABLE orders (...);
INSERT INTO config VALUES ('version', '1.0');

COMMIT; -- Automatic
```

**Multiple Transactions (Manual Control):**
```sql
-- V2__manual_transactions.sql

-- Transaction 1
CREATE TABLE products (...);

-- Transaction 2 (explicit)
BEGIN;
INSERT INTO products (name, price) VALUES ('Product1', 100);
COMMIT;

-- Transaction 3 (explicit)
BEGIN;
CREATE INDEX idx_products_name ON products(name);
COMMIT;
```

### **6.3 Rollback Behavior**

Flyway **does NOT automatically rollback** - you must plan for it:

```sql
-- V3__add_not_null_constraint.sql
-- WRONG: This might fail if there are NULL values

ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

**Better approach:**

```sql
-- V3__prepare_email_not_null.sql
-- Step 1: Fill NULL values

UPDATE users SET email = 'placeholder@example.com' WHERE email IS NULL;

-- V4__enforce_email_not_null.sql
-- Step 2: Add constraint (run after verification)

ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

---

## **7. Configuration Guide: Development to Production**

### **7.1 Environment-Specific Configs**

**application-dev.yml:**
```yaml
spring:
  flyway:
    enabled: true
    clean-disabled: false  # Allow clean in dev
    out-of-order: true     # Allow flexible development
    validate-on-migrate: false  # Skip validation for speed
```

**application-staging.yml:**
```yaml
spring:
  flyway:
    enabled: true
    clean-disabled: true   # No clean in staging
    out-of-order: false    # Enforce order
    validate-on-migrate: true  # Validate everything
```

**application-prod.yml:**
```yaml
spring:
  flyway:
    enabled: true
    clean-disabled: true   # NEVER clean production
    out-of-order: false    # Strict order
    validate-on-migrate: true
    validate-migration-naming: true
    ignore-missing-migrations: false
    ignore-future-migrations: false
```

### **7.2 All Configuration Options**

```yaml
spring:
  flyway:
    # Core Settings
    enabled: true
    locations: classpath:db/migration
    encoding: UTF-8
    
    # Schema Management
    schemas: public
    table: flyway_schema_history
    tablespace: null
    
    # Baseline (for existing databases)
    baseline-on-migrate: false
    baseline-version: 1
    baseline-description: Initial baseline
    
    # Validation
    validate-on-migrate: true
    validate-migration-naming: true
    
    # Safety Settings
    clean-disabled: true  # CRITICAL FOR PRODUCTION
    clean-on-validation-error: false
    
    # Migration Control
    out-of-order: false
    ignore-missing-migrations: false
    ignore-ignored-migrations: false
    ignore-pending-migrations: false
    ignore-future-migrations: false
    
    # Placeholders
    placeholder-replacement: true
    placeholder-prefix: "${"
    placeholder-suffix: "}"
    placeholders:
      app.name: MyApp
      app.version: 1.0.0
    
    # Performance
    batch: false
    mixed: false
    group: false
    
    # Callbacks
    callbacks:
      - com.example.MyCustomCallback
```

### **7.3 Using Placeholders**

**application.yml:**
```yaml
spring:
  flyway:
    placeholder-replacement: true
    placeholders:
      table_prefix: prod_
      admin_email: admin@example.com
```

**Migration file:**
```sql
-- V1__create_tables_with_placeholders.sql

CREATE TABLE ${table_prefix}users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL
);

INSERT INTO config (key, value) VALUES ('admin_email', '${admin_email}');
```

---

## **8. Advanced Topics & Real-World Patterns**

### **8.1 Baseline Migrations (Legacy Databases)**

**Scenario**: You have an existing production database without Flyway.

**Step 1: Create baseline migration**
```sql
-- V1__baseline_existing_schema.sql
-- This represents your current production state

-- Document what already exists (no actual changes)
-- Just for reference:

-- Tables: users, orders, products (already exist)
-- Indexes: idx_users_email, idx_orders_user_id (already exist)
-- Constraints: fk_orders_user_id (already exists)

-- This migration will be marked as "baselined" and skipped
```

**Step 2: Configure baseline**
```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1
    baseline-description: "Existing production schema"
```

**Step 3: New migrations start from V2**
```sql
-- V2__add_user_preferences.sql
-- This and future migrations will be applied
```

### **8.2 Callbacks (Lifecycle Hooks)**

```java
@Component
public class CustomFlywayCallback implements Callback {
    
    private static final Logger log = LoggerFactory.getLogger(CustomFlywayCallback.class);
    
    @Override
    public boolean supports(Event event, Context context) {
        return true;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }
    
    @Override
    public void handle(Event event, Context context) {
        switch (event) {
            case BEFORE_MIGRATE:
                log.info("🚀 Starting database migration...");
                // Take backup, send notification, etc.
                break;
                
            case AFTER_MIGRATE:
                log.info("✅ Database migration completed successfully");
                // Clear caches, restart services, etc.
                break;
                
            case AFTER_MIGRATE_ERROR:
                log.error("❌ Database migration failed!");
                // Send alert, rollback deployment, etc.
                break;
        }
    }
}
```

### **8.3 Java-Based Migrations**

For complex logic that's easier in Java:

```java
@Component
public class V5__ComplexDataMigration implements JavaMigration {
    
    @Override
    public Integer getVersion() {
        return 5;
    }
    
    @Override
    public String getDescription() {
        return "Complex data migration with business logic";
    }
    
    @Override
    public void migrate(Context context) throws Exception {
        try (Statement stmt = context.getConnection().createStatement()) {
            
            // Complex business logic
            ResultSet rs = stmt.executeQuery("SELECT * FROM old_table");
            
            while (rs.next()) {
                // Transform data with Java
                String transformed = complexBusinessLogic(rs.getString("data"));
                
                // Insert into new table
                PreparedStatement insert = context.getConnection()
                    .prepareStatement("INSERT INTO new_table (transformed_data) VALUES (?)");
                insert.setString(1, transformed);
                insert.executeUpdate();
            }
        }
    }
    
    private String complexBusinessLogic(String input) {
        // Business logic that's hard to express in SQL
        return input.toUpperCase().replaceAll("[^A-Z0-9]", "");
    }
}
```

### **8.4 Conditional Migrations (Per Environment)**

```sql
-- V6__create_audit_table.sql

-- Check if running in production
DO $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM pg_settings 
        WHERE name = 'flyway.placeholders.environment' 
        AND setting = 'production'
    ) THEN
        -- Production-specific logic
        CREATE TABLE audit_log (
            id BIGSERIAL PRIMARY KEY,
            user_id BIGINT NOT NULL,
            action VARCHAR(100) NOT NULL,
            details JSONB,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
        );
        
        -- Create partition tables
        CREATE TABLE audit_log_2024_10 PARTITION OF audit_log
            FOR VALUES FROM ('2024-10-01') TO ('2024-11-01');
    ELSE
        -- Development/staging: simple version
        CREATE TABLE audit_log (
            id BIGSERIAL PRIMARY KEY,
            user_id BIGINT NOT NULL,
            action VARCHAR(100) NOT NULL,
            created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
        );
    END IF;
END $$;
```

### **8.5 Blue-Green Deployment Pattern**

```sql
-- V7__add_new_column_blue_green.sql

-- Phase 1: Add nullable column
ALTER TABLE users ADD COLUMN new_address TEXT NULL;

-- Phase 2: Dual-write phase (application writes to both old and new)
-- (Handled in application code for several hours/days)

-- V8__backfill_new_column.sql (run after dual-write is stable)
UPDATE users SET new_address = old_address WHERE new_address IS NULL;

-- V9__make_new_column_required.sql (run after backfill verification)
ALTER TABLE users ALTER COLUMN new_address SET NOT NULL;

-- V10__drop_old_column.sql (final phase after new column is stable)
ALTER TABLE users DROP COLUMN old_address;
```

---

## **9. Team Workflows & Collaboration**

### **9.1 Git Workflow Integration**

**Branching Strategy:**
```
main (production-ready migrations)
  ├── develop (integration branch)
  │   ├── feature/add-user-preferences (V10)
  │   ├── feature/add-order-tracking (V11)
  │   └── feature/add-notifications (V12)
```

**Merge conflicts prevention:**
```bash
# Before creating a new migration:
git pull origin develop

# Check latest migration version
ls -la src/main/resources/db/migration/ | tail -5

# Create next version in sequence
# V13__your_feature.sql
```

### **9.2 Code Review Checklist**

**Migration Code Review Template:**

```markdown
## Migration Review Checklist

- [ ] Version number follows team convention?
- [ ] Migration is idempotent (can safely run multiple times)?
- [ ] Migration is backward compatible with current production code?
- [ ] Large data migrations use batching?
- [ ] Indexes created with CONCURRENTLY (PostgreSQL)?
- [ ] No DROP operations without safety period?
- [ ] Tested on production-sized dataset?
- [ ] Rollback plan documented?
- [ ] Execution time estimated and acceptable?
- [ ] Team notified of breaking changes?
```

### **9.3 Pull Request Template**

```markdown
## Database Migration PR

### Migration Details
- **Version**: V15__add_payment_methods.sql
- **Description**: Add payment methods table and link to users
- **Estimated Execution Time**: < 1 second (new table, no data)

### Breaking Changes
- [ ] Yes
- [x] No

### Rollback Plan
Create V16 to drop the table if needed (rare, only if blocking deployment)

### Testing
- [x] Tested on local database
- [x] Tested on staging database
- [x] Tested with production data volume (simulation)
- [x] Verified application works with and without migration

### Dependencies
- Requires application version 2.3.0+ for payment processing feature

### Reviewer Notes
This is part of the payment gateway integration epic. Table will be empty initially, populated by new feature code.
```

---

## **10. Production Deployment Strategies**

### **10.1 Pre-Deployment Checklist**

```markdown
## Production Migration Checklist

### 1 Week Before
- [ ] Migration reviewed and approved
- [ ] Tested on production-copy database
- [ ] Performance impact assessed
- [ ] Rollback plan created
- [ ] Stakeholders notified

### 1 Day Before
- [ ] Database backup verified
- [ ] Monitoring alerts configured
- [ ] Rollback scripts prepared
- [ ] Team on standby

### Deployment Day
- [ ] Verify low-traffic window
- [ ] Execute migration
- [ ] Verify flyway_schema_history
- [ ] Run smoke tests
- [ ] Monitor for 1 hour

### Post-Deployment
- [ ] Verify data integrity
- [ ] Check application logs
- [ ] Monitor database performance
- [ ] Document any issues
```

### **10.2 Zero-Downtime Migration Pattern**

**Example: Renaming a column**

```sql
-- ❌ WRONG: This causes downtime
-- ALTER TABLE users RENAME COLUMN old_name TO new_name;

-- ✅ RIGHT: Multi-phase approach

-- Phase 1 (V10): Add new column
ALTER TABLE users ADD COLUMN new_name VARCHAR(255);

-- Phase 2 (Application deployment): Dual-write mode
-- App writes to both old_name and new_name
-- App reads from old_name (fallback to new_name)

-- Phase 3 (V11): Backfill data
UPDATE users SET new_name = old_name WHERE new_name IS NULL;

-- Phase 4 (Application deployment): Switch reads
-- App reads from new_name (fallback to old_name)

-- Phase 5 (V12): Drop old column
-- After verifying everything works for 24-48 hours
ALTER TABLE users DROP COLUMN old_name;
```

### **10.3 Deployment Strategies**

**Strategy 1: Automatic Migration on Startup (Default)**
```yaml
spring:
  flyway:
    enabled: true
    # Migrations run automatically when app starts
```
**Pros**: Simple, zero manual steps
**Cons**: App won't start if migration fails

**Strategy 2: Manual Migration Before Deployment**
```bash
# Run migrations separately using Flyway CLI
flyway -configFiles=flyway-prod.conf migrate

# Then deploy application with migrations disabled
# application-prod.yml:
spring.flyway.enabled=false
```
**Pros**: Better control, app deployment independent from DB changes
**Cons**: Requires separate step

**Strategy 3: Canary Deployment**
```yaml
# Deploy to 1 instance first (with migrations enabled)
spring.flyway.enabled=true

# Other instances with migrations disabled
spring.flyway.enabled=false
```
**Pros**: Gradual rollout, can catch issues early
**Cons**: More complex deployment process

---

## **11. Troubleshooting Guide**

### **11.1 Common Errors & Solutions**

#### **Error: "Checksum mismatch"**

```
ERROR: Migration checksum mismatch for migration version 1
Applied to database : 123456789
Resolved locally    : 987654321
```

**Cause**: Someone modified an already-applied migration file.

**Solution**:
```bash
# Option 1: Revert the file to original (PREFERRED)
git checkout HEAD~1 -- src/main/resources/db/migration/V1__*.sql

# Option 2: Repair the checksum (USE WITH CAUTION)
flyway repair

# Option 3: Accept the new checksum (DANGEROUS)
spring.flyway.validate-on-migrate=false  # Temporary
```

**Prevention**:
- Never modify applied migrations
- Use code reviews
- Lock migration files after applying to production

#### **Error: "Migration failed"**

```
ERROR: Migration V5__add_column.sql failed
SQL State  : 42P01
Error Code : 0
Message    : ERROR: relation "users" does not exist
```

**Solution**:
```sql
-- Check flyway_schema_history
SELECT * FROM flyway_schema_history WHERE success = false;

-- Fix the migration file
-- Then repair and retry
flyway repair
flyway migrate
```

#### **Error: "Out of order migration detected"**

```
ERROR: Detected resolved migration not applied to database: 2.5
Conflicting with migration version: 3.0
```

**Cause**: Someone created V2.5 after V3.0 was already applied.

**Solution**:
```yaml
# Development only - allow out-of-order
spring.flyway.out-of-order=true

# Better: Rename to V4__description.sql
# Production: Never allow out-of-order
```

#### **Error: "Failed to obtain JDBC connection"**

```
ERROR: Unable to obtain connection from database
```

**Solution**:
```yaml
# Check datasource configuration
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}  # From environment variable
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 5
      connection-timeout: 30000
      
# Verify database is running
# Check network connectivity
# Verify credentials
```

### **11.2 Debugging Tips**

**Enable verbose logging:**
```yaml
logging:
  level:
    org.flywaydb: DEBUG
    org.springframework.jdbc: DEBUG
```

**Check migration status:**
```sql
-- See all migrations
SELECT 
    installed_rank,
    version,
    description,
    type,
    script,
    success,
    execution_time,
    installed_on
FROM flyway_schema_history
ORDER BY installed_rank;

-- Find failed migrations
SELECT * FROM flyway_schema_history WHERE success = false;

-- Find pending migrations (check application logs)
```

**Validate migrations without applying:**
```bash
flyway validate
flyway info  # Shows pending migrations
```

### **11.3 Recovery Procedures**

**Scenario: Production migration failed mid-execution**

```bash
# Step 1: Assess the damage
SELECT * FROM flyway_schema_history WHERE success = false;

# Step 2: Check what was actually applied
# Review database schema manually

# Step 3: Fix the migration file
# Make it idempotent or fix the error

# Step 4: Repair Flyway state
flyway repair

# Step 5: Retry migration
flyway migrate

# Step 6: Verify success
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;
```

**Scenario: Need to rollback a migration**

```sql
-- Flyway doesn't auto-rollback, so create a new migration

-- Original: V10__add_column.sql
ALTER TABLE users ADD COLUMN middle_name VARCHAR(100);

-- Rollback: V11__remove_middle_name.sql
ALTER TABLE users DROP COLUMN middle_name;
```

---

## **12. Performance Optimization**

### **12.1 Large Table Migrations**

**Problem**: Adding a column to a 100M row table locks it for hours.

**Solution: Use multiple steps**

```sql
-- V10__add_column_step1.sql
-- Add column as nullable (instant operation)
ALTER TABLE large_table ADD COLUMN new_column VARCHAR(255) NULL;

-- V11__add_column_step2.sql
-- Backfill in batches to avoid locks
DO $
DECLARE
    batch_size INTEGER := 10000;
    offset_val INTEGER := 0;
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE large_table
        SET new_column = compute_value(id)
        WHERE id IN (
            SELECT id FROM large_table
            WHERE new_column IS NULL
            ORDER BY id
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        
        -- Commit batch
        COMMIT;
        
        -- Small delay to allow other queries
        PERFORM pg_sleep(0.1);
    END LOOP;
END $;

-- V12__add_column_step3.sql
-- Make NOT NULL after backfill completes
ALTER TABLE large_table ALTER COLUMN new_column SET NOT NULL;
```

### **12.2 Index Creation Best Practices**

```sql
-- ❌ BAD: Locks table during creation
CREATE INDEX idx_users_email ON users(email);

-- ✅ GOOD: Non-blocking index creation (PostgreSQL)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email);

-- For very large tables, consider:
-- 1. Create index during low-traffic window
-- 2. Monitor index creation progress
SELECT 
    now()::TIME,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE query LIKE '%CREATE INDEX%';
```

### **12.3 Migration Execution Time**

**Estimate before running:**

```sql
-- Test on copy of production database
EXPLAIN ANALYZE
ALTER TABLE users ADD COLUMN new_field VARCHAR(255);

-- Expected output:
-- Execution Time: 0.123 ms (for schema change)
-- Note: Actual time depends on table size and row rewrite
```

**Set timeouts:**

```yaml
spring:
  flyway:
    lock-retry-count: 50
    
  datasource:
    hikari:
      connection-timeout: 60000
      validation-timeout: 5000
```

---

## **13. Security Best Practices**

### **13.1 Credential Management**

**❌ NEVER do this:**
```yaml
spring:
  datasource:
    username: admin
    password: secretpassword123  # Hardcoded!
```

**✅ Use environment variables:**
```yaml
spring:
  datasource:
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

**✅ Or use secrets management:**
```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource(
            @Value("${aws.secretsmanager.secret-name}") String secretName) {
        
        // Fetch credentials from AWS Secrets Manager
        String credentials = awsSecretsManager.getSecret(secretName);
        
        return DataSourceBuilder.create()
            .url(extractUrl(credentials))
            .username(extractUsername(credentials))
            .password(extractPassword(credentials))
            .build();
    }
}
```

### **13.2 Migration Security**

```sql
-- ❌ BAD: Exposing sensitive data in migration
INSERT INTO users (email, password) VALUES ('admin@example.com', 'admin123');

-- ✅ GOOD: Use placeholders or secure injection
INSERT INTO users (email, password_hash) 
VALUES ('${admin.email}', crypt('${admin.password}', gen_salt('bf')));
```

### **13.3 Database User Permissions**

**Principle of Least Privilege:**

```sql
-- Create dedicated Flyway user
CREATE USER flyway_user WITH PASSWORD 'secure_random_password';

-- Grant only necessary permissions
GRANT CONNECT ON DATABASE mydb TO flyway_user;
GRANT USAGE ON SCHEMA public TO flyway_user;
GRANT CREATE ON SCHEMA public TO flyway_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO flyway_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO flyway_user;

-- Application user (read/write only, no schema changes)
CREATE USER app_user WITH PASSWORD 'different_secure_password';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

**Configuration:**
```yaml
# Use flyway_user only for migrations
spring:
  flyway:
    user: flyway_user
    password: ${FLYWAY_PASSWORD}
    
  # Use app_user for application runtime
  datasource:
    username: app_user
    password: ${APP_PASSWORD}
```

---

## **14. Testing Migrations**

### **14.1 Unit Testing Migrations**

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.flyway.clean-disabled=false"
})
class FlywayMigrationTest {
    
    @Autowired
    private Flyway flyway;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void shouldApplyAllMigrations() {
        // Clean and migrate
        flyway.clean();
        MigrateResult result = flyway.migrate();
        
        // Verify all migrations succeeded
        assertThat(result.success).isTrue();
        assertThat(result.migrationsExecuted).isGreaterThan(0);
    }
    
    @Test
    void shouldHaveExpectedSchema() {
        // Verify tables exist
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.tables WHERE table_name = 'users'",
            Integer.class
        );
        assertThat(count).isEqualTo(1);
        
        // Verify columns exist
        List<String> columns = jdbcTemplate.queryForList(
            "SELECT column_name FROM information_schema.columns WHERE table_name = 'users'",
            String.class
        );
        assertThat(columns).contains("id", "username", "email", "created_at");
    }
    
    @Test
    void shouldValidateMigrationChecksums() {
        // This will fail if any migration file was modified
        assertThatCode(() -> flyway.validate())
            .doesNotThrowAnyException();
    }
}
```

### **14.2 Integration Testing**

```java
@SpringBootTest
@Sql(scripts = "/test-data.sql", executionPhase = BEFORE_TEST_METHOD)
class DatabaseIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldWorkWithMigratedSchema() {
        // Test that application code works with current schema
        User user = new User("testuser", "test@example.com");
        User saved = userRepository.save(user);
        
        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getCreatedAt()).isNotNull();
    }
}
```

### **14.3 Backward Compatibility Testing**

```java
@SpringBootTest
class BackwardCompatibilityTest {
    
    @Autowired
    private Flyway flyway;
    
    @Test
    void oldApplicationShouldWorkWithNewSchema() {
        // Scenario: New migration adds optional column
        // Old application code should still work
        
        // Run new migration
        flyway.migrate();
        
        // Test with old-style code (without new column)
        jdbcTemplate.update(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            "olduser", "old@example.com"
        );
        
        // Should succeed (new column has default/null)
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM users WHERE username = 'olduser'",
            Integer.class
        );
        assertThat(count).isEqualTo(1);
    }
}
```

### **14.4 Performance Testing**

```java
@SpringBootTest
class MigrationPerformanceTest {
    
    @Test
    void migrationShouldCompleteInReasonableTime() {
        long startTime = System.currentTimeMillis();
        
        flyway.clean();
        flyway.migrate();
        
        long duration = System.currentTimeMillis() - startTime;
        
        // All migrations should complete in < 5 seconds on empty database
        assertThat(duration).isLessThan(5000);
    }
    
    @Test
    void migrationShouldHandleLargeDataset() {
        // Insert test data
        for (int i = 0; i < 100000; i++) {
            jdbcTemplate.update(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                "user" + i, "user" + i + "@example.com"
            );
        }
        
        // Run migration that affects all rows
        long startTime = System.currentTimeMillis();
        flyway.migrate();
        long duration = System.currentTimeMillis() - startTime;
        
        // Should complete in reasonable time
        assertThat(duration).isLessThan(30000); // 30 seconds
    }
}
```

---

## **15. Flyway vs Liquibase: Detailed Comparison**

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| **Learning Curve** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moderate |
| **Configuration** | Convention-based, minimal | XML/YAML/JSON-based, verbose |
| **SQL Support** | Native SQL files | ChangeSet abstractions |
| **Database Support** | 20+ databases | 40+ databases |
| **Rollback** | Manual (create new migration) | Automatic with pro version |
| **Versioning** | Version numbers | ChangeSets with IDs |
| **Team Size** | Better for small-medium teams | Better for large enterprises |
| **Flexibility** | Less flexible, more opinionated | Highly flexible |
| **Spring Boot Integration** | Native, seamless | Requires more configuration |
| **Community** | Large, active | Very large, active |
| **Commercial Features** | Flyway Teams (paid) | Liquibase Pro (paid) |
| **Best For** | SQL-first approach, Spring Boot | Complex enterprise needs |

### **When to Choose Flyway:**
- ✅ You prefer writing SQL directly
- ✅ Using Spring Boot
- ✅ Want minimal configuration
- ✅ Small to medium projects
- ✅ Simple rollback needs

### **When to Choose Liquibase:**
- ✅ Need automatic rollbacks
- ✅ Database-agnostic changesets
- ✅ Complex enterprise requirements
- ✅ Need preconditions and contexts
- ✅ Large, complex projects

---

## **16. Migration Patterns & Anti-Patterns**

### **16.1 Good Patterns ✅**

#### **Pattern: Idempotent Migrations**
```sql
-- Always safe to run multiple times
CREATE TABLE IF NOT EXISTS users (...);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);

INSERT INTO config (key, value) 
VALUES ('version', '1.0')
ON CONFLICT (key) DO NOTHING;
```

#### **Pattern: Explicit Transaction Control**
```sql
-- For operations that need separate transactions
BEGIN;
CREATE TABLE orders (...);
COMMIT;

BEGIN;
CREATE INDEX idx_orders_user ON orders(user_id);
COMMIT;
```

#### **Pattern: Safe Column Addition**
```sql
-- Phase 1: Add nullable column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Phase 2: Populate data
UPDATE users SET phone = '000-000-0000' WHERE phone IS NULL;

-- Phase 3: Make NOT NULL (separate migration after verification)
-- ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

### **16.2 Anti-Patterns ❌**

#### **Anti-Pattern: Modifying Applied Migrations**
```sql
-- ❌ NEVER modify V1__create_users.sql after it's applied
-- ✅ Create V2__modify_users.sql instead
```

#### **Anti-Pattern: Coupling Migrations to Application Code**
```sql
-- ❌ BAD: Migration depends on application logic
-- V5__complex_migration.sql
-- SELECT * FROM users WHERE status = 'PREMIUM';
-- Problem: What if 'PREMIUM' constant changes in code?

-- ✅ GOOD: Self-contained migrations
SELECT * FROM users WHERE status = 'P';  -- Use DB values
```

#### **Anti-Pattern: Large Transactions**
```sql
-- ❌ BAD: Single transaction for millions of rows
UPDATE large_table SET computed_value = expensive_function(id);

-- ✅ GOOD: Batch processing
DO $
BEGIN
    LOOP
        UPDATE large_table 
        SET computed_value = expensive_function(id)
        WHERE id IN (
            SELECT id FROM large_table 
            WHERE computed_value IS NULL 
            LIMIT 1000
        );
        EXIT WHEN NOT FOUND;
        COMMIT;
        PERFORM pg_sleep(0.01);
    END LOOP;
END $;
```

#### **Anti-Pattern: Dropping Data Without Warning**
```sql
-- ❌ EXTREMELY DANGEROUS
DROP TABLE old_user_preferences;

-- ✅ SAFE: Multi-phase approach
-- Phase 1: Stop using table (code change)
-- Phase 2: Rename to mark for deletion (V10)
ALTER TABLE old_user_preferences RENAME TO deprecated_old_user_preferences;

-- Phase 3: Actually drop after verification period (V11, weeks later)
DROP TABLE IF EXISTS deprecated_old_user_preferences;
```

---

## **17. Flyway CLI & Automation**

### **17.1 Flyway Command Line**

**Installation:**
```bash
# macOS
brew install flyway

# Linux
wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz

# Windows
# Download from https://flywaydb.org/download
```

**Configuration file (flyway.conf):**
```properties
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=dbuser
flyway.password=dbpassword
flyway.locations=filesystem:./sql
flyway.baseline-on-migrate=true
flyway.schemas=public
```

**Common Commands:**
```bash
# Show migration status
flyway info

# Apply pending migrations
flyway migrate

# Validate checksums
flyway validate

# Baseline existing database
flyway baseline

# Clean database (DANGEROUS!)
flyway clean

# Repair checksums
flyway repair
```

### **17.2 CI/CD Integration**

**GitHub Actions Example:**
```yaml
name: Database Migrations

on:
  push:
    branches: [ main ]
    paths:
      - 'src/main/resources/db/migration/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Java
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        
    - name: Install Flyway
      run: |
        wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz
        sudo ln -s `pwd`/flyway-9.22.0/flyway /usr/local/bin
        
    - name: Run Migrations on Staging
      env:
        FLYWAY_URL: ${{ secrets.STAGING_DB_URL }}
        FLYWAY_USER: ${{ secrets.STAGING_DB_USER }}
        FLYWAY_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
      run: |
        flyway info
        flyway migrate
        flyway validate
        
    - name: Notify on Success
      if: success()
      run: echo "Migrations applied successfully!"
      
    - name: Notify on Failure
      if: failure()
      run: echo "Migration failed! Rolling back deployment."
```

**Jenkins Pipeline Example:**
```groovy
pipeline {
    agent any
    
    environment {
        DB_URL = credentials('database-url')
        DB_USER = credentials('database-user')
        DB_PASSWORD = credentials('database-password')
    }
    
    stages {
        stage('Validate Migrations') {
            steps {
                sh 'flyway validate'
            }
        }
        
        stage('Apply Migrations') {
            when {
                branch 'main'
            }
            steps {
                sh 'flyway migrate'
            }
        }
        
        stage('Verify') {
            steps {
                sh '''
                    flyway info
                    # Run smoke tests
                    ./gradlew test --tests MigrationSmokeTest
                '''
            }
        }
    }
    
    post {
        failure {
            mail to: 'team@example.com',
                 subject: "Migration Failed: ${env.JOB_NAME}",
                 body: "Check ${env.BUILD_URL} for details"
        }
    }
}
```

### **17.3 Docker Integration**

**Dockerfile:**
```dockerfile
FROM openjdk:17-jdk-slim

# Install Flyway
RUN apt-get update && apt-get install -y wget && \
    wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz && \
    ln -s /flyway-9.22.0/flyway /usr/local/bin/flyway

# Copy migrations
COPY src/main/resources/db/migration /flyway/sql

# Copy Flyway config
COPY flyway.conf /flyway/conf/

WORKDIR /flyway

ENTRYPOINT ["flyway"]
CMD ["migrate"]
```

**Docker Compose:**
```yaml
version: '3.8'

services:
  database:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: dbuser
      POSTGRES_PASSWORD: dbpassword
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
      
  flyway:
    build: .
    depends_on:
      - database
    environment:
      FLYWAY_URL: jdbc:postgresql://database:5432/mydb
      FLYWAY_USER: dbuser
      FLYWAY_PASSWORD: dbpassword
    command: migrate
    
volumes:
  db_data:
```

---

## **18. FAQs & Common Scenarios**

### **Q1: Can I use Flyway with an existing database?**
**A:** Yes! Use baseline:
```yaml
spring.flyway.baseline-on-migrate=true
spring.flyway.baseline-version=1
```
Your first migration (V1) represents the existing state, and future migrations (V2+) add new changes.

### **Q2: How do I handle multiple developers creating migrations simultaneously?**
**A:** Use a version numbering system that reduces conflicts:
```
# Timestamp-based (recommended for teams)
V20241007143000__alice_feature.sql
V20241007143500__bob_feature.sql

# Or use feature branches and merge carefully
```

### **Q3: Can I run migrations outside application startup?**
**A:** Yes!
```yaml
# Disable automatic migration
spring.flyway.enabled=false
```
Then run manually:
```bash
flyway migrate
# Or use API:
flyway.migrate();
```

### **Q4: How do I test migrations in production without applying them?**
**A:** Use dry-run (Flyway Teams feature) or test on production copy:
```bash
# Create production database copy
pg_dump production_db | psql test_db

# Test migrations on copy
flyway -configFiles=flyway-test.conf migrate
```

### **Q5: What if I need to rollback a migration?**
**A:** Flyway doesn't auto-rollback. Create a new "undo" migration:
```sql
-- V10__add_column.sql
ALTER TABLE users ADD COLUMN middle_name VARCHAR(100);

-- If you need to undo, create:
-- V11__remove_middle_name.sql
ALTER TABLE users DROP COLUMN middle_name;
```

### **Q6: Can I use Flyway with MongoDB/NoSQL?**
**A:** Flyway is designed for relational databases. For NoSQL, consider:
- **MongoDB**: mongock, liquibase
- **Cassandra**: Cassandra Migrate
- **Elasticsearch**: Custom scripts

### **Q7: How do I handle encrypted database passwords?**
**A:** Use environment variables or secrets management:
```yaml
spring:
  datasource:
    password: ${DB_PASSWORD:defaultfordev}
```
Or use Spring Cloud Config, Vault, AWS Secrets Manager, etc.

### **Q8: Can migrations run in parallel?**
**A:** No, Flyway acquires a lock to ensure sequential execution. This prevents corruption.

### **Q9: What's the maximum size for a migration file?**
**A:** No hard limit, but keep migrations small and focused. Split large migrations into multiple steps.

### **Q10: How do I version control database stored procedures?**
**A:** Use repeatable migrations:
```sql
-- R__user_statistics_procedure.sql
CREATE OR REPLACE FUNCTION calculate_user_stats()
RETURNS TRIGGER AS $
BEGIN
    -- Procedure logic
    RETURN NEW;
END;
$ LANGUAGE plpgsql;
```

---

## **19. Resources & Tools**

### **Official Resources**
- 📚 [Flyway Documentation](https://flywaydb.org/documentation)
- 💻 [Flyway GitHub](https://github.com/flyway/flyway)
- 🎓 [Flyway Tutorial](https://flywaydb.org/documentation/getstarted/firststeps)
- 📖 [Spring Boot + Flyway Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.data-initialization.migration-tool.flyway)

### **Tools & Plugins**
- **IntelliJ IDEA Plugin**: Flyway support with validation
- **VS Code Extension**: Flyway migration syntax highlighting
- **DBeaver**: Database tool with Flyway integration
- **Flyway Desktop**: GUI for Flyway (commercial)

### **Related Technologies**
- **Liquibase**: Alternative migration tool
- **Test containers**: Test migrations with real databases
- **DbUnit**: Database testing framework
- **JPA**: Complement Flyway with entity mapping

### **Community**
- Stack Overflow: [flyway tag](https://stackoverflow.com/questions/tagged/flyway)
- Reddit: r/java, r/SpringBoot
- Flyway Discussion Forum

---

## **Appendix A: Complete Example Project**

**Project Structure:**
```
spring-boot-flyway-demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/demo/
│   │   │       ├── DemoApplication.java
│   │   │       ├── entity/
│   │   │       │   └── User.java
│   │   │       ├── repository/
│   │   │       │   └── UserRepository.java
│   │   │       └── config/
│   │   │           └── FlywayCallback.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── db/
│   │           └── migration/
│   │               ├── V1__create_users_table.sql
│   │               ├── V2__add_user_preferences.sql
│   │               ├── V3__create_orders_table.sql
│   │               └── R__user_statistics_view.sql
│   └── test/
│       └── java/
│           └── com/example/demo/
│               └── FlywayMigrationTest.java
├── pom.xml
└── README.md
```

This documentation provides everything you need from basics to production-ready practices. Save it, share it with your team, and refer to it whenever you need guidance on database migrations with Flyway!

---

**Last Updated**: October 2024  
**Version**: 1.0  
**Maintained by**: Your Development Team