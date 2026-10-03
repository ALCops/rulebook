# Rulebook

Rulebook is a GitHub template plus automation for managing the ruleset files that drive code analysis in Microsoft Dynamics 365 Business Central AL projects. One repository per organization holds the rules for every analyzer (CodeCop, UICop, AppSourceCop, PerTenantExtensionCop and the ALCops cops), publishes them as URLs, and keeps them current with workflows. AL projects point at those URLs from VS Code, AL-Go for GitHub, Azure DevOps or any other pipeline that runs the AL compiler.

> **Status:** design phase. This repository holds the user-facing documentation only; the template content (rulesets, workflows, skeletons) arrives with the workpackages listed in [ALCops/rulebook-engine](https://github.com/ALCops/rulebook-engine). Nothing here is usable yet.

---

## Contents

1. [What Rulebook does](#1-what-rulebook-does)
2. [How it fits together](#2-how-it-fits-together)
3. [Getting started (planned)](#3-getting-started-planned)
4. [Consumer paths](#4-consumer-paths)
5. [Documentation](#5-documentation)
6. [Related repositories](#6-related-repositories)

---

## 1. What Rulebook does

| Need | Rulebook answer |
|---|---|
| Decide once which diagnostic is shown at which severity, for the whole organization | One **org rulebook repo** created from this template. |
| Different severities for development, pull-request builds and next-major builds | Configurable **stages**. Shipped: `default` (the editor and any consumer without its own stage), `CI` (pull request and release builds), `vNext` (builds against the next platform). Add or remove stages in the settings. |
| Per-tenant extension or AppSource app? | Rulebook does not ask. Both Microsoft cops run at their native severity; a project opts out of the rules written for the other kind with `suppressWarnings` in `app.json`, with its own ruleset file, or by disabling a cop. See [docs/pte-or-appsource.md](docs/pte-or-appsource.md). |
| Start strict or start lenient, and grow | An ordered ladder of **levels**. Shipped: Essential, Recommended, Strict and Complete, each defined as the level below plus the rules it changes. Insert your own level, alias one under another name, or stop publishing one, all in the settings. |
| One URL for `al.ruleSetPath`, the AL-Go `rulesetFile`, or a small local file | One **endpoint** per level and stage: a single flat ruleset file that lists only the diagnostics whose severity differs from the analyzer default, so a compile fetches one small file. Published by a workflow. |
| New analyzer rules must not break a green pipeline | A daily **scan** of the compiler and ALCops packages quarantines new diagnostic ids per stage, reports changed default severities, and regenerates the endpoints. |
| Change a rule without editing dozens of files | A **workflow form** that writes one entry to `overrides.json`, regenerates the endpoints and opens a pull request. |
| Keep the automation up to date without losing your own decisions | An **update workflow** modelled on AL-Go's "Update AL-Go System Files". |

## 2. How it fits together

```mermaid
flowchart LR
    tpl[ALCops/rulebook<br/>this template] -->|Use this template| org[org rulebook repo<br/>base, overrides.json, quarantine,<br/>generated rulesets, workflows]
    eng[ALCops/rulebook-engine<br/>actions, scripts] -.->|referenced by workflows| org
    org -->|Publish workflow: generate + verify| url[(endpoint URLs<br/>GitHub Pages by default)]
    url -->|one fetch| proj[AL project repo<br/>small local ruleset file]
    proj --> vscode[VS Code]
    proj --> algo[AL-Go for GitHub]
    proj --> ado[Azure DevOps / other pipelines]
```

An endpoint URL has the form `<baseUrl>/rulesets/<level>.<stage>.ruleset.json`, or `<baseUrl>/rulesets/<level>.ruleset.json` for the `default` stage. Level and stage names are lowercased in the URL: `https://contoso.github.io/rulebook/rulesets/strict.ci.ruleset.json`, `https://contoso.github.io/rulebook/rulesets/strict.ruleset.json`. Every endpoint is generated from the level's chain of files, the stage file, your `twins` setting, `overrides.json` and quarantine files. It lists the diagnostics whose action differs from the analyzer default and has no includes.

## 3. Getting started (planned)

1. Press **Use this template** and create your org rulebook repo.
2. Set the publish target, the quarantine policy and, if every project you build is of one kind, the `twins` setting in `.github/Rulebook-Settings.json`. The same file lists the levels and stages; add, alias or remove entries if the shipped set does not fit.
3. Run the **Publish** workflow. Your endpoints are live.
4. Copy the skeleton files for your level from `skeletons/` into an AL project as `.rulebook/<stage>.ruleset.json` and point `al.ruleSetPath` (VS Code) or `rulesetFile` (AL-Go) at the file for that stage. A skeleton includes one endpoint URL and nothing else.
5. Opt out of the rules written for the other kind of extension, as described in [docs/pte-or-appsource.md](docs/pte-or-appsource.md). Add project-specific exceptions to the `rules` array of the local file; organization-wide changes go through the **Change rule** workflow into `overrides.json`.

The detailed walkthroughs are listed in [docs/README.md](docs/README.md).

## 4. Consumer paths

| Consumer | Setting | Typical stage | External rulesets |
|---|---|---|---|
| VS Code AL extension | `al.ruleSetPath` (local file or URL) | `default` | `al.enableExternalRulesets`, default true |
| AL-Go for GitHub | `rulesetFile` in `.AL-Go/settings.json`, per workflow via `.github/<Workflow>.settings.json` | `ci`; `vnext` in `.github/NextMajor.settings.json` | `enableExternalRulesets` |
| BcContainerHelper / custom PowerShell | `-rulesetFile` | `ci` | `-enableExternalRulesets` |
| ALOps (Azure DevOps) | ruleset input of the compile task | `ci` | corresponding input |
| `alc` directly | `/ruleset:<path>` | any | `/enableexternalrulesets` (default false) |

## 5. Documentation

User documentation lives in [docs/](docs/README.md) in this repository. Architecture, decisions, references and the workpackages live in [rulebook-engine/docs](https://github.com/ALCops/rulebook-engine/tree/main/docs). When version 1.0 ships, the user documentation moves to [alcops.dev](https://alcops.dev).

**Issues and contributions.** Issues are disabled on this repository. Report problems and ideas in [ALCops/rulebook-engine](https://github.com/ALCops/rulebook-engine/issues); status and order of the work are on the [Rulebook v1 board](https://github.com/orgs/ALCops/projects/1). Organizations that create their own repository from this template handle issues there as they see fit.

## 6. Related repositories

- [ALCops/rulebook-engine](https://github.com/ALCops/rulebook-engine): the composite actions, PowerShell modules, tests and contributor documentation that the workflows in this template call.
- [ALCops/Analyzers](https://github.com/ALCops/Analyzers): the ALCops analyzers whose rules Rulebook manages next to Microsoft's cops.
- [microsoft/AL-Go](https://github.com/microsoft/AL-Go): the CI/CD system whose template and update mechanics Rulebook reuses.

## License

MIT, see [LICENSE](LICENSE).
