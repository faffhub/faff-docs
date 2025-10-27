# Faff Timesheet File Specification

## Overview

Faff timesheet files are **cryptographically signed** JSON documents that represent a compiled, verified record of work done on a specific day. Unlike log files (which are editable TOML), timesheets are:

- **Immutable**: Once signed, contents cannot be changed without invalidating signatures
- **Verifiable**: Digital signatures prove authenticity and integrity
- **Submittable**: Designed to be sent to external systems (HR, project management, clients)
- **Audience-Specific**: Generated for specific recipients with appropriate filtering

Timesheets are generated **from** log files by "Audience" plugins, which:
1. Filter/transform log data for the intended recipient
2. Add metadata (who compiled it, when, for what audience)
3. Cryptographically sign the result

## File Format

### File Extension
`.json`

### File Naming Convention
`{date}-{audience-id}.json`

Examples:
- `2025-03-15-hr-department.json`
- `2025-03-15-client-alpha.json`

### Character Encoding
UTF-8

### Format
JSON with canonicalized serialization for signature verification

## File Structure

### Signed Timesheet

```json
{
  "actor": {
    "name": "Alice Smith",
    "email": "alice@example.com",
    "employee_id": "EMP-12345"
  },
  "version": "Faffage-generated timesheet v1.0 please see faffage.com for details",
  "date": "2025-03-15",
  "compiled": "2025-03-15T18:30:00Z",
  "timezone": "America/New_York",
  "timeline": [
    {
      "intent": {
        "alias": "feature work",
        "role": "engineer",
        "objective": "development",
        "action": "coding",
        "subject": "authentication",
        "trackers": ["PROJ-123"]
      },
      "start": "2025-03-15T09:00:00-05:00",
      "end": "2025-03-15T12:00:00-05:00",
      "note": "Implemented OAuth2 flow"
    }
  ],
  "signatures": {
    "alice@example.com": {
      "ed25519:a1b2c3d4...": "f4e3d2c1b0a9..."
    }
  }
}
```

## Top-Level Fields

### Required Fields

- **`version`** (string): Format version identifier
  - Current: `"Faffage-generated timesheet v1.0 please see faffage.com for details"`
  - Fixed string for compatibility checking

- **`date`** (string): The day this timesheet covers
  - Format: ISO 8601 date (`YYYY-MM-DD`)
  - Example: `"2025-03-15"`

- **`compiled`** (string): When this timesheet was compiled
  - Format: RFC3339 datetime (always UTC, uses `Z` suffix)
  - Example: `"2025-03-15T18:30:00Z"`
  - With microseconds: `"2025-03-15T18:30:00.123456Z"`

- **`timezone`** (string): Timezone for the work day
  - Format: IANA timezone name
  - Example: `"America/New_York"`, `"Europe/London"`, `"UTC"`

- **`timeline`** (array): Sessions for this day
  - Array of Session objects (see [Session Structure](#session-structure))
  - Can be empty array `[]` for days with no work

### Optional Fields

- **`actor`** (object): Information about who did the work
  - Key-value pairs of identifying information
  - Common keys: `"name"`, `"email"`, `"employee_id"`, `"username"`
  - All values are strings
  - Example:
    ```json
    {
      "name": "Alice Smith",
      "email": "alice@example.com",
      "employee_id": "EMP-12345"
    }
    ```
  - Omitted if empty

- **`signatures`** (object): Cryptographic signatures
  - Outer keys: Signer identifiers (typically email or username)
  - Inner keys: Key IDs in format `"ed25519:{sha256-hash-of-public-key}"`
  - Values: Hex-encoded signature bytes
  - Example:
    ```json
    {
      "alice@example.com": {
        "ed25519:a1b2c3d4e5f6...": "f4e3d2c1b0a98765..."
      }
    }
    ```
  - Omitted if no signatures

## Session Structure

Each timeline entry is a JSON object representing a work session:

```json
{
  "intent": {
    "alias": "feature work",
    "role": "engineer",
    "objective": "development",
    "action": "coding",
    "subject": "authentication",
    "trackers": ["PROJ-123", "PROJ-124"]
  },
  "start": "2025-03-15T09:00:00-05:00",
  "end": "2025-03-15T12:00:00-05:00",
  "note": "Implemented OAuth2 flow"
}
```

### Session Fields

- **`intent`** (object, required): Classification of what was done
  - `alias` (string, optional): Short name
  - `role` (string, optional): Worker's role
  - `objective` (string, optional): What was being achieved
  - `action` (string, optional): Activity type
  - `subject` (string, optional): What was worked on
  - `trackers` (array of strings, optional): Project/issue IDs
  - All fields optional, empty array for trackers if none

- **`start`** (string, required): Session start time
  - Format: RFC3339 datetime with timezone offset
  - Examples:
    - `"2025-03-15T09:00:00-05:00"` (EST)
    - `"2025-03-15T14:00:00Z"` (UTC)
    - `"2025-03-15T09:00:00.123456-05:00"` (with microseconds)

- **`end`** (string, optional): Session end time
  - Same format as `start`
  - Omitted for open sessions (still running)

- **`note`** (string, optional): Free-form text note
  - Omitted if not present

## Datetime Formats

All datetime fields use RFC3339 format:

### UTC Timezone
Uses `Z` suffix:
```json
"compiled": "2025-03-15T18:30:00Z"
"start": "2025-03-15T14:00:00Z"
```

With microseconds:
```json
"compiled": "2025-03-15T18:30:00.123456Z"
```

### Non-UTC Timezones
Uses offset format:
```json
"start": "2025-03-15T09:00:00-05:00"
"end": "2025-03-15T17:00:00-04:00"
```

**Important**: The offset shows the **actual** offset at that moment, including DST. The semantic timezone (e.g., "America/New_York") is in the top-level `timezone` field.

## Cryptographic Signatures

### Signing Process

1. **Create Unsigned Timesheet**: All fields except `signatures`
2. **Serialize to Canonical JSON**: Sorted keys, no whitespace
3. **Generate Signature**: Sign bytes with Ed25519 private key
4. **Create Key ID**: SHA-256 hash of public key, hex-encoded, prefixed with `"ed25519:"`
5. **Add to Timesheet**: Store in `signatures` object

### Signature Structure

```json
"signatures": {
  "alice@example.com": {
    "ed25519:a1b2c3d4e5f6...": "f4e3d2c1b0a98765..."
  },
  "bob@example.com": {
    "ed25519:9876543210ab...": "0123456789ab..."
  }
}
```

- **Outer key**: Signer identity (email, username, etc.)
- **Inner key**: `"ed25519:{hex(sha256(public_key))}"`
- **Value**: Hex-encoded Ed25519 signature (128 hex chars = 64 bytes)

### Multiple Signatures

Timesheets can have multiple signatures:
- Same person, multiple keys (key rotation)
- Multiple people (co-signatures, approvals)
- Different identities of same person

### Verification

To verify a signature:

1. **Extract unsigned fields**: Remove `signatures` from JSON
2. **Canonicalize**: Serialize with sorted keys, no whitespace
3. **Decode signature**: Hex decode the signature value
4. **Get public key**: From key ID or external key registry
5. **Verify**: Use Ed25519 verify algorithm

## Metadata (Not in File)

The `TimesheetMeta` struct exists in memory but is **not serialized** to the file:

```rust
// In-memory only, not in JSON
TimesheetMeta {
    audience_id: "hr-department",
    submitted_at: Some(2025-03-15T19:00:00Z),
    submitted_by: Some("alice@example.com")
}
```

This metadata tracks:
- **`audience_id`**: Which audience this was compiled for
- **`submitted_at`**: When it was submitted (if applicable)
- **`submitted_by`**: Who submitted it (if applicable)

Metadata is stored **separately** (e.g., in a database or companion file) to preserve the cryptographic integrity of the signed timesheet.

## Submittable Format

When submitting to external systems, timesheets use a canonical JSON serialization:

```json
{
  "actor": {...},
  "compiled": "2025-03-15T18:30:00Z",
  "date": "2025-03-15",
  "signatures": {...},
  "timeline": [...],
  "timezone": "America/New_York",
  "version": "Faffage-generated timesheet v1.0 please see faffage.com for details"
}
```

**Note**: Keys are alphabetically sorted for canonical form.

## Audience Plugins

Timesheets are compiled by **Audience** plugins that:

### 1. Filter Sessions
- Remove private/sensitive sessions
- Include only relevant projects/trackers
- Redact notes or other fields

### 2. Transform Data
- Round times to quarter-hours
- Aggregate similar sessions
- Rename tracker IDs for external systems

### 3. Add Actor Info
- Populate `actor` with audience-appropriate identity
- Different audiences may see different actor data

### 4. Sign
- Generate cryptographic signature with configured key
- Add to `signatures` field

### Example Plugin Flow

```python
class HRDepartmentAudience(Audience):
    def compile_time_sheet(self, log: Log) -> Timesheet:
        # Filter: only sessions with work-related roles
        sessions = [s for s in log.timeline
                   if s.intent.role in ["engineer", "manager"]]

        # Transform: remove notes
        sessions = [Session(s.intent, s.start, s.end, note=None)
                   for s in sessions]

        # Create unsigned timesheet
        timesheet = Timesheet(
            actor={"name": "Alice Smith", "employee_id": "EMP-12345"},
            date=log.date,
            compiled=datetime.now(UTC),
            timezone=log.timezone,
            timeline=sessions,
            signatures={},
            meta=TimesheetMeta(audience_id="hr-department")
        )

        # Sign it
        signing_key = self.load_signing_key("alice-work-key")
        return timesheet.sign("alice@example.com", signing_key)
```

## Complete Example

```json
{
  "actor": {
    "name": "Alice Smith",
    "email": "alice@example.com",
    "employee_id": "EMP-12345"
  },
  "version": "Faffage-generated timesheet v1.0 please see faffage.com for details",
  "date": "2025-03-15",
  "compiled": "2025-03-15T18:30:00Z",
  "timezone": "America/New_York",
  "timeline": [
    {
      "intent": {
        "alias": "standup",
        "role": "engineer",
        "objective": "coordination",
        "action": "meeting",
        "subject": "team sync",
        "trackers": []
      },
      "start": "2025-03-15T09:00:00-05:00",
      "end": "2025-03-15T09:15:00-05:00"
    },
    {
      "intent": {
        "alias": "feature work",
        "role": "engineer",
        "objective": "development",
        "action": "coding",
        "subject": "authentication",
        "trackers": ["PROJ-123", "PROJ-124"]
      },
      "start": "2025-03-15T09:30:00-05:00",
      "end": "2025-03-15T12:00:00-05:00",
      "note": "Implemented OAuth2 flow"
    },
    {
      "intent": {
        "alias": "code review",
        "role": "engineer",
        "objective": "quality",
        "action": "reviewing",
        "subject": "pull requests",
        "trackers": ["PROJ-125"]
      },
      "start": "2025-03-15T14:00:00-04:00",
      "end": "2025-03-15T15:30:00-04:00"
    }
  ],
  "signatures": {
    "alice@example.com": {
      "ed25519:a1b2c3d4e5f67890abcdef1234567890abcdef1234567890abcdef1234567890": "f4e3d2c1b0a9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba987654"
    }
  }
}
```

## Validation Rules

1. **Required Fields**: `version`, `date`, `compiled`, `timezone`, `timeline` must be present
2. **Date Format**: `date` must be valid ISO 8601 date
3. **Datetime Format**: `compiled`, session `start`/`end` must be valid RFC3339
4. **Timeline Order**: Sessions should be ordered by `start` time (not enforced, but recommended)
5. **Signature Validity**: Signatures must verify against the unsigned content
6. **Key ID Format**: Must be `"ed25519:{64-char-hex}"`
7. **Signature Format**: Must be 128-character hex string (64 bytes)

## Security Considerations

### Signature Verification
Always verify signatures before trusting timesheet data:
- Ensures data hasn't been tampered with
- Proves who created/approved the timesheet
- Detects any modifications after signing

### Key Management
- Private keys should be stored securely (encrypted, keychain, HSM)
- Public keys should be published/registered with audiences
- Key rotation: sign with multiple keys during transition

### Privacy
- Different audiences see different `actor` information
- Audience plugins filter sensitive data before signing
- Signatures prove the filtered version, not the original log

## File Locations

```
.faff/
└── timesheets/
    ├── 2025-03-15-hr-department.json
    ├── 2025-03-15-client-alpha.json
    └── 2025-03-16-hr-department.json
```

Metadata typically stored separately:
```
.faff/
└── timesheet_meta/
    └── 2025-03-15-hr-department.json
    {
      "audience_id": "hr-department",
      "submitted_at": "2025-03-15T19:00:00Z",
      "submitted_by": "alice@example.com"
    }
```

## Differences from Log Files

| Feature | Log File (TOML) | Timesheet (JSON) |
|---------|----------------|------------------|
| **Format** | TOML | JSON |
| **Mutability** | Editable | Immutable (signed) |
| **Purpose** | Daily work tracking | Verified reporting |
| **Audience** | Personal | External recipients |
| **Signatures** | None | Cryptographic |
| **Derived Values** | Yes (comments) | No |
| **Formatting** | Aligned, commented | Canonical JSON |
| **Privacy** | Full detail | Filtered per audience |

## Version History

- **v1.0**: Current version (indicated in `version` field)

## See Also

- [Log File Specification](LOG_FILE_SPEC.md) - Source data for timesheets
- [Plan File Specification](PLAN_FILE_SPEC.md) - Vocabulary and templates
- [Timesheet Model](../core/src/models/timesheet.rs) - Rust implementation
- [Plugin System](../bindings-python/python/faff_core/plugins.py) - Audience plugin base class
