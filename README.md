# licentry-integrate

A Claude Code plugin that integrates [Licentry](https://licentry.cc) licensing
into your product, so you do not have to read the documentation before you start.

## What Licentry is

Licentry is a licensing platform. Your customer redeems a licence key, your
software exchanges it for a short lived, signed, device bound session, and that
session is what decides whether your product runs. The platform holds the parts
an attacker cannot reach: key storage, session state, rate limits, revocation.
What ships to your customer is your binary, and that is where an integration is
right or wrong.

## What this plugin is for

The client side of that protocol has a handful of checks that fail silently. Get
one wrong and your integration passes every test you will ever run, ships, and
protects nothing for the life of the product. The common one is a single line:

```
if (response.headers["X-Licentry-Sig"]) { verify(response) }
```

That looks like defensive code. It fails open on every response where the header
was stripped, which is every response an attacker touches, and nothing anywhere
tells you.

This plugin knows that list. It reads Licentry's live documentation at the moment
you run it, surveys your codebase, asks you the things it cannot work out on its
own, writes the client into your own files, and refuses to sign off an
integration that misses one of the six.

## Install

From the marketplace:

```
/plugin marketplace add licentry/licentry-integrate
/plugin install licentry-integrate@licentry
```

Or download it from your Licentry dashboard and install the folder locally. Both
are the same plugin.

The plugin needs outbound network access to `licentry.cc`. It reads the
documentation live rather than shipping a copy, so a plugin installed a year ago
still writes a client against today's contract. If it cannot reach the
documentation it stops and tells you, rather than working from memory.

## The seven commands

| Command | What it does |
|---|---|
| `/licentry-integrate:setup` | Start here. Reads the documentation, audits your codebase, asks you what it cannot work out, writes the plan. Changes no code. |
| `/licentry-integrate:implement` | Writes the integration into your files, from that plan. Lists every file it changed. |
| `/licentry-integrate:harden` | Turns your product's controls on, one at a time, in an order that cannot lock out customers already running your software. |
| `/licentry-integrate:verify` | Proves your integration refuses: strips a signature, replays a response, breaks the nonce chain, and reports what actually refused. |
| `/licentry-integrate:audit` | Reviews an integration this plugin did not write, from the source. No plan needed. |
| `/licentry-integrate:debug` | Takes one refusal you cannot explain and works out which condition produced it. |
| `/licentry-integrate:check-for-updates` | Tells you whether the documentation or the plugin has changed since your plan was written. |

The usual path is `setup`, `implement`, `verify`, ship, then `harden` once your
customers have the new build. `audit` and `debug` are for the days that follow.

**`verify` or `audit`?** `verify` runs your integration against failure cases and
needs a plan and a client it can execute. `audit` reads code it did not write and
needs neither. If you built it with this plugin, you want `verify`.

## What it does to your code

`/implement` and `/verify` write into your repository. Everything else reads.

Every command that changes a file lists what it changed and why at the end of the
run. Nothing is edited silently. You review it like any other change, and you own
the result: the code is yours, in your files, in your version control, with no
runtime dependency on this plugin.

`/setup` writes one file of its own, `.licentry/plan.md`. Keep it. The other
commands read it, and it records which revision of the documentation your
integration was built against, which is worth having the day something stops
working.

## What it will refuse to do

By design, and these are the point of it rather than rough edges:

- Run `/implement`, `/harden` or `/verify` with no plan.
- Continue when a documentation page will not fetch. It stops and names the page,
  because an agent that carried on would be producing an integration out of
  nothing while looking like it worked.
- Proceed past an unanswered question that blocks a decision, in particular what
  you are protecting and what your device hash is derived from. Neither has a
  safe default.
- Pass an integration that misses one of the six rules, however much else is
  right.
- Write anything that tells a user, a log file or an exit code *why* a licence
  was refused. Licentry answers one generic refusal on purpose, so a stolen key
  list cannot be sorted into live and dead, and a client that guesses hands that
  back.

## The opinion it holds

Six rules, published at [licentry.cc/docs#quickstart](https://licentry.cc/docs#quickstart)
and enforced by every command here:

1. Pin the signing key at build time. Never fetch it at runtime.
2. An absent signature is a refusal.
3. Verify the signature correctly, over the canonical string and not the body.
4. Compare the nonce echo before reading the status.
5. Gate on the outcome, never on "it verified". A verified 403 is still a refusal.
6. Build no oracle.

And one precondition underneath them: a device hash that does not move. An
unstable one does not refuse a request, it revokes the session and records a
device mismatch against a paying customer.

## What you need before you start

- A Licentry vendor account and a product. `/setup` will ask, and will not invent
  a product that does not exist.
- A test licence key, in your environment or your secret store. Never in the
  repository, and the plugin will tell you if it finds one there.
- Whichever language your product is written in. The client side is plain HTTPS
  and JSON, and the only primitive ever in doubt is ECDSA P-256 verification. The
  documentation lists which languages have it in the standard library, and the
  generated code says which package yours needs if it needs one.

## Support

The documentation is at [licentry.cc/docs](https://licentry.cc/docs). Questions
about the platform, and anything this plugin got wrong, go to
support@licentry.cc.

Licensing enforcement raises the cost of a bypass. It does not make one
impossible, and no client running on a machine somebody else owns ever will. This
plugin is built to be honest about which is which.
