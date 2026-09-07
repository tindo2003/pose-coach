# Pose Coach — Feasibility Review and Options Analysis

**Reviewer:** Feynman (verifier specialist)
**Date:** 2026-09-07
**Subject:** `pose-coach-technical-design.md` (65,701 bytes), specifically §1 Design options
**Scope:** Is the project feasible? Are the four options the right four?

---

## 0. Verdict up front

Three findings, in order of how much they should change the plan.

1. **The design's core intuition is correct and now has hard evidence behind it.**
   "The AI is good at language, decent at selection, weak at perception" is supported
   by two independent 2025–2026 benchmarks. The weakness is larger and more specific
   than the doc assumes: it is concentrated in *metric* estimation and *geometric
   grounding*, which is exactly what `support_height`, `support_bbox` and
   `subject_anchor` are asking for. See §1.

2. **One factual premise in §0 Terminology is wrong, and it invalidates the stated
   cons of Options B/C/D.** The doc says phone measurement means plane detection, so
   "a railing never appears, being a thin pipe." ARCore has shipped a per-pixel
   **Depth API** since 2020 that needs no ToF sensor, covers non-planar and
   low-texture surfaces, and as of **May 2026 runs on over 88% of active devices**.
   Railings *do* appear. See §2.1.

3. **The eval design — the part the doc correctly identifies as the whole project —
   has three statistical defects that I quantified by simulation.** The gate cannot
   distinguish a 0.75 system from a 0.85 system; the 5–8 tuning cycles on a fixed
   100-scene set clear the 0.80 gate from noise alone ~73% of the time for a truly
   0.75-quality system; and the 20-location × 15-rater comparison is analysed as
   n=300 when it is effectively n=20, which inflates the false-positive rate to
   0.245–0.382 against a nominal 0.05. See §4.

**Overall:** the project is feasible, the sequencing discipline is genuinely good, and
the "hope Option D wins" instinct is well-placed. But the option set is missing at
least three live options, and the measurement that decides everything is currently
underpowered enough that it could pass a system that does not work.

---

## 1. Evidence on the central bet: can a VLM do the perception job?

The doc bets Weeks 2–3 on the answer. Two benchmarks published since the doc's
assumptions were formed give a fairly sharp answer.

### 1.1 SIBench — metric estimation is the weakest cell in the table

*How Far are VLMs from Visual Spatial Intelligence? A Benchmark-Driven Perspective*
(2025, arXiv:2509.18905) — 23 task settings aggregated from ~20 datasets.

Selected results (Table 3 of the paper; these are multiple-choice accuracies, and the
random-choice baseline for the Basic Perception block is **0.4182**, which is the
number that makes these scores alarming rather than mediocre):

| Task | Qwen2.5-VL-72B | GPT-5 | Gemini-2.5-Pro |
|---|---|---|---|
| Object Size Estimation | 0.609 | 0.765 | 0.597 |
| Height | 0.600 | 0.680 | 0.713 |
| Reach Prediction | 0.675 | 0.575 | 0.550 |
| Relative Distance | 0.568 | 0.683 | 0.546 |
| Spatial Compatibility ("can X fit on/in Y") | 0.603 | 0.561 | 0.710 |
| Object Localization | 0.713 | 0.773 | 0.704 |
| — for contrast — | | | |
| Spatial Relation (left/right/above) | 0.798 | 0.693 | 0.666 |
| Existence ("is there a bench") | 1.000 | 0.925 | 0.900 |

The paper's own summary: models "exhibit poor quantitative reasoning, often failing
to accurately estimate physical quantities such as distance and size… This limitation
likely arises from their reliance on coarse visual heuristics rather than precise
metric representations."

**Read across to the design.** `support_height` is an Object Size / Height judgement.
The `support_ratio` arithmetic in §4.4 needs it in metres. `feasible` — one of the
four eval checkboxes — is a Spatial Compatibility judgement. These are the three
weakest columns. Meanwhile `grounded` in the "does a bench exist here" sense maps to
Existence, which is essentially solved. The doc's four checkboxes are therefore not
equally at risk, and the doc treats them as if they were.

### 1.2 BOP-Ask — the gap to humans on interaction reasoning is very large

*BOP-Ask: Object-Interaction Reasoning for Vision-Language Models* (NYU + NVIDIA,
2026, arXiv:2511.16857). Its thesis is stated in the abstract: VLMs "have achieved
impressive performance on spatial reasoning benchmarks, yet these evaluations mask
critical weaknesses in understanding object interactions."

BOP-Ask-core, with a human baseline from 40+ contributors (Table 3):

| | Human | GPT-5 | Gemini Robotics-ER 1.5 | Qwen-VL 2.5 (3B) |
|---|---|---|---|---|
| Pose (3D IoU ↑) | 54.2 | **9.0** | 24.4 | 26.5 |
| Trajectory (success ↑) | 67.3 | **0** | 43.0 | 0 |
| Object rearrangement (recall % ↑) | 44.1 | **14.8** | 48.9 | 15.2 |
| Spatial reasoning (binary, ↑) | 84.9 | 68.3 | 84.2 | 50.8 |
| Relative depth (binary, ↑) | 87.3 | 74.6 | 88.0 | 49.4 |

The shape of this table is the finding: on **binary relational** questions the best
models are at or near human level; on anything requiring **geometry a body has to
act on**, a frontier general-purpose model scores 9.0 against a human 54.2, and 0 on
trajectories.

**Two honest caveats, which matter:**
- BOP-Ask is cluttered **tabletop** scenes for robot manipulation. Pose Coach is
  outdoor street furniture at 2–3m. Transfer is not established; I am reading the
  *shape* of the failure, not the absolute numbers.
- A specialised model (Gemini Robotics-ER) beats GPT-5 by 2.7× on pose. So this is
  partly a "wrong model class" result, not purely a capability ceiling. That is
  itself useful: it argues for a specialist component, i.e. Option G below.

### 1.3 What this predicts for the Week 0 sanity check

The doc's Week 0 asks "which way does it fail?" and lists three candidates. The
literature predicts the answer fairly confidently:

- *"Describes a bench that isn't in the picture"* — **less likely than the doc
  expects.** Existence is a solved task (0.90–1.00).
- *"Bench is real but tells someone to sit on a 20cm ledge"* — **the predicted
  dominant failure.** Size/compatibility is the weakest cell.
- *"Sentences read like a textbook"* — a style problem, cheaply fixed by examples.

If Week 0 confirms this, the §4.3 filter is *not* the priority (the doc's first
branch), and the feasibility paragraph in the prompt cannot fix it either, because
prompting does not install a metric representation the model lacks. That points at
Option B/F — get the measurement from somewhere other than the model.

---

## 2. Factual corrections to the design document

### 2.1 The Depth API premise is wrong (material)

§0 Terminology states surface measurement means flat-plane detection, and derives
three limits: no railings, nothing on plain walls, and geometry-without-meaning.
Options B, C and D all inherit "no railings, no ledges, no posts" as a stated con.

From Google's own documentation (https://developers.google.com/ar/develop/depth):

- The Depth API returns **per-pixel depth images**, not planes: "Each pixel in a
  depth image is associated with a measurement of how far the scene is from the
  camera."
- It explicitly beats planes on exactly the doc's complaint: "Plane hit-tests only
  work on planar surfaces with texture, whereas **depth hit-tests are more detailed
  and work even on non-planar and low-texture areas**."
- **No ToF sensor required** — depth-from-motion plus ML.
- Range: "robust, accurate depth estimates from 0 to 65 meters away. The most
  accurate results come when the device is half a meter to about five meters away" —
  which brackets the 2–3m posing distance well.
- Coverage: **"As of May 2026, over 88% of active devices support the Depth API"**
  (https://developers.google.com/ar/devices).

The doc's two *other* stated limits survive and should be kept: depth "is only
available after the user has started moving their device around," and "surfaces with
few or no features, such as white walls, will be associated with imprecise depth."
There is also a **Raw Depth API** that ships a per-pixel confidence image, which is
strictly better input for a feasibility rule than a point estimate.

**Consequence:** the con list for Options B/C/D is overstated. A railing at 91cm *is*
measurable. This makes Option B materially stronger than the doc presents it, and it
weakens the argument that Option A must come first.

### 2.2 iOS is the asymmetric platform, not Android

- LiDAR is **Pro-only** on iPhone (12 Pro through 17 Pro; no standard, Plus, Air,
  mini or SE model has it). So the iOS "measured depth" path splits the install base.
- The iOS substitute is monocular depth via Core ML. Apple publishes a converted
  Depth Anything V2 small model (https://huggingface.co/apple/coreml-depth-anything-v2-small),
  and Apple's own Depth Pro paper reports "a 2.25-megapixel depth map in 0.3 seconds"
  (arXiv:2410.02073) — **on a standard GPU, not a phone**, so do not budget that
  latency for mobile.
- ⚠️ **Unverified / caution:** a third-party repo claims ~40–60 ms Depth Anything V2
  inference on iPhone. I could not verify this against a primary source and it is a
  hackathon project. Also note Depth Anything V2 small is a **relative** depth model;
  Pose Coach needs **metric** depth (centimetres), which is a different and harder
  output. Treat "monocular metric depth on iPhone" as **unproven for this use case**
  until measured.

So the platform asymmetry is the reverse of the doc's assumption: **Android has broad
cheap per-pixel depth (88%); iOS has it well only on Pro devices.**

### 2.3 On-device VLMs with image input now exist on both platforms

Verified from Apple's WWDC26 session transcript
(https://developer.apple.com/videos/play/wwdc2026/241/):

> "In addition the on-device model is also gaining Vision capabilities, which unlocks
> entire new categories of applications… Simply insert an image attachment into your
> prompt, together with text… The model supports images in any size and aspect ratio…
> Arbitrary image sizes are allowed, but bear in mind that larger images will consume
> more tokens and incur more latency."

The same session announces a `PrivateCloudComputeLanguageModel` requiring no API key
and storing no prompts. On the Android side, Gemini Nano supports multimodal
on-device prompting via Android 16's AI Core APIs.

⚠️ **Timing caveat:** this is announced for the **iOS 27** cycle. Confirm GA
availability and, critically, *measure quality* before designing around it — a ~3B
on-device model will be considerably weaker at this task than a frontier API model,
and SIBench shows even frontier models are weak here.

**Consequence:** "one call to a third-party HTTP API" is no longer the only way to get
a scene-specific sentence. That is a fifth option the doc does not have.

### 2.4 The product category is already crowded — this affects the null hypothesis

Live shipping apps found on the App Store / Play Store (2026):

| App | What it does | Which doc option it is |
|---|---|---|
| Posei (`id6763751241`) | "AI silhouette guides every angle, then auto-captures. 200+ poses" | ≈ Option D + auto-shutter |
| Spot Pose / Pose AI (`id6764609323`) | "Choose a silhouette, align the person with the guide" | ≈ Option D |
| Pose Genius (`id6755207657`) | "Uses an AI camera to **scan your scene** and show you poses" | claims ≈ Option A/C |
| Photogenik (photogenik.app) | "live coaching for the pose, frame, light, and expression", 656 ratings @ 4.8 | ≈ Option A superset |
| Posed (`id6762599608`) | "realtime AI pose assistant" | ≈ Option A |

**This is the most commercially important finding in this review and it is absent
from the doc.** Option D — 30 silhouettes, pre-written sentences, no calls — is
already a shipped product from at least three vendors. The doc's stated hope ("it is
worth explicitly hoping the generic sentence wins") leads to a product with no
differentiation whatsoever.

That inverts the strategic logic. The *only* defensible version of this product is the
one where scene-specific perception demonstrably beats a generic pose deck — which is
precisely the comparison in §9.6. So §9.6 is not "the test that might let us delete
the AI." It is **the test that determines whether there is a product at all.** It
deserves far more statistical care than it currently gets (§4.3 below).

⚠️ I verified these apps exist and quoted their own marketing copy. I did **not**
evaluate their quality, install base, or whether their "AI scene scanning" claims are
real. That is a recommended next step (§6).

---

## 3. Options the document does not consider

The doc's four options vary *who does perception/selection/language*. All four assume
perception is either "AI guesses" or "ARKit/ARCore planes." Relaxing that assumption
produces four more.

| | Perception | Selection | Language | Calls in use | Status |
|---|---|---|---|---|---|
| A | AI estimates | AI | AI per scene | 1 | in doc |
| B | Planes + AI | AI | AI per scene | 1 | in doc |
| C | Planes | Rules | AI per scene | 1 | in doc |
| D | Planes | Rules | Pre-written | 0 | in doc |
| **E** | **On-device VLM** | on-device | on-device per scene | **0** | **new** |
| **F** | **Per-pixel depth** + AI | AI | AI per scene | 1 | **new** |
| **G** | **Detector supplies boxes**, AI reasons | AI | AI per scene | 2 | **new** |
| **H** | Scene embedding → retrieval | nearest-neighbour | Pre-written | 0 | **new** |

### Option E — Everything on device
Vision-capable on-device model (Apple Foundation Models in iOS 27; Gemini Nano on
Android 16+) does scene reasoning and sentence writing locally.

**Pros.** Zero marginal cost, zero latency from network, works with no signal
(the doc itself notes "good photo spots are frequently places with bad reception"),
privacy story becomes trivially one sentence, no proxy, no API key, no rate limiting,
no provider retention policy to negotiate. Deletes §3 entirely, like Option D, but
*keeps* scene-specific language — which §2.4 says is the only differentiator.
**Cons.** Quality is the open question and is likely well below frontier. Platform
version floor is high and recent. Two implementations, one per platform. Not testable
in Week 2 with a chat window — needs a device harness.
**Why it matters:** it is the only option that gets Option D's operational profile
*and* Option A's differentiation. If it works, it is the best product on this list.

### Option F — Option B, but with real depth instead of planes
Same as B, but the measured-surfaces line is derived from the ARCore Depth API /
Raw Depth (88% of Android devices, no ToF) rather than plane anchors, with LiDAR on
iPhone Pro and a **measured** monocular fallback elsewhere.

**Pros.** Removes the "no railings, no ledges, no posts" con that the doc uses to
argue *against* B/C/D. Gives real centimetres for the `support_ratio` arithmetic in
§4.4, converting "a guess squared" into one estimate. Directly attacks the
predicted-dominant failure from §1.3.
**Cons.** Requires camera motion before depth is valid, which fights the design's
"stability triggers the lock" mechanism — **this is a genuine architectural tension
and should be prototyped early**. Poor on featureless surfaces. iOS non-Pro path is
unproven (§2.2).
**Recommendation: F should replace B as "the expected upgrade."**

### Option G — Detector supplies the coordinates
Invert the §9.2 contingency. Run an open-vocabulary detector (Grounding DINO, OWLv2,
YOLO-World, or on-device Vision/MediaPipe) *first*; hand the VLM a list of named,
localised objects and let it do selection and language only.

**Pros.** Attacks the localisation weakness directly. §1.2's evidence that a
specialised model (Gemini Robotics-ER, 24.4 IoU) beats a generalist (GPT-5, 9.0)
supports the general principle. Makes `support_bbox` trustworthy, which is what
silhouette placement in §6 depends on. Filter rules 1, 2, 3 and 6 become unnecessary.
**Cons.** Second model dependency, which the doc explicitly wants to avoid. Adds
latency. Closed detector vocabulary loses the "fire escape, stack of crates, fallen
tree" flexibility the doc values in Option A.
**Note:** the doc already contemplates this as a §9.2 contingency. My point is that it
should be a *first-class option evaluated in the same cycle*, not a fallback, because
the evidence predicts it will be needed.

### Option H — Retrieval, not generation
Embed the scene (CLIP-class encoder, on-device). Nearest-neighbour against a curated
library of scene→pose pairs built from the reference shoot. Return the stored sentence.

**Pros.** Zero calls, fully deterministic, auditable, no hallucination surface at all
(you can only return a pose someone actually photographed), improves monotonically as
the library grows — which answers the doc's Option D con "no path to getting better
except shooting more poses."
**Cons.** Needs a much larger reference library than 30. Sentences are generic per
pose, so it fails the §9.6 differentiator the same way D does.
**Verdict:** worth naming as the strongest version of "no AI on the live path," and a
better fallback than D, but it does not solve the differentiation problem.

### The option that is missing from the null hypothesis
The doc's Baseline 1 is "random pose from the correct support bucket," which requires
a human to eyeball the support class. Given §2.4, the honest commercial null is
**"a shuffled deck of 30 poses with no perception at all"** — because that is what
several shipping apps do, and it costs nothing. Add it as Baseline 0.

---

## 4. Evaluation design defects (quantified)

I ran the arithmetic rather than asserting it.
Script: `experiments/pose_coach_eval_power.py` · Output: `experiments/pose_coach_eval_power_results.txt`

*Method note: scipy in this environment is ABI-broken (numpy 2.4.6 vs scipy 1.11.4),
so all statistics are implemented exactly in pure Python — exact binomial tail sums
via `math.comb`, Wilson score intervals, and a t-test using the hardcoded two-sided
5% critical value for df=19. Nothing is approximated silently. Simulations use a
fixed seed (20260907); 20,000 trials for Q2, 6,000 for Q3.*

### 4.1 The gate cannot resolve the difference it is being asked to resolve

n=100 scenes, primary metric `at_least_one_good_per_scene`, gate ≥ 0.80:

| Observed | 95% CI (Wilson) | Width |
|---|---|---|
| 78/100 | [0.689, 0.850] | 0.161 |
| 80/100 | [0.711, 0.867] | 0.155 |
| 85/100 | [0.767, 0.907] | 0.140 |

P(a single pass clears ≥80/100) by true rate: 0.70 → **0.016**; 0.75 → **0.149**;
0.80 → 0.559; 0.85 → **0.934**; 0.90 → 0.999.

The good news: a genuinely bad system (0.70) almost never passes. The bad news: a
0.75 system passes 15% of the time and a 0.85 system fails 11% of the time. **A single
100-scene pass cannot separate 0.75 from 0.85.** The zero-bucket criterion is worse:
an observed 8/100 has a 95% CI of [0.041, 0.150], straddling the 10% threshold.

### 4.2 The tuning loop clears the gate from noise alone

The doc runs the same 100 scenes 5–8 times, changes the prompt each cycle, and stops
when the score clears. There is **no held-out scene split**. I simulated the case
where prompt edits do nothing at all (true rate fixed) and the loop reports the best:

| True rate | Mean single pass | Mean best-of-8 | **P(some cycle ≥ 0.80)** |
|---|---|---|---|
| 0.65 | 0.650 | 0.717 | 0.007 |
| 0.70 | 0.699 | 0.764 | 0.125 |
| **0.75** | 0.750 | **0.810** | **0.727** |
| 0.78 | 0.780 | 0.838 | 0.974 |

**A genuinely 0.75-quality system clears the 0.80 gate on some cycle 73% of the
time, purely from selection noise.** This isolates optional stopping only; real
per-cycle model variance would add to it, and genuine prompt improvement would work
against it. The reported passing number is the maximum of 8 correlated draws, and the
doc treats it as an unbiased estimate.

**Fix (cheap, changes nothing else):** split the 100 scenes into 60 tune / 40 holdout
at the start of Week 1. Tune freely on the 60. Run the holdout **once**, at the end,
and gate on that. The doc's own discipline rule — "if it isn't needed to run the eval,
don't build it" — has an obvious companion: *if you tuned on it, you cannot gate on it.*

### 4.3 The photo comparison is analysed at the wrong unit

20 locations × 15 raters = 300 votes. Votes within a location are correlated: if the
direction was bad *at that spot*, most raters agree. I modelled this as a per-location
logit random effect (sd = τ; τ=0.8 mild, τ=1.5 strong).

**Type-I error, true preference exactly 0.50, nominal α=0.05:**

| | Naive pooled n=300 | Cluster-level n=20 |
|---|---|---|
| τ = 0.8 | **0.245** | 0.047 |
| τ = 1.5 | **0.382** | 0.050 |

Pooling 300 votes gives a false-positive rate of 25–38% where 5% is intended. **On a
coin-flip-equal comparison, the naive analysis declares a winner up to ~38% of the
time.** Given §2.4 — that this comparison decides whether the product is
differentiated — this is the single most dangerous defect in the plan.

**Power, using the correct cluster-level test:**

| True preference | τ=0.8 | τ=1.5 |
|---|---|---|
| 0.55 | 0.141 | 0.079 |
| 0.60 | 0.429 | 0.192 |
| 0.65 | 0.780 | 0.377 |
| 0.70 | 0.957 | 0.601 |

Only a *large* true preference (~0.65–0.70+) is detectable at 20 locations. A
real-but-modest edge is invisible.

**Fixes:** analyse at the location level (or fit a mixed-effects model with
location and rater as random effects); increase **locations**, not raters — raters are
cheap and add little, locations are what the test is actually sampling.

### 4.4 "Wins or ties" is an equivalence claim and cannot be made at n=20

§9.6 says Option D becomes available "if the generic sentence wins **or ties**."
Absence of a significant difference is not evidence of equivalence. At n=20 locations:

| AI preferred at | 95% CI (Wilson) |
|---|---|
| 8/20 | [0.219, 0.613] |
| **10/20 ("a tie")** | **[0.299, 0.701]** |
| 12/20 | [0.387, 0.781] |

A 10/20 "tie" is consistent with anything from a ~30/70 loss to a ~70/30 win.
Locations needed to bound the true rate (TOST, α=0.05, power 0.80): ±15pp → ~69;
±10pp → ~155; ±5pp → ~619.

**The project's most consequential decision is being made on roughly an order of
magnitude too little data.** Either declare an explicit equivalence margin and power
for it, or reframe §9.6 honestly as a directional smell test that cannot license
deleting the AI.

### 4.5 §9.2 asks the model to grade its own homework

The bbox agreement check sends a second call — *"What object occupies region
[x1,y1,x2,y2]?"* — and fuzzy-matches against `support_object`. If this is the **same
model on the same image**, errors are correlated: a model that hallucinated a railing
in an empty corner is disproportionately likely to see a railing there when asked
again. This is a structural inference on my part, not a measured result for this
system, but it is the standard concern with self-verification, and there is an active
literature on when self-checking gives "genuine error detection rather than
agreement-biased" agreement (e.g. arXiv:2605.10850, arXiv:2607.08065).

**Fixes, in increasing order of cost:** (a) use a *different* model for the check;
(b) include decoy regions containing nothing, and measure the false-agreement rate as
a control — cheap and it calibrates the whole metric; (c) use an open-vocabulary
detector as the referee, which also gives you the Option G prototype for free.

---

## 5. Smaller technical notes

- **§2.4, 768px.** Reasonable, but the doc's §12 question "is 768px enough?" is
  answered by §9.2, whose own validity is in question (§4.5). Fix §9.2 first.
- **§6.1, `H_REAL` constants.** `{knee: 0.50, hip: 0.95, chest: 1.40}` — the doc
  already flags "knee spans 0.4–0.6m in the wild." SIBench's Object Size Estimation
  scores say the *class label itself* is unreliable, so the error compounds before
  the constant is even applied. Under Option F this whole path deletes.
- **§4.4 `support_ratio`.** The doc's "a guess squared" phrasing is exactly right and
  is the strongest argument in the document for measured depth. It is also the clearest
  argument for Option F over Option B specifically.
- **§3.4, temperature 0.7 for diversity.** Reasonable, but note it directly fights
  reproducibility of the eval — the same scene will produce different cards on
  different cycles, which *adds* to the run-to-run variance modelled in §4.2. Consider
  logging seeds, or running each eval pass k times and averaging.
- **§4.3 filter rule 5 (unique `pose_id`) + backfill.** If the model returns three
  cards and two are dropped, two of three cards are fallback standing poses. The
  contract "always exactly three" is honoured while the *product* silently degrades.
  Recommend logging and gating on **model-card survival rate**, not just card count.
- **Subject height.** §4.4 correctly identifies that `support_ratio` needs subject
  height. Note that a person is a very good metric reference *once they are in frame* —
  on-device human pose estimation (Apple Vision `VNDetectHumanBodyPose`, MediaPipe)
  is free, and if the recommendation ever refreshes after the subject walks over, it
  supplies the scale reference the whole §6.1 estimation chain is trying to guess.
  The CUJ forbids this at first lock, but a *second* pass is not forbidden.

---

## 6. Recommended changes, ranked

**Do before Week 1 (cheap, high leverage):**

1. **Split the 100 scenes 60/40 into tune/holdout.** Gate on the holdout, once.
   Cost: zero. Removes the §4.2 defect entirely.
2. **Add decoy regions to the §9.2 check** and report the false-agreement rate.
   Cost: ~30 minutes. Makes the number interpretable.
3. **Re-plan the photo comparison around locations, not raters.** Target ~40+
   locations if the Option D decision is to be load-bearing; drop to 5–8 raters per
   location to pay for it. Analyse at the location level.
4. **Add Baseline 0: a shuffled 30-pose deck with no perception.** This is what the
   competition ships (§2.4) and it is the real bar.
5. **Correct §0 Terminology** on ARCore depth, and rewrite the B/C/D con lists that
   depend on the plane-detection premise.

**Do during Week 0 (2 hours, unchanged in spirit):**

6. Add a fourth thing to look for: **does it get *heights* right?** Photograph five
   objects whose height you have actually measured with a tape, and check the returned
   `support_height` class against ground truth. The literature predicts this is where
   it breaks; ten minutes of tape measure turns a prediction into a measurement.
7. **Test the on-device model in the same session** (Option E). One extra hour tells
   you whether the zero-call, zero-cost, works-offline path is live.

**Restructure §1:**

8. **Replace Option B with Option F** (per-pixel depth rather than planes) as the
   named expected upgrade.
9. **Promote Option E to a first-class option** and Option G from §9.2 contingency
   to an evaluated alternative.
10. **Reframe §9.6.** It is not "the test that might let us delete the AI." Given the
    competitive landscape, it is "the test that determines whether this product is
    differentiated from three apps already on the App Store." Same experiment,
    opposite hope.

**Keep, unchanged — these are good:**

- The Week 0 → Week 1 → gate sequencing, and deferring tracing to Week 4.
- Scoring on scenes, not cards, and the zero-bucket as the number that matters.
- Four diagnostic checkboxes rather than a single verdict.
- Matched pairs at identical locations, and the explicit naming of the
  Week-1-reference-set comparison as a trap. That reasoning is correct and unusually
  well-argued.
- The 6/24 sentence holdout.
- "Do not start Week 4 on a near miss."

---

## 7. What I did not verify

Stated explicitly so nothing here reads as more settled than it is.

- **No experiment was run against any VLM on any pose-coaching scene.** Every claim
  about model capability is transferred from published benchmarks in adjacent domains
  (SIBench: mixed; BOP-Ask: tabletop robotics). Transfer to outdoor street furniture
  at 2–3m is **not established**.
- **The competitor apps were not tested.** I verified they exist and quoted their own
  store copy. Whether Pose Genius's "AI camera scans your scene" is real is unknown.
- **iOS 27 Foundation Models vision is verified as announced** (Apple's own WWDC26
  transcript), **not** verified as generally available today, and its *quality* on
  this task is entirely unmeasured.
- **The ~40–60ms Depth Anything V2 iPhone latency claim is unverified** and comes from
  a hackathon repository. Depth Anything V2 small is relative, not metric, depth.
- **§4.5 (self-verification correlation) is a structural inference**, not a measured
  result for this system. The cited papers establish the general concern.
- **The simulations in §4 use assumed parameters** — particularly the clustering
  strength τ, which I chose as plausible bracketing values (0.8 and 1.5) rather than
  estimating from data. The Type-I error direction is robust to that choice; the
  precise power numbers are not.
- ARCore's 88% figure is Google's own marketing-adjacent statistic on its own docs
  page; I have no independent audit of it.

---

## 8. Sources

**Papers**
- Yu et al., *How Far are VLMs from Visual Spatial Intelligence? A Benchmark-Driven
  Perspective*, 2025. arXiv:2509.18905 — https://arxiv.org/abs/2509.18905
- Bhat et al. (NYU, NVIDIA), *BOP-Ask: Object-Interaction Reasoning for Vision-Language
  Models*, 2026. arXiv:2511.16857 — https://arxiv.org/abs/2511.16857
- Cheng et al., *SpatialRGPT: Grounded Spatial Reasoning in Vision-Language Models*,
  2024. arXiv:2406.01584 — https://arxiv.org/abs/2406.01584
- Bochkovskii et al. (Apple), *Depth Pro: Sharp Monocular Metric Depth in Less Than a
  Second*, 2024. arXiv:2410.02073 — https://arxiv.org/abs/2410.02073
- *Mapping the Reliability Boundary of Self-Verification in Medical VQA*,
  arXiv:2605.10850 — https://arxiv.org/abs/2605.10850
- *When LLMs Agree, Are They Right? Auditing Self-Consistency*,
  arXiv:2607.08065 — https://arxiv.org/abs/2607.08065

**Primary platform documentation**
- ARCore Depth API — https://developers.google.com/ar/develop/depth
- ARCore Raw Depth API — https://developers.google.com/ar/develop/java/depth/raw-depth
- ARCore supported devices (88% figure, May 2026) — https://developers.google.com/ar/devices
- Apple, *What's new in the Foundation Models framework*, WWDC26 session 241 —
  https://developer.apple.com/videos/play/wwdc2026/241/
- Apple, *Introducing the Third Generation of Apple's Foundation Models* —
  https://machinelearning.apple.com/research/introducing-third-generation-of-apple-foundation-models
- Apple Core ML Depth Anything V2 small — https://huggingface.co/apple/coreml-depth-anything-v2-small
- ARKit plane detection — https://developer.apple.com/documentation/arkit/tracking-and-visualizing-planes

**Competitive landscape**
- Posei — https://apps.apple.com/us/app/posei-ai-pose-camera-guide/id6763751241
- Pose Genius — https://apps.apple.com/us/app/pose-genius/id6755207657
- Spot Pose / Pose AI — https://apps.apple.com/us/app/pose-ai-photo-posing-guide/id6764609323
- Posed: AI Pose Coach — https://apps.apple.com/us/app/posed-ai-pose-coach/id6762599608
- Photogenik — https://photogenik.app/
- AI Posing (Android) — https://play.google.com/store/apps/details?id=com.aiposing.pose

**Artifacts produced by this review**
- `experiments/pose_coach_eval_power.py` — reproducible statistical checks
- `experiments/pose_coach_eval_power_results.txt` — captured output
