# Faff Log File Specification v1.1

## Overview

Faff log files use TOML format to store time-tracking data for a single day. Each file contains metadata about the day and a timeline of work sessions with intent-based classification.

## File Format

### File Extension
`.toml`

### File Naming Convention
Typically named by date, e.g., `2025-03-15.toml`

### Character Encoding
UTF-8

## File Structure

### Header

```toml
# This is a Faff-format log file - see faffage.com for details.
# It has been generated but can be edited manually.
# Changes to rows starting with '#' will not be saved.
version  = "1.1"
date     = "2025-03-15"
timezone = "Europe/London"
# date_format = "HH:mm"
```

**Fields:**
- `version` (string, required): File format version, currently `"1.1"`
- `date` (string, required): ISO 8601 date format (`YYYY-MM-DD`)
- `timezone` (string, required): IANA timezone name (e.g., `"UTC"`, `"America/New_York"`, `"Europe/London"`)
- `date_format` (string, derived): Comment-only field showing time format used in the file
  - `"HH:mm"`: Plain time (no timezone offset needed)
  - `"HH:mmZ"`: Time with timezone offset (used on DST transition days)

### Timeline Entries

Each session is represented as a TOML table array entry:

```toml
[[timeline]]
alias     = "work"
role      = "engineer"
objective = "development"
action    = "coding"
subject   = "features"
trackers  = "PROJ-123"  # Project Name
start     = "09:00"
end       = "10:30"
# duration = "1 hour and 30 minutes"
note      = "Optional note about this session"
```

## Session Fields

### Intent Fields

All intent fields are **optional**:

- **`alias`** (string): Short name/label for the session
  - Auto-generated if not provided: `"{role}: {action} to {objective} for {subject}"`
  - Example: `"work"`, `"meeting"`, `"break"`

- **`role`** (string): Your role during this session
  - Example: `"engineer"`, `"manager"`, `"researcher"`

- **`objective`** (string): What you're trying to achieve
  - Example: `"development"`, `"planning"`, `"review"`

- **`action`** (string): The activity you're performing
  - Example: `"coding"`, `"writing"`, `"testing"`, `"meeting"`

- **`subject`** (string): What you're working on
  - Example: `"features"`, `"bugs"`, `"documentation"`

- **`trackers`** (string or array): Project/issue tracking IDs
  - Single tracker: `trackers = "PROJ-123"`
  - Multiple trackers: `trackers = ["PROJ-123", "TASK-456"]`
  - Inline comments show tracker names: `trackers = "PROJ-123"  # Fix login bug`

### Time Fields

- **`start`** (string, required): Session start time
  - Format: `"HH:MM"` (24-hour format)
  - With offset (DST days): `"HH:MM+HHMM"` or `"HH:MM-HHMM"`
  - Example: `"09:00"`, `"14:30"`, `"02:30+0100"`

- **`end`** (string, optional): Session end time
  - Same format as `start`
  - If omitted, session is "open" (still running)

- **`duration`** (string, derived): Calculated duration (comment only)
  - Auto-calculated as `end - start`
  - Format: Human-readable (e.g., `"1 hour and 30 minutes"`, `"45 minutes"`, `"2 hours, 15 minutes and 30 seconds"`)
  - Prefixed with `#` (not saved if file is re-written)

### Note Field

- **`note`** (string, optional): Free-form text note
  - Example: `"Pair programming with Alice"`
  - Omitted if empty

## Special Formatting

### Derived Values

Lines starting with `--` are converted to comments (prefixed with `#`) when the file is saved. These are calculated values that can be regenerated:

```toml
--duration = "1 hour and 30 minutes"
# Becomes:
# duration = "1 hour and 30 minutes"
```

### Aligned Equals Signs

Field names are automatically right-aligned for readability:

```toml
alias     = "work"
role      = "engineer"
start     = "09:00"
end       = "10:30"
trackers  = "PROJ-123"
```

### Empty Timeline

If no sessions exist:

```toml
# Timeline is empty.
```

## DST (Daylight Saving Time) Handling

On days with DST transitions, times include timezone offsets to resolve ambiguity:

```toml
timezone    = "America/New_York"
# date_format = "HH:mmZ"

[[timeline]]
start = "01:30-0500"  # Before DST "spring forward"
end   = "03:30-0400"  # After DST transition
```

Without offsets on non-DST days:

```toml
# date_format = "HH:mm"

[[timeline]]
start = "09:00"
end   = "17:00"
```

## Complete Example

```toml
# This is a Faff-format log file - see faffage.com for details.
# It has been generated but can be edited manually.
# Changes to rows starting with '#' will not be saved.
version      = "1.1"
date         = "2025-03-15"
timezone     = "UTC"
# date_format = "HH:mm"

[[timeline]]
alias        = "standup"
role         = "engineer"
objective    = "coordination"
action       = "meeting"
subject      = "team sync"
start        = "09:00"
end          = "09:15"
# duration    = "15 minutes"

[[timeline]]
alias        = "feature work"
role         = "engineer"
objective    = "development"
action       = "coding"
subject      = "authentication"
trackers     = ["PROJ-123", "PROJ-124"]
start        = "09:30"
end          = "12:00"
# duration    = "2 hours and 30 minutes"
note         = "Implemented OAuth2 flow"

[[timeline]]
alias        = "code review"
role         = "engineer"
objective    = "quality"
action       = "reviewing"
subject      = "pull requests"
trackers     = "PROJ-125"  # Frontend refactor
start        = "14:00"
end          = "15:30"
# duration    = "1 hour and 30 minutes"
```

## Parsing Notes

1. **Session Order**: Sessions should be sorted by `start` time when writing, but parsers should handle any order
2. **Open Sessions**: A session without an `end` field is considered "open" (currently running)
3. **Active Session**: Only the last session in the timeline can be open
4. **Timezone Awareness**: All times are interpreted in the log's `timezone`, not as UTC
5. **Comments**: Lines starting with `#` are comments and ignored during parsing
6. **Derived Fields**: Fields prefixed with `--` or `#` are regenerated and not preserved across saves

## Validation Rules

1. Each log file represents exactly one day (`date` field)
2. All session times must fall within the log's date (or cross midnight with explicit offset)
3. `start` is required for all sessions
4. If a session has both `start` and `end`, `end` must be after `start`
5. Tracker IDs should be unique within a session's `trackers` list (auto-deduplicated)
6. Only the last session in the timeline can be open (missing `end`)

## Version History

- **v1.1** (current): Current specification
- **v1.0**: Original format (similar structure, minor differences in comments)

## See Also

- [Plan File Specification](PLAN_FILE_SPEC.md)
- [Intent Model Documentation](../core/src/models/intent.rs)
- [Session Model Documentation](../core/src/models/session.rs)
- [Log Model Documentation](../core/src/models/log.rs)
