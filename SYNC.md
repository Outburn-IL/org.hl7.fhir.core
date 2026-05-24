# Syncing the yafva fork with upstream

This fork carries one patch on top of `hapifhir/org.hl7.fhir.core` — the
unversioned-CODESYSTEM_UNSUPPORTED caching change in
`org.hl7.fhir.r5/.../terminologies/utilities/TerminologyCache.java` (see commit
history). All other commits should track upstream verbatim.

Downstream consumer: [`Outburn-IL/yafva.jar`](https://github.com/Outburn-IL/yafva.jar)
resolves the fork's Maven artifacts from
`maven.pkg.github.com/Outburn-IL/org.hl7.fhir.core`.

## Manual sync

```bash
git fetch upstream --tags
# Pick the upstream tag you want to track, e.g. v6.9.8
git checkout -B master upstream/master
# Re-apply the local patch (cherry-pick or 3-way merge — there is only one)
git cherry-pick <patch-sha>
git push origin master --force-with-lease
```

## Publish to GitHub Packages

Tag with `v<upstream-version>-yafva.<n>` and push — the
[`yafva-publish.yml`](.github/workflows/yafva-publish.yml) workflow runs
`mvn deploy` against the `yafva-github-repo` profile.

```bash
git tag v6.9.8-yafva.1
git push origin v6.9.8-yafva.1
```

`<n>` increments per re-publish of the same upstream base (e.g. `-yafva.2` if
the first attempt failed or the patch was refined without bumping upstream).

Manual run (any branch/SHA):

```
Actions → "yafva — Publish to GitHub Packages" → Run workflow → version=6.9.8-yafva.1
```

## Consuming the artifacts

Bump `hapi.fhir.version` in `yafva.jar`'s `pom.xml` to the new
`6.9.8-yafva.<n>` and push. CI pulls from GitHub Packages using
`GITHUB_TOKEN` with `read:packages`.
