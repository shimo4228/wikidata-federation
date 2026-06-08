# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Planned

- Initial public release.

### What it does

A Claude Code skill that registers researchers, papers, and DOI-registered repositories as Wikidata items and cross-links them so that ORCID, Wikidata QID, DOI, and a repository's `graph.jsonld` form one mutually-referencing machine-readable node. It is the identifier-federation step of the [`authorship-strategy`](https://github.com/shimo4228/authorship-strategy) research line, run after a DOI is minted.

### Components

- `skills/wikidata-federation/SKILL.md` — the skill body. Safety discipline, prerequisites, and a six-phase workflow (dedup → metadata resolution → dry run → item creation → `graph.jsonld` anchoring → verification), plus an empirically-derived pitfalls table.
- `skills/wikidata-federation/scripts/gen_payload.py` — spec JSON → `wbeditentity` payload generator that mechanically enforces the author/paper/software property conventions (license↔copyright-status pairing, required qualifiers, `P50` vs `P178` by instance type).
- `skills/wikidata-federation/scripts/wd_api.py` — minimal `wbeditentity` transport client (auth → CSRF → single send, `maxlag=5`, stops and surfaces the raw response on error). Credentials via env var or `~/.config/wikidata/credentials.env` only.
- `skills/wikidata-federation/scripts/add_qid_sameas.py` — format-preserving `graph.jsonld` QID `sameAs` injector (text surgery + semantic tree validation; dry-run by default).

### Scope

The skill assumes a Wikidata write target and a repository carrying a `graph.jsonld`. It runs after the artifact already has an external identifier (ORCID for authors, DOI for works); it does not mint identifiers itself.

### Requirements

- A Wikidata BotPassword (local to the wiki it is created on)
- Credentials via env var or `chmod 600` file — never hardcoded
- Python + `requests` (`uv run --with requests` works)

### Relationship to companion skills

| Skill | Role | When |
|---|---|---|
| `release-doi` | Release runbook that mints the DOI | Before this skill — produces the artifact this skill federates |
| `jsonld-knowledge-graph` | Designs the `graph.jsonld` schema | Before this skill — Phase 4 injects QIDs into that graph |
| `hf-sync` | Hugging Face Datasets mirror sync | After this skill — propagates the updated `graph.jsonld` |
