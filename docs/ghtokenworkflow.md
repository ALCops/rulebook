# The GHTOKENWORKFLOW secret

The update workflow writes to your rulebook repository: it changes files under `.github/workflows/`, pushes a branch and opens a pull request. It does that with a token from the secret `GHTOKENWORKFLOW`, the same secret and value format AL-Go for GitHub uses. An organization that runs AL-Go can reuse its secret and GitHub App.

> **Status:** written 2026-10-07 with the update work package of the engine ([WP07, #9](https://github.com/ALCops/rulebook-engine/issues/9)); the decision is [D44](https://github.com/ALCops/rulebook-engine/blob/main/docs/adr/0044-the-write-token-secret-is-ghtokenworkflow-in-al-go-format.md). How the update uses the token: [updating.md](updating.md) and the engine's [update-mechanics.md](https://github.com/ALCops/rulebook-engine/blob/main/docs/reference/update-mechanics.md) section 6.

## Contents

1. [Why the workflow token is not enough](#1-why-the-workflow-token-is-not-enough)
2. [A GitHub App (recommended)](#2-a-github-app-recommended)
3. [A personal access token](#3-a-personal-access-token)
4. [Where to put the secret](#4-where-to-put-the-secret)
5. [A private template](#5-a-private-template)
6. [When it fails](#6-when-it-fails)

## 1. Why the workflow token is not enough

- The `GITHUB_TOKEN` of a run can never get the Workflows permission, so a push that changes `.github/workflows/` is refused. The update rewrites workflow files.
- A pull request opened with `GITHUB_TOKEN` starts no workflow, so Validate would not run on the update.
- Organizations can forbid GitHub Actions to create pull requests at all.

The update workflow therefore keeps `GITHUB_TOKEN` read-only (`contents: read`, `actions: read`) and writes only with the token from the secret.

## 2. A GitHub App (recommended)

A GitHub App gives the update a short-lived token for one repository at a time. Commits and pull requests show the App as their author, tagged **bot**. The private key never leaves the runner.

1. **Register the App.** For an organization: `https://github.com/organizations/<org>/settings/apps/new`. For a personal account: <https://github.com/settings/apps/new>.
   - Name: anything unique, for example `contoso-rulebook`. Homepage URL: any URL, for example your rulebook repository.
   - Leave the callback URL and the setup URL empty. Under **Webhook**, untick **Active**.
   - **Repository permissions:** Contents **Read and write**, Pull requests **Read and write**, Workflows **Read and write**, Actions **Read-only**. Metadata becomes Read-only by itself. No organization or account permissions.
   - Where it can be installed: only on this account.
   - Create the App and note its **Client ID** (it starts with `Iv`).
2. **Generate a private key**: on the App's page, "Private keys" > "Generate a private key". A `.pem` file downloads.
3. **Install the App**: "Install App" > your organization or account > "Only select repositories" > your rulebook repository (and the template repository if it is private, [section 5](#5-a-private-template)).
4. **Build the secret value.** The value is compressed JSON (one line) with the Client ID and the key. In PowerShell 7:

   ```powershell
   $githubAppClientId = 'Iv23li...'
   $privateKeyFile = 'C:\Users\you\Downloads\contoso-rulebook.2026-10-07.private-key.pem'
   @{ "GitHubAppClientId" = $githubAppClientId; "PrivateKey" = ([string]::Join('', [System.IO.File]::ReadAllLines($privateKeyFile))) } | ConvertTo-Json -Compress -Depth 99 | Set-Clipboard
   ```

   Paste the clipboard as the secret value ([section 4](#4-where-to-put-the-secret)). Or pipe the JSON straight into the GitHub CLI instead of `Set-Clipboard`: `... | gh secret set GHTOKENWORKFLOW -R <owner>/<repository>`.

   The value must be one line. GitHub masks each line of a multi-line secret separately, which would print parts of the JSON in the logs.

On every run the update exchanges the App JSON for an installation token. The token is valid for one hour, limited to the rulebook repository, and has exactly the permissions above. It is masked in the log before anything else runs.

## 3. A personal access token

A personal access token is used as it is, without an exchange. Commits and pull requests show its owner as the author.

- **Fine-grained token** (better): resource owner your organization, only the rulebook repository, repository permissions Contents, Pull requests and Workflows **Read and write**, Actions **Read-only** (Metadata comes by itself).
- **Classic token**: the `workflow` scope (which includes `repo`). It reaches every repository its owner can reach.

Give it a short expiration date and renew it before it runs out: an expired token fails the update (section 6). Store the token itself as the secret value.

## 4. Where to put the secret

- **Repository secret**: in the rulebook repository, Settings > Secrets and variables > Actions > New repository secret, name `GHTOKENWORKFLOW`. Or `gh secret set GHTOKENWORKFLOW -R <owner>/<repository>`.
- **Organization secret**: Organization settings > Secrets and variables > Actions, named `GHTOKENWORKFLOW`, with the rulebook repository in its repository access. An organization that already has the AL-Go secret only adds the rulebook repository to it, and installs the App on the repository.

A secret that is visible to the repository does not by itself give access: the App must also be installed on the repository, or the token must be allowed to reach it.

**Another name.** The workflow reads the secret name from the settings. To use a different secret, for example one kept apart from AL-Go:

```json
"ghTokenWorkflowSecretName": "RULEBOOK_TOKEN"
```

The name may contain letters, digits and underscores and must not start with a digit.

## 5. A private template

The update downloads the template with the workflow token first. A public template, such as `ALCops/rulebook`, needs nothing more. When the template is private (an internal copy of the template, for example), the download is refused, and the update retries with the token from the secret:

- **GitHub App**: install the App on the template repository too. The update asks for a read-only token on it.
- **Personal access token**: the token must be able to read the template repository.

The Validate check reads with the workflow token only, so for a private template it reports "update check skipped". Updating still works.

## 6. When it fails

| Message | Cause | Fix |
|---|---|---|
| "The GHTOKENWORKFLOW secret is needed to update system files. Read https://github.com/ALCops/rulebook/blob/main/docs/ghtokenworkflow.md" | No secret of that name is visible to the repository. | Create it (section 4). With another name, set `ghTokenWorkflowSecretName`. |
| "The GHTOKENWORKFLOW secret could not be used: The GitHub App `<client id>` has no installation on `<repository>` ..." | The App is not installed on the rulebook repository. | Install it (section 2, step 3). |
| "... The GitHub App JSON in the token secret needs GitHubAppClientId and PrivateKey" or "... is not a PEM key" | The secret value is not the JSON of step 4. | Build it again with the one-liner. |
| "... could not get an installation token for `<repository>` (HTTP 422: ...)" | The App lacks one of the permissions in step 1. | Add the permission, then accept the new permissions on the installation. |
| "Failed to update the Rulebook system files. Make sure that the token in the secret GHTOKENWORKFLOW is not expired and may write contents, pull requests and workflows ..." | Pushing failed: an expired personal access token, or no Workflows permission. | Renew the token or add the permission. |
| "Failed to create the pull request ... open the pull request by hand: `<link>`" | The branch was pushed, but the pull request was refused. | Open it from the link; give the token Pull requests write. |
| "update check skipped: Could not get the latest commit of ..." in Validate | A private template, which the workflow token cannot read. | Expected; the update itself uses the secret (section 5). |
