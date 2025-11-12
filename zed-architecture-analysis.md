# Zed Application - Comprehensive Rust Architecture Analysis

**Date:** 2025-11-12
**Repository:** https://github.com/zed-industries/zed
**Purpose:** Analysis of advanced Rust patterns, macros, and design principles for CMS development

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Workspace Structure](#workspace-structure)
3. [Procedural Macros & Code Generation](#procedural-macros--code-generation)
4. [Entity System & State Management](#entity-system--state-management)
5. [Database Patterns (sqlez)](#database-patterns-sqlez)
6. [Settings & Configuration System](#settings--configuration-system)
7. [Async & Concurrency Patterns](#async--concurrency-patterns)
8. [Error Handling Patterns](#error-handling-patterns)
9. [Builder & Fluent API Patterns](#builder--fluent-api-patterns)
10. [Trait-Based Abstractions](#trait-based-abstractions)
11. [Key Architectural Patterns](#key-architectural-patterns)
12. [Design Principles & Best Practices](#design-principles--best-practices)

---

## Executive Summary

Zed is a high-performance code editor built entirely in Rust, featuring over **200 workspace crates** organized in a monorepo structure. The application demonstrates advanced Rust patterns suitable for building complex, scalable applications like a CMS.

### Key Statistics
- **200+** workspace crates
- **10+** procedural macro crates
- **Custom UI Framework:** GPUI (GPU-accelerated UI)
- **Database Layer:** sqlez (type-safe SQLite wrapper)
- **Edition:** Rust 2024

### Core Architectural Innovations

1. **Entity-Component System** - Type-safe state management with automatic lifecycle management
2. **Compile-Time SQL Validation** - SQL syntax errors caught at compile time
3. **Layered Settings System** - Hierarchical configuration with hot-reloading
4. **Async-First Design** - Foreground/background executor separation
5. **Zero-Cost Abstractions** - Heavy use of traits and generics for performance

---

## Workspace Structure

### Monorepo Organization

```
zed/
├── Cargo.toml                    # Workspace root
├── crates/
│   ├── gpui/                     # Core UI framework
│   ├── gpui_macros/              # UI derive macros
│   ├── sqlez/                    # Database layer
│   ├── sqlez_macros/             # SQL validation macros
│   ├── settings/                 # Configuration system
│   ├── settings_macros/          # Settings derive macros
│   ├── db/                       # High-level database API
│   ├── util/                     # Common utilities
│   ├── collections/              # Custom collections
│   ├── rpc/                      # RPC protocol
│   ├── project/                  # Project management
│   ├── workspace/                # Workspace logic
│   └── [190+ more crates]
├── extensions/                   # Extension API
└── tooling/
    └── xtask/                    # Build tooling
```

### Workspace Configuration

**File:** `Cargo.toml`

```toml
[workspace]
resolver = "2"
members = [
    "crates/gpui",
    "crates/sqlez",
    "crates/settings",
    # ... 200+ crates
]

[workspace.package]
edition = "2024"
publish = false

[workspace.dependencies]
# Internal crates with versioning
gpui = { path = "crates/gpui" }
sqlez = { path = "crates/sqlez" }
settings = { path = "crates/settings" }

# External dependencies (shared versions)
anyhow = "1.0.86"
serde = { version = "1.0.221", features = ["derive", "rc"] }
tokio = { version = "1" }
```

### Crate Categorization

| Category | Examples | Purpose |
|----------|----------|---------|
| **Core Framework** | `gpui`, `gpui_macros` | UI framework and macros |
| **Database** | `sqlez`, `sqlez_macros`, `db` | Type-safe database access |
| **Settings** | `settings`, `settings_macros` | Configuration management |
| **Utilities** | `util`, `collections`, `clock` | Common functionality |
| **UI Components** | `ui`, `ui_macros`, `ui_input` | Reusable UI elements |
| **Language** | `language`, `lsp`, `tree-sitter-*` | Code intelligence |
| **Editor** | `editor`, `multi_buffer`, `text` | Core editor logic |
| **Extensions** | `extension`, `extension_api` | Plugin system |

---

## Procedural Macros & Code Generation

### 1. GPUI Macros (`gpui_macros`)

**File:** `crates/gpui_macros/src/gpui_macros.rs`

#### Action Derive Macro

Automatically implements the `Action` trait with JSON schema support and metadata:

```rust
#[derive(Clone, PartialEq, Action)]
#[action(namespace = "workspace")]
pub struct OpenFile {
    pub path: PathBuf,
}

// Generated implementation:
impl gpui::Action for OpenFile {
    fn name(&self) -> &'static str {
        "workspace::OpenFile"
    }

    fn build(value: serde_json::Value) -> Result<Box<dyn Action>> {
        Ok(Box::new(serde_json::from_value::<Self>(value)?))
    }

    fn action_json_schema(
        generator: &mut schemars::SchemaGenerator,
    ) -> Option<Schema> {
        Some(<Self as JsonSchema>::json_schema(generator))
    }

    // ... more methods
}
```

**Features:**
- Namespace support for action organization
- Deprecated aliases for migration
- Documentation extraction from doc comments
- Optional JSON schema generation
- Automatic registration

#### IntoElement Derive

Converts `RenderOnce` types into UI elements:

```rust
#[derive(IntoElement)]
pub struct MyButton {
    label: String,
}

impl RenderOnce for MyButton {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl Element {
        div()
            .child(self.label)
            .on_click(|_, _, _| { /* ... */ })
    }
}

// Now can be used as:
// div().child(MyButton { label: "Click me".into() })
```

#### Test Macro

Enhanced testing with deterministic randomness and retries:

```rust
#[gpui::test]
async fn test_my_feature(mut cx: &TestAppContext) {
    // Test code with GPUI context
}

// With iterations and seeds:
#[gpui::test(iterations = 10, retries = 3)]
async fn test_with_randomness(mut cx: &TestAppContext, mut rng: StdRng) {
    // Deterministic random testing
}
```

**Features:**
- Automatic `TestAppContext` injection
- Deterministic RNG with seed control
- Iteration support for property testing
- Retry support for flaky tests
- Environment variable configuration (`SEED`, `ITERATIONS`)

### 2. SQL Validation Macro (`sqlez_macros`)

**File:** `crates/sqlez_macros/src/sqlez_macros.rs`

Provides **compile-time SQL syntax validation**:

```rust
const CREATE_USERS: &str = sql!(
    CREATE TABLE users(
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT UNIQUE
    ) STRICT;
);

// Compile error example:
// const BAD_SQL: &str = sql!(
//     SELCT * FROM users  -- Typo!
// );
// ❌ Error: Sql Error: near "SELCT": syntax error
```

**Implementation:**

```rust
#[proc_macro]
pub fn sql(tokens: TokenStream) -> TokenStream {
    let (spans, sql) = make_sql(tokens);

    // Create in-memory SQLite connection at compile time
    #[cfg(not(any(target_os = "linux", target_os = "freebsd")))]
    let error = SQLITE.sql_has_syntax_error(sql.trim());

    // Format SQL for consistency
    let formatted = sqlformat::format(&sql, &QueryParams::None, Default::default());

    if let Some((error, error_offset)) = error {
        create_error(spans, error_offset, error, &formatted)
    } else {
        format!("r#\"{}\"#", &formatted).parse().unwrap()
    }
}
```

### 3. Settings Macro (`settings_macros`)

**File:** `crates/settings_macros/src/settings_macros.rs`

Derives the `MergeFrom` trait for settings composition:

```rust
#[derive(Clone, MergeFrom, Deserialize)]
pub struct EditorSettings {
    pub tab_size: Option<usize>,
    pub auto_save: Option<bool>,
    pub format_on_save: Option<bool>,
}

// Generated:
impl MergeFrom for EditorSettings {
    fn merge_from(&mut self, other: &Self) {
        self.tab_size.merge_from(&other.tab_size);
        self.auto_save.merge_from(&other.auto_save);
        self.format_on_save.merge_from(&other.format_on_save);
    }
}
```

---

## Entity System & State Management

### Entity Architecture

**File:** `crates/gpui/src/app/entity_map.rs`

Zed uses a **slot-map based entity system** for type-safe state management with automatic lifecycle tracking.

#### Entity Types

```rust
// Unique identifier for any entity
pub struct EntityId(KeyData);

// Strong handle to entity
pub struct Entity<T> {
    entity_id: EntityId,
    entity_map: Weak<RwLock<EntityRefCounts>>,
    entity_type: PhantomData<T>,
}

// Weak handle (prevents cycles)
pub struct WeakEntity<T> {
    entity_id: EntityId,
    entity_map: Weak<RwLock<EntityRefCounts>>,
    entity_type: PhantomData<T>,
}
```

#### Entity Storage

```rust
pub(crate) struct EntityMap {
    // Type-erased entity storage
    entities: SecondaryMap<EntityId, Box<dyn Any>>,

    // Track which entities were accessed
    accessed_entities: RefCell<FxHashSet<EntityId>>,

    // Reference counting
    ref_counts: Arc<RwLock<EntityRefCounts>>,
}
```

#### Creating Entities

```rust
// In application context
let entity: Entity<MyState> = cx.new(|| MyState::new());

// With entity context
impl MyState {
    fn init(cx: &mut Context<Self>) -> Self {
        let this = cx.entity(); // Get handle to self
        // ... setup
        Self { /* ... */ }
    }
}
```

#### Accessing Entities

```rust
// Read-only access
entity.read(cx).some_field;

entity.read_with(cx, |state, cx| {
    // state: &MyState
    // cx: &App
    state.do_something()
});

// Mutable access
entity.update(cx, |state, cx| {
    // state: &mut MyState
    // cx: &mut Context<MyState>
    state.field = new_value;
    cx.notify(); // Notify observers of change
});
```

#### Lease Pattern (Advanced)

For temporary exclusive access:

```rust
pub struct Lease<T> {
    entity: Option<Box<dyn Any>>,
    id: EntityId,
}

// Move entity to stack temporarily
let mut lease = entity_map.lease(&entity);
// Entity is now on the stack, not in map
lease.do_something_mut();
// Return to map
entity_map.end_lease(lease);
```

**Use case:** Prevents double-borrow panics when entity needs to update itself.

### Context Types

**File:** `crates/gpui/src/app/context.rs`

#### App Context

Global application context:

```rust
pub struct App {
    // Entity storage
    entities: EntityMap,

    // Global state
    globals: HashMap<TypeId, Box<dyn Any>>,

    // Event system
    observers: SubscriberSet<EntityId, ObserverCallback>,
    subscriptions: SubscriberSet<(EntityId, TypeId), SubscriptionCallback>,

    // Executors
    foreground_executor: ForegroundExecutor,
    background_executor: BackgroundExecutor,
}
```

**Global state:**

```rust
// Define global
struct ThemeRegistry(Vec<Theme>);

// Set global
cx.set_global(ThemeRegistry(vec![/* ... */]));

// Access global
let themes = cx.global::<ThemeRegistry>();
```

#### Entity Context

Specialized context for entity mutations:

```rust
pub struct Context<'a, T> {
    app: &'a mut App,
    entity_state: WeakEntity<T>,
}

impl<'a, T> Deref for Context<'a, T> {
    type Target = App;

    fn deref(&self) -> &Self::Target {
        self.app
    }
}

impl<'a, T: 'static> Context<'a, T> {
    // Get handle to current entity
    pub fn entity(&self) -> Entity<T> {
        self.entity_state.upgrade().expect("entity must be alive")
    }

    // Notify observers of changes
    pub fn notify(&mut self) {
        self.app.notify_observers(self.entity_id());
    }

    // Emit events
    pub fn emit<Evt>(&mut self, event: Evt)
    where
        T: EventEmitter<Evt>,
        Evt: 'static,
    {
        self.app.emit_event(self.entity_id(), event);
    }
}
```

### Observer Pattern

```rust
// Observe entity changes
cx.observe(&other_entity, |this: &mut MyState, observed, cx| {
    // Called when observed entity calls cx.notify()
    println!("Entity changed: {:?}", observed.read(cx));
});

// Subscribe to specific events
cx.subscribe(&other_entity, |this: &mut MyState, entity, event: &SomeEvent, cx| {
    // Called when entity emits SomeEvent
    this.handle_event(event);
});
```

**Lifecycle management:**

```rust
// Subscriptions are automatically cleaned up when entity is dropped
pub struct MyState {
    _subscriptions: Vec<Subscription>,
}

impl MyState {
    fn new(cx: &mut Context<Self>) -> Self {
        let sub1 = cx.observe(&other, |_, _, _| {});
        let sub2 = cx.subscribe(&another, |_, _, _, _| {});

        Self {
            _subscriptions: vec![sub1, sub2],
        }
    }
}
```

---

## Database Patterns (sqlez)

**Files:** `crates/sqlez/src/*`

### Architecture Overview

sqlez provides a **type-safe, thread-safe SQLite wrapper** with:
- Compile-time SQL validation
- Thread-local read connections
- Serialized write operations
- Automatic migration system
- Savepoint-based transactions

### 1. Type-Safe Queries

#### Bindable Types

**File:** `crates/sqlez/src/bindable.rs`

```rust
pub trait Bind {
    fn bind(&self, statement: &Statement, start_index: i32) -> Result<i32>;
}

pub trait Column: Sized {
    fn column(statement: &mut Statement, start_index: i32) -> Result<(Self, i32)>;
}
```

**Supported types:**
- Primitives: `i32`, `i64`, `u32`, `u64`, `f32`, `f64`, `bool`
- Text: `&str`, `String`, `Arc<str>`
- Binary: `&[u8]`, `Vec<u8>`
- Paths: `PathBuf`, `Arc<Path>`
- Other: `uuid::Uuid`, `Option<T>`
- **Tuples:** Up to 10 elements

**Example:**

```rust
// Insert with tuple binding
connection.exec_bound::<(String, i64, bool)>(
    "INSERT INTO items (name, count, active) VALUES (?, ?, ?)"
)?((
    "item1".to_string(),
    42,
    true,
))?;

// Select with tuple extraction
let items: Vec<(String, i64, bool)> = connection
    .select("SELECT name, count, active FROM items")?
    ()?;
```

### 2. Thread-Safe Connection

**File:** `crates/sqlez/src/thread_safe_connection.rs`

```rust
pub struct ThreadSafeConnection {
    uri: Arc<str>,
    connections: Arc<ThreadLocal<Connection>>,  // Thread-local reads
    write_queue: Arc<RwLock<WriteQueue>>,       // Serialized writes
}
```

**Architecture:**
- **Read operations:** Use thread-local connections (immediate, no locking)
- **Write operations:** Queued to background thread (serialized)

```rust
// Opens database with migrations
let db = ThreadSafeConnection::builder::<MyDomain>("db.sqlite", true)
    .with_db_initialization_query("PRAGMA journal_mode=WAL")
    .with_connection_initialize_query("PRAGMA foreign_keys=TRUE")
    .build()
    .await?;

// Read (thread-local, no queue)
let users = db.select::<String>("SELECT name FROM users")?()?;

// Write (queued to background thread)
db.write(|conn| {
    conn.exec_bound("INSERT INTO users (name) VALUES (?)")?("Alice")
}).await?;
```

### 3. Migration System

**File:** `crates/sqlez/src/domain.rs`

```rust
pub trait Domain: 'static {
    const NAME: &str;
    const MIGRATIONS: &[&str];

    fn should_allow_migration_change(_index: usize, _old: &str, _new: &str) -> bool {
        false  // Migrations are immutable by default
    }
}

// Example domain
pub struct KeyValueStore(ThreadSafeConnection);

impl Domain for KeyValueStore {
    const NAME: &str = "kv_store";

    const MIGRATIONS: &[&str] = &[
        sql!(
            CREATE TABLE IF NOT EXISTS kv_store(
                key TEXT PRIMARY KEY,
                value TEXT NOT NULL
            ) STRICT;
        ),
        sql!(
            ALTER TABLE kv_store ADD COLUMN created_at INTEGER;
        ),
    ];
}
```

**Migration execution:**

```rust
// Tracks migrations in database
// CREATE TABLE migrations (domain TEXT, step INTEGER, migration TEXT)

pub fn migrate(
    &self,
    domain: &'static str,
    migrations: &[&'static str],
) -> Result<()> {
    for (index, migration) in migrations.iter().enumerate() {
        if let Some(completed) = get_completed_migration(index) {
            if completed != migration {
                anyhow::bail!("Migration {index} changed for {domain}");
            }
            continue;
        }

        self.exec(migration)?;
        store_migration(domain, index, migration)?;
    }
    Ok(())
}
```

### 4. Transaction Support

**File:** `crates/sqlez/src/savepoint.rs`

Uses **SQLite savepoints** for nested transactions:

```rust
pub fn with_savepoint<R, F>(&self, name: &str, f: F) -> Result<R>
where
    F: FnOnce() -> Result<R>,
{
    self.exec(&format!("SAVEPOINT {name}"))?()?;

    let result = f();

    match result {
        Ok(_) => {
            self.exec(&format!("RELEASE {name}"))?()?;
        }
        Err(_) => {
            self.exec(&format!("ROLLBACK TO {name}; RELEASE {name}"))?()?;
        }
    }

    result
}
```

**Usage:**

```rust
conn.with_savepoint("transaction", || {
    conn.exec_bound("INSERT INTO users (name) VALUES (?)")?("Alice")?;

    // Nested savepoint
    conn.with_savepoint("nested", || {
        conn.exec_bound("UPDATE stats SET count = count + 1")?()?;
        Ok(())
    })?;

    Ok(())
})?;
```

### 5. High-Level Query Macro

**File:** `crates/db/src/query.rs`

```rust
query! {
    pub fn get_user(id: i64) -> Result<Option<String>> {
        SELECT name FROM users WHERE id = ?
    }
}

query! {
    pub async fn insert_user(name: String) -> Result<()> {
        INSERT INTO users (name) VALUES (?)
    }
}

// Usage:
let name = get_user(conn, 42)?;
insert_user(conn, "Bob".to_string()).await?;
```

---

## Settings & Configuration System

**Files:** `crates/settings/src/*`

### 1. Settings Architecture

#### Settings Content (Data Layer)

```rust
#[derive(Debug, Default, Clone, Serialize, Deserialize, JsonSchema, MergeFrom)]
#[serde(deny_unknown_fields)]
pub struct SettingsContent {
    #[serde(flatten)]
    pub editor: EditorSettingsContent,

    #[serde(flatten)]
    pub theme: Box<ThemeSettingsContent>,

    pub auto_update: Option<bool>,
    pub vim_mode: Option<bool>,
    // ... more fields
}
```

**Key characteristics:**
- All fields are `Option<T>` for partial configuration
- Uses `#[serde(flatten)]` to merge nested objects
- Implements `MergeFrom` for layered composition

#### Settings Trait (Access Layer)

```rust
pub trait Settings: 'static + Send + Sync + Sized {
    // Convert from merged settings content
    fn from_settings(content: &SettingsContent) -> Self;

    // Register globally
    fn register(cx: &mut App);

    // Access the setting value
    fn get<'a>(path: Option<SettingsLocation>, cx: &'a App) -> &'a Self;
}
```

**Example implementation:**

```rust
pub struct EditorSettings {
    pub tab_size: usize,
    pub auto_save: bool,
    pub format_on_save: bool,
}

impl Settings for EditorSettings {
    fn from_settings(content: &SettingsContent) -> Self {
        Self {
            tab_size: content.editor.tab_size.unwrap_or(4),
            auto_save: content.editor.auto_save.unwrap_or(false),
            format_on_save: content.editor.format_on_save.unwrap_or(false),
        }
    }
}

// Usage:
let settings = EditorSettings::get_global(cx);
println!("Tab size: {}", settings.tab_size);
```

### 2. MergeFrom Pattern

Enables hierarchical settings composition:

```rust
pub trait MergeFrom {
    fn merge_from(&mut self, other: &Self);
}

// For Option<T>: merge if Some
impl<T: Clone + MergeFrom> MergeFrom for Option<T> {
    fn merge_from(&mut self, other: &Self) {
        let Some(other) = other else { return };

        if let Some(this) = self {
            this.merge_from(other);  // Deep merge
        } else {
            self.replace(other.clone());  // Set value
        }
    }
}

// For HashMap: deep merge
impl<K, V> MergeFrom for HashMap<K, V>
where
    K: Clone + Hash + Eq,
    V: Clone + MergeFrom,
{
    fn merge_from(&mut self, other: &Self) {
        for (k, v) in other {
            if let Some(existing) = self.get_mut(k) {
                existing.merge_from(v);  // Merge existing
            } else {
                self.insert(k.clone(), v.clone());  // Insert new
            }
        }
    }
}
```

### 3. Settings Hierarchy

Settings are merged in order (later overrides earlier):

1. **Default settings** (`assets/settings/default.json`)
2. **Extension settings** (from installed extensions)
3. **Global settings** (server-provided)
4. **User settings** (`~/.config/zed/settings.json`)
   - Release channel overrides (dev/nightly/stable)
   - OS-specific overrides (macos/linux/windows)
   - Profile-specific overrides
5. **Server settings** (remote server)
6. **Local project settings** (`.zed/settings.json` in directories)

**Merge logic:**

```rust
pub fn recompute_values(&mut self, cx: &mut App) {
    // Merge global settings
    let mut merged = self.default_settings.clone();
    merged.merge_from_option(self.extension_settings.as_deref());
    merged.merge_from_option(self.global_settings.as_deref());

    if let Some(user) = self.user_settings.as_ref() {
        merged.merge_from(&user.content);
        merged.merge_from_option(user.for_release_channel());
        merged.merge_from_option(user.for_os());
        merged.merge_from_option(user.for_profile(cx));
    }

    self.merged_settings = Rc::new(merged);

    // Recompute all setting values
    for value in self.setting_values.values_mut() {
        let new_value = value.from_settings(&self.merged_settings);
        value.set_global_value(new_value);
    }
}
```

### 4. Location-Aware Settings

Settings can be overridden per directory:

```
project/.zed/settings.json          → Applies to entire project
project/src/.zed/settings.json      → Overrides for src/
```

```rust
pub struct SettingsLocation {
    pub worktree_id: WorktreeId,
    pub path: Arc<RelPath>,
}

// Get settings for specific file
let settings = EditorSettings::get(
    Some(SettingsLocation {
        worktree_id: WorktreeId::from_usize(1),
        path: rel_path("src/main.rs"),
    }),
    cx,
);
```

### 5. Hot-Reloading

**File:** `crates/settings/src/settings_store.rs`

```rust
pub fn watch_config_file(
    executor: &BackgroundExecutor,
    fs: Arc<dyn Fs>,
    path: PathBuf,
) -> mpsc::UnboundedReceiver<String> {
    let (tx, rx) = mpsc::unbounded();

    executor.spawn(async move {
        let (mut events, _) = fs.watch(&path, Duration::from_millis(100)).await;

        // Send initial contents
        if let Ok(contents) = fs.load(&path).await {
            let _ = tx.unbounded_send(contents);
        }

        // Watch for changes
        while events.next().await.is_some() {
            if let Ok(contents) = fs.load(&path).await {
                let _ = tx.unbounded_send(contents);
            }
        }
    }).detach();

    rx
}
```

### 6. JSON Schema Generation

```rust
pub fn json_schema(&self, params: &SettingsJsonSchemaParams) -> Value {
    let mut generator = schemars::generate::SchemaSettings::draft2019_09()
        .with_transform(DefaultDenyUnknownFields)
        .into_generator();

    // Generate base schema
    UserSettingsContent::json_schema(&mut generator);

    // Dynamic replacements for runtime-dependent schemas
    replace_subschema::<LanguageSettingsMap>(&mut generator, || {
        json_schema!({
            "type": "object",
            "properties": params.language_names
                .iter()
                .map(|name| (name.clone(), language_settings_ref.clone()))
                .collect::<Map<_, _>>(),
        })
    });

    generator.root_schema_for::<UserSettingsContent>().to_value()
}
```

---

## Async & Concurrency Patterns

**Files:** `crates/gpui/src/executor.rs`, `crates/gpui/src/app/async_context.rs`

### 1. Dual Executor Architecture

#### ForegroundExecutor

Runs tasks on the **main thread only**:

```rust
pub struct ForegroundExecutor {
    dispatcher: Arc<dyn PlatformDispatcher>,
    not_send: PhantomData<Rc<()>>,  // Intentionally !Send
}

impl ForegroundExecutor {
    pub fn spawn<R>(&self, future: impl Future<Output = R> + 'static) -> Task<R>
    where
        R: 'static,
    {
        // Note: future doesn't need to be Send
        // Task is polled only on main thread
    }
}
```

#### BackgroundExecutor

Runs tasks on **thread pool**:

```rust
pub struct BackgroundExecutor {
    dispatcher: Arc<dyn PlatformDispatcher>,
}

impl BackgroundExecutor {
    pub fn spawn<R>(&self, future: impl Future<Output = R> + Send + 'static) -> Task<R>
    where
        R: Send + 'static,
    {
        // Future must be Send - can be polled on any thread
    }
}
```

**Usage from context:**

```rust
// Foreground task (UI work)
cx.spawn(async move |cx| {
    // Can access UI state
    entity.update(&mut cx, |state, cx| {
        state.do_ui_work();
    })?;
    Ok(())
})

// Background task (CPU-intensive work)
cx.background_spawn(async move {
    // No UI access, must be Send
    expensive_computation().await
})
```

### 2. Task Lifecycle Management

```rust
#[must_use]
pub struct Task<T>(TaskState<T>);

enum TaskState<T> {
    Ready(Option<T>),           // Completed immediately
    Spawned(async_task::Task<T>), // Running
}

impl<T> Task<T> {
    // Create completed task
    pub fn ready(val: T) -> Self {
        Task(TaskState::Ready(Some(val)))
    }

    // Allow task to run without being awaited
    pub fn detach(self) {
        match self {
            Task(TaskState::Spawned(task)) => task.detach(),
            _ => {}
        }
    }
}

// Convenience for error handling
impl<E, T> Task<Result<T, E>>
where
    E: Debug + 'static,
{
    pub fn detach_and_log_err(self, cx: &App) {
        let location = core::panic::Location::caller();
        cx.foreground_executor()
            .spawn(self.log_tracked_err(*location))
            .detach();
    }
}
```

**Patterns:**

```rust
// 1. Automatic cancellation (task dropped)
{
    let _task = cx.spawn(async |cx| { /* work */ });
    // Task cancelled when _task goes out of scope
}

// 2. Detached task (fire-and-forget)
cx.spawn(async |cx| {
    background_work().await
}).detach();

// 3. Stored task (keep alive)
struct MyState {
    _background_task: Task<()>,  // Keeps task alive
}
```

### 3. Async Contexts

#### AsyncApp

```rust
#[derive(Clone)]
pub struct AsyncApp {
    app: Weak<AppCell>,  // Weak reference prevents cycles
    background_executor: BackgroundExecutor,
    foreground_executor: ForegroundExecutor,
}

impl AsyncApp {
    // All methods return Result (app might be dropped)
    pub fn update<R>(&self, f: impl FnOnce(&mut App) -> R) -> Result<R> {
        let app = self.app.upgrade().context("app was released")?;
        let mut lock = app.borrow_mut();
        Ok(lock.update(f))
    }

    pub fn spawn<AsyncFn, R>(&self, f: AsyncFn) -> Task<R>
    where
        AsyncFn: AsyncFnOnce(&mut AsyncApp) -> R + 'static,
        R: 'static,
    {
        let mut cx = self.clone();
        self.foreground_executor.spawn(async move { f(&mut cx).await })
    }
}
```

#### AsyncWindowContext

```rust
#[derive(Clone)]
pub struct AsyncWindowContext {
    app: AsyncApp,
    window: AnyWindowHandle,
}

impl AsyncWindowContext {
    pub fn update<R>(
        &mut self,
        f: impl FnOnce(&mut Window, &mut App) -> R
    ) -> Result<R> {
        self.app.update_window(self.window, |_, window, cx| f(window, cx))
    }
}
```

**Usage pattern:**

```rust
window.spawn(cx, async move |mut cx| {
    // Perform async work
    let data = fetch_data().await?;

    // Update UI on completion
    cx.update(|window, cx| {
        window.show_result(data, cx);
    })?;

    Ok(())
}).detach_and_log_err(cx);
```

### 4. Scoped Concurrency

```rust
pub async fn scoped<'scope, F>(&self, scheduler: F)
where
    F: FnOnce(&mut Scope<'scope>),
{
    let mut scope = Scope::new(self.clone());
    scheduler(&mut scope);

    // Spawn all tasks
    let tasks = scope.futures
        .into_iter()
        .map(|f| self.spawn(f))
        .collect::<Vec<_>>();

    // Wait for all to complete
    for task in tasks {
        task.await;
    }
}
```

**Usage:**

```rust
cx.background_executor().scoped(|scope| {
    for item in items {
        scope.spawn(async move {
            process(item).await;
        });
    }
}).await;
// All tasks completed here
```

### 5. Channel Communication

```rust
use futures::channel::{mpsc, oneshot};

// Unbounded channel
let (tx, mut rx) = mpsc::unbounded();

cx.spawn(async move |cx| {
    while let Some(msg) = rx.next().await {
        process_message(msg, &mut cx);
    }
}).detach();

// Oneshot for single response
let (tx, rx) = oneshot::channel();

cx.spawn(async move {
    let result = compute().await;
    let _ = tx.send(result);
}).detach();

let result = rx.await?;
```

---

## Error Handling Patterns

**Files:** `crates/util/src/util.rs`, CLAUDE.md guidelines

### 1. Core Error Types

#### anyhow::Result

Primary error type for most functions:

```rust
use anyhow::{Context, Result};

pub fn load_config(path: &Path) -> Result<Config> {
    let contents = fs::read_to_string(path)
        .context("Failed to read config file")?;

    let config: Config = serde_json::from_str(&contents)
        .context("Failed to parse config")?;

    Ok(config)
}
```

#### Custom Error Types with thiserror

For domain-specific errors:

```rust
use thiserror::Error;

#[derive(Error, Debug, Clone)]
pub enum LanguageModelError {
    #[error("prompt too large: {tokens} tokens (max: {max_tokens})")]
    PromptTooLarge { tokens: u64, max_tokens: u64 },

    #[error("missing {provider} API key")]
    NoApiKey { provider: String },

    #[error("rate limit exceeded, retry after {retry_after:?}")]
    RateLimited { retry_after: Duration },
}
```

### 2. ResultExt Trait

**File:** `crates/util/src/util.rs`

Extension trait for silent error handling:

```rust
pub trait ResultExt<T> {
    type Ok;

    fn log_err(self) -> Option<Self::Ok>;
    fn warn_on_err(self) -> Option<Self::Ok>;
    fn log_with_level(self, level: log::Level) -> Option<Self::Ok>;
    fn debug_assert_ok(self, reason: &str) -> Self;
}

impl<T, E> ResultExt<T> for Result<T, E>
where
    E: std::fmt::Display,
{
    type Ok = T;

    #[track_caller]  // Captures exact call site
    fn log_err(self) -> Option<T> {
        match self {
            Ok(value) => Some(value),
            Err(error) => {
                let location = std::panic::Location::caller();
                log::error!("{} at {}:{}", error, location.file(), location.line());
                None
            }
        }
    }

    #[track_caller]
    fn warn_on_err(self) -> Option<T> {
        self.log_with_level(log::Level::Warn)
    }
}
```

**Usage:**

```rust
// Silent error logging, continues execution
if let Some(entries) = fs::read_dir(dir).await.log_err() {
    for entry in entries {
        process(entry);
    }
}

// Equivalent to:
match fs::read_dir(dir).await {
    Ok(entries) => {
        for entry in entries {
            process(entry);
        }
    }
    Err(err) => {
        log::error!("Failed to read directory: {}", err);
    }
}
```

### 3. Async Error Handling

```rust
pub trait TryFutureExt {
    type Ok;

    fn log_err(self) -> LogErrorFuture<Self>;
    fn warn_on_err(self) -> LogErrorFuture<Self>;
    fn unwrap(self) -> UnwrapFuture<Self>;
}

impl<F, T, E> TryFutureExt for F
where
    F: Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    type Ok = T;

    fn log_err(self) -> LogErrorFuture<Self> {
        LogErrorFuture {
            future: self,
            level: log::Level::Error,
        }
    }
}
```

**Usage:**

```rust
// Async error logging
let result = fetch_data()
    .await
    .log_err();

if let Some(data) = result {
    process(data);
}
```

### 4. Error Recovery Strategies

#### Strategy 1: Silent Recovery with Defaults

```rust
let config = load_config(&path)
    .log_err()
    .unwrap_or_default();
```

#### Strategy 2: Type Conversion

```rust
pub enum DirenvError {
    NotFound,
    FailedRun,
    InvalidJson,
}

impl From<std::io::Error> for DirenvError {
    fn from(err: std::io::Error) -> Self {
        if err.kind() == std::io::ErrorKind::NotFound {
            DirenvError::NotFound
        } else {
            DirenvError::FailedRun
        }
    }
}

// Usage:
fn load_direnv() -> Result<DirenvConfig, DirenvError> {
    let output = Command::new("direnv")
        .output()?;  // Converts io::Error to DirenvError

    parse_output(&output)?
}
```

#### Strategy 3: Option Chaining

```rust
let file_path = buffer
    .read(cx)
    .file()
    .and_then(|f| f.path().as_ref())
    .map(|p| p.to_path_buf());

let Some(path) = file_path else {
    return Task::ready(None);
};
```

### 5. Project Guidelines (from CLAUDE.md)

**Never do this:**

```rust
// ❌ Silent error discard
let _ = client.request(url).await?;

// ❌ Unwrap in production code
let config = load_config().unwrap();
```

**Do this instead:**

```rust
// ✅ Propagate errors
client.request(url).await?;

// ✅ Log errors with context
if let Some(config) = load_config().log_err() {
    use_config(config);
}

// ✅ Handle errors appropriately
match load_config() {
    Ok(config) => use_config(config),
    Err(err) => show_error_to_user(err),
}
```

**Error propagation to UI:**

```rust
async fn load_and_display(cx: &mut AsyncWindowContext) -> Result<()> {
    let data = fetch_data().await?;  // Propagate errors

    cx.update(|window, cx| {
        window.show_data(data, cx);
    })?;

    Ok(())
}

// Caller handles errors for user feedback
window.spawn(cx, async move |cx| {
    if let Err(err) = load_and_display(&mut cx).await {
        cx.update(|window, cx| {
            window.show_error(&err.to_string(), cx);
        }).log_err();
    }
}).detach();
```

---

## Builder & Fluent API Patterns

**Files:** `crates/gpui/src/util.rs`, `crates/gpui/src/styled.rs`, `crates/ui/src/*`

### 1. FluentBuilder Trait

**File:** `crates/gpui/src/util.rs`

Core trait for conditional fluent APIs:

```rust
pub trait FluentBuilder: Sized {
    fn when(self, condition: bool, then: impl FnOnce(Self) -> Self) -> Self {
        if condition {
            then(self)
        } else {
            self
        }
    }

    fn when_some<T>(
        self,
        option: Option<T>,
        then: impl FnOnce(Self, T) -> Self,
    ) -> Self {
        if let Some(value) = option {
            then(self, value)
        } else {
            self
        }
    }

    fn map<R>(self, then: impl FnOnce(Self) -> R) -> R {
        then(self)
    }
}

// Automatically implemented for all elements
impl<E: IntoElement> FluentBuilder for E {}
```

**Usage:**

```rust
div()
    .when(is_focused, |this| this.border_color(blue()))
    .when_some(tooltip_text, |this, text| this.tooltip(text))
    .map(|this| {
        if is_active {
            this.bg(colors.active)
        } else {
            this.bg(colors.inactive)
        }
    })
```

### 2. Styled Trait (Tailwind-like CSS)

**File:** `crates/gpui/src/styled.rs`

Provides chainable styling methods:

```rust
pub trait Styled: Sized {
    // Layout
    fn flex(mut self) -> Self;
    fn flex_col(mut self) -> Self;
    fn flex_row(mut self) -> Self;
    fn gap(mut self, gap: impl Into<Pixels>) -> Self;

    // Spacing (generated via macros)
    fn p(mut self, padding: impl Into<Pixels>) -> Self;
    fn p_0(mut self) -> Self { self.p(px(0.)) }
    fn p_1(mut self) -> Self { self.p(px(4.)) }
    fn p_2(mut self) -> Self { self.p(px(8.)) }
    // ... p_3 through p_24

    fn m(mut self, margin: impl Into<Pixels>) -> Self;
    fn m_0(mut self) -> Self { /* ... */ }
    // ... m_1 through m_24

    // Sizing
    fn w(mut self, width: impl Into<Length>) -> Self;
    fn h(mut self, height: impl Into<Length>) -> Self;
    fn size(mut self, size: impl Into<Size>) -> Self;

    // Colors
    fn bg(mut self, color: impl Into<Hsla>) -> Self;
    fn text_color(mut self, color: impl Into<Hsla>) -> Self;
    fn border_color(mut self, color: impl Into<Hsla>) -> Self;

    // ... 50+ more methods
}
```

**Usage:**

```rust
div()
    .flex()
    .flex_col()
    .gap_2()
    .p_4()
    .bg(colors.background)
    .border_1()
    .border_color(colors.border)
    .rounded_lg()
    .shadow_md()
    .child("Hello, world!")
```

### 3. ParentElement Trait

**File:** `crates/gpui/src/element.rs`

Tree-building API:

```rust
pub trait ParentElement: Sized {
    fn child(mut self, child: impl IntoElement) -> Self;

    fn children(mut self, children: impl IntoIterator<Item = impl IntoElement>) -> Self {
        for child in children {
            self = self.child(child);
        }
        self
    }
}
```

**Usage:**

```rust
div()
    .child(
        div()
            .child("Header")
    )
    .children(
        items.iter().map(|item| {
            div().child(item.name.clone())
        })
    )
    .child(
        div()
            .child("Footer")
    )
```

### 4. Complex Builder Example: Button

**File:** `crates/ui/src/components/button/button_like.rs`

```rust
pub struct ButtonLike {
    id: ElementId,
    style: ButtonStyle,
    disabled: bool,
    selected: bool,
    tooltip: Option<Box<dyn Fn(&mut Window, &mut App) -> AnyView>>,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App)>>,
    children: SmallVec<[AnyElement; 2]>,
}

impl ButtonLike {
    pub fn new(id: impl Into<ElementId>) -> Self {
        Self {
            id: id.into(),
            style: ButtonStyle::default(),
            disabled: false,
            selected: false,
            tooltip: None,
            on_click: None,
            children: SmallVec::new(),
        }
    }

    pub fn style(mut self, style: ButtonStyle) -> Self {
        self.style = style;
        self
    }

    pub fn disabled(mut self, disabled: bool) -> Self {
        self.disabled = disabled;
        self
    }

    pub fn selected(mut self, selected: bool) -> Self {
        self.selected = selected;
        self
    }

    pub fn tooltip(mut self, tooltip: impl Fn(&mut Window, &mut App) -> AnyView + 'static) -> Self {
        self.tooltip = Some(Box::new(tooltip));
        self
    }

    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }

    pub fn child(mut self, child: impl IntoElement) -> Self {
        self.children.push(child.into_any_element());
        self
    }
}
```

**Usage:**

```rust
ButtonLike::new("save-button")
    .style(ButtonStyle::Filled)
    .tooltip(|_, _| view::tooltip("Save changes"))
    .on_click(|_, window, cx| {
        save_document(window, cx);
    })
    .child(Icon::new(IconName::Save))
    .child("Save")
```

### 5. Into Conversions for Optional Parameters

```rust
impl ButtonLike {
    pub fn size(mut self, size: impl Into<Option<ButtonSize>>) -> Self {
        if let Some(size) = size.into() {
            self.size = size;
        }
        self
    }
}

// Allows both:
button.size(ButtonSize::Large)
button.size(None)
```

### 6. Generic Wrapper Pattern

**File:** `crates/gpui/src/elements/div.rs`

```rust
pub struct Stateful<E> {
    inner: E,
    #[allow(dead_code)]
    entity: AnyEntity,
}

impl<E: IntoElement> Stateful<E> {
    pub fn new(entity: impl Into<AnyEntity>, inner: E) -> Self {
        Self {
            inner,
            entity: entity.into(),
        }
    }
}

// Delegate all Styled methods to inner
impl<E: Styled> Styled for Stateful<E> {
    // All methods forward to self.inner
}

impl<E: ParentElement> ParentElement for Stateful<E> {
    fn child(mut self, child: impl IntoElement) -> Self {
        self.inner = self.inner.child(child);
        self
    }
}
```

**Purpose:** Adds state tracking to any element without duplicating API.

---

## Trait-Based Abstractions

**Files:** `crates/gpui/src/*`, `crates/util/src/util.rs`

### 1. Trait Objects & Dynamic Dispatch

#### Platform Abstraction

```rust
pub trait Platform: Send + Sync {
    fn displays(&self) -> Vec<Rc<dyn PlatformDisplay>>;
    fn primary_display(&self) -> Option<Rc<dyn PlatformDisplay>>;
    fn open_window(&self, options: WindowParams) -> Box<dyn PlatformWindow>;
    fn run(&self, on_finish_launching: Box<dyn FnOnce()>);
    // ... more methods
}

// Usage with trait object
let platform: Arc<dyn Platform> = current_platform();
let window = platform.open_window(params);
```

#### Event Handlers

```rust
pub type ClickHandler = Box<dyn Fn(&ClickEvent, &mut Window, &mut App)>;

pub struct Button {
    on_click: Option<ClickHandler>,
}

impl Button {
    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }
}
```

#### Type-Erased Storage

```rust
use std::any::{Any, TypeId};

pub struct EntityMap {
    entities: SecondaryMap<EntityId, Box<dyn Any>>,
}

impl EntityMap {
    pub fn insert<T: 'static>(&mut self, id: EntityId, value: T) {
        self.entities.insert(id, Box::new(value));
    }

    pub fn get<T: 'static>(&self, id: EntityId) -> Option<&T> {
        self.entities
            .get(id)
            .and_then(|v| v.downcast_ref::<T>())
    }
}
```

### 2. Associated Types

#### Element Trait

```rust
pub trait Element: IntoElement {
    type RequestLayoutState: 'static;
    type PrepaintState: 'static;

    fn request_layout(
        &mut self,
        element_state: Option<Self::RequestLayoutState>,
        cx: &mut WindowContext,
    ) -> (LayoutId, Self::RequestLayoutState);

    fn prepaint(
        &mut self,
        element_state: Option<Self::PrepaintState>,
        layout: &Layout,
        cx: &mut WindowContext,
    ) -> Self::PrepaintState;

    fn paint(
        self,
        element_state: Self::PrepaintState,
        layout: &Layout,
        cx: &mut WindowContext,
    );
}
```

**Purpose:** Each element can have its own state types without boxing.

#### IntoElement Trait

```rust
pub trait IntoElement: Sized {
    type Element: Element;

    fn into_element(self) -> Self::Element;
}

// Example implementation
impl IntoElement for Div {
    type Element = Div;

    fn into_element(self) -> Self::Element {
        self
    }
}

impl IntoElement for &str {
    type Element = Text;

    fn into_element(self) -> Self::Element {
        Text::new(self)
    }
}
```

#### Asset Trait

```rust
pub trait Asset: Clone {
    type Source: Send;
    type Output: Send;

    fn load(
        source: Self::Source,
        cx: &App,
    ) -> impl Future<Output = Result<Self::Output>>;
}

// Example: SVG assets
impl Asset for Svg {
    type Source = SharedString;  // SVG path
    type Output = SvgData;       // Parsed SVG

    async fn load(path: SharedString, cx: &App) -> Result<SvgData> {
        let bytes = cx.asset_source().load(&path).await?;
        parse_svg(&bytes)
    }
}
```

### 3. Trait Bounds & Where Clauses

#### Geometric Types

```rust
#[derive(Clone, Copy, Debug, Default)]
pub struct Point<T: Clone + Debug + Default> {
    pub x: T,
    pub y: T,
}

impl<T> PartialEq for Point<T>
where
    T: Clone + Debug + Default + PartialEq,
{
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}

impl<T> Eq for Point<T>
where
    T: Eq + Clone + Debug + Default,
{}
```

#### Conditional Implementations

```rust
impl<T> Point<T>
where
    T: Clone + Debug + Default + Add<Output = T>,
{
    pub fn add(&self, other: &Self) -> Self {
        Self {
            x: self.x.clone() + other.x.clone(),
            y: self.y.clone() + other.y.clone(),
        }
    }
}
```

### 4. Extension Traits

#### FutureExt

```rust
pub trait FutureExt: Future + Sized {
    fn with_timeout(
        self,
        duration: Duration,
    ) -> impl Future<Output = Result<Self::Output, TimeoutError>> {
        async move {
            let timeout = Timer::after(duration);
            select! {
                result = self.fuse() => Ok(result),
                _ = timeout.fuse() => Err(TimeoutError),
            }
        }
    }
}

impl<F: Future> FutureExt for F {}

// Usage:
let result = fetch_data()
    .with_timeout(Duration::from_secs(30))
    .await?;
```

#### ResultExt (already covered in error handling)

```rust
pub trait ResultExt<T> {
    type Ok;
    fn log_err(self) -> Option<Self::Ok>;
    fn warn_on_err(self) -> Option<Self::Ok>;
}

impl<T, E: Display> ResultExt<T> for Result<T, E> {
    type Ok = T;
    // ... implementation
}
```

#### FluentBuilder (already covered in builder patterns)

```rust
pub trait FluentBuilder: Sized {
    fn when(self, condition: bool, f: impl FnOnce(Self) -> Self) -> Self;
    fn when_some<T>(self, opt: Option<T>, f: impl FnOnce(Self, T) -> Self) -> Self;
}

impl<E: IntoElement> FluentBuilder for E {}
```

### 5. Sealed Traits

**File:** `crates/gpui/src/gpui.rs`

```rust
mod seal {
    pub trait Sealed {}
}

// Only GPUI can implement InputEvent
pub trait InputEvent: seal::Sealed + 'static {
    fn as_any(&self) -> &dyn Any;
}

// Implementations are sealed
impl seal::Sealed for KeyDownEvent {}
impl InputEvent for KeyDownEvent {
    fn as_any(&self) -> &dyn Any {
        self
    }
}

impl seal::Sealed for MouseDownEvent {}
impl InputEvent for MouseDownEvent {
    fn as_any(&self) -> &dyn Any {
        self
    }
}
```

**Purpose:**
- Prevent external crates from implementing the trait
- Allow adding new methods without breaking changes
- Maintain API stability

**Usage in asset loading:**

```rust
pub struct AssetLogger<T> {
    _phantom: PhantomData<fn() -> T>,
}

impl<T: Asset> AssetLogger<T>
where
    T::Source: seal::Sealed,  // Only internal types allowed
{
    pub fn log_load(source: &T::Source) {
        // Internal logging
    }
}
```

---

## Key Architectural Patterns

### 1. Entity-Component System (ECS-like)

Entities store state, contexts provide capabilities:

```rust
// Entity = data container
struct DocumentState {
    text: String,
    cursor: usize,
}

// Context = capabilities
impl DocumentState {
    fn init(cx: &mut Context<Self>) -> Self {
        // Can spawn tasks
        cx.spawn(async |this, cx| {
            // Background work
        }).detach();

        // Can observe other entities
        cx.observe(&other_entity, |this, _, cx| {
            this.on_change(cx);
        }).detach();

        Self::default()
    }
}
```

### 2. Type-State Pattern (Compile-Time Safety)

```rust
pub struct RequestBuilder<State> {
    url: String,
    _state: PhantomData<State>,
}

pub struct NoBody;
pub struct WithBody;

impl RequestBuilder<NoBody> {
    pub fn new(url: String) -> Self {
        Self {
            url,
            _state: PhantomData,
        }
    }

    pub fn body(self, body: String) -> RequestBuilder<WithBody> {
        RequestBuilder {
            url: self.url,
            _state: PhantomData,
        }
    }
}

impl RequestBuilder<WithBody> {
    pub async fn send(self) -> Response {
        // Can only send with body
    }
}
```

### 3. Registry Pattern (Service Locator)

```rust
pub struct LanguageRegistry {
    languages: HashMap<String, Arc<Language>>,
}

impl LanguageRegistry {
    pub fn register(&mut self, name: String, language: Arc<Language>) {
        self.languages.insert(name, language);
    }

    pub fn get(&self, name: &str) -> Option<&Arc<Language>> {
        self.languages.get(name)
    }
}

// Global access
let registry = cx.global::<LanguageRegistry>();
let rust = registry.get("rust")?;
```

### 4. Provider Pattern

```rust
pub trait LanguageModelProvider: 'static {
    fn name(&self) -> String;
    fn models(&self) -> Vec<LanguageModel>;
    fn complete(&self, request: CompletionRequest) -> Task<CompletionStream>;
}

pub struct LanguageModelRegistry {
    providers: Vec<Arc<dyn LanguageModelProvider>>,
}

impl LanguageModelRegistry {
    pub fn register(&mut self, provider: Arc<dyn LanguageModelProvider>) {
        self.providers.push(provider);
    }
}
```

### 5. Observable State

```rust
// State changes trigger observers
entity.update(cx, |state, cx| {
    state.value = new_value;
    cx.notify();  // Triggers all observers
});

// Observers react to changes
cx.observe(&entity, |this, observed, cx| {
    // React to changes
    this.sync_with(observed.read(cx));
});
```

### 6. Slot Map for Entity IDs

Uses `slotmap` crate for stable, reusable IDs:

```rust
slotmap::new_key_type! {
    pub struct EntityId;
}

// Benefits:
// - O(1) lookup
// - Reuses deleted slots
// - Type-safe (can't mix different key types)
// - No ABA problem
```

---

## Design Principles & Best Practices

### 1. From CLAUDE.md Guidelines

**Code Correctness:**
- Prioritize correctness and clarity over performance
- Avoid `unwrap()`, use `?` for error propagation
- Never silently discard errors with `let _ =`
- Use `.log_err()` when ignoring errors is intentional

**Error Handling:**
- Propagate errors to UI layer for user feedback
- Use `anyhow::Result` for most functions
- Use `thiserror` for domain-specific errors
- Always handle errors appropriately

**Async Operations:**
- Ensure errors propagate to UI layer
- Use weak handles in async closures to prevent leaks
- Detach long-running tasks explicitly

**File Organization:**
- Prefer implementing functionality in existing files
- Avoid creating many small files
- Never create `mod.rs` - use `src/module_name.rs`
- For new crates, specify `[lib] path = "crate_name.rs"`

**Variable Naming:**
- Use full words, no abbreviations (e.g., `queue` not `q`)
- Use descriptive names that convey intent

**Comments:**
- Only explain "why", not "what"
- Code should be self-documenting
- Comments for tricky/non-obvious logic only

**Variable Shadowing:**
- Use shadowing to scope clones in async:
```rust
executor.spawn({
    let state = state.clone();
    async move {
        // Use state here
    }
});
```

### 2. Workspace Organization

**Crate Sizing:**
- Keep crates focused and cohesive
- Separate concerns (e.g., `gpui` vs `gpui_macros`)
- Use workspace dependencies for version consistency

**Feature Flags:**
```rust
[features]
default = ["wayland", "x11"]
test-support = ["leak-detection", "util/test-support"]
inspector = ["gpui_macros/inspector"]
```

### 3. Performance Patterns

**Zero-Cost Abstractions:**
- Heavy use of generics and monomorphization
- Trait objects only where dynamic dispatch is necessary
- `SmallVec` for small collections to avoid heap allocations

**Memory Management:**
- Reference counting with `Arc` for shared ownership
- Weak references to prevent cycles
- Slot maps for efficient entity storage

**Compile Times:**
```toml
[profile.dev]
codegen-units = 16  # Faster incremental builds

[profile.release]
codegen-units = 1   # Better optimization
lto = "thin"        # Link-time optimization
```

### 4. Testing Patterns

**Test Organization:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[gpui::test]
    async fn test_feature(cx: &TestAppContext) {
        // Test with GPUI context
    }

    #[gpui::test(iterations = 100)]
    async fn test_with_randomness(cx: &TestAppContext, rng: StdRng) {
        // Property-based testing with deterministic randomness
    }
}
```

**Test Utilities:**
```rust
impl TestAppContext {
    pub fn notifications<T>(&mut self, entity: &Entity<T>) -> impl Stream<Item = ()> {
        // Stream of notifications from entity
    }

    pub fn simulate_input(&mut self, event: InputEvent) {
        // Simulate user input
    }
}
```

### 5. Type Safety Principles

**Newtype Pattern:**
```rust
pub struct WorktreeId(usize);
pub struct BufferId(u64);

// Can't accidentally mix IDs
```

**Phantom Types:**
```rust
pub struct Entity<T> {
    entity_id: EntityId,
    _phantom: PhantomData<T>,
}

// Type-safe entity handles
```

**Builder Pattern for Complex Construction:**
```rust
ThreadSafeConnection::builder::<MyDomain>("db.sqlite", true)
    .with_db_initialization_query("PRAGMA journal_mode=WAL")
    .build()
    .await?
```

---

## Conclusion

Zed demonstrates production-ready Rust patterns for building complex applications:

### Key Takeaways for CMS Development

1. **Procedural Macros:** Reduce boilerplate and provide compile-time safety
2. **Entity System:** Clean state management with automatic lifecycle tracking
3. **Type-Safe Database:** Compile-time SQL validation prevents runtime errors
4. **Settings System:** Hierarchical configuration with hot-reloading
5. **Async Architecture:** Separate foreground/background execution for responsiveness
6. **Error Handling:** Comprehensive error handling without panic risks
7. **Builder APIs:** Ergonomic, discoverable APIs through method chaining
8. **Trait Abstractions:** Flexible abstractions without runtime overhead

### Application to CMS

- **Content Types:** Use entity system for content management
- **Database:** Adapt sqlez patterns for PostgreSQL
- **Settings:** Apply layered settings for tenant/user/project config
- **API Layer:** Use builder patterns for query construction
- **Async Operations:** Background tasks for media processing, indexing
- **Error Handling:** Propagate errors to API responses with proper logging

### Critical Files Reference

| Pattern | File Location |
|---------|---------------|
| Entity System | `crates/gpui/src/app/entity_map.rs` |
| Context Types | `crates/gpui/src/app/context.rs` |
| SQL Macro | `crates/sqlez_macros/src/sqlez_macros.rs` |
| Database Layer | `crates/sqlez/src/thread_safe_connection.rs` |
| Settings Store | `crates/settings/src/settings_store.rs` |
| Async Executors | `crates/gpui/src/executor.rs` |
| Error Utilities | `crates/util/src/util.rs` |
| Builder Traits | `crates/gpui/src/util.rs` |

---

**End of Zed Architecture Analysis**
