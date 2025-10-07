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