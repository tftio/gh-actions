# gh-actions

Shared CI and release workflows for the tftio repositories on GitHub.

This repository is the GitHub Actions counterpart of `tftio/ci-components`, which
served the same purpose on a self-hosted GitLab instance. It exists because the
migration of the collaborative repositories to GitHub could not carry the GitLab
component library with it: those components run jobs inside a CI image published to
a private registry that is reachable only from the network hosting it, and not from
a GitHub-hosted runner.

The full reasoning, including what was rejected and why, is recorded in the private
migration plan that produced this repository.

## Contract

A calling repository declares its own toolchain in its own `mise.toml` and its own
checks in its own mise tasks. These workflows install that toolchain and run that
task. They decide nothing about what a gate means, which is what makes them safe to
share across eighteen repositories that do genuinely different things.

```yaml
jobs:
  ci:
    uses: tftio/gh-actions/.github/workflows/rust-ci.yml@v1.1.0
```

Copy the appropriate file from `templates/` into a repository's
`.github/workflows/ci.yml` and pin the tag.

### Versioning

Consumers pin a full, immutable `vX.Y.Z` tag. There is deliberately no moving `v1`
alias, despite the Actions convention of publishing one: a moving major tag is a
mutable reference, which is the property this fleet has already been bitten by --
four different component versions were pinned across the GitLab instance at
migration time precisely because nobody could see what a pin meant. An immutable
pin makes an upgrade a visible commit in the consuming repository.

## Why this repository is public

Making it public removes GitHub's access rules for calling reusable workflows from
private repositories, which would otherwise have to be configured and maintained
for every consumer while the consumers are private. It holds no credentials and no
secret values, and it must not acquire any: a secret belongs in the calling
repository's own settings, where it is scoped to that repository.

## Why there is no container image

An image can pre-bake the toolchain and the prebuilt cargo helper binaries. It
cannot pre-bake `target/` or a resolved `~/.cargo` for eighteen distinct dependency
graphs, and that is where Rust CI actually spends its time. The image was therefore
dropped rather than republished to GHCR. Dropping it also removes the failure class
that broke `tftio/planner`, where the image's `mise` wrote a different `mise.lock`
than the one the repository had committed and cocogitto then refused to release
against a dirty tree.

The decision is measurable rather than settled by argument, and the migration plan
carries a task to measure it on the pilot repository. If toolchain installation
turns out to dominate cold runs, the question reopens with data.

## Why third-party actions are pinned by SHA

A tag is mutable. An action referenced by tag and later retagged would execute with
these workflows' permissions across every consuming repository at once. Each pin
carries a comment naming the version it corresponds to, so the pin can be audited
without resolving the SHA by hand.

## Rust releases with release-plz

The scratch proof on 2026-09-08 created a release PR and passed CI, but the
post-merge release attempted cargo publication and failed for lack of a token.
No new tag or GitHub Release was created. The pinned CLI requires explicit
`publish = false`; `git_only = true` alone does not disable publication.
The corrected template and guard require explicit publication disabling. Recovery
PR #2 passed CI and its merge produced v0.1.1 and a GitHub Release without cargo
publication. The caller template targets the immutable `v2.2.0` shared tag.

The workflow uses the existing GitHub Actions checkout and GitHub App scaffolding.
It does not introduce an SSH deploy key. Agent-run Git commands continue to use SSH;
hosted checkout retains the established Actions-token authentication.

The additive `release-plz.yml` reusable workflow manages Rust version bumps through
release pull requests, then creates tags and GitHub Releases when those PRs merge.
It uses the existing release bot: the caller passes the `RELEASE_BOT_APP_ID`
repository variable as the `app-id` input and passes `RELEASE_BOT_PRIVATE_KEY`
as a secret. Its PR commits use release-plz's GitHub GraphQL path,
which creates verified commits; version commits do not go to the default branch.

Copy `templates/rust-release-plz.yml` into `.github/workflows/release-plz.yml` and
`templates/release-plz.toml` beside the root `Cargo.toml`. The caller pins `v2.2.0`;
use it only once that tag is published. Do not point it at an older release that
does not contain this workflow. Remove
the old release-please caller, config and manifest in the same consumer change,
so only one mechanism watches the default branch.

The configuration deliberately sets `git_only = true`: release-plz reads version
anchors from matching git tags, but does not by itself skip cargo publication in
the pinned CLI. The workflow requires workspace `publish = false` and rejects
package overrides that enable publication or disable git-only mode.
It accepts no registry credential. A tag must point at the
commit whose manifest contains the same version, not merely carry the right name.
Single-crate repositories use the default `v{{ version }}` tag pattern.

`release_always = false` is also required: the upstream default can release on an
ordinary push, whereas this workflow requires a merged release PR. Keep release
creation and PR updates in separate jobs. The PR job waits for successful release
creation before checking out tags, preventing a stale baseline from generating a
duplicate PR for the version just released. Only the PR job uses concurrency;
workflow-level concurrency could cancel the pending run for a release merge.

The workflow pins release-plz/action v0.5.133 by its resolved commit SHA and the
CLI to 0.3.162. It installs the consumer's declared mise toolchain. This phase
does not enable registry publication anywhere; any existing artifact caller must
also keep publication disabled. Both callers must be reviewed before publication
is enabled, to avoid two publishers for the same version.

The older shared release workflows and templates remain until their last
consumers migrate; their retirement is a separate change. Local linting is not
evidence of the live cycle: the throwaway proof must demonstrate PR creation, PR
CI, merge, a componentless tag and a GitHub Release without registry publication.

References: [configuration](https://release-plz.dev/docs/config),
[GitHub concurrency guidance](https://release-plz.dev/docs/github/quickstart#concurrency),
and [pinned action source](https://github.com/release-plz/action/blob/aec534bbd8631793b9b3b8f1ee6cd886c322e17f/action.yml).

## Existing release-please path (pending consumer migration)

Releases run through **release-please**, which opens a pull request containing the version
bump and changelog. Merging that PR cuts the tag and the GitHub release. A separate
workflow, triggered by `release: published`, builds and publishes what the tag names.

That split is the point: nothing in the release workflow builds an artifact, and nothing
in the artifact workflow decides a version. A hand-edited version cannot reach a published
binary, and a failed build cannot un-cut a release that already exists.

This replaced a cocogitto release-on-merge design on 2026-09-07. cocogitto pushes a freshly
created version commit straight to the default branch, and a rule requiring a status check
rejects it — the commit has never been through CI because it did not exist a moment
earlier. The documented exemption is to name GitHub Actions as a ruleset bypass actor, and
that is unavailable outside an organization:

```
Actor GitHub Actions integration must be part of the ruleset source or owner organization
```

So on a personal account, required checks and cocogitto releases could not coexist.
release-please dissolves the conflict rather than exempting anything: the bump arrives as a
pull request and passes CI on its own merits.

### The release bot is a GitHub App, and this is not optional

A pull request opened with the default `GITHUB_TOKEN` does not trigger `pull_request`
workflows — GitHub's guard against recursive automation. A release PR created that way
would carry no CI check at all and could never satisfy a required one, which would defeat
the entire change. The app token exists to make the release PR *checkable*, not to grant
privilege.

It is also a better shape than the alternative that was rejected: a personal access token
with write access to the default branch, stored in every repository. One app, installed
where it is needed, revocable, not tied to a person's account.

Each consuming repository needs `RELEASE_BOT_APP_ID` and `RELEASE_BOT_PRIVATE_KEY`.

### Per-repository configuration

`release-please-config.json` and `.release-please-manifest.json` live in the consuming
repository. **Seed the manifest with the current version** — an absent or empty manifest
restarts versioning at `0.0.0`.

Rust uses `release-type: "rust"`, which updates `Cargo.toml`, `Cargo.lock` and
`CHANGELOG.md`. Verified against the strategy source rather than assumed: it carries a
`CargoLock` updater applied whenever the file exists.

Python uses `release-type: "simple"` with an `extra-files` TOML updater on
`$.project.version`, because the built-in `python` strategy documents `setup.py` and
`setup.cfg` and does not state whether it writes a PEP 621 version. Python repositories
**must** also run the relock workflow: release-please does not regenerate `uv.lock`, and
the lock records the package's own version, so it lags the release. That failure is
invisible where it is introduced — the release PR's own CI passes, because the lock still
matches the version the branch was cut from, and the gate fails after the merge.

Only a musl binary is produced, and only for Rust. Darwin artifacts were dropped rather
than moved to GitHub-hosted macOS runners, which bill at ten times the Linux rate.

## Lockfiles must cover the CI platform

`mise-action` runs `mise install --locked` whenever a repository commits a
`mise.lock`, so the lockfile must carry an entry for the runner's platform —
`linux-x64` for every gate here. A lockfile generated only on a developer's macOS
machine holds `macos-arm64` entries alone and fails the install step with
`No lockfile URL found for <tool> on platform linux-x64`.

The fix belongs in the consuming repository, not in these workflows:

```bash
mise lock --platform linux-x64,macos-arm64
```

Locked mode is deliberately not disabled to paper over this. A lockfile that does
not cover the platform CI runs on is not pinning CI to anything, and a green run
that silently resolved its own versions would be worth less than a red one that
says so.

## Runners

GitHub-hosted only. A self-hosted runner executing a fork's pull request would run
untrusted code on privately operated hardware, which is the exposure that ruled
self-hosted runners out for these repositories.
