---
description: Turns your product's security controls on, one at a time, in an order that cannot lock out customers already running your software. Each control has something your client must do first, and this checks your code does it before it tells you to flip anything. Run this after your integration ships and works.
argument-hint: "[one control name, to work on just that one]"
allowed-tools: Read, Grep, Glob, WebFetch, Edit
---

Every control in this sequence refuses something. Turned on before the shipped
client handles it, each one refuses your paying customers, and the customers who
notice first are the ones running the oldest build. That is the whole risk here
and it is the reason the order exists.

Load the `licentry-integration` skill first.

## What this command does not do

It does not turn anything on. It works out what is safe to turn on next, proves
your client is ready for it by reading your code, and tells you exactly what to
change and where. **You** flip the control, signed in as the account owner or an
admin seat, on the product page in your dashboard.

That is not caution on the plugin's part, it is how the platform works: the
runtime protections cannot be changed with a vendor API key at all. A key
holding full scopes is refused, and the refusal names the fields it tried to
touch. Confirm the current wording of that on the vendor API page before you
repeat it to the vendor.

This command never asks for a password, an API key or a dashboard session.

## Before anything

Read `.licentry/plan.md`. No plan, no hardening: without it you do not know what
the client does, and the entire job here is deciding whether the client is ready.
Point at `/licentry-integrate:setup`.

Fetch the documentation per `references/fetch-protocol.md`: the manifest, then
every page whose `requiredBy` includes `harden`. You need the controls section
and the launch checklist, and the vendor API page for what each control is
called and what it does.

If the manifest marks the hardening guide as needing a signed-in vendor, you
cannot fetch it. Ask the vendor for it or proceed without it, and say which
parts of your advice are therefore based on the public pages only.

## The order

Confirm this against the live documentation before you use it. If the pages give
a different order today, **the pages win**, and tell the vendor that the plugin
and the documentation disagreed.

1. `requireBuildToken`
2. `requireDpop`
3. `maxConcurrentSessions`
4. `engineParams`
5. `payloadKey`
6. `requireDpopBody`
7. `vendorResponseSigning`
8. `minProtocolVersion`

The first four are the set the launch checklist treats as the baseline. The last
four are the ones that came later and each depends on something the client has to
do first.

## For each control, in order

Do not batch these. One control, then ship, then wait, then the next.

**1. Read what it needs.** Get the control's own section from the live
documentation: what it does, what changes in the client, and the mistake vendors
actually make with it. Do not work from memory and do not work from this file.

**2. Check the client already does it.** Read the vendor's code for the specific
thing the documentation says the client must do first. Report it as a location,
not an impression: the file and line where the header is sent, the key is
generated, the claim is added, the field is read. If you cannot find it, the
answer is no.

**3. Check the field.** A control is safe only when every copy of the client in
use handles it. Ask, or read from the plan: is there a build in the field that
does not? A shipped build that cannot be updated sets the ceiling on this whole
sequence, and it is better to say so now than after a control locks out a user.

**4. Say ready or not ready**, and if not ready, say exactly what to change and
where. Then stop at the first control that is not ready. Working further down
the list is how a vendor ends up turning on control five while control two is
still refusing their own customers.

**5. When it is ready**, give the vendor the steps: what to tick, on which page,
and what to watch afterwards. Then tell them to ship the client first if it is
not already in the field, wait for adoption, and only then flip it.

**6. Record it** in the plan's Controls table: the control, whether the client
prerequisite is in place with its location, what you advised, and the date.

## Two that need saying out loud

**`engineParams` and `payloadKey` are the only two on this list that change what
removing your licence check achieves.** Everything else raises the cost of
removing it. If the vendor seals a boolean, a tier string or a feature flag into
the parameters, they have bought nothing: an attacker opens it once on a machine
with a real licence and writes the values into the patched binary as constants.
The documentation has a worked example of a good parameter set and a bad one.
Read it to them rather than paraphrasing, and apply its test: if the program
still produces correct output with the blob replaced by an empty object, what
was sealed was decoration.

**`vendorResponseSigning` changes which key verifies every response**, refusals
included, and there is no overlap signature to catch a client that pinned the
wrong one. The documentation gives a rollout order. Follow it exactly and do not
compress it, because the failure mode is every shipped copy refusing every
response.

## Report

- Which controls are on now, and where you got that from.
- The next control, whether the client is ready, and the location that proves it.
- What to change if it is not.
- What you could not check and what that leaves uncertain.
