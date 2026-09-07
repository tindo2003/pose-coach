# Pose Coach — Technical Design

Companion to the PRD. The PRD says what and why; this says how. Where the two disagree, the PRD wins on scope and this wins on mechanism.

---

## 1. System overview

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
│  validate → call VLM → schema check         │
│  → grounding filter → backfill → respond    │
└─────────────────────────────────────────────┘
                            │
                            ▼
                  Frontier VLM provider
```

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

768px is sufficient to identify a railing and localise it, and materially cheaper than full resolution in both latency and tokens. If eval shows bbox precision is resolution-bound, revisit — that's a one-constant change.

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

- Structured output via the provider's schema-constrained mechanism, not a "return JSON only" instruction
- `temperature: 0.7` — three cards from one call need genuine spread; near-zero produces three rewordings of one idea
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
| 2 | `0.003 < bbox_area < 0.85` | Whole-frame or pinpoint boxes, both hallucination tells |
| 3 | `subject_anchor` within bbox expanded by 0.15, unless `support_height == "none"` | Anchor unrelated to the object it names |
| 4 | `word_count(direction) <= 15` | Register drift |
| 5 | `pose_id` unique within the response | Diversity, enforced mechanically rather than trusted |
| 6 | `support_height == "chest"` implies `bbox` top edge above `y = 0.75` | Chest-height object located at the bottom of the frame |
| 7 | `pose_id`'s `support_class` compatible with `support_height` | Seated pose assigned to a wall |

Predicate 7 needs the compatibility table in the asset metadata (§5). It's the strongest of the seven, because it catches the model picking a plausible sentence and an incompatible silhouette.

Surviving cards are returned in model order; backfill appends fallbacks.

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

**This schema is the fine-tuning dataset.** Per PRD §12, the preference pairs — which card was tapped, which were shown and ignored, whether a photo followed, whether it survived — cannot be retrofitted. Log from the first build even though training is at minimum a year out. Store the derived record; never the photograph.

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

Agreement is a fuzzy string match against `support_object`. Disagreement rate is the hallucinated-coordinates metric. Coarse-but-correct boxes still agree; confabulated ones don't.

This is what decides whether a dedicated grounding model is needed. Only if disagreement is high does Grounding DINO enter the design — it reintroduces a second model into a deliberately flat stack, so it needs evidence.

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

# DIAGNOSTIC ONLY
per_card_accuracy = mean(good(c) for all c)
failure_by_class  = counts of each of the five classes
bbox_disagreement = fraction failing §9.2
```

**Score on scenes, not cards.** The three-card menu absorbs a bad card at display time, so per-card accuracy reports a failure the product doesn't have. 67% per card spread evenly is a working product; the same 67% concentrated so a third of scenes yield zero good cards is not. The zero-bucket of the histogram is the number that matters.

**Gate:** `at_least_one_good_per_scene >= 0.80`, and zero-bucket under 10%.

### 9.5 Execution test

The rubric can be satisfied by directions that produce boring photographs. So: take a 20-scene sample, physically go, follow the directions, photograph the results. Blind-compare against photographs from the 30 hand-written directions, rated by people who weren't present.

Baselines to beat: random pose from the correct support bucket, and best hand-curated pick. Failing to beat random-from-bucket means the scene understanding contributes nothing — learned in a week for the price of some API calls.

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
- Proxy retains no image bytes and no image-derived cache. Provider retention is whatever the provider's zero-retention or standard policy specifies; select and document it.
- `install_id` is a random UUID for rate limiting, not an advertising or device identifier.
- Telemetry carries a dHash, which is not invertible to an image.
- API key server-side only.
- Reference-set subjects give written consent; the assets are traced outlines with no identifying features, but consent covers the originals.

---

## 12. Open technical questions

**Is 768px enough for usable bbox precision?** §9.2 answers it. If not, the first move is 1024px, which roughly doubles image tokens.

**Does `cos(Δpitch)` hold well enough?** Eyeball validation in step 3. Replacement is a measured lookup table from the reference set, not a more elaborate projection model.

**Is `H_REAL` per height class too coarse?** Knee spans 0.4–0.6m in the wild. If placement scale reads wrong on eval scenes, the alternative is asking the model for a metre estimate directly — with the caveat from PRD §9 that metric estimation is the known weak point, so a bad estimate may be worse than a bucketed guess.

**Does the silhouette earn its place at all?** If the sentence carries everything, §5, §6, and the entire asset pipeline delete themselves and the app becomes three lines of text over a viewfinder. Worth an explicit A/B once there is traffic — and worth noticing that it would be a substantially simpler product.

---

## 13. Timeline

Assumes one engineer working evenings and weekends, roughly 15 hours a week. A full-time pair would compress this to about three weeks but would not change the ordering, because the gate is a measurement, not a build.

### Week 0 — Smoke test

**~2 hours, one evening.**

Walk a few blocks, photograph ten scenes, paste them into a chat with a first draft of the §4.2 prompt. Read the output.

No code. The purpose is to find out which of the five failure classes dominates before committing to a design that assumes a particular one. If ungrounded references dominate, the grounding filter matters most. If infeasibility dominates, the prompt's feasibility paragraph needs the most work. If the sentences read like a photography manual, few-shot is the lever.

**Exit:** a rough sense of the dominant failure mode, and a prompt worth running at scale.

---

### Week 1 — Reference set and harness, in parallel

**Track A — shoot and trace, ~10 hours.**

| | |
|---|---|
| Shoot the 30 | One afternoon. Six locations, log pitch and distance per frame (§8 of the PRD). A phone with a level readout, or a second phone recording the attitude, is enough. |
| Write the 30 sentences | Same afternoon, on the spot. Writing them later from photographs produces worse sentences. |
| Trace to outline PNGs | One to one and a half days. Mark `anchor_norm` by hand, one click each. |
| Metadata records | Two hours. §5.1 for all 30, plus the `support_class` compatibility table the filter's predicate 7 depends on. |

**Track B — eval harness, ~6 hours.**

Directory scaffold, `run.py` with concurrency 8, the §9.2 agreement check, the static-HTML rating page, `report.py` computing §9.4. All against the provider directly.

**Track C — collect eval scenes, ~2 hours.**

100 scene photographs, no people. Do this during the same walks as Track A.

**Exit:** 30 traced assets with metadata; a harness that runs 100 scenes in two minutes.

---

### Weeks 2–3 — Eval, the gate

**~25 hours across two weeks. This is where the project lives or dies.**

The loop is: run 100 scenes, rate 300 cards (~40 min), read the failure-class breakdown, change one thing in the prompt, rerun. Expect five to eight cycles.

| | |
|---|---|
| Cycles 1–3 | Prompt iteration against the rubric. Most movement happens here. Every prompt version keeps its runs and ratings — this is a permanent labelled dataset, not scratch work. |
| Bbox agreement | Runs automatically each cycle. Answers whether a dedicated grounding model is needed. |
| Cycles 4–6 | Diminishing returns. If `at_least_one_good_per_scene` has plateaued below gate, that is the answer. |
| Execution test | Half a day: 20 sampled scenes, physically executed and photographed. |
| Blind comparison | Two days elapsed, an hour of work — 15 raters, assisted versus hand-written baseline. |

**Gate:** `at_least_one_good_per_scene >= 0.80`, zero-bucket under 10%, and a win over random-from-bucket in blind comparison.

**If the gate fails**, the branch depends on which metric missed. Weak sentences with sound geometry means reshoot the reference set and redo the few-shot examples — an afternoon, then rerun. Sound sentences that produce dull photographs means the concept is weaker than hoped, and no amount of app work fixes it. Do not proceed to Week 4 on a near miss.

---

### Week 4 — Proxy and prompt in production shape

**~10 hours.**

Serverless function, the §3.1 contract, schema-constrained call, the seven filter predicates, backfill logic, rate limiting, config-driven prompt and model version. Unit tests on the filter predicates specifically — they are the only real logic in the proxy and each one is a one-line predicate that is easy to get subtly wrong.

Testable end to end with curl and a folder of photographs. No app required.

**Exit:** an endpoint that returns three valid cards for any photograph, including garbage input.

---

### Weeks 5–6 — Client

**~30 hours.**

| | |
|---|---|
| Camera session, dual stream | 4h |
| Stability detector, ring buffer, sharpness | 6h. Tuning `MOTION_THRESHOLD` against real handheld footage takes longer than writing it. |
| IMU sampling, exposure-settling suppression | 3h |
| dHash cache | 2h |
| Proxy client, timeout, retry, fallback | 3h |
| Silhouette placement (§6) | 8h. The single largest client item. Both scale paths, pitch correction, mirroring, roll, bounds clamp. |
| Card strip, tap-to-pin, pan and pinch | 6h |
| Shutter, camera roll | 2h |
| Telemetry (§8) | 3h |

Placement is the piece most likely to overrun, because it is the only part that cannot be verified by reading the code — it has to be looked at, on real scenes, at varied pitch. Budget an afternoon purely for eyeballing it.

**Exit:** installable build, end to end.

---

### Week 7 — Field tuning

**~10 hours.**

Walk with it. The constants in §10 are guesses and this is where they become measurements: motion threshold against real hands, hash tolerance against real re-pointing, pitch squash against real placement.

Also the first honest read on latency — lock to cards, p50 and p95, on cellular rather than office wifi.

---

### Weeks 8–10 — Ship and go quiet

**~5 hours, then wait.**

Ship to the original testers plus ten more. Then stop shipping and read the PRD §11 metrics for two weeks. Sessions per week is the number that matters, and it needs elapsed time rather than effort.

Resisting the urge to ship fixes during this window is the point. Changes mid-observation make the retention read uninterpretable.

---

### Summary

| Week | Phase | Effort | Blocks |
|---|---|---|---|
| 0 | Smoke test | 2h | Everything |
| 1 | Reference set + harness | 18h | Eval |
| 2–3 | **Eval — the gate** | 25h | All build work |
| 4 | Proxy | 10h | Client |
| 5–6 | Client | 30h | Field tuning |
| 7 | Field tuning | 10h | Ship |
| 8–10 | Ship and observe | 5h + wait | Roadmap |

**Total to shipped build: about 100 hours over seven weeks, with a hard gate at week 3 that gates roughly 55 of them.**

The shape worth noticing: a quarter of the effort comes before any product code, and it is the quarter that determines whether the rest is worth doing. Weeks 4 through 7 are ordinary app engineering with no research risk in them. All the uncertainty is front-loaded on purpose.
