# The questions the implementation cannot proceed without

These are not a preference survey. Each one blocks a decision that cannot be
made from the codebase alone, and each has a wrong answer that ships silently.
Ask them after the survey, so you are asking about code you have already read
and can be specific: "you have three entry points, which of them is allowed to
block on a network call" beats "how does your app start".

Ask them in one batch, numbered, with the reason next to each. A vendor answering
eight questions in one message is a two minute conversation. Eight separate
round trips is an afternoon, and they stop answering.

Two of these have no acceptable default and you cannot proceed on a guess:
**what is being protected**, and **the device hash inputs**. If the vendor says
"whatever you recommend" to either, explain why that one is theirs to answer and
ask again. Everything else has a default you may propose, as long as you state it
as a proposal and record their acceptance in the plan.

## 1. What are you actually protecting?

Name the capability, asset, model, dataset or algorithm the licence is meant to
gate.

**Why it blocks.** It decides where the gates go, and whether the product can be
made to *need* the session rather than *consult* it. Values delivered inside the
session that the product cannot invent, an asset decryption key, coefficient
tables, a solver seed, are what survive somebody removing the check. A product
gated on a boolean is one edit away from being bypassed no matter how good the
cryptography above it is. This answer is also what makes the engine parameters
step of `/harden` possible later, or impossible.

**No default.** Nobody outside the vendor's team knows which part of their
product is worth protecting.

## 2. Do you have a vendor account, a product, and a test licence key?

Three separate things. The product carries the controls and its slug is what the
client sends. The test key is what makes any of this runnable.

**Why it blocks.** Without a product there is nothing to configure and no slug to
send. Without a test key nothing can be exercised, so `/verify` degrades to
reading code, which is what `/audit` is for. Also settle here which API origin
this account was issued, because the client must take it from configuration
rather than a literal.

**If the answer is no**, point them at the dashboard to create the account and
the product first. Do not write an integration against a product that does not
exist yet, and never invent a slug.

**The test key goes in the environment or their secret store, never in the
repository**, and neither does a vendor API key. If one is already committed, say
so as a finding.

## 3. Which language, which HTTP client, and can you get the raw response bytes?

**Why it blocks.** The signature is verified over the exact bytes that arrived.
An HTTP layer that only ever returns a parsed object cannot verify anything, and
finding that out after the client is written means writing it twice. Ask for the
library by name and check that raw bytes and response headers are both reachable
before agreeing an approach.

The same answer decides whether ECDSA P-256 verification is in the standard
library for this language or needs one package, which the requirements section of
the documentation lists. Say which in a comment in the generated code rather than
letting the vendor discover it at build time.

## 4. Which platforms do you ship to, and where does a secret live on each?

**Why it blocks.** The access and refresh tokens, and the DPoP private key if
there is one, go into the platform keystore. A file next to the binary is a file
that gets copied with the binary, which undoes device binding entirely. If a
target platform genuinely has nowhere to put a secret, that is a limitation to
write down in the plan and tell the vendor about, not something to paper over.

**Default you may propose:** the standard keystore for each named platform, taken
from the survey.

## 5. What are your device hash inputs, on each platform?

The concrete list of identifiers the hash is derived from.

**Why it blocks.** An unstable hash does not refuse a request, it revokes the
session and records a device mismatch against a paying user. The inputs have to
survive a reboot, a dock, a driver update and a change of power source, and must
not survive being moved to a different machine. Only the vendor knows what their
target hardware actually offers, and the documentation's device binding section
is what tells them which candidates are stable.

**No default.** Propose candidates from the documentation, get the vendor to
choose, and write into the plan that the recipe must be tested across a reboot
before shipping.

## 6. Does the product have to run offline, and for how long?

**Why it blocks.** It decides whether the offline grace path is written at all,
and what the product does when the window ends. The grace token is device bound
and its lifetime is a platform setting the vendor does not control, so the
question to settle is what their product does at the end of it: degrade, stop, or
run reduced. A product with no offline path stops working the first time a user
loses wifi and the vendor takes the support call.

**Default you may propose:** verify the grace token locally, keep running while it
is valid, retry on a slow backoff, and stop the licensed path when it expires.

## 7. What does the user see when the check fails?

**Why it blocks.** Rule 6. Whatever they say, the client shows one generic
message for every refusal: no reason, no code, no difference in wording, no
difference in retry behaviour. Ask what the message should say and who writes it,
because if you do not, someone later fills the gap with "licence expired" and
rebuilds the oracle the server was built to remove.

**Default you may propose:** one message pointing the user at the vendor's own
support, identical for every refusal.

## 8. Do you bind sessions to a device key now, or later?

Proof of possession, which the documentation covers as `requireDpop`, needs the
client to generate a P-256 key pair into the keystore and send the public half at
activate.

**Why it blocks.** Adding it later sends every session in the field back through
activate, so the cheap moment is now and the expensive moment is after the first
release. The same is true of the body hash claim on the proof: it costs nothing
to include while the proof code is being written and it is a second forced
migration afterwards.

**Default you may propose:** write the client side now, leave the product control
off, and turn it on through `/harden` once the fleet has updated.

## 9. Are there shipped builds you cannot update?

**Why it blocks.** It decides what `/harden` can do at all. Every control in the
sequence locks out a client that does not yet handle it, so a fleet the vendor
cannot update sets the ceiling on how far the sequence can go this year. Better
to know before the plan is written than at the moment a control locks out paying
users.

## Recording the answers

Write every answer into the plan, verbatim enough that a later command can act on
it, along with which answers were the vendor's own and which were a default they
accepted. If a question is unanswered when `/implement` runs, `/implement` stops
and names it. That is the design: an integration built on an assumed device hash
recipe is worse than no integration, because it ships.
