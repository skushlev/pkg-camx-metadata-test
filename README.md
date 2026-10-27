# pkg-template

This repository is the reference starter for Debian package repositories
(`pkg-*`) in the Qualcomm Linux ecosystem. It provides the branch layout,
`debian/` skeletons, and thin GitHub workflow callers that plug a new
package into the shared package CI hosted in
[qli-ci](https://github.com/qualcomm-linux/qli-ci).

The workflows in this repository contain no build logic of their own: they
call reusable workflows from `qli-ci`, which build Debian suites through
Debusine and Ubuntu codenames through the `pkg-builder` container.

## Quick Start

To create a new package repository from this template:

1. On this repository's GitHub page, click **"Use this template"**.
2. Select **qualcomm-linux** as the owner. This is required.
3. Name the repository `pkg-<source-package-name>`, where
   `<source-package-name>` is the Debian source package name. This is
   required.
4. Enable **"Include all branches"**. Otherwise only the default `qli-ci`
   branch is copied and the packaging branch skeletons are lost.
5. Get compliance approval and have the `is-pkg-repo` custom property set to
   `true` (see [The `is-pkg-repo` property](#the-is-pkg-repo-property)).
6. Build your real packaging branches (see
   [Setting up the packaging branches](#setting-up-the-packaging-branches)).
7. Configure the repository for QLI CI (see
   [Repository configuration](#repository-configuration)).

## Branches

Branch names follow [DEP-14](https://dep-team.pages.debian.net/deps/dep14/).
The workflows derive the target distribution from the last two
`/`-separated fields of the branch name, `<family>/<suite>`. For example,
`qli/debian/latest` builds for Debian `sid` and `qcom/ubuntu/resolute`
builds for Ubuntu `resolute`.

- **qli-ci**: The default branch. It holds the workflow callers in
  `.github/workflows/` and the boilerplate documents (license, contributing
  guidelines, this README). It contains no upstream code and no `debian/`
  directory.
- **qli/debian/latest**: A starter Debian packaging branch. Debian builds
  and releases run through Debusine.
- **qcom/ubuntu/resolute**: A starter Ubuntu packaging branch. Ubuntu builds
  and releases run in the `pkg-builder` container.

Both packaging branches are orphan starters with placeholder `debian/`
content (`template1`, `template-dev`, `<upstream>`). They are **not** meant
to be used as-is; keep only the families your package needs and rebuild
them as described below.

## Setting up the packaging branches

The packaging branches shipped with this template have no history in common
with any real upstream code. For a real package, construct each packaging
branch you need (for example `qli/debian/latest`) as follows:

1. Clone the upstream source repository.
2. Branch off the upstream release tag being packaged.
3. Copy the `debian/` directory from the matching branch of this template
   as one or more commits, and customize it for your package (`control`,
   `changelog`, `copyright`, `gbp.conf`, `watch`, `upstream/metadata`, ...).
   In particular, `debian/gbp.conf` `debian-branch` must name the branch
   you are creating, and `debian/control` `Vcs-Git`/`Vcs-Browser` must point
   to your `pkg-*` repository.
4. Push the result to your `pkg-*` repository under the same branch name.

This roots the packaging history in the upstream code being packaged rather
than in an unrelated orphan. Packaging is expected to use source format
`3.0 (quilt)`, build with `gbp buildpackage` without extra arguments, and
contain no binaries or vendored code.

Do not hand-edit the workflow files on packaging branches; they are managed
centrally (see [Workflows](#workflows)).

## Validating your packaging locally

Before pushing to a packaging branch, build the package and run
[Lintian](https://lintian.debian.org/) against the resulting `.changes` to
catch policy violations and common packaging mistakes before CI does:

```sh
gbp buildpackage   # or: dpkg-buildpackage -us -uc -b
lintian -EviL +pedantic ../*.changes
```

The flags display experimental tags, info-level extended descriptions, and
the strictest "pedantic" level. Address findings (or add justified entries
to a `lintian-overrides` file) before opening a PR.

## Workflows

All workflow files in this repository are managed from `qli-ci`. Do not
edit them by hand in a `pkg-*` repository: change them in `qli-ci` and let
the change propagate. Each caller pins `@main` and passes
`qli-ci-ref: main`.

### Package workflows

These callers come from
[`qli-ci/pkg-workflows/`](https://github.com/qualcomm-linux/qli-ci/tree/main/pkg-workflows)
and are synced by qli-ci's **Workflows Sync** workflow.

On the default `qli-ci` branch:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `pkg-build.yml` | manual | Build (and test) the package at a given `debian-ref` branch or tag. |
| `pkg-promote.yml` | manual | Promote a packaging branch to a new upstream release tag, opening a PR. |
| `pkg-promote-prebuilt.yml` | manual | Promote a prebuilt (Artifactory tarball) package to a new upstream tag. |
| `pkg-release.yml` | manual | Release the package from a packaging branch, gated by the `Production` / `Ubuntu Production` approval environments. |
| `pkg-pr-hook.yml` | pull request | PR build hook (see below). |
| `stale-issues.yaml` | schedule | Mark and close stale issues and pull requests. |

On every packaging branch:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `pkg-pr-hook.yml` | pull request | On PRs against `qcom/**` and `debian/*` branches, build the PR head before merge and the base branch after merge. Checks appear as `PR Build / ...`. |

### Debusine workflows

Once the repository is enabled for Debusine (see
[Repository configuration](#repository-configuration)), qli-ci's
`update-workflow-files` tool adds the Debusine workflows
(`debusine-*.yml`, `README.debusine.md`) to the default branch and to the
managed Debian packaging branches (`qli/debian/*`, `qli-staging/debian/*`,
and transitionally `qcom/debian/*`). On those branches, PR validation is
reported as the `Debusine CI` check, and pushes to the branch drive
Debusine releases.

## Upstream repository PR check

To build the package against every PR opened in the upstream source
repository, add a caller workflow to the **default branch** of the upstream
repository (GitHub only runs `pull_request_target` workflows from the
default branch). It calls
[`pkg-upstream-pr-build-reusable-workflow.yml`](https://github.com/qualcomm-linux/qli-ci/blob/main/.github/workflows/pkg-upstream-pr-build-reusable-workflow.yml):

```yaml
name: Package Build PR Check

on:
  pull_request_target:
    branches: [ main ]
    types: [ opened, ready_for_review, synchronize ]
    paths-ignore:
      - '.github/**'

permissions:
  contents: read
  packages: read

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  package-build-pr-check:
    uses: qualcomm-linux/qli-ci/.github/workflows/pkg-upstream-pr-build-reusable-workflow.yml@main
    with:
      qli-ci-ref: main
      upstream-repo: ${{ github.event.pull_request.head.repo.full_name }}
      upstream-repo-ref: ${{ github.event.pull_request.head.ref }}
      pkg-repo: ${{ vars.PKG_REPO_GITHUB_NAME }}
      pr-number: ${{ github.event.pull_request.number }}
      suite: noble
    secrets:
      DEB_PKG_BOT_CI_TOKEN: ${{ secrets.DEB_PKG_BOT_CI_TOKEN }}
```

Set `suite` to the distribution codename to build for, and set the
`PKG_REPO_GITHUB_NAME` variable in the upstream repository (see
[Repository variables](#repository-variables)).

## Repository configuration

### Repository management tools

Repository settings for QLI CI are applied with the tools in
[`qli-ci/tools/repo-management/`](https://github.com/qualcomm-linux/qli-ci/tree/main/tools/repo-management),
and their required end state is defined in its
[`SPECIFICATION.md`](https://github.com/qualcomm-linux/qli-ci/blob/main/tools/repo-management/SPECIFICATION.md).
Ask a qli-ci maintainer to run them for your repository:

```sh
./enable-repo pkg-<name>
```

`enable-repo` runs, in order:

- `configure-repo`: default branch, Actions variables (`DEBUSINE_HOST`,
  `DEBUSINE_SCOPE`, `AXIOM_*`), the `Production`, `Ubuntu Production`,
  `Staging` and `Axiom` environments with their required reviewers, and
  visibility of the organization secrets `DEB_PKG_BOT_CI_TOKEN` and
  `DEB_PKG_BOT_CI_QSC_TOKEN`.
- `set-repo-secrets`: `DEBUSINE_USER`, `DEBUSINE_TOKEN`, and the
  per-environment `DEBUSINE_RELEASE_TOKEN`.
- `update-workflow-files`: the Debusine workflow files.

The tools refuse to act on a repository that is not on qli-ci's active-repo
list, or whose `is-pkg-repo` property is not `true`.

### The `is-pkg-repo` property

The `is-pkg-repo` custom property records that compliance approval has been
granted for enabling QLI CI on the repository. It is also how `pkg-*`
repositories are discovered for automation. The repository management tools
will not configure a repository until it is set to `true`, and `--force`
does not bypass this.

### Repository variables

Set these manually; they link the package repository and its upstream:

- **UPSTREAM_REPO_GITHUB_NAME** (in this `pkg-*` repository): the upstream
  repository, used by `pkg-promote.yml`. For example
  `qualcomm-linux/qcom-example-package-source` for `pkg-example`.
- **PKG_REPO_GITHUB_NAME** (in the upstream repository): the package
  repository, used by the upstream PR check. For example
  `qualcomm-linux/pkg-example`.

Optional:

- **IS_TEST_PKG**: set to `true` to run `pkg-release.yml` in test mode,
  releasing into the `qli-staging` Debusine workspace instead of `prod`.

### Runners

Most jobs run on GitHub-hosted arm64 runners. Some jobs (S3 uploads for
AXIOM testing, Ubuntu releases to Artifactory) run on the AWS runners. Ask
**Steve Manley** to enable the repository for the AWS runners.

### GHCR registry access

Ubuntu builds use the `pkg-builder` container image hosted in the
qualcomm-linux GitHub Container Registry. The repository must be granted
`packages:read` access to it. Ask **Mark Matyas** (mmatyas@qti.qualcomm.com)
to add the repository at:

https://github.com/orgs/qualcomm-linux/packages/container/pkg-builder/settings

### Branch protection

Configure a branch ruleset covering the packaging branches (`debian/**`,
`ubuntu/**`, `qcom/**`, `qli/**`):

- Restrict deletions.
- Block force pushes.
- Require a pull request before merging.
- Require the PR checks to pass: `PR Build` on `qcom/**` and `debian/*`
  branches, `Debusine CI` on Debusine-managed branches.
- Allow the `qcom-service-bot` account to bypass the ruleset, so the
  automation can push promotion and release commits.

### Additional settings

- Enable **"Automatically delete head branches"**.
- Allow only merge commits for pull request merges.
- Enable **release immutability** in the upstream repository.
- Give the **qcom-service-bot** GitHub account the **Write** role. While the
  repository is private, add it directly in the repository settings. Once it
  is public, access is managed through mailing lists instead (see
  [Repository roles after going public](#repository-roles-after-going-public)),
  and the bot must be added by its Qualcomm username
  **githubservice**@qti.qualcomm.com rather than by its GitHub handle.

## Making the repository public

Package repositories start private and must be made public once mature.

### Legal approval

The first step is the trademark/legal process. Your point of contact is
[Stephanie Arce](mailto:sarce@qti.qualcomm.com).

[Complete an OSS Contribution Request](https://jira-dc4.qualcomm.com/jira/secure/CreateIssue.jspa?pid=46536&issuetype=13440).
Guidance for the mandatory fields:

- Components: select "Other".
- Summary: a short description such as "Debian packaging repository for X".
- Project name: the repository name, such as "pkg-example".
- Destination URL: the GitHub URL, such as
  "https://github.com/qualcomm-linux/pkg-example".
- Description: explain that this is the Debian packaging repository for
  upstream project X.
- Project Licenses: BSD-3-Clause, unless this changes.
- Contribution plan:
  - Project Description: same as above.
  - General plan with the project: Debian packaging for project X.
  - Leadership positions in the project: yourself or your manager.
  - Is Qualcomm expanding the project by contributing technology
    implementations: likely no.

More information is in the
[new project checklist](https://github.qualcomm.com/pages/osdo/handbook/qcom-github/docs/new-project-checklist/).
Legal approval usually takes a few days to a week.

### Enabling the public repository

Once Legal approves, submit the
[enable-repo form](https://ossops.qualcomm.com/github/enable-repo/). Your
point of contact is [Mark Matyas](mailto:mmatyas@qti.qualcomm.com).

### Repository roles after going public

While private, the repository's creator is its maintainer and can manage
access in the repository settings. Once public, contributors and maintainers
are managed through Qualcomm mailing lists instead, and the existing
contributor list is cleared.

To request a role, go to https://lists.qualcomm.com/ListManager and search
for the list matching the repository. For `pkg-example`, those are:

- `contributors.pkg-example`
- `maintainers.pkg-example`

The new role takes about an hour to appear in GitHub after the list request
is accepted.

## Getting in Contact

For support or inquiries, contact sbeaudoi@qti.qualcomm.com.

## License

pkg-template is licensed under the [BSD-3-Clause License](https://spdx.org/licenses/BSD-3-Clause.html). See [LICENSE.txt](LICENSE.txt) for the full license text.
