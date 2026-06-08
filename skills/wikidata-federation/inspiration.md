# Inspiration / Origin

This file records the canonical context that originated the
`wikidata-federation` skill. Kept separate from `SKILL.md` so the skill
stays portable to other authors who do not share this origin context.

## Origin

The workflow encoded in this skill was extracted from federating the
identifiers of the author's own DOI-registered sibling research
repositories onto Wikidata — [`agent-knowledge-cycle`](https://github.com/shimo4228/agent-knowledge-cycle)
(DOI 10.5281/zenodo.19200726), [`contemplative-agent`](https://github.com/shimo4228/contemplative-agent)
(DOI 10.5281/zenodo.19212118), [`agent-attribution-practice`](https://github.com/shimo4228/agent-attribution-practice)
(DOI 10.5281/zenodo.19652013), and the doctrine line
[`authorship-strategy`](https://github.com/shimo4228/authorship-strategy).
Each repository already carried an ORCID-bearing author identity and a
DOI; the missing piece was the public knowledge-graph node that lets the
rest of the open-data ecosystem (Scholia, SPARQL consumers, the CC0
dump, citation bots) find and cite them without further author effort.
The skill consolidates the create-and-cross-link disciplines that work
shares, with explicit safeguards against the incidents the author hit
while doing it by hand.

## The motivating constraint

Wikidata is a shared, public, human-and-bot-edited database where a
mistaken write is expensive to undo: a duplicate item needs
administrator rights to delete, and a malformed claim propagates into
the CC0 dump before it can be corrected. The skill therefore leads with
constraints rather than capability. The two load-bearing safeguards —
mandatory deduplication before any creation (Phase 0), and full-payload
human approval before every write — were both extracted from the
recognition that the cost of a bad write is asymmetric to the cost of a
check. The pitfalls table in `SKILL.md` records the specific
constraint-violation warnings encountered during real registration runs
(the `P275`/`P6216` copyright-status pairing, the `P1324` required
qualifiers, the `P50`-vs-`P178` conflict on software items) so that the
payload generator can enforce them mechanically rather than relying on
the operator to remember them.

## The self-promotion boundary

A second discipline was extracted from a deliberate restraint rather
than an incident: the skill refuses to create Wikidata items for the
author's own coined concepts and terms. Without an independent source,
such an item reads as self-promotion and is liable to deletion under
Wikidata's notability norms; more importantly, minting it would move the
concept's definitional authority out of the author's own normative layer
(the repository's `graph.jsonld`, where the author controls the
definition) and into a database the author does not control. The skill
keeps Wikidata to the bibliographic skeleton — authors, works, DOIs —
and leaves concept definitions in the repository graph.

## Canonical doctrine repository

The skill operationalizes the identifier-federation discipline of the
upstream research line:

> [`authorship-strategy`](https://github.com/shimo4228/authorship-strategy)

The doctrine repository contains the normative articulation of the
framework; this skill is the operational layer that makes the
Wikidata-federation step of that framework executable without requiring
the agent (or its operator) to re-derive the property conventions and
safety gates on every registration.

## Lineage to existing skills

The skill explicitly hands off to other ecosystem skills at its
boundaries:

- It runs *after* [`claude-skill-release-doi`](https://github.com/shimo4228/claude-skill-release-doi):
  that skill mints the DOI; this skill federates it onto Wikidata.
- Its Phase 4 injects QIDs into the `graph.jsonld` whose schema is
  designed by [`claude-skill-jsonld-knowledge-graph`](https://github.com/shimo4228/claude-skill-jsonld-knowledge-graph).
- After it updates a `graph.jsonld`, the repository's own mirror-sync
  workflow (an `hf-sync`-style step) propagates the change to Hugging
  Face Datasets.

When the dependencies are absent the skill still functions on its own,
but it is one component of a federation workflow rather than a
standalone tool.

## When this skill becomes obsolete

This skill is a scaffold. It loads a specific class of
identifier-federation workflow into a specific class of harness, and the
workflow depends on a specific substrate (the Wikidata `wbeditentity`
API, the Zenodo concept-vs-version DOI model, the schema.org-flavored
`graph.jsonld` shape). When scholarly-identifier infrastructure shifts
substantially — for example, if DOI registries begin emitting
knowledge-graph nodes directly, or if a successor to Wikidata's property
model emerges — this skill should be retired in favor of whatever the
new substrate supports. The identifier-federation *discipline* (one
artifact, many identifiers, all mutually resolving) is the part the
framework commits to preserving across substrate shifts; the specific
property numbers and API calls in this skill are not.
