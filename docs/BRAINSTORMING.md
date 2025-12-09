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

Phases 5, 6, 7, 8 can be done in parallel after Phase 4.

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

## Next Steps

1. **Finalize domain choice**: Confirm task/project manager or pivot
2. **Set up repository structure**: Create directories for test and first language
3. **Implement test harness**: Build `test.nu` and assertion library
4. **Choose first language**: Recommend Rust or Go for first implementation
5. **Phase 1 implementation**: CLI foundation with tests
