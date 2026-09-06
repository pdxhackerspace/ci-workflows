# ci-workflows

Shared GitHub Actions workflows and branch-protection rulesets for pdxhackerspace
projects. One definition here, a short stub in each repo, so CI and release
behaviour cannot drift between projects.

## Principles

**The newest `v*` git tag is the version.** No repository using these workflows should
contain a `VERSION` file, and no commit should ever hand-edit a version number. A release
is a button, not a branch.

**Releases are triggered, not merged.** Merging to the release branch makes code eligible
to ship; a separate manual run decides when and at what version.

**The workflow does not change between solo and team projects.** Everything is a short-lived
branch, a pull request, and passing checks. The only thing that differs is
`required_approving_review_count` in the ruleset — `0` when you work alone, `1` when you
don't. Nobody has to learn a second process when a project gains a contributor.

## Workflows

| Workflow | Purpose |
|---|---|
| `rails-ci.yml` | Lint and test a Rails project, with optional Postgres and Redis services |
| `release.yml` | Compute the next semver, build with it baked in, publish, tag the commit |
| `build-image.yml` | Build one immutable candidate per commit; optionally move a floating tag |
| `promote-image.yml` | Re-tag an existing candidate as a release without rebuilding the app |

There are two release models. Use `release.yml` on its own if you build production at
release time. Use `build-image.yml` plus `promote-image.yml` if you want the bits that
reach production to be the same bits you tested.

### rails-ci.yml

```yaml
name: ci
on:
  pull_request:
    branches: [main]
jobs:
  ci:
    uses: pdxhackerspace/ci-workflows/.github/workflows/rails-ci.yml@v1
    with:
      ruby-version: '4.0.6'
      node-version: '24'
      postgres: true
      redis: true
      database-name: my_app_test
      apt-packages: libvips
      js-test-command: yarn test:js
      extra-env: |
        SOME_API_TOKEN=ci-test-placeholder
```

### release.yml

```yaml
name: release
on:
  workflow_dispatch:
    inputs:
      bump:
        type: choice
        options: [patch, minor, major]
        required: true
      sha:
        type: string
        required: false
      dry_run:
        type: boolean
        default: false
jobs:
  release:
    uses: pdxhackerspace/ci-workflows/.github/workflows/release.yml@v1
    with:
      bump: ${{ inputs.bump }}
      sha: ${{ inputs.sha }}
      dry_run: ${{ inputs.dry_run }}
      image-name: ${{ github.repository_owner }}/my-app
```

Then:

```bash
gh workflow run release.yml -f bump=minor
gh workflow run release.yml -f bump=minor -f dry_run=true   # preview only
```

## Versions the app can report

`APP_VERSION` is baked at build time and is always comparable by eye:

| Build | `APP_VERSION` |
|---|---|
| Release | `0.51.0+abc1234` |
| Candidate / staging | `0.50.1+109.abc1234.staging` |
| Local checkout | `git describe` output, else `dev` |

The suffix after `+` is [semver build metadata](https://semver.org/#spec-item-10), so the
leading `MAJOR.MINOR.PATCH` is what humans read and compare while the commit stays
recoverable. Surface the short form in your UI and keep the full string in a tooltip.

Candidate versions anchor to the newest release tag in the repository rather than to
`git describe`. `describe` walks ancestry, so on a branch that has diverged from the
release branch it anchors to a much older tag and makes the build look *older* than
production.

## Rulesets

`rulesets/main-solo.json` and `rulesets/main-team.json` are the same policy at two review
counts: pull request required, `ci` must pass, no force-push, no deletion.

```bash
bin/apply-ruleset main-solo --dry-run pdxhackerspace/member-zone
bin/apply-ruleset main-solo pdxhackerspace/member-zone
bin/apply-ruleset main-team --all pdxhackerspace
```

Re-running is safe: a ruleset with a matching name is updated in place rather than
duplicated. Both files keep an admin bypass so a genuine emergency has a documented,
audited path that is not "push to main".

## Pinning

Reference workflows by the moving major tag (`@v1`) so every consumer can be fixed by
moving one tag. Pin to a commit SHA where supply-chain rigor matters more than
convenience.

Callers must pass `secrets: inherit` if the reusable workflow needs any secret beyond
`GITHUB_TOKEN`; reusable workflows do not inherit secrets by default.
