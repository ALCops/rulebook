# Overrides

`overrides.json` holds your organization's own rule decisions on top of the shipped levels and stages: "AL0432 off in CI for every level", "LC0015 an error for Strict". Each entry names a diagnostic, an action and the levels and stages it applies to. The endpoints are generated from it together with the level and stage files, your `twins` setting and the quarantine files.

> **Status:** written 2026-10-09 with the change-rule work package of the engine ([WP09, #11](https://github.com/ALCops/rulebook-engine/issues/11)). The precedence below is the engine's contract in [composition.md](https://github.com/ALCops/rulebook-engine/blob/main/docs/rulebook/composition.md), the selector rules are the schema [rulebook-overrides.schema.json](https://github.com/ALCops/rulebook-engine/blob/main/schemas/rulebook-overrides.schema.json) and validation check C10. The usual way to write the file is the Change Rule workflow: [changing-a-rule.md](changing-a-rule.md).

## Contents

1. [The file](#1-the-file)
2. [Selectors](#2-selectors)
3. [Which entry wins](#3-which-entry-wins)
4. [Precedence over the other inputs](#4-precedence-over-the-other-inputs)
5. [An override at the analyzer default](#5-an-override-at-the-analyzer-default)
6. [Editing by hand](#6-editing-by-hand)

## 1. The file

The template ships it empty:

```json
{
  "$schema": "https://raw.githubusercontent.com/ALCops/rulebook-engine/v1/schemas/rulebook-overrides.schema.json",
  "rules": []
}
```

With entries, one per line, the layout the Change Rule workflow writes:

```json
{
  "$schema": "https://raw.githubusercontent.com/ALCops/rulebook-engine/v1/schemas/rulebook-overrides.schema.json",
  "rules": [
    { "id": "AL0432", "action": "None", "levels": ["*"], "stages": ["ci"], "justification": "Obsoletion backlog tracked in DEV-1234" },
    { "id": "LC0015", "action": "Error", "levels": ["strict", "complete"], "stages": ["*"] }
  ]
}
```

| Key | Value |
|---|---|
| `id` | A diagnostic id: two or three capital letters, four digits, an optional `i` (`AL0432`, `PTE0011`, `LC0089i`). |
| `action` | `Error`, `Warning`, `Info`, `Hidden` or `None`. `Default` is not an action. |
| `levels`, `stages` | Selectors, [section 2](#2-selectors). Both are required. |
| `justification` | Optional free text: why your organization made this decision. It stays in this file and in the Change Rule pull request; the endpoints do not carry it. |

The file is org-owned: the update never touches it ([updating.md](updating.md) section 1).

## 2. Selectors

`levels` and `stages` are arrays:

- `["*"]`: every level (or stage) in your settings, including the ones you add later.
- A list of slugs: `["strict"]`, `["strict", "complete"]`. A slug is the lowercased `name` from the settings: the level `Strict` is `strict`, the stage `vNext` is `vnext`, the stage `default` is `default`.
- `*` stands alone: `["*", "strict"]` is rejected. A plain string (`"strict"` without brackets) is rejected too.

A slug that is not in your settings fails validation (C10, "entry `<n>` names unknown level '`<slug>`'; use a slug from the settings or ["*"]"). Removing a level or stage from the settings therefore means removing it from the overrides that name it.

## 3. Which entry wins

Several entries may match the same diagnostic in the same endpoint. The **most specific** one wins: an entry that names a level and a stage beats one with a `*`, which beats one with `*` for both. On a tie, the **later entry in the file** wins.

| Entries for AA0001 | `strict.ci` gets |
|---|---|
| `None` for `*` / `*` | `None` |
| `None` for `*` / `*`, then `Warning` for `strict` / `ci` | `Warning`: two named selectors beat none |
| `Warning` for `strict` / `*`, then `None` for `*` / `ci` | `None`: both name one selector, the later entry wins |

The Change Rule workflow keeps one entry per selection: running it for a selection that has an entry replaces that entry in its place, and a new selection is added at the end.

## 4. Precedence over the other inputs

For each diagnostic in each endpoint, the first of these that has something to say decides ([D19](https://github.com/ALCops/rulebook-engine/blob/main/docs/adr/0019-organization-overrides-and-quarantine-are-generator-inputs.md), [D41](https://github.com/ALCops/rulebook-engine/blob/main/docs/adr/0041-quarantine-wins-over-a-stage-entry-for-an-unmentioned-id.md)):

1. **An override** that matches the diagnostic, level and stage.
2. **The twins setting**: the losing side of a pair of rules that check the same thing is `None` ([pte-or-appsource.md](pte-or-appsource.md)).
3. **The stage file**, but only where the level result is not `None`: a stage never switches a rule on.
4. **The level chain**: the level's own file and the files it is based on.
5. **Quarantine**: `None` for a new diagnostic the stage holds back ([quarantine.md](quarantine.md)).
6. **The analyzer default.**

An override therefore beats everything: it can switch on a rule a level turns off, a rule the twins setting lowers, and a rule only quarantine mentions. It does not remove that quarantine entry; delete it in the same change when the override is your decision ([quarantine.md](quarantine.md) section 4).

## 5. An override at the analyzer default

An endpoint lists only the diagnostics whose action differs from the analyzer default. An override whose action is the default is valid, and its effect is that the diagnostic disappears from the endpoint: the compiler then applies the default by itself, and a project can switch it off with `suppressWarnings` in `app.json` ([al-project.md](al-project.md) section 6). The Change Rule pull request says so per endpoint: "now unlisted in recommended.ci: Warning equals the analyzer default".

An entry that changes nothing at all, because the levels or a more specific entry already give that action everywhere it applies, is not written by the workflow ([changing-a-rule.md](changing-a-rule.md) section 4).

## 6. Editing by hand

You can edit the file directly, for example to give one entry two levels, or to clear a justification. Then:

- **Regenerate the endpoints in the same pull request.** A changed `overrides.json` leaves `rulesets/` out of date; Validate fails with C12 and Publish refuses to deploy. Regenerate locally with the engine module, or merge and run Update Rulebook System Files with "Resolve the latest commit" off, as described in [quarantine.md](quarantine.md) section 4.
- **Keep the layout if you like it.** The Change Rule workflow reads comments and trailing commas, and the next run it makes rewrites the file in its one-entry-per-line layout.
- **Validate checks the file**: the schema, the actions and the selectors (C10).

The Change Rule workflow does all three for one entry, so a hand edit is only needed for what the form does not offer.
