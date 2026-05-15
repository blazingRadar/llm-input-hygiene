# Input Hygiene in Structured LLM Evaluation Pipelines

Public methodology note on a narrow failure mode in structured LLM evaluation pipelines: verdict variance that looked like model nondeterminism but was caused by unstable gate-visible input.

## Current Status

This repository is published-done and archived as a standalone note. Future work in this lane continues elsewhere; this repo preserves the original public artifact and its claim boundary.

The canonical paper is available as:

- [input-hygiene-paper.pdf](input-hygiene-paper.pdf)
- [input-hygiene-paper.md](input-hygiene-paper.md)

## What The Note Shows

- A wall-clock fetch timestamp in gate-visible input caused verdict variance in one structured evaluation pipeline.
- Removing that timestamp from the gate-visible packet restored stable verdicts on byte-identical input for the tested case.
- Attempts to remove other fields that looked like metadata caused verdict drift on boundary cases.
- The durable method is empirical per-field verification with canary cases, not broad cleanup by inspection.

## What It Does Not Show

- It does not prove broad generalization across models, corpora, or production systems.
- It does not identify a universal list of safe fields to remove.
- It does not claim that model-level nondeterminism is absent.
- It does not include the raw case corpus or gate logs.
- It is not a product announcement or a production input-filtering library.

## Verification Surface

This repo is a publication artifact, not a code package. A reviewer can verify:

```bash
pdftotext input-hygiene-paper.pdf - | head
pdfinfo input-hygiene-paper.pdf
```

The empirical claims require the original diagnostic pipeline artifacts, which are not included here. Treat the note as a methodology observation and run the protocol on your own pipeline before relying on it.

## Repository Files

```text
README.md                public landing page
PUBLICATION_MANIFEST.md  claim boundary and repository hygiene
STATUS.md                archival status
input-hygiene-paper.md   source text for the paper
input-hygiene-paper.pdf  rendered paper
```
