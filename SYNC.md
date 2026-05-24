# Syncing the yafva fork with upstream

This fork lives at [`Outburn-IL/org.hl7.fhir.core`](https://github.com/Outburn-IL/org.hl7.fhir.core)
and tracks [`hapifhir/org.hl7.fhir.core`](https://github.com/hapifhir/org.hl7.fhir.core)
plus a small set of yafva commits on top of `master`:

| Commit subject                                                | Why it exists                                                                 |
|---------------------------------------------------------------|-------------------------------------------------------------------------------|
| `fork: cache unversioned CODESYSTEM_UNSUPPORTED responses`    | The one behavioral patch — keeps the terminology cache from re-querying tx.   |
| `build: publish fork artifacts to Outburn-IL GitHub Packages` | Adds the `yafva-github-repo` Maven profile, this doc, and `yafva-publish.yml`.|
| `build(ci): include validation.cli in deploy reactor`         | Without `cli` in the reactor, `report` falls through to a private repo (401). |

Downstream consumer: [`Outburn-IL/yafva.jar`](https://github.com/Outburn-IL/yafva.jar)
resolves these artifacts from `maven.pkg.github.com/Outburn-IL/org.hl7.fhir.core`.

## Manual sync from upstream

```bash
git fetch upstream
git rebase upstream/master                  # carries the 3 yafva commits forward
# If the fork patch conflicts (TerminologyCache.java moved), fix it manually.
git push origin master --force-with-lease
```

After every sync, verify the GitHub Actions workflow state — see
[Disabled workflows](#disabled-workflows) below.

## Publish to GitHub Packages

The fork publishes via the [`yafva-publish.yml`](.github/workflows/yafva-publish.yml)
workflow, which is triggered by tags matching `v*-yafva.*`. **Tags are anchored
on the upstream release commit, not on `master` HEAD** — this keeps the
artifact's version semantically honest (the deployed code matches the upstream
release it claims to be based on).

```bash
# Anchor on the upstream release you want to ship (e.g. v6.9.8)
git checkout --detach <upstream-release-sha>      # e.g. f8b250d58 for v6.9.8

# Cherry-pick the yafva commits (use the 3 SHAs from `git log upstream/master..master`)
git cherry-pick <patch-sha> <publish-infra-sha> <ci-fix-sha>

# Tag and push — the workflow fires on the tag push
git tag -a v6.9.8-yafva.2 -m "yafva fork build of upstream v6.9.8"
git push origin v6.9.8-yafva.2

git checkout master
```

`<n>` in `-yafva.<n>` increments per re-publish of the same upstream base
(e.g. `-yafva.2` supersedes `-yafva.1` if the first attempt failed or the
patch was refined without bumping upstream).

Manual run (any branch/SHA, e.g. when iterating on the workflow itself):

```
Actions → "yafva — Publish to GitHub Packages" → Run workflow → version=6.9.8-yafva.<n>
```

## Consuming the artifacts

In yafva.jar's `pom.xml`, bump `hapi.fhir.version` to the new `-yafva.<n>`
and push. Authentication notes:

- **CI** (`Outburn-IL/yafva.jar`): the default `GITHUB_TOKEN` works because
  the workflow runs inside the same org that owns the packages. Make sure
  the job has `permissions: packages: read`.
- **Local dev**: needs a PAT with `read:packages`. Easiest path:
  ```powershell
  gh auth refresh -h github.com -s read:packages
  $env:GITHUB_PACKAGES_TOKEN = gh auth token
  ```
  and a `~/.m2/settings.xml` `<server id="github-yafva">` referencing
  `${env.GITHUB_PACKAGES_TOKEN}`.

## Disabled workflows

The fork's `master` is rebased from upstream, which means every upstream
workflow file under `.github/workflows/` lands here too. They were
**manually disabled** on the GitHub Actions UI to avoid burning minutes on
upstream-specific CI (CodeQL, Crowdin, OWASP, etc.):

- `bidi-checker.yml`
- `codeql.yml`
- `crowdin.yml`
- `license-check.yml`
- `owasp.yml`
- `scorecard.yml`
- `sql-on-fhir-tests.yml`
- `trivy.yml`

Disable state is per-workflow-id, so upstream edits don't silently re-enable
them. **But if upstream renames a workflow, the renamed copy lands as
active** — re-disable it after each sync.

To re-disable via gh:

```bash
gh api -X PUT repos/Outburn-IL/org.hl7.fhir.core/actions/workflows/<id>/disable
```
