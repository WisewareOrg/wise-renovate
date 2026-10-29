# wise-renovate

The self-hosted Renovate runner for the `WisewareOrg` organization's repositories: a scheduled GitHub
Actions workflow that runs the Renovate CLI as a GitHub App, plus the shared preset every onboarded
repository extends.

## Why self-hosted Renovate

- **It tracks bare version strings the hosted Dependabot integration cannot.** Dependabot resolves
  versions only inside manifests it understands; Renovate's built-in managers (`npm`, `nvm`,
  `github-actions`, `docker`) already cover `engines.node`, `.nvmrc`, `setup-node`'s
  `with: node-version`, and Docker base image tags, so one dependency graph moves together.
- **The bot identity is a GitHub App this org owns, not a third-party app with standing write
  access to every repository it's installed on.** The App is created and installed by the organization
  operating this runner, installed on all of its repositories, and its installation tokens expire
  after one hour
  ([docs.renovatebot.com/modules/platform/github/](https://docs.renovatebot.com/modules/platform/github/)).
- **`config.json` sets no `gitAuthor`.** Running as a GitHub App already attributes commits to the
  App's own bot identity once authenticated via its installation token
  ([docs.renovatebot.com/modules/platform/github/](https://docs.renovatebot.com/modules/platform/github/)),
  so an explicit `gitAuthor` override is unneeded, not an oversight.
- **The cutover is reversible.** The GitHub-hosted Mend Renovate app reads the identical
  `renovate.json` / preset configuration a consuming repository already carries, so returning to the
  hosted app is a platform change, not a configuration rewrite.

## Policy (the shared preset, `default.json`)

| Setting | Value |
|---|---|
| Automerge scope | `minor`, `patch`, `pin`, `pinDigest` (a tag gaining its digest), `digest` and `lockFileMaintenance` (a lockfile regenerated within the ranges its manifest already declares) updates automerge on green; `major` updates wait for review |
| Minimum release age | 3 days across ecosystems, except dependencies under `WisewareOrg/` (repos this organization authors) and the `wise-ci` group below, which propose immediately. Lock file maintenance's npm regeneration keeps this floor through npm's own `--before` resolution flag, not through the branch's stability status — if npm can't resolve with the date cut-off applied, Renovate retries without it and notes this on the PR; this preset adds no age cutoff for uv lock regeneration; native support is tracked in [renovatebot/renovate#41654](https://github.com/renovatebot/renovate/issues/41654) |
| PR creation | `immediate` — a PR opens as soon as Renovate finds an eligible update; the release-age check is enforced through `internalChecksFilter: strict` (Renovate's default), which filters out candidate versions that have not yet passed their minimum release age, so none is proposed |
| Dependency Dashboard | enabled |
| PR limits | none — `prHourlyLimit` and `prConcurrentLimit` are `0`, overriding Renovate's built-in 2 per hour and 10 open, so every eligible update opens its PR on the run that finds it |
| Lock file maintenance | enabled — every lockfile Renovate finds, wherever it sits, is regenerated on every scheduler run that lands on Monday (UTC), so a Monday can bring more than one lock-file-maintenance PR, and a transitive dependency pinned only in a lockfile still picks up published fixes |
| Digest pinning | GitHub Actions and Docker image references are pinned to digests (`helpers:pinGitHubActionDigests`, `docker:pinDigests`) |
| Grouping | `node` — the node-version and Docker datasources for the `node` package, plus the github-releases datasource for `actions/node-versions` (how the `github-actions` manager reports `setup-node`'s `with: node-version`), are grouped into one PR, so `engines.node`, `setup-node`, and a Dockerfile base image move together; `wise-ci` — every `WisewareOrg/wise-ci` dependency a consumer carries, whether a `uses: WisewareOrg/wise-ci/<check>@<sha>` action pin (tracked by a custom regex manager against the `git-refs` datasource) or a `tool.uv.sources` git dependency, is grouped into one PR. The built-in `github-actions` manager is disabled for `WisewareOrg/wise-ci` so it can never open a second, ungrouped update for the same pin |
| Base | [`config:recommended`](https://docs.renovatebot.com/presets-config/#configrecommended) |

Every project extending this preset inherits this policy; a project overrides a setting locally
only by adding its own `packageRules` on top.

## How a project opts in

Add a one-line `renovate.json` at the project's root, pinned to a preset release tag:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>WisewareOrg/wise-renovate#v1.4.0"]
}
```

`config.json`'s `autodiscoverFilter` already matches every `WisewareOrg` repository, and
`requireConfig: "required"` skips any repository with no `renovate.json`, so this file is the whole
enrolment — no change lands in this repository.

## App setup (one-time, by a repository admin)

1. Create a GitHub App owned by the `WisewareOrg` organization. Its name is `wiseware-renovate`.
2. Set these repository permissions exactly as Renovate's GitHub App documentation lists them
   ([docs.renovatebot.com/modules/platform/github/](https://docs.renovatebot.com/modules/platform/github/),
   "Running as a GitHub App"):

   | Permission | Access |
   |---|---|
   | Checks | Read and write |
   | Commit statuses | Read and write |
   | Contents | Read and write |
   | Issues | Read and write |
   | Pull requests | Read and write |
   | Workflows | Read and write |
   | Administration | Read |
   | Dependabot alerts | Read |
   | Members | Read |
   | Metadata | Read |

   Homepage URL, callback URL, and webhook can be left disabled or filled with a placeholder value.
3. Install the App on this repository and on every `WisewareOrg` repository autodiscovery should
   reach — `autodiscoverFilter` matches `WisewareOrg/*`, so that is every repository the App can
   see.
4. Add two secrets to this repository (Settings → Secrets and variables → Actions):
   - `RENOVATE_APP_ID` — the App's **Client ID** (shown on the App's settings page; the workflow
     passes it as `create-github-app-token`'s `client-id` input, which the action's current release
     documents as the primary form — the older `app-id` numeric ID is a deprecated alias).
   - `RENOVATE_APP_PRIVATE_KEY` — the App's private key (generate one on the App's settings page;
     paste the full PEM contents).

## Manual run / dry run

Trigger the workflow from the Actions tab (`workflow_dispatch`) or `gh workflow run renovate.yml`.
The `dryRun` input selects Renovate's dry-run mode for that run:

- `none` — runs for real (the default; also what the schedule uses)
- `extract` — dependency extraction only
- `lookup` — extraction plus version lookups, no branches or PRs
- `full` — a complete dry run: logs what would happen without pushing branches, opening PRs, or
  writing the Dependency Dashboard issue

The scheduled run (every 6 hours, `0 */6 * * *`) never sets a dry-run mode.

## Releasing the preset

Tag a release when `default.json` (or `config.json`) changes:

```
git tag vX.Y.Z
git push --tags
```

Each consuming repository's `renovate.json` pins `extends` to a tag (`#vX.Y.Z`); bump that tag
reference in the consuming repository's own PR to pick up a preset change.
