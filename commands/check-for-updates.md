---
description: Checks whether Licentry's documentation or this plugin has changed since your integration was planned, names which pages moved, and tells you which commands to run again. Run it before you trust anything the other commands told you a while ago.
allowed-tools: Read, Grep, Glob, WebFetch
---

Documentation this plugin read six months ago is not documentation. This command
is how the vendor finds out that something moved, before it costs them a
release.

Load the `licentry-integration` skill first.

## 1. What is installed

Read `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json` for the installed
version. If that path is not resolvable in this environment, say so and ask the
vendor which version they installed rather than guessing.

Read `.licentry/plan.md` if it exists, for the `docsRevision` and the per page
revisions it recorded. No plan is not an error here: it just means there is
nothing to compare the documentation against, and you report on the plugin
version only.

## 2. What is published

Fetch the manifest per `references/fetch-protocol.md`. If it will not fetch, that
is the answer: report it as an outage or a defect and stop. This is one command
where failing to fetch is itself the finding, so make it loud rather than
apologetic.

Check `manifestVersion` first. A shape this plugin does not know means the
plugin is too old to read it, which is a stop and an update.

## 3. Compare

**The plugin.** Installed against `plugin.latestVersion`. If the installed
version is below `plugin.minSupportedVersion`, say so first and treat everything
else as unreliable: below that line the commands were written against a contract
that has moved.

**The documentation.** The plan's `docsRevision` against the manifest's, and
each page's recorded revision against the manifest's. Name the pages that
changed. "Something changed" is not useful, "the controls page and the failure
table changed" tells the vendor what to look at.

## 4. Say what to do

Only what follows from what changed:

- **Nothing changed.** Say that in one line and stop. Do not pad it.
- **The plugin is behind.** Tell them to update it where they installed it from:
  the `/plugin` command in Claude Code for a marketplace install, or a fresh
  download from the dashboard. This command does not modify the plugin itself,
  and any command that offered to would be editing the thing it was checking.
- **A page changed.** Name it, say what it covers, and name the command that
  reads it: the controls page means `/harden`, the endpoint or verification
  pages mean `/audit` and then `/implement` if the audit finds drift, the failure
  table means `/debug` is now working from something different.
- **The plan is old but the docs have not moved.** Say that too. It is the answer
  the vendor most wants and the one an update checker least often gives.

If a page changed and the vendor's integration was built from the old revision,
be clear about the risk without inflating it: a client built against a superseded
field list keeps working until the day it does not, and the cheap check is
`/licentry-integrate:audit` against the current pages.
