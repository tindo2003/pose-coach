# Pose Coach — PRD

**Status:** Draft for build
**Owner:** TBD
**Last updated:** September 2026

---

## 1. Problem

People don't know what to do with their bodies when someone points a camera at them. The friend holding the phone usually doesn't know what to tell them either. The result is a stiff photo, a couple of half-hearted retries, and everyone giving up before anyone's happy with the picture.

The gap isn't camera quality. Phones already take excellent photos. The gap is **direction** — knowing that this particular spot has a low wall you can perch on, that the light is coming from the left so the subject should turn into it, and that the whole thing takes six words to say out loud.

## 2. Opportunity

A frontier vision-language model can look at a scene and describe what's physically in it. If it can also propose what a person should do in that scene — grounded in the actual railing, the actual steps, the actual light — then the direction problem is solvable with one API call and no on-device machine learning.

Whether it can do that *well* is unproven. Section 10 is the gate.

## 3. Target user and use case

**Primary and only case for v1: the pair.** Two friends, one interesting spot, forty seconds of goodwill before someone gets self-conscious. The photographer holds the phone; the subject stands three metres away.

This scoping decision drives most of the architecture. Because a human is standing right there translating the screen into speech, the app never needs to close the gap between a display and a body itself. That removes the entire real-time pose-tracking stack.

**Explicitly out of scope for v1:** the solo tripod case (phone propped up, subject walks out to frame). It's a real use case, it's the one where machine feedback genuinely beats a friend, and it needs a completely different feedback channel — audio cues, full-screen colour states, a countdown. Revisit after the pair case is validated.

## 4. Non-goals

Listed explicitly because each was considered and cut, and each will be proposed again by someone who hasn't read this section.

| Not building | Why |
|---|---|
| Real-time pose estimation (MediaPipe / YOLO-Pose) | The photographer's eyes do this job |
| Keypoint match scoring, red-to-green feedback | Same |
| Auto-shutter on pose match | The photographer's finger is already on the button |
| Vector database, embeddings, kNN retrieval | Generation replaces retrieval; a library can't produce a pose nobody shot |
| Scraped reference corpus | Copyright and biometrics exposure, and scraped photos are unreproducible on a phone |
| Text-to-speech direction | Turns "a friend directing you" into "both of us being supervised by a machine" |
| Photorealistic preview of the actual subject | Identity preservation problems, uncanny valley, consent surface, no upside |
| Accounts, social feed, sharing | Not the problem being solved |

## 5. Core user journey

1. **Open → viewfinder in under a second.** The camera is the home screen. No login, no vibe picker, no onboarding carousel.
2. **Photographer points at the scene.** The subject is typically still walking over. Recommendations fire on the *scene*, before the subject arrives — this is what makes a multi-second API call affordable, because the dead time already exists in the interaction.
3. **Scene lock on stability.** Optical flow below threshold for ~700ms triggers a single capture (frame + IMU pitch) and a single API call. The result freezes. No re-fire until the scene drifts materially — re-querying continuously would swap the target while someone is mid-move.
4. **Three cards appear.** Each shows a silhouette placed into the scene with one sentence beneath. Swipeable.
5. **Photographer taps one.** The silhouette pins to the live viewfinder at its anchor point.
6. **Photographer reads the sentence aloud.** *"Sit on the second step, elbows on your knees, look toward the street."* This is the mechanic.
7. **Subject moves. Photographer frames and shoots.** Standard shutter.
8. **Same spot again → cards served from cache.**

**Timing budget:** open to cards under 5s. Cards to shutter is human-paced, 15–40s.

### Why three cards and not one

A single recommendation reads as an instruction and invites rejection. Three read as a menu and invite choice. Three also absorbs model error at display time — one bad card out of three is survivable, which meaningfully changes the accuracy bar the model has to clear. See §10 for how this changes the eval metric.

### Failure beats

| Condition | Behaviour |
|---|---|
| No usable support detected | Three generic no-support poses. Never an empty state. |
| Card fails the grounding check | Drop it. Show two rather than one wrong. |
| API timeout or error | Bundled fallback poses after 8s. Never an unresolved spinner. |
| Silhouette placed absurdly | Drag to reposition, pinch to scale. Manual override always available. |
| "These are all bad" | One swipe dismisses the strip; it's a normal camera. |

## 6. Functional requirements

**P0 — required to ship**

- Camera viewfinder as launch screen, cold start to preview under 1s
- Optical-flow stability trigger with IMU pitch capture at lock
- Sharpest-frame selection across the stability window (Laplacian variance)
- Single VLM call via proxy, schema-constrained JSON response
- Server-side grounding filter, dropping cards that fail validation
- Three-card strip with placed silhouettes and direction sentences
- Pitch-corrected silhouette placement, mirrored by facing
- Tap to pin silhouette to live viewfinder
- Manual reposition and rescale of pinned silhouette
- Standard shutter and camera roll save
- Client-side perceptual-hash cache, in-memory LRU
- Bundled fallback pose set for timeout and no-support cases
- Basic instrumentation (§11)

**P1 — fast follow**

- Swipe-to-dismiss returning to plain camera
- Mirror toggle on pinned silhouette
- Recent-scenes cache surviving app backgrounding

**P2 — only if silhouettes prove insufficient for confident selection**

- Generated preview on card tap: image-model inpaint, generic faceless figure, deliberately stylised. On tap only, never for all three. A polished render sets an expectation the real photo won't meet, and that gap makes a fine photo feel like a failure.

## 7. Architecture

Nothing runs per-frame except optical flow and IMU sampling. No on-device ML models.

**Live path**

```
frame + pitch → stability trigger → sharpest frame → proxy → VLM
  → schema-validated JSON → grounding filter → pitch-corrected
  silhouette placement → three cards → tap → pin → shutter
```

**Offline, once**

```
shoot 30 references (logging pitch + distance) → trace to outline PNGs
  → write 30 direction sentences → 6 become few-shot examples,
  all 30 become the eval baseline
```

### The proxy

A single serverless endpoint. Receives image and pitch, attaches the provider key, calls the model, validates against schema, filters bad cards, returns survivors.

Thin, but not optional:

- An API key in a mobile binary is an extracted API key
- Rate limiting per install, before someone finds the endpoint
- Prompt and model changes without an app release — during the eval phase the prompt changes daily, and app review is not an iteration loop

Retains nothing. No server-side caching keyed on image content; the privacy story stays simple.

### Request details

- Frame resized to 768px long edge, JPEG quality 80 (~60–100KB). Sufficient to identify a railing and locate it; far cheaper in latency and tokens than a full-resolution capture.
- Pitch passed as a text line alongside the image. It informs placement *and* the recommendation — a steeply angled phone implies a different framing situation.
- Structured output via the provider's schema-constrained mechanism, not a "return JSON only" instruction. The difference is reliable parsing versus writing a fence-stripper.
- Low but non-zero temperature. Zero tends to produce three variations on one idea rather than three distinct options.
- One call returning three cards, not three parallel calls. Three calls cost 3× and give three independent answers with no guarantee of diversity.
- 8s hard timeout, one retry on 5xx, then bundled fallback.

### Response schema

| Field | Purpose |
|---|---|
| `support_object` | What the pose keys on ("concrete steps") |
| `support_bbox` | Placement, and the grounding check |
| `support_height` | ground / knee / hip / chest |
| `subject_anchor` | Normalised x,y contact point — hips on step, shoulder on wall |
| `facing` | toward / away / three-quarter left / right — drives mirroring |
| `framing` | full / waist / close — drives silhouette scale |
| `light_direction` | Where the dominant light comes from |
| `light_quality` | open shade / direct sun / backlit / mottled |
| `pose_id` | Nearest of the 30 traced silhouettes |
| `direction` | One sentence, second person, body-part-first, under 15 words |

### The prompt

The schema enforces shape. The prompt supplies judgment. It carries:

- **Role and task.** Direct a photograph in this scene; propose three poses a person could actually strike here.
- **Grounding constraint.** Only reference objects visible in the frame.
- **Lighting as a generation constraint, not a post-filter.** Favour poses where the subject faces or is side-lit by the dominant source; avoid heavy backlight and mottled tree shadow. This should surface in the sentence itself — "look toward the street" is doing lighting work when the street is where the light is. The two lighting fields exist so the eval can verify it's happening.
- **Diversity constraint.** The three options must differ in kind, not in wording.
- **Register constraint.** Under 15 words, second person, body parts first, no photography jargon.
- **Six hand-written sentences, verbatim.** The highest-return element in the prompt. Examples teach register, length, vocabulary and ordering simultaneously, far better than describing them. This is why the reference set still matters even though it is no longer a retrieval library.

The prompt is not written once and shipped. It is the artifact produced by the eval loop in §10, and the version at the end of that week will look nothing like the version at the start. That difference is most of the product.

### Rendering

Place the `pose_id` silhouette at `subject_anchor`, scaled by `framing`, mirrored by `facing`, then apply a vertical squash proportional to the delta between live IMU pitch and the pitch logged for that reference shot. Crude perspective correction, free, and it closes most of the geometry gap — better than discarding poses whose horizon doesn't match.

Outline rather than filled, so it doesn't obscure what the photographer is composing. Deterministic and instant. No image generation on the live path.

## 8. The reference set

Thirty photographs shot in-house in one afternoon, each paired with one sentence of direction.

Six locations within a few blocks — plain wall, steps, railing, doorway, open pavement, bench. **Vary the support, not the scenery.** A leaning pose is a leaning pose whether the wall is brick or concrete; shooting the same buckets across six neighbourhoods produces near-duplicates at the cost of a day.

Shoot at phone height and phone distance, and **log pitch angle and rough distance for every frame**. That log is what the live pitch correction corrects *from*; reconstructing it from memory three months later is miserable.

Shoot originals rather than scraping. No copyright or biometrics exposure, and scraped aesthetic portraits are made on real cameras in good light — beautiful, and useless as instructions, because nobody reproduces a 50mm golden-hour portrait with a phone at arm's length on a Tuesday. Get written consent from anyone photographed, even by text message.

**Distribution:** 4 full-body unsupported, 4 full-body leaning, 4 full-body seated, 4 waist-up unsupported, 4 waist-up leaning, 3 waist-up seated, 5 close-ups. The close-up tier doesn't split by support, because at head-and-shoulders you can't see what the body is resting on.

**Expect to reshoot at least once.** If the eval shows the poses themselves are weak, that costs an afternoon. Treat the set as consumable, not as an asset.

## 9. Known risks

**Metric scale.** The model reads "railing" well and "railing at 90cm versus 110cm" poorly. It has no depth sensor and no reliable size prior. Coarse height classes are a mitigation, not a fix. Expect this to be the dominant failure mode.

**Spatial coordinates.** Frontier VLMs are imprecise at normalised bounding boxes, and both the grounding check and silhouette placement depend on them. The two failure modes differ: coarse boxes still place a silhouette approximately right, which is acceptable for a selection aid, whereas *hallucinated* boxes are the real problem. The eval tests this by asking for a box, then asking in a separate call what's at those coordinates, and measuring disagreement. Only if that fails is a dedicated grounding model (Grounding DINO or similar) worth adding — it reintroduces a second model into a deliberately flat stack.

**Camera pitch.** Photographers lower the phone for full-body shots and raise it for waist-up. The IMU-driven squash handles this approximately, and approximately is the operative word.

**Taste.** Nothing in the model optimises for the photograph being good. Few-shot examples, the lighting constraint, and the eval are the only defence.

### The five failure classes

Named because they have different fixes and should be measured separately.

| Class | Description | Mitigation |
|---|---|---|
| **Ungrounded** | References something not in the frame | Grounding filter |
| **Infeasible** | Object exists, dimensions wrong (chest-height railing treated as hip-height) | Coarse height classes; partial |
| **Unexecutable** | Correct but unparseable by a human | Few-shot examples |
| **Badly lit** | Geometrically sound, subject backlit or in mottled shade | Lighting constraint in prompt |
| **Unflattering** | Everything right, photo still bad | Eval only |

## 10. Validation — the gate

**This is the product risk. Everything downstream is contingent on it.**

Nobody has demonstrated that a VLM gives good photo direction. Models describe scenes well, judge metric affordances poorly, and are entirely unmeasured on whether their suggestions produce attractive photographs.

**Method.** 100 scene photos, no people in them. Three cards generated per scene. Every card rated on grounded / feasible / clear / well-lit. Bbox agreement check run across all of them.

**Metric: at least one good card per scene — not per-card accuracy.** The three-card menu absorbs a bad card at display time, so per-card scoring reports a failure the product doesn't have. A 67% per-card rate with good spread is a working product; the same 67% concentrated so that a third of scenes produce three bad cards is not. **Measure the distribution, not the mean.**

**Then execute a sample.** Physically go to a set of the scenes, follow the generated directions, take the photographs. Blind-compare against photographs taken from the 30 hand-written directions, rated by people who weren't there.

**Two baselines to beat:**

1. Random pose from the correct support bucket
2. Best hand-curated pick

If it can't beat random-from-bucket, the scene understanding is contributing nothing — and that's been learned in a week for the cost of some API calls.

The eval is not a test to pass. It's the loop that produces the prompt.

## 11. Success metrics

**Gate (pre-build):** at least one good card on ≥80% of scenes, and generated directions beating random-from-bucket in blind comparison.

**Post-ship:**

- Sessions per week per active install — the retention question, and the one that kills apps of this shape
- Card-tap rate — proportion of scene locks where any card is selected
- All-three-swiped rate — consistent rejection means pose choice is more personal than contextual, and a better model won't fix that
- Shots per session and kept-photo rate
- Card position picked (1st / 2nd / 3rd) — informs whether ordering carries signal
- p50 and p95 lock-to-cards latency

## 12. Fine-tuning

**Not for v1.** The reasoning, and the conditions that would change it.

### Why prompting first

**There is no training data, and producing it is the actual problem.** Fine-tuning needs examples of the right answer. Here, "right" means a direction that produced a good photograph — knowable only by having someone execute it and judging the result. Thirty examples is not a fine-tune. A few thousand is, and hand-generating those is roughly a year. The eval loop is the bottleneck either way, and prompting uses it immediately rather than after building a data pipeline.

**The base model isn't missing the capability.** Fine-tuning wins when a behaviour is absent from the base model, or when it needs to be cheap and fast at scale, or when the task is narrow and repetitive. None applies. The model already knows what people do on steps; the constraint is judgment about *this* scene, which is what in-context reasoning is for.

**Iteration speed is decisive during validation.** The prompt will change a dozen times a day during §10. A fine-tune is hours per round and freezes understanding of the task at exactly the moment it's changing fastest.

**Fine-tuning forfeits model upgrades.** It pins to a base checkpoint. Spatial reasoning and metric estimation — the two known weak points — are improving quickly in frontier models. A prompt inherits those gains for free; a fine-tune has to be redone.

### The cheaper ladder, in order

1. **Iterate the prompt against the eval until it plateaus.** Most available gains are here, and it goes further than expected.
2. **Expand few-shot examples.** Six is what fits comfortably; twenty good ones is better. Context windows are large, and this is fine-tuning's cheap cousin.
3. **Retrieve examples rather than fixing them.** Select the six most similar to the current scene from a growing pool. This is what scales as the corpus grows, and it's far cheaper than training.

Only after all three plateau does training become the next lever.

### Preconditions for revisiting

Fine-tuning becomes the right call when **all** of these hold:

- **The product has shipped and is generating preference data at volume.** Which card was tapped, which photograph was kept, which was shared. Thousands of preference pairs produced by users rather than by hand. This is genuine signal about what makes a good direction and it cannot be manufactured pre-launch.
- **The prompt has stopped improving.** Two consecutive eval cycles with no meaningful movement.
- **There's a specific reason to train.** Cost or latency at scale (distilling a frontier model into something smaller and cheaper), or a persistent failure class that prompting demonstrably cannot fix — metric scale estimation is the most likely candidate.

### What the fine-tune would look like

**Most likely form: preference tuning on shipped data.** Pairs of (scene, chosen card) versus (scene, ignored card), weighted by whether the resulting photograph was kept. This trains the thing that actually matters — selection quality — rather than output format, which the schema already handles.

**Second candidate: a narrow spatial model.** If metric scale remains the dominant failure, a small model fine-tuned solely on support-height classification from photographs, sitting alongside the frontier VLM rather than replacing it. Narrow, cheap to label, and it targets the specific weakness rather than the whole task.

**Explicitly not worth training:** output format (the schema handles it), sentence register (few-shot handles it), object recognition (the base model is already good at it).

### Data collection to start now, cheaply

Even with no intent to train for a year, log the following from day one, since retrofitting it is impossible:

- Scene image hash, full JSON response, and which card was tapped
- Whether a photograph was taken after the tap
- Whether that photograph was kept or deleted
- Eval ratings from every §10 cycle, retained as a permanent labelled set

Opt-in, with clear disclosure, and store the derived record rather than the photograph wherever possible.

## 13. Build order

| # | Step | Duration | Purpose |
|---|---|---|---|
| 1 | Ten-photo smoke test | 20 min | Walk, shoot ten scenes, paste into a chat with the prompt, read the output. Reveals which failure class dominates before any code is written. |
| 2 | Shoot and trace the 30 | 2 afternoons | Log pitch and distance. Trace to outline PNGs. |
| 3 | **Eval (§10)** | 1 week | Python script over a folder of 100 photos, hitting the provider directly. No proxy, no phone, no app. The loop should take seconds, not a rebuild. **This is the gate.** |
| 4 | App | 2 weekends | Camera, stability trigger, proxy, one call, three cards, silhouette placement, shutter. |
| 5 | Instrument and observe | 2 weeks quiet | Ship to the original testers plus ten more. Then go quiet and read §11. |

**On framework:** optical flow runs continuously whenever the app is open, so this isn't a zero-per-frame-work app. It's light — downsampled pixel differencing suffices — but it's the one thing that must feel instant. Flutter is workable now that no per-frame ML is involved; native is smoother for a camera-first app with a custom overlay. Let team familiarity decide. "Viable" is not a reason to pick a framework.

## 14. Open questions

**Does a VLM actually give good photo direction?** Step 3 answers it. Steps 4 and 5 only matter if step 3 clears.

**Why does anyone open this instead of the native camera,** which is one swipe from the lock screen? Every answer available so far is a distribution answer rather than a product one, and the architecture doesn't help with it.

**Do photographers accept a suggestion or want to browse?** The all-three-swiped metric answers this after launch, and the answer determines whether the roadmap runs toward better recommendations or a better picker.

**Is the silhouette earning its place,** or is the sentence carrying everything? Worth an explicit A/B once there's traffic. If the sentence is doing all the work, the reference set can shrink to a prompt fixture and the rendering path disappears.
