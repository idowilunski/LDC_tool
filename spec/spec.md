# Local Diagnostics Snapshot — Specification (v1)

## 1. Purpose

The Local Diagnostics Snapshot tool provides a clear, read-only snapshot of selected aspects of a developer’s local system state.
Its goal is to replace assumptions with observed facts, explicitly highlight uncertainty, and reduce misleading conclusions during debugging, onboarding, or investigation.

The tool prioritizes correctness and transparency over completeness or automation.

---

## 2. Behavior

- The tool is invoked as a local CLI command.
- It supports the following inspection modes:
  - `--network`: inspect network-related system state only
  - `--env`: inspect environment variables only
  - `--all`: inspect all supported domains
- If no mode is provided, `--all` is used by default.

For each run, the tool produces a structured, human-readable report.

The report must clearly distinguish between:
- **FACT**: information directly observed by the tool
- **INFERENCE**: information derived from observed facts
- **UNKNOWN**: information that could not be determined

The tool does not modify system state.

---

## 3. Inputs

- Command-line flags (`--network`, `--env`, `--all`)
- Local system state accessible to the current process
- Optional elevated permissions if explicitly granted by the user

The tool does not require network access.

---

## 4. Outputs

- A structured diagnostic report printed to standard output
- The report must:
  - Group information by domain (e.g., Network, Environment)
  - Label each reported item as FACT, INFERENCE, or UNKNOWN
  - Explicitly indicate when information was unavailable due to permissions
  - Explicitly indicate which data was obtained with elevated privileges (if applicable)

The output must avoid implying certainty where none exists.

---

## 5. Constraints

- Read-only operation
- No automatic privilege escalation
- No remediation or “fix” actions
- No background processes
- No persistent state or telemetry

---

## 6. Non-Goals

The tool explicitly does **not**:
- Fix or modify system configuration
- Validate correctness of configuration
- Guarantee completeness of reported data
- Replace human judgment
- Act as a monitoring or alerting system

---

## 7. Assumptions

### User
- The user is a technical user comfortable with CLI tools
- The user understands that diagnostic output may be incomplete
- The user may be operating under time pressure

### Environment
- The tool runs on the local machine
- The tool does not have elevated privileges by default
- Some information may only be available if the user explicitly runs the tool with elevated permissions
- Data obtained with elevated privileges must be explicitly labeled

### Data
- Absence of data does not imply correctness
- Some system information may be inaccessible due to permissions or OS differences

### Operational
- The tool is used for inspection, not remediation
- Incorrect certainty is worse than incomplete information

---

## 8. Success Criteria

The tool is considered successful if:
- A user can clearly distinguish what is known, inferred, and unknown
- The output reduces incorrect assumptions during debugging
- The tool never implies certainty without evidence
- The behavior matches the declared constraints and non-goals
