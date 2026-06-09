---
layout: project
title: "Factored sparse autoencoders for LLM concept manifolds"
summary: "Tested whether factored atlas-of-charts SAEs capture curved concept manifolds better than standard linear dictionaries. Found one narrow positive result, several honest negatives, and one transferable methodological warning about evaluating MoE-style SAE variants."
status: "Closed"
permalink: /projects/factored-sae/
---

Status: Closed. Writeup at [WRITEUP.md](https://github.com/VictoryChianumba/sae-manifold-experiments/blob/main/prototype/WRITEUP.md). Code at [GitHub](https://github.com/VictoryChianumba/sae-manifold-experiments).

Built and evaluated a factored SAE architecture inspired by Goodfire's "Do Sparse Autoencoders Capture Concept Manifolds?" The architecture replaces the standard SAE's dictionary of straight directions with a soft router over multiple "charts" plus per-chart nonlinear decoders, on the hypothesis that curved local charts could capture concept manifolds (years as helix, geography as sphere, colors as wheel) that linear dictionaries can only tile through "shattering" and "dilution." Ran the fair-comparison protocol on SmolLM2-135M for iteration and Llama 3.1-8B for the headline numbers, with 10 seeds for statistical robustness.

The headline result is narrow. After correcting a matched-dimension protocol bug (the soft-routed factored model was being evaluated using the full 12-chart mixture against a strict 3-dimensional PCA baseline, which is not matched-dim), the curvature win survives cleanly only on geography (R² 0.776 ± 0.010 vs PCA 0.472 ± 0.010 at 10 seeds with supervised concept-shaped alignment, ~30 SEM gap). The other four concept manifolds tie PCA or lose. Curvature does real work on the manifold where the geometry is most overtly nonlinear, and does less than expected elsewhere. Causal steering at 8B is modulation rather than control: 1/3 of seeds crosses the decision boundary, 2/3 stop short. The transferable methodological note from the project, beyond the narrow positive: soft routing in MoE-style SAE variants leaks signal into the gating mechanism, producing reconstruction-quality numbers that can't be attributed to any single chart's representation. The failure is invisible at the per-feature level and only visible when a less-expressive baseline beats the supposedly stricter ceiling.

## Concrete contributions

- Built end-to-end SAE training and evaluation pipeline on Llama 3.1-8B (RunPod A100), including activation harvesting, fair-comparison protocol with multi-seed Pareto sweeps, cyclic-aware probe scoring, and a causal steering rig.
- Demonstrated supervised concept-shaped legibility crosses PCA cleanly on geography (R² +0.30 at λ=1, ~30 SEM gap across 10 seeds) and that the same architectural setup achieves a 2 SEM-significant unsupervised crossing via isometry-plus-parsimony at smaller magnitude.
- Documented the soft-routing MoE-SAE evaluation failure mode with empirical numbers: linear factored beats PCA-3 by +0.15 at 135M under the buggy protocol; collapses to -0.02 at 8B once hard top-1 routing closes the leak. Relevant to anyone building MoE-style SAE variants who needs to evaluate them honestly.
- Causal steering reaches control on 1/3 of seeds at 8B, modulation on 2/3. Reframed the project's earlier "modulation only" claim once multi-seed analysis showed the variance was wider than reported.

## Limitations

- The broad hypothesis (curvature helps across multiple curved manifolds) is not supported; the win is concentrated on geography.
- Most results are at small scale (SmolLM2-135M) with one re-run leg at Llama 3.1-8B. Cross-model replication is limited.
- The "Goodfire's own followup is the actual contribution" framing applies: their Ising-coupling clustering on real LLM features is the substantive answer to the same question; this work is a parallel architectural experiment at smaller scope.

## Links

- [WRITEUP.md: detailed writeup with bug-find-and-recovery arc](https://github.com/VictoryChianumba/sae-manifold-experiments/blob/main/prototype/WRITEUP.md)
- [GitHub repository](https://github.com/VictoryChianumba/sae-manifold-experiments)
- [Goodfire's original paper (Bhalla et al. 2026)](https://arxiv.org/abs/2604.28119)
