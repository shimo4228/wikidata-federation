# claude-skill-wikidata-federation

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that registers research artifacts — researchers, papers, and DOI-registered repositories — as **Wikidata items** and cross-links them so that **ORCID ↔ Wikidata ↔ DOI ↔ `graph.jsonld`** form one machine-readable, mutually-referencing node. It is the *identifier-federation* step of the [authorship-strategy](https://github.com/shimo4228/authorship-strategy) research line: the point where a freshly deposited DOI stops being an isolated record and becomes a queryable node in the public knowledge graph.

The skill is **harness-aware where it has to be** (it knows the Wikidata `wbeditentity` API, the property/QID conventions for authors vs papers vs software, the Zenodo concept-vs-version DOI distinction, and the `graph.jsonld` `sameAs` shape) but **safety-first throughout**: every write is presented to a human in full before it is sent, deduplication runs before any creation, and credentials live only in environment variables or a `chmod 600` file — never in code or conversation.

## Why federate identifiers

ORCID, Zenodo, and GitHub are three separate islands. A Wikidata item is the bridge between them. Once the bridge exists:

1. **[Scholia](https://scholia.toolforge.org/)** auto-generates an author profile page from the item.
2. **SPARQL** can query every work an author has produced.
3. The **CC0 dump** propagates the node into downstream databases and LLM training corpora.
4. **Citation bots** grow the graph on their own, without further author effort.

This is the post-deposit half of the work: minting a DOI puts a record into one archive; the Wikidata item is what lets the rest of the open-data ecosystem find and cite it.

## When to use

Apply the skill when **any** of the following hold:

- A new paper or DOI-registered repository has just been published and needs a Wikidata node
- An author item needs to be created (or enriched) so that ORCID, DOI works, and a repo's `graph.jsonld` resolve to one identity
- A repository's `graph.jsonld` needs QIDs injected as `sameAs` anchors after the corresponding Wikidata items exist
- A [Scholia](https://scholia.toolforge.org/) author profile is being set up or repaired

It is the natural **post-release continuation** of [claude-skill-release-doi](https://github.com/shimo4228/claude-skill-release-doi) and the paper-deposit workflow: once a DOI is minted, this skill federates it.

Skip the skill when:

- There is no stable external identifier yet (no ORCID, no DOI) — there is nothing to federate
- The artifact is a private or one-shot deliverable not intended for the public knowledge graph
- You want to mint a Wikidata item for a **coined concept or term of your own** — the skill explicitly refuses this. Without an independent source the item is self-promotion and is liable to deletion, and the concept's definitional authority leaks out of your own normative layer. Concepts belong in the repository's `graph.jsonld`; Wikidata is kept to the bibliographic skeleton.

## Install

### Claude Code

```bash
git clone https://github.com/shimo4228/claude-skill-wikidata-federation
cp -r claude-skill-wikidata-federation/skills/wikidata-federation ~/.claude/skills/wikidata-federation
```

### Other harnesses

Adapt the install path to your harness's skill convention. The workflow itself (dedup → metadata resolution → dry-run → create → graph anchoring → verify) does not depend on Claude Code specifically; only the trigger and invocation mechanism is harness-specific.

## Prerequisites

1. **A Wikidata BotPassword** — created at [`Special:BotPasswords`](https://www.wikidata.org/wiki/Special:BotPasswords). The "edit existing pages" + "create, edit, and move pages" grants are sufficient. BotPasswords are **local to the wiki they are created on** (a `test.wikidata.org` password will not work on `www.wikidata.org`, and vice versa).
2. **Credentials via file or env** — `~/.config/wikidata/credentials.env` (`chmod 600`) holding `WIKIDATA_USERNAME=user@botname` and `WIKIDATA_PASSWORD=...`. The file form is the reliable path in harnesses that reset the shell environment between calls.
3. **Python + `requests`** — runnable with `uv run --with requests` if you do not want a managed environment.

## How it works

The skill runs six phases, each with an explicit gate. It refuses to write until the human has seen and approved the exact payload.

0. **Deduplication (mandatory, first)** — search Wikidata by ORCID (`P496`), by DOI (`P356`), and by label before creating anything. A duplicate item is a quality incident, and deletion requires admin rights, so this gate is non-negotiable. If a match exists, the skill switches from *create* to *add missing statements to the existing item*.
1. **Metadata resolution (Zenodo side)** — resolve the **concept DOI vs version DOI** question with the human (concept DOI is the usual canonical choice), pull the **first-version** publication date (the concept record reports the *latest* version's date), and map `resource_type` to the correct `P31` instance (scholarly article / software / dataset).
2. **Dry run** — validate auth, CSRF, and request shape against the official sandbox item `Q4115189` (or `test.wikidata.org` if a BotPassword exists there) by an alias add → remove round-trip, confirming restoration by read-back.
3. **Item creation** — `gen_payload.py` turns a spec JSON into a `wbeditentity` payload that mechanically enforces the property conventions (author vs paper vs software take different claim sets, copyright-status pairs with license, required qualifiers are auto-attached). The payload is shown to the human, and only after approval does `wd_api.py` transmit it. One item per `wbeditentity` call; created items are read back from `Special:EntityData/<QID>.json`.
4. **`graph.jsonld` anchoring** — `add_qid_sameas.py` injects each QID into the `sameAs` of the matching node, matched by exact identity (`@id` / `sameAs` / `identifier` / `url`); a Person node is only matched on ORCID. The edit is **format-preserving text surgery validated by semantic tree comparison** — it never re-serializes the whole file, keeping the diff to a few lines instead of thousands.
5. **Verification and reporting** — emit every QID + URL, the Scholia author URL, confirm search-index propagation via `haswbstatement`, and list the residual manual tasks (adding the QID to the ORCID profile's links, deferring OpenAlex ID until name resolution is settled).

The full property/QID conventions, the empirically-derived pitfalls table, and the script contracts live in [SKILL.md](skills/wikidata-federation/SKILL.md).

## Safety discipline

This is a skill that **writes to a shared public database**, so it leads with constraints rather than capability:

| Discipline | Mechanism |
|---|---|
| No silent writes | The full payload (or a value diff table) is shown to a human before every send; batch approval only on explicit human opt-in |
| No duplicates | Phase 0 dedup is mandatory and runs before any creation |
| No guessed retries | On any API error the skill surfaces the raw response and stops |
| No leaked credentials | Credentials only via env var or `chmod 600` file; never hardcoded or printed |
| No self-promotion items | Coined concepts/terms are kept out of Wikidata; they live in the repo's `graph.jsonld` |
| No giant graph diffs | `graph.jsonld` edits are text surgery + semantic validation, not whole-file re-serialization |

## What this skill does NOT do

| Concern | Use this instead |
|---|---|
| Mint the DOI / run the release that produces the artifact | [claude-skill-release-doi](https://github.com/shimo4228/claude-skill-release-doi) |
| Deposit a paper to a DOI registry | A paper-deposit workflow (this skill runs *after* it) |
| Design the `graph.jsonld` schema this skill anchors into | [claude-skill-jsonld-knowledge-graph](https://github.com/shimo4228/claude-skill-jsonld-knowledge-graph) |
| Mirror an updated `graph.jsonld` to Hugging Face Datasets | An `hf-sync` workflow |
| Create a Wikidata item for one of your own coined concepts | Nothing — the skill deliberately refuses this (see *When to use*) |

## Related research and skills

- **Doctrine repository**: [authorship-strategy](https://github.com/shimo4228/authorship-strategy) — the normative framework whose identifier-federation discipline this skill operationalizes at the Wikidata layer
- **Peer components** (other component skills of the same framework):
  - [claude-skill-release-doi](https://github.com/shimo4228/claude-skill-release-doi) — the release runbook this skill continues; it mints the DOI, this skill federates it
  - [claude-skill-jsonld-knowledge-graph](https://github.com/shimo4228/claude-skill-jsonld-knowledge-graph) — designs the `graph.jsonld` this skill's Phase 4 injects QIDs into
  - [claude-skill-llms-txt-writer](https://github.com/shimo4228/claude-skill-llms-txt-writer) — the AI-facing-documentation peer in the same Layer 4 tactic
- **Sibling research lines** (research-program level): [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), [Contemplative Agent](https://github.com/shimo4228/contemplative-agent), [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice)

> **Terminology note.** This ecosystem reserves *sibling* for research-line-level peers; at the component-skill level the term *peer component* is used instead.

## License

MIT. See [LICENSE](LICENSE).
