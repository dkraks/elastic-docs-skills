# Slack message templates

Reference templates for §6 of the docs-stack-release skill. Fill placeholders from issue data (§6.1).

## Feature freeze (multi-release)

If all issues share one anticipated release date, use one date; otherwise list per version.

```markdown
Hi everyone! :wave: Today is the feature freeze for multiple releases.

The following releases are scheduled to release <ANTICIPATED_DATE>: <VERSION_LIST>

Please add your release note PRs to the issues linked below:

**Releases & related issues**

(Repeat for each issue in the batch:)
<VERSION> -- [<issue title>](<issue URL>)

---

:bell: **Ping for <VERSION_OR_GROUP_A>**

(One line per product with Slack @mentions equivalent to stakeholders in the issue.)

---

:bell: **Ping for <VERSION_OR_GROUP_B>** *(repeat when 8.x vs 9.x tables differ)*
```

## Outstanding release notes (day before)

Day-before reminder for PRs not yet filed or approved.

```markdown
The <VERSION_LIST> release is scheduled for tomorrow. The following release note PRs are still outstanding:

(Repeat for each outstanding row:)
• <Product> <Stakeholder @mention>

Please file and/or get approval on your PRs today. Tracking issue: <ISSUE_LINK>
```

## Merge release notes (release day)

Release-day message: the release has kicked off and teams can merge.

```markdown
The <VERSION_LIST> release has kicked off. If you haven't already, you can merge your release notes PRs.

<@api-docs-person> it is safe to update APIs.
```

## Docs released (final / 8.x scrape)

Send after docs are live. Include the exact 8.x semver from the issue for the scrape line. If the batch is **9.x-only**, omit the scrape line.

**One version** -- singular; do not use "versions" or "all":

```markdown
Docs for stack version <VERSION> are released!
<Slack @mention> you can start scraping the docs for <8.x_VERSION> now
```

**Multiple versions:**

```markdown
Docs for stack versions <VERSION_LIST> are all released!
<Slack @mention> you can start scraping the docs for <8.x_VERSION> now
```

## Other states

- **Day before:** `#docs` reminder -- reuse Overview dates + issue links; optionally list products still outstanding.
- **Release day:** Short "merge RNs" / "docs live" -- follow checklist timing; names from stakeholders.

## #mission-control thread reply (release day)

Reply in the "begin publishing" thread after confirming docs are live:

```markdown
<VERSION_LIST> docs are live.
```

Keep it short — this thread is noisy and read by many teams.
