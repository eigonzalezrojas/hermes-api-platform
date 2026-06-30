# ADR-003 — Keep `main` as the production branch; prohibit direct commits

## Status

Accepted

## Date

2024-01-01

## Context

The Hermes platform is a portfolio project designed to demonstrate enterprise-grade engineering practices. One of the most visible signals of those practices is the team's branching and integration workflow.

Two workflows were considered:

| Workflow | Description |
|---|---|
| **Trunk-based with direct commits** | All work committed directly to `main`; no feature branches; fast but uncontrolled |
| **Branch-based with PR gating** | Every change on a feature/fix/docs/chore/ci branch; merged to `main` only after CI passes and review is complete |

The relevant constraints:

- GitHub Actions CI is configured to trigger on PRs to `main`. Without PRs, the CI pipeline never runs on changes before they reach `main`.
- The project uses Conventional Commits and Semantic Versioning — a clean commit history on `main` requires discipline that PRs enforce.
- As a portfolio project, the commit history and branch graph are visible to reviewers. A history of `fix: typo`, `wip`, `asdf` commits on `main` signals poor practice.
- The project is developed by a single engineer. The discipline of a PR-based workflow must be self-imposed.

## Decision

**`main` is the stable production branch. No direct commits to `main` are permitted.** Every change — including documentation, configuration, and CI pipeline updates — must go through a branch following the naming convention and be merged via a Pull Request after the CI pipeline passes.

Branch naming convention:

```
feature/<slug>    — new functionality
fix/<slug>        — bug fix
chore/<slug>      — maintenance, dependency updates, configuration
docs/<slug>       — documentation only
ci/<slug>         — CI/CD pipeline changes
```

## Rationale

1. **CI validation before merge**: the `ci.yml` GitHub Actions workflow runs on PRs to `main`. Direct commits to `main` bypass this validation entirely — a broken build could reach the production branch undetected. The PR gate ensures every merge to `main` has passed build, test, and vulnerability checks.

2. **Clean commit history**: PRs are squash-merged or merged with a conventional commit message. The `main` branch history reads as a log of features and fixes, not a sequence of in-progress commits. This is valuable for `git log`, changelog generation, and release tagging.

3. **Branch protection as a forcing function**: GitHub's branch protection rules (required status checks, required reviewer) enforce this decision mechanically — not just as a social norm. Once configured, it is impossible to push directly to `main` even accidentally.

4. **Portfolio signal**: a repository where every feature is developed on a named branch, every merge goes through a PR, and CI is green on `main` at all times demonstrates exactly the workflow a platform engineering team would require.

5. **Conventional Commits alignment**: Conventional Commits (`feat:`, `fix:`, `chore:`, etc.) are meaningful at the PR level. When commits are squash-merged, the PR title becomes the commit on `main` — making the convention auditable and enforceable.

## Consequences

### Positive

- `main` is always in a releasable state — every commit has passed CI.
- The branch graph in GitHub shows the feature development history clearly.
- Semantic versioning automation (e.g., `semantic-release`) can be applied to `main` without filtering noise commits.
- CI costs are predictable — the pipeline runs on PRs, not on every individual commit during development.

### Negative

- **Overhead for a solo developer**: opening a PR, waiting for CI, and merging adds friction compared to a direct commit. This is intentional — the practice must be internalized, not bypassed because it is inconvenient.
- **No "quick fix" shortcut**: even a one-line typo fix requires a branch and a PR. This can feel disproportionate. Mitigation: keep branch lifetimes short (open PR immediately after the first commit) and use draft PRs for work in progress.
- **Accidental direct commits must be reverted, not force-pushed**: if a direct commit reaches `main` before branch protection is configured, use `git revert <sha>` to undo it. `git push --force` on `main` rewrites shared history and must never be used — even on a solo project, it destroys the auditability that the workflow is designed to provide.

## Enforcement

Configure these GitHub branch protection settings on `main`:

- **Require status checks to pass**: `build-and-test` and `dependency-check` from `ci.yml`.
- **Require branches to be up to date before merging**: prevents stale PRs from bypassing failing tests introduced by concurrent merges.
- **Do not allow bypassing required status checks**: applies to repository administrators — no exceptions.
- **Restrict who can push to matching branches**: only allow merges via PR (no direct push from any user).

These settings are configured in the GitHub repository settings UI, not in code. They should be applied immediately when the repository is created.
