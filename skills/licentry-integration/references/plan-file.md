# The plan file

`.licentry/plan.md`, in the root of the vendor's repository. `/setup` writes it.
`/implement`, `/harden` and `/verify` read it and refuse to run without it.

It is a working document the vendor is meant to read, not a machine format.
Keep it in their repository and under their version control, because the record
of which documentation revision an integration was built against is worth having
the day something stops working.

## Rules

- **`/setup` creates it. Other commands append to it.** Never rewrite a section
  another command wrote. If `/setup` runs again, write a new dated run and leave
  the previous one in place, because the difference between two runs is often
  the answer to why something changed.
- **Record the source of every answer**: the vendor said it, or the plugin
  proposed it and the vendor accepted. Six months later nobody remembers which.
- **Record the documentation revision.** A plan built against an older
  `docsRevision` is what `/check-for-updates` reports on.
- **An unanswered question stays in the file as unanswered.** Deleting it turns
  a blocked decision into a silent assumption.
- **No secrets.** No licence keys, no API keys, no tokens. Record where a secret
  lives, never what it is.

## Template

```markdown
# Licentry integration plan

Written by licentry-integrate v<version> on <date>.
Do not delete. /implement, /harden and /verify read this file.

## Documentation read

Manifest docsRevision: <value>
Fetched <timestamp>:

| Page | URL | Revision |
|---|---|---|
| docs | https://licentry.cc/docs | <value> |

Pages not read, and why: <page and reason, or "none">

## The product

- Vendor account: <exists / to be created>
- Product slug sent at activate: <slug>
- API origin, and where the client reads it from: <config location>
- Platforms shipped to: <list>
- Test licence key held in: <environment variable or secret store, never here>

## Survey

One entry per item in codebase-survey.md, with paths and line numbers.

1. Entry points: <paths>
2. Exit paths: <paths>
3. What is protected: <capabilities and where they are invoked>
4. Existing licence or auth code: <paths, or "none found">
5. HTTP client: <library, where requests are built, how raw bytes are reached>
6. Secret storage: <store and API per platform>
7. Build system: <entry point, how a constant is embedded>
8. Threading and process model: <where the session loop will live>
9. Oracle surfaces: <UI, logs, exit codes, crash reporting>
10. Configuration: <mechanism>
11. Offline expectation: <current behaviour on network failure>
12. Tests: <command, framework, HTTP test double>

Findings in existing code, reported and not fixed:
- <path:line, what is wrong, why it matters>

## Answers

One entry per question in vendor-questions.md.

1. What is protected: <answer> (vendor)
2. Account, product, test key: <answer> (vendor)
3. Language and HTTP stack: <answer> (vendor)
4. Platforms and secret storage: <answer> (proposed, accepted)
5. Device hash inputs: <answer> (vendor)
6. Offline requirement: <answer> (proposed, accepted)
7. What the user sees on refusal: <answer> (vendor)
8. Device binding now or later: <answer> (proposed, accepted)
9. Shipped builds that cannot be updated: <answer> (vendor)

Unanswered, and what each one blocks:
- <question, and the decision it blocks>

## What will be built

- Files to create: <paths and what goes in each>
- Files to change: <paths and what changes>
- Where the six rules land: <which file holds verification, which holds the
  session loop, which holds the gates>
- Gates: <each protected capability and how it comes to need the session>

## Controls

| Control | Current | Target | Client prerequisite | Status |
|---|---|---|---|---|

Filled in by /harden. A control is only ticked once its client prerequisite is
shipped and in the field.

## Change log

Appended by every command that edits a file.

### <date> /implement
- <path>: <what changed and why>
```

## Reading it

A command that reads the plan checks three things before acting: that the file
exists, that the `docsRevision` in it matches the manifest it just fetched, and
that the answers it needs are present. A mismatched revision is not a stop, it is
a warning to the vendor with the list of pages that changed. A missing answer is
a stop.
