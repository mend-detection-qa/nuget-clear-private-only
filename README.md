# UVScanTest -- Mend SCA NuGet `<clear />` bypass repro

Mirror of `Pension Services Corporation Ltd / UVScanTest` (TKA-10556).

## What this repo proves

`nuget.config` uses `<clear />` plus a single private package source. The csproj
pins five public-NuGet packages (`Microsoft.NET.Test.Sdk`, `Moq`, `NUnit`,
`NUnit.Analyzers`, `NUnit3TestAdapter`) at exact versions that exist on
`api.nuget.org`. **The private feed does NOT host those exact versions** -- it
hosts same-named packages at sentinel version `4.3.2-mend-probe`.

The discriminator is unambiguous:

| Behaviour | Dep tree contents |
|---|---|
| Mend honours `<clear />` | restore can't find e.g. `NUnit 4.3.2` on the private feed -> restore fails -> dep tree empty (or short with only the `-mend-probe` resolutions if the resolver picks the sentinel) |
| Mend appends `nuget.org` | restore succeeds against public feed -> dep tree shows the real public versions, e.g. `NUnit 4.3.2`, `Moq 4.20.72`, etc. |

A "real 4.3.2" appearing in the dep tree is conclusive evidence the scanner
queried `api.nuget.org` despite the `<clear />` directive.

## Customer-side context (from TKA-10556)

* Customer: Pension Services Corporation Ltd. Mend SCA -- Mend for Azure Repos.
* Customer's private feed `https://packman-corp.k8s-pic.int/...` is internal DNS,
  unreachable from Mend's scan infra. They saw 11 packages resolved in the dep
  tree -- the only feed that can serve those packages is `api.nuget.org`.
* Mend scanner log on the customer's failing scan:

  ```
  handling nuget.config files
  No nuget.config files found: falling-back to creating nuget.config files
  nuget.config files to be handled: nuget.config
  editing nuget.config
  reading nuget.config
  writing nuget.config with new data
  ```

  The scanner says it couldn't find the file (despite it being committed at the
  repo root co-located with the csproj), then synthesises its own and proceeds
  to "write with new data". The customer's `<clear />` never takes effect.

## File parity with customer repo

| File | Notes |
|---|---|
| `Pic.Eft.Globalscape.Tests.csproj` | Identical PackageReferences, framework, `NuGetAuditMode=direct`. Parent-dir references (`..\Pic.Eft.Globalscape\`, `..\stylecop.json`, `..\.editorconfig`) preserved verbatim -- they're broken in the stripped-down customer repo too. |
| `nuget.config` | `<clear />` + single source `PIC-UK-TeamFeed`. URL points at our GHP feed instead of the customer's unreachable internal Nexus. |
| `.whitesource` | `configMode: AUTO`, mirroring customer's `repo-config.json` shape. `baseBranches: []` (defaults to repo default) because we don't have a `production-code-scan` branch. |
| `manifest.json`, `package-lock.json`, `pyproject.toml`, `uv.lock` | Empty/minimal stubs preserving the customer's polyglot tree. Carried in case Mend's pre-step changes behaviour when co-located PM files are present. |

## Diverged on purpose

* **No `production-code-scan` branch.** Customer scans on that name as base; our
  default is `main`. Empty `baseBranches` array uses repo default.
* **Private feed URL.** Customer's `packman-corp.k8s-pic.int` is internal DNS we
  can't reach; substituted with our `mend-detection-qa` GHP NuGet endpoint
  (which is reachable from Mend infra and serves the same-named sentinel
  packages).

## Private-feed contents (published to GHP)

Each package is a trivial `netstandard2.0` shim whose only purpose is to
exist on the private feed with the matching `PackageId` and a sentinel version:

| Package | Sentinel version on private feed |
|---|---|
| `Microsoft.NET.Test.Sdk` | `17.12.0-mend-probe` |
| `Moq`                    | `4.20.72-mend-probe` |
| `NUnit`                  | `4.3.2-mend-probe`  |
| `NUnit.Analyzers`        | `4.6.0-mend-probe`  |
| `NUnit3TestAdapter`      | `4.6.0-mend-probe`  |

Producer csproj projects live outside this repo (in
`../nuget-clear-private-producers/`) so the consumer scan surface stays clean.

## How to read the scan output

* **Dep tree shows `NUnit 4.3.2` (no suffix) + a public-feed `sha1`** -> scanner
  queried `api.nuget.org`. Repro succeeded.
* **Dep tree shows `NUnit 4.3.2-mend-probe`** -> scanner honoured `<clear />`
  and used only the private feed. Bug NOT reproduced on this Mend integration.
* **Dep tree empty / restore failed** -> something earlier broke, look at the
  scanner log for `handling nuget.config files` / `No nuget.config files found`
  -- that's the customer's pre-step in action.