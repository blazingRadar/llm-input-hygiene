# Publication Manifest

This repository is a public methodology artifact about input hygiene in structured LLM evaluation pipelines.

## Included

- A short paper in PDF form.
- The Markdown source for the paper.
- A public README describing the claim boundary.
- A public archival status note.

## Claim Boundary

The strongest claim is narrow: in one structured evaluation pipeline, a wall-clock fetch timestamp in gate-visible input caused verdict variance that disappeared when the timestamp was removed from that gate-visible input. Follow-up cuts showed that other fields which appeared to be metadata could still affect verdicts, so field removal must be empirically tested one field at a time.

The repository does not claim broad validation, production safety, a universal sanitization rule, a complete nondeterminism explanation, or a general benchmark result.

## Repository Hygiene

- No raw gate inputs, case records, API keys, or model logs are included.
- No internal planning notes are required to understand the artifact.
- `STATUS.md` is public-facing and describes only the archival state of this repo.

## Verification Surface

The public verification surface is PDF metadata/text extraction, Markdown link checking, and hygiene scans for secrets or non-public planning material. The empirical result itself is not replayable from this repository alone because the original diagnostic pipeline artifacts are not included.
