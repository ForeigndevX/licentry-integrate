# The checklist `/verify` and `/audit` share

Same list, different depth and different evidence. `/verify` exercises the
integration and reports what it ran. `/audit` reads the integration and reports
what it found. Both use the words below for every item, and both refuse to pass
an integration that fails any of section A.

## The four words

| Word | Means |
|---|---|
| **Pass, executed** | The failure case was actually produced and the client refused. Only `/verify` can award this. |
| **Pass, inspected** | The code was read and it is correct. Name the file and line. |
| **Fail** | The code is wrong, or the executed case did not refuse. Name the file, the line and the consequence in one sentence. |
| **Not checked** | Nothing could be read or run for this item. Say why. Never round this up to a pass. |

A report with items marked "not checked" is an honest report. A report that
quietly omits them is worse than no report, because the vendor reads the absence
as clean.

## Section A. The six rules, and the precondition

Any fail here means the integration does not pass. There is no aggregate score
that survives one of these, because each of them is the whole security value of
everything above it.

**A1. The signing key is pinned at build time, not fetched at runtime.**
Look for a request to a JWKS URL anywhere on the startup or session path. Look
for a key read out of a config file, an environment variable or a downloaded
document, all of which are the same failure wearing different clothes: the key
must be embedded in the artifact and be a map of `kid` to key rather than a
single key, or the next rotation stops every shipped copy.
Executed check: block the JWKS host and confirm the client still verifies, and
point it at a server offering a different key and confirm it refuses.

**A2. An absent signature is a refusal.**
Grep the client for a conditional wrapped around the verify call. Anything of
the shape `if (sig_header_present) { verify }` fails this item even if every
other line is perfect.
Executed check: strip `X-Licentry-Sig` from a real response with a local proxy
and confirm the licensed path does not run.

**A3. The signature is verified correctly.**
Base64 decoded, exactly 64 bytes of raw `r||s` and rejected if not, timestamp
checked on both sides, key selected by `kid` with an unknown `kid` refused, and
the verification performed over the canonical string of status, timestamp and
the lowercase hex sha256 of the exact raw body bytes. A client that hashes a
re-serialised body, or verifies over the body itself, fails.
Executed check: alter one byte of the body in transit and confirm refusal; move
the signature timestamp outside the window in both directions and confirm both
refuse.

**A4. The nonce echo is compared before the status is read.**
Both halves matter. A client that compares the echo after branching on the
status has already acted on an answer to somebody else's question.
Executed check: replay a captured valid response against a fresh request and
confirm refusal.

**A5. The gate is on the outcome, not on "it verified".**
Find every call site that reaches the licensed path. A helper returning only a
body, or a boolean that means "the signature checked out", is the failure. A
verified 403 is a refusal.
Executed check: return a verified refusal and confirm the licensed path does not
run.

**A6. There is no oracle.**
Walk every surface the survey listed: UI strings, logs, exit codes, crash
reports, telemetry, and the retry policy. A refusal must not be mapped to a
licence state anywhere, and retry behaviour must not vary by reason. A distinct
log line per refusal reason is an oracle, because logs are read by whoever owns
the machine.

**A0. The device hash is stable.**
Read the recipe and name every input. Anything that changes with power state,
network interface enumeration order, a docking station, a driver update or a
container restart fails. Confirm it is emitted as lowercase hex, identical on
activate, heartbeat and refresh.
Executed check: compute it twice across a reboot. If a reboot is not possible in
the environment, this is "not checked" and the vendor is told to do it before
shipping, in writing, because the cost of getting it wrong lands on their paying
user rather than on them.

## Section B. Shape, not wire facts

**B1. No centralised verdict.** One symbol that the whole product consults is one
edit from a bypass. Report the symbol and its call sites.

**B2. The program needs the answer.** Ask the question the documentation asks: if
the licensing call were stubbed out to return success, would the product still
work correctly? If yes, say so plainly. It is not a fail of a rule, it is the
finding that decides whether any of this was worth building.

**B3. Verification is not decorative.** If the client would still work with
signature verification commented out, the verification is theatre. This is
worth stating separately from A1 to A3, because a client can pass all three and
still never let the result reach a decision.

**B4. Tokens and keys are in the platform keystore**, not in a file beside the
binary, and nothing sensitive reaches logs or crash reports.

**B5. The API origin is configurable** and no secret is compiled into the client.

## Section C. The lifecycle, against the live documentation

Do not check these from memory. Fetch the endpoint reference, the failure table
and the lifecycle section, and check the client against what they say today.
The items below are the shape of the mistakes, and the current numbers, fields
and status codes come from the page.

- Every request body carries exactly the fields the page lists and nothing else.
  The bodies are strict and an extra field is a refusal on every call.
- The sequence number is handled the way the lifecycle section describes across
  a refresh, and the nonce chain is handled the way it describes across the same
  refresh. These two differ from each other and a client that treats them alike
  breaks on the first beat after every refresh.
- A retry of a lost beat resends the identical body rather than advancing
  anything.
- Refresh is serialised, runs on a schedule rather than on an error, and is
  retried the number of times the page says and no more.
- Every refusal in the failure table has a distinct, correct handler, and the
  terminal ones do not re-activate. Re-activating on everything is the shortest
  error handler and it is wrong on several rows.
- Logout is called on the exit paths the survey found.
- Activate sends an idempotency key generated per attempt and reused only across
  retries of that attempt.
- The protocol version header is sent on every call.
- The offline grace token is verified against the pinned key, and its expiry,
  device claim and revocation claim are all enforced. The device claim is the one
  that turns the token into a file a user can copy to ten machines if it is
  skipped.
- Rate limit responses are backed off with jitter, and a transport failure is
  never read as a licensing verdict in either direction.

## Reporting

Group by section, one line per item, with the word, the location and the
consequence. Then, at the top, the answer to the only question the vendor
actually has:

- **Does it pass**, yes or no. No if any section A item is a fail.
- **What to fix first**, ordered by what fails open soonest.
- **What was not checked** and what it would have covered.
