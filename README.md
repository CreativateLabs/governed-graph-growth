# governed-graph-growth

A technical note on keeping a knowledge graph trustworthy while it grows every day from sources of very
different quality — official gazettes and registers, court and authority decisions, guidance, news and research
signals.

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24099664.svg)](https://doi.org/10.5281/zenodo.24099664)

**Read the note: [NOTE.md](NOTE.md)** · Cite it: [CITATION.cff](CITATION.cff) · DOI: [10.5281/zenodo.24099664](https://doi.org/10.5281/zenodo.24099664) (all versions; v1.0.0: [10.5281/zenodo.24099665](https://doi.org/10.5281/zenodo.24099665))

## Why

Evaluations of grounded AI usually freeze the knowledge base and test the answers. In production, the knowledge
base is the part that moves — and a well-cited answer built on a corrupted node is still well cited. The note
names seven ways a growing graph decays without anyone noticing (authority inflation, identity collision,
identifier synthesis, orphan growth, schema drift, licence leakage, silent staleness) and a method that prevents
or exposes each of them.

## The method in nine lines

| | Principle | Guards against |
|---|---|---|
| P1 | Authority belongs to the source, not to the statement | Authority inflation |
| P2 | Identity is proven before a node may be enriched | Identity collision |
| P3 | Identifiers are never synthesised | Citations that resolve to nothing |
| P4 | Every relation is created by a named rule class with recorded evidence | Unexplainable structure |
| P5 | Schema changes re-open rejected input exactly once | Silent loss under schema drift |
| P6 | Licence terms travel with the data; profiles filter at serving time | Licence leakage |
| P7 | Judgement calls go through reviewed changes | Undocumented decisions |
| P8 | Growth is observed with a few signals, alerting only on failure | Silent staleness, orphan growth |
| P9 | Personal data is minimised at ingestion | Over-collection |

The principles sit on a plain, deterministic and idempotent pipeline — collect, candidate, validate, merge,
link, observe, serve, evaluate — and map onto PROV-O, SHACL, ELI, ECLI, LEI, EuroVoc/NACE and Articles 10 and
12 of the EU AI Act.

## Scope

Method only. The note deliberately contains no rules, parameters, thresholds, source-specific parsing,
prompts, evaluation sets, graph contents or production figures.

It is the companion to **[grounded-ai-evals](https://github.com/adminleo/grounded-ai-evals)**, a four-layer
method for evaluating AI systems that answer with evidence: this note keeps that method's first layer — graph
correctness — true over time.

## Author

Leonardo Bornhäußer · [ORCID 0009-0005-6464-5179](https://orcid.org/0009-0005-6464-5179) ·
[Creativate](https://creativate.tech), the company behind [c:node](https://c-node.ai)

## Licence

Text under [CC BY 4.0](LICENSE). Quote, adapt and build on it with attribution.
