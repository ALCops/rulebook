# New diagnostics and quarantine

The AL compiler, the Microsoft code cops and the ALCops analyzers gain new diagnostics with every release. Your rulebook decides what happens to them: the **Scan Diagnostics** workflow reads the newest packages every day, records every new diagnostic in your catalog and holds it back in the stages you choose (its *quarantine*) until you adopt it. It keeps one pull request with everything it found, so a new rule never reaches your AL projects without a review.

> **Status:** written 2026-10-08 with the scan work package of the engine ([WP08, #10](https://github.com/ALCops/rulebook-engine/issues/10)). Everything below was observed in a live run on a scratch repository or is stated by the engine's contributor reference, [scan-mechanics.md](https://github.com/ALCops/rulebook-engine/blob/main/docs/reference/scan-mechanics.md), which also holds the run table. The scan writes with the same secret as the update: [ghtokenworkflow.md](ghtokenworkflow.md).

## Contents

1. [What the scan does](#1-what-the-scan-does)
2. [Choosing the policy](#2-choosing-the-policy)
3. [Reading the pull request](#3-reading-the-pull-request)
4. [Adopting a quarantined rule](#4-adopting-a-quarantined-rule)
5. [Changed defaults](#5-changed-defaults)
6. [The catalog](#6-the-catalog)
7. [Running it by hand](#7-running-it-by-hand)
8. [Troubleshooting](#8-troubleshooting)

## 1. What the scan does

The scan reads two packages from nuget.org: **Microsoft.Dynamics.BusinessCentral.Development.Tools** (the compiler and the CodeCop, UICop, AppSourceCop and PerTenantExtensionCop analyzers, called `tools` in titles) and **ALCops.Analyzers** (the ALCops cops, called `alcops`). Of each it takes the newest stable version and the newest prerelease, the prerelease only when it is newer than the stable version: an ALCops prerelease such as 1.3.0-beta.1 that is older than the stable 1.3.1 is not scanned.

`catalog/scan-state.json` records which versions it has scanned. When neither package has a new version, the run ends in seconds with the notice "No new package version (...); nothing to do". When there is one, the scan reads every diagnostic of the new version, compares them with `catalog/diagnostics.json` and then:

- adds every new diagnostic to the catalog and to the quarantine files your policy names ([section 2](#2-choosing-the-policy));
- releases quarantined diagnostics that a level file has adopted in the meantime ([section 4](#4-adopting-a-quarantined-rule));
- records changed default severities ([section 5](#5-changed-defaults));
- regenerates the endpoints in `rulesets/`, runs the full validation and puts the result into **one pull request** on the branch `scan-diagnostics/main`.

The workflow runs every day at 04:17 UTC (the `scan.schedule` setting, [section 7](#7-running-it-by-hand)) and whenever you start it under Actions > **Scan Diagnostics** > Run workflow. It needs the `GHTOKENWORKFLOW` secret ([ghtokenworkflow.md](ghtokenworkflow.md)).

## 2. Choosing the policy

There is no default policy: until you set both keys of `quarantine` in `.github/Rulebook-Settings.json`, every run stops before it reads anything, with

> Set quarantine.stages and quarantine.prereleaseStages in .github/Rulebook-Settings.json. Typical choice: quarantine default and ci, leave vnext out so it shows new rules at their default severity.

| Key | Receives |
|---|---|
| `quarantine.stages` | Diagnostics first seen in a **stable** package version, and prerelease diagnostics when they reach a stable version. |
| `quarantine.prereleaseStages` | Diagnostics first seen in a **prerelease**. |

Both are lists of stage names in lowercase, as they appear in your file names (`default`, `ci`, `vnext`, or a stage you added). Two typical policies:

```json
"quarantine": { "stages": ["default", "ci"], "prereleaseStages": ["default", "ci"] }
```

New rules stay off in the editor and in your pipelines until you decide, and `vnext` shows them at their default severity, so builds against the next platform warn you early.

```json
"quarantine": { "stages": [], "prereleaseStages": ["default", "ci"] }
```

A new rule of a stable release is active everywhere at once, at the severity its analyzer gives it; only prerelease rules are held back. `[]` is valid for either key and quarantines nowhere: the new diagnostics are then only recorded in the catalog.

A quarantined diagnostic is `None` in the endpoints of that stage, for every level, until a level file mentions it. Each stage has its own file, `quarantine.<stage>.json`, `quarantine.default.json` included. A stage you add gets its file the first time the scan writes to it; in the live run, adding a stage `Nightly` and `nightly` to `quarantine.stages` created `quarantine.nightly.json` with the next new diagnostic. A file named after a stage your settings do not have is not read by anything; Validate warns about it (C16).

## 3. Reading the pull request

The pull request is titled with the counts first and then the package versions it scanned, for example

- `Scan diagnostics: 3 new ids quarantined (alcops 1.3.1, tools 18.0.43.1464, tools 30.0.42.60748-beta)`
- `Scan diagnostics: 1 quarantine entry released, no new package version`
- `Scan diagnostics: alcops 1.3.1 recorded, no new diagnostics`

The last one is a **record run**: a new package version that brought nothing new. It still opens a pull request, because the catalog and `catalog/scan-state.json` record the version; merge it so the next run does not scan that version again. The parts of a title, in this order: new diagnostics quarantined, new diagnostics recorded (no policy stage took them), diagnostics promoted to stable, defaults changed, quarantine entries released.

The body has these sections, each only when it has something to show:

| Section | Example from the live run |
|---|---|
| Scanned versions | `\| alcops.analyzers \| stable \| 1.3.1 \| none \| 127 \| 2 \| 0 \|`: package, channel, version, the version recorded before, the number of diagnostics, the new ones and the changed defaults. |
| New diagnostics | `\| LC0043 \| LinterCop \| Warning \| Use SecretText type to protect credentials ... \| docs \| alcops.analyzers 1.3.1 (stable) \| default, ci \|`: the default severity (`Warning (disabled)` for a rule that is off by default), the link to its documentation, where it was seen, and where it is quarantined (`nowhere (policy [])` when the policy key is empty). |
| Promoted to stable | A prerelease diagnostic that a stable version now carries, added to `quarantine.stages`. |
| Changed defaults | See [section 5](#5-changed-defaults). |
| Prerelease default changes (not applied) | A prerelease that changes a default; nothing changes until a stable version carries it. |
| Released from quarantine | `\| default \| LC0043 \| base/essential.ruleset.json \|`: the stage, the diagnostic and the level file that now mentions it. |
| Catalog notes | For example "622 seeded ids got their package fields", "Refreshed from the stable descriptors: 1 title, 2 docs links", the diagnostics no analyzer advertises ([section 6](#6-the-catalog)). |
| Changes | Every file the pull request changes. |
| Effective diff | One table per endpoint: before, after, and what decided it, for example `\| LC0043 \| (unknown default) \| None \| quarantine, "New in alcops.analyzers 1.3.1 (stable), quarantined 2026-10-08. Review and adopt." \|`. A diagnostic the catalog did not know before shows `(unknown default)` on the left. |
| Validation warnings | Warnings of the validation of the result; an error stops the run instead. |

**One pull request, rebuilt every run.** The scan does not open a new pull request per run. Every run rebuilds the branch `scan-diagnostics/main` from the current `main` as one commit with the whole current result, and updates the title and body of the open pull request ("Pull request updated: ..."). That keeps it mergeable after other pull requests and makes the body describe exactly what the branch holds. **Do not push to that branch**: the next run replaces your commit. Merge the pull request, or close it; a closed one comes back with the next run, because `main` still lacks the result (to stop that, change the policy or adopt the diagnostics). When `main` already holds the result, or the result went in as a direct commit ([section 7](#7-running-it-by-hand)), the scan closes an open scan pull request and starts its body with a line that says why.

Every quarantine entry carries its reason, for example

```json
{ "id": "LC0043", "justification": "New in alcops.analyzers 1.3.1 (stable), quarantined 2026-10-08. Review and adopt." }
```

## 4. Adopting a quarantined rule

To adopt a diagnostic, mention it in a level file, for example add `{ "id": "LC0043", "action": "Error" }` to `base/essential.ruleset.json` (or to a level file of your own). From then on the level decides and quarantine no longer applies to it. An entry in `overrides.json` also wins over quarantine for the levels and stages it selects, but it does not release the quarantine entry: only a level file does. Validate reports the leftover quarantine entry as a warning (C13), and the next scan removes it from every quarantine file: in the live run that was the pull request `Scan diagnostics: 1 quarantine entry released, no new package version`, with only `quarantine.default.json` and `quarantine.ci.json` changed. This happens even when no package has a new version.

To keep a diagnostic off on purpose, leave it in quarantine, or adopt it with the action `None` so the decision is in your level file instead of the scan's file.

Removing an entry from a quarantine file by hand also adopts it, at its default severity.

## 5. Changed defaults

Your endpoints list only the diagnostics whose action differs from the analyzer's default. When a stable version changes a default severity, or turns a rule on or off by default, the scan updates `defaultSeverity` or `enabledByDefault` in the catalog, records the change as a `defaultChanges` element of that entry, and lists it in the pull request with its effect, for example "now listed at Info in strict.ci; now unlisted in strict.default". A diagnostic whose level action now equals the new default drops out of that endpoint; one whose default moved away from the level action appears in it. The effective action in your AL projects stays what your levels say.

To keep a diagnostic at a fixed severity whatever its analyzer does, give it that action in a level file or in `overrides.json`. A prerelease never changes a default: such a change is listed under "Prerelease default changes (not applied)" and waits for the stable version.

The live run could not show a changed default (nuget.org cannot be steered); the engine's tests prove it with test packages.

## 6. The catalog

`catalog/diagnostics.json` lists every diagnostic your rulebook knows, one entry per line. The template ships it with 628 entries (`id`, `analyzer`, `defaultSeverity`, `enabledByDefault`, `title`, `docs`). These shipped diagnostics are known: the scan never quarantines them; the first scan only adds their package fields. The scan writes:

| Field | Meaning |
|---|---|
| `package`, `firstSeenVersion`, `firstSeenChannel` | The package and the first scanned version that carried the diagnostic, `stable` or `prerelease`. |
| `firstStableVersion` | The first stable version that carried it. |
| `lastSeenVersion` | The newest version that carried it; it only moves forward. |
| `title`, `docs` | Refreshed from every stable version. |
| `advertised: false` | The package defines the diagnostic but no analyzer reports it (for example AS0141 and the ALCops `xx0000` exception diagnostics). It stays in the catalog and is never quarantined. When an analyzer starts reporting it, it is quarantined like a new diagnostic. |
| `deprecated: true` | The analyzer marks the diagnostic as deprecated. |
| `defaultChanges` | One element per changed default of a stable version, [section 5](#5-changed-defaults). |

A diagnostic that a newer package no longer carries keeps its entry, with its old `lastSeenVersion`. In the first live scan, 625 of the 628 shipped entries got package fields; AC0033, AC0034 and TA0002 are documented by ALCops but in no released package yet, so they keep none.

`catalog/scan-state.json` is created by the first scan; the template does not ship it. It holds the version scanned last per package and channel. Once it exists, every diagnostic named in a level file, a stage file, `base/twins.json`, `overrides.json` or a quarantine file must be in the catalog: a missing one is an error in Validate (C7), where it was a warning before the first scan.

## 7. Running it by hand

Actions > **Scan Diagnostics** > Run workflow:

| Input | Default | Use |
|---|---|---|
| Also scan the newest prerelease | on | Off scans only the stable versions; the prerelease entries of `catalog/scan-state.json` stay as they are. |
| Push to this branch instead of updating the scan pull request | off | A direct commit, see below. |

**Direct commit.** With the input on, the result is committed to `main` and the summary goes to the run's job summary only. When the push is refused, because a ruleset or branch protection requires pull requests, the scan opens the pull request instead and says "(the direct commit was refused)". The scheduled run does the same when `commitOptions.createPullRequest` is `false` in your settings.

**The schedule.** `scan.schedule` in your settings is the cron expression of the daily run (shipped `"17 4 * * *"`). The Update Rulebook System Files workflow writes it into `ScanDiagnostics.yaml` ([updating.md](updating.md)). `null` removes the schedule, so the scan runs only by hand. A settings file without a `scan` key, such as one created before the scan existed, keeps the shipped daily schedule.

**Scanning a version again.** The scan skips a version that `catalog/scan-state.json` records. To scan a package channel again, set its entry there to `null`, for example `"stable": null`, and run the workflow.

## 8. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "Set quarantine.stages and quarantine.prereleaseStages in .github/Rulebook-Settings.json. ..." | The policy is not set. See [section 2](#2-choosing-the-policy). |
| "The GHTOKENWORKFLOW secret is needed to scan diagnostics. Read ..." | The secret named by `ghTokenWorkflowSecretName` (default `GHTOKENWORKFLOW`) is missing or empty in this repository. See [ghtokenworkflow.md](ghtokenworkflow.md); its section on failures covers an expired token or an app without access. |
| "The scanned rulebook would not validate: LC0999: LC0999 is not in catalog/diagnostics.json (catalog/scan-state.json exists); add LC0999 to catalog/diagnostics.json or remove it from base/complete.ruleset.json; ..." | A file names a diagnostic the catalog does not know, a typo for example. After the first scan this is an error. Fix the name, or add the entry. Nothing was pushed. |
| "The diagnostics could not be extracted: Extraction failed for ..." | A package version could not be read, for example an analyzer that fails to load. Nothing was pushed and your catalog stays as it was; the run works again once a fixed version of the engine reads the package. |
| "The NuGet packages could not be read: ..." | nuget.org was not reachable or kept answering with an error (the scan retries three times). Run the workflow again later. |
| "The base branch moved during the scan (...); nothing was pushed, the next run will pick it up." | Someone merged into `main` while the scan ran. Run it again, or wait for the next day. |
| The pull request did not update | The run ended with "No new package version (...); nothing to do": nothing new was found, so there was nothing to rebuild. Or a commit was pushed to `scan-diagnostics/main` between the scan reading the branch and pushing it; the run then failed, and the next run rebuilds the branch. |
| A diagnostic you removed from the catalog by hand does not come back | The scan only scans versions it has not recorded. The diagnostic returns with the next package version, or right away when you set the channel's entry in `catalog/scan-state.json` to `null` ([section 7](#7-running-it-by-hand)). |
