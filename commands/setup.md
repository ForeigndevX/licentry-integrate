---
description: Start here. Reads Licentry's live documentation in full, audits your codebase for every place licensing has to touch, asks you the few things it cannot work out on its own, and writes a plan the other commands build from. Changes no code.
argument-hint: "[path to the product, if not the current directory]"
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit
---

The vendor wants a Licentry integration and has not read the documentation.
Your job in this command is everything that has to happen before a line of code
is written. You will not write any integration code here. `/implement` does
that, from the plan you leave behind.

Load the `licentry-integration` skill first. It carries the six rules, the
terminology and the refusal rules, and everything below assumes it.

Work in `$ARGUMENTS` if given, otherwise the current directory.

## 1. Read the documentation

Follow `references/fetch-protocol.md` exactly. Fetch the manifest, then fetch
every page whose `requiredBy` includes `setup`, and read each one **in full**.
Not a summary. The parts a summary drops are the parts that fail silently, and
this is the one command that reads everything so the other six do not have to.

Any fetch that fails a check stops the command. Say which URL and which check,
and do not continue from memory.

Tell the vendor, in one line, which pages you read and their revisions. It is
the only evidence they have that this ran against the current contract.

## 2. Survey the codebase

Work through every item in `references/codebase-survey.md`. All twelve. This is
an audit with a checklist, not a look around: record file paths and line
numbers, and record "none found" where that is the answer, because "none found"
is what tells `/implement` to create rather than extend.

If the repository is large, say what you searched and what you excluded. A
survey that silently skipped a subdirectory produces an integration that misses
an entry point.

## 3. Ask the questions

Now ask the vendor the questions in `references/vendor-questions.md`, in one
numbered batch, each with its reason, each phrased against what you just found
in their code rather than in the abstract.

Two of them have no default and you may not proceed on a guess: what is being
protected, and the device hash inputs. For the rest, propose a default and say
you are proposing it.

Wait for the answers. If the vendor answers some and not others, write down what
is unanswered and what each unanswered question blocks, and say clearly that
`/implement` will stop on those.

## 4. Write the plan

Write `.licentry/plan.md` using the template in `references/plan-file.md`:
the documentation revisions you read, the survey with its paths and line
numbers, every answer with its source, the unanswered questions, and what will
be built where.

Creating that file is the only write this command makes.

## 5. Report

- Which pages you read, with revisions.
- What the survey found, the short version, and anything in their existing code
  that is a problem: a centralised verdict, a committed secret, an HTTP layer
  that cannot produce raw bytes. Report these. Do not fix them here.
- What is still unanswered and what it blocks.
- What happens next: `/licentry-integrate:implement` builds from the plan, and
  the vendor should read the plan first because it is theirs.
