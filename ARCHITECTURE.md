# virgil-cli Architecture

This document describes the **intent**, **logic**, and **patterns** of the virgil-cli codebase.

---

## Table of Contents

1. [Intent](#intent)
2. [High-Level Architecture](#high-level-architecture)
3. [Module Map](#module-map)
4. [Core Logic Flows](#core-logic-flows)
   - [Query Execution](#query-execution)
   - [Audit Execution](#audit-execution)
   - [Server Mode](#server-mode)
   - [S3 Workflow](#s3-workflow)
5. [Data Models](#data-models)
6. [Design Patterns](#design-patterns)
7. [Language Modules](#language-modules)
8. [Audit System](#audit-system)
9. [Performance Decisions](#performance-decisions)
10. [Key Trade-offs and Design Decisions](#key-trade-offs-and-design-decisions)

---

## Intent

virgil-cli is a **multi-language code intelligence and static analysis tool** that operates purely on-demand — no daemon, no database, no pre-built index. Its core value proposition is:

- **Query any codebase symbolically** using a composable JSON query language. Ask "show me all exported functions in `src/api/**` whose name starts with `handle` and are longer than 20 lines."
- **Audit codebases** for tech debt, complexity, code style, security vulnerabilities, scalability issues, and architectural problems — across 12 programming languages.
- **Work without registration** when using S3/R2/MinIO storage. Point it at a bucket prefix and immediately query or audit the code without any setup.
- **Serve as a persistent REST API** (server mode) so cloud services like Virgil Live can query in-memory workspaces without re-parsing on every request.

The tool is intentionally stateless at query time. Project registration only stores metadata (path, language filter, file count). All parsing happens at query time, in parallel, scoped to the files the query actually needs.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        CLI (clap)                        │
│   projects create/list/delete/query   audit   serve      │
└──────────────┬─────────────┬──────────────┬─────────────┘
               │             │              │
    ┌──────────▼──┐  ┌───────▼──────┐  ┌───▼────────────┐
    │  Registry   │  │ Query Engine  │  │  Audit Engine  │
    │  (CRUD,     │  │ (on-demand   │  │  (Pipeline     │
    │  JSON file) │  │  parse+filter)│  │   dispatch)    │
    └─────────────┘  └───────┬──────┘  └────────┬───────┘
                             │                   │
                    ┌────────▼──────────────────▼────────┐
                    │             Workspace               │
                    │  (in-memory Arc<str> file store)    │
                    └────────┬──────────────────┬────────┘
                             │                  │
                    ┌────────▼────┐    ┌────────▼────────┐
                    │  FileSource │    │  tree-sitter    │
                    │  (local or  │    │  (per-language  │
                    │   S3/R2)    │    │   AST parsing)  │
                    └─────────────┘    └─────────────────┘
```

**Data flow summary:**

1. A command is parsed by `clap` into a typed `Cli` struct.
2. The workspace is loaded: files are discovered on disk (or downloaded from S3) and stored in memory as `Arc<str>`.
3. For **query**: tree-sitter parses each relevant file in parallel; symbol extraction and filter pipeline runs; results are formatted as JSON.
4. For **audit**: tree-sitter parses all files in parallel; language-specific `Pipeline` implementations run checks; a `ProjectIndex` enables cross-file analyzers.
5. For **serve**: steps 2–4 run from an in-memory axum HTTP server backed by `tokio::task::spawn_blocking`.

---

## Module Map

```
src/
├── main.rs            CLI entry point — parses args, dispatches to handlers
├── cli.rs             Clap subcommand definitions (all commands, flags, enums)
├── lib.rs             Public API: re-exports all modules for library use
│
├── registry.rs        Project CRUD — ~/.virgil-cli/projects.json (atomic write)
├── workspace.rs       Workspace — loads files into memory, abstracts local/S3
├── file_source.rs     FileSource trait + MemoryFileSource (Arc<str>, zero-copy)
├── discovery.rs       File walker (ignore crate, .gitignore-aware)
│
├── query_lang.rs      JSON query schema — TsQuery, serde untagged filter enums
├── query_engine.rs    Query execution — parse + filter pipeline (rayon parallel)
├── format.rs          Query output formatting — outline/snippet/full/tree/locations/summary
│
├── language.rs        Language enum, extension mapping, tree-sitter binding
├── parser.rs          Tree-sitter parsing, FileMetadata extraction
├── models.rs          Core data structs — SymbolInfo, ImportInfo, CommentInfo
├── signature.rs       Signature extraction — first line up to opening brace
├── call_graph.rs      Call graph traversal (BFS, up/down/both directions)
│
├── s3.rs              S3/R2/MinIO client — list, download, credential resolution
├── server.rs          HTTP server mode (axum, tokio, spawn_blocking)
│
├── languages/
│   ├── mod.rs         Dispatch — compile_symbol_query, extract_symbols, etc.
│   ├── typescript.rs  TS/JS/TSX/JSX tree-sitter queries and extraction
│   ├── c_lang.rs      C
│   ├── cpp.rs         C++
│   ├── csharp.rs      C#
│   ├── rust_lang.rs   Rust
│   ├── python.rs      Python
│   ├── go.rs          Go
│   ├── java.rs        Java
│   └── php.rs         PHP
│
└── audit/
    ├── mod.rs             Re-exports public audit API
    ├── engine.rs          AuditEngine — orchestrates parallel audit runs
    ├── pipeline.rs        Pipeline trait + per-language pipeline routing
    ├── models.rs          AuditFinding, AuditSummary
    ├── format.rs          Audit output (table, JSON, CSV)
    ├── primitives.rs      Shared helpers (node_text, extract_snippet)
    ├── project_index.rs   ProjectIndex — dependency graph, FileEntry, ExportedSymbol
    ├── index_builder.rs   build_index() — parallel extraction + graph construction
    ├── project_analyzer.rs ProjectAnalyzer trait (cross-file analysis)
    ├── analyzers/
    │   ├── mod.rs
    │   ├── circular_deps.rs    Tarjan SCC cycle detection
    │   ├── coupling.rs         Efferent/afferent coupling thresholds
    │   ├── dead_exports.rs     Unused public symbols
    │   ├── dependency_depth.rs BFS import chain depth
    │   └── duplicate_symbols.rs Same (name, kind, signature) across files
    └── pipelines/
        ├── mod.rs             Imports all language pipeline modules
        ├── helpers.rs         Shared: cyclomatic/cognitive complexity, dead code counts
        ├── rust/              tech_debt, complexity, code_style, security, scalability, architecture
        ├── typescript/
        ├── javascript/
        ├── python/
        ├── go/
        ├── java/
        ├── c/
        ├── cpp/
        ├── csharp/
        └── php/
```

---

## Core Logic Flows

### Query Execution

`query_engine::execute(project, query, max, workspace)` is the heart of the query system:

```
execute()
  │
  ├─ 1. Read mode? (query.read is Some)
  │      └─ execute_read() → return file content, optionally sliced by lines
  │
  ├─ 2. Resolve language filter from project entry
  │
  ├─ 3. Build file list from workspace
  │      ├─ Apply files glob pattern(s) (globset)
  │      └─ Apply files_exclude patterns
  │
  ├─ 4. Pre-compile tree-sitter queries
  │      └─ HashMap<Language, Arc<Query>> (one compile per language, shared across threads)
  │
  ├─ 5. Parallel file processing (rayon par_iter)
  │      For each file:
  │      ├─ Create tree-sitter Parser (not Send; created per task)
  │      ├─ Parse source into AST
  │      ├─ extract_symbols() → Vec<SymbolInfo>
  │      ├─ (if has filter) extract_comments() → Vec<CommentInfo>
  │      ├─ Filter symbols:
  │      │   ├─ find filter (kind match: function/class/method/…)
  │      │   ├─ name filter (glob, contains, or regex)
  │      │   ├─ visibility filter (exported / public / private / …)
  │      │   ├─ lines filter (end_line - start_line in range)
  │      │   ├─ inside filter (line range containment in parent symbol)
  │      │   └─ has filter (comment text match or inverse)
  │      ├─ Build QueryResult: signature, docstring, body, preview, parent chain
  │      └─ Emit Vec<QueryResult>
  │
  ├─ 6. Call graph (if query.calls is Some)
  │      └─ traverse_call_graph(workspace, seed_results, direction, depth)
  │
  ├─ 7. Flatten + sort by (file, line)
  ├─ 8. Truncate to max
  └─ 9. Return QueryOutput { results, files_parsed, total }
```

**Name matching** compiles once to one of three matchers:
- `GlobSet` — default, supports `*` wildcards
- `Contains` — substring match (`{"contains": "auth"}`)
- `Regex` — full regex (`{"regex": "^get[A-Z]"}`)

**`find: "function"`** matches both `Function` and `ArrowFunction` symbol kinds.  
**`find: "constructor"`** matches the `Method` kind post-filtered by reserved names (`constructor`, `__init__`, `__construct`, `new`).

---

### Audit Execution

`AuditEngine::run(workspace, index)` coordinates the two-phase static analysis:

```
run()
  │
  ├─ Phase 1: Build pipeline map
  │     └─ pipeline_selector → language-specific pipelines()
  │         e.g., PipelineSelector::TechDebt → tech_debt_pipelines_for_language()
  │         Optionally filtered by pipeline_filter (name strings)
  │
  ├─ Phase 2: Group workspace files by Language
  │
  ├─ Phase 3: Parallel file analysis (rayon, 4 MB stack per thread)
  │     For each file:
  │     ├─ Create Parser, parse AST
  │     ├─ Pre-compute identifier counts (walk entire AST once)
  │     ├─ Run each Pipeline::check_with_ids(tree, source, path, id_counts)
  │     └─ Collect Vec<AuditFinding>
  │
  ├─ Phase 4: Cross-file analysis (if ProjectIndex provided)
  │     └─ Architecture: CircularDepsAnalyzer, CouplingAnalyzer, DependencyDepthAnalyzer
  │        CodeStyle:    DeadExportsAnalyzer, DuplicateSymbolsAnalyzer
  │
  └─ Phase 5: Compute AuditSummary
        ├─ Count by pipeline (unstable sort, descending)
        ├─ Count by pattern
        └─ Build nested by_pipeline_pattern breakdown
```

**ProjectIndex** is built separately by `index_builder::build_index()` before `run()` is called for architecture and code-style categories. It contains the full dependency graph (`edges: HashMap<GraphNode, HashSet<GraphNode>>`), per-file symbol data, and a `known_files` set for import resolution.

---

### Server Mode

`serve` loads the workspace once from S3 and runs an axum HTTP server:

```
serve --s3 s3://bucket/prefix --lang rs --port 0
  │
  ├─ 1. Load workspace from S3 (Workspace::load_from_s3)
  ├─ 2. Build ProjectIndex (for audit endpoints)
  ├─ 3. Bind axum router on 127.0.0.1:port
  │      Routes:
  │      ├─ GET  /health
  │      ├─ POST /query          → spawn_blocking(query_engine::execute)
  │      ├─ POST /audit/summary  → spawn_blocking(AuditEngine::run) × 4 categories
  │      └─ POST /audit/{category}
  ├─ 4. Print ready signal to stdout: {"ready": true, "port": N}
  └─ 5. Serve until SIGTERM/SIGINT
```

All CPU-bound work runs inside `tokio::task::spawn_blocking` because `query_engine::execute` and `AuditEngine::run` use rayon internally and would block tokio worker threads if called directly. A 120-second request timeout wraps each handler.

---

### S3 Workflow

When `--s3 s3://bucket/prefix` is supplied:

```
s3::list_objects(location, languages, exclude_patterns)
  │   └─ paginate ListObjectsV2 via ContinuationToken
  │      filter keys by language extension and exclude globs
  │
s3::download_objects(location, keys, max_file_size)
  │   └─ 64-semaphore bounded tokio concurrency
  │      GetObject → stream body → Arc<str>
  │
Workspace { source: MemoryFileSource, root: "s3://bucket/prefix" }
```

Credentials are resolved in priority order:
1. `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` / `S3_ENDPOINT`
2. `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_ENDPOINT_URL`
3. Standard AWS credential chain (env, profile, instance metadata)

Region defaults to `"auto"` for Cloudflare R2 compatibility.

---

## Data Models

### Symbol and File Data (`src/models.rs`)

```
SymbolKind         — enum of 17 kinds: Function, Class, Method, Variable, Interface,
                     TypeAlias, Enum, ArrowFunction, Struct, Union, Namespace, Macro,
                     Property, Typedef, Trait, Constant, Module

SymbolInfo         — name, kind, file_path, start/end line+column, is_exported

ImportInfo         — source_file, module_specifier, imported_name, local_name,
                     kind (free-form string), is_type_only, line, is_external

CommentInfo        — file_path, text, kind ("line"/"block"/"doc"),
                     start/end line+column, associated_symbol (name + kind)

FileMetadata       — path, name, extension, language, size_bytes, line_count
```

`ImportInfo.kind` is a free-form `String` (not an enum) so each language can define its own import variants (`"include"`, `"using"`, `"use"`, `"require"`, etc.) without modifying a central type.

### Query Results (`src/query_engine.rs`)

```
QueryResult {
    name, kind, file, line, end_line, column,
    exported: bool,
    signature: Option<String>,   -- extracted to opening brace
    docstring: Option<String>,   -- nearest preceding doc comment
    body: Option<String>,        -- full source when body: true
    preview: Option<String>,     -- first N lines when preview: N
    parent: Option<String>,      -- enclosing class/function name
}

QueryOutput {
    results: Vec<QueryResult>,
    files_parsed: usize,
    total: usize,
    read: Option<ReadResult>,    -- populated when query.read is Some
}
```

### Audit Results (`src/audit/models.rs`)

```
AuditFinding {
    file_path, line, column,
    severity: String,  -- "warning" | "info" | "error"
    pipeline: String,  -- e.g. "panic_detection"
    pattern: String,   -- e.g. "unwrap_call"
    message: String,
    snippet: String,   -- code excerpt or signature
}

AuditSummary {
    total_findings, files_scanned, files_with_findings,
    by_pipeline: Vec<(String, usize)>,           -- sorted DESC
    by_pattern: Vec<(String, usize)>,
    by_pipeline_pattern: Vec<(String, Vec<(String, usize)>)>,
}
```

### Project Index (`src/audit/project_index.rs`)

```
GraphNode          — File(String) | Package(String) (Go uses package-level nodes)

ExportedSymbol     — name, kind, signature, start_line

FileEntry          — path, language, line_count, symbol_count,
                     exported_symbols: Vec<ExportedSymbol>,
                     imports: Vec<ImportInfo>

ProjectIndex {
    files:       HashMap<String, FileEntry>,
    edges:       HashMap<GraphNode, HashSet<GraphNode>>,  -- dependency graph
    known_files: HashSet<String>,
}
```

---

## Design Patterns

### 1. Trait-Based Extensibility

Two pluggable abstractions form the backbone of the audit system:

**`Pipeline` (per-file analysis)**

```rust
pub trait Pipeline: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn check(&self, tree: &Tree, source: &[u8], file_path: &str) -> Vec<AuditFinding>;
    fn check_with_ids(&self, tree: &Tree, source: &[u8], file_path: &str,
                      id_counts: &HashMap<String, usize>) -> Vec<AuditFinding>;
}
```

`check_with_ids` carries pre-computed identifier occurrence counts so dead-code pipelines don't re-walk the AST independently.

**`ProjectAnalyzer` (cross-file analysis)**

```rust
pub trait ProjectAnalyzer: Send + Sync {
    fn name(&self) -> &str;
    fn analyze(&self, index: &ProjectIndex) -> Vec<AuditFinding>;
}
```

Operates on a complete `ProjectIndex` (built once) rather than individual files. Used for circular dependency detection, coupling, and dead export analysis.

**`FileSource` (pluggable I/O)**

```rust
pub trait FileSource: Send + Sync {
    fn read_file(&self, relative_path: &str) -> Option<Arc<str>>;
    fn list_files(&self) -> &[String];
    fn file_exists(&self, relative_path: &str) -> bool;
    fn file_size(&self, relative_path: &str) -> Option<u64>;
}
```

`MemoryFileSource` is the only implementation today. S3 downloads populate it at startup; local workspaces read from disk into the same structure. This decouples query/audit logic from I/O source entirely.

---

### 2. Rayon-Parallel, Parser-Per-Thread

tree-sitter's `Parser` is not `Send`. The codebase never shares a parser across threads. Instead:

```rust
files.par_iter().flat_map(|file| {
    let mut parser = create_parser(language).unwrap(); // per-thread instance
    let tree = parser.parse(source, None)?;
    run_pipelines(&tree, source)
})
```

`Query` objects (compiled tree-sitter S-expression patterns) are safe to share and are wrapped in `Arc<Query>` and pre-compiled once into a `HashMap<Language, Arc<Query>>` before the parallel section.

---

### 3. Atomic Registry Writes

The project registry avoids partial writes on crash:

```
write → ~/.virgil-cli/projects.json.tmp
rename → ~/.virgil-cli/projects.json    (atomic on all POSIX systems)
```

---

### 4. Zero-Copy In-Memory Files

File contents are stored as `Arc<str>` in `MemoryFileSource`. Multiple threads can hold references to the same file content without copying. Cloning an `Arc<str>` is a reference count increment, not a memory allocation.

---

### 5. Polymorphic JSON Filters (Untagged Serde Enums)

The query language uses `#[serde(untagged)]` to accept multiple shapes for the same field:

```json
"name": "handle*"                     // Glob string
"name": {"contains": "auth"}          // Substring object
"name": {"regex": "^get[A-Z]"}        // Regex object

"find": "function"                    // Single kind
"find": ["function", "method"]        // Multiple kinds

"has": "@deprecated"                  // Single annotation
"has": ["@deprecated", "@todo"]       // Multiple
"has": {"not": "docstring"}           // Inverse
```

This is implemented via Rust's `#[serde(untagged)]` enum attribute. Serde tries each variant in declaration order until one succeeds.

---

### 6. BFS Call Graph Traversal

Call graph traversal is heuristic (name-based, no type resolution). The algorithm:

1. Seeds come from initial `QueryResult` symbols.
2. A BFS queue tracks `(symbol_name, depth)` pairs.
3. A `visited` `HashSet<(name, line)>` prevents cycles.
4. **Down (callees):** Walk the symbol's AST subtree; collect `call_expression` names.
5. **Up (callers):** Scan all symbols across all files for call expressions referencing this name.
6. Maximum depth is capped at 5 to bound execution time.

The heuristic nature is intentional and documented. For large monorepos, full type-aware call graphs would require a separate language server.

---

### 7. Stack-Based AST Iteration

All recursive-style AST walks in `helpers.rs` use an explicit `Vec<Node>` stack instead of recursive function calls. This prevents stack overflows when auditing deeply nested code on rayon threads limited to 4 MB stack:

```rust
fn walk_all(node: Node, cursor: &mut TreeCursor, f: &mut impl FnMut(Node)) {
    let mut stack = vec![node];
    while let Some(n) = stack.pop() {
        f(n);
        stack.extend(n.children(cursor));
    }
}
```

---

### 8. Tarjan SCC for Circular Dependencies

`CircularDepsAnalyzer` runs Tarjan's Strongly Connected Components algorithm on the `ProjectIndex.edges` graph. Any SCC with ≥ 2 nodes is a cycle. The algorithm uses an explicit stack rather than recursion for the same stack-overflow safety reason described above.

---

### 9. Language-Dispatch via Enum

`Language` is an enum with 12 variants. Language-specific behavior is dispatched through:

- `Language::tree_sitter_language()` → returns the appropriate tree-sitter `Language` binding
- `languages::mod::compile_symbol_query(language)` → returns the compiled `Arc<Query>`
- `pipeline::tech_debt_pipelines_for_language(language)` → returns `Vec<Box<dyn Pipeline>>`

Adding a new language requires: (1) a new enum variant, (2) an extension mapping, (3) a `languages/` module, (4) a `pipelines/` module. No existing code changes are required beyond the dispatch match arms.

---

## Language Modules

Each file in `src/languages/` implements the same three extraction functions for its language family:

| Function | Returns | Used by |
|----------|---------|---------|
| `extract_symbols(tree, source, query, path, lang)` | `Vec<SymbolInfo>` | query engine, index builder |
| `extract_imports(tree, source, query, path, lang)` | `Vec<ImportInfo>` | index builder |
| `extract_comments(tree, source, query, path, lang)` | `Vec<CommentInfo>` | query engine (has filter), docstrings |

S-expression tree-sitter queries are hardcoded as `const` strings within each module. They are compiled once by `languages::mod::compile_symbol_query(language)` and shared via `Arc<Query>` across threads.

### Export Detection Rules by Language

| Language | Exported when… |
|----------|---------------|
| TypeScript/JS | Parent node is `export_statement` |
| Rust | Has `visibility_modifier` child (any `pub` form) |
| Go | First character of name is uppercase |
| Python | Name does not start with `_` |
| Java | Has `public` modifier inside `modifiers` wrapper node |
| C# | Has `public` or `internal` modifier |
| PHP | Top-level symbol, or method/property with `public` visibility |
| C | No `static` storage class (external linkage by default) |
| C++ | Same as C; class members: `public:` section |

### Import Kinds by Language

| Language | Kind string(s) |
|----------|---------------|
| TypeScript/JS | `"static"`, `"dynamic"`, `"require"`, `"re_export"` |
| Rust | `"use"` |
| Go | `"import"` |
| Python | `"import"`, `"from"` |
| Java | `"import"`, `"static"` |
| C# | `"using"` |
| PHP | `"use"`, `"require"`, `"include"` |
| C/C++ | `"include"` |

---

## Audit System

### Pipeline Categories

| Category | `PipelineSelector` | Per-file pipelines | Cross-file analyzers |
|----------|-------------------|-------------------|----------------------|
| Tech Debt | `TechDebt` | panic, unwrap, unsafe, dead_code | — |
| Complexity | `Complexity` | cyclomatic, cognitive, function_length, comment_ratio | — |
| Code Style | `CodeStyle` | duplicate_code, coupling (heuristic) | DeadExports, DuplicateSymbols |
| Security | `Security` | unsafe_memory, injection, race_conditions | — |
| Scalability | `Scalability` | n_plus_one, sync_blocking_in_async, memory_leak | — |
| Architecture | `Architecture` | module_size, api_surface, barrel_file | CircularDeps, Coupling, DepthChain |

### Architecture Thresholds

| Pattern | Condition | Severity |
|---------|-----------|----------|
| `oversized_module` | ≥ 30 symbols OR ≥ 1000 lines | warning |
| `monolithic_export_surface` | ≥ 20 exported symbols | info |
| `anemic_module` | Exactly 1 symbol (excl. entry files) | info |
| `hub_module_bidirectional` | ≥ 5 intra-project imports | info |
| `barrel_file_reexport` | ≥ 5 re-exports | warning |
| `deep_import_chain` | BFS depth ≥ 6 | info |
| `excessive_public_api` | ≥ 10 symbols AND > 80% exported | info |
| `leaky_abstraction_boundary` | Exported type with public fields | warning |
| `high_efferent_coupling` | Depends on ≥ 10 other files | warning |
| `high_afferent_coupling` | ≥ 15 files depend on it | info |

### Complexity Metrics

**Cyclomatic Complexity** = `1 + decision_points + logical_operators + ternaries`

Decision points: `if`, `for`, `while`, `match`/`switch`, `case`, `catch`, `?:` (language-specific).

**Cognitive Complexity** uses a stack to track nesting depth. Control flow increments increase with each nesting level (`+1 + depth`); flat increments (`else if`, `goto`) always add 1.

Both metrics are computed by `helpers::compute_cyclomatic()` and `helpers::compute_cognitive()` using a `ControlFlowConfig` struct so the same logic handles all languages with minimal variation.

---

## Performance Decisions

| Decision | Rationale |
|----------|-----------|
| In-memory `Arc<str>` workspace | One disk read per file; all subsequent accesses are memory reads. Trades RAM for I/O latency. |
| `rayon` parallel file processing | CPU-bound parsing scales linearly with core count. Queries on large codebases complete in < 1 s. |
| 4 MB rayon stack per thread | Deep ASTs can overflow the default 8 MB stack. 4 MB is sufficient with stack-based iteration helpers. |
| Pre-compile `Arc<Query>` per language | Tree-sitter query compilation is expensive. One compile per language per invocation, shared read-only. |
| Lazy globset compilation | `GlobSet` is built once per query invocation from the `files` / `files_exclude` patterns. |
| `unstable_sort` for summary | Findings are not required to have a stable order; `sort_unstable_by` avoids extra allocations. |
| 64-semaphore S3 download | Bounds concurrent HTTP connections to prevent exhausting OS file descriptors or S3 throttling. |
| Streaming tree-sitter iterator | `QueryMatches` implements `StreamingIterator` (not `Iterator`) in tree-sitter 0.25. Avoids buffering all matches. |
| Single-pass summary aggregation | Summary is computed in one `HashMap` pass over findings; no second sort or extra allocation. |

---

## Key Trade-offs and Design Decisions

### On-Demand Parsing vs. Pre-indexing

virgil-cli parses on demand. There is no background daemon. This means:
- **Pro:** Zero setup time, no stale index, works on any directory.
- **Con:** Cold query latency scales with file count. Mitigated by scoping queries with `files` globs and using S3/in-memory workspace for hot paths.

The server mode (`serve`) is the bridge for latency-sensitive use: parse once, serve many.

### S3 Bypasses the Registry

`--s3` completely sidesteps `projects create`. A minimal synthetic `ProjectEntry` with dummy values is constructed so `query_engine::execute()` can reuse the same code path. The S3 workspace root is set to `s3://bucket/prefix`; `execute_read()`'s disk fallback is guarded by `root.exists()` to prevent any filesystem access.

### Import Kind as Free-Form String

`ImportInfo.kind` is `String`, not an enum. This allows PHP to use `"require"` and `"include"`, C to use `"include"`, Rust to use `"use"`, and TypeScript to use `"dynamic"` without requiring a central type change for every new language.

### Per-File vs. Cross-File Analysis Split

`Pipeline` operates on a single file's AST. `ProjectAnalyzer` operates on the complete `ProjectIndex`. This split is intentional:
- Per-file pipelines run purely in parallel with no shared state.
- Cross-file analyzers need the complete dependency graph. Building the index adds overhead, so it is only done for categories that need it (architecture, code-style).

### Circular Dependency Detection Scope

The `Pipeline::check()` interface operates on one file at a time. True cycle detection requires the full graph. To bridge this gap, architecture pipelines use a **per-file proxy** (fan-out counting as a cycle proxy) for the per-file pass, while `CircularDepsAnalyzer` uses the full `ProjectIndex` graph with Tarjan SCC for accurate detection.

### `.h` Files Map to C

`.h` files are treated as C, not C++. C++ headers should use `.hpp`, `.hxx`, or `.hh`. This is a deliberate simplification.

### Python Decorated Definitions

`decorated_definition` nodes wrap the actual `function_definition` or `class_definition`. Extraction unwraps the decorator to get the inner node, and the bare inner definition is skipped if its parent is `decorated_definition` — preventing duplicate symbol entries.

### Go Exports via Naming Convention

Go has no `public` keyword. A symbol is exported if and only if its first character is uppercase. This convention is enforced by the Go compiler; virgil-cli mirrors it.

### Server Binds to `127.0.0.1` by Default

No authentication is implemented in server mode. Binding to localhost prevents accidental network exposure. Override with `--host 0.0.0.0` when network access is needed (e.g., inside a container).

### Ready Signal on Stdout, Diagnostics on Stderr

The calling process (Virgil Live) reads `stdout` to detect when the server is ready: `{"ready": true, "port": N}`. All progress messages, warnings, and errors go to `stderr`. This separation allows a supervisor to reliably detect readiness without parsing noisy diagnostic output.
