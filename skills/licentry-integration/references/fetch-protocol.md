# Fetching the documentation

Licentry's documentation is the contract, and it changes. This plugin therefore
ships no copy of it. Every command reads the live pages at the moment it runs,
which is why a plugin installed six months ago still writes a current client.

Live URLs on their own have one failure mode, and it is the reason for the
manifest: a page that moves returns a 404, and an agent that fetched it and
carried on has produced an integration from nothing while looking like it
worked. Stale content is recoverable. Content that was never read is not.

So: **fetch the manifest first, fetch only what it names, and check every
fetch.**

## The manifest

```
https://licentry.cc/plugin/manifest.json
```

That URL is stable. If it ever has to move, the plugin is the thing that gets
updated, not the vendor's assumptions.

```json
{
  "manifestVersion": 1,
  "docsRevision": "2026-08-04.1",
  "baseUrl": "https://licentry.cc",
  "support": "support@licentry.cc",
  "plugin": {
    "latestVersion": "0.1.0",
    "minSupportedVersion": "0.1.0",
    "marketplace": "https://github.com/licentry/licentry-integrate",
    "notes": "https://licentry.cc/docs"
  },
  "pages": [
    {
      "id": "docs",
      "title": "Integration documentation",
      "url": "https://licentry.cc/docs",
      "covers": "Quickstart, the session endpoints and their exact fields, response signature verification, DPoP, offline grace, device binding, the product controls, the failure table and the launch checklist.",
      "access": "public",
      "revision": "2026-08-04.1",
      "requiredBy": ["setup", "implement", "harden", "verify", "audit", "debug"],
      "mustContain": ["Launch checklist", "X-Licentry-Sig"],
      "sections": [
        {
          "id": "quickstart",
          "title": "Quickstart",
          "url": "https://licentry.cc/docs#quickstart",
          "covers": "The four calls, and the six rules that fail silently."
        }
      ]
    }
  ]
}
```

Field by field:

| Field | What it is for |
|---|---|
| `manifestVersion` | The shape of this file. A number this plugin does not know is a stop, not a guess. |
| `docsRevision` | Moves whenever any page changes. `/check-for-updates` compares it against the value the plan recorded. |
| `plugin.latestVersion` | The newest published plugin version. `/check-for-updates` compares it against the installed one. |
| `plugin.minSupportedVersion` | Below this the protocol has moved far enough that the installed commands are wrong. Stop and tell the vendor to update before doing anything else. |
| `pages[].url` | The only URLs any command fetches. Never construct one. |
| `pages[].covers` | What the page holds, so a command can pick the pages it needs without fetching all of them. |
| `pages[].access` | `public`, or `vendor-session` for a page behind the dashboard sign-in. |
| `pages[].revision` | Per page, so `/check-for-updates` can name what changed instead of saying "something did". |
| `pages[].requiredBy` | Which commands must read this page. A command that skipped one of its own required pages has not finished. |
| `pages[].mustContain` | Short literal strings that appear on the real page. This is how a fetch is checked. |
| `pages[].sections` | Anchors within a large page, so a command can read it in parts without losing anything. |

## Checking a fetch

Do all three. The first two alone let a redirect to a marketing page through.

1. **The fetch returned content.** An error, a timeout or an empty result stops
   the command.
2. **Every string in `mustContain` is present** in what came back. A sign-in
   wall, a 404 page and a redirected landing page all fail this, which is the
   point. Missing markers mean you did not receive the documentation, whatever
   the status code said.
3. **The content answers the question you fetched it for.** If you fetched the
   endpoint reference and got a page with no endpoints on it, that is a failed
   fetch even if the markers matched.

When a fetch fails, stop the whole command and tell the vendor:

- which URL failed and which check it failed
- that the plugin does not work from remembered documentation, deliberately
- to report it to the support address in the manifest, since a page that moved
  without the manifest moving is a defect on Licentry's side

Do not try a different URL. Do not fall back to what you know about the
protocol. Do not continue with the pages that did fetch unless the command says
in so many words which of its work survives a missing page.

## Reading a page in full

`/setup` reads its pages in full rather than in summary, because the parts a
summary drops are the parts that fail silently. Two things make that work on a
page of this size:

- Fetch with a prompt that asks for the **complete text** of the sections you
  need, quoted rather than described. A prompt like "summarise the signature
  section" gets you a summary and you will not notice what it left out.
- Use `sections` to walk a large page anchor by anchor when one fetch will not
  carry the whole thing. The anchors are in the manifest so you never have to
  guess an anchor name.

You have read a page when you can state, without going back, what it says about
the six rules, the exact fields of each call you are about to write, and the
failure branches. If you cannot, you have skimmed it.

## Pages behind the dashboard

A page marked `"access": "vendor-session"` needs a signed-in vendor and cannot
be fetched by this plugin at all. Do not attempt a sign-in and do not ask for
credentials.

Instead: tell the vendor which page it is, what the command needs from it, and
ask them to open it in their browser and either paste the relevant section or
save the page to a local path.

**That page is behind a sign-in for a reason, so it does not go into version
control.** Ask them to save it outside the repository, or somewhere they are
certain is ignored and never published, and say why: it describes what Licentry
detects and how much slack each protection allows, and a copy of it in a public
repository is a copy an attacker reads. Record only the path in the plan under
`Documentation read`, never the content. If the vendor declines,
finish the rest of the command and state explicitly which checks were not made
and what they would have covered. Silence about an unread page is the failure
this whole file exists to prevent.

## Recording what you read

Every command that writes to the plan records, under `Documentation read`: the
`docsRevision` from the manifest, and for each page fetched, its `id`, `url`,
`revision` and the timestamp. That record is what `/check-for-updates` compares
against, and it is what tells a later reader whether the integration was built
against the current contract.

## The one thing never to do

There is a copy of this manifest shape in Licentry's public repository, for
whoever maintains the manifest. It is a description of the format. It is never a
substitute for the live fetch. A command that fills in the gaps from an example
file, from this skill, or from what the model already knows about the protocol
has produced an answer from a file instead of from the documentation, and that
is precisely the failure the manifest was added to make impossible.
