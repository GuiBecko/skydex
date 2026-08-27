# AI Photo Validation — Design

Date: 2026-08-16
Status: approved, ready for implementation planning

## Problem

Today the user picks the `Phenomenon` when registering a capture, and
`CaptureValidationService` checks that claim against the Open-Meteo hourly record. The photo itself
is never looked at. Nothing stops a user pointing the camera at a wall, or at a screenshot of a
storm, as long as a storm happens to be in the forecast for their coordinates.

This design removes the user's choice and puts two automated judgements in its place: Open-Meteo
decides *what* the phenomenon is, and a vision model decides whether the photo is consistent with
it.

## What changes, in one flow

```
1. App takes photo + GPS fix
2. App uploads the photo in the background        POST /api/photos
   └─ server runs the vision model right here     POST /v1/analyze
      └─ not a sky?  422, photo never stored
3. User types title + description
4. App creates the capture, WITHOUT a phenomenon  POST /api/events
5. Server asks Open-Meteo for the WMO code        → Phenomenon (the truth)
   └─ Open-Meteo unavailable? 503, photo not spent, retry works
6. Server compares the cached vision scores against that Phenomenon
   ├─ consistent    → CONFIRMED, XP by rarity
   └─ contradiction → UNCONFIRMED, 0 XP, capture kept with a reason
7. App reveals the phenomenon in CaptureRewardOverlay
```

## Decisions and why

### Inference runs on the server, never on the device

The model's purpose is anti-fraud. A modified client would simply report whatever verdict it
liked — the same weakness `CaptureValidationService` already documents at length for
`locationIsMock`. There is no version of this feature that works with on-device inference.

### The vision service reports evidence; Kotlin decides the verdict

`skydex-vision` returns scores. It does not know what a `Phenomenon` is, what a threshold is, or
what CONFIRMED means. All policy lives in Kotlin, next to the validation semantics that already
live there.

This keeps threshold and matrix changes out of the ML service entirely, and keeps the ML service
testable as a pure function of an image.

### The model runs at upload time, not at capture time

Three reasons, all of them load-bearing:

- **Latency disappears.** The user is typing title and description while the model runs.
- **The result is cached** on `uploaded_photos`, so the model never runs twice for one photo.
- **A vision-service outage costs nothing.** The 503 lands before any photo row exists, before any
  photo is spent, before any Open-Meteo call is paid for.

### Open-Meteo becomes load-bearing, and its outage is now a 503

Previously an Open-Meteo failure downgraded a capture to UNCONFIRMED; the phenomenon still existed
because the user had chosen it. Now Open-Meteo *is* the phenomenon, and `WeatherEvent.phenomenon`
is `NOT NULL`. Without an answer there is no capture to write.

`POST /api/events` therefore answers **503** when Open-Meteo does not respond or the capture falls
outside the 90-minute window. This is cheap because `photoProvenance.consume` runs inside
`CaptureCommitService.commit`, which is reached only *after* the Open-Meteo call: aborting earlier
leaves the photo unspent and the retry valid for the remainder of its 30-minute `MAX_AGE`.

## Architecture

```
┌─────────────┐
│ App Android │
└──────┬──────┘
       │ POST /api/photos (multipart)
       ▼
┌────────────────────┐   POST /v1/analyze      ┌──────────────────┐
│  SkyDex-backend    │────────────────────────▶│  skydex-vision   │
│  (Spring, :3002)   │◀────────────────────────│  FastAPI + CLIP  │
└──────┬─────────────┘   scores                │  Python 3.11 CPU │
       │                                       │  :8000           │
       │ POST /api/events (no phenomenon)      └──────────────────┘
       ▼
   Open-Meteo ──▶ WMO code ──▶ Phenomenon ──▶ verdict from cached scores
```

### New components

| Where | What |
|---|---|
| `skydex-vision/` (new repo) | FastAPI, `POST /v1/analyze`, CLIP ViT-B/32 |
| `docker-compose.yml` | ties `skydex-db` and `skydex-vision` together |
| `services/VisionClient.kt` | RestClient with explicit timeouts, modelled on `OpenMeteoClient` |
| `services/PhotoAuthenticityService.kt` | applies the policy to the scores |
| `models/UploadedPhoto.kt` | + `vision_outdoor_score`, `vision_scores`, `vision_model`, `vision_analyzed_at` |
| `models/WeatherEvent.kt` | + `unconfirmed_reason` |

All new columns are nullable, so `ddl-auto=update` handles them with no manual migration. Existing
captures are untouched and keep the phenomenon their author chose.

### The vision service contract

`POST /v1/analyze`, multipart image in, JSON out:

```json
{
  "outdoor_score": 0.94,
  "phenomenon_scores": {
    "CLEAR": 0.02, "CLOUDY": 0.11, "FOG": 0.04,
    "RAIN": 0.62, "SNOW": 0.01, "STORM": 0.20
  },
  "model": "clip-vit-b32-zeroshot-v1"
}
```

`phenomenon_scores` sums to 1 across the six **visual groups**, not the nine `Phenomenon` values.
See below.

`VisionClient` returns null on any failure — timeout, malformed body, 5xx — exactly as
`OpenMeteoClient` does. `PhotoController` turns a null into a 503.

## The decision policy

Two stages, at different moments, with different severity.

### Stage 1 — "is this the sky?" (upload, hard gate)

CLIP scores the image against two prompt sets:

```
SKY:     "a photo of the sky", "an outdoor photo of clouds",
         "a photograph taken outdoors looking up at the sky"
NOT SKY: "a screenshot of a phone screen", "an indoor photo of a room",
         "a selfie of a person", "a photo of a wall",
         "a photo of a printed photograph", "a close-up of an object"
```

`outdoor_score` is the probability mass on the SKY set. Below **0.60**, `POST /api/photos` answers
**422** and stores nothing.

Rejecting at upload rather than at capture is deliberate: the user finds out immediately, before
writing anything, and junk never reaches the database. The accepted cost is giving an attacker an
oracle to iterate against — acceptable because the threshold is conservative and the attacker could
iterate regardless.

### Stage 2 — "does it match the weather?" (capture, soft gate)

#### Visual groups

The nine `Phenomenon` values collapse into six classes a photograph can actually distinguish:

| Visual group | Phenomenon |
|---|---|
| `CLEAR` | CLEAR_SKY |
| `CLOUDY` | CLOUDS |
| `FOG` | FOG |
| `RAIN` | DRIZZLE, RAIN, RAIN_SHOWER |
| `SNOW` | SNOW |
| `STORM` | THUNDERSTORM, HAILSTORM |

The model is never asked to tell drizzle from rain. They are the same photograph.

#### Contradiction matrix

Rows are the group Open-Meteo reported; columns are the group the photo scored highest.

```
expected \ photo   CLEAR  CLOUDY  FOG   RAIN  SNOW  STORM
CLEAR               ok    BLOCK  BLOCK BLOCK BLOCK BLOCK
CLOUDY             BLOCK   ok     ok    ok   BLOCK  ok
FOG                BLOCK  BLOCK   ok   BLOCK BLOCK BLOCK
RAIN               BLOCK   ok     ok    ok   BLOCK  ok
SNOW               BLOCK  BLOCK  BLOCK BLOCK  ok   BLOCK
STORM              BLOCK   ok     ok    ok   BLOCK  ok
```

The rule behind it: **when the weather is a distinctive phenomenon (CLEAR, FOG, SNOW), the photo
must show it. When the weather is ordinary (CLOUDY, RAIN, STORM), the photo may look like anything
in that neighbourhood.**

Row by row:

- **CLEAR** — only CLEAR. If the sky is clear, the photo shows a clear sky.
- **CLOUDY** — CLEAR blocks. FOG, RAIN and STORM pass: dense low cloud reads as fog, and a heavy
  overcast reads as rain or storm. SNOW blocks.
- **FOG** — only FOG. Fog is unmistakable and is the class CLIP is most reliable on.
- **RAIN** — CLOUDY passes (raindrops are near-invisible in a photograph; what shows is grey sky),
  FOG passes (heavy rain kills visibility), STORM passes. CLEAR and SNOW block.
- **SNOW** — only SNOW. Snow is EPIC rarity, so it is the highest-value fraud target, and snow on
  the ground is among the easiest things for CLIP to identify.
- **STORM** — the same neighbourhood as RAIN. Lightning is rarely in frame; a storm photographs as
  a dark sky or as rain.

#### Only on a confident contradiction

A BLOCK cell blocks only when all three hold:

```
score(expected group) < 0.10
score(top-1 group)    > 0.70
the (expected, top-1) pair is a BLOCK cell
```

#### Night skips stage 2 entirely

At night no human can tell an overcast sky from a clear one either. `is_day` is added to the
existing `hourly=` request in `OpenMeteoClient` — it costs nothing, the call is already being made.

### Outcomes

| Situation | Result |
|---|---|
| Passes both stages | `CONFIRMED`, XP by rarity |
| Fails stage 2 | `UNCONFIRMED`, 0 XP, capture stored with a reason |
| Fails stage 1 | Upload refused, 422, no capture, no photo row |
| Open-Meteo unavailable | 503 on create, photo unspent, retry works |
| Vision service unavailable | 503 on upload, nothing written at all |

Failing stage 2 does not refuse the capture, because the model may be the one that is wrong.
Failing stage 1 does refuse it, because "this is not even the sky" is an objective claim.

### `WeatherEvent.unconfirmedReason`

`PHOTO_CONTRADICTS_WEATHER | IMPLAUSIBLE_TRAVEL | MOCK_LOCATION | null`

Today the app cannot tell the user *why* a capture was not confirmed. With a model in the loop that
stops being acceptable.

Three values, not four, and the missing one is worth naming. Today's most common UNCONFIRMED cause
is "the phenomenon you claimed is not the one Open-Meteo recorded" — and after phase 3 that cause
cannot occur, because there is no user claim left to disagree with. An upstream failure or an
out-of-window capture, which currently land in the same bucket, become a 503 instead. What survives
is exactly these three.

Rows written before phase 3 keep `unconfirmedReason = null`, which reads correctly as "not recorded"
rather than as a wrong reason.

### Photos with no analysis

Between phases, and only there, an `uploaded_photos` row can exist with null vision columns. The
capture path treats null as "not analysed" and skips stage 2 rather than blocking — a photo cannot
be punished for a check that did not run when it was uploaded.

This window is bounded by `PhotoProvenanceService.MAX_AGE`: 30 minutes after phase 2 ships, no
unanalysed photo is citable by any capture.

## Model, dataset and training

### The model

**CLIP ViT-B/32** via `open_clip`. Open weights, MIT licence, no API key, no per-request cost.
350MB, ~250ms per photo on one CPU core. Two output heads, one per stage.

Full fine-tuning of CLIP is out of scope: the development machine has no NVIDIA GPU.

### Three phases

**Phase 1 — zero-shot with prompts.** No training. Write prompts, measure on a public dataset,
adjust, measure again. Prompt wording matters a great deal in CLIP — `"rain"` and `"a photo of a
rainy overcast sky, raindrops, wet"` are not close. Target: visual-group accuracy above 70%,
outdoor accuracy above 95%.

**Phase 2 — linear probe.** Freeze CLIP, extract a 512-dimensional vector per photo, train a
logistic regression on top. Trains in seconds on CPU with roughly 300 photos per group, and
typically gains 10-15 accuracy points over zero-shot. The artefact is a ~30KB `.npz` the service
loads at boot.

**Phase 3 — recalibrate on real data.** Once the app has captures, Open-Meteo has already labelled
them for free. Retrain the probe on real domain photos — phone cameras, Brazil, real angles —
which look nothing like a Kaggle dataset.

The bias to avoid in phase 3: only captures that *passed* the model would otherwise enter the
dataset, so it would reinforce its own errors. The fix is that the label comes from Open-Meteo, not
from the model, and **UNCONFIRMED captures are included too**. There is no loop.

### Datasets

| Dataset | Size | Used for |
|---|---|---|
| Kaggle *Weather Image Recognition* | ~6,900 photos, 11 classes | Head 2. Map the 11 classes onto the 6 visual groups |
| Kaggle *Multi-class Weather* | ~1,500 photos, 4 classes | Held-out test set |
| **Negatives, self-assembled** | ~300 photos | Head 1. Screenshots, walls, selfies, interiors, photos of photos |

The third is the one nobody ships ready-made and the one that matters most — it is literally the
fraud you are trying to block. An afternoon of screenshots and indoor photos covers it.

### Calibrating the thresholds

0.60, 0.10 and 0.70 are starting values. The real ones come from a curve:

1. Run the model over the held-out test set
2. For each candidate threshold, count false positives (honest user blocked) and false negatives
   (fraud accepted)
3. Pick the point where false positives stay **below 2%**

False positives are far worse than false negatives here. Accepted fraud costs fake XP; a blocked
honest user costs a user. The strict matrix above tightens this further — expect the contradiction
threshold to land nearer 0.80 than 0.70.

### Training deliverables

A `train.ipynb` in `skydex-vision` producing: per-class accuracy, a 6x6 confusion matrix, an FP/FN
curve over thresholds, and the probe `.npz`.

## Android app

### The phenomenon selector goes

Removed: `CaptureUiState.phenomenon`, `onPhenomenonSelected`, `MissingPhenomenon`, the selector in
`CaptureScreen`, and `phenomenon` from `CreateWeatherEventRequest`.

On the server the field becomes **accepted and ignored** rather than removed — the same pattern
`capturedAt` and the coordinates already follow in the update handler. An older client keeps
working instead of taking a 400.

### The upload moves earlier

Today the upload happens inside `submit()`. With the model running at upload time, a 422 would only
arrive after the user had written everything — which throws away the reason for rejecting early.

The upload is therefore fired from `onPhotoTaken`, in the background. `submit()` awaits the job
already in flight. The retake-during-upload race is already handled by existing machinery (the
`_state.value.photoFile != photoFile` check and the conditional `uploadedPhotoUrl` write); it is
reused, not reinvented.

### New messages

| Status | When | Copy |
|---|---|---|
| 422 on upload | model says it is not the sky | "Essa foto não parece o céu. Aponta a câmera pra cima e tenta de novo." |
| 503 on upload | vision service down | "Não conseguimos analisar a foto agora. Tenta de novo em instantes." |
| 503 on create | Open-Meteo down | "Não conseguimos conferir o clima agora. Sua foto está salva, é só tentar de novo." |

The 503 path needs no new logic: the existing code only clears `uploadedPhotoUrl` on a 400, so the
cached photo survives and the retry reuses it.

### Unconfirmed captures are kept

`CaptureViewModel.discardUnconfirmed` and `CaptureGateway.delete` are removed. The endpoint stays on
the server and in `CaptureRepository`; it simply loses its only caller.

The original reasoning for auto-deleting was sound when UNCONFIRMED meant "the weather did not match
what you chose". It now means, most of the time, "the model disagreed with your photo" — and with
the strict matrix that will happen to honest users. Silently destroying their photo is the wrong
answer.

`MyCapturesScreen` and `CaptureDetailScreen` gain a visual not-confirmed state showing the reason.
`SkyDexService`, `BadgeService` and `ProfileService` already filter on `CONFIRMED`, so keeping these
rows pollutes neither the collection nor the level nor the species list.

**One thing to revisit:** `BadgeService:66` counts UNCONFIRMED captures for a badge. That count has
been artificially near-zero because the client deleted the rows. It will now count for real, and
what that badge means needs rechecking.

Because the reason is now visible and actionable, this also leaves the natural place for a "dispute
this analysis" action later. Out of scope now; the design does not close the door.

### The reward overlay becomes the reveal

`CaptureRewardOverlay` already receives `phenomenonName`, `rarity`, `confirmed` and `xpAwarded`. It
gains `unconfirmedReason` so it can say why, instead of today's generic copy.

## Implementation order

Each phase is independently verifiable and the app is never broken between them.

| Phase | Scope | Estimate | Note |
|---|---|---|---|
| 1 | `skydex-vision` standalone | 2-3 days | SkyDex untouched |
| 2 | Backend: analysis at upload | 1 day | Old app still works |
| 3 | Backend: Open-Meteo decides | 1 day | The turning point |
| 4 | Android app | 1-2 days | |
| 5 | Recalibrate on real data | — | After production traffic exists |

Roughly a week and a half of focused work through phase 4.

**Phase 1** ends when a `curl` of a photo returns scores and the notebook prints a confusion matrix.

**Phase 2** ships the `VisionClient`, the four columns, the 422 and the 503. The phenomenon still
comes from the user; the app on your phone behaves identically, except a screenshot is now refused.

**Phase 3** flips the source of truth, adds the matrix and adds `unconfirmedReason`. The old app
still works — the user's choice merely becomes decorative.

**Phase 4** removes the selector and ships the new client behaviour.

## Testing

### `skydex-vision` (pytest)

| Test | What it pins |
|---|---|
| Endpoint contract | JSON shape; 400 on a non-image |
| **Accuracy regression** | Runs the held-out set; fails below the recorded baseline |
| Golden set (~30 photos) | 15 real skies, 15 frauds. Seconds to run, fits in CI |

The regression test is the important one. Without it a prompt tweak can degrade the model silently.

### Backend (Testcontainers, as today)

| Test | What it pins |
|---|---|
| **`PhotoAuthenticityServiceTest`** | All 36 matrix cells, as a table. No network, no model |
| `VisionClientTest` | `MockRestServiceServer`: timeout, malformed JSON, 5xx all become null |
| `PhotoControllerTest` | 422 on a low score, 503 when vision is down, the four columns written |
| `WeatherEventControllerTest` | 503 without Open-Meteo, request `phenomenon` ignored, `unconfirmedReason` in the response |
| **Photo not consumed on a 503** | The entire guarantee that retry works |

CLIP does not run in the backend's CI. `VisionClient` is an interface and the tests use a fake; the
real coupling is covered by a manual smoke test.

### App (JUnit + fake gateway, as today)

- Upload fires from `onPhotoTaken`, not from `submit`
- `submit` awaits the in-flight job instead of uploading again
- A 422 shows the not-the-sky message
- **A 503 preserves `uploadedPhotoUrl`** so the retry reuses the photo
- A retake during the early upload does not cache the replaced file's path
- The `MissingPhenomenon` tests are deleted

## Go-live criterion

False positives below **2%** on the golden set. Below that bar phase 3 does not ship, because
blocking honest users costs more than letting fraud through.

## Known risks

1. **Zero-shot may not reach 70% on the phenomenon head.** Likely, in fact. Mitigated by the linear
   probe already being part of phase 1 rather than deferred.
2. **Container size.** Full `torch` pulls CUDA and exceeds 5GB. Install from the CPU-only index
   (`--index-url https://download.pytorch.org/whl/cpu`) — roughly 1.5GB.
3. **Kaggle photos do not look like phone photos.** Well-framed, well-lit, not what users send. This
   is why phase 5 exists, and why the go-live bar is measured on the self-assembled golden set
   rather than on Kaggle.
