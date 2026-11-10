---
title: "Governed growth: keeping a continuously ingested, provenance-first knowledge graph trustworthy"
author: "Leonardo Bornhäußer (ORCID 0009-0005-6464-5179), Creativate Technologies GmbH"
version: "1.0"
date: "2026-11"
type: "Technical note"
license: "CC BY 4.0"
---

# Governed growth: keeping a continuously ingested, provenance-first knowledge graph trustworthy

## Abstract

A knowledge graph that grounds AI answers is only useful if it stays current, and it only stays current if it
grows every day from many sources of very different quality: official gazettes and registers, court and
authority decisions, guidance documents, news and research signals. Each daily run is an opportunity to make the
graph more complete, and an opportunity to make it quietly wrong. This note describes a governance method for
that growth. It treats authority as a property of the source rather than of the statement, requires identity to
be proven before a node may be enriched, forbids synthesised identifiers, creates every relation through a named
rule class that records its evidence, re-validates rejected input after schema changes, carries licence terms
with the data, reserves judgement calls for reviewed changes, and observes growth with a small set of health
signals. The method is independent of any particular store or model and maps onto public standards such as
PROV-O, SHACL and the European Legislation Identifier. It complements a companion method for evaluating
grounded AI systems [1], whose first layer — graph correctness — this note is designed to keep true over time.

## 1. The problem: growth is where grounded systems decay

Evaluation work on retrieval-augmented and graph-grounded systems usually freezes the knowledge base and measures
answers against it. In production the knowledge base is the part that moves. A graph that ingests legal and
market sources daily accumulates errors that no answer-level metric detects, because a well-cited answer built on
a corrupted node is still well cited.

We observed seven recurring failure modes while operating such a graph:

1. **Authority inflation.** A secondary source (a guidance note, a summary, a press release) overwrites or
   outranks the primary text it describes.
2. **Identity collision.** Two different documents are merged because they share a title, a short name or a
   number in a different numbering scheme.
3. **Identifier synthesis.** A pipeline stage constructs an identifier that looks official — a CELEX- or
   ECLI-shaped string — from data that does not prove it. The result is a citation that resolves to nothing, or
   to something else.
4. **Orphan growth.** Nodes are added faster than they are connected, so retrieval finds them but reasoning
   cannot use them.
5. **Schema drift.** The ontology evolves, and input rejected under an older version is lost rather than
   reconsidered.
6. **Licence leakage.** Content whose terms forbid redistribution or commercial use ends up in a profile that
   serves it.
7. **Silent staleness.** A source stops delivering — blocked, renamed, re-versioned — and the graph keeps
   answering from yesterday's state without anyone noticing.

None of these is exotic. All of them are invisible at the moment they happen.

## 2. Principles

### P1 — Authority belongs to the source, not to the statement

Every source is assigned an authority tier before any of its content is ingested: **authoritative** (the
official text or register entry itself), **reported** (a secondary account of an authoritative item, such as
guidance, commentary or a decision summary) or **signal** (an indication worth knowing, such as a preprint or an
announcement, that is not evidence of the law or the fact itself). A node's tier is inherited from its source and
is never raised by later processing. Node types that are reported by nature can never become authoritative, no
matter how many sources repeat them. When a reported source describes an authoritative node, the description is
stored as labelled enrichment next to the authoritative content, not in its place.

### P2 — Identity before enrichment

An incoming item may modify an existing node only after it has proven that it is about that node, using the
identifier scheme of the authoritative source (for example ELI or CELEX for EU acts, ECLI for case law, LEI for
legal entities). Matching on titles, short names or fuzzy text is allowed for *proposing* a link, never for
*merging*. If the proof fails or two identifiers disagree, the conflict is recorded on the node as an explicit
state and left for review; it is not resolved by picking the more recent value.

### P3 — Identifiers are never synthesised

If a source does not prove an identifier, the identifier stays empty. A missing reference is an honest gap that
retrieval can report; a fabricated one is a confident error that every downstream citation inherits. The same
rule applies at finer grain: a reference to a specific article or point is only stored when that article or
point is proven, otherwise the reference stops at the level that is.

### P4 — Every relation has a named reason

Relations are not inferred ad hoc. Each is created by a rule from a small, fixed set of rule classes, and records
which class created it and from what evidence:

- **explicit reference** — the text of one item cites another in a recognised citation format;
- **official metadata** — the publisher's own metadata states the relation (amends, repeals, implements);
- **hub consolidation** — several items independently refer to the same act, which is then materialised as a
  shared node under its official title so the references meet;
- **applicability mapping** — an act is linked to the sectors or entity types it applies to, using a public
  classification such as NACE;
- **curated clustering** — items are grouped under a topic that no source states explicitly; such clusters are
  never authoritative and change only through reviewed changes (P7).

Because every edge names its rule class, an error can be traced to the rule that produced it and corrected for
all edges of that class at once. It also makes the graph's structure explainable to a reviewer who never sees the
code: "these two acts are linked because the second one's official metadata says it amends the first."

### P5 — Schema changes re-open rejected input once

Input is validated against a versioned ontology. Items rejected for schema reasons are kept with the version
they were rejected under. When the ontology version changes, they are re-validated exactly once against the new
version. This turns schema evolution from a source of silent loss into a recoverable event, and the "exactly
once" keeps it from becoming a retry loop.

### P6 — Licence terms travel with the data

Every node carries the licence class of the source it came from, and attribution requirements are kept alongside.
Serving profiles declare which licence classes they may deliver; filtering happens at serving time, on every
request, so that the same graph can serve a public observatory, a commercial API and an on-premises installation
without copies drifting apart. Redistribution terms are checked when a source is admitted, not when someone asks.

### P7 — Judgement goes through review

Deterministic rules handle what can be proven. Where a decision requires judgement — what counts as a topic
cluster, whether a borderline source is authoritative, how to resolve an identity conflict — the change is made
as a reviewed change request against a curated file, not by the pipeline. Human review is thereby confined to the
small set of decisions that need it, and every such decision has an author, a date and a reason.

### P8 — Growth is observed, not assumed

Each run produces a health record with a few signals that map directly onto the failure modes in section 1:
per-source freshness (silent staleness), provenance coverage (identifier synthesis, authority inflation), share
of unconnected nodes (orphan growth), counts of rejected and re-validated candidates by reason (schema drift),
open identity conflicts (identity collision) and licence-filtered volume per profile (licence leakage). Operators
are notified when a signal fails or a source goes stale, and not otherwise; a report that arrives every day
whether or not something is wrong stops being read.

### P9 — Personal data is minimised at ingestion

Sources about natural persons — officers, beneficial owners, persons on sanctions lists — are ingested with the
minimum attributes needed for the purpose, and attributes such as addresses or identity-document numbers are not
stored. Minimisation at ingestion is cheaper and more reliable than filtering at serving time.

## 3. The pipeline

The principles above sit on a deliberately plain pipeline. Each stage is deterministic and idempotent, so that a
run can be repeated after a failure without duplicating or losing anything.

1. **Collect.** Each source has a collector that pulls changes since a stored cursor. The cursor advances only to
   the last item actually emitted, so a capped or interrupted run resumes without gaps.
2. **Candidate.** Every item becomes a typed candidate with provenance attached: source, retrieval time,
   authority tier (P1), licence class (P6), identifiers as proven (P3).
3. **Validate.** Candidates are checked against the versioned ontology (P5), for identity against existing nodes
   (P2) and for licence admissibility (P6). Rejections are kept with their reason.
4. **Merge.** Accepted candidates are merged as a delta into the graph snapshot. Merging never raises authority,
   never overwrites authoritative content with reported content and never resolves an identity conflict silently.
5. **Link.** Rule classes run over the delta and its neighbourhood (P4).
6. **Observe.** The health record is written and evaluated (P8).
7. **Serve.** Profiles filter by licence class and authority tier at request time (P6).
8. **Evaluate.** Before a new snapshot is promoted, it passes graph-level checks — provenance coverage,
   constraint validation, sampled correctness against official sources — as the first layer of the companion
   evaluation method [1].

## 4. Mapping to public standards

| Concern | Standard or practice | Where it applies |
|---|---|---|
| Provenance of nodes and edges | W3C PROV-O [2] | P1, P4, candidate stage |
| Shape and constraint validation | W3C SHACL [3] | P5, validate stage |
| Identifiers for legislation | European Legislation Identifier (ELI) [4], CELEX | P2, P3 |
| Identifiers for case law | European Case Law Identifier (ECLI) [5] | P2, P3 |
| Identifiers for legal entities | ISO 17442 Legal Entity Identifier (LEI) | P2, P3 |
| Subject and sector vocabularies | EuroVoc [6], NACE Rev. 2 | P4 (applicability mapping, clustering) |
| Ranking of statements by reliability | Statement ranks and references in Wikidata [7] | P1 (analogous design) |
| Findable, reusable data | FAIR principles [8] | P6, licence and attribution metadata |
| Data governance and record-keeping for AI | EU AI Act, Art. 10 and Art. 12 [9] | P7, P8 |
| Data minimisation | GDPR, Art. 5(1)(c) | P9 |

## 5. What this note does not contain

This note describes a method. It deliberately does not contain the rules themselves, their parameters or
thresholds, source-specific parsing, prompts, evaluation sets, graph contents or production figures. The method
is meant to be applicable to any graph that grows from heterogeneous sources, and checkable by anyone who
operates one.

## 6. Limitations

- **Authority tiers are a judgement made once per source.** A source that changes character over time (for
  example an official portal that starts publishing commentary) needs its tier revisited; P7 covers the change,
  but nothing detects the need automatically.
- **Identity proof depends on identifier coverage.** Domains without stable public identifiers fall back to
  proposed links that require review, which does not scale linearly.
- **Rule classes trade recall for explainability.** Relations that no rule class can justify are not created,
  even when a human would see them immediately.
- **Health signals detect symptoms, not causes.** A falling freshness signal says a source stopped delivering; it
  does not say whether the source was blocked, moved or retired.

## References

1. L. Bornhäußer. *grounded-ai-evals: a four-layer method for evaluating AI systems that answer with evidence.*
   2026. https://github.com/adminleo/grounded-ai-evals
2. W3C. *PROV-O: The PROV Ontology.* W3C Recommendation, 2013. https://www.w3.org/TR/prov-o/
3. W3C. *Shapes Constraint Language (SHACL).* W3C Recommendation, 2017. https://www.w3.org/TR/shacl/
4. Publications Office of the EU. *European Legislation Identifier (ELI).* https://eur-lex.europa.eu/eli-register/about.html
5. Council of the EU. *Council conclusions inviting the introduction of the European Case Law Identifier (ECLI).*
   OJ C 127, 29.4.2011.
6. Publications Office of the EU. *EuroVoc.* https://op.europa.eu/en/web/eu-vocabularies/eurovoc
7. D. Vrandečić, M. Krötzsch. *Wikidata: a free collaborative knowledgebase.* Communications of the ACM 57(10),
   2014.
8. M. D. Wilkinson et al. *The FAIR Guiding Principles for scientific data management and stewardship.*
   Scientific Data 3, 160018, 2016.
9. Regulation (EU) 2024/1689 (Artificial Intelligence Act), OJ L, 12.7.2024.
