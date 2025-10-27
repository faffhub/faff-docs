# Faff Documentation

This directory contains technical specifications and documentation for the Faff time-tracking system.

## File Format Specifications

### [Log File Specification](LOG_FILE_SPEC.md)
Complete specification for `.toml` log files that store daily time-tracking data.

**Key Topics:**
- File structure and metadata
- Session/timeline entries
- Intent-based classification (role, objective, action, subject)
- Time formats and DST handling
- Tracker references
- Derived values and formatting

**Example:**
```toml
version  = "1.1"
date     = "2025-03-15"
timezone = "UTC"

[[timeline]]
alias    = "feature work"
role     = "engineer"
start    = "09:00"
end      = "12:00"
```

### [Plan File Specification](PLAN_FILE_SPEC.md)
Complete specification for plan/template files that define reusable work configurations.

**Key Topics:**
- Vocabulary lists (roles, actions, objectives, subjects)
- Tracker ID mappings
- Pre-defined session intents
- Validity periods
- Local vs remote plans
- Plan ID generation

**Example:**
```toml
source      = "project-alpha"
valid_from  = "2025-03-01"

roles = ["engineer", "manager"]
actions = ["coding", "meeting"]

[trackers]
"PROJ-123" = "Fix login bug"

[[intents]]
alias = "standup"
role  = "engineer"
```

### [Timesheet File Specification](TIMESHEET_FILE_SPEC.md)
Complete specification for `.json` timesheet files - cryptographically signed, immutable records for external submission.

**Key Topics:**
- JSON structure and required fields
- Cryptographic signatures (Ed25519)
- Actor identification
- Audience-specific filtering
- Canonical JSON serialization
- Signature verification

**Example:**
```json
{
  "actor": {"name": "Alice Smith", "email": "alice@example.com"},
  "version": "Faffage-generated timesheet v1.0 please see faffage.com for details",
  "date": "2025-03-15",
  "compiled": "2025-03-15T18:30:00Z",
  "timezone": "America/New_York",
  "timeline": [...],
  "signatures": {
    "alice@example.com": {
      "ed25519:a1b2c3...": "f4e3d2..."
    }
  }
}
```

## Core Concepts

### Intent Model
An **Intent** represents what you're doing, classified by:
- **Alias**: Short name (e.g., "work", "meeting")
- **Role**: Your role (e.g., "engineer", "manager")
- **Objective**: What you're achieving (e.g., "development", "planning")
- **Action**: Activity type (e.g., "coding", "reviewing")
- **Subject**: What you're working on (e.g., "features", "bugs")
- **Trackers**: Project/issue IDs (e.g., ["PROJ-123"])

All fields are optional. Provides semantic meaning to time tracking.

### Session Model
A **Session** is a time period with an intent:
- **Intent**: What you were doing (see above)
- **Start**: When you started (DateTime with timezone)
- **End**: When you finished (optional - omit for open sessions)
- **Note**: Free-form text note (optional)

Sessions are immutable - operations return new instances.

### Log Model
A **Log** represents one day of work:
- **Date**: The day this log covers
- **Timezone**: IANA timezone name
- **Timeline**: Ordered list of sessions

Logs track active sessions and calculate total time.

### Plan Model
A **Plan** defines vocabulary and templates:
- **Source**: Where the plan comes from (local file or remote URL)
- **Validity Period**: Date range when plan is active
- **Vocabulary**: Lists of roles, actions, objectives, subjects
- **Trackers**: ID → name mappings
- **Intents**: Pre-defined session templates

Plans can be local files or fetched from remote systems via plugins.

## Architecture

```
faff-core/
├── core/                   # Rust core library
│   ├── models/            # Data models (Intent, Session, Log, Plan)
│   ├── storage/           # File I/O abstraction
│   └── plugins.rs         # Python plugin system
├── bindings-python/       # Python bindings (PyO3)
│   ├── python/
│   │   └── faff_core/
│   │       └── plugins.py # Plugin base classes
│   └── src/
│       └── python/        # PyO3 wrapper code
└── docs/                  # This directory
```

### Data Flow

1. **Reading a Log**:
   ```
   TOML file → Log::from_log_file() → Log model → Python/Rust API
   ```

2. **Writing a Log**:
   ```
   Log model → Log::to_log_file() → Formatted TOML → File
   ```

3. **Using a Plan**:
   ```
   Plan file → Plan model → Vocabulary/Intents → CLI autocomplete/quick-start
   ```

4. **Remote Plan**:
   ```
   Plugin → pull_plan() → Plan model → Cache file → Available for use
   ```

5. **Compiling a Timesheet**:
   ```
   Log file → Log model → Audience plugin → Filtered/Transformed →
   Timesheet model → Sign with Ed25519 → JSON file
   ```

6. **Submitting a Timesheet**:
   ```
   Timesheet file → Verify signatures → Audience plugin → External API
   ```

## File Locations

Default directory structure:

```
.faff/
├── logs/
│   ├── 2025-03-15.toml
│   ├── 2025-03-16.toml
│   └── ...
├── plans/
│   ├── local.toml
│   └── project-alpha.toml
├── timesheets/
│   └── 2025-03-15-audience.json
├── plugins/
│   ├── jira.py
│   └── github.py
├── plugin_state/
│   └── jira-instance/
└── config.toml
```

## Python/Rust Integration

The project uses PyO3 for bidirectional Python-Rust integration:

### Rust → Python (Plugin System)
- Core (Rust) embeds Python interpreter
- Loads plugins from `.faff/plugins/`
- Plugins inherit from `PlanSource` or `Audience` base classes
- Plugins receive/return Python-wrapped Rust objects

### Python → Rust (Bindings)
- Python package `faff-core` exposes Rust models
- Python code can create/manipulate Log, Plan, Session objects
- Models are immutable (functional-style updates)
- Type conversions handled automatically by PyO3

## Key Design Principles

1. **Immutability**: Models are immutable - operations return new instances
2. **Type Safety**: Strong typing in Rust, exposed through Python bindings
3. **Timezone Awareness**: All datetime values carry timezone information
4. **Human-Editable Files**: TOML format is readable and manually editable
5. **Extensibility**: Plugin system for remote plan sources and timesheet audiences
6. **Semantic Classification**: Intent model provides meaning beyond raw time data

## Version Information

- **Current Log File Version**: v1.1
- **Plan File Format**: No version field (uses TOML schema validation)
- **Minimum Python**: 3.11+
- **Rust Edition**: 2021

## See Also

- [Release Process](../RELEASE_PROCESS.md) - How to release new versions
- [Plugin Development](../bindings-python/python/faff_core/plugins.py) - Writing plugins
- [Core Models](../core/src/models/) - Rust model implementations

## Contributing

When modifying file formats:
1. Update the relevant specification document
2. Update tests in `core/src/models/tests/`
3. Consider backwards compatibility
4. Increment file format version if breaking changes
5. Document migration path for existing files
