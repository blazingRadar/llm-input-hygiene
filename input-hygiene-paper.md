# Input Hygiene in Structured LLM Evaluation Pipelines

## Empirical Findings on Verdict Variance from Non-Evidence Fields

Author: Nick Cunningham
Date: 2026-04-22
Scope: Diagnostic work on a structured LLM evaluation pipeline. Findings are narrow by design.

## Summary

In a structured LLM evaluation pipeline, we observed verdict variance that initially looked like model nondeterminism and turned out to be driven by non-evidence fields in the gate input. A single non-evidence field, a wall-clock fetch timestamp, was sufficient to induce verdict variance across runs. Removing it restored full determinism on byte-identical input for the tested case.

Attempting to generalize the cleanup to other fields classified as metadata caused verdict drift on boundary cases, forcing reverts. The useful finding is not a list of safe fields but a discipline: field classification cannot be done by inspection alone and requires per-field empirical verification.

This suggests that some apparent nondeterminism in structured evaluation systems may be an artifact of unstable input construction rather than model behavior alone.

This is a methodology note from small-scale diagnostic work. It is not broad validation. It is shared because the pattern is useful for teams building similar evaluation pipelines.

## Context

The pipeline under study had three broad stages: a worker processed evidence, a builder composed a packet for adjudication, and a gate model returned a verdict. Observed verdict variance across repeated runs raised the question of whether the variance was model sampling noise or something in the input.

The investigation was not originally aimed at input hygiene. It surfaced during model-portability work: we were testing whether a structured evaluation pipeline could produce stable verdicts when the underlying gate model was substituted. While testing a GPT-family gate model at temperature 0, we observed verdict variance that did not fit the pattern we expected from model-level nondeterminism alone. The variance was structured and tied to specific cases. Investigating that variance led to the input-hygiene finding.

## First Finding

For one boundary case, five repeated runs produced four verdicts in one direction and one in the other. Setting the gate temperature to 0 and running ten times on truly byte-identical input produced ten identical verdicts. On byte-identical input at temperature 0, the gate model was deterministic in that local test.

Comparing the gate inputs from the original variance-producing runs revealed that the only varying field was a wall-clock timestamp recording when upstream data had been fetched. The timestamp was provenance metadata, not case evidence. We removed it from the gate-visible packet and kept it in downstream records.

After the fix, the same case, run ten times on input that now differed only in non-visible fields, produced identical gate inputs by hash and stable verdicts.

This separated input variance from sampling variance. The verdict drift we had attributed to model behavior was driven by gate-visible input variance.

## Second Finding

Encouraged by the timestamp result, we audited the full gate-visible packet for other non-evidence fields. Several fields looked like metadata: navigation URLs, platform identifiers, profile fields, runner metadata, and similar fields.

We removed a batch of these fields. On the boundary case, five runs produced five identical gate inputs by hash but all returned the opposite verdict from the stable pre-cut baseline. The cut had crossed from hygiene into verdict-affecting territory. At least one field we had classified as metadata contained information the gate was using to reason toward the original verdict. The cut was reverted.

We then tried one field at a time. The next candidate recorded paths an upstream system had fetched. It looked harmless. On the boundary case, removing it flipped verdicts five out of five. It was reverted.

Two cuts had now failed in the same way: fields that appeared to be metadata by inspection affected gate reasoning.

## Canary Discipline

After the first over-aggressive cut, we established a protocol for further field removal:

1. Before a cut, run the proposed-removed pipeline five times on one boundary case and one clear-direction case.
2. Record gate input hashes and verdicts for all runs.
3. Revert cuts that change verdicts on either canary case, regardless of the classification reasoning.
4. Accept cuts that preserve verdicts on both canaries only after a broader rerun confirms no wider drift.

The canary discipline caught the second failed cut. Without it, we would have propagated a verdict-drifting change as infrastructure cleanup.

## What This Means

The narrow finding is solid within this diagnostic setting: a wall-clock timestamp in gate-visible input caused verdict variance that was not model nondeterminism. Removing it improved input determinism.

The broader generalization, that non-evidence fields can be systematically identified and removed by inspection, is not supported by our testing. Field classification by inspection failed at least twice on fields that looked reasonable to remove. The discipline that holds up is empirical verification per field, not global cleanup.

After the initial finding, we applied the same methodology to two additional evaluation pipelines in our own work. One showed no observable drift after cleanup. The other showed verdict drift on a canonical test case. The mixed result reinforces the canary discipline: drift is pipeline-specific, and the methodology catches drift that inspection misses.

Small residual variance may remain even after removing identified variance sources. We did not fully characterize whether that residual variance came from API-level behavior at temperature 0 or from additional input fields we had not identified. That question remains open.

## Scope And Limits

This work was done on curated diagnostic cases in structured evaluation pipelines using a GPT-family gate model at temperature 0. The raw case corpus, gate inputs, model logs, and verdict logs are not included in this public artifact.

Findings should be understood as:

- Direct evidence: one specific field, a wall-clock timestamp, caused verdict variance when included in gate-visible input.
- Direct evidence: empirical field classification is necessary; inspection-based classification failed in this setting.
- Direct evidence: the canary discipline caught classification errors.
- Direct evidence: the methodology surfaced pipeline-specific drift that would otherwise have been absorbed into "model nondeterminism."
- Not established: which other specific fields are safe to remove.
- Not established: whether the pattern generalizes to other pipelines, models, or corpora at scale.

Anyone relying on this note for production decisions should run the protocol on their own pipeline.

## Why This Is Worth Sharing

LLM nondeterminism at temperature 0 has multiple possible causes, including implementation details, batching, hardware, routing, or model-serving behavior. This note adds a simpler possibility for structured evaluation systems: some variance may come from unstable gate-visible input.

Separating input variance from model variance is useful before concluding that a gate is unreliable. The field-hygiene discipline is simple:

1. Inspect the gate input surface for fields that are not evidence about the case being judged.
2. Test candidate removals with canary cases before accepting them as cleanup.
3. Revert cuts that change canary verdicts, regardless of how reasonable the classification seemed.

The methodology is more portable than any specific list of fields.

## Closing Note

This is the output of diagnostic work on a narrow problem that surfaced during broader model-portability testing. It is not a product announcement, a formal research contribution, or a claim to have solved input hygiene at scale. The framing is intentionally narrow: the first fix worked, broader cleanup failed, and empirical per-field verification was the discipline that held up.
