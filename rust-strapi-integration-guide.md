# Rust API + Strapi Schema Integration Guide

**The Challenge:** Strapi generates JSON schemas and TypeScript types automatically. How does Rust know about these types?

**The Solution:** Use Strapi's schema as the single source of truth and auto-generate Rust types from it.

---

## Table of Contents

1. [Integration Architecture](#integration-architecture)
2. [Schema as Contract](#schema-as-contract)
3. [Auto-Generating Rust Types](#auto-generating-rust-types)
4. [Build Pipeline](#build-pipeline)
5. [Runtime Schema Validation](#runtime-schema-validation)
6. [Complete Example](#complete-example)
7. [Alternative Approaches](#alternative-approaches)

---

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Strapi (Node.js)                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Content Type Definitions (Source of Truth)            │ │
│  │  - articles.json                                       │ │
│  │  - products.json                                       │ │
│  │  - categories.json                                     │ │
│  └───────────────────────┬────────────────────────────────┘ │
│                          │                                   │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Strapi Schema Generator                               │ │
│  │  - Generates TypeScript types                          │ │
│  │  - Generates JSON Schema (OpenAPI)                     │ │
│  │  - Generates database migrations                       │ │
│  └───────────────────────┬────────────────────────────────┘ │
└────────────────────────────┼──────────────────────────────────┘
                             │
                             │ Export
                             ▼
                  ┌──────────────────────┐
                  │   schemas/           │
                  │   ├── article.json   │ ◄─── Shared Schema
                  │   ├── product.json   │      (JSON Schema format)
                  │   └── category.json  │
                  └──────────┬───────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
    ┌─────────────────────┐   ┌─────────────────────┐
    │ TypeScript Types    │   │  Rust Code Gen      │
    │ (Strapi generates)  │   │  (We generate)      │
    └─────────────────────┘   └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │  Rust API Layer     │
                              │  ┌───────────────┐  │
                              │  │ models.rs     │  │
                              │  │ (generated)   │  │
                              │  └───────────────┘  │
                              └─────────────────────┘
```

---

## Schema as Contract

### Step 1: Strapi Content Type Definition

**File:** `strapi/src/api/article/content-types/article/schema.json`

```json
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": {
    "singularName": "article",
    "pluralName": "articles",
    "displayName": "Article"
  },
  "options": {
    "draftAndPublish": true
  },
  "attributes": {
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "slug": {
      "type": "uid",
      "targetField": "title",
      "required": true
    },
    "content": {
      "type": "richtext"
    },
    "excerpt": {
      "type": "text",
      "maxLength": 500
    },
    "published_at": {
      "type": "datetime"
    },
    "author": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "plugin::users-permissions.user"
    },
    "categories": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::category.category",
      "inversedBy": "articles"
    },
    "cover": {
      "type": "media",
      "multiple": false,
      "allowedTypes": ["images"]
    },
    "seo": {
      "type": "component",
      "component": "shared.seo",
      "required": false
    }
  }
}
```

### Step 2: Export to Standard JSON Schema

**Strapi Plugin:** Export schemas to standard JSON Schema format

**File:** `strapi/src/plugins/schema-export/server/index.js`

```javascript
module.exports = {
  async bootstrap({ strapi }) {
    const fs = require('fs');
    const path = require('path');

    // Export all content type schemas
    const contentTypes = Object.keys(strapi.contentTypes).filter(
      (uid) => uid.startsWith('api::')
    );

    const schemasDir = path.join(__dirname, '../../../../shared/schemas');
    if (!fs.existsSync(schemasDir)) {
      fs.mkdirSync(schemasDir, { recursive: true });
    }

    for (const uid of contentTypes) {
      const contentType = strapi.contentTypes[uid];
      const jsonSchema = convertToJsonSchema(contentType);

      const filename = path.join(
        schemasDir,
        `${contentType.info.singularName}.json`
      );

      fs.writeFileSync(filename, JSON.stringify(jsonSchema, null, 2));
      console.log(`Exported schema: ${filename}`);
    }
  },
};

function convertToJsonSchema(contentType) {
  const schema = {
    $schema: 'http://json-schema.org/draft-07/schema#',
    title: contentType.info.displayName,
    type: 'object',
    properties: {
      id: { type: 'string', format: 'uuid' },
    },
    required: ['id'],
  };

  // Convert Strapi attributes to JSON Schema
  for (const [attrName, attr] of Object.entries(contentType.attributes)) {
    const property = convertAttribute(attr);
    if (property) {
      schema.properties[attrName] = property;
      if (attr.required) {
        schema.required.push(attrName);
      }
    }
  }

  // Add timestamps
  schema.properties.createdAt = { type: 'string', format: 'date-time' };
  schema.properties.updatedAt = { type: 'string', format: 'date-time' };

  return schema;
}

function convertAttribute(attr) {
  switch (attr.type) {
    case 'string':
    case 'text':
    case 'richtext':
    case 'email':
    case 'uid':
      return {
        type: 'string',
        ...(attr.maxLength && { maxLength: attr.maxLength }),
        ...(attr.minLength && { minLength: attr.minLength }),
      };

    case 'integer':
    case 'biginteger':
      return { type: 'integer' };

    case 'float':
    case 'decimal':
      return { type: 'number' };

    case 'boolean':
      return { type: 'boolean' };

    case 'date':
    case 'datetime':
    case 'time':
      return { type: 'string', format: 'date-time' };

    case 'json':
      return { type: 'object' };

    case 'enumeration':
      return { type: 'string', enum: attr.enum };

    case 'relation':
      // Relations can be expanded or just IDs
      return {
        oneOf: [
          { type: 'string', format: 'uuid' }, // Just ID
          { $ref: `#/definitions/${attr.target.split('.').pop()}` }, // Expanded
        ],
      };

    case 'media':
      return attr.multiple
        ? { type: 'array', items: { $ref: '#/definitions/Media' } }
        : { $ref: '#/definitions/Media' };

    case 'component':
      return attr.repeatable
        ? { type: 'array', items: { $ref: `#/definitions/${attr.component}` } }
        : { $ref: `#/definitions/${attr.component}` };

    default:
      return null;
  }
}
```

**Generated JSON Schema:** `shared/schemas/article.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Article",
  "type": "object",
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "title": { "type": "string", "maxLength": 255 },
    "slug": { "type": "string" },
    "content": { "type": "string" },
    "excerpt": { "type": "string", "maxLength": 500 },
    "published_at": { "type": "string", "format": "date-time" },
    "author": {
      "oneOf": [
        { "type": "string", "format": "uuid" },
        { "$ref": "#/definitions/User" }
      ]
    },
    "categories": {
      "type": "array",
      "items": {
        "oneOf": [
          { "type": "string", "format": "uuid" },
          { "$ref": "#/definitions/Category" }
        ]
      }
    },
    "cover": { "$ref": "#/definitions/Media" },
    "seo": { "$ref": "#/definitions/Seo" },
    "createdAt": { "type": "string", "format": "date-time" },
    "updatedAt": { "type": "string", "format": "date-time" }
  },
  "required": ["id", "title", "slug"]
}
```

---

## Auto-Generating Rust Types

### Option 1: Using `schemafy` (Recommended)

**Cargo.toml:**

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1.0", features = ["serde", "v4"] }

[build-dependencies]
schemafy_lib = "0.6"
```

**build.rs:**

```rust
use std::path::Path;

fn main() {
    // Generate Rust types from JSON Schema
    let schema_dir = Path::new("../shared/schemas");
    let out_dir = std::env::var("OUT_DIR").unwrap();

    // Generate for each schema file
    for entry in std::fs::read_dir(schema_dir).unwrap() {
        let entry = entry.unwrap();
        let path = entry.path();

        if path.extension().and_then(|s| s.to_str()) == Some("json") {
            let schema = std::fs::read_to_string(&path).unwrap();

            let code = schemafy_lib::Generator::builder()
                .with_input_json(&schema)
                .build()
                .generate()
                .unwrap();

            let stem = path.file_stem().unwrap().to_str().unwrap();
            let out_path = Path::new(&out_dir).join(format!("{}.rs", stem));
            std::fs::write(out_path, code).unwrap();
        }
    }

    println!("cargo:rerun-if-changed=../shared/schemas");
}
```

**src/models/mod.rs:**

```rust
// Include generated types
include!(concat!(env!("OUT_DIR"), "/article.rs"));
include!(concat!(env!("OUT_DIR"), "/category.rs"));
include!(concat!(env!("OUT_DIR"), "/product.rs"));

// Re-export with better names
pub use article::Article;
pub use category::Category;
pub use product::Product;
```

**Generated Rust code:**

```rust
// This is auto-generated from article.json
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Article {
    pub id: uuid::Uuid,
    pub title: String,
    pub slug: String,
    pub content: Option<String>,
    pub excerpt: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub published_at: Option<chrono::DateTime<chrono::Utc>>,
    pub author: AuthorRelation,
    pub categories: Vec<CategoryRelation>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub cover: Option<Media>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub seo: Option<Seo>,
    #[serde(rename = "createdAt")]
    pub created_at: chrono::DateTime<chrono::Utc>,
    #[serde(rename = "updatedAt")]
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum AuthorRelation {
    Id(uuid::Uuid),
    Expanded(Box<User>),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum CategoryRelation {
    Id(uuid::Uuid),
    Expanded(Box<Category>),
}
```

### Option 2: Using `typify` (More flexible)

**build.rs:**

```rust
use typify::{TypeSpace, TypeSpaceSettings};

fn main() {
    let schema_content = std::fs::read_to_string("../shared/schemas/article.json").unwrap();

    let mut type_space = TypeSpace::new(TypeSpaceSettings::default());
    type_space
        .add_root_schema(serde_json::from_str(&schema_content).unwrap())
        .unwrap();

    let contents = format!(
        "use serde::{{Deserialize, Serialize}};\n\n{}",
        type_space.to_stream().to_string()
    );

    let out_path = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap())
        .join("article.rs");
    std::fs::write(out_path, contents).unwrap();
}
```

### Option 3: Manual Mapping with Validation

Keep Rust types manually defined but validate against schema at runtime:

**src/models/article.rs:**

```rust
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow, Validate)]
pub struct Article {
    pub id: uuid::Uuid,

    #[validate(length(max = 255))]
    pub title: String,

    pub slug: String,

    pub content: Option<String>,

    #[validate(length(max = 500))]
    pub excerpt: Option<String>,

    pub published_at: Option<chrono::DateTime<chrono::Utc>>,

    // Store as JSON in database, populated on demand
    #[sqlx(default)]
    pub author: Option<serde_json::Value>,

    #[sqlx(default)]
    pub categories: Option<serde_json::Value>,

    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

impl Article {
    /// Validate against JSON Schema at runtime
    pub fn validate_schema(&self) -> Result<(), jsonschema::ValidationError> {
        let schema = include_str!("../../shared/schemas/article.json");
        let schema: serde_json::Value = serde_json::from_str(schema)?;
        let compiled = jsonschema::JSONSchema::compile(&schema)?;

        let instance = serde_json::to_value(self)?;
        compiled.validate(&instance)?;

        Ok(())
    }
}
```

---

## Build Pipeline

### Automatic Sync on Schema Changes

**File:** `Makefile`

```makefile
.PHONY: sync-schemas build-rust

# Run this when Strapi schemas change
sync-schemas:
	@echo "Exporting Strapi schemas..."
	cd strapi && npm run export-schemas
	@echo "Regenerating Rust types..."
	cd rust-api && cargo build

# Watch for schema changes
watch-schemas:
	watchexec -w shared/schemas "make sync-schemas"

# Full rebuild
build-rust: sync-schemas
	cd rust-api && cargo build --release
```

**File:** `strapi/package.json`

```json
{
  "scripts": {
    "export-schemas": "node scripts/export-schemas.js",
    "postinstall": "npm run export-schemas"
  }
}
```

### CI/CD Integration

**File:** `.github/workflows/build.yml`

```yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Strapi dependencies
        run: cd strapi && npm install

      - name: Export Strapi schemas
        run: cd strapi && npm run export-schemas

      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable

      - name: Build Rust API (generates types from schemas)
        run: cd rust-api && cargo build --release

      - name: Run tests
        run: cd rust-api && cargo test
```

---

## Runtime Schema Validation

### Validate Responses Match Schema

**src/middleware/schema_validator.rs:**

```rust
use axum::{
    body::Body,
    http::{Request, Response},
    middleware::Next,
};
use jsonschema::JSONSchema;
use serde_json::Value;
use std::sync::Arc;

pub struct SchemaValidator {
    schemas: Arc<HashMap<String, JSONSchema>>,
}

impl SchemaValidator {
    pub fn new() -> Self {
        let mut schemas = HashMap::new();

        // Load all schemas at startup
        let article_schema = include_str!("../../shared/schemas/article.json");
        let article_schema: Value = serde_json::from_str(article_schema).unwrap();
        schemas.insert(
            "article".to_string(),
            JSONSchema::compile(&article_schema).unwrap(),
        );

        // Load other schemas...

        Self {
            schemas: Arc::new(schemas),
        }
    }

    pub fn validate(&self, schema_name: &str, data: &Value) -> Result<(), String> {
        let schema = self.schemas.get(schema_name)
            .ok_or_else(|| format!("Schema {} not found", schema_name))?;

        schema.validate(data)
            .map_err(|errors| {
                errors
                    .map(|e| format!("{}", e))
                    .collect::<Vec<_>>()
                    .join(", ")
            })
    }
}

// Middleware to validate responses in development
pub async fn validate_response_middleware(
    req: Request<Body>,
    next: Next,
) -> Result<Response<Body>, StatusCode> {
    let response = next.run(req).await;

    #[cfg(debug_assertions)]
    {
        // Only validate in development
        if let Ok(body_bytes) = response.body().to_bytes().await {
            if let Ok(json) = serde_json::from_slice::<Value>(&body_bytes) {
                // Detect schema from path
                let path = response.uri().path();
                if let Some(schema_name) = extract_schema_name(path) {
                    let validator = SchemaValidator::new();
                    if let Err(e) = validator.validate(&schema_name, &json) {
                        tracing::warn!("Response validation failed: {}", e);
                    }
                }
            }
        }
    }

    Ok(response)
}
```

---

## Complete Example

### 1. Strapi Schema Definition

**strapi/src/api/article/content-types/article/schema.json** (already shown above)

### 2. Export Script

**strapi/scripts/export-schemas.js:**

```javascript
const fs = require('fs');
const path = require('path');

// Load Strapi programmatically
const Strapi = require('@strapi/strapi');

async function exportSchemas() {
  const strapi = await Strapi().load();

  const schemasDir = path.join(__dirname, '../../shared/schemas');
  fs.mkdirSync(schemasDir, { recursive: true });

  const contentTypes = Object.keys(strapi.contentTypes).filter(
    (uid) => uid.startsWith('api::')
  );

  for (const uid of contentTypes) {
    const contentType = strapi.contentTypes[uid];
    const jsonSchema = convertToJsonSchema(contentType);

    const filename = path.join(
      schemasDir,
      `${contentType.info.singularName}.json`
    );

    fs.writeFileSync(filename, JSON.stringify(jsonSchema, null, 2));
    console.log(`✅ Exported: ${contentType.info.displayName}`);
  }

  await strapi.destroy();
}

function convertToJsonSchema(contentType) {
  // Implementation from above
  // ...
}

exportSchemas().catch(console.error);
```

### 3. Rust Build Script

**rust-api/build.rs:**

```rust
use std::path::{Path, PathBuf};
use std::fs;

fn main() {
    println!("cargo:rerun-if-changed=../shared/schemas");

    let schema_dir = PathBuf::from("../shared/schemas");
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    if !schema_dir.exists() {
        panic!("Schema directory not found. Run 'npm run export-schemas' in Strapi first.");
    }

    generate_types_from_schemas(&schema_dir, &out_dir);
}

fn generate_types_from_schemas(schema_dir: &Path, out_dir: &Path) {
    for entry in fs::read_dir(schema_dir).unwrap() {
        let entry = entry.unwrap();
        let path = entry.path();

        if path.extension().and_then(|s| s.to_str()) != Some("json") {
            continue;
        }

        let schema_content = fs::read_to_string(&path).unwrap();

        // Use schemafy or typify to generate Rust code
        let rust_code = generate_rust_from_json_schema(&schema_content);

        let stem = path.file_stem().unwrap().to_str().unwrap();
        let out_path = out_dir.join(format!("{}.rs", stem));
        fs::write(out_path, rust_code).unwrap();

        println!("Generated Rust types for: {}", stem);
    }
}

fn generate_rust_from_json_schema(schema: &str) -> String {
    // Use schemafy_lib or typify here
    // For simplicity, shown as placeholder
    schemafy_lib::Generator::builder()
        .with_input_json(schema)
        .build()
        .generate()
        .unwrap()
}
```

### 4. Using Generated Types

**rust-api/src/handlers/articles.rs:**

```rust
use axum::{extract::State, Json};
use crate::models::Article; // Auto-generated from schema
use crate::AppState;

pub async fn get_articles(
    State(state): State<AppState>,
) -> Result<Json<Vec<Article>>, StatusCode> {
    // Query database - Article type matches Strapi exactly
    let articles = sqlx::query_as::<_, Article>(
        "SELECT * FROM articles WHERE published = true ORDER BY published_at DESC"
    )
    .fetch_all(&state.db)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // Serialize to JSON - guaranteed to match Strapi's schema
    Ok(Json(articles))
}
```

---

## Alternative Approaches

### Approach 1: OpenAPI as Source of Truth

Use Strapi's OpenAPI plugin:

```bash
# Install Strapi OpenAPI plugin
npm install @strapi/plugin-documentation
```

**Generate Rust from OpenAPI:**

```toml
[build-dependencies]
openapi-generator = "0.1"
```

```rust
// build.rs
fn main() {
    openapi_generator::generate(
        "../strapi/src/extensions/documentation/documentation/1.0.0/full_documentation.json",
        "src/generated",
    );
}
```

### Approach 2: Database as Source of Truth

Use PostgreSQL to introspect schemas:

```rust
// Query PostgreSQL information_schema
let columns = sqlx::query!(
    r#"
    SELECT column_name, data_type, is_nullable
    FROM information_schema.columns
    WHERE table_name = 'articles'
    "#
)
.fetch_all(&pool)
.await?;

// Generate Rust types from database schema
```

### Approach 3: Shared Proto/gRPC (Advanced)

Define schemas in Protocol Buffers:

```protobuf
// shared/protos/article.proto
syntax = "proto3";

message Article {
  string id = 1;
  string title = 2;
  string slug = 3;
  optional string content = 4;
  // ...
}
```

Generate for both:
- Node.js: `protoc --js_out=. article.proto`
- Rust: `protoc --rust_out=. article.proto`

---

## Best Practices

### 1. Version Your Schemas

```
shared/schemas/
├── v1/
│   ├── article.json
│   └── category.json
└── v2/
    ├── article.json
    └── category.json
```

### 2. Schema Validation in Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_article_matches_schema() {
        let article = Article {
            id: uuid::Uuid::new_v4(),
            title: "Test".to_string(),
            // ...
        };

        let validator = SchemaValidator::new();
        let json = serde_json::to_value(&article).unwrap();

        assert!(validator.validate("article", &json).is_ok());
    }
}
```

### 3. Schema Compatibility Checks

```rust
// Ensure Rust types are compatible with schema
#[test]
fn schema_compatibility() {
    // Load current schema
    let schema = load_schema("article");

    // Load previous schema
    let old_schema = load_schema_version("article", "1.0.0");

    // Check backward compatibility
    assert!(is_backward_compatible(&old_schema, &schema));
}
```

---

## Summary

### The Flow

```
1. Developer defines content type in Strapi
   ↓
2. Strapi generates JSON Schema
   ↓
3. Export to shared/schemas/
   ↓
4. Rust build.rs reads schemas
   ↓
5. Auto-generate Rust types
   ↓
6. Use types in Rust API
   ↓
7. Response matches Strapi's structure perfectly
```

### Key Advantages

✅ **Single Source of Truth:** Strapi defines schemas
✅ **Type Safety:** Rust types auto-generated
✅ **Always in Sync:** Build fails if schemas incompatible
✅ **Zero Manual Work:** Automatic code generation
✅ **Schema Validation:** Runtime checks in development

### Recommended Setup

1. Use **Strapi** to define content types
2. Export to **JSON Schema** format
3. Use **schemafy** or **typify** to generate Rust types
4. Add **build.rs** for automatic regeneration
5. Enable **schema validation** in development
6. Add **CI checks** to ensure compatibility

This way, Rust API stays perfectly synchronized with Strapi's schema definitions!
