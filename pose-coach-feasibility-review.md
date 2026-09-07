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

---

## 9. Decision: which option to build

### 9.1 The recommendation

**Build Option A, and cut precise silhouette placement from v1.**

The doc already picks A, on sequencing grounds — it is the only option testable in
Week 2 with a prompt and a folder of photographs. That argument survives. But the
evidence supports a better reason, and the better reason implies a scope cut.

**The model's weakness and the product's differentiator are in different places.**

Where the VLM is bad is *metric and geometric*: Object Size Estimation 0.60–0.77
against a 0.42 random baseline (§1.1); 3D pose IoU of 9.0 against a human 54.2
(§1.2).

What the differentiating feature needs is *semantic*. "Sit on the second step, look
toward the water" requires knowing a step is there (Existence: 0.90–1.00, solved),
that there is more than one, and where the water sits relative to it (Spatial
Relation: 0.67–0.80, adequate). It does not require centimetres.

Centimetres are required by exactly one subsystem: silhouette placement in §6. The
whole scale chain — `H_REAL`, `px_per_metre`, `target_fig_px` — hangs off
`support_height` and `support_bbox`, the two least reliable fields in the schema.

So the design has coupled its riskiest dependency to its least differentiating
feature. Every competitor in §2.4 already ships silhouette overlays. None of them
can say "the second step."

### 9.2 The scope cut

Put the silhouette at a sensible default — centred, sized by `framing`, anchored
around the lower third — and let the photographer drag it. §6.5 already specifies
pan and pinch; §6.4 already concedes "the photographer can drag, and dragging is
expected."

What that buys:

- §6 — 8 hours, and the doc's own "item most likely to overrun" — leaves v1.
- The dependency on the model's weakest capability disappears.
- §12's question "does the silhouette earn its place at all?" gets answered by
  usage data instead of by 8 hours of projective geometry.
- Options B, C, D and F stop being urgent, because what made measurement necessary
  was the placement arithmetic, not the sentence.

A then reduces to: one call, one image, three sentences, a generic draggable
silhouette. Smaller than what the doc specifies, and it isolates the single bet that
determines whether the product exists.

### 9.3 What I would not build, and why

| Option | Verdict |
|---|---|
| **F** (per-pixel depth) | Strongest on paper, and §3 argues for it. But it responds to a measurement nobody has taken. It also carries an unresolved conflict: ARCore depth needs camera *motion*, the scene lock fires on camera *stillness*. Do not buy that problem before you know you need it. |
| **E** (on-device) | Best endgame profile — no calls, no bill, no signal needed, privacy in one sentence, and it keeps scene-specific language. Likely where this lands in 2027. But a ~3B on-device model, on a task where frontier models score near random on the relevant sub-skills, is not a Week-2 bet. Test it for an hour in Week 0; do not architect for it. |
| **G** (detector supplies boxes) | The right upgrade *if* placement stays. Since placement leaves v1, so does G. |
| **D**, **H** | Already shipped by at least three vendors (§2.4). A floor, not a destination. |
| **B**, **C** | Superseded by F on the perception side; both inherit the same "responds to an untaken measurement" objection. |

### 9.4 The honest counter-argument

There is a real case against this. The three-card menu already absorbs bad cards —
the doc's own scoring logic says a scene yielding one good card out of three is a
scene the product handles fine. If a photographer discards "sit on that 20cm ledge"
at a glance, metric error is cheap and I am over-weighting it.

I think that is correct for the *sentence* and wrong for the *silhouette*. A badly
placed overlay is visible on every card rather than discarded with one. Which is the
same conclusion from the other direction.

### 9.5 What would change my mind

If the Week 0 tape-measure check (§6, recommendation 6) shows `support_height`
classes are accurate — say >85% against measured ground truth on 20 objects — then
metric perception is not the bottleneck, placement is cheap, and A-as-specified is
fine with no scope cut. That check costs ten minutes and a tape measure and is the
highest-information-per-minute item in the entire plan.

---

## 10. Drop-in replacement for §1 of the design document

Written to be pasted over the existing §1. Corrects the depth premise, adds the four
missing options, and states the recommendation.

---

### 1. Design options

There are three jobs in this system, and every option below is a different answer to
who does which.

| Job | What it means |
|---|---|
| **Perception** | What is physically here, how big is it, how high, where's the light |
| **Selection** | Given all that, which pose should the person strike |
| **Language** | Write the sentence the photographer says out loud |

The AI is unambiguously good at language, decent at selection, and weak at
perception. That weakness is now measured rather than assumed, and it is narrower
than it sounds. On *semantic* questions — is there a bench, is the water to the left
— published benchmarks put frontier models at or near human level. On *metric*
questions — how high is that, would a person fit — the same models sit barely above
chance. Object size estimation scores 0.60–0.77 against a 0.42 random baseline.

That split is the reason the options exist, and it also tells you which parts of this
system are at risk. The sentence is semantic. The silhouette placement is metric.

**A note on what the phone can measure.** An earlier draft of this document said the
phone only finds flat surfaces, so a railing never appears. That is true of plane
detection and false of what is actually available. ARCore's Depth API returns
per-pixel depth, needs no time-of-flight sensor, works on non-planar and low-texture
surfaces, and covered over 88% of active Android devices as of May 2026. Railings do
appear. Two real limits remain: depth only becomes valid once the user has moved the
device, and featureless surfaces like white walls still return imprecise values. On
iOS the picture is worse, not better — LiDAR is Pro-only, and the monocular fallback
for everything else is unproven for metric output.

### The eight options

| | Perception | Selection | Language | Calls while in use |
|---|---|---|---|---|
| **A** | AI estimates | AI | AI, written per scene | 1 |
| **B** | Phone measures planes, AI interprets | AI | AI, written per scene | 1 |
| **C** | Phone measures planes | Your rules | AI, written per scene | 1 |
| **D** | Phone measures planes | Your rules | Pre-written, one per pose | 0 |
| **E** | On-device model | On-device model | On-device, per scene | 0 |
| **F** | Per-pixel depth, AI interprets | AI | AI, written per scene | 1 |
| **G** | Detector supplies boxes, AI interprets | AI | AI, written per scene | 2 |
| **H** | Scene embedding | Nearest neighbour | Pre-written, one per pose | 0 |

---

#### Option A — AI does everything

Send one photograph, get back three poses with sentences. Nothing else runs.

**Pros**
- Simplest thing that could work. One call, no sensors, no rules to maintain.
- Behaves identically on every phone, including old and low-end ones.
- Handles anything in a scene, including things you never anticipated — a fire
  escape, a stack of crates, a fallen tree.
- Its strength is the differentiator. Naming what is actually here is the one thing
  the shipped competition cannot do.

**Cons**
- It is guessing at size, and the benchmarks say this is the failure to expect most.
- It is also guessing at where things are in the frame, which is what places the
  silhouette.
- Every recommendation needs signal. No signal, no cards.

---

#### Option B — Phone measures planes, AI does the rest

As A, plus a line of measured flat surfaces in the request.

**Pros**
- Attacks the size failure directly. Still one call. Measuring is free.
- Same code as A plus one line — testable in the same week without committing.

**Cons**
- The phone reports geometry, not meaning. A 44cm surface could be a step or a bin lid.
- Planes miss railings, ledges and posts, which are among the better things to pose
  against. **Option F fixes this and should be preferred.**
- Needs visible surface detail and reasonable light.

---

#### Option C — Your rules pick the pose, AI writes the words

**Pros**
- When it picks wrong you can read your own rules and see why.
- Selection is deterministic. The AI is confined to the job it is clearly good at.
- Cheaper per call.

**Cons**
- You write and tune the rules, and they are rigid.
- Inherits every blind spot of plane measurement.
- Loses the AI's ability to notice something you never thought of.

---

#### Option D — No AI while you're using it

**Pros**
- Instant, free, works with no signal, nothing leaves the phone, least code.

**Cons**
- The sentence cannot mention where you actually are.
- **This product already exists.** At least three apps ship silhouette overlays with
  pre-written poses today. Choosing D means shipping into a solved category with no
  differentiator. That is the decisive objection, and it is commercial rather than
  technical.

---

#### Option E — Everything on the device

A vision-capable on-device model does the scene reasoning and writes the sentence
locally. Announced for iOS 27; Gemini Nano offers the equivalent on recent Android.

**Pros**
- Every operational advantage of D — instant, free, offline, private, no proxy, no
  API key, no rate limiting, no provider retention policy — while *keeping*
  scene-specific language.
- The only option that is both cheap to run and differentiated.

**Cons**
- Quality is unmeasured and likely well below frontier, on a task where frontier is
  already weak.
- Recent OS floor, and two implementations, one per platform.
- Not testable with a chat window. Needs a device harness.

---

#### Option F — Option B with real depth

As B, but the measured surfaces come from per-pixel depth rather than plane anchors.

**Pros**
- Removes the railings-and-ledges blind spot that argues against B, C and D.
- Gives real centimetres for the support-to-subject ratio, turning a guess squared
  into a single guess.
- Directly attacks the predicted dominant failure.

**Cons**
- **Depth needs camera motion; the scene lock fires on camera stillness.** That is a
  genuine architectural conflict and needs prototyping before it is committed to.
- Poor on featureless surfaces.
- The iOS non-Pro path is unproven.

---

#### Option G — A detector supplies the coordinates

Run an open-vocabulary detector first; hand the model a list of named, located
objects and let it do selection and language only.

**Pros**
- Attacks the localisation weakness where it lives. Makes the box trustworthy, which
  is what placement depends on.
- Most of the grounding filter becomes unnecessary.

**Cons**
- A second model dependency on a deliberately simple stack. Adds latency.
- A closed vocabulary loses the fire-escape-and-fallen-tree flexibility.

---

#### Option H — Retrieval instead of generation

Embed the scene, nearest-neighbour against a library of scene-to-pose pairs, return
the stored sentence.

**Pros**
- Zero calls, fully deterministic, auditable, no hallucination surface at all — you
  can only return a pose someone actually photographed.
- Improves as the library grows, which answers D's "no path to getting better".

**Cons**
- Needs a far larger library than 30.
- Sentences stay generic, so it loses the differentiator the same way D does.

---

### What this document specifies

**This document specifies Option A, with silhouette placement cut from v1**, and
treats **F** as the expected upgrade if measurement turns out to be the bottleneck,
and **E** as the expected upgrade if it does not.

The reason is that the model's weakness and the product's differentiator sit in
different places. The sentence is semantic and the model is good at that. The
placement is metric and the model is bad at that. Cutting placement to a default
position with drag-to-adjust removes the dependency on the weak capability, defers
the eight riskiest hours in the build, and leaves the one bet that decides whether
there is a product: does naming what is actually here beat a generic pose deck.

Sections that change by option:

| Section | A (as now specified) | E | F | D |
|---|---|---|---|---|
| §2 Client pipeline | as written | as written | add depth capture at lock | as written |
| §4.2 Prompt | as written | shortened for a smaller model | add measured-surfaces line | not used |
| §4.3 Filter | rules 4, 5, 7 only | same, on-device | rules 1, 2, 6 become checks against measurement | not used |
| §6 Placement | **default position + drag** | same | full geometry becomes viable | same |
| §3 Proxy | as written | **deleted entirely** | as written | **deleted entirely** |

### How to choose

Three decisions, all of which come out of work already planned.

**Decision 0 — can it judge height at all?** Week 0. Photograph ten objects whose
height you have measured with a tape. Check the returned height class against the
tape. If it is right more than four times in five, placement is cheap and the scope
cut above is unnecessary. Ten minutes, and it is the highest-information item in the
whole plan.

**Decision 1 — is perception the bottleneck?** Read the failure breakdown by class.
If most bad cards are wrong about size or location, F is the move. If most are about
pose choice or wording, measuring will not help.

**Decision 2 — do scene-specific sentences beat generic ones?** This is the
differentiator, not an optimisation. If generic sentences win or tie, the product is
Option D, and Option D is a category with several incumbents and nothing to
distinguish this entry. Treat a tie as a red flag rather than a simplification
opportunity — and note that at 20 locations a tie cannot be distinguished from a
30/70 loss, so the comparison needs more locations before it can carry this weight.
