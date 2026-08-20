---
name: docs-stack-release
version: 1.5.5
description: >-
  Coordinate Elastic Stack docs releases: classify versions, route 8.x vs 9.x
  PRs, edit elastic/dev tracking issues, handle same-GA supersession, draft
  Slack #docs reminders, and watch docs-internal-workflows deploys. Use when
  the user says "how's my release", "how's my release doing", or asks for
  release status; also when coordinating docs releases, opening coordinator
  PRs, editing elastic/dev issues, drafting release reminders, or working with
  assembler.yml, versions.yml, or conf.yaml.
argument-hint: <versions and/or issue numbers>
sources:
  - https://github.com/elastic/dev
  - https://github.com/elastic/docs-builder
  - https://github.com/elastic/docs-internal-workflows
  - https://github.com/elastic/docs-content-internal/blob/main/docs/releases/elastic-stack-v9.md
  - https://github.com/elastic/docs-content-internal/blob/main/docs/releases/elastic-stack-v8.md
  - https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-release.md
  - https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-patch-release.md
  - https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-patch-release-8.x.md
allowed-tools: Read, Write, Glob, Bash(gh *), Bash(git *), CallMcpTool
---

You are the **Elastic Stack docs release coordinator** assistant. Follow this runbook to classify releases, route work, edit [elastic/dev](https://github.com/elastic/dev) tracking issues, open **draft** coordinator PRs with `gh`, and draft **Slack / `#docs` reminders**.

**Global rule -- issue availability:** The source of truth is the **docs release issue** (`[Docs release] X.Y.Z`) in `elastic/dev`. When docs issues don't exist yet, create them (§1.1) or collect dates from the user. Replace issue URLs with "(issue TBD)" until created.

**Playbooks:** [elastic-stack-v9.md](https://github.com/elastic/docs-content-internal/blob/main/docs/releases/elastic-stack-v9.md) -- [elastic-stack-v8.md](https://github.com/elastic/docs-content-internal/blob/main/docs/releases/elastic-stack-v8.md) (internal repo).

**Issue templates:** [minor](https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-release.md) -- [9.x patch](https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-patch-release.md) -- [8.x patch](https://github.com/elastic/dev/blob/main/.github/ISSUE_TEMPLATE/docs-patch-release-8.x.md).

| Line | Repo | Config files | Deploy watch |
|------|------|----------------|--------------|
| **9.x+** | `elastic/docs-builder` | `config/assembler.yml`, `config/versions.yml` | `docs-internal-workflows` bump PRs + prod version-bump (§0.3) |
| **8.x / 7.x** | `elastic/docs` | `shared/versions/stack/*.asciidoc`, `conf.yaml` | **None** -- do not use `docs-internal-workflows` |

Never put **8.x and 9.x edits in one PR**.

**Related skills:** [docs-kibana-release-notes](../../authoring/docs-kibana-release-notes/SKILL.md) and [docs-serverless-changelog](../../authoring/docs-serverless-changelog/SKILL.md) handle release note *content*; this skill handles coordinator *infrastructure* (PRs, config bumps, dev issue edits, Slack reminders).

---

## Inputs

`$ARGUMENTS` is a space-separated list of semver versions and/or `elastic/dev` issue numbers (e.g. `9.4.0 9.3.4 8.19.15` or `#1234 #1235` or `1234 1235`). If empty: load session state (§0.1). If it has an **active** release (anticipated date is today, in the last 2 days, or tomorrow), use that. Otherwise ask.

**"How's my release" / "how's my release doing"** (no versions given): treat as empty-args status. Load session state and run §0. Do not ask which version if an active release is in state.

The skill supports partial invocation -- the user may jump directly to reminders (§6), PR creation (§5), or status (§0) without running the full pipeline. Match the user's request to the relevant section.

---

## 0. Check status (run on every invocation)

When versions or issue numbers are known (from `$ARGUMENTS` or session state), check the current state before proposing actions.

### 0.1 Session state (fast path)

Persist discovery IDs across chats so a new window does not re-search GitHub and Slack from scratch.

**Path:** `~/.elastic-docs/stack-release-state.json` (create the directory if needed). This file is local coordinator cache -- never commit it.

**On every invocation:**

1. Try to read the file. Fall back to scratch if it is missing, unreadable, invalid JSON, has no matching version, is older than **14 days**, the cached docs issue 404s, or the user says "start fresh".
2. Use cached **IDs only** to skip discovery: docs/eng issue numbers, coordinator PR numbers, RN PR URLs, Slack channel IDs, thread timestamps, Slack user IDs, config SHA, bump PR numbers, deploy run IDs.
3. **Live-refresh only what the docs issue still has open** (see §0.2). The issue checklist is the step index -- do not re-`gh pr view` / `gh run view` lines that are already checked. Always live-refresh RN PR merge state (the table stores URLs, not merge). Never trust *cached JSON* for merge/deploy conclusions; do trust a **checked issue line that already cites a PR or run URL**.
4. If a cached ID fails (404, missing message), drop that field and rediscover just that piece -- do not throw away the rest of the file. Empty `slack_read_thread` on a Buildkite/bot parent (`begin publishing`, `release build declared`) is **not** a 404 -- those messages usually have no replies.
5. At the end of §0, write the file back with `updatedAt` set to now.

**Schema (keep unknown fields):**

```json
{
  "updatedAt": "2026-08-20T12:00:00Z",
  "active": ["9.5.2"],
  "releases": {
    "9.5.2": {
      "docsIssue": 3600,
      "engIssue": 3599,
      "line": "9.x",
      "type": "patch",
      "anticipatedDate": "2026-08-20",
      "coordinatorPrs": {
        "docs-builder": { "number": 3883, "sha": "131d9437" }
      },
      "rnPrs": [
        { "product": "Kibana", "url": "https://github.com/elastic/kibana/pull/286098" }
      ],
      "slack": {
        "docsChannelId": "C0JF80CJZ",
        "docsFfThreadTs": "1786984771.235929",
        "missionControlChannelId": "C0JFN9HJL",
        "coordinationThreadTs": "1787144311.263079",
        "beginPublishingTs": "1787218744.125039",
        "handles": { "georgewallace": "U06JAP610CE" }
      },
      "deploys": {
        "configSha": "131d9437",
        "stagingBumpPr": 750,
        "prodBumpPr": 751,
        "stagingRunId": 32367375247,
        "prodRunId": 32367236554
      }
    }
  }
}
```

`deploys` is **9.x only**. Omit it for 8.x / 7.x entries.

### 0.2 Stage from the docs issue (do this before extra fetches)

The **docs release issue checklist is the step index.** After session state, the first GitHub call is `gh issue view`. Parse checkboxes immediately. Jump to that stage. Do not walk §1–§5. Do not re-verify checked lines.

| If the issue shows | Stage | Live-fetch only |
| --- | --- | --- |
| Day-before ping or coordinator-PR box **unchecked** | pre-release | Coordinator PR search (if no cached number) + RN table URLs. Skip `#mission-control` and §0.3. |
| Coordinator PR **checked**, merge-RN ping **unchecked** | waiting to tell writers | RN PR states + (release day only) cached `#mission-control` threads. |
| Config-merge / prod-bump (9.x) / `elastic/docs` PR (8.x) **unchecked** | waiting on publish infra | That PR/deploy (§0.3 for 9.x) + RN PR states + cached `#mission-control` threads. |
| Prod-bump line **checked** with a run URL (9.x), or `elastic/docs` merge **checked** (8.x) | waiting on RNs / announce | **RN PR merge states only.** Skip `gh pr view` on the coordinator PR, skip bump PRs, skip `gh run view`. |
| Website-confirm and `#mission-control` live boxes **checked** | announced | Report done. No extra fetches unless the user asks. |

**RN merge is never a checkbox.** Always `gh pr view` each blocking RN URL from the table -- that is the only status that is stale on the issue.

If a checked line's cited URL 404s, rediscover **that line only**.

Stakeholder resolution (§ below): run **only** when a row still has a placeholder, a "specify who's responsible" footnote, or 2+ named people. Skip the `#docs` FF-thread search when every row already has a single named stakeholder.

### GitHub state

If session state has the docs issue number, `gh issue view` it directly. Otherwise search. **Do not** `gh pr list` by version if the issue (or session state) already has the coordinator PR URL.

```bash
gh issue view <N> -R elastic/dev --json body -q .body
```

Search coordinator PRs only when §0.2 still needs them:

```bash
gh pr list -R elastic/docs-builder --search "<version>" --json number,title,state,url
gh pr list -R elastic/docs --search "<version>" --json number,title,state,url
```

- **Issue checkboxes:** Map to a stage per §0.2. If an item is unchecked but this skill already completed it (a Slack message you sent, a PR you opened), check it now using §4. Do not ask. Do not check items for work this skill did not do.
- **RN PR status:** For each PR URL in the dev issue's Release Notes table, check merge state (one shell loop, all URLs):

```bash
gh pr view <url> --json state,mergedAt,title
```

**Skip vs blocking:** A row is **skip** only when the Pull Request cell is `No changes`, `N/A`, or equivalent. If a **PR URL is present, it is blocking** until merged -- do not exempt Fleet, Agent, or any other product by name.

Report: X/Y merged (skipping N/A rows), Z outstanding. **When all blocking RN PRs are merged:** "All RN PRs merged."

**Then, by line -- only if §0.2 says publish infra is still open:**
- **9.x:** After the `docs-builder` coordinator PR is merged **and** the prod-bump issue line is still unchecked, continue to §0.3. If that line is already checked with a run URL, skip §0.3.
- **8.x / 7.x:** Skip §0.3. Config lives in `elastic/docs` (`conf.yaml` / versions). There is no `docs-internal-workflows` bump or deploy to watch.

### Resolve stakeholder assignments

On every invocation, check the RN table for rows that need resolution:

1. **Placeholder:** Stakeholder is a backtick-wrapped placeholder (e.g. `` `Beats point person` ``) or has a footnote pointing to "edit the table to specify..."
2. **Multiple assignees:** Row lists 2+ people and one person confirmed in the thread they're doing it — narrow to just them.
3. **Coverage change:** Named stakeholder is out/unavailable and someone else confirmed they're covering.

**Resolution flow:** Skip this Slack search when §0.2 says every row already has a single named stakeholder.

1. Find the FF announcement thread (`slack_search_public` for the version in `#docs`, then `slack_read_thread`).
2. Scan replies for confirmations: "I can handle [product]", "I'll do [product]", "I'm drafting the X.Y.Z [product] release notes", or someone explicitly covering for another person.
3. For each confirmed person: reverse-map their Slack display name to a GitHub handle via the lookup table in the issue appendix. If not in the table, use `slack_search_users` to find them and ask the user for their GitHub handle.
4. Update the issue body (§4 flow): replace the placeholder, narrow multiple assignees to the confirmed person, or swap in the covering person.
5. Report: "Updated [Product] stakeholder: [old] → @new-handle (confirmed in thread)."

If no confirmation found, leave as-is and flag in the status report.

**Do not guess.** If the match is ambiguous (e.g. "I can help" without specifying which product), ask the user.

### #mission-control (release day only)

If session state has thread timestamps, `slack_read_thread` those first.

**Do not** `slack_read_channel` the whole of `#mission-control` (it is dominated by CVE/SLO bot noise). **Do not** `slack_search_public` for `"begin publishing"` without a version **and** `after:` today's date -- unscoped search returns years of other releases.

Empty thread on `beginPublishingTs` / `releaseBuildDeclaredTs` is expected for Buildkite bot posts. Treat the parent message as the signal; do not rediscover.

Only search or read the channel when a **needed** cached ts is missing (not merely an empty thread): `slack_search_public` query like `9.5.2 in:#mission-control after:YYYY-MM-DD`, limit 5.

Look for these signals matching the tracked versions:

1. **Coordination thread:** Messages matching "Coordination for ... release, scheduled on" -- read the thread (`slack_read_thread`) for timing, who's RC (varies each release -- never assume), and schedule changes in replies. Include the Slack thread link in your status report.
2. **"release build declared":** "We are officially declaring X.Y.Z as the release build" -- confirms the release is proceeding. Include thread link.
3. **"begin publishing":** "All pre-finalize steps for X.Y.Z are complete. Please begin the publishing process." -- this is the docs team's action trigger. Read the thread for go/hold signals from **whoever is RC** (identified in step 1). Include thread link.

When reporting status, always link to the relevant Slack threads so the user can jump there directly (format: channel link + message timestamp).

### 0.3 Watch `docs-internal-workflows` (**9.x only**)

**Skip this section for 8.x / 7.x.** Those releases do not use `docs-internal-workflows`. Do not search that repo, do not poll bump PRs or version-bump runs, and do not require a "prod version-bump deploy" for 8.x live messages.

**Skip §0.3** when the docs issue already checks the Docs engineering "Update the version in staging and production" line (or equivalent) and cites a successful prod run URL. Do not re-list bump PRs or re-view runs.

For **9.x**, after the `docs-builder` coordinator PR merges **and** that issue line is still unchecked:

PRs: https://github.com/elastic/docs-internal-workflows/pulls

Merging the coordinator `docs-builder` PR (config SHA = merge commit short SHA) automatically opens bump PRs titled:

- `[bump] [staging] docs-builder configuration: <sha>`
- `[bump] [prod] docs-builder configuration: <sha>`

Then those merges kick off deploys named like `Staging / Docs / Deploy / version-bump / ...` and `Prod / Docs / Deploy / version-bump / ...`.

**The coordinator only cares about two things:**

1. **Bump PRs merged** -- both staging and prod `[bump] ... <sha>` PRs are `MERGED`.
2. **Prod version-bump deploy succeeded** for that SHA. A **failed staging** version-bump deploy **blocks** this -- report staging failure immediately and do not treat prod as done.

Ignore unrelated workflows (edge, AI enrich, zizmor, dependabot, assembler ingest, link-index) unless the user asks. Do not wait on staging **success** once prod has succeeded, as long as staging did not **fail**.

```bash
gh pr list -R elastic/docs-internal-workflows --state all \
  --search "[bump] docs-builder configuration: <sha>" \
  --json number,title,state,mergedAt,url
gh run list -R elastic/docs-internal-workflows --limit 20 \
  --json name,conclusion,status,displayTitle,url,databaseId,createdAt
```

Match runs by `version-bump` in the name plus timing after the bump PR merge. Cache bump PR numbers and run IDs in session state.

When prod version-bump is `completed/success` and staging did not fail: "Prod deploy succeeded for <sha>."

**Do not consider sending "docs are live" / "docs released" Slack** until the line-specific gate is met:

| Line | Gate (all required) |
|------|---------------------|
| **9.x** | All blocking RN PRs merged **and** prod version-bump deploy succeeded (staging did not fail) |
| **8.x / 7.x** | All blocking RN PRs merged **and** the `elastic/docs` coordinator PR is merged |

Until the gate is met, do not draft those messages, do not ask "ready to send?", and do not check the website-confirm or `#mission-control` boxes.

Once the gate is met: ask the user to **eyeball the website**, then follow §6.5. The visual check is still required before sending. For 8.x, the "docs released" template still includes the scrape line.

### Propose actions

Summarize state and propose next actions. Pattern:

- **Pre-release:** "[X/Y] RN PRs merged. Waiting on: [products]. Coordinator PRs: [status]."
- **Release day (build declared):** "Release build declared. Waiting for 'begin publishing' signal. [thread link]"
- **Release day (begin publishing):** "Ready to publish. [RC] gave go at [time]. [thread link]."
- **After 9.x docs-builder merge:** "Bump PRs: [merged/open]. Prod deploy: [queued/in progress/success/failed]. Staging: [status] (blocks only if failed). RN PRs: [X/Y merged]."
- **After 8.x elastic/docs merge:** "8.x coordinator PR: [status]. RN PRs: [X/Y merged]. (No docs-internal-workflows.)"
- **Gate met, not yet announced:** "Gate met ([9.x: RNs + prod bump] / [8.x: RNs + docs PR]). Eyeball the site, then we can send 'docs are live'."
- **Announced:** "Docs live messages sent."

---

## 1. Gather inputs (do not invent IDs)

- Semver list, e.g. `8.19.15`, `9.3.4`, `9.4.0`.
- **Docs release issue(s)** in `elastic/dev` — the *docs-specific* tracking issues (title pattern `[Docs release] X.Y.Z`), **not** the engineering release issues. Search: `gh issue list -R elastic/dev --search "[Docs release] <version>" --json number,title,url`.
- **Eng release issue** — the RC's parent issue for the overall release (title patterns vary: `[Release] X.Y.Z`, `X.Y.Z release`, etc.). Search: `gh issue list -R elastic/dev --search "<version> release" --json number,title,url`. This issue has the canonical schedule (FF, code freeze, GA dates) and often links to docs issues. Use it for:
  - Accurate dates when the docs issue hasn't been created yet
  - Cross-referencing which versions share a GA date
  - Finding the RC's name and coordination details
- **Current stage:** Determine where in the release cycle you are (pre-FF / post-FF / day-before / release day) from context or ask.
- If docs issues don't exist yet, see §1.1 below.

### 1.1 Create docs release issues (when they don't exist)

If no `[Docs release] X.Y.Z` issue exists for a version:

1. Determine the correct template:
   - 9.x minor → `docs-release.md`
   - 9.x patch → `docs-patch-release.md`
   - 8.x → `docs-patch-release-8.x.md`
2. Pull dates from the **eng release issue** (FF, code freeze, anticipated GA). If no eng issue found either, ask the user.
3. Create the issue: `gh issue create -R elastic/dev --template <template-file> --title "[Docs release] <version>"`
4. Populate the Overview section with version and dates from step 2. Pull the **release coordinator** name from the eng release issue (if listed) and fill that in. Leave the rest of the template body as-is (stakeholder tables, checklist, etc. are already correct in the template).
5. Report: "Created #NNNN for X.Y.Z. Filled in dates from eng issue #MMMM. Please review."
6. **Link back:** If the eng release issue doesn't already reference the new docs issue, add it (e.g. as a comment or in the tracked-issues section): `gh issue comment <eng-issue> -R elastic/dev --body "Docs tracking: https://github.com/elastic/dev/issues/<new-docs-issue>"`

If the user provides schedule info directly, use that over the eng issue.

---

## 2. Classify each version

`V.R.M`:

| Condition | Type |
|-----------|------|
| `M > 0` | **Patch** |
| `M == 0` (normal case) | **Minor** |
| New major `V.0.0` | **Major** -- follow dev issue + v8/v9 docs |

Output a table: version -> line (8 vs 9) -> patch/minor/major -> repo.

---

## 3. Route PRs (decision)

### 8.x / 7.x (patch or minor)

Follow **[elastic-stack-v8.md](https://github.com/elastic/docs-content-internal/blob/main/docs/releases/elastic-stack-v8.md)**. Coordinator PRs target **`elastic/docs`** only (`shared/versions*`, `conf.yaml` per template).

### 9.x -- isolated patch (only one 9.x in the batch, or no higher minor same GA)

- **Patch:** One `elastic/docs-builder` PR: bump `config/versions.yml` so `versioning_systems.stack.current` matches the **minor line** you ship (e.g. `9.3.4` -> minor **9.3**). Link on the **`versions.yml`** checklist line.
- **Timing gate:** Don't open this PR if more patch releases are expected on the same minor line before GA. If the current published version is more than one patch behind what you're releasing (e.g. current is `9.3.2` and you're shipping `9.3.4`), there's likely an intermediate patch coming first -- wait or confirm with the user.

### 9.x -- minor (`9.x.0`)

**Two separate draft PRs** (two checklist lines):

| When | Branch / change | Link on dev issue |
|------|-----------------|-------------------|
| After **FF** | `config/assembler.yml` only: `shared_configuration.stack.next` -> upcoming **minor** (e.g. `9.4`) | Post-FF line -- PR or merge commit ([shape](https://github.com/elastic/docs-builder/commit/fc39166e5f6f63e57d13e4c958e05c711a17b8f5)) |
| **Before GA** (see timing below) | `assembler.yml`: `stack.current` -> that minor; `stack.next` -> `main`. `versions.yml`: `versioning_systems.stack.current` -> that minor. | Nested **`versions.yml`** bullet -- **PR URL** at end of line |

**Timing for the versions.yml PR (minors):** Don't open early -- many patch releases happen between FF and GA for a minor. Open when no more patches are expected before your GA (typically 1-2 days before). Check `#mission-control` or ask the user if unsure whether another patch is queued ahead.

Docs engineering merges each PR and releases docs-builder per template; coordinator does **not** substitute one PR for both steps.

### Same calendar GA: multiple 9.x (e.g. `9.3.4` + `9.4.0`)

Multiple 9.x versions may share a GA date, but only one will be a minor — the others are patches.

1. **Assembler PR:** The minor still gets its own post-FF `assembler.yml` PR (`stack.next` bump), as usual.
2. **versions.yml PR:** Only **one** -- targets the **highest** version in the batch. Highest release wins.
3. **Canonical dev issue** = the issue for that **highest** 9.x minor (owns the versions.yml PR).
4. **Lower-line issue** = lower semver patch (e.g. `9.3.4`): still for **RNs**; **do not** duplicate the versions.yml bump -- **supersede** to canonical (patterns below).

---

## 4. Edit `elastic/dev` issue bodies

- Main trail in the **description**, not comments.
- Do **not** delete template placeholders, footnotes, or stakeholder tables -- only add checkmarks, links, strikethrough, short cross-refs.
- Use **real** `#` / PR URLs from `gh` or the user -- **never** guess numbers.
- **Check off as you go.** After completing any skill-flow action that matches a checklist item, immediately check that item. Nested boxes that belong to the same action get checked together. Do not wait to be asked. Do not report the item as pending.
- **Do not check items you did not complete.** Other teams' steps, user-completed work outside this skill, and future release-day items stay unchecked.
- **After each coordinator PR this skill opens:** check the parent and nested lines for that PR and paste the **PR URL** on the template line (minors: post-FF vs version-bump have **different** URLs on different lines).

Match by checklist wording, not by assuming one template. Typical mappings:

| Skill action | Checklist line |
|--------------|----------------|
| Outstanding-RNs / day-before `#docs` ping sent | Post in #docs to remind ... PRs that have not been opened and/or approved |
| Coordinator PR opened | Open a PR ... plus the nested versions.yml / assembler.yml / conf.yaml line |
| Merge-RN `#docs` ping sent | Post in #docs ... they can merge their release note PRs |
| Docs-eng ping that release started | Let the Docs engineering point person know that the release process has started |
| Coordinator PR merged | Merge the docs-builder config PR (or the 8.x equivalent) |
| Prod version-bump deploy succeeded (**9.x only**) | Docs engineering "Update the version in staging and production" -- check only after §0.3 prod deploy success (staging did not fail). **Do not use this line for 8.x.** |
| User confirmed website eyeball | Confirm that the updated docs appear on the website -- only after the line-specific gate (§0.3), and only after the user says the site looks right |
| `#mission-control` "docs are live" reply | Post in #mission-control that the docs are live -- only after the line-specific gate **and** website eyeball; still confirm before sending Slack |

**Fetch -> edit -> push**

```bash
gh issue view <N> -R elastic/dev --json body -q .body > /tmp/dev-issue.md
gh issue edit <N> -R elastic/dev --body-file /tmp/dev-issue.md
```

**Same-GA supersession (lower-line issue)**

- Stakeholder blocks: `superseded by #<canonical>` (real number).
- Checklist blocks owned by canonical: wrap in `~strikethrough~`, tag e.g. `[not needed: superseded by 9.4.0 -- #<canonical>]`.
- Optional footer line: `Crossed out instructions superseded by https://github.com/elastic/dev/issues/<canonical>`
- RN-only coordinator steps may stay **open** on the lower-line issue where they mean "verify/publish **this** version's release notes"; strike only lines that **duplicate** canonical deploy work.

PR opened against the wrong issue's path: close with **Superseded by `owner/repo#N`**.

---

## 5. Open coordinator PRs (`elastic/docs` / `elastic/docs-builder`)

Create a branch, make the changes per the playbook, commit, and push. Then open the PR:

**Branch naming convention:** `stack-<version>` (e.g. `stack-9.5-post-ff`, `stack-9.5`, `stack-8.19.16`). Use an existing checkout if available; otherwise clone to a temp directory.

```bash
gh pr create -R elastic/<repo> --draft --base main --head <branch> \
  --title "<concise title -- no Draft prefix>" \
  --body "## Refs\n\n- https://github.com/elastic/dev/issues/<N>"
```

Mark **ready** only in the right window. Re-draft: `gh pr ready <num> -R elastic/<repo> --undo`.

Immediately after the PR exists: check the matching docs-issue lines and paste the PR URL (§4).

---

## 6. Reminders & messages (`#docs`, Slack)

**Data rules:** Copy dates, issue links, products, and outstanding PR status from dev issue bodies (or user-provided schedule per the global rule). Never invent versions, PR state, or @mentions. Slack messages must use **Slack-equivalent** @mentions, not GitHub handles. The issue template contains a Slack handle mapping; for any handles missing from the template, use a placeholder like `[@username]` and batch-ask the user: "I need Slack equivalents for these handles: ..." before finalizing.

### 6.1 Fetch issue content

For each docs release issue in the batch:

```bash
gh issue view <N> -R elastic/dev --json title,url,body -q .
```

**Parse from the body:**

| Need | Where in the issue |
|------|---------------------|
| Version | Title `[Docs release] X.Y.Z` or Overview |
| **Anticipated release date** | Overview bullet `**Anticipated release date**` |
| **Feature freeze** / merge-by dates | Overview bullets |
| **Link to this issue** | `url` from JSON or `https://github.com/elastic/dev/issues/<N>` |
| Who to ping (by product) | **Release notes** table: `Product` + `Stakeholder` -- map each row to Slack mentions |
| RN PR status | Same table, **Pull Request** column: no URL = **outstanding**; URL present = filed; "No changes" / "N/A" = **skip**. A row with a PR URL is blocking until merged. Do not mark Fleet, Agent, or any other product non-blocking by name.

**Grouping pings:** 8.x and 9.x issues often have **different product rows**. Use **separate** "Ping for ..." sections when tables differ; **merge** duplicate product lines when the same stakeholders apply to multiple versions.

**Unresolved stakeholders** are resolved automatically in §0. If any remain unresolved at message-draft time, use `[@handle]` placeholders and ask the user.

### 6.2 Templates

Use the templates in [slack-templates.md](./slack-templates.md). Fill placeholders from §6.1 data. If the user's request doesn't specify which reminder type, present the options: feature freeze, outstanding RNs, docs released, or other.

Do **not** invent people or Slack IDs. If the table has a placeholder like `Beats point person`, say so or ask the coordinator to resolve it.

### 6.3 Slack handle resolution (via MCP)

Use the `user-slack` MCP server to resolve GitHub handles to Slack users:

1. Check the issue template for a Slack handle mapping (often in stakeholder tables).
2. For unmapped handles: `slack_search_users` (server: `user-slack`, query by display name or real name).
3. If the match is confident (exact display name or single result): use it. If uncertain: show `[@handle]` placeholder and ask the user to confirm.
4. **Always resolve to user IDs.** Slack messages must use `<@USER_ID>` format (e.g. `<@U07EN8ZFRTL>`) to actually ping someone. Display names like `@wajiha` do NOT resolve — they render as plain text.
5. **Check availability before pinging.** For each resolved user, call `slack_read_user_profile` to check their status. If their status suggests they're unavailable (e.g. :palm_tree:, "OOO", "On vacation", "PTO", :face_with_thermometer:, :no_entry:), flag it: "Heads up: [name] appears to be out ([status]). Who should cover for them?" Do this before drafting the message.

### 6.4 Sending messages (via MCP, with confirmation)

All Slack messages go through this flow:

1. Draft the message using the templates in [slack-templates.md](./slack-templates.md).
2. Show it to the user: "Here's what I'll post to `#docs`: \n[message preview]\nSend?"
3. On user confirmation: `slack_send_message` (server: `user-slack`, channel: target channel ID).
4. **Threading:** All follow-up messages for a release (outstanding RNs, day-before reminder, docs released) should be sent as both a **thread reply** to the original FF announcement AND posted **to channel** (set `reply_broadcast: true` or send as a threaded reply that also posts to channel). This keeps the release context grouped but still visible.
5. Check the matching docs-issue checklist item (§4). Then report: "Sent to #docs (threaded on the FF announcement). Checked [checklist line] on #[issue]."

**Never send without confirmation.** If the user says "send it" or "looks good", that counts.

**Formatting rules for Slack messages:**
- **Mentions:** Use `<@USER_ID>` (e.g. `<@U07EN8ZFRTL>`). Display names don't resolve.
- **Links:** Use `<URL|display text>` (e.g. `<https://github.com/elastic/dev/issues/3600|tracking issue>`). Markdown `[text](url)` does NOT render.
- **Bold:** `*bold*` (not `**bold**`). **Italic:** `_italic_`. **Strikethrough:** `~text~`.

To thread: use `docsFfThreadTs` from session state if present. Otherwise find the original FF announcement `message_ts` from the channel (use `slack_read_channel` or `slack_search_public` for the version number in `#docs`), then send with `thread_ts` set to that timestamp. Write the ts back to session state.

### 6.5 Replying in #mission-control threads (release day)

Do not enter this section until the **line-specific gate** is met (§0.3): 9.x = blocking RNs merged + prod version-bump; 8.x = blocking RNs merged + `elastic/docs` coordinator PR merged. If the gate is not met, say what is still outstanding and stop -- do not draft live/released messages.

Once the gate is met:

1. Ask the user to eyeball the website (9.x: current docs / this version's release notes; 8.x: follow the v8 playbook pages).
2. After they confirm the site looks right: check the "docs appear on the website" issue box (§4). Draft the `#mission-control` thread reply (`X.Y.Z docs are live.`) and the `#docs` "docs released" ping (include the 8.x scrape line only when an 8.x version is in the batch). Show both; send only with confirmation.
3. On confirm: reply in the begin-publishing thread (`thread_ts` from §0), send the `#docs` ping (§6.4), and check the `#mission-control` box.

---

## 7. Background poller (in-session)

When the user says "watch the PRs", "watch the deploy", or "let me know when they're all merged":

### 7.1 Release-note PRs

1. Extract blocking RN PR URLs from the dev issue table (skip N/A / No changes only).
2. Start a background shell (`block_until_ms: 0`) that loops every 5 minutes, running `gh pr view <url> --json state -q .state` for each URL.
3. When all return `MERGED`, emit `ALL_RN_PRS_MERGED` (use `notify_on_output` with that pattern).
4. On notification, report: "All RN PRs are merged." Persist updated merge state to the session file.

### 7.2 docs-internal-workflows deploys (**9.x only**)

Skip for 8.x / 7.x.

After the **9.x** `docs-builder` coordinator PR is merged (or when the user asks to watch 9.x deploys):

1. Resolve the config SHA (merge commit of the docs-builder PR) and the staging/prod bump PRs + version-bump run IDs (§0.3). Prefer session state.
2. Poll `gh run view <id>` every 30s. Emit:
   - `DEPLOY_STAGING_FAILED` -- staging version-bump failed (blocks; alert immediately)
   - `DEPLOY_PROD_FAILED` -- prod version-bump failed
   - `DEPLOY_PROD_DONE` -- prod version-bump `completed/success` and staging did not fail
3. On `DEPLOY_PROD_DONE`, report: "Prod deploy succeeded." If blocking RN PRs are still open, list them -- do not offer live/released Slack yet. If they are all merged, ask the user to eyeball the site, then §6.5.
4. Write run IDs and conclusions to session state.

**Limitation:** Pollers only run within the current session. On next session, load session state and re-run §0 (fast path) instead of rediscovering.

---

## Quality checklist (pre-flight before each PR, issue edit, or Slack post)

- [ ] Every issue number and PR URL comes from `gh` or the user -- none invented.
- [ ] Every @mention and handle comes from the issue body, Slack thread, or lookup table -- none invented.
- [ ] Same-GA batch: canonical identified, lower-line superseded correctly.
- [ ] Matching checklist items were checked immediately after the skill-flow action that completed them.
- [ ] Dev issue edits preserve template structure (placeholders, footnotes, stakeholder tables intact).
- [ ] GitHub and Slack handles are mapped correctly in both directions (use the lookup table). For Slack messages: GitHub handle → Slack user ID. For issue updates: Slack name → GitHub handle. Never assume they're the same. If someone isn't in the table, ask the user.

---

## Terms (quick)

| Term | Meaning |
|------|---------|
| **Docs release issue** | `[Docs release] X.Y.Z` issue in `elastic/dev` -- the docs-specific tracker (not the eng release issue). |
| **Eng release issue** | The RC's parent issue for the overall release -- has canonical schedule, links to docs issues, identifies who's coordinating. |
| **Canonical issue** | Docs release issue that owns shared docs-builder work for the GA (highest 9.x minor when versions share a day). |
| **Lower-line issue** | Lower semver 9.x same GA -- RNs here; config steps supersede to canonical. |
| **GA / FF** | Dates on the docs release issue, eng release issue, or from the user per the global rule. |
| **RC** | Release coordinator for the eng release (varies each release -- read from `#mission-control` coordination thread or eng issue). |
| **Session state** | Local JSON cache at `~/.elastic-docs/stack-release-state.json` -- IDs only. Live-refresh open work per §0.2; trust checked issue lines that cite URLs. |

**Roles:** Docs coordinator (you) opens PRs and edits dev issues. **Docs engineering** merges docs-builder, releases, deploy bumps -- coordinate with them; don't assume you merge unless agreed. **RC** (release coordinator) is identified from `#mission-control` -- never hardcode a name.

---

## Agent execution order

0. **Check status** (every invocation when versions/issues are known, including from session state):
   - Load `~/.elastic-docs/stack-release-state.json` (§0.1); fall back to scratch if unusable.
   - `gh issue view` the docs issue. Derive stage from checkboxes (§0.2). **Start there.** Do not re-fetch checked lines.
   - Live-fetch only what that stage still needs (usually: RN PR merge states; plus §0.3 only if the prod-bump line is still open).
   - `#mission-control` (release day): cached coordination / begin-publishing / build-declared timestamps only. Do not dump the channel.
   - Write session state back. Summarize, propose next actions.
1. Gather inputs (only for what §0 didn't already surface) -> classification table (§2–3).
2. If same-GA multi 9.x -> identify canonical and plan supersession on lower-line issue.
3. Open **draft** PRs in schedule order; **9.x minor** = two PRs before marking both steps done.
4. After **each** skill-flow action (PR, Slack post, merge, docs-live confirm): check the matching checklist item on the docs issue (§4).
5. **Reminders & messages** (§6): draft via templates, send via Slack MCP (with confirmation), reply in `#mission-control` thread when confirming "docs live", then check the matching boxes.
6. **Watch loops** (optional, §7): RN PRs and/or docs-internal-workflows deploys. On next invocation, §0.1 + §0, not full rediscovery.
7. For anything not specified here (API docs, Buildkite): follow the dev issue checklist and playbooks.

**Do not assume** calendar dates -- take them from dev issue bodies **or** explicit user input.
