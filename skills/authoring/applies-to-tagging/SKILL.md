---
name: docs-applies-to-tagging
version: 1.2.0
description: Validate and generate applies_to tags in Elastic documentation, including for cumulative docs across versions and deployment types. Use when writing new docs pages, reviewing existing pages for correct applies_to usage, deciding whether to preserve or replace existing version-scoped content, or when content changes lifecycle state (preview, beta, GA, deprecated, removed).
argument-hint: <file-or-directory-or-intent>
context: fork
allowed-tools: Read, Grep, Glob, Edit, CallMcpTool, WebFetch
sources:
  - https://elastic.github.io/docs-builder/syntax/applies/
  - https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/reference
  - https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/guidelines
  - https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/badge-placement
  - https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/example-scenarios
---
<!-- Copyright Elasticsearch B.V. and/or licensed to Elasticsearch B.V. under one
or more contributor license agreements. See the NOTICE file distributed with
this work for additional information regarding copyright
ownership. Elasticsearch B.V. licenses this file to you under
the Apache License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License.
You may obtain a copy of the License at

	http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License. -->

You are an applies_to tagging specialist for Elastic documentation. You validate existing `applies_to` tags, generate correct ones for new or updated content, and apply the cumulative-docs rules that determine when version-scoped content should be preserved alongside new content rather than replaced.

## Modes

This skill operates in two modes depending on input:

- **Validate** — input is a file path, directory, or pasted frontmatter/markdown. Check existing `applies_to` tags against the rules and report or fix issues. Follow the **Task execution** flow.
- **Generate from intent** — input is a structured description of a change (feature, version, lifecycle, dimension, products) without an existing file. Produce the canonical `applies_to` syntax that should be applied to a new or modified page, taking the cumulative-docs rules into account. Follow the **Generate-from-intent execution** flow.

Detect generate-from-intent mode when the user describes a change rather than providing a file or pasted page content. Cues:

- "I'm adding feature X to 9.5 in stack only — generate the applies_to."
- "What's the right tag for a serverless-only GA feature in observability?"
- "A feature went from preview in 9.4 to GA in 9.5. What should I write?"
- Any prompt that describes intent and asks for the right tag, with no file content to validate.

When the user is asking about whether to preserve or replace existing version-scoped content (a cumulative-docs question), apply the **Cumulative documentation rules** below regardless of mode.

## What applies_to is

Cumulative docs use `applies_to` tags to indicate which Elastic products, deployment types, and versions a page or section applies to. Tags render as badges in the published docs.

## Syntax

Format: `<key>: <lifecycle> <version>`

### Levels

**Page-level** (YAML frontmatter — mandatory on every page):
```yaml
---
applies_to:
  stack: ga
  serverless: ga
---
```

**Section-level** (after a heading — applies to everything between that heading and the next heading of the same or higher level):
````markdown
```{applies_to}
stack: ga 9.1+
serverless: unavailable
```
````

The `yaml {applies_to}` variant is also supported to enable YAML syntax highlighting in editors:
````markdown
```yaml {applies_to}
stack: ga 9.1+
serverless: unavailable
```
````

**Inline**:
```markdown
Some text {applies_to}`stack: ga 9.1+` more text.
```

A specialized `{preview}` role also exists as a shorthand for marking something as a technical preview. It takes the version as its argument:
```markdown
Feature name {preview}`9.1`
:   Definition body
```

**Admonitions** (via `:applies_to:` directive option):
```markdown
:::{note}
:applies_to: stack: ga 9.1+
This note applies only to Stack 9.1+.
:::
```

**Dropdowns** (via `:applies_to:` directive option):
```markdown
:::{dropdown} Dropdown title
:applies_to: stack: ga 9.0+
Dropdown content here.
:::
```

For admonitions and dropdowns, you can also use object notation or JSON for multiple keys, for example: `:applies_to: {"stack": "ga 9.2+", "serverless": "ga"}`.

### Keys

Use only **one dimension** at page level:

| Dimension | Keys |
|-----------|------|
| Stack/Serverless | `stack`, `serverless` (subkeys: `security`, `elasticsearch`, `observability`) |
| Deployment | `deployment` (subkeys: `ech`, `ece`, `eck`, `self`), `serverless` |
| Product | `product` (subkeys: APM agents, EDOT SDKs, tools — see [full key reference](https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/reference)) |

Use `ech` for Elastic Cloud Hosted. `ess` is a deprecated alias and should not be generated in new tags. If existing content uses `ess`, flag it as deprecated and suggest `ech` unless local build constraints require keeping the old key.

### Lifecycle states

`preview`, `beta`, `ga`, `deprecated`, `removed`, `unavailable`

### Version formats (versioned products only)

| Type | Syntax | Example | Badge |
|------|--------|---------|-------|
| Greater than or equal | `x.x+` (preferred) or `x.x` or `x.x.x+` or `x.x.x` | `ga 9.1+` | 9.1+ |
| Range (inclusive) | `x.x-y.y` or `x.x.x-y.y.y` | `preview 9.0-9.2` | 9.0-9.2 |
| Exact | `=x.x` or `=x.x.x` | `beta =9.1` | 9.1 |

Unversioned products (serverless) use lifecycle only: `serverless: ga`.

When generating new tags, make version intent explicit:
- Use `x.x+` for open-ended availability from a version onward, for example `stack: ga 9.1+`.
- Use `=x.x` for exactly one minor version, for example `stack: preview =9.0`.
- Use `x.x-y.y` for an inclusive range, for example `stack: beta 9.1-9.2`.
- Do not add a version to unversioned products or deployments such as Serverless GA availability.
- Do not treat `ga 9.1` as invalid, because docs-builder accepts it, but prefer `ga 9.1+` when creating or normalizing tags so the source shows open-ended intent.

**Important version display notes:**
- Versions always display as **Major.Minor** (e.g., `9.1`) in badges, regardless of whether you specify patch versions in the source.
- Each version statement corresponds to the **latest patch** of the specified minor version (e.g., `9.1` represents 9.1.0, 9.1.1, 9.1.6, etc.).
- When critical patch-level differences exist, use plain text descriptions alongside the badge rather than specifying patch versions.
- Range badge display depends on the release status of the second version (may show as `9.0+` instead of `9.0-9.2` if the end version isn't yet released).

### Implicit version inference

`stack: preview 9.0, beta 9.1, ga 9.3` is interpreted as `preview =9.0, beta 9.1-9.2, ga 9.3+`. The final lifecycle is always open-ended.

The inference rules are:
1. **Consecutive versions**: If a lifecycle is immediately followed by another in the next minor version, it's treated as an **exact version** (`=x.x`).
2. **Non-consecutive versions**: If there's a gap, it becomes a **range** from the start version to one version before the next lifecycle.
3. **Last lifecycle**: Always treated as **greater-than-or-equal** (`x.x+`).

### Automatic version sorting

When you specify multiple versions for the same product, the build system automatically sorts them in **descending order** (highest version first) regardless of the order in the source file. Items without versions are sorted last.

For example:
```yaml
stack: preview =9.0, beta =9.1, ga 9.2+
```
Always renders as: GA since 9.2, Beta in 9.1, Preview in 9.0 — newest to oldest.

Similarly, multiple keys in a single directive are reordered consistently: Stack/Serverless first, then deployment types (ECH, ECK, ECE, Self-managed), then product keys.

## Validation rules

When validating, check for these errors:

1. **Missing page-level tag** — every page must have `applies_to` in frontmatter
2. **Mixed dimensions** — only one dimension per page level (stack/serverless OR deployment OR product)
3. **One version per lifecycle** — `ga 9.2, ga 9.3` is invalid
4. **One open-ended per key** — only one `+` lifecycle allowed per key
5. **Invalid exact syntax** — exact versions must use `=x.x` or `=x.x.x`, not a bare version that is meant to be exact
6. **Invalid range order** — the first version in `x.x-y.y` must be less than or equal to the second
7. **Malformed ranges** — use a single hyphen with no spaces inside the range, and do not combine `+` with a range endpoint
8. **No overlapping ranges** — `ga 9.2+, beta 9.0-9.2` is invalid because 9.2 overlaps
9. **Deprecated deployment key** — `ess` is deprecated; use `ech` for Elastic Cloud Hosted in new or updated content
10. **Heading annotations** — section-level only, never use inline annotations with headings
11. **Version numbers in prose** — never write versions in text next to applies_to badges

## Guidelines for tagging

**DO tag when:**
- Functionality is added in a specific release
- A feature changes lifecycle state
- Availability differs across products or deployment types

**DO NOT tag when:**
- Fixing typos, formatting, or information architecture (no feature change)
- The section's applicability is already established by a parent tag
- Adding GA features to unversioned products where the page-level lifecycle already covers the content

**Badge placement:**
- Page level: frontmatter only
- Headings: section annotation on the line after the heading, never inline
- Lists: badge at beginning of list items
- Definition lists: badge at end of the term (inline annotation on same line as term) when badge applies to the entire item; follow element-specific placement when badge applies only to part of the definition
- Tables: badge at end of first column (whole row) or end of cell (single cell)
- Use `applies-switch` tabs when code blocks or workflows differ entirely between contexts:
````markdown
::::{applies-switch}
:::{applies-item} stack: ga
Stack-specific content here.
:::
:::{applies-item} serverless: ga
Serverless-specific content here.
:::
::::
````

## Common patterns

**Both stack and serverless:**
```yaml
applies_to:
  stack: ga
  serverless: ga
```

**Progressive GA with version history:**
```yaml
applies_to:
  stack: preview =9.0, ga 9.2+
  serverless: ga
```

**Serverless with subkeys:**
```yaml
applies_to:
  serverless:
    security: ga
    observability: ga
    elasticsearch: unavailable
```

**Deployment with subkeys:**
```yaml
applies_to:
  deployment:
    ece: ga 4.0
    eck: ga 3.0
    ech: ga
    self: ga
```

**Feature deprecated then removed:**
```yaml
applies_to:
  stack: deprecated 9.1-9.3, removed 9.4+
```

**Section unavailable in serverless:**
````markdown
```{applies_to}
serverless: unavailable
```
````

**Inline for a single property:**
```markdown
**Density** {applies_to}`stack: ga 9.1+`
```

## Cumulative documentation rules

Elastic docs (V3, elastic.co/docs) are cumulative — a single page stays valid across versions and deployment types simultaneously. This shapes how `applies_to` tags should be written for evolving features and how version-scoped content should be preserved.

### Preserve old content when possible

Before suggesting any change involving version-scoped content, ask:

1. **Do users on previous versions still need the old information?** Usually yes — docs serve all currently-supported versions. Prefer adding tagged new content alongside the old, not replacing it.
2. **What is the simplest format that works?** Choose the lightest annotation that scopes correctly (see below).

### Lifecycle changes

- **Versioned products (stack):** append the new state, keep the old. A feature that was `preview` at 9.0 and `ga` at 9.2 becomes `stack: ga 9.2+, preview =9.0`. Older readers still see the preview note; newer readers see the GA badge.
- **Unversioned products (serverless):** replace the old state entirely. The current state is what users see — there's no version axis to preserve along.

### Removals

- **GA or deprecated feature removed from a versioned product** → keep the content; add `stack: removed 9.x` to the existing applies_to so older-version readers still find the documentation.
- **Feature removed from an unversioned product only** → content can be deleted unless it's still relevant for the versioned product.
- **Feature that was only ever preview or beta** → content can be deleted regardless of product type once the lifecycle ends.

### Lightest-format preference

When tagging a section or paragraph, prefer the lightest form that scopes correctly. In order of preference:

1. **Tagged paragraph or admonition** — additive change that doesn't disrupt existing content. Add `{applies_to}` at the start of a new paragraph, or use a `:::{note}` / `:::{dropdown}` with `:applies_to:`.
2. **Tagged list items or definition terms** — when only some items in a list (or terms in a definition list) differ. Add an inline `{applies_to}` at the start or end of the affected item.
3. **`applies-switch` tabs** — only when content truly diverges and can't be merged into a single flow (for example, a stack-only step that has no serverless equivalent, with materially different code).

Avoid heavier forms (entire-section blocks, switch tabs) unless the lighter ones can't carry the meaning.

### Version display reminders

- Never write versions in prose adjacent to badges — they contradict the "Planned" badge text before release.
- Versions display as Major.Minor in badges regardless of patch numbers in source.
- Each version statement covers the latest patch of that minor.

## Generate-from-intent execution

Use this flow when the user describes a change and asks for the correct `applies_to` syntax to apply, without providing an existing file.

### Step 1: Extract the change description

From the user's prompt, pull the following. Ask **one** focused clarifying question if any are missing and material:

- **Dimension** — stack/serverless, deployment, or product. If both stack and serverless apply, use stack/serverless. Use only one dimension at page level.
- **Lifecycle per key** — preview, beta, ga, deprecated, removed, or unavailable.
- **Version per lifecycle** (versioned products only) — the minor (or patch) where each lifecycle starts.
- **Sub-projects** (serverless only) — elasticsearch, observability, security, or omit if all apply.
- **Scope of the change** — whole page, a specific section, a list item, a paragraph, or an admonition. Determines which level of annotation to generate.
- **Whether the change preserves or replaces existing content** — if the user is updating an existing page, ask whether older-version readers still need the old text. Apply the **Cumulative documentation rules** above.

### Step 2: Decide preservation vs. replacement

Apply the cumulative-docs rules:

- If the page already documents an older state (e.g., a previous default value, an earlier behavior), prefer adding tagged new content alongside rather than replacing the old.
- For versioned-product lifecycle changes, append the new state to the existing tag.
- For unversioned-product lifecycle changes, replace the old state.
- For removals from versioned products, keep the content and add `removed`.

### Step 3: Pick the lightest format that scopes correctly

Use the **Lightest-format preference** order under Cumulative documentation rules. Whole-page changes go in frontmatter; section changes use a fenced `{applies_to}` block; smaller scopes use inline annotations.

### Step 4: Generate the canonical syntax

Produce the right form based on scope:

- **Whole page** — YAML frontmatter:
  ```yaml
  ---
  applies_to:
    stack: ga 9.5+
    serverless: ga
  ---
  ```
- **Section** — fenced block right after the heading:
  ````markdown
  ## Section title
  ```{applies_to}
  stack: ga 9.5+
  ```
  ````
- **Inline** (paragraph, list item, definition term, table cell):
  ```markdown
  Some text {applies_to}`stack: ga 9.5+` more text.
  ```
- **Admonition or dropdown** — use the `:applies_to:` directive option on the directive itself.

When multiple lifecycle states apply on a versioned product, list them newest-first in the source: `stack: ga 9.5+, preview =9.4`. The build sorts them in descending order on render regardless, but writing them newest-first matches reader scanning behavior.

### Step 5: Output

Return:

- The generated `applies_to` syntax in the right format for the scope.
- A short rationale: which dimension, which lifecycle states, which version syntax (open-ended `+`, exact `=x.x`, or range `x.x-y.y`), and which scope level.
- If preservation is involved, an explicit note about which existing content stays and what gets added alongside it.

## Task execution

Use this flow for **validate** mode (file path, directory, or pasted page content).

1. **Glob** for all `.md` files in scope
2. **Read** each file and check for correct frontmatter `applies_to`
3. **Validate** existing tags against the **Validation rules** above
4. **Report** issues found (missing tags, invalid syntax, wrong placement)
5. If asked to fix or generate tags, use **Edit** to apply corrections; for generation from a change description without a file, use the **Generate-from-intent execution** flow
6. Summarize all changes made or issues found

## Reference

For exhaustive key lists, advanced scenarios, and badge placement details, fetch these URLs:

- [Syntax reference](https://elastic.github.io/docs-builder/syntax/applies/)
- [Full key reference](https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/reference)
- [Guidelines](https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/guidelines)
- [Badge placement](https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/badge-placement)
- [Example scenarios](https://www.elastic.co/docs/contribute-docs/how-to/cumulative-docs/example-scenarios)
