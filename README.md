# Codex-CLI
how to set up codex cli and program it to store a local memory bank for project consistency.
(Yes you have to fill in the blanks for yourself)

# Universal Codex CLI Local Memory Setup

## How to Make Codex CLI Keep Up With a Large Project

For long-running projects, do not rely on one enormous chat or automatic memory alone.

A reliable Codex project uses four layers:

1. **A local Codex home directory** for saved state, transcripts, configuration, and memories.
2. **Codex local memories** for recalling useful information from previous sessions.
3. **An `AGENTS.md` file** containing permanent project instructions.
4. **Project-memory documents inside the repository** recording the verified state of the project.

OpenAI recommends keeping required project guidance in `AGENTS.md` or checked-in documentation. Automatic memories should be treated as a recall aid, not the only source of critical rules.

---

# 1. Choose a Storage Layout

Use two separate locations:

```text
<PROJECT_ROOT>
<CODEX_STORAGE_ROOT>/<PROJECT_NAME>
```

Example on Windows:

```text
D:\Projects\ExampleApp
D:\CodexHomes\ExampleApp
```

Example on macOS or Linux:

```text
/home/user/projects/example-app
/home/user/.codex-projects/example-app
```

Do not place the project-specific Codex home inside the Git repository unless its contents are properly excluded from Git.

Codex stores local state under `CODEX_HOME`. When `CODEX_HOME` is not set, Codex uses `~/.codex`. Common stored state can include configuration, local history, memories, logs, caches, and authentication information.

## Shared versus isolated storage

There are two valid approaches.

### Shared Codex home

```text
~/.codex
```

This is easier and allows Codex to share memories across projects.

### Separate Codex home for each project

```text
~/.codex-projects/project-a
~/.codex-projects/project-b
```

This provides stronger separation and reduces the risk of Codex mixing unrelated project context.

For large, private, or long-term projects, separate project-specific Codex homes are usually the safer choice.

---

# 2. Create a Project Launcher

The launcher ensures that Codex:

* Opens in the correct repository.
* Uses the correct local storage.
* Does not accidentally mix memories between projects.

## Windows PowerShell launcher

Open the PowerShell profile:

```powershell
if (-not (Test-Path $PROFILE)) {
    New-Item -ItemType File -Force $PROFILE | Out-Null
}

notepad $PROFILE
```

Add this function:

```powershell
function Start-ProjectCodex {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]$ProjectRoot,

        [Parameter(Mandatory = $true)]
        [string]$CodexHome
    )

    if (-not (Test-Path -LiteralPath $ProjectRoot)) {
        throw "Project directory does not exist: $ProjectRoot"
    }

    New-Item `
        -ItemType Directory `
        -Force `
        -Path $CodexHome | Out-Null

    $previousCodexHome = $env:CODEX_HOME
    $locationPushed = $false

    try {
        $env:CODEX_HOME = $CodexHome

        Push-Location -LiteralPath $ProjectRoot
        $locationPushed = $true

        codex
    }
    finally {
        if ($locationPushed) {
            Pop-Location
        }

        $env:CODEX_HOME = $previousCodexHome
    }
}
```

Reload the profile:

```powershell
. $PROFILE
```

Launch a project:

```powershell
Start-ProjectCodex `
    -ProjectRoot "D:\Projects\ExampleApp" `
    -CodexHome "D:\CodexHomes\ExampleApp"
```

Create a separate launcher invocation for each major project.

---

## macOS or Linux launcher

Open the appropriate shell configuration file:

```bash
nano ~/.zshrc
```

Or:

```bash
nano ~/.bashrc
```

Add this function:

```bash
start_project_codex() {
    local project_root="$1"
    local codex_home="$2"

    if [ -z "$project_root" ] || [ -z "$codex_home" ]; then
        echo "Usage: start_project_codex <project-root> <codex-home>"
        return 1
    fi

    if [ ! -d "$project_root" ]; then
        echo "Project directory does not exist: $project_root"
        return 1
    fi

    mkdir -p "$codex_home" || return 1

    (
        export CODEX_HOME="$codex_home"
        cd "$project_root" || exit 1
        codex
    )
}
```

Reload the shell:

```bash
source ~/.zshrc
```

Or:

```bash
source ~/.bashrc
```

Launch a project:

```bash
start_project_codex \
    "$HOME/projects/example-app" \
    "$HOME/.codex-projects/example-app"
```

---

# 3. Configure Local Memories and History

The configuration file is:

```text
<CODEX_HOME>/config.toml
```

Example on Windows:

```text
D:\CodexHomes\ExampleApp\config.toml
```

Example on macOS or Linux:

```text
~/.codex-projects/example-app/config.toml
```

Add:

```toml
[features]
memories = true

[memories]
generate_memories = true
use_memories = true
disable_on_external_context = false

# Optional extended retention for long-running projects.
max_rollout_age_days = 90
max_unused_days = 365
max_raw_memories_for_consolidation = 1024

[history]
persistence = "save-all"

# 200 MiB maximum local history file size.
max_bytes = 209715200
```

Do not create duplicate TOML sections. When `[features]`, `[memories]`, or `[history]` already exists, add the settings to the existing section.

The current Codex configuration supports local memory generation and use, configurable retention periods, consolidation limits, and local transcript persistence. Local memories are disabled by default until the feature is enabled.

## What the settings mean

### `generate_memories`

Allows eligible new sessions to become inputs for future memory generation.

### `use_memories`

Allows Codex to inject relevant existing memories into future sessions.

### `disable_on_external_context`

When set to `true`, sessions using outside context such as web search or MCP tools are excluded from memory generation.

Setting it to `false` allows those sessions to remain eligible.

### `max_rollout_age_days`

Controls how old a previous session can be and still qualify for memory generation.

### `max_unused_days`

Controls how long an unused memory remains eligible for consolidation.

### `max_raw_memories_for_consolidation`

Controls how many recent raw memories Codex can retain for consolidation.

### `history.persistence`

Controls whether local session transcripts are saved.

### `history.max_bytes`

Limits the local history file size. When the file exceeds the limit, Codex removes older entries while preserving newer history.

## Important memory behavior

Memory generation is not necessarily immediate. Codex can wait until a session has been idle before processing it and may skip memory processing when remaining usage is below its configured threshold.

Do not manually edit generated memory files as the primary way of controlling project knowledge. They should be treated as generated state.

---

# 4. Make the Project a Git Repository

Codex normally treats the Git root as the project root.

From the project folder, run:

```bash
git rev-parse --show-toplevel
```

If the repository already exists, Git prints its root directory.

When it is not yet a Git repository, initialize it only after reviewing the directory:

```bash
git init
```

Before adding files, create an appropriate `.gitignore` for the project’s language, engine, framework, build system, generated files, credentials, and large binary assets.

Then review:

```bash
git status
```

Do not blindly run `git add .` in an unfamiliar project. First confirm that secrets, build output, caches, downloaded models, temporary files, and large generated assets are excluded.

Codex starts instruction discovery at the project root, which is typically the Git root, and works down toward the current working directory.

---

# 5. Create the Permanent `AGENTS.md`

Start Codex in the project root:

```bash
codex
```

Inside Codex, run:

```text
/init
```

The `/init` command creates an `AGENTS.md` scaffold in the current directory. Review and customize it before depending on it.

Replace or expand the generated file with the following template.

```markdown
# Project Instructions

## Project Identity

Project name: [PROJECT NAME]

Primary purpose:

[Describe what the project is intended to accomplish.]

Primary users:

[Describe who will use the project.]

Current lifecycle stage:

[Planning, prototype, active development, production, maintenance, etc.]

## Non-Negotiable Constraints

- Do not make changes outside the requested scope.
- Do not silently redesign existing systems.
- Do not remove existing functionality without explicit approval.
- Do not add paid services, paid dependencies, or subscriptions without approval.
- Do not add production dependencies without explaining the need.
- Do not delete source files, user-created assets, data, backups, or documentation
  without explicit approval.
- Do not store secrets, passwords, API keys, private certificates, or access
  tokens in tracked files.
- Prefer small, reversible changes.
- Create checkpoints before migrations, destructive operations, or major
  architectural changes.
- Clearly label assumptions.
- Never report unfinished or unverified work as complete.

## Required Reading Before Work

Before making substantial changes:

1. Read this file completely.
2. Read `docs/project-memory/CURRENT_STATE.md`.
3. Read `docs/project-memory/ARCHITECTURE.md`.
4. Read `docs/project-memory/DECISIONS.md`.
5. Read `docs/project-memory/ROADMAP.md`.
6. Read `docs/project-memory/SOURCE_INDEX.md` when the task depends on external
   documents, specifications, datasets, manuscripts, references, or assets.
7. Inspect `git status`.
8. Inspect relevant recent commits.
9. Inspect the existing implementation before proposing a replacement.

Do not assume that a documented feature exists. Verify it in the repository.

## Scope Control

For every task:

1. Restate the requested outcome.
2. Identify the files likely to be affected.
3. Identify risks and dependencies.
4. Make only the minimum changes necessary.
5. Do not extend the task into unrelated cleanup or refactoring.
6. Stop and explain before making an irreversible or destructive change.
7. Preserve existing behavior unless the task explicitly changes it.

## Development Rules

- Follow the repository’s existing architecture and conventions.
- Prefer maintainable solutions over unnecessary complexity.
- Avoid duplicate implementations.
- Avoid placeholder code being presented as production-ready.
- Add or update tests for important behavior.
- Add error handling for expected failure cases.
- Document non-obvious behavior.
- Verify external APIs, packages, commands, and configuration before relying on
  them.
- Avoid upgrading unrelated dependencies during a focused task.

## Required Verification

After modifying code or configuration:

1. Run relevant tests.
2. Run the appropriate formatter.
3. Run linting.
4. Run type checking when applicable.
5. Run the build or compile check.
6. Inspect the resulting diff.
7. Confirm that unrelated files were not changed.
8. Report any check that could not be run and explain why.

## Required Project-Memory Updates

After every substantial task:

1. Update `docs/project-memory/CURRENT_STATE.md`.
2. Record architectural or behavioral decisions in
   `docs/project-memory/DECISIONS.md`.
3. Update `docs/project-memory/ROADMAP.md` when priorities or milestones change.
4. Add a dated entry to `docs/project-memory/SESSION_LOG.md`.
5. Update `docs/project-memory/ARCHITECTURE.md` when system structure changes.
6. Update `docs/project-memory/SOURCE_INDEX.md` when new source material is
   introduced.

## Required Closeout Report

At the end of every substantial task, report:

- What was requested.
- What was changed.
- Which files were changed.
- What was tested.
- The test results.
- Known problems or limitations.
- Assumptions made.
- Decisions requiring user approval.
- The exact recommended next task.

## Project-Specific Constraints

Add the project’s actual constraints here, such as:

- Supported operating systems.
- Hardware limitations.
- Required language or framework versions.
- Performance budgets.
- Accessibility requirements.
- Offline operation requirements.
- Data-retention rules.
- Security requirements.
- Deployment restrictions.
- Licensing restrictions.
- Backward-compatibility requirements.
```

Codex reads `AGENTS.md` before beginning work. It can combine global instructions from `CODEX_HOME` with repository instructions and more specific nested instructions. Files closer to the active directory take precedence over broader instructions.

By default, the combined project-instruction limit is 32 KiB. Keep `AGENTS.md` focused and move detailed specifications, architecture, logs, and project history into separate documents.

---

# 6. Create the Durable Project-Memory Folder

Inside the repository, create:

```text
docs/
└── project-memory/
    ├── CURRENT_STATE.md
    ├── ARCHITECTURE.md
    ├── DECISIONS.md
    ├── ROADMAP.md
    ├── SESSION_LOG.md
    ├── SOURCE_INDEX.md
    ├── BACKLOG.md
    └── GLOSSARY.md
```

## Windows PowerShell

```powershell
$MemoryDirectory = "docs\project-memory"

New-Item `
    -ItemType Directory `
    -Force `
    -Path $MemoryDirectory | Out-Null

$MemoryFiles = @(
    "CURRENT_STATE.md",
    "ARCHITECTURE.md",
    "DECISIONS.md",
    "ROADMAP.md",
    "SESSION_LOG.md",
    "SOURCE_INDEX.md",
    "BACKLOG.md",
    "GLOSSARY.md"
)

foreach ($File in $MemoryFiles) {
    New-Item `
        -ItemType File `
        -Force `
        -Path (Join-Path $MemoryDirectory $File) | Out-Null
}
```

## macOS or Linux

```bash
mkdir -p docs/project-memory

touch \
  docs/project-memory/CURRENT_STATE.md \
  docs/project-memory/ARCHITECTURE.md \
  docs/project-memory/DECISIONS.md \
  docs/project-memory/ROADMAP.md \
  docs/project-memory/SESSION_LOG.md \
  docs/project-memory/SOURCE_INDEX.md \
  docs/project-memory/BACKLOG.md \
  docs/project-memory/GLOSSARY.md
```

---

# 7. Use Standard Project-Memory Templates

## `CURRENT_STATE.md`

```markdown
# Current Project State

Last verified:

Verified by:

## Working

- [Confirmed working functionality]

## Partially Implemented

- [Feature]
  - Working:
  - Missing:
  - Known problems:

## Not Implemented

- [Planned but absent functionality]

## Broken or Blocked

- [Problem]
  - Cause:
  - Evidence:
  - Required resolution:

## Current Test Status

- Unit tests:
- Integration tests:
- Build:
- Lint:
- Type check:
- Manual verification:

## Active Risks

- [Risk]

## Recommended Next Task

[One exact, dependency-aware next task]
```

## `ARCHITECTURE.md`

```markdown
# Architecture

Last verified:

## System Overview

[Describe the system at a high level.]

## Main Components

### Component Name

Purpose:

Responsibilities:

Inputs:

Outputs:

Dependencies:

Source location:

## Data Flow

[Describe how information moves through the system.]

## Storage

[Describe files, databases, caches, remote storage, and ownership.]

## External Services

[Describe external APIs and services.]

## Build and Deployment

[Describe how the project is built and deployed.]

## Security Boundaries

[Describe trust boundaries, credentials, permissions, and sensitive data.]

## Known Architectural Debt

- [Issue]
```

## `DECISIONS.md`

```markdown
# Decision Log

## Decision Template

### YYYY-MM-DD — Decision Title

Status: Proposed | Accepted | Replaced | Rejected

Context:

Decision:

Reasoning:

Alternatives considered:

Consequences:

Affected files or systems:

Replaces:
```

## `ROADMAP.md`

```markdown
# Project Roadmap

## Current Milestone

Goal:

Completion criteria:

Dependencies:

## Milestones

### Milestone 1

Objective:

Required work:

Completion criteria:

Risks:

Status:

## Deferred Work

- [Deferred item and reason]
```

## `SESSION_LOG.md`

```markdown
# Session Log

## YYYY-MM-DD — Session Title

Requested work:

Completed:

Files changed:

Verification performed:

Results:

Decisions made:

Known issues:

Next recommended task:
```

## `SOURCE_INDEX.md`

```markdown
# Source Index

## Source Template

### Source Name

Location:

Type:

Purpose:

Authority level:

Last reviewed:

Relevant areas:

Notes:
```

## `BACKLOG.md`

```markdown
# Backlog

## Immediate

- [ ] Task

## Near Term

- [ ] Task

## Later

- [ ] Task

## Blocked

- [ ] Task
  - Blocked by:
```

## `GLOSSARY.md`

```markdown
# Project Glossary

## Term

Definition:

Related systems:

Source of truth:
```

---

# 8. Give Codex Its First Audit Task

After creating the files, give Codex this prompt:

```text
Read AGENTS.md completely.

Audit the entire repository without modifying production code.

Populate the documents under docs/project-memory:

- CURRENT_STATE.md
- ARCHITECTURE.md
- DECISIONS.md
- ROADMAP.md
- SESSION_LOG.md
- SOURCE_INDEX.md
- BACKLOG.md
- GLOSSARY.md

Requirements:

1. Verify every claim against actual repository files.
2. Inspect the Git working tree and relevant recent commits.
3. Identify what currently works.
4. Identify what is incomplete.
5. Identify what is missing.
6. Identify broken paths, dead code, duplicate systems, placeholders, stale
   documentation, and unverified assumptions.
7. Clearly distinguish verified facts from assumptions.
8. Do not modify production code during this audit.
9. Do not install dependencies.
10. Do not redesign the project.
11. End with one exact recommended next task.

The repository, not previous conversation memory, is the source of truth.
```

This initial audit creates a verified baseline instead of asking Codex to reconstruct the project from uncertain conversational memory.

---

# 9. Enable Memories in Codex

Inside an interactive Codex session, run:

```text
/memories
```

Enable:

```text
Use existing memories
Generate new memories
```

The `/memories` command controls whether the session can use existing local memories and whether it can contribute to future memories.

For long sessions, also establish a persistent goal:

```text
/goal Continue developing this project while following AGENTS.md, preserving
verified existing behavior, and keeping docs/project-memory current.
```

A session goal helps keep active work focused, but it does not replace `AGENTS.md` or repository documentation.

---

# 10. Resume the Correct Session

Always enter the correct project folder and use the same `CODEX_HOME`.

Then run:

```bash
codex resume --last
```

Codex scopes `--last` to the current working directory unless `--all` is supplied.

To choose from available sessions:

```bash
codex resume
```

To include sessions from other working directories:

```bash
codex resume --all
```

Be careful with `--all`, because it can expose sessions belonging to unrelated projects.

---

# 11. Verify That Instructions Are Loading

From the repository root, run:

```bash
codex --ask-for-approval never \
  "Summarize the active instructions and list the instruction files you loaded."
```

Codex should report:

1. Global instructions from `CODEX_HOME`, when present.
2. The repository-root `AGENTS.md`.
3. Any more specific nested `AGENTS.md` or `AGENTS.override.md` files.

Codex rebuilds its instruction chain at the beginning of each run or interactive session. Restart Codex after changing instruction files.

Also verify the current storage location.

## PowerShell

```powershell
$env:CODEX_HOME
```

## macOS or Linux

```bash
echo "$CODEX_HOME"
```

Verify the repository root:

```bash
git rev-parse --show-toplevel
```

These two values must point to the intended Codex storage and intended project.

---

# 12. Add Optional Global Instructions

A user can place universal working preferences in:

```text
<CODEX_HOME>/AGENTS.md
```

Example:

```markdown
# Global Codex Working Agreements

- Inspect existing code before creating replacements.
- Do not make unrelated changes.
- Explain destructive operations before executing them.
- Do not add paid services without approval.
- Do not expose secrets.
- Run relevant verification after changes.
- Clearly separate verified facts from assumptions.
- Prefer small, reversible commits.
```

Project-specific rules should remain in the repository’s `AGENTS.md`.

Codex loads global guidance from its home directory first and then layers project-specific guidance over it.

---

# 13. Keep the Memory System Healthy

At the beginning of a substantial task, instruct Codex to:

```text
Read AGENTS.md and the required project-memory documents before making changes.
Verify CURRENT_STATE.md against the repository if the relevant area may have
changed.
```

At the end of every substantial task, instruct Codex to:

```text
Update the project-memory documents required by AGENTS.md. Record only verified
facts. Run the required checks and provide a complete closeout report.
```

Periodically request a memory audit:

```text
Audit docs/project-memory against the current repository.

Identify stale statements, contradictions, completed backlog items, missing
decisions, undocumented architecture changes, and incorrect next steps.

Update documentation only. Do not modify production code.
```

Automatic memory can become stale. The project-memory documents must be regularly verified against the actual repository.

---

# 14. Back Up the Project Correctly

Back up both:

```text
<PROJECT_ROOT>
<CODEX_HOME>
```

The repository contains the authoritative code and documentation.

The Codex home contains local configuration, memory, transcripts, logs, caches, and potentially authentication information.

Before sharing or transferring a Codex home:

* Review its contents.
* Remove or protect authentication files.
* Do not publish `auth.json`.
* Do not commit Codex storage to a public repository.
* Treat session history and memory as potentially sensitive.
* Sign in normally on a replacement computer rather than casually copying credentials.

On the new machine:

1. Restore the repository.
2. Restore the noncredential parts of the project’s Codex home.
3. Install Codex CLI.
4. Sign in.
5. Set the same `CODEX_HOME`.
6. Enter the repository root.
7. Verify `AGENTS.md` discovery.
8. Run `codex resume`.
9. Audit the project-memory documents before continuing development.

---

# 15. Common Mistakes

## Depending only on automatic memory

Memory is not a guaranteed source of truth. Important rules and current project state belong in repository files.

## Using one Codex home for unrelated projects

This can cause unrelated memories, histories, and configuration to overlap.

## Starting Codex from the wrong directory

The current directory affects project-root discovery, instruction loading, and session resume behavior.

## Making `AGENTS.md` enormous

Codex reads a limited amount of project guidance. Keep permanent rules concise and place detailed information in referenced documents.

## Never updating `CURRENT_STATE.md`

A stale project-state file can be worse than having no state file.

## Letting Codex update documentation without verification

Documentation should reflect actual files, test results, commits, and observed behavior—not assumptions.

## Manually editing generated memories

Generated memory files are implementation state, not the primary project-control interface.

## Allowing silent scope expansion

The project instructions should explicitly prohibit unrelated cleanup, redesigns, dependency upgrades, and speculative changes.

## Losing the Codex home during migration

Restoring the repository alone does not restore local Codex memory and history.

---

# Recommended Operating Rule

Use this hierarchy whenever sources disagree:

1. **Current repository files and test results**
2. **Approved project decisions**
3. **Project-memory documentation**
4. **`AGENTS.md` rules**
5. **Current user instructions**
6. **Automatic Codex memories**
7. **Old chat transcripts**

The repository records what is true.

`AGENTS.md` records how Codex must work.

Project-memory documents record what has been verified.

Automatic memory helps Codex recall useful context.

Together, these provide much more reliable project continuity than a single long-running chat.
