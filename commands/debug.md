---
description: Takes one refusal you cannot explain, for example a 401 nonce_mismatch or a 400 on every heartbeat, works out which of the conditions actually produced it, and tells you what to change. One symptom at a time. Changes nothing.
argument-hint: "the refusal, for example \"401 nonce_mismatch on every second heartbeat\""
allowed-tools: Read, Grep, Glob, WebFetch
---

A refusal has a small, knowable set of conditions behind it. The job here is to
eliminate all but one, using the documentation and the vendor's own code, rather
than to guess from the name of the error.

Load the `licentry-integration` skill first.

The symptom is `$ARGUMENTS`. If it is empty or vague, ask for it before doing
anything else: a status code, the `error` field if there is one, and which route.

## 1. Get the evidence

Ask for what you do not have, and ask for all of it at once:

- The exact status code and the exact `error` string.
- Which route, and on which call in the sequence: the first beat, the beat after
  a refresh, every call, one customer only.
- When it started, and what changed just before. A deploy, a new build, a control
  turned on, a machine moved.
- The request body the client sent, field names only if the vendor prefers.
  **Never ask for a licence key, a token, an API key or a device hash value.**
  Field names and shapes are what you need, and values are what leaks.
- Whether it reproduces, and on how many machines.

If they can only give you some of this, work with what there is and say which
conditions you could not eliminate.

## 2. Read what the documentation says produces it

Fetch per `references/fetch-protocol.md`, every page whose `requiredBy` includes
`debug`. The failure table names the causes and the correct recovery for each
refusal, and the endpoint reference gives the exact accepted fields, which is
where a surprising number of these end: the bodies are strict, so one unexpected
field refuses every call on that route.

List every condition the documentation gives for this refusal, and only then
start eliminating. A refusal with four causes debugged from the most famous one
is three wrong answers waiting.

## 3. Eliminate against their code

For each candidate condition, find the line in the vendor's client that decides
it, and say whether that line does the right thing. This is where the answer
comes from: a candidate you have not looked for in the code has not been
eliminated.

The method, on one example. `401 nonce_mismatch` on the beat after a refresh has
a documented cause that is different from the same refusal in the middle of a
session, and the two have different fixes, and the fix for one makes the other
worse. So establish *where in the sequence* it happens before you diagnose it,
and confirm what the current documentation says about that position rather than
assuming the chain behaves the way the sequence number does.

Timing questions are usually a clock, ordering questions are usually two things
running at once, and "it worked yesterday" is usually a control that got turned
on or a field that got added to a strict body.

## 4. The one you are not meant to be able to explain

Some refusals are deliberately uniform. Activation answers the same generic
refusal, on a padded schedule, whether the key is unknown, expired, revoked,
suspended, flagged or a canary, precisely so that nobody can sort a stolen key
list into live and dead.

When the symptom is that refusal, say so plainly:

- The client cannot tell which it was, and it must not try. Any client logic that
  distinguishes them rebuilds the oracle the platform removed.
- The vendor can look up their own licence in their own dashboard, which is the
  legitimate route to the answer and the one to point them at.
- What to check on their side is everything that is not the licence state: the
  product slug, the protocol version header, whether a control was turned on, the
  build token, the exact fields in the body.

Do not narrow a generic refusal by probing the API with variations. That is
building the oracle from the outside.

## 5. Answer

- The condition, or the shortlist you could not narrow further and the evidence
  that would separate them.
- The fix, with the file and line to change.
- Whether it affects one machine or every copy in the field, since that decides
  whether it is a support reply or a release.
- What you assumed, if anything.

Nothing is changed by this command. If the fix is code,
`/licentry-integrate:implement` writes it, and it will list what it wrote.
