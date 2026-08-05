# Surveying the vendor's codebase

A licensing integration is not one file. It is a session that starts when the
program starts, beats while it runs, ends when it exits, and gates things
scattered across the product. Every one of those sites has to be found before a
line is written, because the ones that get missed are the ones that fail open:
an exit path with no logout holds a concurrency seat for days, and a feature
nobody gated is a feature that works in an unlicensed copy.

This is an audit, not a glance. Work through every item. Record the answer even
when it is "none found", because "none found" is what tells `/implement` it has
to create something rather than extend it.

For every item, record **file paths and line numbers**, not impressions. The
plan is read later by a command that will edit those exact lines.

## The checklist

### 1. Where the program starts
Every entry point, not just the obvious one. Main, service start, worker
process, CLI subcommand, plugin host, test harness. A product with two entry
points and one activation call has one entry point that runs unlicensed.
Record: each entry point, and whether it runs long enough to hold a session.

### 2. Where the program exits
Clean shutdown, user sign-out, window close, signal handler, crash handler.
Logout has to be called on the clean paths, and the vendor has to be told which
paths cannot call it. A session that is never logged out holds its concurrency
seat until its refresh window closes, which is days, and on the common cap of
one that means the user's own next launch evicts them.
Record: each exit path, and whether it is reachable synchronously enough to make
a network call.

### 3. What is being protected
The features, assets, models, algorithms or data that the licence is supposed
to gate. This is the item most surveys skip and it is the one that decides
whether the integration is worth anything, because a check the program does not
need is a check that can be removed.
Record: each protected capability, where it is invoked, and what it consumes
that could plausibly be delivered inside the session instead of compiled in.

### 4. Existing licence, entitlement or authentication code
Anything already doing this job: a previous licensing vendor, a home grown key
check, a login, a trial timer, a feature flag service.
Record: what it is, what calls it, whether it stays alongside Licentry or is
replaced, and above all whether it centralises a verdict in one boolean that the
rest of the product consults. If it does, that is a finding for the plan, not
something to quietly extend.

### 5. The HTTP client
Which library, how it is configured, whether TLS verification has been touched,
whether there is an interceptor layer, and crucially **whether the raw response
bytes are reachable**. Signature verification hashes the exact bytes that
arrived, so a client that only ever hands back a parsed object cannot verify
anything and has to be worked around before any of this is possible.
Record: the library, where a request is built, and how to get raw body bytes and
response headers out of it.

### 6. Where a secret can live on this platform
The OS keystore or equivalent: DPAPI or CNG on Windows, Keychain on macOS,
libsecret or the kernel keyring on Linux, Keystore on Android, Keychain on iOS.
Tokens go there, not in a file next to the binary. If the product ships to a
platform with no keystore, that is a constraint the plan has to state plainly
rather than a detail to settle later.
Record: the store, the API to reach it, and what the product already keeps there.

### 7. The build system
How a release is built, how a version number gets in, and where a constant can
be embedded. Two things have to be embedded at build time: the pinned signing
keys and, once the vendor uses them, the build token. Both are per release.
Record: the build entry point, how a generated constant reaches the binary, and
whether the release pipeline can run one extra step.

### 8. Threading and process model
Refresh has to be serialised, because two threads refreshing at once produce a
reused refresh token, and reuse is the platform's cleanest key sharing signal.
Heartbeats have to be serialised for the same reason. And a `runId` is one value
per process start.
Record: how the product does background work, and where a single owner for the
session loop can live.

### 9. Every surface that could become an oracle
The UI strings, the log statements, the exit codes, the crash reporter, the
support diagnostics, the telemetry. Any of them can leak which refusal happened,
and that is the distinction the server spends a padded delay removing.
Record: where licensing failures would be shown, logged or reported, and who
controls the wording.

### 10. Configuration
Where a base URL, an environment switch or a timeout would live. The API origin
has to be configurable rather than compiled in as a literal, and no secret goes
anywhere near the client.
Record: the config mechanism, and whether it is user editable at runtime, which
changes what an attacker can point the client at.

### 11. The offline story
Whether this product is ever expected to run without a network, and what it
currently does when a call fails. A timeout is not a licensing verdict, and a
product with no offline path stops working the first time a user loses wifi.
Record: the current behaviour on network failure, and what the vendor wants it
to be.

### 12. Tests, and how to run one
Whether there is a test suite, how it runs, and whether a fake or proxied HTTP
layer is available. `/verify` needs somewhere to exercise a stripped header and
a broken nonce chain, and a product with no seam for that gets manual checks
instead.
Record: the test command, the test framework, and any existing HTTP test double.

## What the survey produces

A section in `.licentry/plan.md` with one entry per item, each carrying file
paths and line numbers, or the words "none found" and what that implies for the
implementation. Anything that turned out to be a defect in the vendor's existing
code, a centralised verdict most often, goes in the plan as a finding with its
location, and is reported to the vendor at the end of the run rather than fixed
in passing.
