# AGENTS.md

## Scope

This file applies to the repository root.

These instructions are mandatory for AI agents editing files in this repository.

## Linting Policy (Always Required)

Whenever an agent creates, edits, renames, or deletes a file, it must run the
relevant linter(s) for that file type before finishing.

If multiple file types are changed, run all corresponding linters.

## File Type → Required Linter

### Shell scripts

Applies to:

- `*.sh`
- shell scripts with shebangs (`#!/bin/bash`, `#!/usr/bin/env bash`, etc.)

Required:

- `shellcheck --enable=all <file>`

### YAML files

Applies to:

- `*.yml`
- `*.yaml`

Required:

- `yamllint <file>`

### Ansible YAML files

Applies to:

- `defaults/**/*.yml`
- `defaults/**/*.yaml`
- `meta/**/*.yml`
- `meta/**/*.yaml`
- `tasks/**/*.yml`
- `tasks/**/*.yaml`
- `tests/**/*.yml`
- `tests/**/*.yaml`
- any Ansible playbooks, task files, vars files, or metadata in this role repo

Required:

- `ansible-lint <file>` or `ansible-lint .` when role-level context is more representative
- `yamllint <file>`

Agents must run `ansible-lint` every time an Ansible file is created or
modified, including files under `tests/`, even if other repo-wide lint commands
already pass.

### Markdown files

Applies to:

- `*.md`

Required:

- `markdownlint <file>` (or `markdownlint "**/*.md"` for repo-wide validation)

### Python files

Applies to:

- `*.py`

Required:

- `ruff check <file>`

## Execution Rules

1. Lint after each meaningful change set and before final handoff.
2. Do not skip linting because a change is "small".
3. Every newly created or modified Ansible file must be validated with
   `ansible-lint` before finishing the task.
4. If a linter is unavailable, report it clearly and provide the exact install command.
5. Prefer targeted linting for changed files, then run broader linting if needed.
6. Fix lint errors introduced by the change.
7. Lint issues must be resolved in code/content; do not silence, suppress, or
   bypass rules unless an explicit, documented exception is approved.
8. For shell scripts, always run `shellcheck --enable=all` and treat reported
   findings (including info-level checks) as actionable.
9. Do not embed Python scripts/snippets inside Bash scripts or Bash command
   strings. If the task is assigned to Bash, implement it in Bash.
10. Shell automation must remain compatible with both macOS and Linux Bash.
   Avoid GNU-only flags or syntax and avoid adding dependencies on non-native
   shell tools unless the dependency is already an explicit, documented project
   requirement.
11. Do not commit user-specific absolute filesystem paths (for example,
    home-directory paths from local machines). Use repository-relative paths,
    and use `~` only when a home-relative path is genuinely required.
12. Do not hand-edit generated or vendored content under `.ansible/`; update the
    role source files in the repository root and regenerate or reinstall test
    dependencies when needed.
13. When renaming externally created live-test resources, keep cleanup tasks
    compatible with the previous names long enough to remove resources left by
    interrupted older runs.
14. When adding cloud quota or allowance preflight checks, run current-resource
    discovery first and gate only the creation path so idempotent re-runs do
    not fail when the account is already at quota.
15. Test rescue blocks must re-raise or fail after logging unless the recovered
    state is intentionally acceptable and documented in the task.
16. When parsing provider metadata booleans, compare normalized expected values
    instead of relying on broad truthiness filters for arbitrary strings.
17. When editing network configuration, replace only the route or setting owned
    by this role and preserve unrelated existing entries.
18. When changing Jenkinsfile publication or live-test behavior, keep
    `docs/jenkins-ci.md`, `docs/ansible-galaxy-release.md`, and `README.md`
    aligned with the actual split between Jenkins parameters, credential
    bindings, and Workspace commands.
19. Jenkins operator choices must remain per-build controls, not fixed
    credential-style environment values. Keep live-test enablement and target,
    release version selection, and GitHub/Galaxy publication gates as build
    parameters or an equivalent explicit Jenkins input surface.

## Changelog Policy (Always Required)

1. Whenever code or behavior is changed, update `CHANGELOG.md` in the same task.
2. Whenever documentation is added or updated, mention it in
   `CHANGELOG.md` in the same task.
3. If an `Unreleased` section exists, add changes there instead of creating a
   new dated release.
4. Do not assign or change a release date for an unreleased section unless
   requested by the user or the change is part of a release finalization
   process.
5. Only create or date a release entry when the release is actually being
   finalized.
6. Group entries under clear headings (for example: Added, Changed, Fixed)
   and keep the wording concise.

## README Update Policy (Always Required)

1. Whenever repository documentation is added, renamed, moved, or deleted,
   update the root `README.md` in the same task.
2. Keep the `README.md` table of contents aligned with the current document
   structure and available repository guidance.
3. Keep the `README.md` maintainer, support, publication, and installation
   details aligned with the current repository state.

## Suggested Commands

- Shell: `shellcheck --enable=all path/to/file.sh`
- YAML: `yamllint path/to/file.yml`
- Ansible: `ansible-lint .`
- Markdown: `markdownlint AGENTS.md README.md CHANGELOG.md TODO.md`
- Python: `ruff check path/to/file.py`

## Notes

- This policy is strict by default.
- Any exception must be explicitly documented in the task output with reason.
