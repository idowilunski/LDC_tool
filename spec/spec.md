# Local Diagnostics Snapshot — Specification (v1)

## 1. Purpose

The Local Diagnostics Snapshot tool provides a clear, read-only snapshot of selected aspects of a developer's local system state.
Its goal is to replace assumptions with observed facts, explicitly highlight uncertainty, and reduce misleading conclusions during debugging, onboarding, or investigation.

The tool prioritizes correctness and transparency over completeness or automation.

---

## 2. Platform Scope

This specification targets:
- Linux (common distributions)
- Windows (Windows 10 and later)

macOS is out of scope for v1.

---

## 3. Behavior

### 3.1 Invocation

The tool is invoked as a local CLI command.

### 3.2 Inspection Modes

The tool supports the following inspection modes:

| Flag | Domain | Inspection Targets |
|------|--------|-------------------|
| `--network` | Network | Active interfaces, IP addresses, default gateway, DNS configuration, listening ports |
| `--env` | Environment | Environment variables visible to the current process, PATH entries with existence checks |
| `--all` | All | All domains supported by the current version |

- If no mode flag is provided, `--all` is used by default.
- Multiple mode flags combine additively.
- `--all` always refers to all inspection domains supported by the current version.

### 3.3 Output

The tool produces a structured, human-readable diagnostic report printed to stdout.

By default, the output is plain text intended for direct human consumption.

If the `--json` flag is provided, the tool outputs structured JSON to stdout.
No formal or stable JSON schema is guaranteed.

The report must clearly distinguish between:
- **FACT**: information obtained directly from the system
- **INFERENCE**: information derived from one or more FACTs
- **UNKNOWN**: information that could not be retrieved, with reason

Observed information does not imply correctness or functionality.

Each report must include a brief disclaimer explaining that:
- FACT indicates observation, not correctness
- INFERENCE indicates derived information
- UNKNOWN indicates missing or inaccessible information

### 3.4 Privilege Handling

- The tool does not escalate privileges automatically.
- If run with elevated privileges, additional information may be collected.
- Data obtained with elevated privileges must be explicitly labeled.
- Permission-denied cases must be reported as UNKNOWN with reason.

---

## 4. Inputs

- Command-line flags
- Local system state accessible to the current process
- Elevated permissions if explicitly granted by the user

The tool does not require network access.

---

## 5. Outputs

- A diagnostic report grouped by inspection domain
- Each reported item labeled as FACT, INFERENCE, or UNKNOWN
- Explicit indication of permission-related limitations

### 5.1 Exit Codes

- Exit code `0`: A report was successfully produced, even if some items are UNKNOWN
- Non-zero exit code: Tool-level failure (e.g., invalid arguments, unrecoverable error)

---

## 6. Constraints

- Read-only operation
- No automatic privilege escalation
- No remediation or system modification
- No telemetry
- No persistent state  
  (temporary files must be cleaned up; diagnostic log files are permitted)

---

## 7. Non-Goals

The tool explicitly does **not**:
- Fix or modify system configuration
- Validate correctness of configuration
- Guarantee completeness of reported data
- Replace human judgment
- Act as a monitoring or alerting system
- Guarantee a stable machine-readable output schema

---

## 8. Assumptions

### User
- The user is a technical CLI user
- Diagnostic output may be incomplete
- FACT means "observed", not "correct"
- The user may be operating under time pressure

### Environment
- The tool runs on the local machine
- Some information may require elevated privileges
- Execution context may affect visibility of system data

### Data
- Absence of data does not imply correctness
- Some system information may be inaccessible

### Operational
- The tool is used for inspection, not remediation
- Incorrect certainty is worse than incomplete information

---

## 9. Success Criteria

The tool is successful if:
- Users can clearly distinguish FACT, INFERENCE, and UNKNOWN
- Output reduces incorrect assumptions
- The tool never implies certainty without evidence
- Behavior matches declared constraints and non-goals
