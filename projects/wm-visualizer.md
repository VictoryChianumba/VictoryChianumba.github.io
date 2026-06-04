---
layout: project
title: "Interpretability tooling for IRIS-class world models"
summary: "A working SAE feature explorer for IRIS world models, with a worked case study showing that activation-magnitude feature ranking, the LLM-SAE default, surfaces the wrong features on a world model."
status: "Closed"
permalink: /projects/wm-visualizer/
---

Status: Closed. Writeup at [SAE interpretability conventions don't transfer cleanly to world models](/2026/06/03/wm-visualizer/). Code at [GitHub](https://github.com/VictoryChianumba/world-model-interpretability).

Built an interactive tool for inspecting and steering sparse-autoencoder features in the residual stream of IRIS, a transformer world model trained on Atari Breakout. The tool runs the model live, extracts residual-stream activations through forward hooks, runs them through a trained SAE, and presents features on a canvas where you can pin them, search for them, and intervene on them, pushing a feature's direction into the residual stream and watching the model's imagined rollout diverge from baseline over the next twenty frames.

The central finding is a concrete worked example. The top-ranked feature by activation magnitude (#1364) turned out to be the most strongly *anti*-collision feature in the dictionary: its mean activation was 5.57 when the ball was mid-flight versus 0.82 at paddle contact. The genuine collision detectors (#1199, #120) ranked much lower by magnitude despite reproducing across episodes at Spearman +0.977. A user watching the top-firing feature during a collision would see it stay flat and conclude the tool was broken; the actual problem is that magnitude ranking surfaces persistent state-trackers over brief event detectors because duration x strength of firing dominates the score. Temporal stability ranking surfaced the collision detectors that magnitude buried (four of its top five were collision-correlated). The transferable observation, since this is the kind of thing that affects anyone porting LLM-SAE tooling to a substrate with slowly-evolving state: activation magnitude is a poor importance signal outside LLMs, and the failure mode is silent. Users blame the SAE rather than the ranking.

## Concrete contributions

- Built an end-to-end interactive SAE feature explorer for IRIS-class world models: FastAPI + React, WebSocket-streamed activations, canvas-based feature pinning, multi-feature interventions, N-step imagined rollouts with token-divergence trajectory readout.
- Identified and case-studied the magnitude-ranking failure mode with reproducible numbers: feature #1364's mu_c 0.82 vs mu_a 5.57, Spearman +0.977 across episodes for the underlying collision-correlation ranking, four-of-five overlap between temporal stability and collision detection.
- Demonstrated a magnitude-coupling confound in causal-importance pipelines: magnitude-relative injection makes causal scores secretly co-vary with activation (corr = +0.37), invisible at the per-feature level and only visible at population-level cross-check. Fixed the confound, hit a deeper deterministic-environment limitation, measured it (cross-set Spearman plateaued at +0.49), and demoted causal importance from a discovery ranking against a pre-committed Spearman >= 0.6 threshold.
- Documented six transferable observations about doing SAE interpretability on this substrate: tokenizer compression limits pixel-space readout, single-frame priming produces near-static rollouts, argmax decoding limits intervention smoothness, magnitude rankings fail, magnitude-relative causal injection collapses, autoregressive rollouts are dispatch-bound.

## Limitations

- One game (Breakout), one model (IRIS), one SAE configuration. No cross-game or cross-architecture replication. The findings are observations from one substrate, not laws.
- Small SAE (2048 features) trained on ~80k vectors; the harvest is on the low side of what's typical for clean feature recovery.
- Causal importance characterized but not shipped as a primary ranking; batched rollouts would likely clear the robustness threshold but were not implemented.

## Links

- [Blog post: detailed writeup of all six findings](/2026/06/03/wm-visualizer/)
- [GitHub repository](https://github.com/VictoryChianumba/world-model-interpretability)
