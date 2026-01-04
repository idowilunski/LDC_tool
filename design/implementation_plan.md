# Implementation Plan: Local Diagnostics Snapshot Tool (v1)

## Overview

This plan translates the specification into discrete modules with clear boundaries, explicit interfaces, and defined execution order.

---

## Module Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CLI Entry Point                        │
│                  (argument parsing, orchestration)          │
└─────────────────────┬───────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│ Privilege       │     │ Mode Resolver   │
│ Detector        │     │                 │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Inspector Registry                        │
│         (dispatches to domain-specific inspectors)          │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼                           ▼
┌───────────────┐           ┌───────────────┐
│ Network       │           │ Environment   │
│ Inspector     │           │ Inspector     │
└───────┬───────┘           └───────┬───────┘
        │                           │
        └─────────────┬─────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Report Assembler                          │
│              (collects, organizes, labels)                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Output Formatter                          │
│                  (plain text or JSON)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Module Definitions

### Module 1: CLI Entry Point

**Responsibility**: Parse arguments, validate input, orchestrate execution flow, handle top-level errors, return exit codes.

**Inputs**:
- Raw command-line arguments (`argv`)

**Outputs**:
- Parsed configuration object containing:
  - `modes: Set<"network" | "env">`
  - `outputFormat: "text" | "json"`
- Exit code on failure

**Behavior**:
1. Parse flags: `--network`, `--env`, `--all`, `--json`
2. If no mode flags provided → treat as `--all`
3. If `--all` provided → expand to all supported modes
4. Multiple mode flags → combine additively (union)
5. Invalid arguments → print usage, exit non-zero

**Error Handling**:
- Unknown flags → exit code `1`, print error to stderr
- Conflicting or malformed input → exit code `1`, print error to stderr

---

### Module 2: Privilege Detector

**Responsibility**: Determine whether the current process has elevated privileges.

**Inputs**:
- (none; reads from OS)

**Outputs**:
- `PrivilegeContext` object:
  - `isElevated: boolean`
  - `detectionMethod: string` (for transparency)

**Platform-Specific Logic**:

| Platform | Detection Method |
|----------|------------------|
| Linux | Check effective UID == 0 |
| Windows | Check if running as Administrator (token inspection) |

**Error Handling**:
- If detection fails → return `isElevated: false` with reason logged
- Never assume elevation; default to unprivileged

---

### Module 3: Mode Resolver

**Responsibility**: Expand mode flags into a list of inspection domains to execute.

**Inputs**:
- Parsed mode flags from CLI Entry Point

**Outputs**:
- Ordered list of domain identifiers: `["network", "env"]`

**Behavior**:
- `--all` expands to all domains supported in v1
- Duplicate modes are deduplicated
- Order is deterministic (alphabetical or defined)

---

### Module 4: Inspector Registry

**Responsibility**: Map domain identifiers to inspector implementations, dispatch calls, collect results.

**Inputs**:
- List of domain identifiers
- `PrivilegeContext`

**Outputs**:
- Collection of `InspectionResult` objects (one per domain)

**Behavior**:
- For each domain, invoke the corresponding inspector
- Pass privilege context to each inspector
- Collect all results regardless of individual failures

**Error Handling**:
- If an inspector throws unexpectedly → capture as domain-level UNKNOWN
- Never let one domain failure abort other domains

---

### Module 5: Network Inspector

**Responsibility**: Collect network-related system state.

**Inputs**:
- `PrivilegeContext`

**Outputs**:
- `InspectionResult` containing `DiagnosticItem[]` for:
  - Active interfaces
  - IP addresses (IPv4, IPv6)
  - Default gateway
  - DNS configuration
  - Listening ports

**Platform-Specific Approaches**:

| Target | Linux Approach | Windows Approach |
|--------|----------------|------------------|
| Interfaces | Read `/sys/class/net/`, `ip addr` parsing | WMI or `ipconfig` parsing |
| Gateway | Parse `/proc/net/route` or `ip route` | `route print` or WMI |
| DNS | Read `/etc/resolv.conf` (file contents only) | `ipconfig /all` |
| Listening ports | Parse `/proc/net/tcp`, `/proc/net/udp` or `ss` | `netstat -an` or API |

**Labeling Rules**:
- Deterministic parsing of OS-provided files, commands, or APIs is considered direct observation (FACT) in v1.
- Direct observation → FACT
- Deterministic derivation from explicitly listed FACT items → INFERENCE
- INFERENCE items must reference the FACT items they were derived from
- Heuristic, probabilistic, or confidence-based inference is not permitted in v1
- Permission denied or unavailable → UNKNOWN with reason
- If timeouts are implemented, timeout → UNKNOWN with reason "operation timed out"
- Data obtained via elevation → mark `requiresElevation: true`
- In v1, the Network Inspector does not produce INFERENCE items; all reported items are either FACT or UNKNOWN.

**Error Handling**:
- File not found → UNKNOWN with `"file not found"` reason
- Permission denied → UNKNOWN with `"permission denied"` reason
- Unexpected format or parse failure → UNKNOWN with `"parse error"` reason
- Timeouts must not abort inspection; continue with remaining targets
- Inspector-level errors must not discard successfully collected items


---

### Module 6: Environment Inspector

**Responsibility**: Collect environment and PATH-related system state.

**Inputs**:
- `PrivilegeContext`

**Outputs**:
- `InspectionResult` containing `DiagnosticItem[]` for:
  - Environment variables visible to current process
  - PATH entries with existence checks

**Data Collection**:

| Target | Approach |
|--------|----------|
| Environment variables | Read from process environment |
| PATH entries | Split PATH variable, check each directory exists |

**Labeling Rules**:
- Variable exists with value → FACT
- PATH directory exists → FACT
- PATH directory does not exist → FACT (observed non-existence)
- Cannot determine existence → UNKNOWN

**Sensitive Data Handling** (careful handling required):
- Environment variables may contain secrets
- Do NOT redact by default (user expects to see values)
- Consider future flag for redaction (out of scope for v1)

---

### Module 7: Report Assembler

**Responsibility**: Organize inspection results into a structured report.

**Inputs**:
- Collection of `InspectionResult` objects
- `PrivilegeContext`

**Outputs**:
- `DiagnosticReport` object:
  - `generatedAt: timestamp`
  - `platform: string`
  - `privilegeContext: PrivilegeContext`
  - `domains: Map<domainId, DiagnosticItem[]>`
  - `disclaimer: string`

**Behavior**:
- Group items by domain
- Attach metadata (timestamp, platform)
- Include standard disclaimer text
- Preserve labeling (FACT / INFERENCE / UNKNOWN)
- Preserve per-item `requiresElevation` flags without modification
- Preserve inspector-level errors (`InspectionResult.inspectorErrors`) for output rendering
- Do not discard internal justification fields (e.g., `sources`), even if not rendered in v1 output


---

### Module 8: Output Formatter

**Responsibility**: Render the report to stdout in the requested format.

**Inputs**:
- `DiagnosticReport`
- `outputFormat: "text" | "json"`

**Outputs**:
- Formatted string written to stdout

**Plain Text Format**:
- Per-item elevation status must be rendered when `requiresElevation=true`
- Inspector-level errors must be rendered inline with affected items and must not be silently dropped

```
Local Diagnostics Snapshot
Generated: <timestamp>
Platform: <platform>
Privileges: <elevated|standard>

─── NETWORK ───────────────────────────────
[FACT][ELEVATED] Interface: eth0 - 192.168.1.100
[FACT] Default Gateway: 192.168.1.1
[UNKNOWN] DNS servers (permission denied)

─── ENVIRONMENT ───────────────────────────
[FACT] HOME=/home/user
[FACT] PATH entry: /usr/bin (exists)
[FACT] PATH entry: /opt/missing (does not exist)

─── DISCLAIMER ────────────────────────────
FACT = observed, not necessarily correct
INFERENCE = derived from observed data
UNKNOWN = could not retrieve
```

**JSON Format**:
- Serialize `DiagnosticReport` to JSON
- No schema guarantees (per spec)
- Include all labels and metadata
- Include per-item `requiresElevation`
- Include inspector-level errors
- Include disclaimer text

---

## Data Model

### DiagnosticItem

```
DiagnosticItem {
  label: "FACT" | "INFERENCE" | "UNKNOWN"
  domain: string
  key: string
  value: any | null
  reason: string | null         // populated for UNKNOWN
  requiresElevation: boolean    // true if collected with elevated privileges
  sources: string[]             // what was observed to produce this item
}
```

### InspectionResult

```
InspectionResult {
  domain: string
  items: DiagnosticItem[]
  inspectorErrors: string[]     // internal errors during inspection
}
```

### DiagnosticReport

```
DiagnosticReport {
  generatedAt: string (ISO 8601)
  platform: "linux" | "windows"
  privilegeContext: PrivilegeContext
  domains: Map<string, DiagnosticItem[]>
  disclaimer: string
}
```

---

## Execution Order

```
1. CLI Entry Point
   ├─ Parse arguments
   ├─ Validate input
   └─ On failure → exit non-zero

2. Privilege Detector
   └─ Detect elevation status

3. Mode Resolver
   └─ Expand flags to domain list

4. Inspector Registry
   └─ For each domain (can be parallel or sequential):
       ├─ Network Inspector (if requested)
       └─ Environment Inspector (if requested)

5. Report Assembler
   └─ Combine results, attach metadata

6. Output Formatter
   └─ Render to stdout

7. Exit
   └─ Exit code 0 (report produced) or non-zero (tool failure)
```

---

## Error-Handling Strategy

| Error Type | Location | Handling |
|------------|----------|----------|
| Invalid arguments | CLI Entry Point | Exit code 1, print usage to stderr |
| Privilege detection failure | Privilege Detector | Default to unprivileged, log reason |
| File not found | Inspectors | Report as UNKNOWN with reason |
| Permission denied | Inspectors | Report as UNKNOWN with "permission denied" |
| Parse error | Inspectors | Report as UNKNOWN with "parse error" |
| Unexpected exception | Inspector Registry | Catch, report domain as UNKNOWN, continue |
| Output write failure | Output Formatter | Exit non-zero |

**Guiding Principle**: Partial success is acceptable. A report with UNKNOWN items is better than no report.

---

## Platform Abstraction Points

To support Linux and Windows without code duplication:

1. **Inspectors** should have platform-specific implementations behind a common interface
2. **Privilege detection** requires platform-specific logic
3. **CLI parsing** is platform-agnostic
4. **Report assembly and formatting** are platform-agnostic

Suggested structure:
```
inspectors/
  network/
    interface.ts (or similar)
    linux.ts
    windows.ts
  environment/
    interface.ts
    common.ts (likely same for both platforms)
```

---

## Constraints Alignment Check

| Spec Constraint | Plan Alignment |
|-----------------|----------------|
| Read-only operation | All inspectors only read; no writes |
| No automatic privilege escalation | Privilege Detector only observes |
| No remediation | No modification modules |
| No telemetry | No network calls, no external reporting |
| No persistent state | No state files; output to stdout only |
| FACT/INFERENCE/UNKNOWN labeling | Built into DiagnosticItem model |
| Exit code 0 on partial success | Defined in execution flow |

---

## Open Questions for Implementation Phase

1. **Concurrency**: Should domain inspectors run in parallel? (Suggest: sequential for simplicity in v1)
2. **Logging**: Should diagnostic logs be written? (Spec permits; suggest: stderr with `--verbose` flag, out of scope for v1)
3. **Timeout**: Should inspectors have timeouts? (Suggest: no for v1; system calls should be fast)
4. **Encoding**: UTF-8 assumed for all output?

---

## Summary

This plan defines 8 modules with explicit responsibilities:

1. **CLI Entry Point** — argument parsing, orchestration
2. **Privilege Detector** — elevation status
3. **Mode Resolver** — flag expansion
4. **Inspector Registry** — dispatch and collection
5. **Network Inspector** — network domain data
6. **Environment Inspector** — environment domain data
7. **Report Assembler** — structure and metadata
8. **Output Formatter** — text or JSON rendering

Execution is sequential, error handling prioritizes partial success, and all spec constraints are preserved.
