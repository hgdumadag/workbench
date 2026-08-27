# AI Development Workbench: Development Plan

## 1. Purpose and product principles

The Workbench will give developers one local entry point for starting an AI-assisted
development session. It will load a project's policy, validate whether a requested AI
tool is allowed, capture an appropriately redacted audit record, and launch the normal
development tools. It is an orchestration and governance layer, not a replacement for
VS Code, ChatGPT, coding agents, or model gateways.

The implementation should follow these principles:

1. **Default deny:** an unknown project, classification, tool, or logging mode fails
   closed with an actionable explanation.
2. **Policy before convenience:** policy is evaluated before prompts are copied, sent,
   or handed to another process.
3. **Data minimization:** store only what a project's logging policy permits. Never
   store model responses or credentials in version 1.
4. **Explicit capture:** prompts originate in the Workbench; do not use keyboard hooks,
   browser scraping, or clipboard monitoring.
5. **Local first:** project configuration and the journal remain local. Network access
   is only used by an explicitly selected AI client.
6. **Replaceable integrations:** launchers, providers, redactors, and reporters use
   narrow interfaces so products can change without redesigning the core.

## 2. Version 1 scope

### In scope

- A Python CLI with `workbench start`, `prompt`, `status`, `report`, `project validate`,
  and `doctor` commands.
- YAML project manifests with schema validation and safe defaults.
- Three classifications: `public`, `internal`, and `restricted`.
- Three prompt logging modes: `full`, `redacted`, and `metadata_only`.
- A local SQLite journal containing sessions, prompt events, policy decisions, and
  project snapshots.
- Git context discovery (repository root, branch, and commit) without modifying a repo.
- Pluggable launch actions for VS Code and a terminal, beginning with Windows/PowerShell.
- An explicit prompt composer that can save then copy an allowed prompt to the clipboard.
- Summary reports by date, project, tool, classification, and activity tag.
- Structured audit logs for policy denials and administrative changes.

### Explicitly deferred

- Model responses, conversation transcripts, or source-code snapshots.
- Keystroke, browser DOM, global clipboard, or background application monitoring.
- Automatic prompt submission to third-party web pages.
- LiteLLM, Open WebUI, Langfuse, browser extensions, and terminal-agent wrappers.
- Multi-user synchronization, cloud-hosted journals, and a central policy server.
- AI-generated categorization or journal summaries.

These deferrals keep the first release useful, testable, and small enough to complete
without making premature trust decisions.

## 3. Primary user journeys

### Start a governed session

1. Run `workbench start weekly-ai-briefing`.
2. Load and validate the manifest; reject unsafe or unknown values.
3. Display the classification, logging mode, allowed tools, workspace, and Azure context.
4. Require confirmation when changing cloud context or starting a restricted session.
5. Create a session record and launch the configured applications.
6. Show a session ID and the command for opening the prompt composer.

### Compose a web prompt

1. Run `workbench prompt --tool chatgpt` within the active session.
2. Re-evaluate project policy for the selected tool.
3. Collect the prompt and optional activity/tags in a local UI or terminal editor.
4. Apply the logging policy before writing any record.
5. Write the event transactionally, then copy the original prompt to the clipboard.
6. Optionally open/focus the configured website. The user pastes and submits manually.

For `metadata_only`, the prompt text must never be passed to the database layer. For
`redacted`, only the transformed value may cross that boundary.

### Review activity

1. Run `workbench report --month 2026-08`.
2. Query local, policy-permitted fields only.
3. Display totals and grouped results, clearly distinguishing prompts from sessions.
4. Permit CSV or JSON export only after warning about its sensitivity and recording the
   export event.

## 4. Configuration contract

Use versioned manifests so later schema changes can be migrated deliberately:

```yaml
schema_version: 1
project: audit-sensitive-project
classification: restricted

workspace:
  path: C:\Projects\audit-sensitive

cloud:
  azure:
    tenant: corporate
    subscription: jgs-audit-prod

ai_policy:
  allowed:
    - azure-openai
    - local-model
  prohibited:
    - chatgpt-web
    - claude-web
    - external-api

prompt_logging:
  mode: metadata_only
  redaction_profile: restricted-default

launch:
  - type: vscode
  - type: terminal
```

Validation rules include:

- `project` is a stable, unique slug; display names are separate.
- Classification and logging mode combinations must meet a centrally defined minimum.
  Restricted projects default to `metadata_only` and may never use `full`.
- An item in `prohibited` overrides an item in `allowed`.
- Paths must be absolute after environment expansion and must resolve to an approved
  local workspace; network paths are rejected initially.
- Launch action and tool identifiers come from registries, not arbitrary shell strings.
- Unknown keys are errors to prevent misspellings from silently weakening policy.
- Secrets, API keys, and tokens are forbidden fields.

Policy defaults belong in a shipped, versioned baseline. Project manifests may make the
baseline stricter but may not weaken it without an explicit, separately audited local
override. The effective policy and its hash are saved with each session.

## 5. Proposed architecture

```text
CLI / prompt UI
      |
Application services (start, compose, report, doctor)
      |
      +-- Manifest loader and schema validator
      +-- Policy engine ----> immutable decision + reason codes
      +-- Context collectors (Git, Azure, OS)
      +-- Redaction pipeline
      +-- Launch adapters (VS Code, terminal, browser)
      +-- Journal repository ----> SQLite
```

Recommended package boundaries:

```text
src/workbench/
  cli.py                 # Typer commands; no policy logic
  config/                # models, loader, schema migrations
  policy/                # effective-policy calculation and decisions
  context/               # Git, Azure CLI, OS/session context
  prompts/               # composer workflow and redaction
  launchers/             # allow-listed application adapters
  journal/               # migrations and repository interfaces
  reports/               # queries and output formats
  security/              # permissions, hashing, secret detection
tests/
  unit/
  integration/
  acceptance/
```

The domain and policy layers must not import Typer, Streamlit, SQLite, clipboard, or
provider SDKs. This permits deterministic unit tests and a later GUI without duplicating
policy behavior.

## 6. Journal data design

Use SQLite migrations from the first commit. Enable foreign keys and WAL mode; use
transactions for session and prompt-event writes.

### Core tables

- `projects`: stable project ID, slug, manifest path, and first/last seen timestamps.
- `policy_snapshots`: canonical effective policy JSON, SHA-256, schema version, and time.
- `sessions`: project and policy IDs, start/end time, working directory, Git context,
  cloud-context identifiers, application version, and status.
- `prompt_events`: session, tool, model when known, activity, tags, logging mode,
  redacted text or null, salted content hash or null, timestamp, and outcome.
- `policy_decisions`: action, resource/tool, allow/deny, reason code, policy hash, and time.
- `exports`: query range, format, destination classification, record count, and time.
- `schema_migrations`: applied migration and checksum.

Do not store credentials, access tokens, AI responses, clipboard history, files, diffs, or
full environment-variable sets. Treat hashes as potentially identifying: use a local
keyed HMAC when duplicate detection is needed, and omit it entirely when policy requires.

## 7. Security and privacy design

Before implementation, create a lightweight threat model covering prompt disclosure,
policy bypass, unsafe process invocation, manifest tampering, database theft, insecure
exports, and accidental backups.

Required controls for version 1:

- Use parameterized SQL and strict typed validation.
- Invoke allow-listed executables without `shell=True`; pass arguments as arrays.
- Resolve executable and workspace paths and reject unsafe path transitions.
- Re-check policy at the point of action, not only at session start.
- Redact before persistence; test that forbidden raw strings never reach repository mocks.
- Set restrictive file permissions on the configuration directory and database, and fail
  with guidance if permissions cannot be enforced.
- Rely on BitLocker/FileVault or an approved encrypted volume initially; document the
  residual risk. Evaluate SQLCipher separately rather than claiming SQLite is encrypted.
- Use OS/Azure credential stores and existing CLI identity. Store only non-secret context
  identifiers, and mask them in normal output where appropriate.
- Provide retention and purge commands with preview, confirmation, transactional deletion,
  and an audit event that contains no deleted prompt content.
- Prevent restricted prompt export by default; make override behavior explicit and audited.
- Ensure logs and exception traces never include prompt text.

Redaction should begin conservatively with deterministic rules for configured identifiers,
email addresses, and project-specific terms. The UI must label redaction as risk reduction,
not guaranteed anonymization, and offer a preview before copying or storing when permitted.

## 8. Delivery phases

### Phase 0: decisions and threat model (2-3 days)

- Confirm supported OS (Windows first), Python version, installation mechanism, project
  manifest location, and corporate classification rules.
- Define tool IDs, classification matrix, policy reason codes, and the data-retention policy.
- Write misuse cases and decide whether encrypted-volume protection is sufficient for the
  pilot.

**Exit criteria:** approved product brief, threat model, policy matrix, manifest schema,
and definition of done. No prompt data is collected in this phase.

### Phase 1: foundation and policy engine (1 week)

- Scaffold the Python package, linting, typing, tests, and migration framework.
- Implement manifest discovery, strict validation, baseline merging, and policy decisions.
- Implement Git and OS context collectors with graceful, visible degradation.
- Add `project validate`, `status`, and `doctor`.

**Exit criteria:** policy truth-table tests pass; malformed/unknown configuration fails
closed; commands work inside and outside Git repositories.

### Phase 2: session launcher (1 week)

- Implement session lifecycle and the SQLite repositories.
- Add safe VS Code and terminal launch adapters for Windows.
- Add Azure context inspection; make any context switch explicit and confirmable.
- Implement `workbench start` with a dry-run mode showing every planned side effect.

**Exit criteria:** an allowed project launches the intended tools and records a policy
snapshot; a prohibited or ambiguous configuration launches nothing.

### Phase 3: prompt journal (1-2 weeks)

- Implement terminal prompt composition first, then a minimal local GUI/hotkey if needed.
- Add tool selection, activities, tags, clipboard copy, and optional browser focus.
- Implement `full`, `redacted`, and `metadata_only` pipelines.
- Add retention/purge operations and sensitive-output-safe error handling.

**Exit criteria:** acceptance tests prove each logging mode, prohibited tools never receive
prompts, and raw restricted prompts do not appear in the database, logs, or traces.

### Phase 4: reporting and pilot readiness (1 week)

- Implement period, project, tool, classification, and activity reports.
- Add guarded CSV/JSON export and an export audit record.
- Package the CLI, create setup/upgrade/backup guidance, and document recovery.
- Run a small pilot using synthetic/public prompts before any internal data.

**Exit criteria:** reports reconcile with seeded data, migrations upgrade and roll back on a
copy of the database, and a pilot user can install, start, prompt, report, and purge using
the documentation.

### Phase 5: optional integrations (after pilot evidence)

Prioritize from observed friction rather than building all integrations:

1. `workbench codex` / `workbench claude` wrappers with explicit prompt capture boundaries.
2. API dispatch through an adapter, preserving the same point-of-action policy check.
3. LiteLLM as a separately deployed gateway for provider routing and budgets.
4. Open WebUI as an optional client of the gateway.
5. Local-only analytics or AI summaries over policy-permitted journal fields.
6. Langfuse only if trace-level observability becomes a demonstrated requirement.

Each integration needs its own data-flow diagram and threat-model update before release.

## 9. Testing strategy

- **Unit:** schema edge cases, policy matrices, baseline precedence, redaction, canonical
  hashes, path validation, report calculations, and reason codes.
- **Property-based:** arbitrary manifests never bypass classification invariants; arbitrary
  prompt input does not leak through metadata-only or error paths.
- **Integration:** temporary SQLite databases, real Git fixture repositories, mocked Azure
  CLI output, migration compatibility, process argument construction, and file permissions.
- **Acceptance:** start a public/internal/restricted project, attempt allowed and denied
  tools, generate each journal mode, report, export, purge, and recover from interruption.
- **Security:** dependency and secret scans, malicious YAML/path/process arguments, journal
  inspection for canary secrets, and backup/export handling.
- **Platform:** Windows and PowerShell are the release gate; add macOS/Linux CI only when
  their launch adapters become supported.

Use synthetic canary values in tests and scan the database, logs, temporary directories,
and captured stderr to demonstrate non-disclosure.

## 10. Operational requirements

- A `doctor` command checks Python/app versions, manifest/database permissions, SQLite
  integrity, Git/VS Code/Azure CLI availability, workspace paths, and migration state.
- Database backup is local, explicit, encrypted by the destination volume, and never runs
  while a partial transaction can be copied. Restore is tested before the pilot.
- Migrations create a backup and refuse to proceed when checksums or free-space checks fail.
- Structured logs contain event IDs and reason codes, not prompt text.
- Semantic application versions and manifest/database schema versions are independent.
- Telemetry is off by default; version 1 has no remote telemetry endpoint.

## 11. Product metrics and pilot feedback

Measure usefulness without collecting prompt content:

- Successful versus denied session starts, grouped by reason code.
- Percentage of AI prompts initiated through the Workbench (user-reported initially).
- Time from `start` to ready, and from composer open to clipboard copy.
- Validation and launcher failure rates.
- Number of policy exceptions requested.
- User confidence in classification and logging behavior.

Do not optimize for raw prompt volume. The pilot succeeds when users consistently begin
governed sessions, understand the active policy, and can find prior activity without
creating an unacceptable sensitive-data copy.

## 12. Key decisions required before coding

1. What exact corporate classifications and AI-provider rules are authoritative?
2. Is Windows the only initial target, and may the CLI depend on PowerShell 7?
3. Where may manifests, the database, backups, and exports reside?
4. Is encrypted-disk protection sufficient, or is SQLCipher required before restricted use?
5. Which fields must be retained, for how long, and who may purge/export them?
6. Should Azure context be verified only, or may the Workbench change it?
7. Is a terminal composer acceptable for the pilot, or is a global hotkey/UI a release gate?
8. Which tool identifiers and browser URLs are approved in each classification?

Until these are resolved, implementation can use synthetic policy fixtures, but restricted
real-world prompts should not be admitted.

## 13. Definition of done for version 1

Version 1 is complete when:

- A clean machine can install and validate the application from documented steps.
- Every action resolves a valid effective policy and records a stable reason code.
- Public, internal, and restricted project acceptance suites pass.
- No raw prompt persists in `metadata_only`, and only transformed text persists in
  `redacted`; canary scans confirm this across storage and logs.
- Launch commands cannot execute arbitrary manifest-provided shell content.
- Reports reconcile exactly with seeded journal events and respect policy restrictions.
- Backup, migration, purge, and recovery procedures have been exercised.
- Security review findings rated critical or high are resolved.
- A synthetic/public-data pilot completes successfully before internal or restricted use.

The next release should be selected from pilot evidence. A gateway or unified UI should
not be treated as mandatory unless the core launcher, policy, and journal workflow has
proved useful.
