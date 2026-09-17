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

## Private git dependencies over SSH

Some consumers depend on another private tftio repository fetched over SSH: a cargo
git dependency, or a mise tool built from such a repository. A GitHub-hosted runner
has no SSH key, and GitHub refuses a keyless SSH fetch even of a public repository.

Both CI workflows and `release-plz.yml` accept one optional secret,
`ssh-private-key`. release-plz runs cargo against the repository, so a release fails
on the same unreachable dependency CI does; `tftio/asana-cli` could not open a release
PR until it passed the key there too (run 35028608827). The calling repository stores a
read-only deploy key for the dependency in its own secrets and passes it through:

```yaml
jobs:
  ci:
    uses: tftio/gh-actions/.github/workflows/rust-ci.yml@vX.Y.Z
    secrets:
      ssh-private-key: ${{ secrets.PLANNER_DEPLOY_KEY }}
```

The workflow writes the key before the toolchain install, pins GitHub's ed25519 host
key rather than trusting `ssh-keyscan`, makes cargo fetch git dependencies through
the git CLI so the key is used, and fails the job if the key does not authenticate.
Without the secret the step does nothing.

This keeps the rule above intact: the key lives in the consuming repository's
settings, scoped there, and this repository still holds no secret values. A deploy
key is read-only and grants access to exactly one repository, so one key for a
dependency can serve every repository that consumes it.

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
No new tag or GitHub Release was created. The pinned CLI requires an explicit,
consistent release policy: `git_only = true, publish = false` for git-only
releases, or `git_only = false, publish = true` for registry publication.
Recovery PR #2 passed CI and its merge produced v0.1.1 and a GitHub Release
without cargo publication.

The workflow uses the existing GitHub Actions checkout and GitHub App scaffolding.
It needs no SSH deploy key of its own; a caller with a private git dependency passes
one as the optional `ssh-private-key` secret described above. Agent-run Git commands
continue to use SSH; hosted checkout retains the established Actions-token
authentication.

The additive `release-plz.yml` reusable workflow manages Rust version bumps through
release pull requests, then creates tags and GitHub Releases when those PRs merge.
It uses the existing release bot: the caller passes the `RELEASE_BOT_APP_ID`
repository variable as the `app-id` input and passes `RELEASE_BOT_PRIVATE_KEY`
as a secret. Its PR commits use release-plz's GitHub GraphQL path,
which creates verified commits; version commits do not go to the default branch.

Copy `templates/rust-release-plz.yml` into `.github/workflows/release-plz.yml` and
`templates/release-plz.toml` beside the root `Cargo.toml`. Replace `vX.Y.Z` with
the published immutable tag that contains the desired workflow; never use a branch
or moving major alias. Remove
the old release-please caller, config and manifest in the same consumer change,
so only one mechanism watches the default branch.

The template deliberately sets `git_only = true, publish = false`, which keeps a
repository git-only: release-plz reads version anchors from matching git tags and
skips `cargo publish`. A caller deliberately opts every package into publication
by changing both workspace values to `git_only = false, publish = true`. A
workspace can opt in only named packages by keeping the workspace pair and adding
both values as `false` and `true`, respectively, to a `[[package]]` entry. The
workflow rejects missing, partial, or inconsistent pairs, so an accidental
`publish = true` does not run. An opted-in crate must be configured for crates.io
trusted publishing for its repository, workflow, and environment before it can
publish. The release job has `id-token: write`; release-plz exchanges its OIDC
identity for a short-lived token as needed, so the workflow accepts no registry
credential. A tag must point at the
commit whose manifest contains the same version, not merely carry the right name.
Single-crate repositories use the default `v{{ version }}` tag pattern.

Every caller repository is private, so each git operation release-plz runs needs a
credential. `actions/checkout` persists its token behind an `includeIf` keyed to the
workspace's `.git` path, and release-plz updates an open release PR from a copy of
the repository under `/tmp` that the `includeIf` does not match. On 2026-09-15 that
fetch failed in `tftio/planner` ("could not read Username"); release-plz closed the
open release PR, opened a replacement with unresolved stash conflict markers in
`CHANGELOG.md`, `Cargo.lock` and `Cargo.toml`, and reported success. Checkout now
persists no credential, and a step exports the app token to later steps as git
environment configuration (`GIT_CONFIG_COUNT`/`KEY`/`VALUE`), which applies in every
repository copy. After `release-pr`, the workflow fails if any file the release PR
adds or modifies contains a conflict marker.

`release_always = false` is also required: the upstream default can release on an
ordinary push, whereas this workflow requires a merged release PR. Keep release
creation and PR updates in separate jobs. The PR job waits for successful release
creation before checking out tags, preventing a stale baseline from generating a
duplicate PR for the version just released. Only the PR job uses concurrency;
workflow-level concurrency could cancel the pending run for a release merge.

The workflow pins release-plz/action v0.5.133 by its resolved commit SHA and the
CLI to 0.3.162. It installs the consumer's declared mise toolchain. Publication
is enabled only by a reviewed caller configuration; git-only callers retain the
template's policy. The workflow uses trusted publishing, not a
`CARGO_REGISTRY_TOKEN` secret, so a crate not configured at crates.io fails closed
rather than falling back to a long-lived token.

The older shared release workflows and templates remain until their last
consumers migrate; their retirement is a separate change. Local linting is not
evidence of the live cycle: the throwaway proof must demonstrate PR creation, PR
CI, merge, a componentless tag and a GitHub Release without registry publication.

References: [configuration](https://release-plz.dev/docs/config),
[GitHub concurrency guidance](https://release-plz.dev/docs/github/quickstart#concurrency),
[trusted publishing](https://release-plz.dev/docs/github/quickstart#trusted-publishing),
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

## Shared Renovate preset

`default.json` is a Renovate config preset for the fleet, covering the four
inputs it actually uses: cargo manifests and lockfiles, `mise.toml` tool pins,
PEP 621 (uv) Python dependencies, and GitHub Actions references -- including
this repository's own reusable-workflow tags. A consuming repository extends
it with a one-line stub instead of restating policy:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>tftio/gh-actions"]
}
```

`renovate.json` at the repository root is this repository's own self-config,
and it extends the same preset (`github>tftio/gh-actions`, which resolves to
`default.json` when no preset name is given). It exists as a separate file
from the preset on purpose: Renovate's `github>owner/repo` reference resolves
to `default.json` unless a preset name follows a colon, so a consumer writing
`github>tftio/gh-actions` reaches the preset while this repository's own
Renovate run reads `renovate.json` for its self-config, and the two files
never collide. `github>tftio/gh-actions:renovate` would instead resolve to
`renovate.json`, which is reserved for self-config for that reason -- the
preset is never published under that name.

### Policy

- **Grouping.** Cargo, PEP 621 and mise minor/patch updates are each grouped
  into one weekly pull request per consuming repository; major updates of each
  are left ungrouped for individual review. The `tftio/gh-actions` reusable
  workflow pin is its own group, updated with `rangeStrategy: pin` so a bump
  always proposes a new immutable `vX.Y.Z` tag -- never a range and never a
  moving alias. All other GitHub Actions step references (already pinned by
  resolved commit SHA with a trailing version comment) are grouped separately.
  A `lockfile maintenance` group refreshes lockfiles with no other change.
- **Schedule.** Weekly, before 06:00 on Monday, for every group; vulnerability
  alerts run on their own schedule (`at any time`).
- **Vulnerability alerts.** Enabled, both Renovate's own `vulnerabilityAlerts`
  and `osvVulnerabilityAlerts`, labelled `renovate` and `security`.
- **Commit convention.** Semantic commits are enabled and default to
  `build(deps): ...`, matching the fleet's conventional-commit convention for
  dependency bumps; GitHub Actions groups (both the shared-workflow pin and
  step pins) commit as `ci: ...` instead, since those changes touch workflow
  definitions rather than a dependency manifest.
- **Automerge is explicitly `false`**, at the top level and again on the
  vulnerability-alerts block, and `platformAutomerge` is also `false`. This is
  a deliberate, recorded decision, not an oversight: the Renovate bot holds
  write access to every repository it runs against, and no repository gains
  automerge without a decision recorded in the consolidation plan that
  introduced this preset. Changing it is a policy decision, made here, once,
  for every consumer at the same time.

### Visibility and preset resolution

This repository is public (see "Why this repository is public" above), which
matters for how Renovate resolves a preset it references. The Mend Renovate
GitHub App can read a **private** preset only when the app is installed on
the repository that hosts it, in addition to being installed on the
consuming repository, and only within the same account or organization.
Because this repository is public, that restriction does not apply: any
repository with the Renovate app installed -- public or private -- can
resolve `github>tftio/gh-actions` without the app also being installed here,
the same way it can reference any other public GitHub repository's preset.
Installing the app here as well is still worthwhile so this repository gets
its own dependency updates (see `renovate.json` above), but it is not a
precondition for a consumer resolving the preset.
