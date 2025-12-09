# Abraham Brainstorming

A polyglot CLI application: one spec, many implementations, unified tests.

---

## Project Requirements

Abraham is a polyglot project: a single CLI application implemented in multiple
programming languages. All implementations share a common test suite written in
Nushell, ensuring behavioral parity across languages.

### Core Requirements

| Requirement | Description |
|-------------|-------------|
| **Modern CLI** | Subcommands, flags, help text, version info |
| **Configuration** | Config file (JSON) + environment variable overrides |
| **Database** | SQLite for persistent storage |
| **JSON** | Parse and serialize JSON for data exchange |
| **Design Patterns** | Showcase Strategy and Composite patterns naturally |
| **Modularity** | Phased implementation, features can be added incrementally |
| **Colored Logging** | Structured log output to stderr with color |
| **Pretty Output** | Colored tables and formatted display to stdout |

### Repository Structure

```
abraham/
├── README.md
├── docs/
│   └── BRAINSTORMING.md     # This document
├── test/
│   ├── test.nu              # Main test runner: ./test/test.nu <lang>
│   ├── lib/
│   │   └── assert.nu        # Assertion library
│   └── cases/               # Test suites by feature
│       ├── cli.nu
│       ├── config.nu
│       ├── crud.nu
│       └── ...
├── rust/                    # Rust implementation
├── go/                      # Go implementation
├── python/                  # Python implementation
├── typescript/              # TypeScript implementation
└── <lang>/                  # Additional implementations
```

### Testing Philosophy

- **Black-box testing**: Tests invoke the compiled binary, not internal APIs
- **Language-agnostic**: Same test suite runs against every implementation
- **Single command**: `./test/test.nu <lang>` tests any implementation

---

## Table of Contents

1. [Domain: Task/Project Manager](#domain-taskproject-manager)
2. [Implementation Phases](#implementation-phases)
3. [Test Framework](#test-framework)

---

## Domain: Task/Project Manager

### Why This Domain?

- **Universally understood**: Everyone knows what tasks and projects are
- **Rich enough for patterns**: Natural hierarchies and varying behaviors
- **Practical SQLite use**: CRUD operations, queries, relationships
- **Clear CLI mapping**: Actions map naturally to subcommands

### Data Model

```
Project
├── id: integer (primary key)
├── name: string
├── status: enum (active, archived)
├── created_at: timestamp
└── updated_at: timestamp

Task
├── id: integer (primary key)
├── project_id: integer (foreign key, nullable)
├── parent_id: integer (self-reference, nullable) ──► Composite Pattern
├── title: string
├── description: string (nullable)
├── status: enum (pending, in_progress, done, cancelled)
├── priority: enum (low, medium, high, urgent)
├── due_date: date (nullable)
├── created_at: timestamp
└── updated_at: timestamp
```

### Composite Pattern: Task Hierarchy

Tasks can contain subtasks, forming a tree:

```
Task: "Release v1.0"
├── Task: "Write documentation"
│   ├── Task: "API reference"
│   └── Task: "User guide"
├── Task: "Fix critical bugs"
│   ├── Task: "Memory leak in parser"
│   └── Task: "Crash on empty input"
└── Task: "Update dependencies"
```

**Behaviors that cascade down the tree:**
- Completing a parent marks all children complete
- Deleting a parent deletes all descendants
- Progress calculation aggregates child status
- Blocking: parent can't complete until children complete

**Interface concept (pseudocode):**

```
interface TaskNode {
    fn id() -> int
    fn title() -> string
    fn status() -> Status
    fn children() -> list<TaskNode>
    fn add_child(task: TaskNode)
    fn remove_child(id: int)
    fn progress() -> float  // 0.0 to 1.0, aggregates children
}
```

### Strategy Pattern: Pluggable Behaviors

#### 1. Output Formatting Strategy

Different ways to render task lists:

| Strategy | Description |
|----------|-------------|
| `TableFormatter` | Pretty ASCII/Unicode tables with colors |
| `JsonFormatter` | Machine-readable JSON output |
| `TreeFormatter` | Indented tree view for hierarchies |
| `CompactFormatter` | One-line-per-task, minimal |
| `MarkdownFormatter` | Markdown-formatted for export |

**Interface:**

```
interface OutputFormatter {
    fn format(tasks: list<Task>) -> string
}
```

**CLI usage:**

```bash
abraham task list --format table
abraham task list --format json
abraham task list --format tree
```

#### 2. Priority/Sorting Strategy

Different algorithms for ordering tasks:

| Strategy | Description |
|----------|-------------|
| `ByPriority` | Urgent > High > Medium > Low |
| `ByDueDate` | Earliest due first, null last |
| `ByCreated` | Oldest first |
| `Eisenhower` | Urgent+Important matrix quadrants |
| `WeightedScore` | Composite score from multiple factors |

**Interface:**

```
interface SortStrategy {
    fn sort(tasks: list<Task>) -> list<Task>
}
```

#### 3. Filter Strategy

Different ways to filter task lists:

| Strategy | Description |
|----------|-------------|
| `ByStatus` | Filter by pending/done/etc |
| `ByProject` | Filter by project membership |
| `ByDueRange` | Due within date range |
| `Overdue` | Past due date, not done |
| `Composite` | Combine multiple filters (AND/OR) |

### CLI Command Structure

```
abraham [global-options] <command> [command-options]

Global Options:
    --config <path>     Config file path (default: ~/.config/abraham/config.json)
    --db <path>         Database path (default: ~/.local/share/abraham/data.db)
    -v, --verbose       Increase log verbosity (can repeat: -vvv)
    --no-color          Disable colored output
    --help, -h          Show help
    --version           Show version

Commands:

  init
      Initialize database and create default config.
      Options:
          --force       Overwrite existing database

  config
      Manage configuration.
      Subcommands:
          get <key>             Get a config value
          set <key> <value>     Set a config value
          list                  List all config values
          path                  Show config file path

  project
      Manage projects.
      Subcommands:
          create <name>         Create a new project
          list                  List all projects
          show <id>             Show project details with tasks
          rename <id> <name>    Rename a project
          archive <id>          Archive a project
          delete <id>           Delete project (must be empty or use --force)

  task
      Manage tasks.
      Subcommands:
          add <title>           Create a new task
              --project, -p <id>    Assign to project
              --parent <id>         Make subtask of another task
              --priority <level>    Set priority (low|medium|high|urgent)
              --due <date>          Set due date (YYYY-MM-DD)
              --description, -d <text>
          
          list                  List tasks
              --project, -p <id>    Filter by project
              --status <s>          Filter by status
              --priority <p>        Filter by priority
              --due-before <date>   Due before date
              --due-after <date>    Due after date
              --overdue             Show only overdue tasks
              --format <fmt>        Output format (table|json|tree|compact)
              --sort <strategy>     Sort strategy (priority|due|created)
          
          show <id>             Show task details (with subtasks)
          
          edit <id>             Edit a task
              --title <text>
              --description <text>
              --priority <level>
              --due <date>
              --status <status>
          
          done <id>             Mark task as done
              --recursive, -r       Also mark all subtasks done
          
          delete <id>           Delete a task
              --recursive, -r       Also delete all subtasks

  export
      Export data.
      Subcommands:
          json                  Export all data as JSON
              --output, -o <file>   Output file (default: stdout)
              --pretty              Pretty-print JSON
          
          csv                   Export tasks as CSV
              --output, -o <file>   Output file (default: stdout)

  import
      Import data.
      Subcommands:
          json <file>           Import from JSON
              --merge               Merge with existing data
              --replace             Replace all existing data
```

### Configuration

**Config file structure (`config.json`):**

```json
{
    "database": {
        "path": "~/.local/share/abraham/data.db"
    },
    "defaults": {
        "priority": "medium",
        "format": "table",
        "sort": "priority"
    },
    "display": {
        "colors": true,
        "unicode": true,
        "date_format": "%Y-%m-%d"
    },
    "logging": {
        "level": "info"
    }
}
```

**Environment variable overrides:**

| Variable | Overrides |
|----------|-----------|
| `ABRAHAM_CONFIG` | Config file path |
| `ABRAHAM_DB` | Database path |
| `ABRAHAM_NO_COLOR` | Disable colors (if set) |
| `ABRAHAM_LOG_LEVEL` | Log level |

**Priority: CLI flags > Environment > Config file > Defaults**

### HTTP API

Abraham provides an optional HTTP server exposing a REST API for tasks and
projects. This enables integration with web frontends, mobile apps, and
external automation tools.

#### Starting the Server

```
abraham serve [options]

Options:
    --port, -p <port>     Port to listen on (default: 8080)
    --host <host>         Host to bind to (default: 127.0.0.1)
    --db <path>           Database path (overrides config)
    --cors                Enable CORS headers for browser access
    --read-only           Disable write operations (GET only)
```

**Examples:**

```bash
# Start server on default port
abraham serve

# Start on custom port, allow external access
abraham serve --port 3000 --host 0.0.0.0

# Read-only mode for public access
abraham serve --read-only --cors
```

#### API Endpoints

Base URL: `http://{host}:{port}/api/v1`

##### Projects

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/projects` | List all projects |
| `POST` | `/projects` | Create a new project |
| `GET` | `/projects/{id}` | Get project by ID |
| `PUT` | `/projects/{id}` | Update project |
| `DELETE` | `/projects/{id}` | Delete project |
| `GET` | `/projects/{id}/tasks` | List tasks in project |

##### Tasks

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/tasks` | List all tasks |
| `POST` | `/tasks` | Create a new task |
| `GET` | `/tasks/{id}` | Get task by ID |
| `PUT` | `/tasks/{id}` | Update task |
| `DELETE` | `/tasks/{id}` | Delete task |
| `GET` | `/tasks/{id}/subtasks` | List subtasks |
| `POST` | `/tasks/{id}/done` | Mark task as done |

#### Request/Response Formats

All requests and responses use JSON with `Content-Type: application/json`.

##### Create Project

```http
POST /api/v1/projects
Content-Type: application/json

{
    "name": "Home Renovation"
}
```

Response (201 Created):

```json
{
    "id": 1,
    "name": "Home Renovation",
    "status": "active",
    "created_at": "2025-01-15T10:30:00Z",
    "updated_at": "2025-01-15T10:30:00Z"
}
```

##### Create Task

```http
POST /api/v1/tasks
Content-Type: application/json

{
    "title": "Paint living room",
    "project_id": 1,
    "priority": "high",
    "due_date": "2025-02-01",
    "description": "Use the blue paint from the garage"
}
```

Response (201 Created):

```json
{
    "id": 1,
    "title": "Paint living room",
    "project_id": 1,
    "parent_id": null,
    "status": "pending",
    "priority": "high",
    "due_date": "2025-02-01",
    "description": "Use the blue paint from the garage",
    "created_at": "2025-01-15T10:35:00Z",
    "updated_at": "2025-01-15T10:35:00Z"
}
```

##### Create Subtask

```http
POST /api/v1/tasks
Content-Type: application/json

{
    "title": "Buy paint supplies",
    "parent_id": 1,
    "priority": "medium"
}
```

##### List Tasks with Filters

```http
GET /api/v1/tasks?status=pending&priority=high&project_id=1&sort=due
```

Query parameters:

| Parameter | Description |
|-----------|-------------|
| `status` | Filter by status (pending, in_progress, done, cancelled) |
| `priority` | Filter by priority (low, medium, high, urgent) |
| `project_id` | Filter by project |
| `parent_id` | Filter by parent task (use `null` for root tasks) |
| `due_before` | Filter by due date (ISO 8601) |
| `due_after` | Filter by due date (ISO 8601) |
| `overdue` | Filter overdue tasks (true/false) |
| `sort` | Sort order (priority, due, created) |
| `include_subtasks` | Include subtask tree (true/false) |

##### Update Task

```http
PUT /api/v1/tasks/1
Content-Type: application/json

{
    "title": "Paint living room blue",
    "priority": "urgent",
    "status": "in_progress"
}
```

Only provided fields are updated (partial update).

##### Mark Task Done

```http
POST /api/v1/tasks/1/done
Content-Type: application/json

{
    "recursive": true
}
```

Marks task and optionally all subtasks as done.

##### Delete Task

```http
DELETE /api/v1/tasks/1?recursive=true
```

#### Error Responses

Errors return appropriate HTTP status codes with a JSON body:

```json
{
    "error": {
        "code": "NOT_FOUND",
        "message": "Task with ID 42 not found"
    }
}
```

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid request body or parameters |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `CONFLICT` | Conflict (e.g., deleting project with tasks) |
| 422 | `VALIDATION_ERROR` | Validation failed |
| 500 | `INTERNAL_ERROR` | Server error |

#### Health Check

```http
GET /api/v1/health
```

Response:

```json
{
    "status": "ok",
    "database": "connected",
    "version": "0.1.0"
}
```

#### Configuration

Server settings in `config.json`:

```json
{
    "server": {
        "host": "127.0.0.1",
        "port": 8080,
        "cors": false,
        "read_only": false
    }
}
```

Environment variable overrides:

| Variable | Description |
|----------|-------------|
| `ABRAHAM_SERVER_HOST` | Bind host |
| `ABRAHAM_SERVER_PORT` | Bind port |
| `ABRAHAM_SERVER_CORS` | Enable CORS (true/false) |

---

## Implementation Phases

Each phase builds on the previous, allowing incremental implementation.

### Phase 1: CLI Foundation

**Goal:** Parse commands, show help, colored output to stderr.

**Features:**
- [ ] Parse global options and subcommands
- [ ] Implement `--help` for all commands
- [ ] Implement `--version`
- [ ] Colored log output to stderr (error, warn, info, debug)
- [ ] Respect `--no-color` and `NO_COLOR` environment variable

**Exit codes:**
- `0` - Success
- `1` - General error
- `2` - Invalid arguments / usage error

**Test coverage:**
- Help text appears for each command
- Version output format
- Invalid command produces error + usage hint
- Log levels filter correctly
- Colors disabled with flag/env

### Phase 2: Configuration

**Goal:** Load config from file and environment.

**Features:**
- [ ] Read JSON config file
- [ ] Default config location (`~/.config/abraham/config.json`)
- [ ] Override with `--config` flag
- [ ] Override with environment variables
- [ ] `config get`, `config set`, `config list` commands
- [ ] Create default config if missing

**Test coverage:**
- Config file is read correctly
- Missing config uses defaults
- Env vars override file values
- CLI flags override env vars
- `config set` persists changes
- Invalid config produces clear error

### Phase 3: Database Setup

**Goal:** Initialize and connect to SQLite database.

**Features:**
- [ ] `init` command creates database with schema
- [ ] Migrations support for schema evolution
- [ ] Database path from config/env/flag
- [ ] Connection pooling (if applicable to language)
- [ ] Graceful handling of locked database

**Schema (initial):**

```sql
CREATE TABLE projects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER REFERENCES projects(id),
    parent_id INTEGER REFERENCES tasks(id),
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    priority TEXT NOT NULL DEFAULT 'medium',
    due_date TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_parent ON tasks(parent_id);
CREATE INDEX idx_tasks_status ON tasks(status);
```

**Test coverage:**
- `init` creates database file
- `init --force` overwrites existing
- Schema is correct
- Migrations apply cleanly

### Phase 4: Basic CRUD Operations

**Goal:** Create, read, update, delete for projects and tasks.

**Features:**
- [ ] `project create`, `project list`, `project show`, `project delete`
- [ ] `task add`, `task list`, `task show`, `task edit`, `task delete`
- [ ] Basic table output for lists
- [ ] ID-based lookups

**Test coverage:**
- Create and retrieve project
- Create and retrieve task
- Update task fields
- Delete removes from database
- List shows all items
- Show displays single item details
- Foreign key constraints enforced

### Phase 5: Task Hierarchy (Composite Pattern)

**Goal:** Implement parent-child relationships for tasks.

**Features:**
- [ ] `task add --parent <id>` creates subtask
- [ ] `task show <id>` displays subtask tree
- [ ] `task list --format tree` shows hierarchy
- [ ] `task done --recursive` completes subtasks
- [ ] `task delete --recursive` removes subtasks
- [ ] Progress calculation from subtask status

**Test coverage:**
- Create nested tasks (3+ levels deep)
- Tree display is correctly indented
- Recursive done marks all descendants
- Recursive delete removes all descendants
- Cannot make task its own ancestor (cycle detection)
- Progress aggregates correctly

### Phase 6: Output Formatting (Strategy Pattern)

**Goal:** Multiple output formats via pluggable formatters.

**Features:**
- [ ] `--format table` (default) - colored ASCII tables
- [ ] `--format json` - machine-readable
- [ ] `--format tree` - hierarchical view
- [ ] `--format compact` - minimal one-liner per task
- [ ] Column alignment and truncation
- [ ] Unicode box-drawing (respects config)

**Test coverage:**
- Each format produces expected output structure
- JSON is valid and parseable
- Table columns align correctly
- Tree indentation is correct
- Colors respect `--no-color`

### Phase 7: Sorting and Filtering (Strategy Pattern)

**Goal:** Flexible querying with pluggable strategies.

**Features:**
- [ ] `--sort priority|due|created`
- [ ] `--status pending|done|all`
- [ ] `--priority low|medium|high|urgent`
- [ ] `--overdue` filter
- [ ] `--due-before` and `--due-after`
- [ ] Combine multiple filters

**Test coverage:**
- Each sort order is correct
- Filters reduce result set correctly
- Combined filters work (AND logic)
- Empty results handled gracefully

### Phase 8: Import/Export

**Goal:** Data portability via JSON.

**Features:**
- [ ] `export json` - full database dump
- [ ] `export json --pretty` - formatted output
- [ ] `export csv` - tasks only, flat
- [ ] `import json --merge` - add to existing
- [ ] `import json --replace` - overwrite all
- [ ] Validation of import data

**Test coverage:**
- Export produces valid JSON/CSV
- Import restores data correctly
- Round-trip: export then import yields same data
- Invalid import data produces clear error
- Merge preserves existing data
- Replace removes old data

### Phase 9: HTTP API

**Goal:** REST API for external integration.

**Features:**
- [ ] `serve` command starts HTTP server
- [ ] `--port`, `--host`, `--cors`, `--read-only` flags
- [ ] `/api/v1/projects` CRUD endpoints
- [ ] `/api/v1/tasks` CRUD endpoints
- [ ] Query parameter filtering (status, priority, project, etc.)
- [ ] `/api/v1/tasks/{id}/done` action endpoint
- [ ] `/api/v1/health` health check
- [ ] JSON error responses with appropriate status codes
- [ ] Graceful shutdown on SIGINT/SIGTERM

**Test coverage:**
- Server starts and responds to health check
- Create project via POST, verify via GET
- Create task via POST, verify via GET
- Update task via PUT
- Delete task via DELETE
- Filter tasks via query parameters
- Subtask creation with parent_id
- Mark done with recursive flag
- Error responses for invalid requests
- Error responses for not found
- CORS headers when enabled
- Read-only mode rejects writes

### Phase Summary Table

| Phase | Focus | Patterns | Deps |
|-------|-------|----------|------|
| 1 | CLI Foundation | - | - |
| 2 | Configuration | - | 1 |
| 3 | Database Setup | - | 2 |
| 4 | Basic CRUD | - | 3 |
| 5 | Task Hierarchy | Composite | 4 |
| 6 | Output Formatting | Strategy | 4 |
| 7 | Sorting/Filtering | Strategy | 4 |
| 8 | Import/Export | - | 4 |
| 9 | HTTP API | - | 4 |

Phases 5, 6, 7, 8, 9 can be done in parallel after Phase 4.

---

## Test Framework

### Philosophy

- **Black-box testing**: Test the compiled binary, not internals
- **Language-agnostic**: Same tests run against all implementations
- **Declarative**: Tests describe expected behavior, not how to verify
- **Composable**: Build complex tests from simple assertions

### Test Runner: `test/test.nu`

```nu
#!/usr/bin/env nu

# Main entry point
def main [
    lang: string           # Language to test (rust, go, python, etc.)
    --filter: string       # Filter tests by name pattern
    --verbose (-v)         # Show detailed output
] {
    let binary = get-binary $lang
    
    print $"Testing ($lang) implementation: ($binary)"
    
    # Verify binary exists/works
    verify-binary $binary
    
    # Run test suites
    let results = [
        (run-suite "cli" $binary $filter $verbose)
        (run-suite "config" $binary $filter $verbose)
        (run-suite "database" $binary $filter $verbose)
        (run-suite "crud" $binary $filter $verbose)
        (run-suite "hierarchy" $binary $filter $verbose)
        (run-suite "format" $binary $filter $verbose)
        (run-suite "filter" $binary $filter $verbose)
        (run-suite "export" $binary $filter $verbose)
    ]
    
    # Summary
    print-summary $results
    
    # Exit with appropriate code
    if ($results | any { $in.failed > 0 }) {
        exit 1
    }
}

# Get path to binary for a language
def get-binary [lang: string] -> string {
    match $lang {
        "rust" => "./rust/target/release/abraham",
        "go" => "./go/build/abraham",
        "python" => "python ./python/src/abraham",
        "typescript" => "node ./typescript/dist/abraham.js",
        "zig" => "./zig/zig-out/bin/abraham",
        _ => { error make { msg: $"Unknown language: ($lang)" } }
    }
}
```

### Assertion Library: `test/lib/assert.nu`

Core assertions for testing CLI applications:

```nu
# ─────────────────────────────────────────────────────────────
# Exit Code Assertions
# ─────────────────────────────────────────────────────────────

# Assert command exits with expected code
def "assert exit" [
    result: record    # Result from `complete` (has exit_code, stdout, stderr)
    expected: int     # Expected exit code
    --msg: string     # Optional message on failure
] {
    if $result.exit_code != $expected {
        let message = ($msg | default $"Expected exit code ($expected), got ($result.exit_code)")
        fail-assertion $message $result
    }
}

# Assert command succeeds (exit code 0)
def "assert ok" [
    result: record
    --msg: string
] {
    assert exit $result 0 --msg ($msg | default "Expected success (exit 0)")
}

# Assert command fails (exit code != 0)
def "assert fails" [
    result: record
    --msg: string
] {
    if $result.exit_code == 0 {
        fail-assertion ($msg | default "Expected failure (exit != 0)") $result
    }
}

# ─────────────────────────────────────────────────────────────
# Output Content Assertions
# ─────────────────────────────────────────────────────────────

# Assert stdout contains substring
def "assert stdout contains" [
    result: record
    expected: string
    --msg: string
] {
    if not ($result.stdout | str contains $expected) {
        fail-assertion ($msg | default $"Expected stdout to contain: ($expected)") $result
    }
}

# Assert stdout equals exactly (trimmed)
def "assert stdout eq" [
    result: record
    expected: string
    --msg: string
] {
    let actual = ($result.stdout | str trim)
    let expected_trimmed = ($expected | str trim)
    if $actual != $expected_trimmed {
        fail-assertion ($msg | default $"Expected stdout to equal: ($expected_trimmed)") $result
    }
}

# Assert stdout matches regex
def "assert stdout matches" [
    result: record
    pattern: string
    --msg: string
] {
    if not ($result.stdout =~ $pattern) {
        fail-assertion ($msg | default $"Expected stdout to match: ($pattern)") $result
    }
}

# Assert stderr contains substring
def "assert stderr contains" [
    result: record
    expected: string
    --msg: string
] {
    if not ($result.stderr | str contains $expected) {
        fail-assertion ($msg | default $"Expected stderr to contain: ($expected)") $result
    }
}

# Assert stdout is empty
def "assert stdout empty" [
    result: record
    --msg: string
] {
    if ($result.stdout | str trim | str length) > 0 {
        fail-assertion ($msg | default "Expected stdout to be empty") $result
    }
}

# Assert stderr is empty
def "assert stderr empty" [
    result: record
    --msg: string
] {
    if ($result.stderr | str trim | str length) > 0 {
        fail-assertion ($msg | default "Expected stderr to be empty") $result
    }
}

# ─────────────────────────────────────────────────────────────
# JSON Assertions
# ─────────────────────────────────────────────────────────────

# Assert stdout is valid JSON and return parsed value
def "assert json" [
    result: record
    --msg: string
] -> any {
    try {
        $result.stdout | from json
    } catch {
        fail-assertion ($msg | default "Expected valid JSON in stdout") $result
    }
}

# Assert JSON output has expected field value
def "assert json field" [
    result: record
    path: string       # Dot-separated path, e.g., "data.tasks.0.title"
    expected: any
    --msg: string
] {
    let json = (assert json $result)
    let actual = ($json | get $path)
    if $actual != $expected {
        fail-assertion ($msg | default $"Expected ($path) to be ($expected), got ($actual)") $result
    }
}

# Assert JSON array has expected length
def "assert json length" [
    result: record
    path: string       # Path to array
    expected: int
    --msg: string
] {
    let json = (assert json $result)
    let array = ($json | get $path)
    let actual = ($array | length)
    if $actual != $expected {
        fail-assertion ($msg | default $"Expected ($path) to have length ($expected), got ($actual)") $result
    }
}

# ─────────────────────────────────────────────────────────────
# Table Output Assertions
# ─────────────────────────────────────────────────────────────

# Assert output contains a row with given values
def "assert table row" [
    result: record
    values: list<string>    # Values that should appear in the same row
    --msg: string
] {
    let lines = ($result.stdout | lines)
    let found = ($lines | any { |line|
        $values | all { |val| $line | str contains $val }
    })
    if not $found {
        fail-assertion ($msg | default $"Expected table row containing: ($values)") $result
    }
}

# Assert output has expected number of data rows (excluding header)
def "assert table rows" [
    result: record
    expected: int
    --header-lines: int = 1     # Lines before data (header + separator)
    --msg: string
] {
    let lines = ($result.stdout | lines | where { $in | str trim | str length > 0 })
    let data_lines = ($lines | length) - $header_lines
    if $data_lines != $expected {
        fail-assertion ($msg | default $"Expected ($expected) data rows, got ($data_lines)") $result
    }
}

# ─────────────────────────────────────────────────────────────
# File System Assertions
# ─────────────────────────────────────────────────────────────

# Assert file exists
def "assert file exists" [
    path: string
    --msg: string
] {
    if not ($path | path exists) {
        fail-assertion ($msg | default $"Expected file to exist: ($path)")
    }
}

# Assert file does not exist
def "assert file not exists" [
    path: string
    --msg: string
] {
    if ($path | path exists) {
        fail-assertion ($msg | default $"Expected file to not exist: ($path)")
    }
}

# Assert file contains string
def "assert file contains" [
    path: string
    expected: string
    --msg: string
] {
    assert file exists $path
    let content = (open $path | into string)
    if not ($content | str contains $expected) {
        fail-assertion ($msg | default $"Expected file ($path) to contain: ($expected)")
    }
}

# ─────────────────────────────────────────────────────────────
# Database Assertions (via CLI queries)
# ─────────────────────────────────────────────────────────────

# Assert task exists with given title
def "assert task exists" [
    binary: string
    db: string
    title: string
    --msg: string
] {
    let result = (run $binary ["task" "list" "--format" "json" "--db" $db])
    assert ok $result
    let tasks = (assert json $result)
    let found = ($tasks | any { $in.title == $title })
    if not $found {
        fail-assertion ($msg | default $"Expected task with title: ($title)") $result
    }
}

# Assert task count equals expected
def "assert task count" [
    binary: string
    db: string
    expected: int
    --status: string
    --msg: string
] {
    mut args = ["task" "list" "--format" "json" "--db" $db]
    if $status != null {
        $args = ($args | append ["--status" $status])
    }
    let result = (run $binary $args)
    assert ok $result
    let tasks = (assert json $result)
    let actual = ($tasks | length)
    if $actual != $expected {
        fail-assertion ($msg | default $"Expected ($expected) tasks, got ($actual)") $result
    }
}

# ─────────────────────────────────────────────────────────────
# Helpers
# ─────────────────────────────────────────────────────────────

# Run a command and capture result
def run [
    binary: string
    args: list<string>
] -> record {
    ^$binary ...$args | complete
}

# Fail with formatted error message
def fail-assertion [
    message: string
    result?: record
] {
    mut output = $"ASSERTION FAILED: ($message)"
    if $result != null {
        $output = $"($output)\n  stdout: ($result.stdout | str trim)\n  stderr: ($result.stderr | str trim)\n  exit: ($result.exit_code)"
    }
    error make { msg: $output }
}
```

### Test Structure: Example Suite

```nu
# test/cases/crud.nu

use ../lib/assert.nu *

# Test fixture: fresh database for each test
def with-temp-db [block: closure] {
    let db = (mktemp -t "abraham_test_XXXXXX.db")
    try {
        do $block $db
    } finally {
        rm -f $db
    }
}

# ─────────────────────────────────────────────────────────────
# Project Tests
# ─────────────────────────────────────────────────────────────

def "test project create" [binary: string] {
    with-temp-db { |db|
        # Initialize database
        run $binary ["init" "--db" $db] | assert ok
        
        # Create a project
        let result = (run $binary ["project" "create" "My Project" "--db" $db])
        assert ok $result
        assert stdout contains $result "Created project"
        
        # Verify it appears in list
        let list_result = (run $binary ["project" "list" "--db" $db])
        assert ok $list_result
        assert table row $list_result ["My Project" "active"]
    }
}

def "test project create duplicate name allowed" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["project" "create" "Duplicate" "--db" $db] | assert ok
        run $binary ["project" "create" "Duplicate" "--db" $db] | assert ok
        
        # Both should exist
        let result = (run $binary ["project" "list" "--format" "json" "--db" $db])
        assert json length $result "" 2
    }
}

def "test project delete empty" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["project" "create" "ToDelete" "--db" $db] | assert ok
        
        let result = (run $binary ["project" "delete" "1" "--db" $db])
        assert ok $result
        
        # Verify gone
        let list_result = (run $binary ["project" "list" "--format" "json" "--db" $db])
        assert json length $list_result "" 0
    }
}

def "test project delete with tasks requires force" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["project" "create" "HasTasks" "--db" $db] | assert ok
        run $binary ["task" "add" "A task" "--project" "1" "--db" $db] | assert ok
        
        # Should fail without --force
        let result = (run $binary ["project" "delete" "1" "--db" $db])
        assert fails $result
        assert stderr contains $result "has tasks"
        
        # Should succeed with --force
        let force_result = (run $binary ["project" "delete" "1" "--force" "--db" $db])
        assert ok $force_result
    }
}

# ─────────────────────────────────────────────────────────────
# Task Tests
# ─────────────────────────────────────────────────────────────

def "test task add minimal" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        
        let result = (run $binary ["task" "add" "Buy groceries" "--db" $db])
        assert ok $result
        assert stdout contains $result "Created task"
        
        assert task exists $binary $db "Buy groceries"
    }
}

def "test task add with all options" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["project" "create" "Home" "--db" $db] | assert ok
        
        let result = (run $binary [
            "task" "add" "Clean garage"
            "--project" "1"
            "--priority" "high"
            "--due" "2025-12-31"
            "--description" "Before new year"
            "--db" $db
        ])
        assert ok $result
        
        # Verify all fields
        let show = (run $binary ["task" "show" "1" "--format" "json" "--db" $db])
        assert ok $show
        assert json field $show "title" "Clean garage"
        assert json field $show "project_id" 1
        assert json field $show "priority" "high"
        assert json field $show "due_date" "2025-12-31"
        assert json field $show "description" "Before new year"
    }
}

def "test task done marks complete" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["task" "add" "Finish report" "--db" $db] | assert ok
        
        let result = (run $binary ["task" "done" "1" "--db" $db])
        assert ok $result
        
        # Verify status
        let show = (run $binary ["task" "show" "1" "--format" "json" "--db" $db])
        assert json field $show "status" "done"
    }
}

def "test task list filtering" [binary: string] {
    with-temp-db { |db|
        run $binary ["init" "--db" $db] | assert ok
        run $binary ["task" "add" "Low task" "--priority" "low" "--db" $db] | assert ok
        run $binary ["task" "add" "High task" "--priority" "high" "--db" $db] | assert ok
        run $binary ["task" "add" "Done task" "--db" $db] | assert ok
        run $binary ["task" "done" "3" "--db" $db] | assert ok
        
        # Filter by priority
        let high = (run $binary ["task" "list" "--priority" "high" "--format" "json" "--db" $db])
        assert json length $high "" 1
        
        # Filter by status
        let pending = (run $binary ["task" "list" "--status" "pending" "--format" "json" "--db" $db])
        assert json length $pending "" 2
        
        let done = (run $binary ["task" "list" "--status" "done" "--format" "json" "--db" $db])
        assert json length $done "" 1
    }
}
```

### Running Tests

```bash
# Test a specific language
./test/test.nu rust

# Test with filter
./test/test.nu go --filter "project"

# Verbose mode
./test/test.nu python --verbose

# Test all languages
for lang in rust go python typescript zig; do
    ./test/test.nu $lang
done
```

### Assertion Summary Table

| Category | Assertion | Description |
|----------|-----------|-------------|
| **Exit Code** | `assert ok` | Exit code is 0 |
| | `assert fails` | Exit code is not 0 |
| | `assert exit` | Exit code equals expected |
| **Stdout** | `assert stdout contains` | Contains substring |
| | `assert stdout eq` | Equals exactly (trimmed) |
| | `assert stdout matches` | Matches regex pattern |
| | `assert stdout empty` | Is empty |
| **Stderr** | `assert stderr contains` | Contains substring |
| | `assert stderr empty` | Is empty |
| **JSON** | `assert json` | Is valid JSON, returns parsed |
| | `assert json field` | Field at path equals value |
| | `assert json length` | Array at path has length |
| **Table** | `assert table row` | Row contains all values |
| | `assert table rows` | Has N data rows |
| **Files** | `assert file exists` | File exists |
| | `assert file not exists` | File does not exist |
| | `assert file contains` | File contains string |
| **Domain** | `assert task exists` | Task with title exists |
| | `assert task count` | Task count matches |

---

## Language Implementations

Each implementation should be idiomatic to its language while maintaining CLI
compatibility. Languages may add extended functionality where their ecosystem
provides natural advantages.

### Go

**Why Go fits well:**
- Excellent CLI tooling ecosystem
- Fast compilation, single binary output
- Strong SQLite support via CGO
- Simple concurrency for potential future features

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `github.com/spf13/cobra` | Industry standard, subcommands, completions |
| Config | `github.com/spf13/viper` | Pairs with Cobra, env/file/flag merging |
| SQLite | `github.com/mattn/go-sqlite3` | CGO-based, full SQLite support |
| SQLite (no CGO) | `modernc.org/sqlite` | Pure Go alternative |
| HTTP | `net/http` + `github.com/go-chi/chi` | Stdlib router + chi for REST |
| HTTP (alt) | `github.com/gin-gonic/gin` | Full-featured, fast |
| Logging | `log/slog` | Stdlib structured logging (Go 1.21+) |
| Colors | `github.com/fatih/color` | Simple terminal colors |
| Tables | `github.com/rodaine/table` | Lightweight table formatting |
| JSON | `encoding/json` | Stdlib, sufficient for needs |

**Project Structure:**

```
go/
├── go.mod
├── go.sum
├── main.go                  # Entry point
├── cmd/
│   ├── root.go              # Root command, global flags
│   ├── init.go              # init command
│   ├── config.go            # config subcommands
│   ├── project.go           # project subcommands
│   ├── task.go              # task subcommands
│   └── export.go            # export/import commands
├── internal/
│   ├── config/
│   │   └── config.go        # Config loading, merging
│   ├── db/
│   │   ├── db.go            # Connection, migrations
│   │   ├── project.go       # Project repository
│   │   └── task.go          # Task repository
│   ├── model/
│   │   ├── project.go       # Project domain type
│   │   └── task.go          # Task domain type, composite
│   ├── format/
│   │   ├── formatter.go     # Strategy interface
│   │   ├── table.go
│   │   ├── json.go
│   │   └── tree.go
│   └── log/
│       └── log.go           # Colored stderr logging
└── build/
    └── abraham              # Output binary
```

**Idiomatic Patterns:**
- Interfaces for Strategy pattern (Formatter, Sorter, Filter)
- Embedded structs for Composite (Task contains []Task)
- Context propagation for cancellation
- Table-driven tests in `_test.go` files

**Go-Specific Extensions:**
- Shell completions via Cobra (`abraham completion bash/zsh/fish`)
- `--json` flag on all commands for scripting (Cobra makes this trivial)
- Potential: concurrent task operations

---

### Rust

**Why Rust fits well:**
- Excellent CLI libraries with derive macros
- Strong type system for domain modeling
- Enums with data perfect for patterns
- Memory safety, single binary

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `clap` | Derive macros, subcommands, completions |
| Config | `config` | Multi-source config merging |
| SQLite | `rusqlite` | Mature, well-maintained |
| SQLite (async) | `sqlx` | Compile-time query checking |
| HTTP | `axum` | Modern, tower-based, async |
| HTTP (alt) | `actix-web` | High performance, mature |
| Logging | `tracing` | Structured, async-aware |
| Colors | `owo-colors` | Zero-allocation coloring |
| Tables | `comfy-table` | Feature-rich table formatting |
| JSON | `serde` + `serde_json` | Industry standard |
| Errors | `thiserror` | Derive error types |
| Errors (app) | `anyhow` | Ergonomic error handling |

**Project Structure:**

```
rust/
├── Cargo.toml
├── Cargo.lock
└── src/
    ├── main.rs              # Entry point
    ├── cli.rs               # Clap command definitions
    ├── config.rs            # Config loading
    ├── db/
    │   ├── mod.rs
    │   ├── migrations.rs
    │   ├── project.rs       # Project queries
    │   └── task.rs          # Task queries
    ├── model/
    │   ├── mod.rs
    │   ├── project.rs
    │   └── task.rs          # Task with children (Composite)
    ├── format/
    │   ├── mod.rs           # Formatter trait (Strategy)
    │   ├── table.rs
    │   ├── json.rs
    │   └── tree.rs
    ├── sort/
    │   ├── mod.rs           # Sorter trait (Strategy)
    │   └── strategies.rs
    ├── error.rs             # Custom error types
    └── log.rs               # Logging setup
```

**Idiomatic Patterns:**
- `enum` for status/priority with `#[derive(Serialize, Deserialize)]`
- Traits for Strategy pattern (`trait Formatter`, `trait Sorter`)
- Recursive `Task` struct for Composite pattern
- `Result<T, Error>` throughout, `?` operator
- Builder pattern for complex queries

**Rust-Specific Extensions:**
- Shell completions via Clap (`abraham completions bash`)
- Man page generation via `clap_mangen`
- `--format=ron` output (Rust Object Notation)
- Compile-time SQL validation with `sqlx`

---

### Python

**Why Python fits well:**
- Rapid prototyping
- Rich ecosystem
- Easy to read implementation
- Good for scripting extensions

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `typer` | Modern, type-hint based (builds on Click) |
| Config | `pydantic-settings` | Validation, env support |
| SQLite | `sqlite3` | Stdlib, sufficient |
| SQLite (ORM) | `sqlmodel` | SQLAlchemy + Pydantic |
| HTTP | `fastapi` | Modern async, auto OpenAPI docs |
| HTTP (alt) | `flask` | Simple, synchronous |
| Logging | `rich.logging` | Beautiful logging |
| Colors/Tables | `rich` | Best-in-class terminal formatting |
| JSON | `json` | Stdlib |
| Packaging | `hatch` or `poetry` | Modern project management |

**Project Structure:**

```
python/
├── pyproject.toml
├── src/
│   └── abraham/
│       ├── __init__.py
│       ├── __main__.py      # Entry: python -m abraham
│       ├── cli/
│       │   ├── __init__.py
│       │   ├── main.py      # Typer app, root command
│       │   ├── config.py    # config subcommands
│       │   ├── project.py   # project subcommands
│       │   ├── task.py      # task subcommands
│       │   └── export.py    # export/import
│       ├── config.py        # Settings management
│       ├── db/
│       │   ├── __init__.py
│       │   ├── connection.py
│       │   ├── migrations.py
│       │   └── repository.py
│       ├── model/
│       │   ├── __init__.py
│       │   ├── project.py
│       │   └── task.py      # Pydantic models
│       └── format/
│           ├── __init__.py
│           ├── base.py      # Protocol/ABC (Strategy)
│           ├── table.py
│           ├── json.py
│           └── tree.py
└── tests/                   # Unit tests (pytest)
```

**Idiomatic Patterns:**
- Type hints throughout (Python 3.10+)
- `Protocol` or `ABC` for Strategy pattern
- Dataclasses or Pydantic models for domain types
- Context managers for database connections
- Decorators for command registration (Typer)

**Python-Specific Extensions:**
- REPL mode: `abraham shell` for interactive task management
- Plugin system via entry points
- `--format=yaml` output (PyYAML is ubiquitous)
- Rich markdown rendering for task descriptions

---

### Zig

**Why Zig fits well:**
- No hidden control flow, explicit allocations
- Comptime for zero-cost abstractions
- C interop for SQLite
- Small, fast binaries

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `zig-clap` | Declarative argument parsing |
| SQLite | `sqlite-zig` | Zig bindings to SQLite C API |
| HTTP | `zap` | Blazingly fast, wraps facil.io |
| HTTP (alt) | `std.http` | Stdlib server (basic) |
| JSON | `std.json` | Stdlib JSON parser |
| Logging | Custom | Build on `std.log` |
| Colors | Custom | ANSI escape sequences |
| Tables | Custom | No mature library, implement |

**Project Structure:**

```
zig/
├── build.zig
├── build.zig.zon            # Package dependencies
└── src/
    ├── main.zig             # Entry point
    ├── cli.zig              # Argument parsing
    ├── config.zig           # Config loading
    ├── db/
    │   ├── db.zig           # SQLite connection
    │   ├── migrate.zig
    │   └── repo.zig         # Queries
    ├── model/
    │   ├── project.zig
    │   └── task.zig         # Task struct with children
    ├── format/
    │   ├── format.zig       # Formatter interface (fn ptr)
    │   ├── table.zig
    │   ├── json.zig
    │   └── tree.zig
    └── log.zig              # Colored logging
```

**Idiomatic Patterns:**
- Tagged unions for enums (Status, Priority)
- Function pointers or vtables for Strategy pattern
- Allocator-aware data structures
- `ArrayList(Task)` for children (Composite)
- Comptime validation where possible

**Zig-Specific Extensions:**
- Memory usage statistics (`--debug-alloc`)
- Explicit allocator choice (`--allocator=arena|gpa|c`)
- Minimal binary size focus

---

### Nim

**Why Nim fits well:**
- Python-like syntax, C-like performance
- Powerful macro system
- Good C interop for SQLite
- Single binary output

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `cligen` | Auto-generates CLI from procs |
| CLI (alt) | `docopt.nim` | Docstring-based parsing |
| SQLite | `db_sqlite` | Stdlib database module |
| HTTP | `jester` | Sinatra-like micro framework |
| HTTP (alt) | `prologue` | Full-featured web framework |
| JSON | `std/json` | Stdlib |
| Logging | `std/logging` | Stdlib |
| Colors | `terminal` | Stdlib terminal colors |
| Tables | Custom | Simple implementation |

**Project Structure:**

```
nim/
├── abraham.nimble           # Package manager config
├── src/
│   ├── abraham.nim          # Entry point
│   ├── cli.nim              # Command definitions
│   ├── config.nim           # Config management
│   ├── db/
│   │   ├── db.nim           # Connection
│   │   ├── migrate.nim
│   │   └── repo.nim
│   ├── model/
│   │   ├── project.nim
│   │   └── task.nim         # Object variants
│   └── format/
│       ├── formatter.nim    # Base type + methods
│       ├── table.nim
│       ├── json.nim
│       └── tree.nim
└── tests/
```

**Idiomatic Patterns:**
- Object variants for enums with data
- Method-based dispatch for Strategy (concept or method)
- `ref` objects for Composite tree
- Templates/macros for boilerplate reduction
- `{.raises: [].}` for effect tracking

**Nim-Specific Extensions:**
- Compile-time SQL validation macros
- Hot code reloading during development
- JavaScript target for web version

---

### Crystal

**Why Crystal fits well:**
- Ruby-like syntax, compiled
- Strong type inference
- Null safety
- Good concurrency primitives

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `commander` or `clip` | Command-line parsing |
| CLI (alt) | `admiral` | Object-oriented CLI |
| SQLite | `crystal-sqlite3` | SQLite bindings |
| SQLite (ORM) | `jennifer` | Active Record style |
| HTTP | `kemal` | Sinatra-like, fast |
| HTTP (alt) | `amber` | Full-featured framework |
| JSON | `JSON` (stdlib) | Built-in |
| Logging | `Log` (stdlib) | Built-in structured logging |
| Colors | `colorize` (stdlib) | Built-in |
| Tables | Custom | Simple implementation |

**Project Structure:**

```
crystal/
├── shard.yml                # Dependency manager
├── shard.lock
└── src/
    ├── abraham.cr           # Entry point
    ├── cli/
    │   ├── cli.cr           # Command definitions
    │   ├── config_cmd.cr
    │   ├── project_cmd.cr
    │   └── task_cmd.cr
    ├── config.cr
    ├── db/
    │   ├── db.cr
    │   ├── migrate.cr
    │   └── repo.cr
    ├── model/
    │   ├── project.cr
    │   └── task.cr
    └── format/
        ├── formatter.cr     # Abstract class / module
        ├── table.cr
        ├── json.cr
        └── tree.cr
```

**Idiomatic Patterns:**
- Abstract classes or modules for Strategy pattern
- Classes with `property` for domain models
- `Nil` union types for nullable fields
- Blocks for iteration patterns
- Macros for reducing boilerplate

**Crystal-Specific Extensions:**
- Fiber-based concurrent operations
- Compile-time type checking
- `--release` optimized builds

---

### C++

**Why C++ fits well:**
- Maximum performance
- Rich standard library (C++17/20)
- Mature SQLite integration
- Industry-proven patterns

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `CLI11` | Modern, header-only |
| CLI (alt) | `argparse` | Python argparse-style |
| Config | `nlohmann/json` | Also handles config files |
| SQLite | `SQLiteCpp` | C++ wrapper around SQLite |
| HTTP | `cpp-httplib` | Header-only, simple |
| HTTP (alt) | `drogon` | High-performance async |
| JSON | `nlohmann/json` | Industry standard |
| Logging | `spdlog` | Fast, feature-rich |
| Colors | `fmt` | Has color support |
| Tables | `tabulate` | Header-only tables |
| Build | `CMake` | Standard build system |
| Package | `vcpkg` or `conan` | Dependency management |

**Project Structure:**

```
cpp/
├── CMakeLists.txt
├── vcpkg.json               # Dependencies
└── src/
    ├── main.cpp             # Entry point
    ├── cli/
    │   ├── cli.hpp
    │   ├── cli.cpp
    │   └── commands/
    │       ├── init.cpp
    │       ├── config.cpp
    │       ├── project.cpp
    │       └── task.cpp
    ├── config/
    │   ├── config.hpp
    │   └── config.cpp
    ├── db/
    │   ├── database.hpp
    │   ├── database.cpp
    │   ├── repository.hpp
    │   └── repository.cpp
    ├── model/
    │   ├── project.hpp
    │   ├── task.hpp         # Task with vector<Task>
    │   └── types.hpp        # Enums
    └── format/
        ├── formatter.hpp    # Abstract base class
        ├── table.cpp
        ├── json.cpp
        └── tree.cpp
```

**Idiomatic Patterns:**
- Abstract base classes + virtual for Strategy
- `std::unique_ptr<Task>` / `std::vector<Task>` for Composite
- `std::optional<T>` for nullable fields
- `std::variant` for tagged unions
- RAII for resource management
- Move semantics for performance

**C++-Specific Extensions:**
- `--parallel` for concurrent task operations (C++17 parallel algorithms)
- Memory-mapped database for large datasets
- `constexpr` validation where possible

---

### C

**Why C fits well:**
- Maximum control
- Minimal dependencies
- Direct SQLite API access
- Educational value

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `argtable3` | Feature-rich argument parsing |
| CLI (alt) | `getopt` (POSIX) | Minimal, standard |
| SQLite | `sqlite3` | Direct C API |
| HTTP | `libmicrohttpd` | GNU, lightweight embedded |
| HTTP (alt) | `mongoose` | Single-file, embedded |
| JSON | `cJSON` | Lightweight, common |
| JSON (alt) | `json-c` | More features |
| Logging | Custom | fprintf to stderr |
| Colors | Custom | ANSI escape codes |
| Tables | Custom | Manual formatting |
| Build | `make` or `CMake` | Standard |

**Project Structure:**

```
c/
├── Makefile
├── include/
│   ├── abraham.h            # Main header
│   ├── cli.h
│   ├── config.h
│   ├── db.h
│   ├── model.h
│   ├── format.h
│   └── log.h
└── src/
    ├── main.c               # Entry point
    ├── cli/
    │   ├── cli.c            # Argument parsing
    │   ├── cmd_init.c
    │   ├── cmd_config.c
    │   ├── cmd_project.c
    │   └── cmd_task.c
    ├── config.c
    ├── db/
    │   ├── db.c             # Connection, migrations
    │   └── repo.c           # CRUD operations
    ├── model/
    │   ├── project.c
    │   └── task.c           # Task with linked list children
    └── format/
        ├── format.c         # Function pointer dispatch
        ├── table.c
        ├── json.c
        └── tree.c
```

**Idiomatic Patterns:**
- Function pointers for Strategy pattern
- Structs with pointers for Composite (linked list or dynamic array)
- Tagged unions for enums with data
- Explicit memory management (malloc/free)
- Error codes + out parameters

**C-Specific Extensions:**
- Static builds with musl for portability
- Valgrind-clean memory usage
- Minimal binary size (`-Os`, `strip`)

---

### D

**Why D fits well:**
- C-like performance with modern features
- Excellent metaprogramming
- Good C interop for SQLite
- Garbage collection (optional)

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `darg` | Declarative argument parsing |
| CLI (alt) | `std.getopt` | Stdlib option parsing |
| SQLite | `d2sqlite3` | D bindings |
| HTTP | `vibe.d` | Full-featured async framework |
| HTTP (alt) | `hunt-framework` | High-performance |
| JSON | `std.json` | Stdlib |
| Logging | `std.experimental.logger` | Stdlib |
| Colors | Custom | ANSI sequences |
| Tables | Custom | Format with std.format |
| Build | `dub` | Standard package manager |

**Project Structure:**

```
d/
├── dub.json                 # Package config
└── source/
    ├── app.d                # Entry point
    ├── cli/
    │   ├── cli.d
    │   ├── config_cmd.d
    │   ├── project_cmd.d
    │   └── task_cmd.d
    ├── config.d
    ├── db/
    │   ├── database.d
    │   ├── migrate.d
    │   └── repo.d
    ├── model/
    │   ├── project.d
    │   └── task.d           # Task class with children
    └── format/
        ├── formatter.d      # Interface
        ├── table.d
        ├── json.d
        └── tree.d
```

**Idiomatic Patterns:**
- Interfaces for Strategy pattern
- Classes with garbage collection for Composite
- `Nullable!T` for optional fields
- Ranges for lazy iteration
- UFCS (Uniform Function Call Syntax) for fluent APIs
- Templates for generic code

**D-Specific Extensions:**
- `@nogc` paths for performance-critical code
- Compile-time function evaluation (CTFE)
- BetterC mode for minimal runtime

---

### Ruby

**Why Ruby fits well:**
- Exceptional developer ergonomics
- Rich ecosystem of gems
- Expressive syntax for domain modeling
- Strong metaprogramming capabilities

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `thor` | Industry standard, subcommands |
| CLI (alt) | `dry-cli` | Modern, compositional |
| Config | `anyway_config` | Env + file + defaults |
| SQLite | `sqlite3` | Standard gem |
| SQLite (ORM) | `sequel` | Lightweight, powerful |
| HTTP | `sinatra` | Classic micro framework |
| HTTP (alt) | `roda` | Fast, routing tree |
| JSON | `json` | Stdlib |
| Logging | `logger` | Stdlib |
| Colors | `pastel` | Elegant terminal colors |
| Tables | `tty-table` | Feature-rich tables |
| Packaging | `bundler` | Dependency management |

**Project Structure:**

```
ruby/
├── Gemfile
├── Gemfile.lock
├── bin/
│   └── abraham               # Executable entry point
├── lib/
│   ├── abraham.rb            # Main module
│   ├── abraham/
│   │   ├── version.rb
│   │   ├── cli.rb            # Thor application
│   │   ├── commands/
│   │   │   ├── init.rb
│   │   │   ├── config.rb
│   │   │   ├── project.rb
│   │   │   └── task.rb
│   │   ├── config.rb         # Configuration management
│   │   ├── db/
│   │   │   ├── connection.rb
│   │   │   ├── migrate.rb
│   │   │   └── repository.rb
│   │   ├── model/
│   │   │   ├── project.rb
│   │   │   └── task.rb       # With children association
│   │   ├── format/
│   │   │   ├── base.rb       # Strategy base class
│   │   │   ├── table.rb
│   │   │   ├── json.rb
│   │   │   └── tree.rb
│   │   └── server.rb         # Sinatra API
│   └── abraham.rb
├── spec/                     # RSpec tests
└── abraham.gemspec
```

**Idiomatic Patterns:**
- Modules and mixins for shared behavior
- Duck typing for Strategy pattern
- Blocks and procs for callbacks
- Method chaining for fluent APIs
- `Struct` or plain classes for domain models

**Ruby-Specific Extensions:**
- Interactive console: `abraham console` (IRB with context)
- Rake tasks for maintenance operations
- `--format=yaml` output (native YAML support)
- Hot reloading in development

---

### OCaml

**Why OCaml fits well:**
- Strong static typing with inference
- Algebraic data types perfect for domain modeling
- Pattern matching for exhaustive handling
- Excellent for correctness

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `cmdliner` | Declarative, composable |
| Config | `yojson` | JSON parsing for config |
| SQLite | `sqlite3-ocaml` | OCaml bindings |
| SQLite (alt) | `caqti` | Async-capable, connectors |
| HTTP | `dream` | Modern, easy to use |
| HTTP (alt) | `cohttp` + `lwt` | Lower-level, flexible |
| JSON | `yojson` | Common JSON library |
| JSON (alt) | `ppx_yojson_conv` | Deriving for types |
| Logging | `logs` | Logging infrastructure |
| Colors | Custom | ANSI escape sequences |
| Tables | Custom | Format with Printf |
| Build | `dune` | Standard build system |
| Package | `opam` | Package manager |

**Project Structure:**

```
ocaml/
├── dune-project
├── abraham.opam
├── bin/
│   ├── dune
│   └── main.ml               # Entry point
├── lib/
│   ├── dune
│   ├── cli.ml                # Command definitions
│   ├── cli.mli               # Interface
│   ├── config.ml
│   ├── db/
│   │   ├── db.ml             # Connection
│   │   ├── migrate.ml
│   │   └── repo.ml           # Queries
│   ├── model/
│   │   ├── project.ml
│   │   ├── task.ml           # Task with children list
│   │   └── types.ml          # Shared types
│   ├── format/
│   │   ├── formatter.ml      # Module type (Strategy)
│   │   ├── table.ml
│   │   ├── json.ml
│   │   └── tree.ml
│   └── server/
│       ├── server.ml         # Dream handlers
│       └── routes.ml
└── test/
    ├── dune
    └── test_abraham.ml
```

**Idiomatic Patterns:**
- Algebraic data types (variants) for Status, Priority
- Module types (signatures) for Strategy pattern
- First-class modules for runtime strategy selection
- Records for domain models
- `option` type for nullable fields
- Result monad for error handling

**OCaml-Specific Extensions:**
- Exhaustive pattern matching (compiler-enforced)
- Type-safe SQL with Caqti
- Property-based testing with QCheck
- Native code compilation for performance

---

### Elixir

**Why Elixir fits well:**
- Excellent concurrency model (BEAM VM)
- Pattern matching and immutability
- Supervisor trees for reliability
- Strong web framework ecosystem

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `optimus` | Feature-rich CLI parsing |
| CLI (alt) | Custom | escript with OptionParser |
| Config | `config` | Built-in config system |
| SQLite | `exqlite` | NIF-based SQLite |
| SQLite (alt) | `ecto_sqlite3` | Ecto adapter |
| HTTP | `plug` + `bandit` | Composable, modern |
| HTTP (alt) | `phoenix` | Full framework (overkill) |
| JSON | `jason` | Fast JSON encoding |
| Logging | `logger` | Built-in |
| Colors | `io_ansi` | Built-in ANSI |
| Tables | `table_rex` | Table formatting |
| Build | `mix` | Standard build tool |

**Project Structure:**

```
elixir/
├── mix.exs
├── mix.lock
├── config/
│   ├── config.exs
│   └── runtime.exs
├── lib/
│   ├── abraham.ex            # Application entry
│   ├── abraham/
│   │   ├── cli.ex            # CLI parsing, dispatch
│   │   ├── cli/
│   │   │   ├── init.ex
│   │   │   ├── config.ex
│   │   │   ├── project.ex
│   │   │   └── task.ex
│   │   ├── config.ex         # Runtime config
│   │   ├── db/
│   │   │   ├── repo.ex       # Ecto repo or raw queries
│   │   │   ├── migrate.ex
│   │   │   └── project.ex
│   │   │   └── task.ex
│   │   ├── model/
│   │   │   ├── project.ex    # Struct + changeset
│   │   │   └── task.ex       # With subtasks
│   │   ├── format/
│   │   │   ├── formatter.ex  # Behaviour (Strategy)
│   │   │   ├── table.ex
│   │   │   ├── json.ex
│   │   │   └── tree.ex
│   │   └── server/
│   │       ├── router.ex     # Plug router
│   │       └── endpoint.ex
├── priv/
│   └── repo/
│       └── migrations/
└── test/
    ├── test_helper.exs
    └── abraham_test.exs
```

**Idiomatic Patterns:**
- Behaviours for Strategy pattern (`@callback`)
- Structs with embedded children for Composite
- Pattern matching in function heads
- Pipe operator for transformations
- GenServer for stateful operations (optional)
- Supervision trees for server reliability

**Elixir-Specific Extensions:**
- Live dashboard for server monitoring
- Distributed operation across nodes
- Hot code upgrades
- `:observer` for runtime introspection
- Escript for single-file distribution

---

### TypeScript (Deno)

**Why Deno fits well:**
- First-class TypeScript support
- Secure by default (permissions)
- Single executable compilation
- Modern standard library
- URL-based imports (no node_modules)

**Recommended Libraries:**

| Purpose | Library | Notes |
|---------|---------|-------|
| CLI | `@std/cli` | Deno stdlib flags parsing |
| CLI (alt) | `cliffy` | Full-featured, subcommands |
| Config | Custom | JSON + Deno.env |
| SQLite | `@db/sqlite` | Deno SQLite via FFI |
| SQLite (alt) | `sqlite3` | WASM-based alternative |
| HTTP | `@std/http` | Deno stdlib server |
| HTTP (alt) | `hono` | Fast, lightweight framework |
| JSON | Built-in | Native JSON support |
| Logging | `@std/log` | Deno stdlib logging |
| Colors | `@std/fmt/colors` | Deno stdlib colors |
| Tables | Custom | Simple implementation |
| Test | `@std/testing` | Deno stdlib testing |

**Project Structure:**

```
typescript/
├── deno.json                 # Config, tasks, imports
├── deno.lock
├── mod.ts                    # Main entry point
├── src/
│   ├── cli/
│   │   ├── mod.ts            # CLI setup with cliffy
│   │   ├── init.ts
│   │   ├── config.ts
│   │   ├── project.ts
│   │   └── task.ts
│   ├── config/
│   │   └── mod.ts            # Config loading
│   ├── db/
│   │   ├── mod.ts            # Connection
│   │   ├── migrate.ts
│   │   ├── project.ts        # Project queries
│   │   └── task.ts           # Task queries
│   ├── model/
│   │   ├── mod.ts
│   │   ├── project.ts        # Type definitions
│   │   └── task.ts           # Task with children
│   ├── format/
│   │   ├── mod.ts            # Formatter interface
│   │   ├── table.ts
│   │   ├── json.ts
│   │   └── tree.ts
│   ├── server/
│   │   ├── mod.ts            # HTTP server setup
│   │   └── routes.ts         # API routes
│   └── log.ts                # Logging setup
└── tests/
    ├── cli_test.ts
    ├── db_test.ts
    └── api_test.ts
```

**Idiomatic Patterns:**
- Interfaces for Strategy pattern
- Classes or type aliases for domain models
- Discriminated unions for enums with data
- `async/await` throughout
- Type guards for runtime validation
- Zod for schema validation (optional)

**Deno-Specific Extensions:**
- `deno compile` for single binary distribution
- Permission flags (`--allow-read`, `--allow-net`)
- Built-in test runner and formatter
- Import maps for dependency management
- `deno task` for scripts (in deno.json)
- WebAssembly support for SQLite

**deno.json example:**

```json
{
    "name": "@abraham/cli",
    "version": "0.1.0",
    "exports": "./mod.ts",
    "tasks": {
        "dev": "deno run --allow-read --allow-write --allow-env --allow-ffi mod.ts",
        "build": "deno compile --allow-read --allow-write --allow-env --allow-ffi --output abraham mod.ts",
        "test": "deno test --allow-read --allow-write --allow-env --allow-ffi",
        "serve": "deno run --allow-read --allow-write --allow-env --allow-ffi --allow-net mod.ts serve"
    },
    "imports": {
        "@std/": "jsr:@std/",
        "@db/sqlite": "jsr:@db/sqlite",
        "cliffy/": "https://deno.land/x/cliffy@v1.0.0-rc.3/",
        "hono": "jsr:@hono/hono"
    }
}
```

---

## Language Comparison Summary

| Language | Binary | CLI Library | SQLite | HTTP | Difficulty |
|----------|--------|-------------|--------|------|------------|
| **Go** | Single | Cobra | go-sqlite3 | chi / gin | Easy |
| **Rust** | Single | Clap | rusqlite | axum | Medium |
| **Python** | Script | Typer | sqlite3 | FastAPI | Easy |
| **Zig** | Single | zig-clap | C interop | zap | Hard |
| **Nim** | Single | cligen | db_sqlite | jester | Medium |
| **Crystal** | Single | admiral | sqlite3 | kemal | Medium |
| **C++** | Single | CLI11 | SQLiteCpp | cpp-httplib | Medium |
| **C** | Single | argtable3 | sqlite3 | libmicrohttpd | Hard |
| **D** | Single | darg | d2sqlite3 | vibe.d | Medium |
| **Ruby** | Script | thor | sqlite3 | sinatra | Easy |
| **OCaml** | Single | cmdliner | sqlite3-ocaml | dream | Hard |
| **Elixir** | Script | optimus | exqlite | plug | Medium |
| **TypeScript** | Single | cliffy | @db/sqlite | hono | Easy |

