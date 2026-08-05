---
name: licentry-integration
description: The shared contract every licentry-integrate command runs on. Load it before /setup, /implement, /harden, /verify, /audit, /debug or /check-for-updates, and load it any time you are writing, reviewing or debugging code that calls a Licentry session endpoint (activate, heartbeat, refresh, logout, validate), pinning a response signing key, verifying X-Licentry-Sig, handling an offline grace token, computing a device hash, or turning on a product control such as requireDpop or requireBuildToken. It carries the six rules that fail silently, the rule that documentation is fetched live and never remembered, and the refusal rules that stop a command producing an answer out of nothing.
---

# The contract this plugin runs on

A Licentry integration that "works" is not the goal. An integration that fails
loudly when something is wrong is the goal, because the failures that matter
here are the ones that never announce themselves. Four of the six rules below
pass every test a vendor will ever run, ship, and protect nothing for the life
of the product.

You are working in a vendor's own repository. The vendor is the paying customer
whose product you are licensing. Their end customers are users. A product is the
thing the vendor licenses, a licence or key is what the user redeems, a session
is the live runtime check, a build is a watermarked artifact, a control is a per
product security toggle. Keep those words straight in everything you write,
including code comments, because the vendor reads them.

## Documentation is fetched, never remembered

Everything about the wire protocol lives on Licentry's own pages and is fetched
at the moment you need it. This plugin deliberately carries almost no protocol
facts, because a copied fact is a fact that drifts, and a client built from a
stale copy is exactly the failure this plugin exists to prevent.

Read `references/fetch-protocol.md` before the first fetch of any command. In
short: fetch the manifest at

```
https://licentry.cc/plugin/manifest.json
```

first, then fetch only the pages the manifest names, and check every fetch
landed on real documentation before using a word of it. A page that will not
fetch stops the command. It does not degrade into working from memory.

The six rules below are the one exception, held locally because they are the
reason this plugin exists. They are also published at
`https://licentry.cc/docs#quickstart`. If the fetched page and this file
disagree, **the fetched page wins**, tell the vendor which line differed, and
report it to support@licentry.cc. A disagreement here is a defect on our side
and the vendor should not be the one absorbing it.

## The six rules that fail silently

Every command that touches the integration checks all six. `/audit` and
`/verify` refuse to pass an integration that misses one. There is no severity
ordering to negotiate: each of these is the difference between a security
control and a decoration.

**1. Pin the signing key at build time. Never fetch it at runtime.**
Bake in every key the JWKS returns, as a map of `kid` to public key. Whoever
owns the machine owns its DNS, its hosts file and its trust store, so a client
that fetches its trust anchor at startup gets handed the attacker's key and
then verifies every forged response perfectly. Fetching the anchor you use to
decide whether to trust the answer is ceremony, not verification. The JWKS
endpoints exist for build time and for rotation.

**2. An absent signature is a refusal.**
If `X-Licentry-Sig` is missing, stop, and do not read the body. There are real
paths on which the server sends no signature header at all: a product using its
own signing key whose account key cannot be loaded is answered unsigned on
purpose, because a signature from a key the client did not pin is
indistinguishable from a forgery. So "no header" is a state the client will
genuinely meet, and it is also what an attacker produces by stripping one
header. `if (response.has(SIG)) { verify }` is one line of plausible defensive
code that fails open on every stripped header, silently, forever.

**3. Verify the signature correctly.**
Base64 decode the header. Require exactly 64 bytes of raw `r||s`, not DER.
Require the timestamp within 300 seconds of now, checked on **both** sides,
because a one sided check is defeated by moving the clock back. Select the key
by `kid`, and treat a `kid` you do not pin as a failure rather than a skip.
Then verify ECDSA P-256 over the ASCII string
`"<status>|<ts>|<lowercase sha256 hex of the exact raw body bytes>"`.
**It does not sign the body.** Hashing the parsed and re-serialised JSON, or
verifying over the body bytes directly, fails every response and usually gets
diagnosed as a wrong key.

**4. Compare the nonce echo before reading the status.**
Send a fresh random `clientNonce` on every call and reject the response if the
echo is missing or different, before any branch on the status code. The
signature binds a response to its own body, not to this request, so without the
echo one captured signed success replays forever against the vendor's own
client.

**5. Gate on `accepted`, never on `verify`.**
A verified 403 is still a refusal. A helper that returns only the body invites
`if (verify(res)) { unlock() }`, which turns an authentic refusal into an
unlock. Verify, then branch on the status, then read a field, and make sure no
catch-all branch reaches the licensed path.

**6. Build no oracle.**
Never map a refusal to a licence state in the UI, in logs, in exit codes or in
timing, and never write a retry policy that varies by refusal reason. Activate
answers one generic refusal covering unknown, expired, revoked, suspended,
flagged and canary keys, on a padded schedule, so a stolen key list cannot be
sorted into live and dead. A client that guesses at the reason hands that
distinction straight back. This is a deliberate anti-oracle decision, not an
omission in the response.

### Before all six: a device hash that does not move

This is a precondition rather than a rule, and it is the only item whose
failure lands on the vendor's paying customer instead of on the vendor.

An unstable device hash does not refuse the request. It **revokes the session**,
records a device mismatch that the sharing evidence view reads, and answers
`403 license_suspended`. So a recipe that shifts between runs cuts a paying user
off mid-session and manufactures evidence that they were sharing their key.

It fails after a reboot, a dock, a driver update or a change of power source. It
does not fail on the second test run, which is why it reaches production. Derive
it from identifiers that do not move, emit identical lowercase hex on every
call, and tell the vendor in writing to test it across a reboot before shipping.

## Never centralise the verdict

Do not write, and do not leave in place, a single `IsValid()`, `isLicensed`,
`checkLicense()` or equivalent that the whole product consults. One symbol is
one edit away from returning a constant, and that defeats every piece of
cryptography on the wire. Check in more than one place and at more than one
moment, and make the program need the session's answer rather than consult it:
values delivered inside the session that the product cannot invent are what
survive the check being patched out.

If the vendor asks for a single entry point because it is tidier, say plainly
what it costs and let them decide. Do not quietly build it and do not quietly
refuse.

## Say what you wrote

Every command that edits a file in the vendor's repository lists, at the end of
its run: each file it created or changed, what changed, and which rule or
requirement the change serves. Never edit silently, never bundle an unrelated
improvement into the same run, and never write a licence key, an API key or any
other secret into source.

## Refuse rather than guess

Stop, and say exactly which of these it was, when:

- the manifest will not fetch or is not valid JSON in the expected shape
- a page the command needs does not return the documentation
- `/implement`, `/harden` or `/verify` is run with no plan file
- a question in `references/vendor-questions.md` has no answer and the code
  cannot be written without it
- the documentation and this skill disagree about a rule
- something the command needs to check is unreachable, for example a page that
  requires a signed-in vendor session

A command that produced nothing and said why is a good outcome. A command that
produced a plausible integration from an assumption is the outcome this plugin
was built to make impossible.

## The plan file

`/setup` writes `.licentry/plan.md` in the vendor's repository. Every other
command that touches the integration reads it. Its shape and its rules are in
`references/plan-file.md`. If it is absent, `/implement`, `/harden` and
`/verify` stop and point at `/licentry-integrate:setup`.

## References

Read the one you need, when you need it. Every path below is relative to this
skill's own directory, not to the vendor's repository.

| File | Read it when |
|---|---|
| `references/fetch-protocol.md` | Before the first fetch of any command. The manifest, the checks on every fetch, and reading a large page in full. |
| `references/codebase-survey.md` | `/setup`, and `/audit` when there is no plan. The checklist for finding every site the integration touches. |
| `references/vendor-questions.md` | `/setup`. The questions the implementation cannot proceed without, and why each one blocks. |
| `references/integration-checklist.md` | `/verify` and `/audit`. The shared pass and fail checklist, and what counts as evidence. |
| `references/plan-file.md` | Any command that reads or writes the plan. |
