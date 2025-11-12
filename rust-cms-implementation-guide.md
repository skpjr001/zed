# Building a CMS in Rust with Axum & PostgreSQL

**Date:** 2025-11-12
**Inspired by:** Zed Architecture + Strapi CMS Design
**Tech Stack:** Rust + Axum + PostgreSQL + SQLx/SeaORM

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Architecture](#project-architecture)
3. [Database Layer Design](#database-layer-design)
4. [Entity & Content Type System](#entity--content-type-system)
5. [API Layer Implementation](#api-layer-implementation)
6. [Query Builder & Filtering](#query-builder--filtering)
7. [Relation Handling](#relation-handling)
8. [Plugin System](#plugin-system)
9. [Settings & Configuration](#settings--configuration)
10. [Authentication & Permissions](#authentication--permissions)
11. [Async Patterns & Performance](#async-patterns--performance)
12. [Error Handling & Logging](#error-handling--logging)
13. [Code Generation & Macros](#code-generation--macros)
14. [Complete Implementation Examples](#complete-implementation-examples)

---

## Executive Summary

This guide presents a **production-ready architecture** for building a headless CMS in Rust, combining:

- **Zed's patterns:** Entity system, type-safe database, procedural macros, async architecture
- **Strapi's features:** Content types, relations, plugins, RBAC, extensibility
- **Modern Rust ecosystem:** Axum, PostgreSQL, SQLx/SeaORM, Tower

### Target Architecture

```
CMS Application
├── Core Framework (inspired by GPUI)
│   ├── Entity system for content management
│   ├── Type-safe database layer
│   └── Context-based API access
│
├── Content Type System (inspired by Strapi)
│   ├── Schema definition & validation
│   ├── Automatic API generation
│   └── Lifecycle hooks
│
├── Plugin System
│   ├── Provider pattern for extensibility
│   ├── Registry for dynamic components
│   └── Lifecycle management
│
└── REST/GraphQL API Layer
    ├── Auto-generated CRUD endpoints
    ├── Query builder with filtering
    └── Relation population
```

---

## Project Architecture

### Workspace Structure

```
cms/
├── Cargo.toml                    # Workspace root
├── crates/
│   ├── cms-core/                 # Core framework
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── entity.rs         # Entity system
│   │       ├── context.rs        # Application context
│   │       └── registry.rs       # Service registry
│   │
│   ├── cms-db/                   # Database layer
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── connection.rs     # Connection pooling
│   │       ├── query_builder.rs  # Type-safe queries
│   │       ├── migrations.rs     # Migration system
│   │       └── transaction.rs    # Transaction management
│   │
│   ├── cms-db-macros/           # Database macros
│   │   └── src/
│   │       ├── lib.rs
│   │       └── query_macro.rs    # SQL validation
│   │
│   ├── cms-schema/              # Content type schema
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── content_type.rs   # Type definitions
│   │       ├── attributes.rs     # Attribute types
│   │       └── relations.rs      # Relation types
│   │
│   ├── cms-api/                 # REST API layer
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── router.rs         # Auto route generation
│   │       ├── controller.rs     # CRUD handlers
│   │       └── middleware.rs     # API middleware
│   │
│   ├── cms-services/            # Business logic
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── entity_service.rs # Entity CRUD
│   │       └── query_service.rs  # Query building
│   │
│   ├── cms-auth/                # Authentication
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── jwt.rs
│   │       ├── rbac.rs          # Role-based access
│   │       └── permissions.rs
│   │
│   ├── cms-plugins/             # Plugin system
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── plugin.rs         # Plugin trait
│   │       └── lifecycle.rs      # Lifecycle hooks
│   │
│   ├── cms-settings/            # Configuration
│   │   └── src/
│   │       ├── lib.rs
│   │       └── settings.rs
│   │
│   ├── cms-macros/              # Procedural macros
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── content_type.rs   # #[derive(ContentType)]
│   │       └── api_handler.rs    # #[derive(ApiHandler)]
│   │
│   └── cms-server/              # Main server
│       └── src/
│           └── main.rs
│
├── plugins/                     # Optional plugins
│   ├── upload-s3/
│   ├── email-smtp/
│   └── graphql/
│
└── examples/
    └── blog-cms/                # Example implementation
```

### Cargo.toml (Workspace)

```toml
[workspace]
resolver = "2"
members = [
    "crates/cms-core",
    "crates/cms-db",
    "crates/cms-db-macros",
    "crates/cms-schema",
    "crates/cms-api",
    "crates/cms-services",
    "crates/cms-auth",
    "crates/cms-plugins",
    "crates/cms-settings",
    "crates/cms-macros",
    "crates/cms-server",
]

[workspace.package]
edition = "2024"
version = "0.1.0"

[workspace.dependencies]
# Internal crates
cms-core = { path = "crates/cms-core" }
cms-db = { path = "crates/cms-db" }
cms-schema = { path = "crates/cms-schema" }
cms-api = { path = "crates/cms-api" }
cms-services = { path = "crates/cms-services" }
cms-macros = { path = "crates/cms-macros" }

# Async runtime
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"

# Web framework
axum = { version = "0.7", features = ["macros"] }
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.5", features = ["full"] }

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono", "json"] }
# OR
sea-orm = { version = "1.0", features = ["sqlx-postgres", "runtime-tokio-native-tls"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Error handling
anyhow = "1.0"
thiserror = "2.0"

# Utilities
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Authentication
jsonwebtoken = "9.0"
argon2 = "0.5"

# Schema
schemars = "1.0"
```

---

## Database Layer Design

### 1. Connection Pool & Management

**File:** `crates/cms-db/src/connection.rs`

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};
use std::sync::Arc;
use anyhow::Result;

#[derive(Clone)]
pub struct Database {
    pool: Arc<PgPool>,
}

impl Database {
    pub async fn connect(database_url: &str) -> Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(100)
            .min_connections(5)
            .connect(database_url)
            .await?;

        Ok(Self {
            pool: Arc::new(pool),
        })
    }

    pub fn pool(&self) -> &PgPool {
        &self.pool
    }

    /// Execute a query with automatic retry on connection errors
    pub async fn execute_with_retry<F, T>(&self, f: F) -> Result<T>
    where
        F: Fn(&PgPool) -> futures::future::BoxFuture<'_, Result<T>>,
    {
        let mut retries = 3;
        loop {
            match f(&self.pool).await {
                Ok(result) => return Ok(result),
                Err(e) if retries > 0 && is_retriable(&e) => {
                    retries -= 1;
                    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
                }
                Err(e) => return Err(e),
            }
        }
    }
}

fn is_retriable(err: &anyhow::Error) -> bool {
    // Check if error is a connection error
    err.to_string().contains("connection")
}
```

### 2. Type-Safe Query Builder

**File:** `crates/cms-db/src/query_builder.rs`

Inspired by Zed's sqlez patterns:

```rust
use sqlx::{Postgres, QueryBuilder, Arguments};
use serde::{Serialize, Deserialize};

pub struct TypedQueryBuilder<'a> {
    builder: QueryBuilder<'a, Postgres>,
}

impl<'a> TypedQueryBuilder<'a> {
    pub fn new(sql: &'a str) -> Self {
        Self {
            builder: QueryBuilder::new(sql),
        }
    }

    /// Bind a value with type safety
    pub fn bind<T>(mut self, value: T) -> Self
    where
        T: 'a + Send + sqlx::Encode<'a, Postgres> + sqlx::Type<Postgres>,
    {
        self.builder.push_bind(value);
        self
    }

    /// Bind an optional value
    pub fn bind_option<T>(mut self, value: Option<T>) -> Self
    where
        T: 'a + Send + sqlx::Encode<'a, Postgres> + sqlx::Type<Postgres>,
    {
        match value {
            Some(v) => self.builder.push_bind(v),
            None => self.builder.push("NULL"),
        };
        self
    }

    /// Add WHERE conditions
    pub fn where_clause(mut self, column: &str, op: &str) -> Self {
        self.builder.push(" WHERE ");
        self.builder.push(column);
        self.builder.push(" ");
        self.builder.push(op);
        self.builder.push(" ");
        self
    }

    /// Add ORDER BY
    pub fn order_by(mut self, column: &str, direction: SortOrder) -> Self {
        self.builder.push(" ORDER BY ");
        self.builder.push(column);
        match direction {
            SortOrder::Asc => self.builder.push(" ASC"),
            SortOrder::Desc => self.builder.push(" DESC"),
        };
        self
    }

    /// Add pagination
    pub fn paginate(mut self, limit: i64, offset: i64) -> Self {
        self.builder.push(" LIMIT ");
        self.builder.push_bind(limit);
        self.builder.push(" OFFSET ");
        self.builder.push_bind(offset);
        self
    }

    /// Build the query
    pub fn build(self) -> sqlx::query::Query<'a, Postgres, sqlx::postgres::PgArguments> {
        self.builder.build()
    }
}

#[derive(Debug, Clone, Copy)]
pub enum SortOrder {
    Asc,
    Desc,
}
```

### 3. Query Macro (Compile-Time Validation)

**File:** `crates/cms-db-macros/src/query_macro.rs`

```rust
use proc_macro::TokenStream;
use quote::quote;

/// Validates SQL at compile time using sqlx's offline mode
#[proc_macro]
pub fn query(input: TokenStream) -> TokenStream {
    let sql = input.to_string();

    // Use sqlx::query! for compile-time checking
    let expanded = quote! {
        sqlx::query!(#sql)
    };

    TokenStream::from(expanded)
}

/// Type-safe query with explicit return type
#[proc_macro]
pub fn query_as(input: TokenStream) -> TokenStream {
    // Parse input as: Type, "SQL"
    // Returns sqlx::query_as! macro invocation
    unimplemented!("Implement query_as macro")
}
```

**Usage:**

```rust
use cms_db_macros::query;

// Compile-time SQL validation
let users = query!(
    "SELECT id, name, email FROM users WHERE active = $1",
    true
)
.fetch_all(&pool)
.await?;
```

### 4. Migration System

**File:** `crates/cms-db/src/migrations.rs`

Inspired by Zed's domain-based migrations:

```rust
use anyhow::Result;
use async_trait::async_trait;

#[async_trait]
pub trait Migration: Send + Sync {
    fn version(&self) -> i64;
    fn name(&self) -> &str;
    async fn up(&self, db: &Database) -> Result<()>;
    async fn down(&self, db: &Database) -> Result<()>;
}

pub struct MigrationRunner {
    migrations: Vec<Box<dyn Migration>>,
}

impl MigrationRunner {
    pub fn new() -> Self {
        Self {
            migrations: Vec::new(),
        }
    }

    pub fn add(mut self, migration: Box<dyn Migration>) -> Self {
        self.migrations.push(migration);
        self.migrations.sort_by_key(|m| m.version());
        self
    }

    pub async fn run(&self, db: &Database) -> Result<()> {
        // Create migrations table
        sqlx::query(
            r#"
            CREATE TABLE IF NOT EXISTS _migrations (
                version BIGINT PRIMARY KEY,
                name TEXT NOT NULL,
                applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
            )
            "#,
        )
        .execute(db.pool())
        .await?;

        // Get applied migrations
        let applied: Vec<i64> = sqlx::query_scalar(
            "SELECT version FROM _migrations ORDER BY version"
        )
        .fetch_all(db.pool())
        .await?;

        // Run pending migrations
        for migration in &self.migrations {
            if applied.contains(&migration.version()) {
                continue;
            }

            tracing::info!("Running migration: {}", migration.name());

            // Run in transaction
            let mut tx = db.pool().begin().await?;

            migration.up(db).await?;

            sqlx::query(
                "INSERT INTO _migrations (version, name) VALUES ($1, $2)"
            )
            .bind(migration.version())
            .bind(migration.name())
            .execute(&mut *tx)
            .await?;

            tx.commit().await?;

            tracing::info!("Completed migration: {}", migration.name());
        }

        Ok(())
    }
}

// Example migration
pub struct CreateUsersTable;

#[async_trait]
impl Migration for CreateUsersTable {
    fn version(&self) -> i64 {
        1
    }

    fn name(&self) -> &str {
        "create_users_table"
    }

    async fn up(&self, db: &Database) -> Result<()> {
        sqlx::query(
            r#"
            CREATE TABLE users (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                email VARCHAR(255) NOT NULL UNIQUE,
                password_hash TEXT NOT NULL,
                name VARCHAR(255),
                active BOOLEAN NOT NULL DEFAULT true,
                created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
                updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
            )
            "#,
        )
        .execute(db.pool())
        .await?;

        Ok(())
    }

    async fn down(&self, db: &Database) -> Result<()> {
        sqlx::query("DROP TABLE users")
            .execute(db.pool())
            .await?;
        Ok(())
    }
}
```

### 5. Transaction Support

**File:** `crates/cms-db/src/transaction.rs`

```rust
use sqlx::{PgPool, Postgres, Transaction};
use anyhow::Result;

pub struct TransactionContext<'a> {
    tx: Transaction<'a, Postgres>,
}

impl<'a> TransactionContext<'a> {
    pub async fn begin(pool: &'a PgPool) -> Result<Self> {
        let tx = pool.begin().await?;
        Ok(Self { tx })
    }

    pub async fn commit(self) -> Result<()> {
        self.tx.commit().await?;
        Ok(())
    }

    pub async fn rollback(self) -> Result<()> {
        self.tx.rollback().await?;
        Ok(())
    }

    pub fn tx_mut(&mut self) -> &mut Transaction<'a, Postgres> {
        &mut self.tx
    }
}

/// Helper function for transactions
pub async fn with_transaction<F, T>(pool: &PgPool, f: F) -> Result<T>
where
    F: for<'a> FnOnce(&mut Transaction<'a, Postgres>) -> futures::future::BoxFuture<'a, Result<T>>,
{
    let mut tx = pool.begin().await?;

    let result = f(&mut tx).await;

    match result {
        Ok(value) => {
            tx.commit().await?;
            Ok(value)
        }
        Err(e) => {
            tx.rollback().await?;
            Err(e)
        }
    }
}
```

**Usage:**

```rust
use cms_db::with_transaction;

let result = with_transaction(db.pool(), |tx| {
    Box::pin(async move {
        // Insert user
        let user_id = sqlx::query_scalar(
            "INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id"
        )
        .bind("user@example.com")
        .bind("hash")
        .fetch_one(&mut **tx)
        .await?;

        // Create user profile
        sqlx::query(
            "INSERT INTO profiles (user_id, bio) VALUES ($1, $2)"
        )
        .bind(user_id)
        .bind("Bio text")
        .execute(&mut **tx)
        .await?;

        Ok(user_id)
    })
}).await?;
```

---

## Entity & Content Type System

### 1. Content Type Schema

**File:** `crates/cms-schema/src/content_type.rs`

Inspired by Strapi's content type system:

```rust
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ContentTypeSchema {
    pub uid: String,
    pub kind: ContentTypeKind,
    pub info: ContentTypeInfo,
    pub options: ContentTypeOptions,
    pub attributes: HashMap<String, Attribute>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "camelCase")]
pub enum ContentTypeKind {
    CollectionType,
    SingleType,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ContentTypeInfo {
    pub display_name: String,
    pub singular_name: String,
    pub plural_name: String,
    pub description: Option<String>,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize, JsonSchema)]
pub struct ContentTypeOptions {
    pub draft_and_publish: Option<bool>,
    pub timestamps: Option<bool>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(tag = "type", rename_all = "lowercase")]
pub enum Attribute {
    String {
        required: Option<bool>,
        unique: Option<bool>,
        min_length: Option<usize>,
        max_length: Option<usize>,
        default: Option<String>,
    },
    Text {
        required: Option<bool>,
        default: Option<String>,
    },
    RichText {
        required: Option<bool>,
    },
    Email {
        required: Option<bool>,
        unique: Option<bool>,
    },
    Password {
        required: Option<bool>,
        min_length: Option<usize>,
    },
    Integer {
        required: Option<bool>,
        min: Option<i64>,
        max: Option<i64>,
        default: Option<i64>,
    },
    BigInteger {
        required: Option<bool>,
    },
    Float {
        required: Option<bool>,
    },
    Decimal {
        required: Option<bool>,
        precision: Option<u8>,
        scale: Option<u8>,
    },
    Boolean {
        required: Option<bool>,
        default: Option<bool>,
    },
    Date {
        required: Option<bool>,
    },
    DateTime {
        required: Option<bool>,
    },
    Time {
        required: Option<bool>,
    },
    Json {
        required: Option<bool>,
    },
    Enumeration {
        required: Option<bool>,
        values: Vec<String>,
        default: Option<String>,
    },
    Uid {
        target_field: Option<String>,
        required: Option<bool>,
    },
    Relation {
        relation: RelationType,
        target: String,
        mapped_by: Option<String>,
        inversed_by: Option<String>,
    },
    Component {
        component: String,
        repeatable: Option<bool>,
        required: Option<bool>,
    },
    DynamicZone {
        components: Vec<String>,
        required: Option<bool>,
    },
    Media {
        multiple: Option<bool>,
        allowed_types: Option<Vec<String>>,
        required: Option<bool>,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(rename_all = "camelCase")]
pub enum RelationType {
    OneToOne,
    OneToMany,
    ManyToOne,
    ManyToMany,
}
```

### 2. Content Type Derive Macro

**File:** `crates/cms-macros/src/content_type.rs`

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(ContentType, attributes(content_type))]
pub fn derive_content_type(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let expanded = quote! {
        impl ContentType for #name {
            fn type_name() -> &'static str {
                stringify!(#name)
            }

            fn schema() -> ContentTypeSchema {
                // Generate schema from struct fields
                Self::generate_schema()
            }
        }

        impl #name {
            fn generate_schema() -> ContentTypeSchema {
                // Use schemars to generate JSON schema
                let schema_generator = schemars::gen::SchemaGenerator::default();
                let schema = schema_generator.into_root_schema_for::<#name>();

                // Convert to ContentTypeSchema
                ContentTypeSchema::from_json_schema(schema)
            }
        }
    };

    TokenStream::from(expanded)
}
```

**Usage:**

```rust
use cms_macros::ContentType;
use cms_schema::*;

#[derive(ContentType, Serialize, Deserialize, JsonSchema)]
#[content_type(
    kind = "collection",
    display_name = "Article",
    singular = "article",
    plural = "articles"
)]
pub struct Article {
    #[serde(default)]
    pub id: uuid::Uuid,

    pub title: String,

    pub slug: String,

    pub content: String,

    pub published: bool,

    #[serde(default)]
    pub published_at: Option<chrono::DateTime<chrono::Utc>>,

    // Relation
    pub author: Relation<User>,

    // Component
    pub seo: Component<SeoMetadata>,

    #[serde(default)]
    pub created_at: chrono::DateTime<chrono::Utc>,

    #[serde(default)]
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

// Relation wrapper
pub struct Relation<T> {
    pub id: Option<uuid::Uuid>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<Box<T>>,
}

// Component wrapper
pub struct Component<T> {
    #[serde(flatten)]
    pub data: T,
}
```

### 3. Entity Service (CRUD Operations)

**File:** `crates/cms-services/src/entity_service.rs`

Inspired by Strapi's EntityService:

```rust
use anyhow::Result;
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use serde_json::Value;
use uuid::Uuid;

#[async_trait]
pub trait EntityService: Send + Sync {
    /// Find multiple entities
    async fn find_many(
        &self,
        uid: &str,
        params: FindManyParams,
    ) -> Result<FindManyResult>;

    /// Find one entity
    async fn find_one(
        &self,
        uid: &str,
        id: Uuid,
        params: FindOneParams,
    ) -> Result<Option<Value>>;

    /// Create entity
    async fn create(
        &self,
        uid: &str,
        data: Value,
    ) -> Result<Value>;

    /// Update entity
    async fn update(
        &self,
        uid: &str,
        id: Uuid,
        data: Value,
    ) -> Result<Value>;

    /// Delete entity
    async fn delete(
        &self,
        uid: &str,
        id: Uuid,
    ) -> Result<Value>;

    /// Count entities
    async fn count(
        &self,
        uid: &str,
        filters: Option<Value>,
    ) -> Result<i64>;
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FindManyParams {
    pub filters: Option<Value>,
    pub sort: Option<Vec<SortOption>>,
    pub pagination: Option<Pagination>,
    pub populate: Option<Value>,
    pub fields: Option<Vec<String>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FindOneParams {
    pub populate: Option<Value>,
    pub fields: Option<Vec<String>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FindManyResult {
    pub data: Vec<Value>,
    pub pagination: PaginationInfo,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SortOption {
    pub field: String,
    pub order: SortOrder,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum SortOrder {
    Asc,
    Desc,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Pagination {
    pub page: Option<i64>,
    pub page_size: Option<i64>,
    pub start: Option<i64>,
    pub limit: Option<i64>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PaginationInfo {
    pub page: i64,
    pub page_size: i64,
    pub page_count: i64,
    pub total: i64,
}
```

**Implementation:**

```rust
pub struct PostgresEntityService {
    db: Database,
    schema_registry: Arc<SchemaRegistry>,
}

#[async_trait]
impl EntityService for PostgresEntityService {
    async fn find_many(
        &self,
        uid: &str,
        params: FindManyParams,
    ) -> Result<FindManyResult> {
        let schema = self.schema_registry.get(uid)?;

        // Build query
        let mut query = TypedQueryBuilder::new(&format!(
            "SELECT * FROM {}",
            schema.table_name()
        ));

        // Apply filters
        if let Some(filters) = params.filters {
            query = self.apply_filters(query, filters)?;
        }

        // Apply sorting
        if let Some(sort) = params.sort {
            for sort_option in sort {
                query = query.order_by(&sort_option.field, sort_option.order.into());
            }
        }

        // Apply pagination
        let pagination = params.pagination.unwrap_or_default();
        let page = pagination.page.unwrap_or(1);
        let page_size = pagination.page_size.unwrap_or(25);
        let offset = (page - 1) * page_size;

        query = query.paginate(page_size, offset);

        // Execute query
        let rows = query.build()
            .fetch_all(self.db.pool())
            .await?;

        // Get total count
        let total = self.count(uid, params.filters).await?;

        let data: Vec<Value> = rows
            .into_iter()
            .map(|row| self.row_to_json(&row, &schema))
            .collect::<Result<_>>()?;

        Ok(FindManyResult {
            data,
            pagination: PaginationInfo {
                page,
                page_size,
                page_count: (total as f64 / page_size as f64).ceil() as i64,
                total,
            },
        })
    }

    async fn create(
        &self,
        uid: &str,
        data: Value,
    ) -> Result<Value> {
        let schema = self.schema_registry.get(uid)?;

        // Run lifecycle hooks
        self.run_lifecycle_hook(uid, "beforeCreate", &data).await?;

        // Build insert query
        let columns: Vec<String> = data.as_object()
            .unwrap()
            .keys()
            .cloned()
            .collect();

        let placeholders: Vec<String> = (1..=columns.len())
            .map(|i| format!("${}", i))
            .collect();

        let sql = format!(
            "INSERT INTO {} ({}) VALUES ({}) RETURNING *",
            schema.table_name(),
            columns.join(", "),
            placeholders.join(", ")
        );

        // Execute with transaction
        let result = with_transaction(self.db.pool(), |tx| {
            Box::pin(async move {
                let row = self.execute_insert(&sql, &data, tx).await?;
                let entity = self.row_to_json(&row, &schema)?;

                // Handle relations
                self.handle_relations(uid, &entity, &data, tx).await?;

                Ok(entity)
            })
        }).await?;

        // Run lifecycle hooks
        self.run_lifecycle_hook(uid, "afterCreate", &result).await?;

        Ok(result)
    }
}
```

---

## API Layer Implementation

### 1. Auto-Generated REST Routes

**File:** `crates/cms-api/src/router.rs`

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{Path, State, Query},
    Json,
};
use std::sync::Arc;

pub struct ApiRouter {
    entity_service: Arc<dyn EntityService>,
    schema_registry: Arc<SchemaRegistry>,
}

impl ApiRouter {
    pub fn new(
        entity_service: Arc<dyn EntityService>,
        schema_registry: Arc<SchemaRegistry>,
    ) -> Self {
        Self {
            entity_service,
            schema_registry,
        }
    }

    /// Generate routes for all content types
    pub fn build_routes(self) -> Router {
        let mut router = Router::new();

        // For each content type, create CRUD routes
        for (uid, schema) in self.schema_registry.all() {
            let plural = &schema.info.plural_name;

            router = router
                // GET /api/articles
                .route(
                    &format!("/api/{}", plural),
                    get(Self::find_many),
                )
                // GET /api/articles/:id
                .route(
                    &format!("/api/{}/:id", plural),
                    get(Self::find_one),
                )
                // POST /api/articles
                .route(
                    &format!("/api/{}", plural),
                    post(Self::create),
                )
                // PUT /api/articles/:id
                .route(
                    &format!("/api/{}/:id", plural),
                    put(Self::update),
                )
                // DELETE /api/articles/:id
                .route(
                    &format!("/api/{}/:id", plural),
                    delete(Self::delete),
                );
        }

        router.with_state(Arc::new(self))
    }

    /// Handler: Find many
    async fn find_many(
        State(api): State<Arc<ApiRouter>>,
        Path(uid): Path<String>,
        Query(params): Query<FindManyParams>,
    ) -> Result<Json<FindManyResult>, ApiError> {
        let result = api.entity_service
            .find_many(&uid, params)
            .await?;

        Ok(Json(result))
    }

    /// Handler: Find one
    async fn find_one(
        State(api): State<Arc<ApiRouter>>,
        Path((uid, id)): Path<(String, Uuid)>,
        Query(params): Query<FindOneParams>,
    ) -> Result<Json<Value>, ApiError> {
        let entity = api.entity_service
            .find_one(&uid, id, params)
            .await?
            .ok_or(ApiError::NotFound)?;

        Ok(Json(entity))
    }

    /// Handler: Create
    async fn create(
        State(api): State<Arc<ApiRouter>>,
        Path(uid): Path<String>,
        Json(data): Json<Value>,
    ) -> Result<Json<Value>, ApiError> {
        let entity = api.entity_service
            .create(&uid, data)
            .await?;

        Ok(Json(entity))
    }

    /// Handler: Update
    async fn update(
        State(api): State<Arc<ApiRouter>>,
        Path((uid, id)): Path<(String, Uuid)>,
        Json(data): Json<Value>,
    ) -> Result<Json<Value>, ApiError> {
        let entity = api.entity_service
            .update(&uid, id, data)
            .await?;

        Ok(Json(entity))
    }

    /// Handler: Delete
    async fn delete(
        State(api): State<Arc<ApiRouter>>,
        Path((uid, id)): Path<(String, Uuid)>,
    ) -> Result<Json<Value>, ApiError> {
        let entity = api.entity_service
            .delete(&uid, id)
            .await?;

        Ok(Json(entity))
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("Not found")]
    NotFound,

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Internal error: {0}")]
    Internal(#[from] anyhow::Error),
}

// Implement IntoResponse for ApiError
impl axum::response::IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            ApiError::NotFound => (
                axum::http::StatusCode::NOT_FOUND,
                "Resource not found".to_string(),
            ),
            ApiError::Database(e) => (
                axum::http::StatusCode::INTERNAL_SERVER_ERROR,
                format!("Database error: {}", e),
            ),
            ApiError::Internal(e) => (
                axum::http::StatusCode::INTERNAL_SERVER_ERROR,
                format!("Internal error: {}", e),
            ),
        };

        let body = serde_json::json!({
            "error": {
                "status": status.as_u16(),
                "message": message,
            }
        });

        (status, Json(body)).into_response()
    }
}
```

### 2. Middleware Stack

**File:** `crates/cms-api/src/middleware.rs`

```rust
use tower_http::{
    cors::{CorsLayer, Any},
    trace::TraceLayer,
    compression::CompressionLayer,
};
use tower::ServiceBuilder;

pub fn middleware_stack() -> ServiceBuilder<
    tower::layer::util::Stack<
        TraceLayer,
        tower::layer::util::Stack<CorsLayer, CompressionLayer>
    >
> {
    ServiceBuilder::new()
        // Logging
        .layer(TraceLayer::new_for_http())
        // CORS
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
                .allow_methods(Any)
                .allow_headers(Any)
        )
        // Compression
        .layer(CompressionLayer::new())
}
```

---

## Query Builder & Filtering

**File:** `crates/cms-db/src/query_builder.rs` (extended)

### Filter Operations

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum FilterValue {
    Simple(Value),
    Operator(FilterOperator),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilterOperator {
    #[serde(rename = "$eq", skip_serializing_if = "Option::is_none")]
    pub eq: Option<Value>,

    #[serde(rename = "$ne", skip_serializing_if = "Option::is_none")]
    pub ne: Option<Value>,

    #[serde(rename = "$in", skip_serializing_if = "Option::is_none")]
    pub r#in: Option<Vec<Value>>,

    #[serde(rename = "$notIn", skip_serializing_if = "Option::is_none")]
    pub not_in: Option<Vec<Value>>,

    #[serde(rename = "$lt", skip_serializing_if = "Option::is_none")]
    pub lt: Option<Value>,

    #[serde(rename = "$lte", skip_serializing_if = "Option::is_none")]
    pub lte: Option<Value>,

    #[serde(rename = "$gt", skip_serializing_if = "Option::is_none")]
    pub gt: Option<Value>,

    #[serde(rename = "$gte", skip_serializing_if = "Option::is_none")]
    pub gte: Option<Value>,

    #[serde(rename = "$contains", skip_serializing_if = "Option::is_none")]
    pub contains: Option<String>,

    #[serde(rename = "$notContains", skip_serializing_if = "Option::is_none")]
    pub not_contains: Option<String>,

    #[serde(rename = "$containsi", skip_serializing_if = "Option::is_none")]
    pub containsi: Option<String>,

    #[serde(rename = "$startsWith", skip_serializing_if = "Option::is_none")]
    pub starts_with: Option<String>,

    #[serde(rename = "$endsWith", skip_serializing_if = "Option::is_none")]
    pub ends_with: Option<String>,

    #[serde(rename = "$null", skip_serializing_if = "Option::is_none")]
    pub null: Option<bool>,

    #[serde(rename = "$notNull", skip_serializing_if = "Option::is_none")]
    pub not_null: Option<bool>,

    #[serde(rename = "$between", skip_serializing_if = "Option::is_none")]
    pub between: Option<[Value; 2]>,
}

/// Build WHERE clause from filters
pub fn build_where_clause(
    filters: &Value,
    params: &mut Vec<Value>,
) -> Result<String> {
    let obj = filters.as_object()
        .ok_or_else(|| anyhow::anyhow!("Filters must be an object"))?;

    let conditions: Vec<String> = obj.iter()
        .map(|(key, value)| {
            build_condition(key, value, params)
        })
        .collect::<Result<_>>()?;

    if conditions.is_empty() {
        Ok("1=1".to_string())
    } else {
        Ok(conditions.join(" AND "))
    }
}

fn build_condition(
    field: &str,
    value: &Value,
    params: &mut Vec<Value>,
) -> Result<String> {
    // Handle logical operators
    if field == "$or" {
        let conditions: Vec<String> = value.as_array()
            .ok_or_else(|| anyhow::anyhow!("$or must be an array"))?
            .iter()
            .map(|v| build_where_clause(v, params))
            .collect::<Result<_>>()?;

        return Ok(format!("({})", conditions.join(" OR ")));
    }

    if field == "$and" {
        let conditions: Vec<String> = value.as_array()
            .ok_or_else(|| anyhow::anyhow!("$and must be an array"))?
            .iter()
            .map(|v| build_where_clause(v, params))
            .collect::<Result<_>>()?;

        return Ok(format!("({})", conditions.join(" AND ")));
    }

    // Parse operator
    if let Ok(op) = serde_json::from_value::<FilterOperator>(value.clone()) {
        build_operator_condition(field, &op, params)
    } else {
        // Simple equality
        params.push(value.clone());
        Ok(format!("{} = ${}", field, params.len()))
    }
}

fn build_operator_condition(
    field: &str,
    op: &FilterOperator,
    params: &mut Vec<Value>,
) -> Result<String> {
    let mut conditions = Vec::new();

    if let Some(v) = &op.eq {
        params.push(v.clone());
        conditions.push(format!("{} = ${}", field, params.len()));
    }

    if let Some(v) = &op.ne {
        params.push(v.clone());
        conditions.push(format!("{} != ${}", field, params.len()));
    }

    if let Some(values) = &op.r#in {
        let placeholders: Vec<String> = values.iter()
            .map(|v| {
                params.push(v.clone());
                format!("${}", params.len())
            })
            .collect();

        conditions.push(format!("{} IN ({})", field, placeholders.join(", ")));
    }

    if let Some(v) = &op.lt {
        params.push(v.clone());
        conditions.push(format!("{} < ${}", field, params.len()));
    }

    if let Some(v) = &op.lte {
        params.push(v.clone());
        conditions.push(format!("{} <= ${}", field, params.len()));
    }

    if let Some(v) = &op.gt {
        params.push(v.clone());
        conditions.push(format!("{} > ${}", field, params.len()));
    }

    if let Some(v) = &op.gte {
        params.push(v.clone());
        conditions.push(format!("{} >= ${}", field, params.len()));
    }

    if let Some(s) = &op.contains {
        params.push(Value::String(format!("%{}%", s)));
        conditions.push(format!("{} LIKE ${}", field, params.len()));
    }

    if let Some(s) = &op.containsi {
        params.push(Value::String(format!("%{}%", s)));
        conditions.push(format!("{} ILIKE ${}", field, params.len()));
    }

    if let Some(s) = &op.starts_with {
        params.push(Value::String(format!("{}%", s)));
        conditions.push(format!("{} LIKE ${}", field, params.len()));
    }

    if let Some(s) = &op.ends_with {
        params.push(Value::String(format!("%{}", s)));
        conditions.push(format!("{} LIKE ${}", field, params.len()));
    }

    if let Some(true) = op.null {
        conditions.push(format!("{} IS NULL", field));
    }

    if let Some(true) = op.not_null {
        conditions.push(format!("{} IS NOT NULL", field));
    }

    if let Some([min, max]) = &op.between {
        params.push(min.clone());
        let min_idx = params.len();
        params.push(max.clone());
        let max_idx = params.len();
        conditions.push(format!("{} BETWEEN ${} AND ${}", field, min_idx, max_idx));
    }

    if conditions.is_empty() {
        Ok("1=1".to_string())
    } else {
        Ok(conditions.join(" AND "))
    }
}
```

**Usage example:**

```json
{
  "filters": {
    "$or": [
      {
        "title": {
          "$containsi": "rust"
        }
      },
      {
        "author": {
          "name": {
            "$eq": "John Doe"
          }
        }
      }
    ],
    "published": true,
    "views": {
      "$gte": 1000
    }
  }
}
```

---

## Relation Handling

### Population Strategy

**File:** `crates/cms-services/src/population.rs`

```rust
use serde_json::Value;
use std::collections::HashMap;

pub struct PopulationService {
    db: Database,
    schema_registry: Arc<SchemaRegistry>,
}

impl PopulationService {
    /// Populate relations in entities
    pub async fn populate(
        &self,
        uid: &str,
        entities: &mut [Value],
        populate_config: &Value,
    ) -> Result<()> {
        let schema = self.schema_registry.get(uid)?;

        // Parse populate configuration
        let populate_fields = self.parse_populate_config(populate_config)?;

        // Group by relation type for efficient querying
        for (field_name, nested_populate) in populate_fields {
            let attribute = schema.attributes.get(&field_name)
                .ok_or_else(|| anyhow::anyhow!("Unknown field: {}", field_name))?;

            match attribute {
                Attribute::Relation { relation, target, .. } => {
                    self.populate_relation(
                        entities,
                        &field_name,
                        relation,
                        target,
                        nested_populate.as_ref(),
                    ).await?;
                }
                _ => {}
            }
        }

        Ok(())
    }

    async fn populate_relation(
        &self,
        entities: &mut [Value],
        field_name: &str,
        relation: &RelationType,
        target: &str,
        nested_populate: Option<&Value>,
    ) -> Result<()> {
        match relation {
            RelationType::ManyToOne | RelationType::OneToOne => {
                self.populate_to_one(entities, field_name, target, nested_populate).await
            }
            RelationType::OneToMany | RelationType::ManyToMany => {
                self.populate_to_many(entities, field_name, target, nested_populate).await
            }
        }
    }

    async fn populate_to_one(
        &self,
        entities: &mut [Value],
        field_name: &str,
        target_uid: &str,
        nested_populate: Option<&Value>,
    ) -> Result<()> {
        // Collect all foreign key IDs
        let ids: Vec<Uuid> = entities.iter()
            .filter_map(|entity| {
                entity.get(field_name)
                    .and_then(|v| v.as_str())
                    .and_then(|s| Uuid::parse_str(s).ok())
            })
            .collect();

        if ids.is_empty() {
            return Ok(());
        }

        // Fetch related entities in single query
        let sql = format!(
            "SELECT * FROM {} WHERE id = ANY($1)",
            target_uid
        );

        let rows = sqlx::query(&sql)
            .bind(&ids)
            .fetch_all(self.db.pool())
            .await?;

        // Build lookup map
        let mut related_map: HashMap<Uuid, Value> = HashMap::new();
        for row in rows {
            let id: Uuid = row.try_get("id")?;
            let entity = self.row_to_json(&row)?;
            related_map.insert(id, entity);
        }

        // Populate nested relations if requested
        if let Some(nested) = nested_populate {
            let mut related_entities: Vec<Value> = related_map.values().cloned().collect();
            self.populate(target_uid, &mut related_entities, nested).await?;

            // Update map with populated entities
            for entity in related_entities {
                if let Some(id_str) = entity.get("id").and_then(|v| v.as_str()) {
                    if let Ok(id) = Uuid::parse_str(id_str) {
                        related_map.insert(id, entity);
                    }
                }
            }
        }

        // Attach to entities
        for entity in entities.iter_mut() {
            if let Some(id_str) = entity.get(field_name).and_then(|v| v.as_str()) {
                if let Ok(id) = Uuid::parse_str(id_str) {
                    if let Some(related) = related_map.get(&id) {
                        entity[field_name] = related.clone();
                    }
                }
            }
        }

        Ok(())
    }

    async fn populate_to_many(
        &self,
        entities: &mut [Value],
        field_name: &str,
        target_uid: &str,
        nested_populate: Option<&Value>,
    ) -> Result<()> {
        // Get entity IDs
        let entity_ids: Vec<Uuid> = entities.iter()
            .filter_map(|e| {
                e.get("id")
                    .and_then(|v| v.as_str())
                    .and_then(|s| Uuid::parse_str(s).ok())
            })
            .collect();

        if entity_ids.is_empty() {
            return Ok(());
        }

        // Query join table
        let join_table = format!("{}_{}_links", target_uid, field_name);

        let sql = format!(
            "SELECT parent_id, related_id, relation_order
             FROM {}
             WHERE parent_id = ANY($1)
             ORDER BY relation_order",
            join_table
        );

        let links: Vec<(Uuid, Uuid, i32)> = sqlx::query_as(&sql)
            .bind(&entity_ids)
            .fetch_all(self.db.pool())
            .await?;

        // Get unique related IDs
        let related_ids: Vec<Uuid> = links.iter()
            .map(|(_, related_id, _)| *related_id)
            .collect();

        // Fetch all related entities
        let sql = format!(
            "SELECT * FROM {} WHERE id = ANY($1)",
            target_uid
        );

        let rows = sqlx::query(&sql)
            .bind(&related_ids)
            .fetch_all(self.db.pool())
            .await?;

        let mut related_map: HashMap<Uuid, Value> = HashMap::new();
        for row in rows {
            let id: Uuid = row.try_get("id")?;
            let entity = self.row_to_json(&row)?;
            related_map.insert(id, entity);
        }

        // Populate nested if requested
        if let Some(nested) = nested_populate {
            let mut related_entities: Vec<Value> = related_map.values().cloned().collect();
            self.populate(target_uid, &mut related_entities, nested).await?;

            for entity in related_entities {
                if let Some(id_str) = entity.get("id").and_then(|v| v.as_str()) {
                    if let Ok(id) = Uuid::parse_str(id_str) {
                        related_map.insert(id, entity);
                    }
                }
            }
        }

        // Group links by parent
        let mut links_by_parent: HashMap<Uuid, Vec<(Uuid, i32)>> = HashMap::new();
        for (parent_id, related_id, order) in links {
            links_by_parent.entry(parent_id)
                .or_default()
                .push((related_id, order));
        }

        // Attach to entities
        for entity in entities.iter_mut() {
            if let Some(id_str) = entity.get("id").and_then(|v| v.as_str()) {
                if let Ok(id) = Uuid::parse_str(id_str) {
                    if let Some(links) = links_by_parent.get(&id) {
                        let mut related: Vec<Value> = links.iter()
                            .filter_map(|(related_id, _)| {
                                related_map.get(related_id).cloned()
                            })
                            .collect();

                        entity[field_name] = Value::Array(related);
                    }
                }
            }
        }

        Ok(())
    }
}
```

---

## Plugin System

**File:** `crates/cms-plugins/src/plugin.rs`

Inspired by Zed's registry pattern and Strapi's plugin system:

```rust
use async_trait::async_trait;
use std::any::Any;
use std::sync::Arc;

#[async_trait]
pub trait Plugin: Send + Sync + 'static {
    /// Plugin unique identifier
    fn name(&self) -> &str;

    /// Plugin version
    fn version(&self) -> &str {
        "0.1.0"
    }

    /// Register phase - called before bootstrap
    async fn register(&self, app: &mut App) -> Result<()> {
        Ok(())
    }

    /// Bootstrap phase - called after all plugins registered
    async fn bootstrap(&self, app: &mut App) -> Result<()> {
        Ok(())
    }

    /// Cleanup on shutdown
    async fn destroy(&self, app: &mut App) -> Result<()> {
        Ok(())
    }

    /// Provide custom routes
    fn routes(&self) -> Vec<PluginRoute> {
        Vec::new()
    }

    /// Provide custom services
    fn services(&self) -> HashMap<String, Box<dyn Any + Send + Sync>> {
        HashMap::new()
    }

    /// Provide custom content types
    fn content_types(&self) -> Vec<ContentTypeSchema> {
        Vec::new()
    }

    /// Provide custom middlewares
    fn middlewares(&self) -> Vec<PluginMiddleware> {
        Vec::new()
    }
}

pub struct PluginRoute {
    pub method: http::Method,
    pub path: String,
    pub handler: Arc<dyn PluginRouteHandler>,
}

#[async_trait]
pub trait PluginRouteHandler: Send + Sync {
    async fn handle(&self, req: axum::extract::Request) -> axum::response::Response;
}

pub struct PluginMiddleware {
    pub name: String,
    pub handler: Arc<dyn tower::Layer<axum::Router>>,
}

/// Plugin registry
pub struct PluginRegistry {
    plugins: HashMap<String, Arc<dyn Plugin>>,
}

impl PluginRegistry {
    pub fn new() -> Self {
        Self {
            plugins: HashMap::new(),
        }
    }

    pub fn register(&mut self, plugin: Arc<dyn Plugin>) {
        let name = plugin.name().to_string();
        self.plugins.insert(name, plugin);
    }

    pub fn get(&self, name: &str) -> Option<&Arc<dyn Plugin>> {
        self.plugins.get(name)
    }

    pub async fn run_lifecycle(
        &self,
        phase: PluginLifecycle,
        app: &mut App,
    ) -> Result<()> {
        for plugin in self.plugins.values() {
            match phase {
                PluginLifecycle::Register => plugin.register(app).await?,
                PluginLifecycle::Bootstrap => plugin.bootstrap(app).await?,
                PluginLifecycle::Destroy => plugin.destroy(app).await?,
            }
        }
        Ok(())
    }
}

pub enum PluginLifecycle {
    Register,
    Bootstrap,
    Destroy,
}
```

**Example plugin:**

```rust
pub struct S3UploadPlugin {
    config: S3Config,
}

#[async_trait]
impl Plugin for S3UploadPlugin {
    fn name(&self) -> &str {
        "upload-s3"
    }

    async fn register(&self, app: &mut App) -> Result<()> {
        // Register upload provider
        app.register_provider("upload", Arc::new(S3UploadProvider::new(self.config.clone())));
        Ok(())
    }

    fn routes(&self) -> Vec<PluginRoute> {
        vec![
            PluginRoute {
                method: http::Method::POST,
                path: "/api/upload".to_string(),
                handler: Arc::new(UploadHandler),
            },
        ]
    }
}
```

---

## Settings & Configuration

**File:** `crates/cms-settings/src/settings.rs`

Inspired by Zed's settings system:

```rust
use serde::{Deserialize, Serialize};
use schemars::JsonSchema;

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Settings {
    pub server: ServerSettings,
    pub database: DatabaseSettings,
    pub auth: AuthSettings,
    pub plugins: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ServerSettings {
    pub host: String,
    pub port: u16,
    pub base_url: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct DatabaseSettings {
    pub url: String,
    pub max_connections: u32,
    pub min_connections: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct AuthSettings {
    pub jwt_secret: String,
    pub jwt_expiration: u64,
    pub password_min_length: usize,
}

impl Settings {
    /// Load from environment and config file
    pub fn load() -> Result<Self> {
        let mut settings = config::Config::builder()
            .add_source(config::File::with_name("config/default"))
            .add_source(config::Environment::with_prefix("CMS").separator("__"))
            .build()?;

        Ok(settings.try_deserialize()?)
    }
}
```

---

## Authentication & Permissions

### JWT Authentication

**File:** `crates/cms-auth/src/jwt.rs`

```rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: Uuid,  // User ID
    pub email: String,
    pub roles: Vec<String>,
    pub exp: usize,  // Expiration
    pub iat: usize,  // Issued at
}

pub struct JwtService {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    expiration: u64,
}

impl JwtService {
    pub fn new(secret: &str, expiration: u64) -> Self {
        Self {
            encoding_key: EncodingKey::from_secret(secret.as_bytes()),
            decoding_key: DecodingKey::from_secret(secret.as_bytes()),
            expiration,
        }
    }

    pub fn generate_token(&self, user_id: Uuid, email: String, roles: Vec<String>) -> Result<String> {
        let now = chrono::Utc::now().timestamp() as usize;

        let claims = Claims {
            sub: user_id,
            email,
            roles,
            exp: now + self.expiration as usize,
            iat: now,
        };

        let token = encode(&Header::default(), &claims, &self.encoding_key)?;
        Ok(token)
    }

    pub fn validate_token(&self, token: &str) -> Result<Claims> {
        let token_data = decode::<Claims>(
            token,
            &self.decoding_key,
            &Validation::default(),
        )?;

        Ok(token_data.claims)
    }
}
```

### RBAC Middleware

**File:** `crates/cms-auth/src/middleware.rs`

```rust
use axum::{
    http::Request,
    middleware::Next,
    response::Response,
    extract::State,
};

pub async fn auth_middleware<B>(
    State(jwt): State<Arc<JwtService>>,
    mut req: Request<B>,
    next: Next<B>,
) -> Result<Response, StatusCode> {
    // Extract token from Authorization header
    let token = req.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // Validate token
    let claims = jwt.validate_token(token)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Insert claims into request extensions
    req.extensions_mut().insert(claims);

    Ok(next.run(req).await)
}

pub async fn require_permission(
    permission: &str,
) -> impl Fn(Request<B>, Next<B>) -> impl Future<Output = Result<Response, StatusCode>> {
    move |req, next| async move {
        let claims = req.extensions()
            .get::<Claims>()
            .ok_or(StatusCode::UNAUTHORIZED)?;

        // Check if user has required permission
        if !has_permission(&claims.roles, permission).await {
            return Err(StatusCode::FORBIDDEN);
        }

        Ok(next.run(req).await)
    }
}
```

---

## Complete Implementation Examples

### Example 1: Blog CMS

**Content Types:**

```rust
// Article content type
#[derive(ContentType, Serialize, Deserialize, JsonSchema)]
#[content_type(
    kind = "collection",
    display_name = "Article",
    singular = "article",
    plural = "articles"
)]
pub struct Article {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub published: bool,
    pub published_at: Option<DateTime<Utc>>,
    pub author: Relation<User>,
    pub categories: Vec<Relation<Category>>,
    pub tags: Vec<Relation<Tag>>,
    pub featured_image: Option<Relation<Media>>,
    pub seo: Component<SeoMetadata>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Category content type
#[derive(ContentType, Serialize, Deserialize, JsonSchema)]
pub struct Category {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub parent: Option<Relation<Category>>,
    pub articles: Vec<Relation<Article>>,
    pub created_at: DateTime<Utc>,
}
```

**Main Application:**

```rust
// crates/cms-server/src/main.rs

use cms_core::App;
use cms_db::Database;
use cms_api::ApiRouter;
use cms_plugins::PluginRegistry;
use cms_settings::Settings;

#[tokio::main]
async fn main() -> Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt::init();

    // Load settings
    let settings = Settings::load()?;

    // Connect to database
    let db = Database::connect(&settings.database.url).await?;

    // Run migrations
    run_migrations(&db).await?;

    // Create app
    let mut app = App::new(db, settings);

    // Register content types
    app.register_content_type::<Article>();
    app.register_content_type::<Category>();
    app.register_content_type::<User>();

    // Initialize plugins
    let mut plugins = PluginRegistry::new();
    plugins.register(Arc::new(S3UploadPlugin::new(s3_config)));
    plugins.register(Arc::new(GraphQLPlugin::new()));

    // Run plugin lifecycle
    plugins.run_lifecycle(PluginLifecycle::Register, &mut app).await?;
    plugins.run_lifecycle(PluginLifecycle::Bootstrap, &mut app).await?;

    // Build API router
    let router = app.build_router()
        .layer(middleware_stack());

    // Start server
    let addr = SocketAddr::from(([0, 0, 0, 0], settings.server.port));
    tracing::info!("Server listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(router.into_make_service())
        .await?;

    Ok(())
}
```

### Example 2: API Usage

**Creating an article:**

```bash
curl -X POST http://localhost:3000/api/articles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "data": {
      "title": "Getting Started with Rust",
      "slug": "getting-started-rust",
      "content": "Rust is a systems programming language...",
      "published": true,
      "author": "uuid-here",
      "categories": ["uuid1", "uuid2"],
      "tags": ["rust", "programming"]
    }
  }'
```

**Querying with filters:**

```bash
curl -X GET 'http://localhost:3000/api/articles?filters[$and][0][published][$eq]=true&filters[$and][1][author][name][$containsi]=john&populate[author]=true&populate[categories]=true&sort[0]=published_at:desc&pagination[page]=1&pagination[pageSize]=10'
```

**GraphQL query:**

```graphql
query {
  articles(
    filters: { published: { eq: true } }
    sort: ["published_at:desc"]
    pagination: { page: 1, pageSize: 10 }
  ) {
    data {
      id
      attributes {
        title
        slug
        excerpt
        published_at
        author {
          data {
            attributes {
              name
              email
            }
          }
        }
        categories {
          data {
            attributes {
              name
              slug
            }
          }
        }
      }
    }
    meta {
      pagination {
        page
        pageSize
        pageCount
        total
      }
    }
  }
}
```

---

## Conclusion

This guide provides a comprehensive architecture for building a production-ready CMS in Rust, combining:

### Key Takeaways

1. **Type Safety:** Leverage Rust's type system for compile-time guarantees
2. **Async Performance:** Use Tokio + Axum for high-performance async I/O
3. **Database Safety:** SQLx provides compile-time SQL validation
4. **Extensibility:** Plugin system allows customization without core changes
5. **Developer Experience:** Procedural macros reduce boilerplate
6. **API Flexibility:** Auto-generated REST + optional GraphQL
7. **Production Ready:** Comprehensive error handling, logging, auth

### Next Steps

1. Implement media upload and processing
2. Add GraphQL layer with async-graphql
3. Implement full-text search with Meilisearch/Elasticsearch
4. Add caching layer with Redis
5. Implement webhooks for content events
6. Add admin panel UI (e.g., with Leptos/Dioxus)
7. Implement content versioning
8. Add multi-tenancy support

### Critical Implementation Files

| Component | File Path |
|-----------|-----------|
| Database Connection | `crates/cms-db/src/connection.rs` |
| Query Builder | `crates/cms-db/src/query_builder.rs` |
| Migrations | `crates/cms-db/src/migrations.rs` |
| Content Type Schema | `crates/cms-schema/src/content_type.rs` |
| Entity Service | `crates/cms-services/src/entity_service.rs` |
| API Router | `crates/cms-api/src/router.rs` |
| Plugin System | `crates/cms-plugins/src/plugin.rs` |
| Auth Middleware | `crates/cms-auth/src/middleware.rs` |

This architecture provides a solid foundation for building a scalable, maintainable, and performant CMS in Rust.

---

**End of Implementation Guide**
