# Project-Agnostic Release Audit Prompt

Use this prompt to audit, prepare, and cut a software release in any repository. Replace bracketed placeholders with project-specific values when known. If a placeholder is unknown, inspect the repository before deciding.

## Prompt

You are cutting a release for the current project.

Do not start from memory. Inspect the current repository, include every committed and uncommitted release-lane change in the release notes, and treat configured public/private boundaries, artifact leaks, missing docs, failed checks, and version drift as release blockers.

Never bump versions, tag, deploy, publish, push release branches, sync public mirrors, or create release objects unless the release version and side effects are explicitly approved by the user or the repository's release instructions.

### Project Configuration

Fill this section from repo evidence before acting:

- Private repository: `[private repo or N/A]`
- Public repository or mirror: `[public repo or N/A]`
- Release version: `[approved version or unknown]`
- Release tag: `[tag format, for example vX.Y.Z]`
- Primary branch: `[main/master/trunk]`
- Stable/release branch: `[stable/release branch or N/A]`
- Package managers: `[npm/pnpm/yarn/pip/uv/poetry/go/cargo/etc.]`
- Deployment targets: `[hosting/platforms or N/A]`
- Public artifacts: `[packages, archives, images, docs, installers, generated assets]`
- Private-only surfaces: `[hosted app, billing, admin, cloud control plane, secrets, internal docs, client-specific content]`
- Public/releasable surfaces: `[docs, plugin, library, CLI, SDK, self-hosted runtime, examples, public schemas]`
- Boundary/leak rules: `[paths, imports, env vars, schema tables, product IDs, client names, private docs]`
- Required post-release workflow checks: `[CI workflows, image publish, provenance, registry, docs deploy]`

### Step 1 - Repo Intake

Gather exact release scope:

```bash
git status --short --branch
git describe --tags --abbrev=0 2>/dev/null || true
git log --oneline --decorate "$(git describe --tags --abbrev=0 2>/dev/null)"..HEAD 2>/dev/null || git log --oneline --decorate -n 50
git diff --name-only
git ls-files --others --exclude-standard
```

Read applicable release instructions:

```bash
rg --files -g 'AGENTS.md' -g 'CLAUDE.md' -g 'RELEASE*' -g 'CHANGELOG*' -g 'CONTRIBUTING*' -g 'package.json' -g 'pyproject.toml' -g 'go.mod' -g 'Cargo.toml' -g 'Makefile' -g 'justfile' -g 'Taskfile*' -g '.github/workflows/*'
rg -n -i 'release|publish|deploy|version|changelog|mirror|artifact|provenance|sbom|secret|private|public' .
```

Identify every changed Markdown file and every changed user-facing code surface. Each must either be updated for the release or intentionally left unchanged with a reason.

### Step 2 - Health Checks

Build a repo-native check matrix from package scripts, CI workflows, release docs, and build files. Run dependent checks sequentially and independent checks in parallel where safe.

Recommended categories:

1. Formatting, lint, static analysis, and typecheck.
2. Unit, integration, e2e, smoke, migration, installer, CLI, API, workflow, and packaging tests.
3. Production or preview builds.
4. Docs generation, changelog sync, navigation validation, link checks, examples, and install/uninstall instructions.
5. Security quick checks.
6. Public/private boundary and artifact leak checks.
7. Version and branch alignment checks.

Any failure blocks the release unless it is explicitly documented as an external/infrastructure limitation and the closest available substitute check passes.

### Step 3 - Generic Command Discovery

Prefer commands already defined by the project:

```bash
test -f package.json && node -e "const p=require('./package.json'); console.log(p.scripts || {})"
test -f pyproject.toml && sed -n '1,220p' pyproject.toml
test -f Makefile && sed -n '1,220p' Makefile
test -f justfile && sed -n '1,220p' justfile
find .github/workflows -maxdepth 1 -type f -print 2>/dev/null
```

Common fallback commands, only when applicable:

```bash
npm test
npm run lint
npm run typecheck
npm run build
pnpm test
pnpm lint
pnpm typecheck
pnpm build
pytest
ruff check .
mypy .
go test ./...
cargo test
cargo clippy --all-targets --all-features
```

### Step 4 - Docs Contract

Verify user-facing docs match the release:

- Changelog exists and has the approved version/date.
- Release notes cover all committed changes since the last tag and relevant uncommitted release-lane changes.
- Install, uninstall, upgrade, migration, integration, and self-hosted/deployment docs reflect changed behavior.
- Docs navigation and generated docs indexes include new pages.
- README, package docs, CLI help, examples, and website copy agree on version and commands.
- Release date is on or before today's date.
- Removed or renamed features have migration/deprecation notes.

### Step 5 - Boundary And Leak Checks

If the project has a public mirror, open-source subset, customer-facing artifact, self-hosted bundle, package, container, docs site, or generated installer, treat boundary validation as a release blocker.

Verify public/releasable outputs include only approved surfaces and exclude:

- `.env*`, generated secrets, private tokens, private keys, and deployment-only config.
- Billing, pricing, admin, hosted control-plane, internal auth, tenant provisioning, and support tooling unless explicitly public.
- Internal docs, agent instructions, private migrations, local state, private scripts, and generated credentials.
- Client/customer names, private product IDs, private URLs, support-assistant names, and client-specific examples.
- Tests, fixtures, debug dumps, source maps, or migrations that the project does not intend to ship.

Use native listing tools for artifacts:

```bash
tar -tzf "[artifact.tar.gz]" | rg -n "[private path or token patterns]"
unzip -l "[artifact.zip]" | rg -n "[private path or token patterns]"
```

Both leak-check commands must return no matches for private patterns.

For public mirrors or filtered trees, run the dry-run first when available and verify:

- Intended public paths are included.
- Private paths are excluded or rewritten away.
- Public imports do not reference private modules.
- Public cron/jobs/routes/schema do not schedule or expose private hosted functionality.
- Public docs do not mention private env vars, pricing IDs, customer names, or internal deployment details.

### Step 6 - Security Quick Check

Search for high-signal issues and inspect changed security-sensitive paths:

```bash
rg -n -i 'sk-|api[_-]?key|secret|token|BEGIN .*PRIVATE|password|passwd|eval\(|new Function|dangerouslySetInnerHTML|TODO.*security|FIXME.*security|PUBLIC_.*SECRET|PRIVATE_.*PUBLIC' .
```

Verify applicable invariants:

- User data mutations derive identity from trusted auth or validated API-key context.
- Read/update/delete paths enforce ownership or tenant boundaries.
- Permanent delete and archive/restore paths are covered by tests or smoke checks.
- Telemetry is opt-in where required and never sends sensitive content when it should send stats only.
- Email, webhooks, analytics, and background jobs are opt-in or fail closed.
- Provider keys use provider-specific env names; client bearer auth uses a project-specific API key name.
- Bootstrap tokens, API keys, and reset codes are stored hashed; cleartext is returned only once.
- Dynamic execution, deserialization, file extraction, and shell calls are constrained.

### Step 7 - Version And Branch Alignment

Approved version must match every project-defined surface, commonly:

- Package manifests and lockfiles.
- CLI/plugin/app manifests.
- README, docs introduction, install scripts, changelog, release data, generated docs, examples.
- Archive names, image tags, Helm charts, Terraform variables, templates, and package metadata.
- Primary and stable/release branch expectations.

Do not invent version changes. If the version is not approved, report drift and stop before bumping/tagging.

### Step 8 - Fix Issues

Apply simple release, docs, packaging, and tooling fixes directly. Rerun the narrow check that failed, then rerun the full required release check set before final readiness.

For complex production behavior changes, either fix with tests or stop before publishing and report the blocker. Do not bury production behavior changes inside release cleanup.

Docs are part of the release. User-facing behavior, installer behavior, integration paths, deployment target differences, public/private boundaries, and security-sensitive behavior must be documented in the same release lane.

### Step 9 - Package Artifacts

Use project-native packaging commands. If no command exists, inspect CI/release scripts before creating a new one.

For each artifact:

- Build from a clean, known source tree.
- Verify filenames and embedded versions.
- Inspect contents for private paths and secrets.
- Verify checksums, signatures, SBOM, provenance, and registry metadata when the project supports them.
- Verify download routes or published asset URLs when applicable.

### Step 10 - Deploy

Only deploy for an approved release.

Before production deploy, verify required env variable names exist without printing values. Confirm changed code fails closed if config is absent.

Deploy only changed targets required by the release. After deploy, inspect logs and health checks. Record exact blockers for unavailable services and run the closest meaningful smoke test.

### Step 11 - Sync Public Mirror Or Public Outputs

Only sync/push public outputs for an approved release.

Run dry-run first when available. Then sync and verify remote branch heads, included/excluded paths, and absence of private content.

```bash
git ls-remote "[public repo URL]" refs/heads/main refs/heads/stable
```

Adjust branch names for the project.

### Step 12 - Tag And Create Release Objects

Only for the explicitly approved release version:

```bash
RELEASE_TAG="[vX.Y.Z]"
git fetch --tags origin
! git rev-parse --verify "$RELEASE_TAG" >/dev/null 2>&1
! git ls-remote --exit-code --tags origin "refs/tags/$RELEASE_TAG" >/dev/null 2>&1
git tag -a "$RELEASE_TAG" -m "[Project Name] $RELEASE_TAG"
git push origin "$RELEASE_TAG"
```

Create or verify release objects on required hosts. A git tag alone is not enough when the project expects GitHub/GitLab release pages, package registry release entries, or container/image workflows.

Verify:

- Release object title, notes, draft/prerelease/latest status, target commit, and URL.
- Remote tag and dereferenced annotated tag.
- Public mirror tag and target, if applicable.
- Package/image publish workflows, signatures, SBOM/provenance, and registry availability.

Do not overwrite existing tags without explicit destructive approval.

### Step 13 - Final Report

Final report must include:

- Issues found and fixed, including boundary/security/artifact issues.
- Current version and release tag.
- Changed files or commit range used for release notes.
- Docs, changelog, installer, package, and artifact status.
- Test/typecheck/lint/build results and skipped checks with exact reasons.
- Deployment/log status and environment readiness result, if applicable.
- Private and public branch commit hashes, if applicable.
- Release tag name and private/public/package/image targets.
- Release URLs and registry/artifact URLs.
- Post-release workflow status, including pending/failing workflows.
- Warnings worth watching and remaining risks.

Do not call the release complete while blockers remain.
