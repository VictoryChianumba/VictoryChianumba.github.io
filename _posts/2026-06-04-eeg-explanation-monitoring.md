---
layout: post
title: "When the explanation lies: adversarial attacks that fool a model without disturbing its interpretation"
date: 2026-06-04
---

A specific family of adversarial attacks reliably breaks an EEG classifier's prediction while leaving its Integrated Gradients explanation essentially untouched. If you were watching the explanation to catch attacks, which is one of the things interpretability is supposed to be good for, these attacks would walk straight past you.

The claim is narrow and worth stating precisely up front. In this setup, with this explanation method and these attacks, the explanation-based detector caught the standard attacks almost perfectly and the low-pass ones not at all. The interesting part is that "quiet to the explanation" doesn't mean "quiet to a human looking at the raw signal", and that the attacker pays a real, measurable price for that quietness. It is a trade, not a free lunch.

## The setup

The project trained classifiers to decode motor imagery from EEG: the task is to look at a few seconds of brain signal recorded while someone imagines moving their left hand, right hand, feet, or tongue, and predict which one they imagined. This is a standard brain-computer-interface benchmark (BCI Competition IV Dataset 2a). I worked with four subjects from that dataset (subjects 1, 3, 8, and 9), chosen for data quality rather than at random, which is a limitation I'll come back to.

I trained four architectures: EEGNet and DeepConvNet (both convolutional networks designed for EEG), CTNet (a convolution-plus-transformer hybrid), and EEGMamba (a convolution-plus-state-space hybrid). At the time I ran these experiments there was no official EEGMamba release, so I built my own adaptation of a Mamba-style state-space block bolted onto an EEG front end. It's FFT-based rather than fully selective and I'm flagging it as an approximation rather than a faithful reimplementation. The architecture choice barely matters to the result, so not much rides on this.

Clean classification accuracy averaged about 69% on these four subjects. That's the honest baseline: competent models on a hard four-class problem, not heavily tuned. The published literature for these architectures on this dataset reports a wide band (low 60s for plain reimplementations up to the low-to-mid 80s with per-subject tuning), with different protocols making direct comparison confounded. My numbers sit inside that band but below the tuned state-of-the-art. The point of this work is relative: how explanation stability behaves under attack, so the absolute accuracy isn't the claim. I selected subjects 1/3/8/9 where the models trained to accuracies comparable to published per-subject results and dropped subjects where decoding was near chance, on the reasoning that an explanation-stability experiment is only meaningful on a model that actually learned the task.

## What Integrated Gradients actually computes

When one of these models makes a prediction, you can ask which parts of the input mattered. Integrated Gradients is a standard answer. Because it's the load-bearing tool in this post, it's worth understanding what it actually does rather than waving at "feature importance."

The naive version is to take the gradient of the model's output with respect to each input feature: each EEG channel at each time point. A large gradient means: nudge this feature a little, the model's confidence moves a lot. The problem is that raw gradients are local. Neural networks saturate: once a feature has pushed the model firmly toward a class, its gradient can flatten to near zero even though that feature is exactly why the model decided what it did. A feature can be decisive and have a near-zero gradient at the final input, and naive gradient attribution will wrongly call it unimportant.

IG fixes this by integrating along a path. It picks a baseline: a reference input representing "nothing," typically an all-zeros trace. Then it imagines a straight line from the baseline to the actual input, computes gradients at each step along that path, and accumulates them weighted by how far each feature travelled from baseline to input. The result captures the feature's contribution across the whole journey, not just at the saturated endpoint. A feature that did decisive work early in the path gets credit even if its gradient at the final input is zero.

Two free choices are buried in that procedure: the baseline, and the fact that IG's sensitivity is shaped by where along the path the gradients are large. Keep that second point in mind; it's the most plausible reason the low-pass attacks turn out to be invisible to it.

## Why we'd trust it as a safety check

The reason this matters for safety rather than just for curiosity is the implicit promise. If the explanation faithfully tracks what the model is doing, it should be a monitor. When the model behaves normally, the explanation looks normal. When something corrupts the model's decision, the explanation looks corrupted too. So one proposed use of interpretability is exactly this: watch the explanation, and if it drifts, you know something is wrong even if you can't see the attack directly.

The point of what follows is that this promise holds for some attacks and fails completely for others, and the failure is the dangerous direction.

## The attacks

An adversarial attack adds a small perturbation to the input so the model gets it wrong. I used five:

- **FGSM** and **PGD**: standard gradient-based attacks. FGSM takes one step in the direction that most increases the model's error; PGD takes many smaller steps.
- **DeepFool**: finds a minimal perturbation that pushes the input across the decision boundary.
- **FGSM-LP** and **PGD-LP**: low-pass variants of the two standard attacks. The perturbation is smoothed (Gaussian, sigma 3.0) so it contains only low-frequency content, which for EEG is a rough analogue of how real physiological signals propagate. The motivation was physiological plausibility: a perturbation that looks like it could be signal rather than noise. The effect on the explanation was not the goal; it was the surprise.

Attack success rate (the fraction of attempts that flip the prediction), and the resulting adversarial accuracy:

| Attack | ASR | Adversarial accuracy |
|---|---:|---:|
| PGD | 0.95 | 0.05 |
| FGSM | 0.92 | 0.08 |
| DeepFool | 0.88 | 0.12 |
| FGSM-LP | 0.66 | 0.34 |
| PGD-LP | 0.61 | 0.39 |

The standard attacks are devastating. The low-pass variants succeed about two-thirds of the time. If all you cared about was breaking the model, you'd never reach for the LP attacks: they're the worse weapon on the metric an attacker supposedly cares most about.

Hold that, because it's the price.

## The finding

The question this project was about: when an attack succeeds at flipping the prediction, does the explanation change too?

I measured explanation stability with the Spearman rank correlation between the IG heatmap on the clean input and the IG heatmap on the attacked input. A correlation near 1.0 means the explanation barely moved: the model is pointing at the same features in the same rank order, even though its prediction changed. A low correlation means the explanation scrambled. Averaged over all runs:

| Attack | Spearman (clean vs. attacked IG) |
|---|---:|
| FGSM-LP | 0.998 |
| PGD-LP | 0.998 |
| DeepFool | 0.91 |
| FGSM | 0.87 |
| PGD | 0.87 |

The low-pass attacks leave the explanation almost perfectly intact. The standard attacks move it.

Turn this into what a defender would actually compute: an explanation-attack-success-rate (E-ASR). Pick a threshold on the Spearman correlation; count an attack as having "fooled the explanation" if it succeeds at flipping the prediction and drives the explanation below that threshold. At a threshold of 0.97:

| Attack | E-ASR |
|---|---:|
| FGSM | 97.5% |
| PGD | 97.5% |
| DeepFool | 87.5% |
| FGSM-LP | 0.0% |
| PGD-LP | 0.0% |

This is the whole post in one table. The standard attacks disturb the explanation almost every time they succeed: a monitor would catch them. The low-pass attacks disturb it zero percent of the time. Not rarely. Not occasionally. In this data, an explanation-based detector watching for IG drift would have caught none of the successful LP attacks.

A stable explanation under attack sounds like good news: robustness, even. But here it means the opposite. The model's prediction has been corrupted; the explanation says nothing is wrong. The monitor you installed to catch exactly this situation stays silent precisely when you most need it to fire. The explanation is giving false assurance: a green light on a broken system.

So the LP attacks are, from a defender's point of view, the dangerous ones: not because they're the strongest (they're the weakest) but because they're invisible to the check. An attacker willing to accept a lower success rate can buy near-total invisibility to explanation monitoring. That is the trade.

## The threshold is doing some work, and I won't hide it

The E-ASR numbers for the standard attacks depend heavily on where you put the threshold. At 0.97, standard-attack E-ASR is ~97%. Lower it to 0.90 and it drops to around 46%; raise expectations and it climbs. The "95-97%" figure for the loud attacks is real but it's a function of a choice I made, and a skeptical reader is right to discount it accordingly.

What does not depend on the threshold is the LP result. The LP explanations sit at 0.998: so far above any reasonable threshold that they read as 0.0% E-ASR no matter where you draw the line between 0.90 and 0.99. The contrast between the two families is the robust finding. The exact height of the standard-attack bar is not. If you take one number from this post, take the LP zero, not the standard 97%.

## Architecture barely matters

A natural follow-up: maybe some of these architectures are more vulnerable than others, and that's really what's driving the result. I went looking for an architecture effect and didn't find one.

Broken down by model and attack, the pattern is the same everywhere: every architecture's explanation collapses under the standard attacks and survives the LP attacks. When I aggregated across random seeds to avoid counting the same model-run multiple times as independent evidence, what looked like a difference between architectures shrank to nothing: a between-architecture comparison that had seemed significant on the raw runs (Fisher's z, p around 0.05) became clearly non-significant after aggregation (p around 0.71).

The transferable lesson is partly methodological: the architecture "effect" lived entirely in the pseudo-replication. Before aggregating seeds I had 1,360 rows and a tempting story about state-space models behaving differently. The story did not survive proper aggregation. The answer to "is this a property of a particular architecture?" is no: it's a property of the attack. The low-pass structure of the perturbation is what keeps IG quiet, regardless of what model is being attacked.

## Why this might be happening

Recall the two free choices flagged in the IG walkthrough: the baseline, and the fact that IG's sensitivity is shaped by where along the baseline-to-input path the gradients are large. A low-pass perturbation only touches the low-frequency content of the signal: a smooth, broad nudge rather than the jagged, high-frequency noise of a standard attack. The plausible story is that this smooth perturbation moves the model's decision along a route that doesn't much change the rank order of accumulated gradients from the blank baseline. The model ends up at a different class, but the path-integrated attribution lands on roughly the same features in roughly the same order, so the Spearman rank correlation stays near 1.0. The standard attacks, by contrast, inject high-frequency structure that the early layers respond to sharply, scrambling the ranking. This is consistent with how IG is constructed; I haven't isolated it experimentally.

What that explanation implies, if it's right: the invisibility may be specific to IG, and even specific to the all-zeros baseline I used. A different explanation method, or even IG with a different baseline, might catch these attacks. I didn't test either, so I can't claim "explanations are blind to LP attacks." Only "IG, with a zero baseline, is blind to LP attacks, here." The general version is a hypothesis this work raises, not establishes.

## Limitations

These are observations from one corner of one problem. They are not laws.

- **One dataset, four selected subjects.** The selection is itself a confound: I dropped subjects where decoding was near chance, on the reasoning that an explanation-stability experiment is only meaningful on a model that actually learned the task. That choice is defensible but it does mean the result is about explanations on well-decoding models specifically.
- **One explanation method, one baseline.** The invisibility may be partly IG-specific, as discussed.
- **The E-ASR threshold is a free parameter.** The standard-attack magnitudes move a lot with it. Only the LP-vs-standard contrast is threshold-robust.
- **EEGMamba is an FFT-based approximation, not a faithful Mamba reimplementation.** The architecture-level conclusions barely depend on it, but it isn't a faithful state-space model.
- **No neuroscience ground truth.** I'm measuring whether the explanation changes, not whether it was ever correct. A stable explanation is not a validated one: "the explanation didn't move" is a statement about consistency, not accuracy.
- **The LP attacks are weaker.** This is a limitation on the threat, not the analysis: an attacker using them gives up roughly a third of their success rate. Whether the stealth is worth that cost depends entirely on whether anyone is actually watching the explanation.

## Takeaway

A low-pass adversarial attack can flip an EEG classifier's prediction while leaving its Integrated Gradients explanation essentially unchanged. Standard attacks disturb the explanation almost every time they succeed; LP attacks disturb it zero percent of the time across any reasonable threshold. The attacker pays for the stealth with a lower success rate, so it's a trade, not a defeat of the defense; but the trade is available, and it points the wrong way for anyone who wanted interpretability to be a free safety net.

If you're building anything that relies on explanation drift as an attack monitor, test it against attacks designed not to disturb the explanation, because "the explanation didn't move" turns out to be engineerable, and the cost is bounded.
