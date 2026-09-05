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

## Runners

GitHub-hosted only. A self-hosted runner executing a fork's pull request would run
untrusted code on privately operated hardware, which is the exposure that ruled
self-hosted runners out for these repositories.
