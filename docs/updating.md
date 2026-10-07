# Updating your rulebook

New versions of the template bring new level content, stage changes, workflow fixes and new files. The **Update Rulebook System Files** workflow pulls such a version into your rulebook repository as a pull request. That pull request also contains the endpoints regenerated under your own overrides and quarantine, and your own decisions are kept.

> **Status:** written 2026-10-07 with the update work package of the engine ([WP07, #9](https://github.com/ALCops/rulebook-engine/issues/9)). The contributor reference, with the file classes in detail and the live test run, is the engine's [update-mechanics.md](https://github.com/ALCops/rulebook-engine/blob/main/docs/reference/update-mechanics.md). The update needs a write token; setting it up is in [ghtokenworkflow.md](ghtokenworkflow.md).

## Contents

1. [What an update changes](#1-what-an-update-changes)
2. [Running the update](#2-running-the-update)
3. [How you hear about an update](#3-how-you-hear-about-an-update)
4. [The pull request](#4-the-pull-request)
5. [Removing a shipped level, stage or file](#5-removing-a-shipped-level-stage-or-file)
6. [Running the update on a schedule](#6-running-the-update-on-a-schedule)
7. [Direct commit](#7-direct-commit)
8. [Troubleshooting](#8-troubleshooting)

## 1. What an update changes

The update compares what the template ships with what your repository holds, file by file. Only files the template ships are ever compared, written or removed: a file that exists only in your repository is never touched, whatever its name.

| Your files | What the update does |
|---|---|
| Workflows under `.github/workflows/` that the template ships, the shipped level files in `base/` (`essential`, `recommended`, `strict`, `complete`), `base/twins.json`, the shipped stage files in `stages/` (`ci.json`, `vnext.json`), `skeletons/README.md`, and `.github/RELEASENOTES.copy.md` and the issue form under `.github/ISSUE_TEMPLATE/` when the template ships them (planned) | **Replaced** by the template version. A hand edit is reverted, and the pull request shows the revert. Change rules in `overrides.json` instead (or add your own level or stage file). |
| `.github/Rulebook-Settings.json` | **Kept**. Only the values of `templateUrl`, `templateSha` and `$schema` change; everything else keeps its text and formatting, except that line endings become LF. `templateUrl` must be there (without it the update fails with "No templateUrl in .github/Rulebook-Settings.json ..."); `templateSha` is inserted when missing; `$schema` is inserted only when the template's settings carry one. |
| `rulesets/` (the endpoints) and `skeletons/*.ruleset.json` | **Regenerated** from the new level and stage files and your settings, `overrides.json`, quarantine files and catalog. A level or stage you added gets its endpoints and skeletons in the same pull request. |
| `overrides.json`, `quarantine.*.json`, `catalog/`, `README.md`, level and stage files you added, your own workflows, `docs/`, `site/data/**` | **Never touched.** |
| `site/**` (the dashboard; planned, the template does not ship it yet) | **Overwritten only when you have not changed the file.** A file you changed is kept; when the template changed it too, it is listed under "Skipped: local changes in site/" in the pull request so you can take over what you need. A site file you deleted comes back while the template ships it (list it in `unusedRulebookFiles` to keep it away). This is the shipped `"updateMode": "skip"`; any value other than `"overwrite"` behaves the same. With `"site": { "updateMode": "overwrite" }` site files are replaced like system files. |

In the workflows, the update also fills in the current template URL (the dispatch form shows "current is https://github.com/ALCops/rulebook@main"). When the template ships the Change rule workflow (planned), the update also writes its `levels` and `stages` choice lists from your settings, so a level you add appears in the form after the next update.

Before anything is pushed, the update runs the full validation on the updated repository. If the result has an error, the run fails with the findings and nothing is pushed. The findings come from the repository as it would be after the update, so an error your repository already has stops the update too. Fix it first. Warnings go into the pull request.

## 2. Running the update

Actions > **Update Rulebook System Files** > Run workflow. The form has three inputs:

| Input | Default | Use |
|---|---|---|
| Template repository URL | empty: the `templateUrl` of your settings, `https://github.com/ALCops/rulebook@main` | Another template or another branch, as `owner/repo@branch` or a full URL. The run then records the new URL in your settings, and later runs follow it. |
| Resolve the latest commit | on | Off re-applies the template commit recorded in `templateSha`, for example to regenerate after you changed your settings. |
| Push to this branch instead of opening a pull request | off | See [section 7](#7-direct-commit). |

The first run after you created the repository from the template records the template commit in `templateSha` and fills in the template URL in the update workflow. Those two files are the whole pull request. After you merge it, a second run says "No updates available".

The run needs the `GHTOKENWORKFLOW` secret ([ghtokenworkflow.md](ghtokenworkflow.md)). The workflow's own token stays read-only, because it can never change workflow files.

**Pinning a version.** `templateUrl` names a branch. While it is `@main` you follow the latest template. When the template publishes version branches (planned with the release work package of the engine), set `templateUrl` to such a branch, for example `https://github.com/ALCops/rulebook@v1`, to take only the fixes of that version.

## 3. How you hear about an update

The Validate workflow, which runs on every pull request and on every push to `main`, runs the same comparison without writing anything, because its Validate step keeps `checkForUpdates` at its default `'true'`. It annotates the run:

| Annotation | Meaning |
|---|---|
| notice "No updates available" | Your system files match the template. |
| notice "template commit `<sha7>` not recorded; run Update Rulebook System Files once" | Nothing would change but `templateSha` (and, the first time, the template URL in the update workflow): a repository fresh from the template, or a template that moved to a commit that changes none of your managed files (a documentation-only commit, for example). Running the update once records the commit, and the notice goes away. |
| warning "Updates available: run the Update Rulebook System Files workflow (`<n>` files)" | The template moved, or a system file was edited by hand. The job summary lists the files under "Template update check". |
| warning "update check skipped: `<reason>`" | The check could not run: the template could not be read, the updated rulebook would not validate, or something else failed. |

None of these annotations fails Validate or counts as a warning, also with `failOnWarning`. The check reads the template with the workflow token only, so it is skipped for a private template, which that token cannot read; the update itself still works with the secret. A public template is read fine (verified on a normal pull request; a pull request from a fork was not tried, and its token can still read public repositories). Other skip reasons, such as an exhausted API rate limit, carry their own text after "update check skipped:".

There is no lasting switch to turn the check off yet. `checkForUpdates: 'false'` on the Validate step of `Validate.yaml` works until the next update, which replaces the workflow with the template version and turns the check back on; a settings key for it is planned ([ALCops/rulebook-engine#67](https://github.com/ALCops/rulebook-engine/issues/67)). Since the notice or warning never fails Validate, the practical answer until then is to leave the check on.

## 4. The pull request

The title says which commit of your branch and which template commit it is about:

```
[main@02a7339] Update Rulebook System Files from ALCops/rulebook - fe725f5
```

The branch is `update-rulebook-system-files/<branch>/<yyMMddHHmmss>`, and the pull request gets the labels in `commitOptions.pullRequestLabels` (shipped: `rulebook`; GitHub creates a label that does not exist yet). The body has these sections, in this order:

1. **Changes**: every file with its class (overwrite, settings, generated, customizable) and created, modified or deleted.
2. **Effective diff**: one table per endpoint with the rules whose action changes, before and after, and the file that decides (for example `level:recommended` or `override`). This is what changes for your AL projects.
3. **Skipped: local changes in site/**, when site files were kept.
4. **Notes**, for example a file the template no longer ships.
5. **Validation warnings**, when the updated repository has any.
6. **Release notes**, when the template ships `.github/RELEASENOTES.copy.md` (planned): the part that is newer than your copy, or "No release notes available". Without that file the section is left out.

A very long body is shortened below GitHub's limit: the release notes go first, then endpoint tables. The job summary of the run has the full lists. The Validate workflow runs on the pull request like on any other.

**Open update pull requests.** A second run while the update pull request is open warns "Pull request already exists: `<url>`" and creates nothing, as long as the title is the same. The guard compares titles, and the title contains both the commit of your branch and the template commit. So the next run opens a second update pull request when something was merged into your branch, or when the template moved, while the first one was open (with a weekly schedule, a template that moves every week does this). Merge the newer one and close the older.

## 5. Removing a shipped level, stage or file

Removing a level or stage from `levels` or `stages` in the settings stops publishing it. Its file stays, and the next update would bring a changed version back. To stop the update from writing a shipped file, and to have it deleted, list the file in `unusedRulebookFiles` by its path in the repository, with `/`:

```json
"unusedRulebookFiles": ["stages/vnext.json", "base/complete.ruleset.json"]
```

A bare file name (`vnext.json`) is rejected by the settings check. Validate reads the same list: a file in `base/` or `stages/` that no settings entry uses is a warning (C9) unless it is listed there. Do not list a file the remaining levels still need: removing `base/essential.ruleset.json` while `Recommended` is based on `Essential` makes the update fail its validation.

When a new template version no longer ships a file you have, the update leaves it in place and adds the note "The template no longer ships `<path>`; list it in unusedRulebookFiles to remove it" to the pull request. When the list names a file the template does not ship any more and you no longer have, the note says the entry can be removed.

## 6. Running the update on a schedule

Set a five-field cron expression in the settings:

```json
"update": { "schedule": "0 6 * * 1" }
```

Run the update once. It adds a `schedule:` trigger with that cron to `UpdateRulebookSystemFiles.yaml` in its pull request; set `"schedule": null` and run it again to remove the trigger (both verified in the live run). After the merge, GitHub runs the workflow on that schedule from the default branch only (code-derived: a scheduled run was not observed in the live run). A run by hand uses the branch you choose under "Use workflow from". A scheduled run always resolves the latest template commit, and opens a pull request unless `commitOptions.createPullRequest` is `false`.

## 7. Direct commit

With "Push to this branch instead of opening a pull request" on, the update commits to the branch directly. The commit message is the usual title, and there is no duplicate check. When branch protection or a ruleset refuses the push, the run warns "The direct push to main was refused; creating a pull request instead" and opens the pull request instead.

## 8. Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| "The GHTOKENWORKFLOW secret is needed to update system files. Read https://github.com/ALCops/rulebook/blob/main/docs/ghtokenworkflow.md" (with `ghTokenWorkflowSecretName` set, the message names that secret) | The secret does not exist, or this repository cannot see it (an organization secret limited to other repositories, or a secret with another name). | Create it ([ghtokenworkflow.md](ghtokenworkflow.md)). With another name, set `ghTokenWorkflowSecretName` in the settings. |
| "The GHTOKENWORKFLOW secret could not be used: ..." | The App is not installed on this repository, the App JSON is incomplete, or the token request was refused. | Install the App on the repository, rebuild the secret with the one-liner, check the App's permissions. |
| "Failed to update the Rulebook system files. Make sure that the token in the secret GHTOKENWORKFLOW is not expired and may write contents, pull requests and workflows of `<repository>` ..." | Cloning or pushing failed: an expired personal access token, or a token without the Workflows permission. | Renew the token or add the permission. |
| "Failed to create the pull request for the Rulebook system files ..." | The pull request step failed. When a branch was already pushed, the error starts with "Branch `<name>` was pushed." and carries the link to open the pull request by hand: only opening it was refused (permissions, or an organization policy). Without that sentence nothing was pushed: the token could not read the branch or the open pull requests before cloning. | With the link, open the pull request by hand; in both cases check the token's Pull requests permission and that it can read the repository. |
| "Pull request already exists: `<url>`" | An update pull request for the same commits is open. | Review and merge or close it. |
| "The updated rulebook would not validate: ..." (with "failure validation") | The repository after the update has an error, often one it already had. | Fix the reported file (run Validate on a pull request to see the same findings), then run the update again. |
| "no .github/workflows in the template" | `templateUrl` points at a repository that is not a Rulebook template. | Correct `templateUrl` or the dispatch input. |
| "Could not get the latest commit of https://github.com/`<owner>`/`<repo>`@`<branch>` (HTTP 404: ...)" | The template or branch does not exist, or it is private and the token cannot read it. | Check the URL. For a private template, install the GitHub App on the template repository too ([ghtokenworkflow.md](ghtokenworkflow.md)). |
| A file you edited comes back | It is a system file; the update replaces it. | Put the change in `overrides.json` or your own file, or list the file in `unusedRulebookFiles`. |
