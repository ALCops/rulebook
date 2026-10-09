# Rulebook user documentation

Index of the user-facing documentation. The pages marked *planned* do not exist yet and are written in work package WP11 of rulebook-engine ([#13](https://github.com/ALCops/rulebook-engine/issues/13)).

| Page | Intent | Status |
|---|---|---|
| `getting-started.md` | From "Use this template" to a live endpoint in ten minutes: settings, first publish, first AL project. | planned |
| `concepts.md` | Levels (an ordered ladder; each level is the level it is based on plus the rules it changes), stages (one delta file each; `default` always exists and has no suffix in the URL), endpoints (sparse: only deviations from the analyzer defaults), overrides, quarantine, skeletons, the twins setting. Both levels and stages are configurable. One page to read before anything else. | planned |
| [`pte-or-appsource.md`](pte-or-appsource.md) | Why both Microsoft cops always run, which rules contradict between per-tenant extensions and AppSource apps, and the three ways to opt out: disable a cop, `suppressWarnings` in `app.json`, a project ruleset file. Ready-made lists for both kinds of project. | written |
| [`hosting.md`](hosting.md) | Where your endpoints are served from: GitHub Pages setup, `baseUrl`, custom domains, how fast changes arrive, what happens when the repository goes private, Publish error messages. The other targets (public dist repo, Azure Blob Storage, Gist) are planned. | written (GitHub Pages) |
| [`al-project.md`](al-project.md) | Rulebook in an AL project, the facts: the `.rulebook/` folder with one file per stage and why, the init script, the settings per consumer, project exceptions, exactly when `suppressWarnings` works, AL1033 and refreshing VS Code. | written |
| `al-go.md` | Walkthrough for AL-Go building on `al-project.md`: `rulesetFile` per workflow, NextMajor, shipping `.rulebook/` in a custom AL-Go template. The `GHTOKENWORKFLOW` secret both share is in [`ghtokenworkflow.md`](ghtokenworkflow.md). | planned |
| `azure-devops.md` | ALOps compile task inputs, BcContainerHelper parameters, custom `alc` calls. | planned |
| `vscode.md` | Walkthrough for VS Code building on `al-project.md`: workspace settings, the Problems pane, when the editor re-fetches, troubleshooting. | planned |
| `levels.md` | What each shipped level (Essential, Recommended, Strict, Complete) contains, how to pick one, how to add or insert your own, how to publish a level under another name, how to stop publishing one. The same for stages (`default`, `CI`, `vNext`). | planned |
| [`changing-a-rule.md`](changing-a-rule.md) | The Change Rule workflow: the form, reading its pull request (before and after per endpoint, with the input that decides), when nothing is written, pull request or direct commit, your own levels and stages in the form, troubleshooting. | written |
| [`overrides.md`](overrides.md) | The `overrides.json` file: entries and selectors (slugs or `*`), which entry wins, precedence over the twins setting, the level and stage files and the quarantine, an override at the analyzer default, editing by hand. | written |
| [`quarantine.md`](quarantine.md) | New diagnostics and quarantine: the daily scan, choosing the quarantine policy, reading its pull request, adopting a quarantined rule, changed default severities, the catalog and its flags, running it by hand, troubleshooting. | written |
| [`updating.md`](updating.md) | The Update Rulebook System Files workflow: what is replaced, kept, regenerated or never touched, the site files, running it and pinning a version, the check in Validate, the pull request, removing a shipped level or stage with `unusedRulebookFiles`, the schedule, direct commit, troubleshooting. | written |
| [`ghtokenworkflow.md`](ghtokenworkflow.md) | The `GHTOKENWORKFLOW` secret the update writes with: why the workflow token is not enough, a GitHub App step by step with the one-liner for the secret, a personal access token, organization secrets and `ghTokenWorkflowSecretName`, a private template, failure messages. | written |
| `migration.md` | Importing an existing ruleset tree (blob-hosted layouts, StefanMaron/RulesetFiles) into Rulebook. | planned |
| `troubleshooting.md` | AL1033 and AL0767, unreachable endpoints, validation errors. Token failures are in [`ghtokenworkflow.md`](ghtokenworkflow.md) section 6, update failures in [`updating.md`](updating.md) section 8. | planned |

Contributor documentation (architecture, decisions, compiler internals, AL-Go mechanics) is in [rulebook-engine/docs](https://github.com/ALCops/rulebook-engine/tree/main/docs); the work packages are issues on the [Rulebook v1 board](https://github.com/orgs/ALCops/projects/1).
