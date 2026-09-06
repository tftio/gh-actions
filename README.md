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
    uses: tftio/gh-actions/.github/workflows/rust-ci.yml@v1
```

Copy the appropriate file from `templates/` into a repository's
`.github/workflows/ci.yml` and pin the tag.

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

## Releases

`rust-release.yml` drives cocogitto against the contract a repository already
exposes through its own `cog.toml` and mise tasks — `release:version` as the
pre-bump hook, `build` reading `$BUILD_TARGET`, `release:publish` gated on
`PUBLISH_ENABLED`. Migrating a repository does not change how it releases.

It runs as a single job rather than the two stages the GitLab component used. That
decomposition cannot be carried over: a tag pushed with the default `GITHUB_TOKEN`
does not trigger further workflow runs, so a tag-triggered build job would never
fire. The alternative is a long-lived personal access token stored in every
repository purely to defeat that safeguard, which this design declines to hold.

The release policy is checked before any version is computed, so a repository that
declares `publish: true` without a `CARGO_REGISTRY_TOKEN` fails with nothing
written — no bump commit, no tag, nothing to unwind.

Only a musl binary is produced. Darwin artifacts were dropped rather than moved to
GitHub-hosted macOS runners, which bill at ten times the Linux rate.

### Branch protection

The release job pushes a version commit and a tag directly to the default branch,
so a rule requiring pull requests will block every release unless the workflow can
bypass it. Configure the ruleset to require the CI status check and add GitHub
Actions to the bypass list; do not simply drop the protection. This must be
verified on each repository as it is configured rather than assumed to work.

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
