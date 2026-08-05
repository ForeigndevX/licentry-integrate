---
description: Builds the licensing integration into your own files, from the plan that /setup wrote. Run this second. It lists every file it changed and why, and it stops rather than guessing if the plan is missing or an answer it needs is not in it.
argument-hint: "[part to build, for example \"session loop\" or \"offline\"]"
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit, Bash
---

You are writing code into a vendor's own product. It will ship. Load the
`licentry-integration` skill first and hold the six rules in front of you while
you write, because four of them fail silently and this command is where they get
built in or missed.

## Before anything

Read `.licentry/plan.md`. **No plan, no implementation.** If it is not there,
say so and point at `/licentry-integrate:setup`. Do not survey the codebase
yourself as a substitute: `/setup` also asks the vendor questions you cannot
answer from source, and a plan reconstructed from code alone is missing exactly
those answers.

If the plan lists an unanswered question that blocks what you are about to
build, stop and name it. Building on an assumed device hash recipe is worse than
building nothing, because it ships.

Then fetch the documentation per `references/fetch-protocol.md`: the manifest,
then every page whose `requiredBy` includes `implement`. If the plan's
`docsRevision` differs from the manifest's, tell the vendor which pages changed
before you write anything, since the plan may have been made against different
field lists.

If `$ARGUMENTS` names one part, build that part and leave the rest.

## What to build, in this order

The order is not arbitrary. Each step is useless until the one above it exists.

**1. Verification.** Signature verification, the nonce echo comparison, and the
pinned key map. Nothing else in the integration is a security control until this
is in place, so it is written first and everything else calls into it. Build it
so that it cannot be called wrongly: one function that takes the raw bytes, the
headers and the nonce that was sent, and returns either the parsed body **and**
the status, or a refusal. Never a bare boolean, and never a function that
returns only the body, because that shape invites `if (verify(res)) { unlock() }`
and turns an authentic refusal into an unlock.

**2. The device hash.** From the inputs the plan records, emitted as lowercase
hex, identical on every call. Put a comment above it saying it must be tested
across a reboot and what happens if it moves: the session is revoked and a
device mismatch is recorded against a paying user.

**3. Activate.** Exactly the fields the endpoint reference lists and no others.
The bodies are strict and one extra field is a refusal on every call.

**4. The loop.** Heartbeat, refresh, logout, with the sequence and nonce rules
the lifecycle section gives. Write the loop, not a single call: every real bug
lives in the second beat and after. Serialise it, one owner, per the plan's
threading section.

**5. The failure branches.** Every row of the failure table with its own
handler. Not a catch-all, and no re-activation on the rows where re-activation
is wrong or terminal.

**6. Offline**, if the plan says the product needs it. Verify the grace token
against the pinned key and enforce its expiry, its device claim and its
revocation claim. Skipping the device claim turns the token into a file a user
copies to ten machines.

**7. The gates.** The protected capabilities from the plan. Make the program
need the session's answer rather than consult it. No single `IsValid()`, no one
boolean the whole product reads, and more than one moment of checking. If the
vendor's product can be made to depend on values that arrive inside the session
and cannot be invented locally, say where, because that is the only part of this
that survives someone removing the check.

## While you write

- **Start from the published samples.** The documentation carries samples in the
  vendor's language and they are tested against the contract. Take the shape from
  them rather than inventing one. If the language has no published sample, say so
  and build from the endpoint reference, and tell the vendor which parts you had
  to derive.
- **Say in a comment whether P-256 verification is standard library in this
  language or needs a package**, and name the package. The vendor should not find
  that out at build time.
- **No secret goes into source.** Not the licence key, not an API key. The API
  origin comes from configuration, not a literal.
- **No em dashes or en dashes**, in code or in comments, and no wording that
  reads as machine written. The vendor reads these comments.
- **Never write anything that maps a refusal to a licence state**: not in a UI
  string, not in a log line, not in an exit code, and not in a retry policy that
  varies by reason.
- Match the surrounding style of their codebase even where yours would differ.
- Do not improve adjacent code. Note it and move on.

## Afterwards

If the plan records a build or test command, offer to run it. Ask first, and if
it fails, show the output rather than describing it.

Append to the plan's change log: every file created or changed, what changed,
and which rule or requirement it serves.

Then report, and this part is not optional:

- **Every file you touched, with what you wrote in it and why.** Never edit
  silently.
- What you did not build, and why: an unanswered question, a part the vendor
  deferred, a platform with no keystore.
- What the vendor has to do by hand: pin the keys into the build, test the
  device hash across a reboot, put the test key in the environment.
- That `/licentry-integrate:verify` is what proves this refuses when it should,
  and that reading it is not the same as proving it.
