# Pose Coach — Technical Design

Companion to the PRD. The PRD says what and why; this says how. Where the two disagree, the PRD wins on scope and this wins on mechanism.

---

## 0. Terminology

Every term in this document that isn't standard software engineering. Defined once here, used plainly everywhere else.

**Vision-language model.** A model that accepts an image and text together in one request and returns text. You send a JPEG plus a written instruction; it replies. It is not something we train or host — it is a third-party HTTP API, the way Stripe is. Written as "the model" throughout.

**Prompt.** The written instruction sent alongside the image. It is a plain-text file. Editing it changes behaviour with no code change and no deploy, which is why it lives in server config rather than the app binary (§3.4).

**Examples in the prompt.** Sample outputs pasted into the prompt so the model matches their style. Our six hand-written direction sentences are there for this reason. It is not training — it is showing the model what "good" looks like inside the request itself, and it costs nothing but a few hundred words per call. This is the mechanism that keeps sentences short and speakable, and it's why the reference set's sentences matter even though we no longer look poses up from a library.

**Schema-constrained response.** Every major provider lets you attach a JSON Schema to the request and guarantees the reply conforms to it. Not a suggestion in the prompt — enforced during generation. This is why §4.1's schema is load-bearing: `support_height` cannot come back as `"about a metre"` when the schema says it's an enum of five values. Format is solved; we never write a parser for prose.

**Temperature.** A number, roughly 0 to 1, controlling how much the model varies its output. Near 0 makes it near-deterministic; higher makes it explore. We use 0.7 because one request must return three *different* poses, and a near-deterministic model tends to return one idea worded three ways.

**Hallucination.** The model stating something false with the same confidence as something true — here, describing a railing that isn't in the photograph, or returning coordinates for an object it invented. It is the dominant correctness risk in this system, and §4.3's filter exists entirely to catch it before a card reaches the screen.

**Grounded / ungrounded.** A card is *grounded* if the object it names is actually visible in the photograph, and the coordinates it gives actually contain that object. *Ungrounded* means it isn't. This is the property §4.3 checks and §9.2 measures.

**Tokens.** The billing and size unit for these APIs. An image consumes tokens proportional to its pixel dimensions, so image size drives both cost and latency directly. This is the reason we upload at 768px rather than full resolution (§2.4).

**Object-detection model.** A separate, specialised model that takes a photograph and a word ("railing") and returns a box around that object. More accurate at coordinates than a general vision-language model, but it's a second dependency. Grounding DINO is one such model. §9.2 decides whether we need one; the answer is "only if the measurement says so."

**Fine-tuning.** Continuing to train a model on your own examples so it internalises them permanently, rather than being shown examples in each request. Requires thousands of labelled examples and pins you to one model version. Out of scope — see PRD §12 for the full reasoning and the conditions that would change it.

**Surface measurement.** Both iPhone and Android can measure flat surfaces in front of the camera, using frameworks already built into the operating system (ARKit and ARCore — the same ones behind furniture-preview apps). As the camera moves slightly, the phone compares what shifted between frames; nearby things shift more than distant ones, which is how it works out distance. When enough points sit on a common flat surface, it reports that surface with real measurements: flat-side-up or upright, size in metres, distance away, and height off the ground in centimetres.

A bench arrives as something like *flat-side-up, 44cm off the ground, 1.4m × 0.45m, 2.3m away.* Measured, not guessed. On the phone, continuously, at no cost.

Three limits, all of which matter here. It doesn't know what anything **is** — a flat surface 44cm up could be a step, a planter, a bin lid or a car bonnet. It only finds **flat** surfaces, so a railing never appears, being a thin pipe. And it needs visible detail to compare between frames, so a blank wall in flat light gives it nothing.

**The gate.** Weeks 2–3 of §13: the measurement that decides whether the model is good enough at this task to be worth building an app around. It is pass/fail. Roughly two-thirds of the total work sits behind it and is not started until it passes. Where the timeline says an artifact is **required to run the eval**, it means the measurement cannot be performed without it, so it must exist beforehand; **not required until the app** means it can wait, and should.

---

## 1. Design options

There are three jobs in this system, and every option below is a different answer to who does which.

| Job | What it means |
|---|---|
| **Perception** | What is physically here, how big is it, how high, where's the light |
| **Selection** | Given all that, which pose should the person strike |
| **Language** | Write the sentence the photographer says out loud |

The AI is unambiguously good at language, decent at selection, and weak at perception — it has no way to measure anything and works from a rough sense of how big things usually are. That weakness is the reason the options exist.

### The four options

| | Perception | Selection | Language | Calls while in use |
|---|---|---|---|---|
| **A** | AI estimates | AI | AI, written per scene | 1 |
| **B** | Phone measures, AI interprets | AI | AI, written per scene | 1 |
| **C** | Phone measures | Your rules | AI, written per scene | 1 |
| **D** | Phone measures | Your rules | Pre-written, one per pose | 0 |

---

#### Option A — AI does everything

Send one photograph, get back three poses with sentences. Nothing else runs.

**Pros**
- Simplest thing that could work. One call, no sensors, no rules to maintain.
- Behaves identically on every phone, including old and low-end ones.
- Nothing to build beyond what this document already specifies.
- Handles anything in a scene, including things you never anticipated — a fire escape, a stack of crates, a fallen tree.

**Cons**
- It is guessing at size, and this is the failure to expect most. A ledge that's too narrow to sit on, a railing at chest height treated as hip height.
- Also guessing at where things are in the frame, which is what places the silhouette. §9.2 exists to find out how often those coordinates are invented.
- Every recommendation needs signal. No signal, no cards.

---

#### Option B — Phone measures, AI does the rest

Identical to A, except the phone measures the flat surfaces it can see before the photo goes out, and one line of text goes along with it: *"measured surfaces: flat-side-up, 44cm high, at frame position (0.5, 0.7); upright, 2.1m tall, at (0.1–0.4)."*

The AI still decides everything. It has just stopped guessing at height.

**Pros**
- Directly attacks the failure most likely to dominate.
- Still one call. Measuring is free, on-device, already built into the phone.
- Real heights in centimetres replace the estimation trick in §6.1, so silhouette scaling gets more accurate too.
- Same code as A plus one line in the request — you can test both in the same week without committing.

**Cons**
- The phone reports geometry, not meaning. A 44cm flat surface could be a step or a bin lid, so the AI still has to work out what it's looking at.
- Railings, ledges and posts never get measured, because they aren't flat surfaces — and those are among the better things to pose against.
- Needs visible surface detail and reasonable light. A plain wall in flat light returns nothing.
- Two code paths for old phones without good support, or a raised minimum OS version.

---

#### Option C — Your rules pick the pose, AI writes the words

The phone measures. Your own code decides which pose, using rules you wrote:

```
flat-side-up, 40–55cm high, at least 40cm deep   → seated poses
upright, taller than a person, wide enough        → leaning poses
flat-side-up, 85–105cm high                       → perching poses
nothing found                                     → standing poses
```

The AI never chooses. It takes the pose your code picked and writes a sentence for this particular spot — *"sit on the second step, look toward the water."*

**Pros**
- When it picks something wrong you can read your own rules and see exactly why. Nothing to debug by re-reading a prompt and guessing.
- Selection is deterministic: same scene, same answer, every time.
- The AI is confined to writing sentences, which is the one job it's clearly good at.
- Cheaper per call, since the request is smaller and the reply is shorter.

**Cons**
- You have to write and tune the rules yourself, and they're rigid. Real places don't sort neatly into buckets.
- Inherits every blind spot of the measuring: no railings, nothing on plain walls, nothing in poor light.
- Loses the AI's ability to notice something you never thought of.
- More total code than A or B, and the rules need maintaining as you learn.

---

#### Option D — No AI while you're using it

Same as C, but the sentences were written once in Week 1, one per pose. The phone measures, the rules pick, the app shows a sentence you already wrote.

**Pros**
- Instant. No waiting at all, so the whole scene-lock and dead-time design becomes unnecessary.
- Free to run. No API bill that grows with users.
- Works with no signal — and good photo spots are frequently places with bad reception.
- Nothing leaves the phone, so the privacy story is one sentence long.
- By far the least code.

**Cons**
- The sentence can't mention anything about where you actually are. "Sit on the step" rather than "sit on the second step, look toward the water." Whether that difference matters is the open question below.
- Entirely limited to the 30 poses and the situations your rules cover.
- Still inherits every blind spot of the measuring.
- No path to getting better except shooting more poses and writing more rules.

---

### What this document specifies

**This document specifies Option A, and treats B as the expected upgrade.** The reason is sequencing rather than conviction: A is the only option testable in Week 2 with nothing but a prompt and a folder of photographs, and its results tell you whether any of the others are needed.

Sections that change by option:

| Section | A | B | C | D |
|---|---|---|---|---|
| §2 Client pipeline | as written | add surface measurement at lock | same as B | same as B |
| §4.2 Prompt | as written | add measured-surfaces line | selection instructions removed | not used |
| §4.3 Filter | seven rules | rules 1, 2, 6 become checks against measurement | mostly unnecessary | not used |
| §6.1 Scale | estimate height from box size | use the measured height | same as B | same as B |
| §3 Proxy | as written | as written | smaller request | **deleted entirely** |

### How to choose

Both decisions come out of the eval you are already running in Weeks 2–3. Neither needs extra infrastructure.

**Decision 1 — is perception the bottleneck?** Read the failure breakdown by class. If most bad cards are wrong about size or about where things are, A is not good enough on its own and B is the move. If most bad cards are about which pose was chosen or how the sentence reads, measuring won't help and the choice is between A and C.

**Decision 2 — do scene-specific sentences actually beat generic ones?** For the 20 spots in the final test, produce both versions: the AI's sentence for that spot, and the plain pre-written one for the same pose. Photograph both, judge both blind.

This is the decision that matters most, because **if generic sentences win or tie, Option D is available** — and D is a dramatically simpler product than anything else on this list. It is worth explicitly hoping for.

---

## 1b. System overview — Option A

Three components. No database, no user accounts, no persistent server state.

```
┌─────────────────────────────────────────────┐
│ Mobile client                               │
│                                             │
│  CameraSession ──► StabilityDetector        │
│       │                   │                 │
│       │                   ▼                 │
│       │            FrameSelector            │
│       │                   │                 │
│       ▼                   ▼                 │
│  IMUSampler ────► RecommendationController  │
│                           │                 │
│                    ┌──────┴──────┐          │
│                    ▼             ▼          │
│              SceneCache     ProxyClient ────┼──► HTTPS
│                    │             │          │
│                    └──────┬──────┘          │
│                           ▼                 │
│                   SilhouettePlacer          │
│                           │                 │
│                           ▼                 │
│                     CardStripView           │
└─────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────┐
│ Proxy (single serverless function)          │
│  validate → call model → check response     │
│  → filter bad cards → backfill → respond    │
└─────────────────────────────────────────────┘
                            │
                            ▼
                    Model provider API
```

Under Option B, one box is added on the client — a surface measurer running alongside the stability detector, whose output joins the request. Nothing else in the diagram changes.

Under Option D, everything below the client disappears.

**Asset bundle** ships with the app: 30 silhouette outlines, their metadata, and 3 fallback poses. Total under 500KB.

---

## 2. Client pipeline

### 2.1 Camera session

Rear camera only. Front-camera is out of scope — the whole CUJ assumes the photographer is looking at the back of the subject's environment.

Two streams off one session:

- **Analysis stream.** 320×240 or nearest supported, YUV, ~15fps. Feeds stability detection. Never leaves the device except as the single locked frame.
- **Capture stream.** Full sensor resolution, JPEG on shutter. Goes to the camera roll. **Never uploaded.**

The distinction matters for the privacy story: exactly one downsampled frame per scene lock goes to the network, and the photograph the user actually takes never does.

### 2.2 Stability detection

Optical flow in the PRD is loose language. The implementation is frame differencing, which is adequate and about fifty times cheaper.

```
on analysis frame f (15fps):
    g   = grayscale(f) downsampled to 64×48
    d   = mean(|g - g_prev|) / 255      # normalized [0,1]
    push d onto ring buffer (16 samples ≈ 1.07s)
    g_prev = g

    stable = all(d_i < MOTION_THRESHOLD for last 10 samples)   # ≈ 700ms
```

| Constant | Value | Notes |
|---|---|---|
| `MOTION_THRESHOLD` | 0.012 | Tune on real footage; handheld micro-jitter sits around 0.004–0.008 |
| `STABLE_SAMPLES` | 10 | 700ms at 15fps |
| `RELOCK_COOLDOWN` | 2000ms | Minimum between locks, regardless of motion |

**Exposure settling.** The first ~800ms after the camera opens or after a large scene change produces large frame deltas from auto-exposure, not from motion. Suppress locking until the exposure and white-balance states report converged, or failing that, for a fixed 800ms after session start.

**Hysteresis.** Once locked, do not re-lock until the scene *unlocks*, defined by `d > MOTION_THRESHOLD × 3` for 5 consecutive samples. Without this the detector oscillates at the threshold and refires constantly.

### 2.3 Frame selection

Stability means the camera stopped moving; it does not mean the frame is sharp or well exposed. Keep a ring buffer of the last 8 analysis-stream frames, and on lock choose the sharpest by variance of Laplacian:

```
sharpness(f) = var(convolve(grayscale(f), laplacian_3x3))
```

Request that frame's higher-resolution counterpart from the capture stream if the platform supports it; otherwise upscale is not needed because the target is only 768px.

### 2.4 Upload preparation

- Resize longest edge to **768px**, preserving aspect
- JPEG quality **80**
- Strip all EXIF, including GPS
- Expected payload 60–100KB

768px is enough to identify a railing and locate it in the frame, and much cheaper than full resolution in both latency and API cost, since these providers bill images by pixel dimensions. If the eval shows box precision is limited by resolution, revisit — that's a one-constant change.

### 2.5 IMU sampling

At lock, read device attitude and derive:

- `pitch_deg` — rotation about the device's horizontal axis. 0° = camera optical axis level with the horizon, negative = tilted down.
- `roll_deg` — for detecting a non-level horizon.

Sample as the mean over the stability window rather than a single instant, to reject hand tremor.

`roll_deg` is not sent to the model. It's used client-side: if `|roll| > 8°`, the placed silhouette is counter-rotated so it stays vertical relative to the world rather than the frame.

### 2.6 Scene cache

Difference hash (dHash), 64-bit, computed on the locked frame at 9×8 grayscale.

```
lookup(hash):
    for (h, cards, ts) in lru:                # capacity 12
        if popcount(hash XOR h) <= HASH_TOLERANCE and now - ts < TTL:
            return cards
    return None
```

| Constant | Value | Rationale |
|---|---|---|
| `HASH_TOLERANCE` | 6 bits | ~9% of the hash. Tolerant of exposure shifts and small reframing; tight enough that a different wall in the same building misses |
| `TTL` | 20 min | Light moves. Beyond this the lighting fields are stale |
| Capacity | 12 scenes | In-memory only, cleared on process death |

Cache is client-side only. The proxy retains nothing keyed on image content, which keeps the privacy story to one sentence.

### 2.7 Threading

| Work | Thread |
|---|---|
| Analysis frame callback, differencing, sharpness | Dedicated camera background thread |
| dHash | Same background thread |
| JPEG encode | Background |
| Network | Async, cancellable on unlock |
| Silhouette transform and draw | Main / render thread |

The analysis callback must complete well inside 66ms. At 64×48 the differencing is trivial; Laplacian variance is the expensive part, so compute it only on frames retained in the ring buffer, not every frame.

---

## 3. Proxy

Single stateless endpoint. Node or Python serverless function, no VPC, no database.

### 3.1 Contract

```
POST /v1/recommend
Content-Type: application/json

{
  "request_id":     "uuid",
  "image_b64":      "<jpeg>",
  "pitch_deg":      -8.4,
  "client_version": "1.0.0",
  "install_id":     "opaque-uuid"     // rate limiting only
}
```

```
200 OK
{
  "request_id":    "uuid",
  "cards":         [Card, Card, Card],   // always exactly 3
  "model_version": "provider-model-id",
  "prompt_version": "2026-09-07a",
  "latency_ms":    2840
}
```

**The response always contains exactly three cards.** If the model returns fewer, or cards are dropped by the filter, the proxy backfills from the bundled fallback set and marks them. The client never branches on partial results — it draws what it's given.

Errors return the same shape with all three cards from fallback and `"degraded": true`. The client shows cards regardless; a spinner that resolves into nothing is the worst outcome in the whole flow.

### 3.2 Card object

```json
{
  "source":          "model" | "fallback",
  "support_object":  "concrete steps",
  "support_bbox":    [0.31, 0.58, 0.79, 0.86],
  "support_height":  "knee",
  "subject_anchor":  [0.52, 0.71],
  "facing":          "three_quarter_left",
  "framing":         "full",
  "light_direction": "right",
  "light_quality":   "open_shade",
  "pose_id":         "seated_steps_elbows_knees",
  "direction":       "Sit on the second step, elbows on your knees, look toward the street"
}
```

`support_bbox` is `[x1, y1, x2, y2]` normalized to the frame, origin top-left.
`subject_anchor` is `[x, y]` normalized, the point where the body contacts the support — hips on a step, shoulder on a wall, feet on the ground for unsupported poses.

### 3.3 Rate limiting

Per `install_id`: 60 requests/hour, 400/day. Well above realistic use (a session locks maybe 10–20 scenes) and low enough to bound abuse if the endpoint is discovered. Return 429 with the fallback-card body rather than an error the client has to handle specially.

### 3.4 Provider call

- Attach the §4.1 JSON Schema to the request so the provider enforces the response shape during generation. Do not instead write "return JSON only" in the prompt and hope — that is the difference between reliable parsing and writing a fence-stripper.
- `temperature: 0.7` — the setting that controls output variability. One request has to produce three genuinely different poses; near-zero makes the model return one idea worded three ways.
- 8s hard timeout, one retry on 5xx or timeout, then fallback
- Model id and prompt version pinned in config, changeable without an app release

---

## 4. The model call

### 4.1 Response schema

```json
{
  "type": "object",
  "required": ["cards"],
  "properties": {
    "cards": {
      "type": "array", "minItems": 3, "maxItems": 3,
      "items": {
        "type": "object",
        "required": ["support_object","support_bbox","support_height",
                     "subject_anchor","facing","framing","light_direction",
                     "light_quality","pose_id","direction"],
        "properties": {
          "support_object":  {"type": "string", "maxLength": 40},
          "support_bbox":    {"type": "array", "minItems": 4, "maxItems": 4,
                              "items": {"type": "number", "minimum": 0, "maximum": 1}},
          "support_height":  {"enum": ["none","ground","knee","hip","chest"]},
          "subject_anchor":  {"type": "array", "minItems": 2, "maxItems": 2,
                              "items": {"type": "number", "minimum": 0, "maximum": 1}},
          "facing":          {"enum": ["toward","away",
                                       "three_quarter_left","three_quarter_right"]},
          "framing":         {"enum": ["full","waist","close"]},
          "light_direction": {"enum": ["front","back","left","right","above","diffuse"]},
          "light_quality":   {"enum": ["open_shade","direct_sun","backlit",
                                       "mottled","overcast","low_light"]},
          "pose_id":         {"enum": [ /* the 30 ids */ ]},
          "direction":       {"type": "string", "maxLength": 110}
        }
      }
    }
  }
}
```

`pose_id` as an enum is doing real work: it constrains the model to poses that have a traced silhouette, so placement can never fail for want of an asset. It also means the model is choosing from a vocabulary while writing a scene-specific sentence — the hybrid, not pure generation.

### 4.2 Prompt

```
You are directing a photograph. A person is about to be photographed in the
scene shown. The photographer will read your direction out loud to them.

Camera pitch: {pitch_deg}° from horizontal (negative = tilted down).

Propose THREE poses that could be struck in THIS scene, right now.

GROUNDING
Only reference objects clearly visible in the image. Do not invent a bench,
a railing, or a wall. If nothing is available to sit or lean on, use
support_height "none" and propose standing poses.

FEASIBILITY
Check the object's height against a standing adult before using it. A railing
at chest height cannot be perched on. A step 15cm deep cannot be sat on
comfortably. If unsure of a surface's height, prefer a pose that does not
depend on it.

LIGHT
Identify the dominant light source and favour poses where the subject faces
it or is lit from the side. Avoid poses that put the subject directly
backlit, or standing in mottled shade under foliage. Where it fits
naturally, let the gaze direction in your sentence carry this — "look
toward the water" is doing lighting work if that is where the light is.

VARIETY
The three must differ in kind, not in wording. Vary framing and support.
Do not return the same pose_id twice.

THE SENTENCE
Under 15 words. Second person. Body parts first, then gaze. Plain words a
friend would use out loud. No photography jargon, no adjectives about mood,
no explanation.

Examples of the register:
  "Sit on the step, elbows on your knees, lean forward a bit"
  "Back against the wall, put one foot flat on the wall behind you"
  "Hook both arms over the railing behind you, let your shoulders drop"
  "Turn your back mostly to me, then look at the camera over your shoulder"
  "Hands in your pockets, weight on one leg, let the other relax"
  "Elbow on your knee, chin resting on your hand"

ANCHOR
subject_anchor is where the body touches the support: hips for seated,
shoulder or back for leaning, feet for standing. Give it as a point in the
image where that contact should happen.
```

The prompt is versioned and lives in proxy config. During eval it changes daily; that is the point of the proxy existing.

### 4.3 Grounding filter

Runs server-side after schema validation. Each predicate drops the card.

| # | Predicate | Catches |
|---|---|---|
| 1 | `x2 > x1 + 0.02` and `y2 > y1 + 0.02` | Degenerate boxes |
| 2 | `0.003 < bbox_area < 0.85` | Whole-frame or pinpoint boxes. Both are signs the model invented the object rather than found it. |
| 3 | `subject_anchor` within bbox expanded by 0.15, unless `support_height == "none"` | Anchor unrelated to the object it names |
| 4 | `word_count(direction) <= 15` | Register drift |
| 5 | `pose_id` unique within the response | Diversity, enforced mechanically rather than trusted |
| 6 | `support_height == "chest"` implies `bbox` top edge above `y = 0.75` | Chest-height object located at the bottom of the frame |
| 7 | `pose_id`'s `support_class` compatible with `support_height` | Seated pose assigned to a wall |

Predicate 7 needs the compatibility table in the asset metadata (§5). It's the strongest of the seven, because it catches the model picking a plausible sentence and an incompatible silhouette.

Surviving cards are returned in model order; backfill appends fallbacks.

---

### 4.4 The three factors

A pose is only good if three things line up: something to interact with, a decent backdrop, and a body that can actually do it. The design so far only handles the first.

The three differ in a way that determines where each one lives:

| Factor | Where it comes from | How often it changes | Cost to add |
|---|---|---|---|
| **Support** | The photo | Every scene | Already built |
| **Background** | The photo | Every scene | Four schema fields, one prompt paragraph |
| **Subject** | Not in the photo at all | Once per session | A real design problem — see below |

#### Background

The system currently asks *what can you lean on* and never asks *what is behind you*. A bin, a parked car, a bright sign, a pole growing out of someone's head — these ruin photographs at least as reliably as an awkward pose does.

It costs almost nothing to add. Same photo, same call, four more fields:

| Field | Values |
|---|---|
| `background_quality` | clean / busy / cluttered |
| `background_problem` | Short phrase, or empty. "Bins behind the bench", "parked cars" |
| `background_feature` | Short phrase, or empty. Something worth *using*: an archway, a mural, a view |
| `subject_placement_note` | Optional. "Stand a metre left of the bench to clear the bins" |

`background_feature` is the half people forget. Avoiding clutter is defensive; noticing an arch and saying *"stand under the arch so it frames you"* is a different and better instruction. It's also a pose reason that has nothing to do with what you can sit on, which the current design cannot express at all.

**The conflict, and the policy.** Support says where you *can* sit. Background says where you *should* stand. They disagree constantly — the bench is perfect and there's a wheelie bin directly behind it.

Three possible resolutions:

| Policy | Effect |
|---|---|
| Support wins | Background breaks ties only. Simple, and produces photos with bins in them. |
| Background wins | Clean backdrop preferred even at the cost of a standing pose. Safe, and throws away good affordances. |
| **Surface it** | Return the rating per card and show it. The photographer decides. |

**Take the third.** The photographer is standing there and can see things the model can't — that the bin is being emptied right now, that the "clutter" is their friend's dog. Showing `background_quality` as a small marker on each card costs one glyph and hands the judgement to the person best placed to make it.

Add a rule to the prompt: when a spot has a clear background problem, at least one of the three cards must avoid it, even if that means a less interesting pose. Never return three cards that all put the subject in front of the same bins.

#### Subject

Harder, because of a tension the CUJ created deliberately.

**Recommendations fire before the subject walks over.** That's what makes the wait affordable — the dead time already exists. So at the moment cards are generated, the app has never seen the person it is directing.

That matters more than it sounds:

- A pose that assumes trousers doesn't work in a dress
- Crouching on wet ground doesn't work in light clothes
- Climbing doesn't work in heels
- Sitting on the floor doesn't work for someone with a bad knee
- Someone 1.55m and someone 1.85m do not fit the same railing

Four ways to handle it:

**1. Ignore it.** Keep all 30 poses clothing- and body-agnostic. Simplest, and it deletes roughly a third of the more interesting poses — every crouch, every floor-sit, every foot-up-on-the-wall.

**2. Ask at first run.** Three or four taps: what they're wearing, footwear, happy to sit on the ground, roughly how tall. Tag all 30 poses with what they require and filter. Cheap, deterministic, no photograph of anyone, no change to the privacy posture.

The problem is social, not technical. *"How tall is your friend, and is she wearing a skirt?"* is a strange thing to make someone answer while that friend stands three metres away waiting.

**3. Learn from rejection, within the session.** No setup at all. Default to permissive — every pose is offered. When the photographer swipes a card away, record what that pose required, and stop offering poses with that requirement for the rest of the session. Swipe away a crouch, get no more crouches.

Zero friction, nothing to ask, self-correcting, and it degrades gracefully — the cost of being wrong is one wasted card, which the three-card menu was already designed to absorb.

**4. One photo at session start.** Photographer takes one shot of the subject. The app derives a text description — *"trousers, flat shoes, about 1.7m, carrying a bag"* — and discards the image. Subject details are stable for a whole session, so this happens once and never disturbs the per-scene timing.

**Recommendation: 3 as the default, with 2 available as an explicit shortcut.** Rejection-learning fits the CUJ's insistence on the camera being the home screen with nothing before it, and it gets most of the benefit for none of the awkwardness. Option 2 becomes a small "she's in a dress" toggle for anyone who wants to skip the first wasted card. Option 4 is the upgrade if tags turn out too coarse, and it changes the privacy story, so it needs its own decision.

**Pose requirement tags.** Whichever route, the 30 poses need tagging during Week 1. This is a column in the metadata, not new infrastructure:

```
requires_trousers      crouch, wide stance, foot-up-on-wall, floor-sit
requires_clean_ground  floor-sit, kneel, lying
requires_flat_shoes    crouch, climb, walk-toward, stairs
requires_flexibility   deep crouch, floor-sit, kneel
requires_height_range  perch poses, which depend on the support's real height
```

#### Where subject and support interact numerically

Every other constraint is a filter. This one is arithmetic.

A 91cm railing is hip height for someone 1.8m tall and nearly waist-high for someone 1.55m. So "can they perch on this" is not a property of the railing — it's a property of the ratio.

```
support_ratio = support_height_m / subject_height_m

0.45 – 0.55   → seat height. Sitting works.
0.50 – 0.62   → perch height. Perching and hip-leaning work.
> 0.75        → too high to use as a support. Lean against, don't sit on.
```

This is the one place where knowing the subject's height genuinely improves an answer rather than just filtering an option, and it is also the strongest argument for the measured-surface options in §1. Under Option A the support height is a guess and the subject height is a guess, so the ratio is a guess squared. Under B, C or D the support height is measured in centimetres and only the subject's height is estimated.

#### What this changes elsewhere

| Section | Change |
|---|---|
| §4.1 Schema | Four background fields added |
| §4.2 Prompt | A background paragraph; the subject description passed in as text when available |
| §4.3 Filter | New rule: not all three cards may share the same `background_problem` |
| §5.1 Asset metadata | Five requirement tags per pose |
| §8 Telemetry | Log which requirement caused a swipe-away, so rejection-learning has something to learn from |
| §9 Eval | See below — this is the part that doesn't come free |

#### The eval gap

The 100 eval scenes are deliberately empty of people. That tests support and background perfectly well — both are properties of the scene.

**It tests subject fit not at all.** Nothing in the current harness would catch the model confidently telling a person in a dress to crouch on wet pavement.

The check needed is small and separate: take 20 of the 100 scenes, run each one three times with different subject descriptions passed in — *"trousers and trainers"*, *"dress and heels"*, *"trousers, avoids sitting on the ground"* — and answer two questions.

1. **Did the recommendations actually change?** If they're identical across all three, the subject description is being ignored and the whole mechanism is decorative.
2. **Did they change sensibly?** No crouching in the heels run; no floor-sitting in the third.

Sixty cards, about ten minutes of reading. Add it as a fourth step in the Weeks 2–3 loop, run once every couple of cycles rather than every cycle.

---

## 5. Silhouette assets

### 5.1 Format

30 outline PNGs with transparency, 512px tall, plus one metadata record each:

```json
{
  "pose_id":         "seated_steps_elbows_knees",
  "asset":           "seated_steps_elbows_knees.png",
  "anchor_norm":     [0.48, 0.62],
  "figure_height_norm": 0.71,
  "ref_pitch_deg":   -6.0,
  "ref_distance_m":  2.4,
  "framing":         "full",
  "facing_native":   "three_quarter_left",
  "support_class":   ["knee","ground"]
}
```

- `anchor_norm` — contact point in asset-local normalized coordinates. This is the point that gets placed at the card's `subject_anchor`.
- `figure_height_norm` — the figure's height as a fraction of the original reference frame's height. Needed to convert a target on-screen figure height into an asset scale factor.
- `ref_pitch_deg` — logged at shoot time. Without it there is nothing to correct *from*, which is why §8 of the PRD insists on logging it.

### 5.2 Authoring

Trace in a vector tool, export outline-only PNG at 512px. Mark the anchor by hand — one click per asset. Budget a day to a day and a half for all 30.

Outline rather than filled: a filled silhouette obscures exactly the part of the frame the photographer is composing.

---

## 6. Placement

The one piece of real geometry in the system.

### 6.1 Scale — primary path

**Under Options B, C and D this section is unnecessary** — the phone measured the surface, so the real height in centimetres is already known and goes straight into the arithmetic below in place of `H_REAL`. What follows is the Option A estimation, which exists only because the AI cannot measure.

When the card names a support object with a known height class, the bbox gives a metric reference at roughly the subject's depth.

```
H_REAL = {"knee": 0.50, "hip": 0.95, "chest": 1.40}   # metres, top of object

bbox_h_px      = (y2 - y1) * viewport_h
px_per_metre   = bbox_h_px / H_REAL[support_height]
target_fig_px  = ASSUMED_SUBJECT_H * px_per_metre     # ASSUMED_SUBJECT_H = 1.70
```

This is better than guessing from `framing` because it uses something measured in the actual frame rather than a constant.

**It only holds when the bbox's vertical extent is the object's height.** True for railings, walls, benches, low walls. False for steps (the bbox is the tread, seen from above, and its pixel height is a function of viewing angle, not of the step's rise) and for ground.

```
BBOX_HEIGHT_IS_METRIC = {"railing", "wall", "bench", "ledge", "low wall",
                         "planter", "fence", "post", "table"}
```

Match `support_object` against this set by keyword. On a miss, or when `support_height` is `none` or `ground`, fall through.

### 6.2 Scale — fallback path

```
FRAMING_FIGURE_FRACTION = {"full": 0.80, "waist": 1.35, "close": 3.20}
target_fig_px = FRAMING_FIGURE_FRACTION[framing] * viewport_h
```

Values above 1.0 are correct and intentional: a waist-up framing means the notional full figure is larger than the viewport and gets clipped by it.

### 6.3 Transform

```
scale   = target_fig_px / (asset_px_h * figure_height_norm)
sx, sy  = scale, scale

# pitch correction — heuristic
dpitch  = radians(live_pitch_deg - ref_pitch_deg)
sy     *= clamp(cos(dpitch), 0.78, 1.00)

# mirror
mirror  = needs_mirror(facing, facing_native)
if mirror: sx = -sx

# translate: asset anchor lands on subject_anchor
anchor_px = (anchor_norm.x * asset_px_w * sx, anchor_norm.y * asset_px_h * sy)
target_px = (subject_anchor.x * viewport_w, subject_anchor.y * viewport_h)
translate = target_px - anchor_px

# horizon
if abs(roll_deg) > 8: rotate by -roll_deg about target_px
```

**On the pitch correction.** As the camera pitches away from the reference angle, a standing figure's vertical extent in the projection compresses. The true relationship depends on subject distance and vertical position in frame; `cos(Δpitch)` is a first-order approximation that captures the direction and rough magnitude. The clamp at 0.78 bounds the damage when the approximation breaks at steep angles.

This is a heuristic and should be labelled one. Validate by eye during step 3 of the build order: place silhouettes onto scenes photographed at a range of known pitches and look at them. If it reads wrong, the next move is a small lookup table measured from the reference set rather than a more elaborate model.

### 6.4 Bounds

If the transformed bounding box extends past the viewport by more than 15% of its own dimension, clamp the translation to bring it inside and mark the card `needs_adjustment`. Do not drop it — the photographer can drag, and dragging is expected.

### 6.5 Manual override

Pan and pinch on the pinned silhouette, standard gesture recognisers. Overridden transforms are not persisted; each scene lock starts fresh.

---

## 7. Fallback set

Three poses from the 30, chosen for requiring no support and reading well anywhere:

| pose_id | direction |
|---|---|
| `standing_weight_hip_pockets` | Hands in your pockets, weight on one leg, let the other relax |
| `standing_half_turn_look_back` | Turn your back mostly to me, then look at the camera over your shoulder |
| `standing_walk_toward` | Just walk toward me normally, don't look at the camera |

Anchored at `[0.5, 0.92]` with `framing: "full"`, `support_height: "none"`. Used for timeouts, rate limits, all-cards-filtered, and no-support scenes.

---

## 8. Telemetry

Opt-in, disclosed at first run. Events are small and contain no imagery.

```json
{
  "event":        "scene_lock" | "card_tap" | "shutter" | "photo_kept"
                  | "strip_dismissed" | "override_drag",
  "session_id":   "uuid",
  "scene_hash":   "16-hex",           // dHash, not reversible to an image
  "request_id":   "uuid",
  "ts":           1757260800,
  "card_index":   1,
  "card_source":  "model",
  "pose_id":      "seated_steps_elbows_knees",
  "prompt_version": "2026-09-07a",
  "latency_ms":   2840,
  "degraded":     false
}
```

`photo_kept` fires on a delayed sweep of the camera roll for images taken in-session, checked once on next launch. This is the event that matters most and it cannot be captured at shutter time.

**This schema is the training dataset, if there is ever a training phase.** Per PRD §12, the record of which card was tapped, which were shown and ignored, whether a photo followed, and whether it survived cannot be reconstructed after the fact. Log it from the first build even though any training is at minimum a year out. Store the derived record; never the photograph.

---

## 9. Eval harness

Runs against the provider directly. No proxy, no phone, no app. The loop must be seconds.

```
eval/
  scenes/           100 jpgs, no people in frame
  prompts/          2026-09-07a.txt, 2026-09-08a.txt, ...
  runs/             {prompt_version}.jsonl
  ratings/          {prompt_version}.csv
  report.py
```

### 9.1 Generate

`run.py --prompt 2026-09-07a --scenes scenes/` fans out across 100 scenes with concurrency 8, writes one JSONL line per scene containing the full response, latency, and post-filter survivors. Roughly two minutes and a few dollars per full pass.

### 9.2 Bbox agreement check

Automated, no human. For each card, a second independent call:

```
"What object occupies the region [x1,y1,x2,y2] of this image?
 Answer with a short noun phrase, or 'nothing identifiable'."
```

Agreement is a fuzzy string match against `support_object`. The disagreement rate measures how often the model invents coordinates. The distinction it draws matters: a box that is roughly right but sloppy still agrees, because the object really is in there; a box for an object that was never in the photograph does not.

This decides whether a separate object-detection model is needed to supply coordinates. Only if disagreement is high does one enter the design — it adds a second model dependency to a deliberately simple stack, so it needs evidence first.

### 9.3 Human rating

Local static HTML: scene image, overlaid bbox, the sentence, four checkboxes.

| Flag | Question |
|---|---|
| `grounded` | Does the named object exist there? |
| `feasible` | Could a person actually do this, at that object's real size? |
| `clear` | Would a friend understand this said out loud? |
| `well_lit` | Would the subject be decently lit in this position? |

Four clicks per card, 300 cards, about 40 minutes per pass.

### 9.4 Metrics

```python
good = grounded and feasible and clear and well_lit

# PRIMARY
at_least_one_good_per_scene = mean(any(good(c) for c in scene) for scene in scenes)

# DISTRIBUTION — the point of the primary metric
histogram(sum(good(c) for c in scene) for scene in scenes)   # buckets 0,1,2,3

# OPTION DECISION — see §1
# If size and location errors dominate, Option A is not enough on its own
# and Option B is the move. If selection and wording dominate, measuring
# won't help.
perception_share = (n_failed_feasible + n_bbox_disagree) / n_failed_total

# DIAGNOSTIC ONLY
per_card_accuracy = mean(good(c) for all c)
failure_by_class  = counts of each of the five classes
bbox_disagreement = fraction failing §9.2
```

**Score on scenes, not cards.** The three-card menu absorbs a bad card at display time, so per-card accuracy reports a failure the product doesn't have. 67% per card spread evenly is a working product; the same 67% concentrated so a third of scenes yield zero good cards is not. The zero-bucket of the histogram is the number that matters.

**Gate:** `at_least_one_good_per_scene >= 0.80`, and zero-bucket under 10%.

### 9.5 Execution test

The four checkboxes can all be satisfied by directions that produce boring photographs. So the rubric is not the last word — someone has to go outside and take the pictures.

**Sample.** 20 scenes drawn from the 100.

**The two baselines, in order of importance.**

**Baseline 1 — random pose from the correct support bucket.** Look at the scene yourself, decide by eye whether it offers a knee-height, hip-height, chest-height or no support, then pick a pose at random from the 30 that fits that bucket. This is the cheap comparison and it is the one that can kill the project. If model-directed photographs don't beat it, the model isn't using the scene — it's picking plausible poses, which a dropdown does for free and offline.

Run this first. It costs nothing and it is the fastest way to learn the idea doesn't work.

**Baseline 2 — a human directing on the spot.** At each of the 20 locations, direct a person yourself and take the photograph, *before* looking at what the model produced for that scene. Then follow the model's direction and take a second photograph.**This on-the-spot step is required, and the reason is a trap worth naming.** The obvious approach is to compare against the 30 reference photographs from Week 1. That comparison is close to meaningless, because those were shot at the reference locations and the model's were shot at the eval locations. Any difference could be the locations rather than the directions. Only matched pairs at identical locations, same light, same day, same subject, isolate the variable being tested.

The Week 1 photographs are the wrong instrument here. They exist to supply the sentences, not to serve as a comparison set.

**Holdout.** Six of the 30 sentences are pasted into the prompt as style examples. **The remaining 24 are never shown to the model** and are held back as an untouched reference for how a human writes these. Choose which six early and don't rotate them — a sentence the model has seen is not evidence about a sentence it hasn't.

**Judging.** Strip context, pair the photographs, show them to 15 people who weren't present, ask which one they'd rather have of themselves. Do not judge your own.

**Result:** win over Baseline 1 is mandatory. Parity with Baseline 2 is the target; losing narrowly to a human directing in person is acceptable, because the human isn't in the product.

### 9.6 The sentence-specificity test — decides Option D

Run alongside §9.5, at the same 20 locations, for no extra travel.

For each location, take a **third** photograph directed by the **plain pre-written sentence** for that same pose — the generic one from the reference set, with nothing specific to the location in it. So at each spot you have the AI's *"sit on the second step, elbows on your knees, look toward the water"* and the generic *"sit on the step, elbows on your knees, lean forward a bit."*

Judge blind, same 15 raters, same question.

**Why this is the most consequential measurement in the project.** If the generic sentence wins or ties, the AI is contributing nothing on the live path, and **Option D becomes available** — no calls, no waiting, no API bill, works with no signal, and about a third of the code. That is a dramatically better product on every axis except one.

If the specific sentence wins clearly, the AI stays, and the only remaining question is where the measurements come from — Option A or B.

It is worth explicitly hoping the generic sentence wins.

---

## 10. Configuration

Everything tunable in one place, because most of these are guesses that eval will move.

| Constant | Value | Where |
|---|---|---|
| `MOTION_THRESHOLD` | 0.012 | Client |
| `STABLE_SAMPLES` | 10 | Client |
| `RELOCK_COOLDOWN_MS` | 2000 | Client |
| `UNLOCK_MULTIPLIER` | 3.0 | Client |
| `UPLOAD_LONG_EDGE` | 768 | Client |
| `JPEG_QUALITY` | 80 | Client |
| `HASH_TOLERANCE_BITS` | 6 | Client |
| `CACHE_TTL_MIN` | 20 | Client |
| `ROLL_CORRECTION_DEG` | 8 | Client |
| `PITCH_SQUASH_FLOOR` | 0.78 | Client |
| `ASSUMED_SUBJECT_H_M` | 1.70 | Client |
| `REQUEST_TIMEOUT_MS` | 8000 | Proxy |
| `TEMPERATURE` | 0.7 | Proxy |
| `RATE_LIMIT_HR` | 60 | Proxy |
| `prompt_version` | `2026-09-07a` | Proxy config |
| `model_id` | pinned | Proxy config |

Client constants ship in the binary for the first build. If tuning proves noisy, move them to a config fetched at launch with bundled defaults — worth doing before any wide release.

---

## 11. Privacy and security

- One 768px frame per scene lock leaves the device. Nothing else does.
- The photograph the user takes never leaves the device.
- EXIF stripped, including GPS, before upload.
- Proxy retains no image bytes and no image-derived cache. The model provider's own retention policy applies to what we send them; most offer a no-retention tier. Pick one explicitly and write down which.
- `install_id` is a random UUID for rate limiting, not an advertising or device identifier.
- Telemetry carries a dHash, which is not invertible to an image.
- API key server-side only.
- Reference-set subjects give written consent; the assets are traced outlines with no identifying features, but consent covers the originals.

---

## 12. Open technical questions

**Is 768px enough for usable box precision?** §9.2 answers it. If not, the first move is 1024px, which roughly doubles the per-image API cost.

**Does `cos(Δpitch)` hold well enough?** Eyeball validation in step 3. Replacement is a measured lookup table from the reference set, not a more elaborate projection model.

**Is `H_REAL` per height class too coarse?** Knee spans 0.4–0.6m in the wild. If placement scale reads wrong on eval scenes, the alternative is asking the model for a metre estimate directly — with the caveat from PRD §9 that metric estimation is the known weak point, so a bad estimate may be worse than a bucketed guess.

**Does the silhouette earn its place at all?** If the sentence carries everything, §5, §6, and the entire asset pipeline delete themselves and the app becomes three lines of text over a viewfinder. Worth an explicit A/B once there is traffic — and worth noticing that it would be a substantially simpler product.

---

## 13. Timeline

One engineer, evenings and weekends, roughly 15 hours a week. A full-time pair would compress the build weeks but not Weeks 2–3, because those are a measurement and take as long as they take.

Weeks 2–3 decide whether the rest happens. Everything before them exists to make that measurement possible; everything after them is ordinary app engineering that is only worth doing if the measurement passes.

---

### Week 0 — Twenty minutes of sanity checking

**~2 hours.**

Walk a few blocks. Photograph ten empty spots — a bench, some steps, a wall, a railing, open pavement.

Open a chat with the model, paste in a first draft of the §4.2 prompt, attach a photo, and read what comes back. Do this ten times.

You are looking for **which way it fails**, because that determines what the rest of the design has to defend against:

- It describes a bench that isn't in the picture → the §4.3 filter is the priority
- The bench is real but it tells someone to sit on a 20cm ledge → the feasibility paragraph in the prompt needs the work
- The sentences read like a photography textbook → the style examples are the lever

No code. No repo. Just a chat window and a phone.

**You now have:** a rough sense of the dominant failure, and a prompt worth testing properly.

---

### Week 1 — Building the test

**~13 hours.** Three things, and two of them happen on the same walk.

#### The test questions (~2h)

Photograph **100 empty scenes**. No people in any of them. Benches, steps, walls, railings, doorways, open paths. These are what you will hand the model 5–8 times over the next fortnight to see whether it can look at an empty space and work out what a person should do there.

Shoot them in the same kinds of places you shot the reference set. A hundred scenes with no benches or walls in them tests the wrong thing.

#### The rules and the baseline (~5h)

On the same walk, photograph **30 poses with a person in them**, and write the direction sentence for each one on the spot — *"Sit on the step, elbows on your knees, lean forward a bit."* Written afterwards from the photograph, they come out worse.

Log the camera pitch and rough distance for every frame. You will need it in Week 6 and reconstructing it later is miserable.

That gives you four things:

| | |
|---|---|
| **6 sentences** | Pasted into the prompt as examples of what a good answer sounds like. Pick which six now and never rotate them. |
| **24 sentences** | Held back. Never shown to the model. They are your untouched record of how a human writes these. A sentence the model has seen is not evidence about one it hasn't. |
| **30 `pose_id` strings** | The menu. The schema restricts the model to choosing from this exact list, so the list has to exist before the first request. |
| **30 photographs + pitch log** | Not needed until Week 4, when you trace them. |

#### The grading machine (~6h)

A Python script that fires all 100 scenes at the model in parallel and writes the responses to a JSONL file. Plus a plain local webpage that shows one card at a time — the scene photo, the box the model drew on it, the sentence — with four checkboxes underneath.

**Four checkboxes, not pass/fail**, because a single verdict tells you the score dropped without telling you which rule to rewrite:

| | |
|---|---|
| `grounded` | Does the object it named actually exist there? |
| `feasible` | Could a person really do that, given the object's real size? |
| `clear` | Would a friend understand this said out loud? |
| `well_lit` | Would the subject be decently lit standing there? |

#### Deliberately not done this week: tracing

Turning the 30 photographs into silhouette outlines is 12 hours, and **the eval never renders a silhouette** — a rater looks at a photo, a box and a sentence. The app doesn't need them until Week 6.

So tracing waits until Week 4. If Weeks 2–3 fail, that's 12 hours not spent drawing artwork for a product that isn't being built. And if they fail in the way that means reshooting the reference set, there's nothing to re-trace.

**The rule, for anything added to this plan later: if it isn't needed to run the eval, don't build it until the eval passes.**

**You now have:** 100 scenes, 6 example sentences, 24 held back, a list of 30 pose ids, and a script that grades them.

---

### Weeks 2–3 — Sitting the test

**~25 hours. This is where the project lives or dies.**

The same five steps, 5 to 8 times.

#### 1. Run it

`python run.py --prompt 2026-09-07a`. It sends all 100 scene photos to the model, each with your written rules and your 6 example sentences attached, 8 at a time. Two minutes, a few dollars.

#### 2. Read what came back

Exactly 3 suggestions per scene. **300 rows.** Each one names an object, gives coordinates for it, picks a pose id from your list of 30, and writes a sentence.

#### 3. The coordinate check runs itself

For every card, the script makes a *second, separate* call: *"What object is at these coordinates?"* If the answer doesn't match what the card claimed, the model made the coordinates up.

No human involved. This one number decides whether you eventually need a second specialised model just for locating objects — and until it says so, you don't.

#### 4. Grade it

Open the local webpage. 300 cards, four clicks each, about 40 minutes.

**The score is per scene, not per card.** This is the part most likely to be got wrong:

```
Wrong:  how many of the 300 cards were good?
Right:  how many of the 100 scenes produced at least one good card?
```

The app shows three and the photographer picks one, so a scene with one good card out of three is a scene the product handles fine. Card-level accuracy would report 33% on a scene that works.

The number that actually matters is **how many scenes produced zero good cards**, because that is the only case the user experiences as broken.

#### 5. Change one thing and go again

If it keeps proposing seats on chest-high fences, add a line to the feasibility paragraph. If sentences drift long and literary, tighten the register instruction. One change per cycle, so you know what moved the number.

Keep every version's runs and ratings. That accumulating set of graded cards is a real asset, not scratch work.

**Expect most of the improvement in cycles 1–3, and diminishing returns after.** If the score has stopped moving and is still under threshold, that is the answer, not a reason for a ninth cycle.

#### The final exam

Once the webpage score clears, go outside.

**20 locations from the 100.** At each one:

1. **Direct a person yourself first, before looking at the model's answer**, and take that photograph.
2. Then follow the model's direction and take a second photograph.
3. Then follow the **plain pre-written sentence** for that same pose — the generic one, with nothing about the location in it — and take a third.
4. Separately, pick a pose at random from the 30 that fits that spot's support type, and take a fourth.

Photo 3 is what decides whether you need an AI in the app at all (§9.6). If the generic sentence does as well as the AI's location-specific one, **Option D** in §1 becomes available: no calls, no waiting, no bill, works with no signal, a third of the code.

The first-before-looking part matters, and so does doing it on location. The obvious shortcut — comparing against the 30 reference photographs from Week 1 — is close to worthless, because those were taken somewhere else. Any difference could be the location rather than the direction. **Only matched pairs at the same spot, same light, same day, same person, isolate what you are testing.**

Then strip context, pair them up, and show them to 15 people who weren't there. Ask which one they'd rather have of themselves.

**Passing means all three of:**

| | |
|---|---|
| At least one good card on **≥80% of scenes** | |
| **Under 10% of scenes** producing zero good cards | |
| Model-directed photos **beat random-pose-from-the-right-bucket** | Mandatory. Losing here means the model isn't using the scene at all — it's picking plausible poses, which a dropdown does for free and offline. |

Parity with the on-the-spot human is the target. Losing narrowly to a person directing in the room is fine; that person isn't in the product.

**If it fails:** sound geometry but weak sentences → reshoot the reference set, swap the six examples, rerun. That's an afternoon. Sound sentences that still make dull photographs → the concept is weaker than hoped and no app work fixes it. **Do not start Week 4 on a near miss.**

**You also leave these two weeks having chosen an option from §1**, which determines what Weeks 4–7 actually build:

| What the results showed | Build |
|---|---|
| Generic sentences did as well as location-specific ones | **Option D.** Delete the server entirely. Weeks 4–5 become writing the rules table instead of a proxy, and the app gets simpler. |
| Location-specific sentences won, and size errors dominated the failures | **Option B.** As specced, plus surface measurement at scene lock and one extra line in the request. |
| Location-specific sentences won, and failures were about pose choice or wording | **Option A.** Build exactly what this document specifies. |

---

### Weeks 4–5 — The server, and finally the tracing

**~22 hours.** No mobile app yet.

#### The proxy (~10h)

One serverless function implementing §3.1. It takes an image and a pitch angle, attaches your API key, calls the model with the schema attached, runs the seven filter rules, backfills any dropped cards from the fallback set, and returns exactly three.

Unit-test the seven filter rules specifically. They are the only real logic in the whole server and each is a one-line predicate that's easy to get subtly wrong.

Testable end to end with `curl` and a folder of photos. Still no phone involved.

#### Tracing (~12h)

The work deferred from Week 1. Trace each of the 30 photographs into an outline PNG, click the anchor point on each — the spot where the body touches the support, hips for seated, shoulder for leaning, feet for standing — and write the §5.1 metadata, including the pitch you logged in Week 1.

The two tasks interleave well. Tracing is evening work for while the proxy is blocked on a response.

**You now have:** an endpoint that always returns three cards, and 30 placeable assets.

**"Always" is the contract, and Week 4 is where you verify it.** The endpoint has no response that leaves the client with nothing to draw. Send it each of these with `curl` and confirm all of them come back with three cards:

| Input | Why it happens in real use |
|---|---|
| A photo of the ground or the sky | Photographer hasn't raised the phone yet |
| A wall filling the entire frame | Standing too close |
| A near-black frame | Indoors, or dusk |
| A motion-blurred frame | Stability detector fired on a bad frame |
| A scene with nothing to sit or lean on | Not broken input — just an open field |
| A 2-byte file, or a truncated JPEG | Network hiccup mid-upload |
| A valid, well-composed scene | The normal case |

Separately, the *model's* reply can be unusable even when the photo is fine — it names a bench that isn't there, returns coordinates outside the frame, picks the same pose three times, or writes a 40-word sentence. The seven filter rules in §4.3 drop those cards.

In every one of these cases the proxy backfills from the three bundled fallback poses and still returns three. That is why the client has no empty state and no error branch for missing content: it draws what it is given.

---

### Weeks 6–7 — The app

**~30 hours.** Now you write the mobile app.

| | |
|---|---|
| Camera session, both streams | 4h |
| Stability detector, ring buffer, sharpness | 6h — tuning `MOTION_THRESHOLD` against real handheld footage takes longer than writing the code |
| IMU sampling, exposure-settling suppression | 3h |
| Scene cache | 2h |
| Network client, timeout, retry, fallback | 3h |
| **Silhouette placement (§6)** | **8h** — both scale paths, pitch correction, mirroring, roll, bounds clamp |
| Card strip, tap-to-pin, drag and pinch | 6h |
| Shutter and camera roll | 2h |
| Telemetry | 3h |

**Placement is the item most likely to overrun**, because it's the only part you cannot verify by reading the code. It has to be looked at, on real scenes, at varied camera angles. Budget an afternoon purely for standing outside looking at silhouettes and deciding whether they sit right.

---

### Week 8 — Field tuning

**~10 hours.**

Walk around with it. Every constant in §10 is currently a guess, and this is where guesses become measurements: motion threshold against real hands, cache tolerance against real re-pointing, pitch correction against real placement.

Also the first honest latency number — lock to cards, median and 95th percentile, on cellular rather than office wifi.

---

### Weeks 9–11 — Ship, then stop touching it

**~5 hours, then wait.**

Ship to the five original pairs plus ten more. Then go quiet for two weeks and read the PRD §11 metrics.

Sessions per week is the number that matters and it needs elapsed time, not effort. **Shipping fixes during the observation window makes the retention read uninterpretable**, which is the hardest discipline item on this list.

---

### Summary

| Week | What | Hours | |
|---|---|---|---|
| 0 | Ten photos into a chat window | 2 | Before the eval |
| 1 | 100 scenes, 30 poses, grading script | 13 | Before the eval |
| **2–3** | **Run it, grade it, fix the prompt, ×6** | **25** | **Decides everything after** |
| 4–5 | Server proxy, trace the silhouettes | 22 | Only if it passed |
| 6–7 | Mobile app | 30 | Only if it passed |
| 8 | Field tuning | 10 | Only if it passed |
| 9–11 | Ship and observe | 5 + wait | Only if it passed |

**About 107 hours over eight working weeks, plus two weeks of watching.**

**40 hours happen before the eval; 67 happen only if it passes.** Moving the tracing to Week 4 is what shifted 12 of those hours from the first column to the second, at no cost, because the assets weren't needed until Week 6 anyway.

None of the first 40 hours involve writing product code. Weeks 4 through 8 are ordinary engineering with no open questions in them. The uncertainty is front-loaded deliberately.
