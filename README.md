# Input Hygiene in Structured LLM Evaluation Pipelines
### Empirical Findings on Verdict Variance from Non-Evidence Fields

Short methodology note based on diagnostic work in structured LLM evaluation pipelines.

Focus: separating input-driven variance from model behavior and identifying when "model nondeterminism" is actually pipeline-induced.

**Author:** Nick Cunningham  
**Date:** 2026-04-22

---

## Summary

In a structured LLM evaluation pipeline, we observed verdict variance that initially looked like
model nondeterminism and turned out to be driven by non-evidence fields in the gate's input. A
single non-evidence field (wall-clock fetch timestamp) was sufficient to induce verdict variance
across runs. Removing it restored full determinism on byte-identical input. Attempting to generalize
the cleanup to other fields classified as metadata caused verdict drift on boundary cases, forcing
reverts. The useful finding is not a list of safe fields but a discipline: field classification cannot be
done by inspection alone and requires per-field empirical verification.

This implies that a portion of reported LLM nondeterminism in structured evaluation systems may
be an artifact of unstable input construction rather than inherent model behavior.

This is a methodology note from small-scale diagnostic work. It is not broad validation. It is shared
because the pattern is the kind of thing teams building similar pipelines might want to watch for.

## Context

We were running a structured evaluation pipeline where a worker processes evidence, a builder
composes a packet for adjudication, and a gate model returns a verdict. Observed verdict variance
across repeated runs on similar inputs raised the question of whether the variance was model
sampling noise or something in the input.

## How The Finding Surfaced

The investigation was not originally aimed at input hygiene. We were testing whether the
evaluation pipeline could be made model-agnostic — specifically, whether a deterministic gate
could produce stable verdicts when the underlying gate model was substituted. The motivating
question was portability: if the pipeline's behavior depends on a specific gate model, the pipeline is
tied to that model's quirks. If the pipeline's behavior is stable across comparable gate models at
temperature 0, the pipeline has a more durable foundation.

While testing GPT as one of the gate models under this frame, we observed verdict variance that
did not fit the pattern documented for model-level nondeterminism. The variance was too

structured and too tied to specific cases to be attributed to floating-point or batching effects alone.
Investigating that variance is what led to the input hygiene finding. The paper's contribution is
therefore a byproduct of model-agnosticism work, not a study designed to find it.

## The First Finding

For one boundary case, five repeated runs produced four verdicts in one direction and one in the
other. Setting the gate temperature to 0 and running ten times on truly byte-identical input
produced ten identical verdicts. On byte-identical input at temperature 0, the gate model was
deterministic.

Comparing the gate inputs from the original variance-producing runs revealed that the only varying
field was a wall-clock timestamp recording when upstream data had been fetched. The timestamp
was provenance metadata, not case evidence. We removed it from the gate-visible packet and
kept it in downstream records.

After the fix:

●   The same case, run ten times on input that now differed only in non-visible fields, produced
identical gate inputs by hash.
●   Verdicts were stable.

This separated input-variance from sampling-variance. The verdict drift we had attributed to model
behavior was in fact driven by input variance the model was responding to.

## The Second Finding

Encouraged by the timestamp result, we audited the full gate-visible packet for other non-evidence
fields. We classified fields as evidence, metadata, or unclear. Several fields looked like pure
metadata: navigation URLs, platform identifiers, profile fields, runner metadata, and so on.

We removed a batch of these fields. On the boundary case, five runs produced five identical gate
inputs by hash but all returned the opposite verdict from the stable pre-cut baseline. The cut had
crossed from hygiene into verdict-affecting territory. At least one field we had classified as
metadata contained information the gate was using to reason toward the original verdict. The cut
was reverted.

We then adopted a one-field-at-a-time approach. The next candidate was a field recording the
paths an upstream system had fetched. It looked harmless. On the boundary case, removing it
flipped verdicts five out of five. It was reverted.

Two cuts had now failed on the same pattern: fields that appeared to be metadata by inspection
turned out to affect gate reasoning.

## The Canary Discipline

After the first over-aggressive cut, we established a protocol for any further field removal:

1    Before a cut, run the proposed-removed pipeline five times on one boundary case (a case
near judgment boundary where behavior is most sensitive) and one clear-direction case.
2    Record gate input hashes and verdicts for all runs.
3    Cuts that change verdicts on either canary case, regardless of the classification reasoning, are
reverted.
4    Cuts that preserve verdicts on both canaries are accepted only after a broader rerun confirms
no wider drift.

The canary discipline caught the second failed cut. Without it, we would have propagated a
verdict-drifting change as an infrastructure cleanup.

## What This Means

The narrow finding is solid: wall-clock timestamps in gate-visible input cause verdict variance that
is not model nondeterminism. Removing them improves input determinism.

The broader generalization, that non-evidence fields can be systematically identified and removed
to clean up gate input, is not supported by our testing. Field classification by inspection has failed
at least twice on fields that looked reasonable to remove. The discipline that holds up is empirical
verification per field, not global cleanup.

After the initial finding, we applied the same input-cleaning methodology to two additional
evaluation pipelines in our own work. One showed no observable drift after cleanup, suggesting
that pipeline's gate input was already robust to the fields we removed. The other showed verdict
drift on a canonical test case, suggesting the cleanup pattern was not uniformly safe and that
specific fields mattered in ways that required per-pipeline verification. The mixed results across
pipelines reinforce the canary discipline rather than weaken it: the methodology catches drift that
inspection misses, and the drift is pipeline-specific.

Even after removing the identified variance sources, small residual variance may remain. We did
not fully characterize whether this residual variance is API-level nondeterminism at temperature 0
(which is documented in the literature) or additional non-evidence fields we have not yet identified.
The question is open.

## Scope and Limits

This work was done on evaluation pipelines using GPT at temperature 0, on curated case corpora
assembled for diagnostic purposes rather than broad validation. The methodology was exercised

through:

●      repeated-verdict probes on two canary cases (five runs each per cut attempt) to test verdict
stability after proposed field removals;
●      full pipeline reruns after each accepted cleanup to confirm no wider drift;
●      a larger corpus characterization to confirm pipeline behavior remained stable across input
variations;
●      application of the methodology to two additional pipelines with differing outcomes (one showed
no drift, one showed drift on a canonical case).

Findings should be understood as:

●      Direct evidence: one specific field (wall-clock timestamp) causes verdict variance when in gate
input.
●      Direct evidence: empirical field classification is necessary; inspection-based classification has
failed.
●      Direct evidence: the canary discipline catches classification errors.
●      Direct evidence: the methodology surfaces pipeline-specific drift that would otherwise be
silently absorbed into "model nondeterminism."
●      Not established: which other specific fields are safe to remove.
●      Not established: whether the pattern generalizes to other pipelines, models, or corpora at scale.

This is a methodology observation from diagnostic work, not a validated finding. Anyone building
similar pipelines may find the discipline useful. Anyone relying on this note for production
decisions should run their own verification.

## Why This Is Worth Sharing

Temperature-0 nondeterminism is well-documented in the LLM literature. Most of that work
attributes variance to sampling, batching, hardware, or Mixture-of-Experts routing. Our observation
is that a meaningful portion of what looks like model nondeterminism in a structured evaluation
pipeline may actually be input variance from fields that don't belong in gate input in the first place.
Separating these is useful before concluding that a gate is "nondeterministic" or "unreliable."

The field hygiene discipline is simple enough to adopt without major architecture changes:

1    Inspect the gate's input surface for fields that are not evidence about the case being judged.
2    Test candidate removals with canary cases before accepting them as cleanup.

3    Revert cuts that change canary verdicts, regardless of how reasonable the classification
seemed.

The methodology is more portable than any specific list of fields.

## Closing Note

This is the output of diagnostic work on a narrow problem that surfaced during broader
model-agnosticism testing. It is not a product announcement, a research contribution in any formal
sense, or a claim to have solved anything at scale. The framing is honest: we were testing whether
a structured evaluation pipeline could produce stable verdicts across gate models, we observed
variance that didn't fit the documented patterns for model nondeterminism, the first fix worked, the
broader generalization did not, and the discipline that held up is empirical-per-field verification. If
any of this is useful to others doing similar work, it is shared in that spirit.

If you run this protocol on your own evaluation pipeline and find something interesting, I would be
glad to hear about it.
