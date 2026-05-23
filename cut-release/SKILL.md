---
name: cut-release
description: Project-agnostic release execution workflow for auditing, fixing, packaging, deploying, mirroring, tagging, and reporting a software release. Use when the user invokes $cut-release or asks to cut a release, prepare release notes, audit a release candidate, verify version alignment, publish artifacts, sync a public mirror, or perform release-blocker checks before tagging.
---

# Cut Release

## Overview

Use this skill to turn a repository's current state into a verified release candidate or an actual release, depending on the user's authorization and the project's release rules.

Do not start from memory. Inspect the current repository, include committed and uncommitted changes in the release scope, and treat configured public/private boundaries, artifact leaks, missing docs, failed checks, and version drift as release blockers.

When the user asks for a reusable standalone prompt, read and adapt `references/project-agnostic-release-prompt.md`.

## Release Authority

Classify the request before doing side-effectful work:

- **Audit/prep only**: Run intake, checks, notes, docs/tooling fixes, and blocker reporting. Do not tag, deploy, publish, push, or bump versions.
- **Approved release**: If the user explicitly provides or confirms the version/tag and asks to cut/publish, perform the full release flow within project rules.
- **Ambiguous version**: Do not invent a version. Discover current version surfaces, report the candidate, and stop before version bump/tag/release publication unless project instructions already approve it.

External side effects include production deploys, package publishes, image pushes, public mirror syncs, branch pushes, tags, and GitHub/GitLab releases. Perform them only when the release is explicitly approved or the repo's release instructions clearly authorize them.

## Intake

Read applicable repo instructions first: `AGENTS.md`, `CLAUDE.md`, `.codex/`, `.github/`, `docs/release*`, `RELEASE*`, `CONTRIBUTING*`, package manager files, workflow files, and deployment docs.

Gather exact scope:

```bash
git status --short --branch
git describe --tags --abbrev=0 2>/dev/null || true
git log --oneline --decorate "$(git describe --tags --abbrev=0 2>/dev/null)"..HEAD 2>/dev/null || git log --oneline --decorate -n 50
git diff --name-only
git ls-files --others --exclude-standard
```

Identify:

- Current version, intended version, release tag format, and all version surfaces.
- Package manager and canonical checks (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Makefile`, `justfile`, `Taskfile`, CI workflows).
- User-facing surfaces: docs, changelog, release notes, installers, CLIs, APIs, config files, examples, screenshots, generated assets, and public downloads.
- Artifact surfaces: archives, packages, containers, SBOM/provenance, docs sites, installers, migrations, database schema, and generated bundles.
- Boundary surfaces: public mirrors, open-source subsets, hosted/private code, paid/admin code, secrets, deployment-only config, customer/client-specific content, generated credentials, and internal docs.

Every changed Markdown file and every changed user-facing code surface must either be updated for the release or intentionally left unchanged with a reason.

## Plan The Checks

Prefer repo-native commands over invented commands. Build a check matrix from scripts and workflows already present, then run dependent checks sequentially and independent checks in parallel where safe.

Common check categories:

- Formatting, lint, static analysis, typecheck.
- Unit, integration, e2e, smoke, migration, installer, CLI, API, workflow, and packaging tests.
- Production or preview builds.
- Docs generation, changelog sync, navigation validation, link checks, examples, and install/uninstall instructions.
- Security quick checks for secrets, auth, ownership checks, data deletion, eval/dynamic execution, unsafe deserialization, telemetry content, opt-in defaults, and hashed token storage.
- Boundary/leak checks for public mirrors and release artifacts.
- Version alignment across manifests, lockfiles, docs, installers, changelogs, release data, archives, image tags, and package metadata.

Any failure is a blocker unless it is explicitly documented as an external/infrastructure limitation and the closest meaningful substitute check passes.

## Fix And Recheck

Apply simple release, docs, packaging, and tooling fixes directly. Keep diffs small and project-native.

Do not make complex production behavior changes as a hidden release chore. If such a blocker appears, either fix it with appropriate tests or stop before publishing and report the blocker.

For cleanup/refactor work discovered during release prep, preserve behavior with existing tests or add focused regression coverage before changing behavior-adjacent code.

After each fix, rerun the narrow failed check. Before claiming release readiness, rerun the full required check set or explain any skipped command with the exact reason.

## Package And Audit Artifacts

Use project-native packaging commands. If no command exists, inspect existing CI/release scripts before creating a new one.

For every artifact that will be public or customer-visible:

- Inspect contents with native listing tools (`tar -tzf`, `unzip -l`, package metadata commands, image inspection, generated file lists).
- Check for private paths, secrets, tests, migrations, customer names, pricing/product IDs, admin/billing/control-plane internals, and deployment-only config.
- Verify versioned filenames, checksums/signatures/SBOM/provenance when the project supports them.
- Verify download routes or published asset URLs when applicable.

Release artifacts that cannot be inspected are blockers unless the release instructions explicitly accept that limitation.

## Deploy, Mirror, Tag, Publish

Only proceed here for an approved release.

Before deploy, verify required production environment names exist without printing secret values. Confirm changed code fails closed when required configuration is absent.

For public mirrors or filtered releases:

- Run the dry-run/sync check first.
- Verify included and excluded path sets against the discovered boundary rules.
- Confirm generated public files do not import private modules, schedule private jobs, reference private env vars, or expose private schema/tables/routes.
- After sync, verify remote branch heads and tags.

For tags/releases:

```bash
git fetch --tags origin
git rev-parse --verify "$RELEASE_TAG" >/dev/null 2>&1 && echo "local tag exists"
git ls-remote --exit-code --tags origin "refs/tags/$RELEASE_TAG" >/dev/null 2>&1 && echo "remote tag exists"
```

Do not overwrite existing tags unless the user explicitly requests that destructive operation. Create annotated tags when the project has no contrary rule. A git tag alone is not enough when the project expects GitHub/GitLab release objects, package registry releases, or image workflows.

After publishing, verify release objects, targets, remote tags, branch heads, package/image availability, signatures, SBOM/provenance, and post-tag workflows.

## Release Notes

Draft notes from inspected evidence, not memory. Include committed changes since the last tag and relevant uncommitted changes that are part of the release lane.

Cover:

- User-visible changes.
- Fixes and security/privacy boundary changes.
- Docs, install, migration, deprecation, and compatibility notes.
- Artifact, package, image, or deployment changes.
- Known issues, skipped checks, and operational warnings.

Use the project's release-note style if one exists.

## Final Report

Report with evidence:

- Current version and release tag.
- Issues found and fixed, especially boundary/security/artifact issues.
- Changed files or committed range used for the release notes.
- Docs, changelog, installer, package, and artifact status.
- Test/typecheck/lint/build results and skipped checks with exact reasons.
- Deployment status and environment readiness result, if applicable.
- Private/public mirror branch hashes, if applicable.
- Release URLs, tag targets, package/image links, and post-release workflow state, if applicable.
- Warnings worth watching and remaining risks.

Do not call the release complete while known blockers remain.
