# AL-Go for GitHub

How an AL-Go for GitHub repository compiles against your rulebook: the skeleton files in the app, `rulesetFile` and `enableExternalRulesets` in the AL-Go settings, the next-major build on the `vnext` stage, and how to see that it works.

> **Status:** written 2026-10-10 with WP11 of the engine ([#13](https://github.com/ALCops/rulebook-engine/issues/13)). The AL-Go behaviour below is taken from the AL-Go documentation and source on that date ([settings.md](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md), `ReadSettings.psm1`, `RunPipeline.ps1`, `CompileApps/Compile.ps1`) and BcContainerHelper's `Run-AlPipeline`. It was not run in an AL-Go repository; the compiler behaviour (AL1033 aborts the compile) was observed with `alc` in the engine's spikes. The project-side facts are in [al-project.md](al-project.md).

## Contents

1. [What you need](#1-what-you-need)
2. [Put the skeletons in the app](#2-put-the-skeletons-in-the-app)
3. [Point AL-Go at the ci file](#3-point-al-go-at-the-ci-file)
4. [The next-major build on vnext](#4-the-next-major-build-on-vnext)
5. [Stages and workflows](#5-stages-and-workflows)
6. [Check that it works](#6-check-that-it-works)
7. [When the endpoint fails](#7-when-the-endpoint-fails)
8. [Shipping the files in a custom AL-Go template](#8-shipping-the-files-in-a-custom-al-go-template)
9. [Troubleshooting](#9-troubleshooting)

## 1. What you need

- An AL-Go for GitHub repository, created from [AL-Go-PTE](https://github.com/microsoft/AL-Go-PTE) or [AL-Go-AppSource](https://github.com/microsoft/AL-Go-AppSource).
- Your published rulebook: `<baseUrl>/` opens the index page ([getting-started.md](getting-started.md)).
- The level your apps build against, for example `strict` ([levels.md](levels.md)).

## 2. Put the skeletons in the app

Run the init script in the app folder, the folder with `app.json` ([al-project.md](al-project.md) section 3):

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/ALCops/rulebook-engine/main/scripts/Get-RulebookSkeletons.ps1 -OutFile $env:TEMP/Get-RulebookSkeletons.ps1
& $env:TEMP/Get-RulebookSkeletons.ps1 -BaseUrl https://contoso.github.io/rulebook -Level strict
```

It writes `.rulebook/default.ruleset.json`, `.rulebook/ci.ruleset.json` and `.rulebook/vnext.ruleset.json` next to `app.json`. Commit the folder. VS Code uses `default`, AL-Go uses `ci` and `vnext`.

## 3. Point AL-Go at the ci file

**Where the path starts.** AL-Go resolves a relative `rulesetFile` against the **project folder**, the folder that holds `.AL-Go/settings.json`, not against the app folder: `Run-AlPipeline` runs with the project folder as its base folder, and the workspace-compilation path joins the value with the project folder. In a single-project repository the project folder is the repository root, and the app sits in a subfolder such as `MyApp/`. So the path includes the app folder.

In the project settings file `.AL-Go/settings.json`, for an app in `MyApp/`:

```json
{
  "rulesetFile": "MyApp/.rulebook/ci.ruleset.json",
  "enableExternalRulesets": true,
  "enableCodeCop": true,
  "enableUICop": true
}
```

| Setting | Value | AL-Go documentation |
|---|---|---|
| `rulesetFile` | The `ci` skeleton, relative to the project folder. Empty by default: AL-Go never picks up a ruleset file by itself. | [rulesetFile](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#rulesetFile) |
| `enableExternalRulesets` | `true`. Default `false`; without it the URL include of the skeleton fails with AL1033 (section 7). | [enableExternalRulesets](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#enableExternalRulesets) |
| `enableCodeCop`, `enableUICop` | `true`. Both default to `false`, and a rule of an analyzer that does not run never fires, whatever the endpoint says. | [enableCodeCop](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#enableCodeCop), [enableUICop](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#enableUICop) |
| `enablePerTenantExtensionCop`, `enableAppSourceCop` | Leave them: AL-Go turns on the cop of the project type (PTE or AppSource app). Opting out of the other kind's rules is [pte-or-appsource.md](pte-or-appsource.md). | [enablePerTenantExtensionCop](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#enablePerTenantExtensionCop), [enableAppSourceCop](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#enableAppSourceCop) |
| `failOn` | Default `error`: a diagnostic the endpoint sets to Error fails the build, a Warning does not. `warning` or `newWarning` fail on warnings too. | [failOn](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#failOn) |

The ALCops analyzers run in AL-Go through the `PipelineInitialize.ps1` hook of [ALCops/AL-Go](https://github.com/ALCops/AL-Go); without them the endpoint's `LC` and other ALCops ids do nothing.

**Several apps in one project.** `rulesetFile` is one value per project and applies to every app the project builds. Point it at the files of one app, or keep one `.rulebook/` folder in the project folder for all its apps and point VS Code at that folder too ([vscode.md](vscode.md) section 3).

**Several projects in one repository.** Each project has its own `.AL-Go/settings.json`, and its paths start at its own folder (`projectA/.AL-Go/settings.json` with `"rulesetFile": "MyApp/.rulebook/ci.ruleset.json"` means `projectA/MyApp/...`).

## 4. The next-major build on vnext

The **Test Next Major** workflow builds against the next major version of Business Central. AL-Go reads a workflow-specific settings file `.github/<workflow name>.settings.json` after the project settings, so its values win. The workflow name is the `name:` of the workflow with leading and trailing spaces and characters not allowed in a file name removed: `' Test Next Major'` becomes `Test Next Major`. The AL-Go-PTE and AL-Go-AppSource templates ship `.github/Test Next Major.settings.json` already (with the `artifact` setting); add `rulesetFile` to it:

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/AL-Go-Actions/v10.0/.Modules/settings.schema.json",
  "artifact": "////nextmajor",
  "cacheImageName": "",
  "versioningStrategy": 15,
  "rulesetFile": "MyApp/.rulebook/vnext.ruleset.json"
}
```

Keep the values the template shipped and add only the `rulesetFile` line. The path starts at the project folder here too. A `.github/<workflow>.settings.json` applies to every project of the repository; for one project only, use `<project>/.AL-Go/Test Next Major.settings.json` ([where are the settings located](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#where-are-the-settings-located)).

## 5. Stages and workflows

| Rulebook stage | Skeleton | AL-Go workflow | Settings file |
|---|---|---|---|
| `default` | `.rulebook/default.ruleset.json` | none: VS Code ([vscode.md](vscode.md)) | `.vscode/settings.json` |
| `ci` | `.rulebook/ci.ruleset.json` | CI/CD, Pull Request Build, Test Current, Test Next Minor | `.AL-Go/settings.json` |
| `vnext` | `.rulebook/vnext.ruleset.json` | Test Next Major | `.github/Test Next Major.settings.json` |

A stage you added to your rulebook ([levels.md](levels.md) section 7) gets its own skeleton; point the workflow that should use it at that file through its own `.github/<workflow name>.settings.json`.

## 6. Check that it works

An endpoint lists only the rules whose action differs from the analyzer default, so any listed rule shows whether the ruleset reached the compiler.

1. Pick a probe from `<baseUrl>/rulesets/<level>.ci.ruleset.json`. On Strict and Complete, `AA0247` ("Use namespaces", CodeCop, default Info) is listed at `Warning`; on Essential, `AA0137` (an unused variable, default Warning) is listed at `None`.
2. On a branch, add code that breaks it: for `AA0247` a codeunit without a `namespace` line; for `AA0137` a local variable that is never used. Open a pull request.
3. Expect the log of the **Pull Request Build** to report `AA0247` as a warning for that codeunit (without the ruleset it stays at Info), or no `AA0137` at all (without the ruleset it is a warning). With `failOn` at `error` the build still passes; set `failOn` to `warning` on the branch if you want to see it fail.

Remove the probe afterwards. If the log shows the default instead, the ruleset was not applied: see section 9.

## 7. When the endpoint fails

When the compiler cannot load the endpoint (wrong URL, site down, external rulesets off, invalid JSON), it reports **AL1033** and drops the whole ruleset, your project exceptions included. On the command line `alc` does not go on with the analyzer defaults: it writes no `.app` and ends with exit code 1, also when the failing URL is the include of a local skeleton (observed in [spike (a)](https://github.com/ALCops/rulebook-engine/blob/main/docs/reference/spikes/a-hosts-and-skeleton-include.md) and [spike (c)](https://github.com/ALCops/rulebook-engine/blob/main/docs/reference/spikes/c-alc-on-ubuntu.md)). AL-Go compiles with the same compiler (through BcContainerHelper's `Compile-AppInBcContainer` or `Compile-AppWithBcCompilerFolder`, or `altool` with workspace compilation), so the compile step fails and the build fails with it. That follows from the compiler facts; it was not observed in an AL-Go run.

Should a build ever pass with AL1033 in its log, treat that as a failure by hand: AL-Go's script override `CompileAppInBcContainer.ps1` in `.AL-Go` replaces the compile call ([script overrides](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#scriptoverrides)), and a wrapper there can throw when the compiler output contains `AL1033`. Microsoft warns that script overrides are tied to `Run-AlPipeline` and will change ([Customizing AL-Go](https://github.com/microsoft/AL-Go/blob/main/Scenarios/CustomizingALGoForGitHub.md#adding-custom-scripts)), so use this only if you see the case.

## 8. Shipping the files in a custom AL-Go template

An organization that creates its AL-Go repositories from its own [custom AL-Go template](https://github.com/microsoft/AL-Go/blob/main/Scenarios/CustomizingALGoForGitHub.md#using-custom-template-repositories) can put the settings of sections 3 and 4 into that template, so every new repository starts with them. The `.rulebook/` files belong to each app (they carry the level of that app and its exceptions), so they are downloaded per app with the init script rather than copied from a template. AL-Go's update keeps project settings and only updates system files ([custom template files](https://github.com/microsoft/AL-Go/blob/main/Scenarios/CustomizingALGoForGitHub.md#using-custom-template-files)).

## 9. Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| `error AL1033: An error occurred while loading the included rule set file ...` | The endpoint could not be loaded: the URL in the skeleton is wrong, the site is down, `enableExternalRulesets` is not `true`, or the file is not valid JSON. | Open the include URL of the skeleton in a browser; set `enableExternalRulesets` to `true` (section 3); check the Publish run of your rulebook. |
| `error AL0767: The URL '...' cannot be used as the ruleset path ...` | `rulesetFile` is a URL and external rulesets are off. | Point `rulesetFile` at the local skeleton, and set `enableExternalRulesets` to `true`. |
| "Ruleset file specified in settings.rulesetFile not found at path ..." or a "cannot find path" error for the ruleset | The path does not start at the project folder (workspace compilation checks it before compiling). | Prefix the app folder: `MyApp/.rulebook/ci.ruleset.json` (section 3). |
| A rule shows at its analyzer default, not at the action the endpoint lists | `rulesetFile` is not set for that workflow, its analyzer is off (`enableCodeCop`, `enableUICop`, ALCops not installed), or the project has an exception in its `.rulebook` file. | Check the build log for the ruleset file and the analyzers; set the cop settings (section 3); look at the `rules` of the skeleton. |
| Warnings from the endpoint do not fail the build | `failOn` is `error` (the default). | Set `failOn` to `warning` or `newWarning` if that is your policy. |
| The next-major build uses the `ci` rules | The settings file name does not match the workflow name, or `rulesetFile` was added to `.AL-Go/settings.json` only. | Use `.github/Test Next Major.settings.json` with the `vnext` path (section 4). |
| The init script stops with "is not published" | The level is not in your rulebook, or the spelling differs. | Use one of the levels the message lists ([al-project.md](al-project.md) section 9). |
