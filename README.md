# Mend SCA -- NuGet `<clear/>` + private-only feed repro

## What this repo proves

A NuGet `nuget.config` that uses `<packageSources><clear/>...</packageSources>`
followed by a single private source is the standard NuGet idiom for
**"resolve only from this feed, never fall through to nuget.org"**.

The customer reports that on a Mend for Azure Repos SCA scan the resolver
appends nuget.org anyway and surfaces public packages in the dep tree. This
repo is the minimal reproduction.

## Layout

| File | Why it's here |
|---|---|
| `MendNugetClearRepro.csproj` | Consumer. One `PackageReference` to `MendDetectionQa.NugetClearProbe@1.0.0`. |
| `nuget.config` | `<clear/>` + GitHub Packages NuGet as the only source. Customer-intent shape. |
| `.whitesource` | `configMode: LOCAL` so Mend reads `whitesource.config` below. |
| `whitesource.config` | UA properties intended to keep the scanner from injecting nuget.org. |
| `Program.cs` | Compiles; not load-bearing for the scan. |
| `private-package/` | Producer. Builds `MendDetectionQa.NugetClearProbe.nupkg` for publishing to GHP. See its README for the publish flow. |

## Setup -- one-time

We use **GitHub Packages NuGet** under the `mend-detection-qa` org as the
private feed. The Mend internal Nexus (`nexus.mendinfra.com`) is Maven-format
only -- verified via `/service/rest/v1/repositories` -- so it cannot be used.

1. Create the repo `mend-detection-qa/nuget-clear-private-only` on GitHub.
2. Publish the dummy package: see `private-package/README.md` for the
   `dotnet pack` + `dotnet nuget push` commands. Needs `GH_TOKEN` with
   `write:packages` scope.
3. Push this directory's contents to that repo's `main` branch.
4. Accept Mend's "Configure Mend for GitHub.com" PR when it appears.

## How to run the scan

1. Push this directory to a repo under the Mend-monitored Azure DevOps project.
2. Wait for Mend's SCA check to run, or trigger it manually.
3. Open the Security Check details -> Scan Logs.

## What to look for in the logs (the actual question)

The repro answers two questions in one scan:

**Q1: Did Mend honor `<clear/>`?**

Search the scanner log for these strings:

- `nuget.org` -- if it appears as an *active* source in any line resembling
  `Added package source 'nuget.org'` or `Restoring from https://api.nuget.org/...`,
  the scanner appended it. That's the bug.
- `Using credentials from config` / `package source ... private` -- expected.
- `dotnet restore` invocation lines -- copy the full command. If the scanner
  added `--source https://api.nuget.org/v3/index.json` or wrote a temporary
  `NuGet.Config` next to your project, that's the injection path.

**Q2: Did any resolved dependency actually come from nuget.org?**

In the dep tree (update-request JSON) look at each dependency's `sha1` / coordinates.
Any package that is on nuget.org but NOT on your private feed is conclusive
evidence of fall-through. (`Contoso.Internal.Utilities` won't be on nuget.org;
if its transitive closure pulls e.g. `Newtonsoft.Json` and that resolves from
nuget.org instead of failing, the scanner went around `<clear/>`.)

## Expected vs actual

| Behavior | Expected (per `<clear/>` spec) | What customer reports |
|---|---|---|
| Active sources during restore | `private` only | `private` + nuget.org |
| Dep tree contents | Packages from `private` only | Includes packages sourced from nuget.org |
| Restore failure on missing transitive | Fail fast on private feed | Silently satisfied from nuget.org |

## Supported levers tried in `whitesource.config`

- `nuget.preferredEnvironment=dotnet` -- forces `dotnet restore` (more
  config-respecting than legacy `nuget.exe`).
- `nuget.resolveNugetConfigs=true` -- newer UA flag to honor the in-repo config.
- `nuget.runPreStep=true` -- runs restore once so the dep tree is read from
  `project.assets.json` instead of re-resolved by the scanner.

If even with all three set the dep tree still contains nuget.org packages,
that's the supportability question to escalate -- there is no further
customer-side knob today.

## Confirmed non-applicable

- Reusing `nexus.mendinfra.com/repository/maven-local` -- Maven format, NuGet
  client cannot consume it. Verified against the Nexus repositories API.