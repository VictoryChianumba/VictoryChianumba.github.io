---
layout: project
title: "Adversarial attacks that evade explanation-based monitoring in EEG classifiers"
summary: "Demonstrated that low-pass adversarial perturbations can flip an EEG classifier's prediction while leaving its Integrated Gradients explanation essentially intact."
status: "Closed"
permalink: /projects/robust-eeg-models/
---

Status: Closed. Writeup at [When the explanation lies](/2026/06/04/eeg-explanation-monitoring/). Code at [GitHub](https://github.com/VictoryChianumba/robust-eeg-models). MSc thesis work, University of Bath.

Tested whether Integrated Gradients explanations can serve as a monitor against adversarial attacks on EEG classifiers. The implicit promise of such monitoring is that if an attack corrupts a model's prediction, the explanation should reveal the corruption, so a defender watching the explanation can catch attacks they couldn't otherwise see. The setup was four classifier architectures (EEGNet, DeepConvNet, CTNet, and a state-space hybrid loosely modeled on Mamba) trained on motor-imagery decoding from BCI Competition IV Dataset 2a, attacked with five methods: three standard (FGSM, PGD, DeepFool) and two with low-pass-filtered perturbations originally designed for physiological plausibility rather than for explanation evasion.

The central finding: low-pass attacks flipped predictions while leaving the Integrated Gradients explanation almost perfectly intact (Spearman rank correlation 0.998 between clean and attacked attribution maps), so an explanation-based monitor would have caught 0% of successful LP attacks across any reasonable detection threshold. Standard attacks, by contrast, disturbed the explanation in 87-97% of successful attempts. The attacker pays a real cost for that stealth: LP attacks succeed 61-66% of the time versus 88-95% for standard attacks, so this is a trade-off rather than a clean defeat of the defense. But it is a trade available to an attacker who values invisibility over raw effectiveness, and a stable explanation over a corrupted prediction is false assurance, not robustness. The implication for interpretability-as-safety-check is that any such monitor needs to be tested against attacks designed not to disturb the explanation, since "the explanation didn't move" is engineerable.

## Concrete contributions

- Demonstrated a specific, exploitable blind spot in Integrated Gradients as an adversarial-attack monitor: Spearman 0.998 between clean and LP-attacked IG maps, E-ASR 0.0% at thresholds spanning 0.90-0.99, holding across four architectures.
- Quantified the trade-off explicitly: LP attacks succeed ~30 percentage points less often than standard attacks, so the stealth comes at a measurable cost rather than free.
- Designed the low-pass attack variants whose interaction with IG produced the headline finding. The attacks were motivated by physiological plausibility (Gaussian-smoothed perturbations that look like signal rather than noise), and the explanation-evasion property was an empirical observation, not a designed-in target.
- Built the end-to-end adversarial robustness pipeline: four EEG architectures, five attacks, multi-seed evaluation across four subjects with physiologically-scaled perturbation budgets.

## Limitations

- One dataset (BCI IV-2a), four selected subjects (selection itself is a confound). One explanation method, one baseline choice for it. The general version of "explanations are blind to LP attacks" is a hypothesis this work raises, not establishes.
- The headline E-ASR magnitudes for standard attacks are threshold-dependent (46% at 0.90, 97% at 0.97); only the LP-vs-standard contrast is threshold-robust.
- The EEGMamba implementation is an FFT-based approximation rather than a faithful Mamba reimplementation; flagged as such rather than presented as state-space.

## Links

- [Blog post: detailed writeup with attacks, findings, and methodology](/2026/06/04/eeg-explanation-monitoring/)
- [GitHub repository](https://github.com/VictoryChianumba/robust-eeg-models)
