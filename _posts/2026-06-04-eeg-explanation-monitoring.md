---
layout: post
title: "When the explanation lies: adversarial attacks that fool a model without disturbing its interpretation"
date: 2026-06-04
---

This post is about a single finding from a project on adversarial robustness in EEG classifiers, and one thing it implies for anyone who wants to use interpretability methods as a safety check. The finding is this: a particular family of adversarial attacks can reliably break a model's prediction while leaving its Integrated Gradients explanation essentially untouched. If you were watching the explanation to catch attacks, which is one of the things interpretability is supposed to be good for, these attacks would walk straight past you.

I want to be careful about the size of this claim, so let me state it precisely up front and then spend the rest of the post earning it. The claim is *not* that explanations are useless, or that all stealthy attacks evade all monitoring. The claim is narrow and empirical: in this setup, with this explanation method and these attacks, the explanation-based detector caught the loud attacks almost perfectly and the quiet ones not at all. The interesting part is that "quiet" here means quiet *to the explanation specifically*, not quiet to a human looking at the raw signal, and that the attacker pays a real, measurable price for that quietness. It is a trade, not a free lunch.

## The setup

The project trained classifiers to decode motor imagery from EEG: the task is to look at a few seconds of brain signal recorded while someone imagines moving their left hand, right hand, feet, or tongue, and predict which one they imagined. This is a standard brain-computer-interface benchmark (BCI Competition IV Dataset 2a). I worked with four subjects from that dataset (subjects 1, 3, 8, and 9), chosen for data quality rather than at random, which is a limitation I'll come back to.

I trained four model architectures: EEGNet and DeepConvNet (both convolutional networks designed for EEG), CTNet (a convolution-plus-transformer hybrid), and EEGMamba (a convolution-plus-state-space hybrid). A note on that last one: at the time I ran these experiments there was no official EEGMamba release, so I built my own adaptation of a Mamba-style state-space block bolted onto an EEG front end. It is an approximation, FFT-based rather than fully selective, and I'm flagging that here rather than presenting it as a faithful reimplementation. As it turns out, the architecture choice barely matters to the result, so not much rides on this.

Across the four models the clean classification accuracy averaged about 69% on these four subjects. That's the honest baseline. These are not heavily tuned, state-of-the-art decoders; they're competent models on a hard four-class problem. Everything below is measured relative to that 69% starting point.

It's worth being upfront about where that number sits, because a careful reader will ask. Across all nine subjects of the dataset (before I selected the four), the same models averaged closer to 55%. The published literature for these architectures on this dataset reports a wide band, anywhere from the low 60s for plain reimplementations up to the low-to-mid 80s when hyperparameters are tuned per subject or stronger variants are used. My numbers sit inside that band but below the tuned state-of-the-art, and I want to be careful not to dress them up: the reported figures come from differing train/test protocols (session-1-train/session-2-test, per-subject tuning, various cross-validation schemes), so a direct head-to-head "mine vs theirs" table would be confounded by protocol rather than informative about model quality. I'm not going to manufacture that comparison. The point of this work is *relative*, how explanation stability behaves under attack, so the absolute accuracy isn't the claim, and modest baselines don't weaken the finding. I selected the four subjects (1, 3, 8, 9) where the models trained to accuracies comparable to published per-subject results and dropped the subjects where decoding was near chance, on the reasoning that an explanation-stability experiment is only meaningful on a model that actually learned the task. That selection is also a limitation, and I return to it at the end.

### What the explanation is, and how Integrated Gradients actually works

When one of these models makes a prediction, you can ask *which parts of the input mattered*. Integrated Gradients (IG) is a standard way to answer that. Because it's the load-bearing tool in this whole post, it's worth walking through what it actually computes rather than waving at "feature importance."

Start with the naive version. The simplest way to ask "which inputs mattered" is to take the gradient of the model's output for the predicted class with respect to each input feature: each EEG channel at each time point. A large gradient means: nudge this feature a little and the model's confidence moves a lot, so this feature is locally important. The problem is that raw gradients are *local*. Neural networks saturate: once a feature has pushed the model firmly toward a class, its gradient can flatten to near zero even though that feature is exactly why the model decided what it did. The gradient at the input tells you about the slope right where you're standing, not about how you got there. So a feature can be decisive and yet have a near-zero gradient, and naive gradient attribution will wrongly call it unimportant.

IG fixes this by not looking only at the input point. It picks a *baseline*: a reference input that represents "nothing," typically all zeros, which for EEG is a flat, signal-free trace. Then it imagines a straight line in input space from that baseline to the actual input: a sequence of inputs that are 0%, 10%, 20%, ... 100% of the way from "blank signal" to "this trial." At each step along that path it computes the gradient, and then it averages those gradients and multiplies by how far each feature travelled from baseline to input. Concretely: for each feature, (feature value minus baseline value) times the average gradient along the path. The integral is over the path; the "integrated" in the name is literally this accumulation of gradients from baseline to input.

The reason this is better than the naive gradient is that it captures the feature's contribution *across the whole journey*, not just at the saturated endpoint. A feature that mattered early in the path, that did the work of moving the model toward the class before the output flattened, gets credit, even if its gradient at the final input is zero. IG also has a clean accounting property: the attributions sum to the difference between the model's output at the input and its output at the baseline. Every bit of the model's confidence is allocated to some feature. The result is a heatmap over the signal: these electrodes at these moments are why the model said "left hand," weighted by how much each contributed to moving the model away from the blank baseline.

There are two free choices buried in that procedure, and both matter later. The first is the baseline: an all-zeros EEG trace is the natural choice but not the only one, and the attribution is defined *relative to it*. The second is that IG is built on the path from baseline to input, so its sensitivity is shaped by *where along that path the gradients are large*. Keep that second point in mind; it's the most plausible reason the low-pass attacks turn out to be invisible to it.

### Why we'd trust it as a safety check

The reason this is interesting for safety, rather than just for satisfying curiosity, is the implicit promise behind it. If the explanation faithfully tracks what the model is doing, then it should be a *monitor*. When the model is behaving normally, the explanation should look normal. When something corrupts the model's decision, the explanation should look corrupted too: different electrodes, different timing, a scrambled heatmap. So one proposed use of interpretability is exactly this: watch the explanation, and if it drifts, you know something is wrong even if you can't see the attack directly.

The whole point of what follows is that this promise holds for some attacks and fails completely for others, and the failure is the dangerous direction.

## The attacks

An adversarial attack adds a small, deliberately chosen perturbation to the input so the model gets it wrong. I used five:

- **FGSM** and **PGD**: standard gradient-based attacks. FGSM takes one step in the direction that most increases the model's error; PGD takes many smaller steps. These are the workhorses of the adversarial literature.
- **DeepFool**: finds a minimal perturbation that pushes the input across the decision boundary.
- **FGSM-LP** and **PGD-LP**: low-pass variants of the two standard attacks. The perturbation is smoothed (Gaussian, sigma 3.0) so that it contains only low-frequency content, which for EEG is a rough analogue of how real physiological signals propagate. The motivation was physiological plausibility: a perturbation that looks like it could be signal rather than noise.

That last design choice, smoothing the perturbation, was made for a reason that had nothing to do with explanations. It was about making the attack look like a believable EEG artifact. The effect on the explanation was not the goal; it was the surprise.

### How well each attack works

First, do the attacks even succeed at their actual job: flipping the prediction? Here the answer splits cleanly. Attack success rate (ASR), the fraction of attempts that change the model's output:

| Attack | ASR | Adversarial accuracy |
|---|---:|---:|
| PGD | 0.95 | 0.05 |
| FGSM | 0.92 | 0.08 |
| DeepFool | 0.88 | 0.12 |
| FGSM-LP | 0.66 | 0.34 |
| PGD-LP | 0.61 | 0.39 |

The standard attacks are devastating: PGD drives accuracy from 69% down to 5%. The low-pass variants are noticeably weaker: they succeed about 60-66% of the time, and the model still classifies a third of the perturbed inputs correctly. So if all you cared about was breaking the model, you would never reach for the LP attacks. They are the worse weapon, on the metric an attacker supposedly cares most about.

Hold that thought, because it's the price.

## The finding: the explanation only notices the loud attacks

Now the question this project was actually about. When an attack succeeds at flipping the prediction, does the explanation change too?

I measured explanation stability with the Spearman rank correlation between the IG heatmap on the clean input and the IG heatmap on the attacked input. A correlation near 1.0 means the explanation barely moved: the model is pointing at the same features in the same rank order, even though its prediction changed. A low correlation means the explanation scrambled.

Averaged over all runs, by attack:

| Attack | Spearman (clean vs. attacked IG) |
|---|---:|
| FGSM-LP | 0.998 |
| PGD-LP | 0.998 |
| DeepFool | 0.91 |
| FGSM | 0.87 |
| PGD | 0.87 |

The low-pass attacks leave the explanation almost perfectly intact. A rank correlation of 0.998 means that for practical purposes the heatmap is unchanged. The standard attacks move it, though even they move it less than you might hope.

It's clearer if you turn this into the thing a defender would actually compute: an *explanation attack success rate* (E-ASR). Pick a threshold on the Spearman correlation; count an attack as having "fooled the explanation" if it succeeds at flipping the prediction *and* drives the explanation below that threshold. This is the rate at which a Spearman-watching monitor would actually fire. At a threshold of 0.97:

| Attack | E-ASR |
|---|---:|
| FGSM | 97.5% |
| PGD | 97.5% |
| DeepFool | 87.5% |
| FGSM-LP | 0.0% |
| PGD-LP | 0.0% |

This is the whole post in one table. The standard attacks disturb the explanation almost every time they succeed; a monitor would catch them. The low-pass attacks disturb it *zero* percent of the time. Not rarely. Not occasionally. In this data, an explanation-based detector watching for IG drift would have caught none of the successful LP attacks.

### Which direction is the bad one

It's worth being explicit about why the zero is the alarming number rather than the reassuring one, because it's easy to read it the wrong way.

A stable explanation under attack *sounds* like good news: robustness, even. But here it means the opposite. The model's prediction has been corrupted; the explanation says nothing is wrong. The monitor you installed to catch exactly this situation stays silent precisely when you most need it to fire. The explanation is giving false assurance: a green light on a broken system.

So the LP attacks are, from a defender's point of view, the dangerous ones. Not because they're the strongest (they're the weakest) but because they're invisible to the check. An attacker who is willing to accept a lower success rate can buy near-total invisibility to explanation monitoring. That is the trade. The standard attacks are loud and effective; the LP attacks are quiet and only moderately effective. If the defender is watching the explanation, "quiet" may be worth more than "effective."

### The threshold is doing some work, and I won't hide it

The E-ASR numbers for the standard attacks depend heavily on where you put the threshold, and I want to be straight about that because it's the most obvious way the result could be oversold. At a threshold of 0.97, standard-attack E-ASR is ~97%. Lower the threshold to 0.90 and it drops to around 46%; raise expectations and it climbs. The "95-97%" figure for the loud attacks is real but it is a function of a choice I made, and a skeptical reader is right to discount it accordingly.

What does *not* depend on the threshold is the LP result. The LP explanations sit at 0.998: so far above any reasonable threshold that they read as 0.0% E-ASR no matter where you draw the line between 0.90 and 0.99. The contrast between the two families is the robust finding. The exact height of the standard-attack bar is not. If you take one number from this post, take the LP zero, not the standard 97%.

## Architecture barely matters

A natural follow-up: maybe some of these architectures are more vulnerable than others, and that's really what's driving the result. I went looking for an architecture effect and mostly didn't find one.

Broken down by model and attack, the pattern is the same everywhere: every architecture's explanation collapses under the standard attacks and survives the LP attacks. The convolutional models, the transformer hybrid, and the state-space hybrid all tell the same story. When I aggregated across random seeds to avoid counting the same model-run multiple times as independent evidence, what looked like a difference between architectures shrank to nothing. A between-architecture comparison that had seemed significant on the raw runs (Fisher's z, p around 0.05) became clearly non-significant after aggregation (p around 0.71).

The transferable lesson here is partly methodological: the architecture "effect" lived entirely in the pseudo-replication. Before aggregating seeds I had 1,360 rows and a tempting story about state-space models behaving differently. The story did not survive proper aggregation. It's a small, concrete example of how easy it is to find an effect that is really just the same few models counted many times.

So the answer to "is this a property of a particular architecture?" is: as far as I can tell, no. It's a property of the *attack*. The low-pass structure of the perturbation is what keeps IG quiet, regardless of what model is being attacked.

## Why this might be happening

I'll keep this section short and clearly speculative, because I don't have the evidence to settle it.

Recall the two free choices I flagged in the IG walkthrough: the baseline, and the fact that IG's sensitivity is shaped by where along the baseline-to-input path the gradients are large. A low-pass perturbation only touches the low-frequency content of the signal; it's a smooth, broad nudge rather than the jagged, high-frequency noise of a standard attack. The plausible story is that this smooth perturbation moves the model's decision along a route that doesn't much change the *rank order* of accumulated gradients from the blank baseline. The model ends up at a different class, but the path-integrated attribution lands on roughly the same features in roughly the same order, so the Spearman rank correlation stays near 1.0. The standard attacks, by contrast, inject high-frequency structure that the early layers respond to sharply, which shows up as large gradients on different features along the path and scrambles the ranking. This is a hypothesis consistent with how IG is constructed; I haven't isolated it experimentally.

But notice what that explanation implies. If the invisibility comes from an interaction between the smoothing and the way IG accumulates gradients along its path, then it may be specific to IG, and even specific to the all-zeros baseline I used. A different explanation method, or even IG with a different baseline, might catch these attacks. I did not test either, so I can't claim "explanations are blind to LP attacks"; only "IG, with a zero baseline, is blind to LP attacks, here." The general version of the claim is a hypothesis this work raises, not one it establishes.

## What was built

The concrete artifact behind this is a set of EEG models, the five attacks, an IG-based explanation pipeline, and the analysis code that produces the tables above from the raw per-run results. The raw output is a row per (attack, model, subject, seed, perturbation budget) with the clean and adversarial accuracies, the ASR, and the IG stability metrics; the analysis aggregates across seeds, applies the E-ASR thresholding, and produces the by-attack and by-architecture breakdowns. None of this is large or fancy. It's enough to support the finding and no more, which is the right size for what it is.

## Methodological notes

A few decisions shaped these numbers, and since the discipline behind them is half the point of the post, they're worth making explicit rather than burying in a footnote.

**Seed aggregation, and the architecture effect that wasn't.** Each (attack, model, subject) configuration was run with five random seeds, giving 1,360 rows of raw results. The temptation is to treat all 1,360 as independent data points and run statistics on them directly. That's pseudo-replication: five seeds of the same model on the same subject are not five independent observations of "how this architecture behaves," they're five noisy looks at one thing. When I first analysed the raw rows, a between-architecture difference looked significant (Fisher's z on the correlation, p around 0.05): the state-space model seemed to behave differently. After aggregating across seeds *before* testing, so that each configuration contributed one number, the same comparison came out clearly non-significant (p around 0.71). The effect had been living entirely in the inflated sample size. This is the single methodological thing I'd most want a reader to take away: the architecture story was an artifact of counting the same models many times, and it dissolved the moment the unit of analysis was set correctly.

**The perturbation budget is in physiological units, and that's both a feature and a limitation.** Adversarial perturbations are usually bounded by an epsilon in the input's native scale. For images that's pixel intensity (the familiar 4/255, 8/255). Here the inputs are EEG voltages, so I bounded perturbations in microvolts per channel, chosen to be physiologically plausible: small enough that the perturbed signal could pass as real EEG. That makes the attacks meaningful in the domain. It also means the budgets are *not* comparable to the standard vision-literature epsilons, so you can't read these results as "equivalent to an 8/255 attack on images." The choice was deliberate for plausibility; the incomparability is the price.

**Why Spearman, not a raw difference.** Explanation stability is measured as the Spearman rank correlation between clean and attacked IG maps. The reason for rank correlation rather than, say, mean absolute difference of attribution values is that what a monitor cares about is whether the *same features* are flagged as important in the *same order*, not whether the absolute attribution magnitudes shifted up or down uniformly. A uniform rescaling of attributions would change an L2 difference but leave the rank order, and the story it tells, intact. Spearman captures "is the explanation pointing at the same things," which is the property relevant to detection.

**The E-ASR threshold is a reported choice, not a discovered constant.** As covered above, the explanation-attack-success-rate requires a threshold on the Spearman correlation, and the standard-attack numbers move a lot with it (from ~46% at 0.90 to ~97% at 0.97). I report the threshold explicitly rather than picking the most dramatic value and presenting it as fact. The LP result is threshold-robust across that whole range, which is why it, and not the standard-attack magnitude, is the load-bearing finding.

**What I had to leave incomplete.** Integrated Gradients is expensive to compute at scale, and compute was the binding constraint throughout (everything ran on Colab). The region-of-interest analysis, looking at *which* electrodes the explanations emphasised, rather than just whether the ranking moved, was only completed for one subject. So the spatial story is suggestive at best, and I don't lean on it. The headline finding doesn't need it: the LP-vs-standard contrast holds on the aggregate stability metric across all four subjects.

## Limitations

- **One dataset, four subjects.** Everything here is BCI IV-2a, subjects 1/3/8/9, chosen for data quality. I have no evidence the effect generalizes to other EEG datasets, other subjects, or other domains. It might; I didn't check.
- **One explanation method.** Everything is Integrated Gradients. As discussed, the invisibility may be partly an IG-specific phenomenon. A monitor built on a different attribution method might not have this blindspot: untested.
- **The E-ASR threshold is a free parameter.** The standard-attack magnitudes move a lot with it. Only the LP-vs-standard contrast is threshold-robust.
- **EEGMamba is an approximation.** No official release existed when I ran this, so it's my own state-space adaptation, FFT-based rather than fully selective. The architecture-level conclusions barely depend on it, but it is not a faithful Mamba.
- **No neuroscience ground truth.** I'm measuring whether the explanation *changes*, not whether it was ever *correct*. A stable explanation is not a validated one; I have no independent ground truth for what the "right" attribution should be, so "the explanation didn't move" is a statement about consistency, not accuracy.
- **The LP attacks are weaker.** This is a limitation on the threat, not the analysis: an attacker using them gives up roughly a third of their success rate. Whether the stealth is worth that cost depends entirely on whether anyone is actually watching the explanation.

These are observations from one corner of one problem. They are not laws.

## Takeaway

Two things.

First, the concrete finding: a low-pass adversarial attack can flip an EEG classifier's prediction while leaving its Integrated Gradients explanation essentially unchanged (Spearman 0.998, E-ASR 0.0% against an explanation monitor), whereas standard attacks disturb the explanation almost every time they succeed. If you were relying on explanation drift to catch attacks, the quiet attacks are exactly the ones you'd miss, and a stable explanation over a corrupted prediction is false assurance, not robustness. The attacker pays for this stealth with a lower success rate, so it's a trade, not a defeat of the defense; but the trade is available, and it points the wrong way for anyone who wanted interpretability to be a free safety net.

Second, the discipline that surfaced it: the architecture "effect" I initially thought I'd found was pseudo-replication, and it vanished under seed aggregation; the headline E-ASR for standard attacks is threshold-dependent and I've said so rather than picking the flattering number and moving on. The finding that survived is the one that's robust to those checks, the LP-vs-standard contrast, and that's the only one I'm willing to stand behind.
