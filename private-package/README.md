# Dummy package — `MendDetectionQa.NugetClearProbe`

A trivial netstandard2.0 class library that exists only so we have a NuGet
package distinctive enough to prove which feed served it. The package name
intentionally does NOT exist on nuget.org; any resolved-from-nuget.org line
in the scan log is conclusive evidence the scanner appended that source.

## Publish to GitHub Packages

GitHub Packages needs:

1. A Personal Access Token (classic) with `write:packages` (and `read:packages`)
   scope, OR a fine-grained PAT with **Packages: read & write** on the
   `mend-detection-qa` org. Export it as `GH_TOKEN`.
2. The repo `mend-detection-qa/nuget-clear-private-only` to exist (we push it
   in the next step). GHP binds the package to the repo named in `<RepositoryUrl>`.

```bash
# from temp_autotest_workflow/nuget-clear-private-only/private-package
dotnet pack -c Release

# Add GHP as a NuGet source, scoped to this shell (no global pollution)
dotnet nuget add source https://nuget.pkg.github.com/mend-detection-qa/index.json \
  --name github-mend-detection-qa \
  --username mend-detection-qa \
  --password "$GH_TOKEN" \
  --store-password-in-clear-text

dotnet nuget push bin/Release/MendDetectionQa.NugetClearProbe.1.0.0.nupkg \
  --source github-mend-detection-qa \
  --api-key "$GH_TOKEN"
```

Verify via UI: `https://github.com/orgs/mend-detection-qa/packages?repo_name=nuget-clear-private-only`

## Re-publishing a new version

GHP rejects overwriting an existing version. Bump `<Version>` in the .csproj
(e.g. `1.0.1`), re-pack, re-push. Update the consumer's `PackageReference`
version to match.

## Auth at scan time

GitHub Packages does **not** allow anonymous NuGet reads — Mend's scanner
needs credentials to actually download the package. For this repro we do
**not** need restore to succeed; we only need the scan logs to show what
sources the resolver tried. If `nuget.org` shows up in those logs despite
`<clear/>`, the question is answered regardless of restore success.

If you do want successful restore (cleaner dep tree in Mend's UI), put a
read-only PAT in the consumer-side `nuget.config`'s `packageSourceCredentials`
block — but only after making the repo PRIVATE so the PAT isn't world-readable.