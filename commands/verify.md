---
description: Proves an integration actually refuses, instead of reading it. Strips a signature header, alters a body, replays a response, breaks the nonce chain, and reports which cases refused and which did not. Wants a plan and a runnable client. To review code this plugin did not write, use /audit instead.
argument-hint: "[one case to run, for example \"stripped signature\"]"
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit, Bash
---

Reading an integration tells you what it looks like. Running it against a
hostile server tells you what it does. The failures that matter here all look
correct on the page, so this command exists to produce the failure and watch what
happens.

Load the `licentry-integration` skill first.

## Before anything

Read `.licentry/plan.md`. No plan means you do not know what this client was
supposed to do, and you are doing an `/audit` rather than a verification. Say so
and point at `/licentry-integrate:audit`.

Fetch the documentation per `references/fetch-protocol.md`, every page whose
`requiredBy` includes `verify`. You are checking behaviour against the current
contract, not against what the plan assumed.

## The harness

Most of these cases need a server that answers wrongly on purpose. Build one, in
the vendor's own test setup, using the seam the survey recorded.

Two rules about it, and they are not negotiable because getting them wrong
creates the vulnerability this command exists to find:

- **The test signing key exists in test configuration only.** The client under
  test pins it in its test build. It never enters a release artifact. Before you
  finish, grep the release build path for the test key id and report what you
  find.
- **The harness never becomes a fallback.** If the client can be made to talk to
  it by configuration alone in a release build, that is a finding, and a serious
  one: it means an attacker can do the same.

If the product has no seam for this, say so. Then run what you can and mark the
rest "not checked". Do not simulate a refusal by reading the code and asserting
what it would have done. That is inspection, and it gets the inspection word.

## The cases

Every one of these is a case where the client must refuse. A client that
continues has failed, and a client that refuses for the wrong reason has also
failed, so check the reason.

1. **No signature header.** Strip `X-Licentry-Sig` from an otherwise valid
   response. The licensed path must not run.
2. **Altered body.** Change one byte of the body, leave the signature. Refusal.
3. **Stale timestamp, both directions.** Move the signature timestamp outside
   the window forwards, then backwards. Both refuse. A client that only checks
   one side is defeated by setting the clock back.
4. **Unknown key id.** Sign with a key the client does not pin. Refusal, not a
   fetch, not a skip.
5. **Wrong signature form.** Send a DER encoded signature instead of raw
   `r||s`, and send one that is not 64 bytes. Both refuse.
6. **Replay.** Capture a valid response and replay it against a later request
   with a different nonce. Refusal, on the echo, before the status is read.
7. **A verified refusal.** Return a correctly signed refusal. The signature
   verifies and the licensed path must still not run.
8. **Fetch instead of pin.** Block the JWKS host, then offer a different key at
   it. The client must be unaffected by both, because it should never be asking.
9. **The nonce chain.** Retry a beat with the identical body and confirm it is
   accepted as a retry. Then advance the sequence while resending an old nonce
   and confirm that is not treated as one.
10. **Across a refresh.** Refresh, then beat. This is the one case a short test
    never reaches and a real install reaches every day.
11. **Device hash stability.** Compute it twice in the same process, then across
    a process restart, then across a reboot if the environment allows one. If it
    does not, mark it not checked and tell the vendor in writing to do it before
    shipping, because this failure lands on their paying user.
12. **Network gone.** Kill connectivity mid-session. The client either runs on a
    valid offline grace token with its expiry, device and revocation claims all
    enforced, or it stops. "Unlicensed but running" is a failure.
13. **Logout on exit.** Take each exit path the survey listed and confirm the
    logout call is made on the ones that can make it.

Then walk sections A and B of `references/integration-checklist.md` for anything
the cases above did not reach, especially the oracle surfaces, which are read
rather than run.

If `$ARGUMENTS` names one case, run that case only and say so.

## The verdict

Use the four words from the checklist: pass executed, pass inspected, fail, not
checked. One line per case, with the file and line for anything that failed.

**Any section A failure means the integration does not pass.** Say that plainly
at the top, first line, before the detail. Do not soften it into a score, and do
not let twelve passes outweigh one rule that fails open, because the one that
fails open is the one that decides whether any of the twelve mattered.

List what was not checked and what it would have covered. If you wrote or changed
any file to build the harness, list every one, what it does, and confirm that
nothing in it reaches a release build.
