# Note taxonomy — the kinds of notes & when to write each

The knowledge base is useful months later only if each piece of knowledge lands in the **right kind of
note**. Seven types, two "modes" (learn vs operate):

| # | Type | Mode | Lives in | Length | When to write it | Shape |
|---|---|---|---|---|---|---|
| A | **Learning note** | learn | level folders (`04-networking/…-deep-dive.md`) | long | to *understand* a concept | the teaching format (concept→why→internals→example→org example→arch→commands→debug→prod→security→mistakes→takeaways→exercise) |
| B | **Cheatsheet** | operate | `cheatsheets/` | 1 screen | when you'll need an answer *in seconds* | a `Problem → Command` table, nothing else |
| C | **Runbook** | operate | `runbooks/` | medium | a repeatable *procedure* (push an image, rebuild the custom pg base) | numbered steps + copy-paste commands + verify step |
| D | **Troubleshooting guide** | operate | `troubleshooting/` | medium | a *failure mode* seen more than once | symptom → decision tree → root cause → fix → prevention |
| E | **ADR** (architecture decision) | learn | `adr/` | short | a *decision with trade-offs* (Alpine vs distroless; DB in K8s?) | Context · Options · Decision · Consequences |
| F | **Case study** | learn | `case-studies/` | medium | a *per-service* deep dive (platform-demo, postgresql-noavx512) | the real Dockerfile line-by-line + why + what to improve |
| G | **Checklist** | operate | `checklists/` | 1 screen | a *gate* you run every time (prod image review, security) | `[ ]` items, each a pass/fail |

## Rules
- **One idea, one note.** A learning note explains; a cheatsheet answers; don't blur them (the user asked
  for A + B explicitly, the rest are additive).
- **Every learning note ends with an org exercise** and links its cheatsheet + troubleshooting sibling.
- **Cross-link, don't duplicate** (see below). The ADR holds the *decision*; the learning note holds the
  *explanation*; the cheatsheet holds the *command*. Link between them.
- **A note without a real project reference is suspect** — if it's pure theory, it belongs upstream in a
  book, not here. This KB's edge is the laboratory.

## Cross-linking convention
At the bottom of every note, a **See also** block:
```
## See also
- Learn → 04-networking/docker-networking-deep-dive.md
- Fast → cheatsheets/networking-cheatsheet.md
- Fix  → troubleshooting/container-cannot-reach-postgres.md
- K8s  → 11-kubernetes/ (Services) · _meta/docker-to-kubernetes-map.md
- Case → case-studies/ktayl-core-billing.md
```
This turns the KB into a graph, not a pile of files. Mirrors the platform's own `[[memory]]` linking +
the `evidence-and-proof.md` three-pillar split (case study / proof / pattern).

## Naming
`NN-<area>/<topic>-<kind>.md` for learning (`04-networking/docker-dns-deep-dive.md`);
`<area>-cheatsheet.md`, `<verb>-<thing>-runbook.md`, `<symptom>.md`, `adr-NNN-<decision>.md`,
`<service>.md` (case study). Lowercase-kebab throughout.
