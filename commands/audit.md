---
description: Reviews a Licentry integration this plugin did not write, from the source, and reports what fails open, with file and line. No plan and no running client needed, so it works on inherited or third party code. If you built the integration with /implement and want it exercised rather than read, use /verify.
argument-hint: "[path to the integration, if it is not obvious]"
allowed-tools: Read, Grep, Glob, WebFetch
---

Somebody else wrote this integration, or it was written before this plugin
existed, and the question is whether it protects anything. You are reading, not
running. Nothing in this command changes a file.

Load the `licentry-integration` skill first.

## No plan is needed, and that is the point

`/verify` needs a plan and a runnable client. This command needs neither. If a
plan happens to exist, read it for context and say where the code and the plan
disagree, which is often the finding.

Fetch the documentation per `references/fetch-protocol.md`, every page whose
`requiredBy` includes `audit`. An audit against remembered field lists is worth
nothing, because the whole question is whether this code matches the contract as
it stands today.

## 1. Find the integration

Start in `$ARGUMENTS` if given. Otherwise find every place the product talks to
the licensing runtime: the session paths, the signature headers, the licence key
handling, the device hash, the token storage, the keystore calls, the build token
header. Then find every place the result of a licensing call is *used*, which is
usually a different set of files and is where the bypasses live.

Say what you searched. An audit that missed a directory is an audit that missed
whatever was in it.

## 2. Walk the checklist

`references/integration-checklist.md`, sections A, B and C, every item, in
order. Everything is "pass inspected", "fail" or "not checked", because you are
not running anything here. Never award "pass executed" from this command.

Read the actual lines. The failures this finds are small and specific:

- a conditional wrapped around the verify call, which fails open on every
  stripped header for the life of the product
- a key fetched at startup instead of pinned, which verifies the attacker's
  responses perfectly
- a verification over the body instead of the canonical string, which usually
  appears as a client that verifies nothing because the vendor gave up and
  commented it out
- an echo compared after the status is read, or never compared
- a helper that returns only the body, and a call site shaped
  `if (verify(res)) { unlock() }`
- one `IsValid()` that the whole product consults
- a log line, a UI string or an exit code that names which refusal happened
- a device hash built from something that moves

For each finding: the file, the line, what is wrong, and what it costs in one
sentence. Not a lecture. The vendor is going to fix these in order and they need
to know which one to fix first.

## 3. The question underneath

Beyond the checklist, answer the one thing that decides whether any of it
mattered: **if the licensing call were stubbed out to return success, would this
product still work?** If it would, say so at the top. Every control on the wire
is above a check that can be deleted, and a vendor who has not been told that
plainly will spend their budget in the wrong place.

## Report

- **Does it pass**, yes or no, first line. No if any section A item failed.
- Findings, ordered by what fails open soonest, each with its location.
- What was not checked and why.
- Nothing was changed. Say that too, so nobody goes looking for edits.

If the vendor wants these fixed, `/licentry-integrate:setup` then
`/licentry-integrate:implement` is the route, and it starts by asking them the
questions this command had to work around.
