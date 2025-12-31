# Local Diagnostics Snapshot — Specification (v1)

## 1. Purpose

The Local Diagnostics Snapshot tool provides a clear, read-only snapshot of selected aspects of a developer's local system state.
Its goal is to replace assumptions with observed facts, explicitly highlight uncertainty, and reduce misleading conclusions during debugging, onboarding, or investigation.

The tool prioritizes correctness and transparency over completeness or automation.

---

## 2. Platform Scope

This specification targets:
- **Linux** (common distributions)
- **Windows** (Windows 10 and later)

macOS is out of scope for v1.

---

## 3. Behavior

### 3.1 Invocation

The tool is invoked as a local CLI command.

### 3.2 Inspection Modes

The tool supports the following mode flags:

| Flag | Domain | Inspection Targets |
|------|--------|-------------------|
| `--network` | Network | Active interfaces, IP addresses, default gateway, DNS configuration, listening ports |
| `--env` | Environment | Environment variables visible to the current process, PATH entries with existence checks |
| `--all` | All | All domains supported by the current version |

- If no mode flag is provided, `--all` is used by default.
- Multiple mode flags combine additively (e.g., `--network --env` inspects both domains).
- `--all` always means all domains supported by the current tool version. As new domains are added in future versions, `--all` will automatically include them.

### 3.3 Output Format

- **Default**: Human-readable plain text report to stdout
- **`--json`**: Structured JSON output to stdout (no formal schema; structure may change between versions)

### 3.4 Standard Flags

| Flag | Behavior |
|------|----------|
| `--help` | Display usage information and exit |
| `--version` | Display tool version and exit |
| `--quiet` / `-q` | Suppress progress output to stderr (errors still shown) |
| `--log-dir <path>` | Override log file location (default: system temp directory) |

### 3.5 Classification Labels

The report must clearly distinguish between:

| Label | Definition |
|-------|------------|
| **FACT** | Information obtained directly from a system call or API response |
| **INFERENCE** | Information derived or computed from one or more FACTs |
| **UNKNOWN** | Information that could not be retrieved (with reason) |

This is a strict classification: any derivation from observed data, however trivial, is an INFERENCE.

### 3.6 Output Semantics Disclaimer

Every report must include a brief disclaimer explaining:
- FACT means "observed by the tool," not "correct" or "working"
- INFERENCE means derived from observations
- UNKNOWN means the tool could not retrieve the information

Detailed semantics must also be documented in `--help` output and external documentation.

### 3.7 Execution Context Detection

The tool must detect and label (as metadata) when running in:
- A container environment
- A remote session (SSH)

This is informational labeling only, not a warning.

### 3.8 Privilege Handling

- The tool auto-detects whether it is running with elevated privileges at runtime.
- If elevated, it attempts privileged operations and labels the resulting data as obtained with elevation.
- If not elevated, it attempts all operations and reports permission-denied cases as UNKNOWN with reason.
- The tool never automatically escalates privileges.

### 3.9 Stderr Output

The tool writes to stderr:
- Progress indicators during execution
- Error messages

Use `--quiet` / `-q` to suppress progress output (errors still shown).

The tool does not modify system state.

---

## 4. Inputs

- Command-line flags (mode flags, output flags, standard flags)
- Local system state accessible to the current process
- Elevated permissions if the tool is invoked with elevation

The tool does not require network access.

---

## 5. Outputs

### 5.1 Report Structure

A diagnostic report printed to stdout containing:
- Execution context metadata (timestamp, platform, elevation status, container/SSH detection)
- Semantics disclaimer
- Domain sections (Network, Environment) based on selected modes
- Each item labeled as FACT, INFERENCE, or UNKNOWN
- Explicit indication of permission-denied cases
- Explicit indication of data obtained with elevated privileges
- Dedicated **Errors** section collecting all errors encountered during the run

### 5.2 Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | Report was produced (regardless of UNKNOWN items) |
| Non-zero | Tool-level failure (crash, invalid arguments, etc.) |

The presence of UNKNOWN items does not affect the exit code.

---

## 6. Constraints

- Read-only operation
- No automatic privilege escalation
- No remediation or "fix" actions
- No background processes
- No telemetry
- **Temporary files**: Allowed during execution if cleaned up before exit
- **Log files**: Allowed for errors and execution flow; written to system temp directory by default (configurable via `--log-dir`)

---

## 7. Non-Goals

The tool explicitly does **not**:
- Fix or modify system configuration
- Validate correctness of configuration (FACT ≠ correct)
- Guarantee completeness of reported data
- Replace human judgment
- Act as a monitoring or alerting system
- Provide a stable JSON schema (structure may change between versions)

---

## 8. Assumptions

### User
- The user is a technical user comfortable with CLI tools
- The user understands that diagnostic output may be incomplete
- The user understands that FACT means "observed," not "correct"
- The user may be operating under time pressure

### Environment
- The tool runs on the local machine (Linux or Windows)
- The tool does not have elevated privileges by default
- Some information may only be available with elevated permissions
- Data obtained with elevated privileges must be explicitly labeled
- The tool may run inside a container or remote session

### Data
- Absence of data does not imply correctness
- Some system information may be inaccessible due to permissions or OS differences
- PATH entries may reference non-existent directories

### Operational
- The tool is used for inspection, not remediation
- Incorrect certainty is worse than incomplete information

---

## 9. Success Criteria

The tool is considered successful if:
- A user can clearly distinguish what is known (FACT), derived (INFERENCE), and unavailable (UNKNOWN)
- The output reduces incorrect assumptions during debugging
- The tool never implies certainty without evidence
- The semantics disclaimer is present in every report
- The behavior matches the declared constraints and non-goals
- Errors are collected and reported in a dedicated section
- Execution context (container, SSH) is detected and labeled
