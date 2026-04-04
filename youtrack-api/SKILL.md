Reference guide for YouTrack REST API. Use when building integrations against any YouTrack instance.

---

# YouTrack REST API Reference

## Base URL & Auth

- Base: `https://<instance>/youtrack/api/` (self-hosted) or `https://<instance>.myjetbrains.com/youtrack/api/`
- Auth: Bearer token in `Authorization` header, or via the Go `drichardson/youtrack` library which handles it via `Api.Token`
- Tokens are per-user, generated at **Profile → Account Security**

---

## Issue Fields

Request fields via `?fields=` — nested fields use parentheses.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | Internal DB id (e.g. `2-1234`) — not the readable key |
| `idReadable` | string | Human-readable key (e.g. `QW-123`) — use this in URLs and display |
| `summary` | string | Issue title |
| `description` | string | Full description |
| `created` | int64 | Unix milliseconds |
| `resolved` | int64 | Unix milliseconds; 0/absent if not resolved |
| `customFields(...)` | array | See custom fields section below |

Always request `idReadable` instead of (or alongside) `id` when you need to display or link issues.

---

## Custom Fields

Request with: `customFields($type,id,projectCustomField($type,id,field($type,id,name)),value(...))`

**Value shape varies by `$type`:**

| `$type` | `value` shape | How to get the value |
|---------|--------------|----------------------|
| `SingleEnumIssueCustomField` | `{name, id, $type}` | `value.name` |
| `MultiEnumIssueCustomField` | `[{name, id, $type}, ...]` | array of names |
| `SimpleIssueCustomField` | primitive (float, string) | `value` directly |
| `StateIssueCustomField` | `{name, isResolved, $type}` | `value.name` |
| `SingleUserIssueCustomField` | `{name, login, $type}` | `value.name` |
| `PeriodIssueCustomField` | `{minutes, $type}` | `value.minutes` |

For enum/user value fields, request: `value($type,id,name,localizedName)`  
For simple/primitive fields, `value` alone suffices — primitives are returned as-is regardless of subfield spec.

Look up the field by `projectCustomField.field.name` to identify it by display name.

---

## Search Query Syntax

Filters are combined with spaces. String values with spaces use `{curly braces}`.

```
state: {In Progress}           # filter by state name (braces required for multi-word values)
state: Done                    # no braces needed if single word
#Resolved                      # shorthand: matches all final/resolved statuses across all workflows
state: Resolved                # equivalent shorthand — both map to every "resolved" state automatically
project: QW                    # single project
project: QW,RPM                # multiple values are comma-separated (no spaces around comma)
project: -SQ                   # exclude a project (negative filter)
project: QW,RPM -SQ            # combine positive and negative
resolved date: 2026-01-01 .. 2026-03-31   # date range (IMPORTANT: "resolved date", not "resolved")
created date: 2026-01-01 .. today
updated: {Last week}           # relative date
assignee: me
```

**Common gotchas:**
- Multiple values for any filter field are **comma-separated**: `project: QW,RPM`, `state: Done,{In Progress}` — not space-separated
- The date filter field is `resolved date` (two words), not `resolved`. Using `resolved:` alone returns a 400 query parse error
- Prefer `#Resolved` over a specific state name when you want "any resolved state" — it works across projects with different workflow naming

---

## Activity / History API

Fetch state transition history for an issue:

```
GET /api/issues/{idReadable}/activities?fields=timestamp,added(name),$type&categories=CustomFieldCategory
```

**Key points:**
- Use `CustomFieldCategory`, not `StateCategory` — the latter may not return state changes depending on the instance version
- Returns all custom field changes, not just state changes — filter by `$type == "CustomFieldActivityItem"` and `added[].($type) == "StateBundleElement"` to isolate state transitions
- **`added` is mixed type** — for state changes it's an array of objects, but for date/period fields it may be a bare number or null. Always parse `added` as `json.RawMessage` first, then try to unmarshal as array; skip on failure
- Events are returned in chronological order; the first matching event is the earliest entry into a state
- Timestamps are Unix milliseconds

**Minimal Go structs:**

```go
type StateEventValue struct {
    Name string `json:"name"`
    Type string `json:"$type"`
}
type StateEvent struct {
    Timestamp int64           `json:"timestamp"`
    Added     json.RawMessage `json:"added"` // NOT []StateEventValue — mixed types
}

// Finding first entry into a state:
func firstEntry(history []StateEvent, stateName string) int64 {
    for _, event := range history {
        var items []json.RawMessage
        if err := json.Unmarshal(event.Added, &items); err != nil {
            continue // added is not an array
        }
        for _, raw := range items {
            var v StateEventValue
            if json.Unmarshal(raw, &v) == nil &&
               v.Type == "StateBundleElement" &&
               strings.EqualFold(v.Name, stateName) {
                return event.Timestamp
            }
        }
    }
    return 0
}
```

---

## Performance Notes

- There is no server-side push of resolved date into a dedicated indexed field accessible via simple `resolved:` filter — use `resolved date: from .. to` instead
- Activity history requires one HTTP call per issue — show a progress indicator for large sets
- Default result set may be unbounded depending on instance config; filter server-side as much as possible (state, project, date) before processing client-side
