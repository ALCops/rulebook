# Rulebook user documentation

Index of the user-facing documentation. The pages marked *planned* do not exist yet and are written in work package WP11 of rulebook-engine ([#13](https://github.com/ALCops/rulebook-engine/issues/13)).

| Page | Intent | Status |
|---|---|---|
| `getting-started.md` | From "Use this template" to a live endpoint in ten minutes: settings, first publish, first AL project. | planned |
| `concepts.md` | Levels (an ordered ladder; each level is the level it is based on plus the rules it changes), stages (one delta file each; `default` always exists and has no suffix in the URL), endpoints (sparse: only deviations from the analyzer defaults), overrides, quarantine, skeletons, the twins setting. Both levels and stages are configurable. One page to read before anything else. | planned |
| [`pte-or-appsource.md`](pte-or-appsource.md) | Why both Microsoft cops always run, which rules contradict between per-tenant extensions and AppSource apps, and the three ways to opt out: disable a cop, `suppressWarnings` in `app.json`, a project ruleset file. Ready-made lists for both kinds of project. | written |
| [`hosting.md`](hosting.md) | Where your endpoints are served from: GitHub Pages setup, `baseUrl`, custom domains, how fast changes arrive, what happens when the repository goes private, Publish error messages. The other targets (public dist repo, Azure Blob Storage, Gist) are planned. | written (GitHub Pages) |
| `al-go.md` | Plug-and-play walkthrough for AL-Go for GitHub: `rulesetFile`, `enableExternalRulesets`, per-workflow settings for CI/CD and NextMajor, the GHTOKENWORKFLOW secret. | planned |
| `azure-devops.md` | ALOps compile task inputs, BcContainerHelper parameters, custom `alc` calls. | planned |
| `vscode.md` | `al.ruleSetPath`, when VS Code re-fetches a remote ruleset, project exceptions in the local file. | planned |
| `levels.md` | What each shipped level (Essential, Recommended, Strict, Complete) contains, how to pick one, how to add or insert your own, how to publish a level under another name, how to stop publishing one. The same for stages (`default`, `CI`, `vNext`). | planned |
| `changing-a-rule.md` | The Change rule workflow: opt in, opt out, raise, lower, with justification, as a pull request that regenerates the endpoints. | planned |
| `overrides.md` | The `overrides.json` file: selectors for levels and stages (lowercase names or `*`), precedence over the twins setting, the level and stage files and the quarantine. | planned |
| `new-rules.md` | The daily scan, the quarantine policy per stage (one quarantine file per stage in your settings), prerelease packages and vNext, adopting a quarantined rule, changed default severities. | planned |
| `updating.md` | The Update Rulebook System Files workflow: what is overwritten, what is preserved, scheduling, pinning a version. | planned |
| `migration.md` | Importing an existing ruleset tree (blob-hosted layouts, StefanMaron/RulesetFiles) into Rulebook. | planned |
| `troubleshooting.md` | AL1033 and AL0767, unreachable endpoints, token failures, validation errors. | planned |

Contributor documentation (architecture, decisions, compiler internals, AL-Go mechanics) is in [rulebook-engine/docs](https://github.com/ALCops/rulebook-engine/tree/main/docs); the work packages are issues on the [Rulebook v1 board](https://github.com/orgs/ALCops/projects/1).
