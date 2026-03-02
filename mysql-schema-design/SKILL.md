---
name: mysql-schema-design
description: This skill should be used when designing MySQL database schemas, creating tables, or reviewing database structures. It applies when writing CREATE TABLE statements, designing indexes, choosing field types, or ensuring compliance with high-concurrency internet business standards. Triggers on requests like "create a MySQL table", "design database schema", "review table structure", or when working with SQL DDL statements.
---

# MySQL Schema Design

## Overview

Apply production-grade MySQL schema design standards for high-concurrency, large-scale internet applications. This skill provides comprehensive guidelines for table structure, field types, indexing strategies, and naming conventions based on industry best practices.

## When to Use This Skill

Use this skill when:
- Creating new MySQL tables or databases
- Reviewing existing table structures
- Designing indexes for query optimization
- Choosing appropriate field types and constraints
- Implementing multi-tenant data isolation
- Setting up soft delete mechanisms
- Designing versioned or relational tables

## Core Design Principles

### 1. Required Standard Fields

Every table MUST include these base fields:

```sql
`id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
`created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`deleted_at` bigint NOT NULL DEFAULT '0' COMMENT '删除时间'
```

### 2. Multi-Tenant Isolation

Business tables MUST include:

```sql
`space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID'
```

### 3. Audit Trail

Tables requiring audit tracking MUST include:

```sql
`created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
`updated_by` varchar(128) NOT NULL DEFAULT '' COMMENT '修改人'
```

### 4. Field Constraints

- All fields MUST be `NOT NULL` with appropriate default values
- All fields MUST have `COMMENT` annotations
- Use `utf8mb4` character set with `utf8mb4_general_ci` collation
- Use `InnoDB` storage engine

## Quick Reference: Common Patterns

### Primary Key
```sql
PRIMARY KEY (`id`)
```

### Unique Constraint (with soft delete)
```sql
UNIQUE KEY `uk_space_id_name` (`space_id`, `name`, `deleted_at`)
```

### Multi-Tenant Index
```sql
KEY `idx_space_id_status` (`space_id`, `status`, `deleted_at`)
```

### Field Type Selection

| Use Case | Type | Example |
|----------|------|---------|
| ID fields | `bigint unsigned` | `id`, `user_id`, `space_id` |
| Short strings | `varchar(128)` or `varchar(255)` | `name`, `email` |
| Descriptions | `varchar(1024)` or `varchar(2048)` | `description` |
| Long text/errors | `blob` | `err_msg`, `content` |
| Status codes | `int unsigned` or `tinyint` | `status`, `type` |
| Counters | `int` or `bigint unsigned` | `version_num`, `count` |
| JSON config | `json` | `spec`, `features` |
| Phone numbers | `varchar(20)` | `phone` |
| Money | `bigint` (store as cents) | `amount` |

## Table Templates

### Basic Business Table

Use for standard entity tables (users, products, orders, etc.):

```sql
CREATE TABLE IF NOT EXISTS `table_name`
(
    `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',

    -- Business fields
    `name` varchar(255) NOT NULL DEFAULT '' COMMENT '名称',
    `description` varchar(1024) NOT NULL DEFAULT '' COMMENT '描述',
    `status` varchar(128) NOT NULL DEFAULT '' COMMENT '状态',
    `spec` json DEFAULT NULL COMMENT '规格配置',

    -- Audit fields
    `created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
    `updated_by` varchar(128) NOT NULL DEFAULT '' COMMENT '修改人',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    `deleted_at` bigint NOT NULL DEFAULT '0' COMMENT '删除时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_space_id_name` (`space_id`, `name`, `deleted_at`),
    KEY `idx_space_id_status` (`space_id`, `status`, `deleted_at`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '表注释';
```

### Relationship Table

Use for many-to-many relationships:

```sql
CREATE TABLE IF NOT EXISTS `parent_child_ref`
(
    `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',
    `parent_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '父对象 ID',
    `child_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '子对象 ID',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_parent_child` (`space_id`, `parent_id`, `child_id`),
    KEY `idx_parent_id` (`space_id`, `parent_id`),
    KEY `idx_child_id` (`space_id`, `child_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '关联表';
```

### Version Management Table

Use for entities requiring version history:

```sql
CREATE TABLE IF NOT EXISTS `entity_version`
(
    `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',
    `entity_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '实体 ID',
    `version` varchar(64) NOT NULL DEFAULT '' COMMENT '版本号',
    `version_num` int NOT NULL DEFAULT '0' COMMENT '数字版本号',
    `content` blob COMMENT '版本内容',
    `change_log` json NOT NULL COMMENT '变更日志',
    `created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_entity_version` (`space_id`, `entity_id`, `version_num`),
    KEY `idx_entity_id` (`space_id`, `entity_id`, `created_at`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '版本表';
```

## Index Design Strategy

### Index Ordering Priority

1. `space_id` (multi-tenant isolation) - ALWAYS FIRST
2. Equality conditions (e.g., `status = 'active'`)
3. Range conditions (e.g., `created_at > '2024-01-01'`)
4. Sort fields (e.g., `ORDER BY updated_at`)

### Index Naming Conventions

- Primary key: `PRIMARY KEY`
- Unique index: `uk_` + field combination (e.g., `uk_space_id_name`)
- Regular index: `idx_` + field combination (e.g., `idx_space_id_status`)

### Index Best Practices

- Limit to 5 indexes per table
- Maximum 5 fields per index
- Include `deleted_at` in unique indexes for soft delete support
- Always start with `space_id` for multi-tenant queries
- Avoid indexing frequently updated or low-cardinality fields

## Naming Conventions

### Tables
- Lowercase with underscores: `user`, `dataset_version`
- Singular form: `user` not `users`
- Relationship tables: `parent_child_ref`

### Fields
- Lowercase with underscores
- ID fields: `id`, `user_id`, `space_id`
- Time fields: `created_at`, `updated_at`, `deleted_at`
- Boolean fields: `is_active`, `user_verified`

## Critical Rules (Must Follow)

1. ✅ Use InnoDB storage engine
2. ✅ Use utf8mb4 character set
3. ✅ All fields must have COMMENT annotations
4. ✅ All fields must be NOT NULL with defaults
5. ✅ Include standard timestamp fields
6. ✅ Include `space_id` for business tables
7. ✅ Soft delete with `deleted_at` bigint field
8. ❌ NO foreign key constraints (handle in application)
9. ❌ NO stored procedures, views, triggers, or events
10. ❌ NO ENUM types (use TINYINT or VARCHAR)
11. ❌ NO TEXT/BLOB for frequently queried fields
12. ❌ NO storing large files or photos in database

## SQL Writing Standards

When generating SQL queries:

1. ✅ Explicitly list columns (NO `SELECT *`)
2. ✅ Specify columns in INSERT statements
3. ✅ Use parameterized queries to prevent injection
4. ❌ NO implicit type conversions
5. ❌ NO functions in WHERE clauses
6. ❌ NO negative queries (NOT, !=, <>)
7. ❌ NO leading % in LIKE queries
8. ❌ Use IN instead of OR conditions

## Workflow

When creating or reviewing table schemas:

1. **Identify table type**: Basic business, relationship, or versioned
2. **Start with appropriate template** from above
3. **Add business-specific fields** following type guidelines
4. **Design indexes** based on query patterns
5. **Verify compliance** with critical rules
6. **Add comprehensive COMMENT annotations**
7. **Review index strategy** for multi-tenant and soft delete support

## Reference Documentation

For comprehensive details including:
- Complete field type specifications
- Detailed indexing strategies
- Performance optimization guidelines
- 58到家数据库30条军规 (58.com Database 30 Rules)
- Real-world examples

Refer to: `references/mysql-schema-design-guide.md`

Use grep to search the reference guide:
```bash
grep -i "keyword" references/mysql-schema-design-guide.md
```

Common search patterns:
- `grep -i "json"` - JSON field usage
- `grep -i "索引"` - Index design (Chinese)
- `grep -i "index"` - Index design (English)
- `grep -i "varchar"` - String field sizing
- `grep -i "timestamp"` - Time field patterns
