# skydex-vision Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone HTTP service that scores a photograph on two questions — "is this an outdoor sky?" and "which of six weather groups does it look like?" — and returns the numbers without interpreting them.

**Architecture:** A FastAPI app wrapping CLIP ViT-B/32. Prompt embeddings are computed once at boot; each request encodes one image and takes two softmaxes over precomputed text features. All scoring arithmetic lives in pure functions that never touch the model, so the policy logic is unit-tested without loading 350MB of weights. An optional linear probe (a `.npz` of logistic-regression coefficients) overrides the zero-shot phenomenon head when present.

**Tech Stack:** Python 3.11, FastAPI, uvicorn, PyTorch (CPU-only build), `open_clip_torch`, Pillow, NumPy, scikit-learn (training only), pytest.

**Spec:** `docs/superpowers/specs/2026-08-16-ai-photo-validation-design.md`

## Global Constraints

- Python **3.11**. The developer machine has Python 3.14, which PyTorch has no wheels for. All work happens inside the container or a 3.11 virtualenv.
- PyTorch must be installed from the **CPU-only index** (`--index-url https://download.pytorch.org/whl/cpu`). The default index pulls CUDA and takes the image past 5GB. There is no NVIDIA GPU on this machine.
- CLIP model: **`ViT-B-32`**, pretrained **`laion2b_s34b_b79k`**, via `open_clip`.
- The service **never** decides whether a capture is valid. It returns `outdoor_score`, `phenomenon_scores` and `model`. No thresholds, no verdicts, no knowledge of `Phenomenon` or `ValidationStatus`.
- The six visual groups, exactly these names: `CLEAR`, `CLOUDY`, `FOG`, `RAIN`, `SNOW`, `STORM`.
- `phenomenon_scores` sums to 1.0 across all six groups (within 1e-6).
- Service listens on port **8000**.
- Code and comments in **English**.
- Never commit or push without asking the user first.

---

## File Structure

```
skydex-vision/
  requirements.txt          runtime deps (no torch — see Dockerfile)
  requirements-dev.txt      pytest and training deps
  Dockerfile                python:3.11-slim + CPU torch + baked model weights
  docker-compose.yml        skydex-db + skydex-vision together
  README.md                 how to run, how to retrain
  .gitignore
  app/
    __init__.py
    prompts.py              the two prompt sets. Data only, no logic.
    scoring.py              pure arithmetic over similarity vectors. No torch.
    model.py                CLIP load + encode. The only file that imports torch.
    probe.py                optional linear-probe load and apply.
    main.py                 FastAPI app and the /v1/analyze route.
  tests/
    __init__.py
    test_scoring.py         pure-function tests. Fast, no model.
    test_prompts.py         guards the group names and set sizes.
    test_api.py             route contract against a stubbed model.
    test_accuracy.py        golden-set regression. Loads the real model.
  data/
    golden/
      manifest.csv          filename,label — the committed source of truth
      images/               30 JPEGs
  training/
    train_probe.py          builds probe.npz from a labelled folder
    train.ipynb             the report: accuracy, confusion matrix, FP/FN curve
```

Why `scoring.py` holds no torch: it is where every number the backend depends on is produced, so it must be testable in milliseconds with hand-written inputs. `model.py` is the only file that has to be slow.

---

### Task 1: Project skeleton with a health endpoint

**Files:**
- Create: `skydex-vision/requirements.txt`
- Create: `skydex-vision/requirements-dev.txt`
- Create: `skydex-vision/.gitignore`
- Create: `skydex-vision/app/__init__.py`
- Create: `skydex-vision/app/main.py`
- Create: `skydex-vision/tests/__init__.py`
- Test: `skydex-vision/tests/test_api.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `app.main.app` — the FastAPI instance every later task mounts routes on.

- [ ] **Step 1: Create the directory and the dependency files**

```bash
mkdir -p ~/Documentos/workspace-becker/skydex-vision/{app,tests,data/golden/images,training}
cd ~/Documentos/workspace-becker/skydex-vision
git init
```

`requirements.txt`:
```
fastapi==0.115.0
uvicorn[standard]==0.30.6
python-multipart==0.0.9
pillow==10.4.0
numpy==1.26.4
open_clip_torch==2.26.1
```

`requirements-dev.txt`:
```
-r requirements.txt
pytest==8.3.3
httpx==0.27.2
scikit-learn==1.5.2
jupyter==1.1.1
matplotlib==3.9.2
```

`.gitignore`:
```
__pycache__/
*.pyc
.venv/
.pytest_cache/
*.npz
.ipynb_checkpoints/
```

`torch` is deliberately absent from both files. It has to come from the CPU index with its own `pip install` line, and putting it here would let someone install the 5GB CUDA build by accident.

- [ ] **Step 2: Create the virtualenv on Python 3.11**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
python3.11 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install torch==2.4.1 --index-url https://download.pytorch.org/whl/cpu
.venv/bin/pip install -r requirements-dev.txt
```

If `python3.11` is not on the machine: `sudo apt install python3.11 python3.11-venv`.

- [ ] **Step 3: Write the failing test**

`tests/test_api.py`:
```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health_reports_ok():
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

`tests/__init__.py` and `app/__init__.py` are both empty files.

- [ ] **Step 4: Run the test and watch it fail**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
.venv/bin/pytest tests/test_api.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.main'`.

- [ ] **Step 5: Write the minimal implementation**

`app/main.py`:
```python
"""HTTP surface of the SkyDex vision service.

This service answers two questions about a photograph and nothing else:
whether it looks like an outdoor sky, and which of six weather groups it
resembles. It returns numbers. It does not know what a capture is, what a
threshold is, or what the SkyDex backend will do with the answer — that
policy lives in Kotlin, next to the rest of the validation semantics.
"""

from fastapi import FastAPI

app = FastAPI(title="skydex-vision", version="1.0.0")


@app.get("/health")
def health() -> dict[str, str]:
    """Liveness only. It deliberately does not touch the model: an endpoint
    that loads 350MB of weights to answer is not a health check."""
    return {"status": "ok"}
```

- [ ] **Step 6: Run the test and watch it pass**

```bash
.venv/bin/pytest tests/test_api.py -v
```

Expected: 1 passed.

- [ ] **Step 7: Commit**

Ask the user before committing. Once they approve:

```bash
git add requirements.txt requirements-dev.txt .gitignore app tests
git commit -m "feat: skeleton FastAPI service with a health endpoint"
```

---

### Task 2: Prompt sets

**Files:**
- Create: `skydex-vision/app/prompts.py`
- Test: `skydex-vision/tests/test_prompts.py`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `SKY_PROMPTS: list[str]`
  - `NOT_SKY_PROMPTS: list[str]`
  - `OUTDOOR_PROMPTS: list[str]` — `SKY_PROMPTS + NOT_SKY_PROMPTS`, in that order
  - `GROUP_ORDER: list[str]` — the six group names in a fixed order
  - `GROUP_PROMPTS: dict[str, list[str]]` — group name to its prompts
  - `PHENOMENON_PROMPTS: list[str]` — every group's prompts concatenated in `GROUP_ORDER`
  - `GROUP_SIZES: list[tuple[str, int]]` — `(group name, prompt count)` in `GROUP_ORDER`

- [ ] **Step 1: Write the failing test**

`tests/test_prompts.py`:
```python
from app import prompts


def test_group_order_is_the_six_visual_groups():
    assert prompts.GROUP_ORDER == ["CLEAR", "CLOUDY", "FOG", "RAIN", "SNOW", "STORM"]


def test_every_group_has_prompts():
    for group in prompts.GROUP_ORDER:
        assert prompts.GROUP_PROMPTS[group], f"{group} has no prompts"


def test_phenomenon_prompts_is_the_groups_concatenated_in_order():
    expected = [p for group in prompts.GROUP_ORDER for p in prompts.GROUP_PROMPTS[group]]

    assert prompts.PHENOMENON_PROMPTS == expected


def test_group_sizes_matches_the_concatenation():
    assert [name for name, _ in prompts.GROUP_SIZES] == prompts.GROUP_ORDER
    assert sum(count for _, count in prompts.GROUP_SIZES) == len(prompts.PHENOMENON_PROMPTS)


def test_outdoor_prompts_puts_sky_first():
    assert prompts.OUTDOOR_PROMPTS[: len(prompts.SKY_PROMPTS)] == prompts.SKY_PROMPTS
```

The ordering tests are not pedantry. `scoring.py` slices these lists by position, so a reordering that nothing catches would silently swap what "sky" means.

- [ ] **Step 2: Run the test and watch it fail**

```bash
.venv/bin/pytest tests/test_prompts.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.prompts'`.

- [ ] **Step 3: Write the implementation**

`app/prompts.py`:
```python
"""The text side of both CLIP heads.

Prompt wording carries most of the accuracy in a zero-shot CLIP classifier.
"rain" and "a photo of a rainy overcast sky, raindrops, wet ground" are not
close in embedding space, and the gap between them is several accuracy points.
Treat this file as a tunable parameter, not as boilerplate — and re-run
tests/test_accuracy.py after every edit.
"""

# --- Head 1: is this an outdoor sky? -------------------------------------------------
#
# The NOT_SKY set is the fraud catalogue: it is what people actually point a
# camera at when they are not pointing it at the sky.

SKY_PROMPTS = [
    "a photo of the sky",
    "an outdoor photo of clouds in the sky",
    "a photograph taken outdoors looking up at the sky",
]

NOT_SKY_PROMPTS = [
    "a screenshot of a phone screen",
    "an indoor photo of a room",
    "a selfie of a person's face",
    "a photo of a blank wall",
    "a photo of a printed photograph",
    "a close-up photo of an object on a table",
]

OUTDOOR_PROMPTS = SKY_PROMPTS + NOT_SKY_PROMPTS

# --- Head 2: which visual group? -----------------------------------------------------
#
# Six groups, not nine phenomena. Drizzle, rain and rain showers are the same
# photograph; so are thunderstorms and hailstorms. Asking CLIP to separate them
# would be asking it to guess.

GROUP_ORDER = ["CLEAR", "CLOUDY", "FOG", "RAIN", "SNOW", "STORM"]

GROUP_PROMPTS: dict[str, list[str]] = {
    "CLEAR": [
        "a photo of a clear blue sky with no clouds",
        "a photo of bright sunshine in a cloudless sky",
    ],
    "CLOUDY": [
        "a photo of an overcast grey sky full of clouds",
        "a photo of a cloudy sky with thick white clouds",
    ],
    "FOG": [
        "a photo of thick fog with very low visibility",
        "a photo of a misty foggy landscape",
    ],
    "RAIN": [
        "a photo of a rainy overcast sky, raindrops, wet ground",
        "a photo taken during rainfall, wet street, grey sky",
    ],
    "SNOW": [
        "a photo of snow falling, snow covered ground",
        "a photo of a snowy winter landscape",
    ],
    "STORM": [
        "a photo of a dark dramatic storm sky with lightning",
        "a photo of heavy thunderstorm clouds, very dark sky",
    ],
}

PHENOMENON_PROMPTS = [p for group in GROUP_ORDER for p in GROUP_PROMPTS[group]]

GROUP_SIZES = [(group, len(GROUP_PROMPTS[group])) for group in GROUP_ORDER]
```

- [ ] **Step 4: Run the test and watch it pass**

```bash
.venv/bin/pytest tests/test_prompts.py -v
```

Expected: 5 passed.

- [ ] **Step 5: Commit**

Ask the user first, then:

```bash
git add app/prompts.py tests/test_prompts.py
git commit -m "feat: prompt sets for the outdoor and phenomenon heads"
```

---

### Task 3: Pure scoring functions

**Files:**
- Create: `skydex-vision/app/scoring.py`
- Test: `skydex-vision/tests/test_scoring.py`

**Interfaces:**
- Consumes: `app.prompts.GROUP_SIZES`, `app.prompts.SKY_PROMPTS`.
- Produces:
  - `softmax(values: Sequence[float], temperature: float = 100.0) -> list[float]`
  - `outdoor_score(similarities: Sequence[float], sky_count: int) -> float`
  - `group_scores(similarities: Sequence[float], group_sizes: Sequence[tuple[str, int]]) -> dict[str, float]`

These three are the entire numeric contract of the service. Everything else is plumbing.

- [ ] **Step 1: Write the failing test**

`tests/test_scoring.py`:
```python
import math

import pytest

from app.scoring import group_scores, outdoor_score, softmax


def test_softmax_sums_to_one():
    result = softmax([0.1, 0.2, 0.3], temperature=1.0)

    assert math.isclose(sum(result), 1.0, abs_tol=1e-9)


def test_softmax_ranks_the_largest_input_highest():
    result = softmax([0.1, 0.9, 0.2], temperature=1.0)

    assert result.index(max(result)) == 1


def test_softmax_is_stable_for_large_inputs():
    # Temperature 100 is CLIP's logit scale, so a similarity of 0.9 becomes 90.
    # A naive exp() on that is fine, but on a batch with a wide spread it is not:
    # this test pins the max-subtraction that keeps it from overflowing to inf.
    result = softmax([0.9, 0.1, 0.05], temperature=100.0)

    assert all(math.isfinite(value) for value in result)
    assert math.isclose(sum(result), 1.0, abs_tol=1e-9)


def test_outdoor_score_is_the_mass_on_the_sky_prompts():
    # Three sky prompts, two not-sky prompts. The sky ones dominate.
    similarities = [0.30, 0.30, 0.30, 0.05, 0.05]

    score = outdoor_score(similarities, sky_count=3)

    assert score > 0.99


def test_outdoor_score_is_low_when_the_not_sky_prompts_win():
    similarities = [0.05, 0.05, 0.05, 0.40, 0.40]

    score = outdoor_score(similarities, sky_count=3)

    assert score < 0.01


def test_group_scores_sums_to_one_across_the_groups():
    sizes = [("CLEAR", 2), ("CLOUDY", 2), ("RAIN", 2)]
    similarities = [0.1, 0.1, 0.5, 0.4, 0.2, 0.2]

    scores = group_scores(similarities, sizes)

    assert set(scores) == {"CLEAR", "CLOUDY", "RAIN"}
    assert math.isclose(sum(scores.values()), 1.0, abs_tol=1e-6)


def test_group_scores_credits_a_group_with_both_of_its_prompts():
    # CLOUDY holds the two strongest prompts, so it must win even though each
    # individual prompt is only slightly ahead.
    sizes = [("CLEAR", 2), ("CLOUDY", 2)]
    similarities = [0.20, 0.20, 0.25, 0.25]

    scores = group_scores(similarities, sizes)

    assert scores["CLOUDY"] > scores["CLEAR"]


def test_group_scores_rejects_a_length_mismatch():
    # A silent mismatch would shift every group's slice by one and produce
    # confident, wrong answers. Fail loudly instead.
    with pytest.raises(ValueError):
        group_scores([0.1, 0.2], [("CLEAR", 2), ("CLOUDY", 2)])
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
.venv/bin/pytest tests/test_scoring.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.scoring'`.

- [ ] **Step 3: Write the implementation**

`app/scoring.py`:
```python
"""Turning CLIP cosine similarities into the two score sets the API returns.

Nothing here imports torch. That is the point: these are the only numbers the
SkyDex backend ever sees, so they must be testable in milliseconds against
hand-written inputs rather than only against a 350MB model.
"""

import math
from collections.abc import Sequence

# CLIP's trained logit scale. Cosine similarities live in a narrow band around
# 0.2-0.3, so a softmax over the raw values is nearly uniform and says nothing.
# Multiplying by 100 is what the model was trained with and what makes the
# distribution informative.
DEFAULT_TEMPERATURE = 100.0


def softmax(values: Sequence[float], temperature: float = DEFAULT_TEMPERATURE) -> list[float]:
    """A numerically stable softmax over ``values`` scaled by ``temperature``.

    The max is subtracted before exponentiating. Without it, a similarity of
    0.9 at temperature 100 becomes exp(90), which is finite but close enough to
    the ceiling that a slightly wider spread overflows to inf and poisons the
    whole vector with nan.
    """
    if not values:
        raise ValueError("softmax needs at least one value")

    scaled = [value * temperature for value in values]
    ceiling = max(scaled)
    exponentiated = [math.exp(value - ceiling) for value in scaled]
    total = sum(exponentiated)
    return [value / total for value in exponentiated]


def outdoor_score(similarities: Sequence[float], sky_count: int) -> float:
    """The probability mass sitting on the sky prompts.

    ``similarities`` must be ordered sky-prompts-first, which is what
    ``prompts.OUTDOOR_PROMPTS`` guarantees and ``test_prompts.py`` pins.
    """
    if sky_count <= 0 or sky_count >= len(similarities):
        raise ValueError(
            f"sky_count must split the vector, got {sky_count} for {len(similarities)} values"
        )

    probabilities = softmax(similarities)
    return sum(probabilities[:sky_count])


def group_scores(
    similarities: Sequence[float],
    group_sizes: Sequence[tuple[str, int]],
) -> dict[str, float]:
    """Collapse per-prompt probabilities into one probability per visual group.

    A group owns several prompts, and a photograph that matches any of them
    matches the group — so the group's score is the sum of its prompts' shares,
    not the best of them. Summing is what lets a group win on two decent prompts
    against a rival group's one strong prompt, which is the behaviour we want:
    the prompts within a group are alternative phrasings of the same claim.
    """
    expected = sum(count for _, count in group_sizes)
    if expected != len(similarities):
        raise ValueError(
            f"expected {expected} similarities for these groups, got {len(similarities)}"
        )

    probabilities = softmax(similarities)

    scores: dict[str, float] = {}
    offset = 0
    for name, count in group_sizes:
        scores[name] = sum(probabilities[offset : offset + count])
        offset += count
    return scores
```

- [ ] **Step 4: Run the test and watch it pass**

```bash
.venv/bin/pytest tests/test_scoring.py -v
```

Expected: 8 passed.

- [ ] **Step 5: Commit**

Ask the user first, then:

```bash
git add app/scoring.py tests/test_scoring.py
git commit -m "feat: pure scoring functions for both CLIP heads"
```

---

### Task 4: CLIP wrapper and the /v1/analyze route

**Files:**
- Create: `skydex-vision/app/model.py`
- Modify: `skydex-vision/app/main.py`
- Test: `skydex-vision/tests/test_api.py`

**Interfaces:**
- Consumes: `app.prompts` (all exports), `app.scoring.outdoor_score`, `app.scoring.group_scores`.
- Produces:
  - `app.model.VisionModel` — a class with `.name: str` and `.analyze(image_bytes: bytes) -> tuple[float, dict[str, float]]` returning `(outdoor_score, group_scores)`
  - `app.model.get_model() -> VisionModel` — the FastAPI dependency, cached
  - `POST /v1/analyze` — multipart field `file`, response `{"outdoor_score": float, "phenomenon_scores": {6 groups}, "model": str}`

- [ ] **Step 1: Write the failing test**

Replace `tests/test_api.py` entirely:
```python
import io

import pytest
from fastapi.testclient import TestClient
from PIL import Image

from app.main import app, get_model
from app.prompts import GROUP_ORDER


class StubModel:
    """Stands in for CLIP so the route contract can be tested in milliseconds.

    The real model is exercised by tests/test_accuracy.py, which is a different
    kind of test with a different failure meaning: this one breaks when the HTTP
    contract changes, that one breaks when the model gets worse.
    """

    name = "stub-v0"

    def __init__(self, outdoor: float = 0.9):
        self.outdoor = outdoor
        self.calls: list[bytes] = []

    def analyze(self, image_bytes: bytes) -> tuple[float, dict[str, float]]:
        self.calls.append(image_bytes)
        scores = {group: 0.1 for group in GROUP_ORDER}
        scores["RAIN"] = 0.5
        return self.outdoor, scores


@pytest.fixture
def stub():
    model = StubModel()
    app.dependency_overrides[get_model] = lambda: model
    yield model
    app.dependency_overrides.clear()


@pytest.fixture
def client():
    return TestClient(app)


def jpeg_bytes(colour: str = "blue") -> bytes:
    buffer = io.BytesIO()
    Image.new("RGB", (64, 64), colour).save(buffer, format="JPEG")
    return buffer.getvalue()


def test_health_reports_ok(client):
    assert client.get("/health").json() == {"status": "ok"}


def test_analyze_returns_the_documented_shape(client, stub):
    response = client.post(
        "/v1/analyze",
        files={"file": ("sky.jpg", jpeg_bytes(), "image/jpeg")},
    )

    assert response.status_code == 200
    body = response.json()
    assert set(body) == {"outdoor_score", "phenomenon_scores", "model"}
    assert body["model"] == "stub-v0"
    assert body["outdoor_score"] == pytest.approx(0.9)
    assert set(body["phenomenon_scores"]) == set(GROUP_ORDER)


def test_analyze_passes_the_uploaded_bytes_to_the_model(client, stub):
    payload = jpeg_bytes("red")

    client.post("/v1/analyze", files={"file": ("sky.jpg", payload, "image/jpeg")})

    assert stub.calls == [payload]


def test_analyze_rejects_a_file_that_is_not_an_image(client, stub):
    response = client.post(
        "/v1/analyze",
        files={"file": ("notes.txt", b"this is not an image", "text/plain")},
    )

    assert response.status_code == 400
    assert "image" in response.json()["detail"].lower()


def test_analyze_rejects_a_missing_file(client, stub):
    assert client.post("/v1/analyze").status_code == 422
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
.venv/bin/pytest tests/test_api.py -v
```

Expected: `ImportError: cannot import name 'get_model' from 'app.main'`.

- [ ] **Step 3: Write the model wrapper**

`app/model.py`:
```python
"""The only file in this service that loads PyTorch.

Text embeddings for both prompt sets are computed once, at construction, and
kept on the instance. A request therefore costs exactly one image encode plus
two matrix multiplies against small precomputed matrices — roughly 250ms on one
CPU core for ViT-B/32.
"""

import io
from functools import lru_cache

import open_clip
import torch
from PIL import Image, UnidentifiedImageError

from app.prompts import GROUP_SIZES, OUTDOOR_PROMPTS, PHENOMENON_PROMPTS, SKY_PROMPTS
from app.scoring import group_scores, outdoor_score

MODEL_ARCHITECTURE = "ViT-B-32"
MODEL_WEIGHTS = "laion2b_s34b_b79k"


class InvalidImageError(ValueError):
    """The uploaded bytes are not an image Pillow can open."""


class VisionModel:
    """CLIP, plus the two precomputed text matrices it is queried against."""

    def __init__(self) -> None:
        self._model, _, self._preprocess = open_clip.create_model_and_transforms(
            MODEL_ARCHITECTURE, pretrained=MODEL_WEIGHTS
        )
        self._model.eval()
        tokenizer = open_clip.get_tokenizer(MODEL_ARCHITECTURE)

        self._outdoor_features = self._encode_text(tokenizer, OUTDOOR_PROMPTS)
        self._phenomenon_features = self._encode_text(tokenizer, PHENOMENON_PROMPTS)

        self.name = f"clip-{MODEL_ARCHITECTURE.lower()}-zeroshot-v1"

    def _encode_text(self, tokenizer, texts: list[str]) -> torch.Tensor:
        with torch.no_grad():
            features = self._model.encode_text(tokenizer(texts))
        # Normalising here means the dot product below IS the cosine similarity,
        # so nothing downstream has to remember to divide.
        return features / features.norm(dim=-1, keepdim=True)

    def embed(self, image_bytes: bytes) -> torch.Tensor:
        """The normalised image embedding. Exposed because training reuses it."""
        try:
            image = Image.open(io.BytesIO(image_bytes)).convert("RGB")
        except (UnidentifiedImageError, OSError) as error:
            raise InvalidImageError("The uploaded file is not a readable image") from error

        with torch.no_grad():
            features = self._model.encode_image(self._preprocess(image).unsqueeze(0))
        return features / features.norm(dim=-1, keepdim=True)

    def analyze(self, image_bytes: bytes) -> tuple[float, dict[str, float]]:
        """``(outdoor_score, phenomenon_scores)`` for one photograph."""
        image_features = self.embed(image_bytes)

        outdoor_similarities = (image_features @ self._outdoor_features.T)[0].tolist()
        phenomenon_similarities = (image_features @ self._phenomenon_features.T)[0].tolist()

        return (
            outdoor_score(outdoor_similarities, sky_count=len(SKY_PROMPTS)),
            group_scores(phenomenon_similarities, GROUP_SIZES),
        )


@lru_cache(maxsize=1)
def load_model() -> VisionModel:
    """Built once per process. The first call takes several seconds."""
    return VisionModel()
```

- [ ] **Step 4: Write the route**

Replace `app/main.py` entirely:
```python
"""HTTP surface of the SkyDex vision service.

This service answers two questions about a photograph and nothing else:
whether it looks like an outdoor sky, and which of six weather groups it
resembles. It returns numbers. It does not know what a capture is, what a
threshold is, or what the SkyDex backend will do with the answer — that
policy lives in Kotlin, next to the rest of the validation semantics.

Keeping the split there rather than here is what lets a threshold change ship
without touching Python, and what keeps this service testable as a pure
function of an image.
"""

from fastapi import Depends, FastAPI, File, HTTPException, UploadFile
from pydantic import BaseModel

from app.model import InvalidImageError, VisionModel, load_model

app = FastAPI(title="skydex-vision", version="1.0.0")


class AnalyzeResponse(BaseModel):
    outdoor_score: float
    phenomenon_scores: dict[str, float]
    model: str


def get_model() -> VisionModel:
    """Indirection so tests can override the model without importing torch."""
    return load_model()


@app.get("/health")
def health() -> dict[str, str]:
    """Liveness only. It deliberately does not touch the model: an endpoint
    that loads 350MB of weights to answer is not a health check."""
    return {"status": "ok"}


@app.post("/v1/analyze", response_model=AnalyzeResponse)
async def analyze(
    file: UploadFile = File(...),
    model: VisionModel = Depends(get_model),
) -> AnalyzeResponse:
    """Score one photograph.

    A 400 means the bytes are not an image. There is no status here for "this
    photo is fraudulent" because this service does not have that opinion.
    """
    payload = await file.read()
    try:
        outdoor, phenomenon = model.analyze(payload)
    except InvalidImageError as error:
        raise HTTPException(status_code=400, detail=str(error)) from error

    return AnalyzeResponse(
        outdoor_score=outdoor,
        phenomenon_scores=phenomenon,
        model=model.name,
    )
```

- [ ] **Step 5: Run the tests and watch them pass**

```bash
.venv/bin/pytest tests/ -v
```

Expected: all pass. The first run downloads the CLIP weights only if something imports `load_model()` for real — the API tests use the stub, so they stay fast.

- [ ] **Step 6: Smoke-test against the real model**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
.venv/bin/uvicorn app.main:app --port 8000 &
sleep 30   # first boot downloads ~350MB of weights
curl -s -F "file=@/path/to/any/sky/photo.jpg" http://localhost:8000/v1/analyze | python3 -m json.tool
kill %1
```

Expected: an `outdoor_score` above 0.8 for a real sky photograph, and six groups summing to 1.

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add app/model.py app/main.py tests/test_api.py
git commit -m "feat: CLIP-backed /v1/analyze endpoint"
```

---

### Task 5: Golden set and the accuracy regression test

**Files:**
- Create: `skydex-vision/data/golden/manifest.csv`
- Create: `skydex-vision/data/golden/images/` (30 JPEGs)
- Create: `skydex-vision/tests/test_accuracy.py`

**Interfaces:**
- Consumes: `app.model.load_model`, `app.prompts.GROUP_ORDER`.
- Produces: the go-live measurement. Nothing imports from this task; the SkyDex integration plan's go-live criterion is measured by it.

- [ ] **Step 1: Assemble the golden set**

Put 30 JPEGs in `data/golden/images/`. This is manual work and it is the most valuable half hour in the whole plan — the public datasets do not contain the fraud you are trying to block.

**15 real skies** (`sky` rows), from your own phone, spread across:
- 4 clear, 4 cloudy, 3 rain, 2 storm, 1 fog, 1 night sky

**15 frauds** (`fraud` rows):
- 4 screenshots of a phone screen (a weather app, a chat, anything)
- 3 photos of an indoor room
- 2 selfies
- 2 photos of a blank wall
- 2 photos of a printed photograph or a monitor showing a sky
- 2 close-ups of an object on a table

Resize everything to a 1024px long edge so the repository stays small:
```bash
cd ~/Documentos/workspace-becker/skydex-vision/data/golden/images
sudo apt install -y imagemagick   # if not present
mogrify -resize 1024x1024\> -quality 85 *.jpg
```

- [ ] **Step 2: Write the manifest**

`data/golden/manifest.csv` — one row per image, `filename,label`, where `label` is `sky` or `fraud`:
```csv
filename,label
sky_clear_01.jpg,sky
sky_clear_02.jpg,sky
sky_clear_03.jpg,sky
sky_clear_04.jpg,sky
sky_cloudy_01.jpg,sky
sky_cloudy_02.jpg,sky
sky_cloudy_03.jpg,sky
sky_cloudy_04.jpg,sky
sky_rain_01.jpg,sky
sky_rain_02.jpg,sky
sky_rain_03.jpg,sky
sky_storm_01.jpg,sky
sky_storm_02.jpg,sky
sky_fog_01.jpg,sky
sky_night_01.jpg,sky
fraud_screenshot_01.jpg,fraud
fraud_screenshot_02.jpg,fraud
fraud_screenshot_03.jpg,fraud
fraud_screenshot_04.jpg,fraud
fraud_indoor_01.jpg,fraud
fraud_indoor_02.jpg,fraud
fraud_indoor_03.jpg,fraud
fraud_selfie_01.jpg,fraud
fraud_selfie_02.jpg,fraud
fraud_wall_01.jpg,fraud
fraud_wall_02.jpg,fraud
fraud_screen_01.jpg,fraud
fraud_screen_02.jpg,fraud
fraud_object_01.jpg,fraud
fraud_object_02.jpg,fraud
```

Rename your files to match, or edit the manifest to match your files. The names carry no meaning to the code — only the `label` column does.

- [ ] **Step 3: Write the regression test**

`tests/test_accuracy.py`:
```python
"""Golden-set regression for the outdoor head.

This is the test that decides whether phase 3 of the SkyDex integration ships.
It is slow — it loads the real model — so it is marked and excluded from the
fast suite. Run it after every prompt edit and after every probe retrain.

    .venv/bin/pytest tests/test_accuracy.py -v -s -m slow
"""

import csv
from pathlib import Path

import pytest

from app.model import load_model

GOLDEN = Path(__file__).parent.parent / "data" / "golden"

# The threshold the SkyDex backend will use for stage 1. Keep this in sync with
# skydex.vision.outdoor-min in the backend's application.properties — a drift
# between the two means this test measures something nobody runs.
OUTDOOR_THRESHOLD = 0.60

# The go-live bar from the spec. False positives are honest skies rejected as
# fraud, and they are far worse than the reverse: accepted fraud costs fake XP,
# a rejected honest user costs a user.
MAX_FALSE_POSITIVE_RATE = 0.02

# Deliberately loose. Some fraud is genuinely hard for CLIP (a high-quality
# photo of a monitor showing a sky is very nearly a photo of a sky), and
# tightening this at the cost of the rate above would be the wrong trade.
MIN_FRAUD_CAUGHT_RATE = 0.70


def load_manifest() -> list[tuple[Path, str]]:
    with (GOLDEN / "manifest.csv").open() as handle:
        rows = list(csv.DictReader(handle))
    return [(GOLDEN / "images" / row["filename"], row["label"]) for row in rows]


@pytest.mark.slow
def test_outdoor_head_meets_the_go_live_bar():
    model = load_model()
    manifest = load_manifest()
    assert manifest, "the golden set is empty — see Task 5 step 1"

    false_positives, skies = 0, 0
    caught, frauds = 0, 0
    misses: list[str] = []

    for path, label in manifest:
        assert path.exists(), f"manifest names a missing file: {path}"
        outdoor, _ = model.analyze(path.read_bytes())
        accepted = outdoor >= OUTDOOR_THRESHOLD

        if label == "sky":
            skies += 1
            if not accepted:
                false_positives += 1
                misses.append(f"REJECTED A REAL SKY  {path.name}  outdoor={outdoor:.3f}")
        else:
            frauds += 1
            if not accepted:
                caught += 1
            else:
                misses.append(f"ACCEPTED A FRAUD     {path.name}  outdoor={outdoor:.3f}")

    false_positive_rate = false_positives / skies
    caught_rate = caught / frauds

    print(f"\nfalse positive rate: {false_positive_rate:.1%} (bar: {MAX_FALSE_POSITIVE_RATE:.0%})")
    print(f"fraud caught rate:   {caught_rate:.1%} (bar: {MIN_FRAUD_CAUGHT_RATE:.0%})")
    for line in misses:
        print("  " + line)

    assert false_positive_rate <= MAX_FALSE_POSITIVE_RATE
    assert caught_rate >= MIN_FRAUD_CAUGHT_RATE
```

- [ ] **Step 4: Register the marker so pytest does not warn**

Create `skydex-vision/pytest.ini`:
```ini
[pytest]
markers =
    slow: loads the real CLIP model; excluded from the fast suite
addopts = -m "not slow"
```

`addopts` means a bare `pytest` runs only the fast tests. The slow one needs `-m slow` explicitly.

- [ ] **Step 5: Run the fast suite and confirm the slow test is skipped**

```bash
.venv/bin/pytest -v
```

Expected: the scoring, prompt and API tests run; `test_outdoor_head_meets_the_go_live_bar` is deselected.

- [ ] **Step 6: Run the regression test for real**

```bash
.venv/bin/pytest tests/test_accuracy.py -v -s -m slow
```

Expected on a first run: it may well **fail**. That is the test doing its job. If it fails, edit `app/prompts.py` — add or reword prompts in `NOT_SKY_PROMPTS` to cover whatever got through — and run it again. The printed misses tell you exactly which images to write prompts for.

Do not lower `OUTDOOR_THRESHOLD` to make it pass. The threshold is a product decision recorded in the spec; the prompts are the tunable part.

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add data/golden pytest.ini tests/test_accuracy.py app/prompts.py
git commit -m "test: golden-set regression for the outdoor head"
```

---

### Task 6: Linear probe training and loading

**Files:**
- Create: `skydex-vision/training/train_probe.py`
- Create: `skydex-vision/training/train.ipynb`
- Create: `skydex-vision/app/probe.py`
- Modify: `skydex-vision/app/model.py`
- Test: `skydex-vision/tests/test_probe.py`

**Interfaces:**
- Consumes: `app.model.VisionModel.embed`, `app.prompts.GROUP_ORDER`.
- Produces:
  - `app.probe.Probe` — `.groups: list[str]`, `.apply(embedding: numpy.ndarray) -> dict[str, float]`
  - `app.probe.load_probe(path: pathlib.Path) -> Probe | None` — `None` when the file is absent
  - `VisionModel.analyze` uses the probe for the phenomenon head when one is loaded, and zero-shot otherwise. `VisionModel.name` reflects which.

- [ ] **Step 1: Download the training data**

Two Kaggle datasets. Both need a free Kaggle account.

```bash
cd ~/Documentos/workspace-becker/skydex-vision
mkdir -p data/train data/test
```

- *Weather Image Recognition* (jehanbhathena) — ~6,900 photos, 11 classes. Unzip into `data/train/`.
- *Multi-class Weather Dataset* (pratik2901) — ~1,500 photos, 4 classes. Unzip into `data/test/`.

Add `data/train/` and `data/test/` to `.gitignore` — they are hundreds of megabytes and they are not yours to redistribute:
```bash
printf 'data/train/\ndata/test/\n' >> .gitignore
```

Map the 11 source classes onto the six groups. Write this mapping in `training/train_probe.py` (step 3) so it is version-controlled rather than done by hand:

| Source class | Group |
|---|---|
| `dew`, `frost` | `FOG` |
| `fogsmog` | `FOG` |
| `glaze`, `rime` | `SNOW` |
| `hail` | `STORM` |
| `lightning` | `STORM` |
| `rain` | `RAIN` |
| `rainbow` | `CLOUDY` |
| `sandstorm` | *dropped* |
| `snow` | `SNOW` |

The source set has no clear-sky and no plain-cloudy class. Fill those two from `data/test/` (`shine` → `CLEAR`, `cloudy` → `CLOUDY`) and hold back the rest of that dataset for testing. `sandstorm` is dropped because it maps to no SkyDex phenomenon and would only teach the probe a class it can never usefully predict.

- [ ] **Step 2: Write the failing test**

`tests/test_probe.py`:
```python
import numpy as np
import pytest

from app.probe import Probe, load_probe
from app.prompts import GROUP_ORDER


def a_probe(dimensions: int = 4) -> Probe:
    # Two groups, hand-built so the expected output is calculable by hand.
    return Probe(
        groups=["CLEAR", "RAIN"],
        weights=np.array([[1.0, 0.0, 0.0, 0.0], [0.0, 1.0, 0.0, 0.0]]),
        bias=np.array([0.0, 0.0]),
    )


def test_apply_returns_one_score_per_group():
    scores = a_probe().apply(np.array([1.0, 0.0, 0.0, 0.0]))

    assert set(scores) == {"CLEAR", "RAIN"}


def test_apply_sums_to_one():
    scores = a_probe().apply(np.array([0.3, 0.7, 0.1, 0.2]))

    assert sum(scores.values()) == pytest.approx(1.0, abs=1e-6)


def test_apply_favours_the_group_whose_weights_match_the_embedding():
    scores = a_probe().apply(np.array([5.0, 0.0, 0.0, 0.0]))

    assert scores["CLEAR"] > scores["RAIN"]


def test_apply_rejects_an_embedding_of_the_wrong_width():
    with pytest.raises(ValueError):
        a_probe().apply(np.array([1.0, 2.0]))


def test_load_probe_returns_none_when_the_file_is_absent(tmp_path):
    assert load_probe(tmp_path / "nothing.npz") is None


def test_load_probe_round_trips_a_saved_probe(tmp_path):
    path = tmp_path / "probe.npz"
    original = a_probe()
    np.savez(
        path,
        groups=np.array(original.groups),
        weights=original.weights,
        bias=original.bias,
    )

    loaded = load_probe(path)

    assert loaded is not None
    assert loaded.groups == original.groups
    np.testing.assert_allclose(loaded.weights, original.weights)


def test_load_probe_rejects_groups_outside_the_known_set(tmp_path):
    # A probe trained on labels the backend has never heard of would return
    # scores nothing downstream can read. Refuse it at load rather than serve it.
    path = tmp_path / "probe.npz"
    np.savez(
        path,
        groups=np.array(["CLEAR", "TORNADO"]),
        weights=np.zeros((2, 4)),
        bias=np.zeros(2),
    )

    with pytest.raises(ValueError, match="TORNADO"):
        load_probe(path)


def test_group_order_is_what_a_probe_may_use():
    assert "TORNADO" not in GROUP_ORDER
```

- [ ] **Step 3: Run the test and watch it fail**

```bash
.venv/bin/pytest tests/test_probe.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.probe'`.

- [ ] **Step 4: Write the probe module**

`app/probe.py`:
```python
"""The trained half of the phenomenon head.

A probe is a multinomial logistic regression over frozen CLIP embeddings —
512 inputs, one output per visual group. It trains in seconds on CPU and
typically beats zero-shot prompting by 10-15 accuracy points, which is why it
exists; it is not a fine-tune of CLIP itself, which this machine has no GPU for.

The artefact is a ~30KB .npz. When one is present the service uses it; when it
is absent the service falls back to zero-shot and says so in its model name.
"""

from dataclasses import dataclass
from pathlib import Path

import numpy as np

from app.prompts import GROUP_ORDER


@dataclass
class Probe:
    groups: list[str]
    weights: np.ndarray  # shape (len(groups), embedding_dim)
    bias: np.ndarray  # shape (len(groups),)

    def apply(self, embedding: np.ndarray) -> dict[str, float]:
        """Group probabilities for one normalised CLIP embedding."""
        flat = np.asarray(embedding, dtype=np.float64).reshape(-1)
        if flat.shape[0] != self.weights.shape[1]:
            raise ValueError(
                f"probe expects {self.weights.shape[1]}-dimensional embeddings, "
                f"got {flat.shape[0]}"
            )

        logits = self.weights @ flat + self.bias
        # Max subtraction for the same overflow reason as scoring.softmax.
        exponentiated = np.exp(logits - logits.max())
        probabilities = exponentiated / exponentiated.sum()
        return {group: float(value) for group, value in zip(self.groups, probabilities)}


def load_probe(path: Path) -> Probe | None:
    """Read a probe from ``path``, or ``None`` if there is nothing there.

    An unreadable or wrongly-labelled probe raises rather than being skipped.
    Silently falling back to zero-shot would mean a deploy that thought it
    shipped a trained model and did not, with no signal anywhere.
    """
    if not path.exists():
        return None

    payload = np.load(path, allow_pickle=False)
    groups = [str(group) for group in payload["groups"]]

    unknown = [group for group in groups if group not in GROUP_ORDER]
    if unknown:
        raise ValueError(f"probe was trained on unknown groups: {', '.join(unknown)}")

    return Probe(groups=groups, weights=payload["weights"], bias=payload["bias"])
```

- [ ] **Step 5: Run the test and watch it pass**

```bash
.venv/bin/pytest tests/test_probe.py -v
```

Expected: 8 passed.

- [ ] **Step 6: Wire the probe into the model**

In `app/model.py`, add the imports:
```python
import os
from pathlib import Path

import numpy as np

from app.probe import Probe, load_probe
```

Add the constant next to the other two:
```python
PROBE_PATH = Path(os.environ.get("SKYDEX_PROBE_PATH", "data/probe.npz"))
```

At the end of `VisionModel.__init__`, replace the `self.name = ...` line with:
```python
        self._probe: Probe | None = load_probe(PROBE_PATH)
        suffix = "probe-v1" if self._probe else "zeroshot-v1"
        self.name = f"clip-{MODEL_ARCHITECTURE.lower()}-{suffix}"
```

Replace the body of `analyze`:
```python
    def analyze(self, image_bytes: bytes) -> tuple[float, dict[str, float]]:
        """``(outdoor_score, phenomenon_scores)`` for one photograph.

        The outdoor head is always zero-shot. Only the phenomenon head is
        trained, because the fraud catalogue in NOT_SKY_PROMPTS is a moving
        target that prompts express better than a fixed training set does.
        """
        image_features = self.embed(image_bytes)

        outdoor_similarities = (image_features @ self._outdoor_features.T)[0].tolist()
        outdoor = outdoor_score(outdoor_similarities, sky_count=len(SKY_PROMPTS))

        if self._probe is not None:
            phenomenon = self._probe.apply(image_features[0].numpy())
        else:
            phenomenon_similarities = (image_features @ self._phenomenon_features.T)[0].tolist()
            phenomenon = group_scores(phenomenon_similarities, GROUP_SIZES)

        return outdoor, phenomenon
```

- [ ] **Step 7: Write the training script**

`training/train_probe.py`:
```python
"""Train the linear probe and write data/probe.npz.

    .venv/bin/python training/train_probe.py data/train data/probe.npz

Expects ``data/train`` to hold one subdirectory per source class. The mapping
onto the six visual groups is SOURCE_TO_GROUP below; source classes absent from
it are skipped, which is how sandstorm is dropped.
"""

import sys
from collections import Counter
from pathlib import Path

import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, confusion_matrix

from app.model import VisionModel
from app.prompts import GROUP_ORDER

SOURCE_TO_GROUP = {
    "dew": "FOG",
    "frost": "FOG",
    "fogsmog": "FOG",
    "glaze": "SNOW",
    "rime": "SNOW",
    "snow": "SNOW",
    "hail": "STORM",
    "lightning": "STORM",
    "rain": "RAIN",
    "rainbow": "CLOUDY",
    "cloudy": "CLOUDY",
    "shine": "CLEAR",
    "sunrise": "CLEAR",
    # "sandstorm" is deliberately absent: it maps to no SkyDex phenomenon, so
    # training on it would only teach a class that can never be acted on.
}

IMAGE_SUFFIXES = {".jpg", ".jpeg", ".png"}


def embed_folder(model: VisionModel, root: Path) -> tuple[np.ndarray, list[str]]:
    vectors, labels = [], []
    for class_dir in sorted(root.iterdir()):
        if not class_dir.is_dir():
            continue
        group = SOURCE_TO_GROUP.get(class_dir.name.lower())
        if group is None:
            print(f"skipping unmapped class {class_dir.name}")
            continue
        images = [p for p in sorted(class_dir.iterdir()) if p.suffix.lower() in IMAGE_SUFFIXES]
        print(f"{class_dir.name} -> {group}: {len(images)} images")
        for path in images:
            try:
                vectors.append(model.embed(path.read_bytes())[0].numpy())
                labels.append(group)
            except Exception as error:  # a corrupt file must not stop a 6,900-image run
                print(f"  skipping {path.name}: {error}")
    return np.stack(vectors), labels


def main() -> None:
    source = Path(sys.argv[1])
    destination = Path(sys.argv[2])

    model = VisionModel()
    features, labels = embed_folder(model, source)
    print(f"\n{len(labels)} embeddings, distribution: {Counter(labels)}")

    classifier = LogisticRegression(max_iter=2000, C=1.0, multi_class="multinomial")
    classifier.fit(features, labels)

    predictions = classifier.predict(features)
    print("\n--- on the training set (optimistic by construction) ---")
    print(classification_report(labels, predictions))
    print(confusion_matrix(labels, predictions, labels=sorted(set(labels))))

    groups = list(classifier.classes_)
    unknown = [group for group in groups if group not in GROUP_ORDER]
    if unknown:
        raise SystemExit(f"refusing to save a probe with unknown groups: {unknown}")

    destination.parent.mkdir(parents=True, exist_ok=True)
    np.savez(
        destination,
        groups=np.array(groups),
        weights=classifier.coef_,
        bias=classifier.intercept_,
    )
    print(f"\nwrote {destination} ({destination.stat().st_size} bytes)")


if __name__ == "__main__":
    main()
```

- [ ] **Step 8: Train it**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
.venv/bin/python training/train_probe.py data/train data/probe.npz
```

Expected: several minutes of embedding (that is the slow part — 6,900 forward passes on CPU), then a classification report and a saved `.npz` of roughly 30KB. The logistic regression itself takes seconds.

- [ ] **Step 9: Write the report notebook**

`training/train.ipynb` — create it with `.venv/bin/jupyter notebook` and give it these cells, in order. This notebook is the deliverable that turns "I used CLIP" into a piece of ML work with numbers behind it.

Cell 1 — embed the held-out test set:
```python
from pathlib import Path
import numpy as np
from app.model import VisionModel
from training.train_probe import embed_folder

model = VisionModel()
features, labels = embed_folder(model, Path("data/test"))
print(len(labels), set(labels))
```

Cell 2 — accuracy and confusion matrix, zero-shot vs probe:
```python
from sklearn.metrics import accuracy_score, classification_report, confusion_matrix
from app.probe import load_probe
from app.prompts import GROUP_SIZES
from app.scoring import group_scores
import torch

probe = load_probe(Path("data/probe.npz"))

zero_shot, probed = [], []
for vector in features:
    sims = (torch.tensor(vector) @ model._phenomenon_features.T).tolist()
    zero_shot.append(max(group_scores(sims, GROUP_SIZES).items(), key=lambda kv: kv[1])[0])
    probed.append(max(probe.apply(vector).items(), key=lambda kv: kv[1])[0])

print("zero-shot:", accuracy_score(labels, zero_shot))
print("probe:    ", accuracy_score(labels, probed))
print(classification_report(labels, probed))
```

Cell 3 — the 6x6 confusion matrix as a figure:
```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay
from app.prompts import GROUP_ORDER

present = [g for g in GROUP_ORDER if g in set(labels)]
fig, ax = plt.subplots(figsize=(7, 7))
ConfusionMatrixDisplay.from_predictions(labels, probed, labels=present, ax=ax, xticks_rotation=45)
ax.set_title("Visual group confusion — linear probe")
plt.tight_layout()
plt.savefig("training/confusion_matrix.png", dpi=120)
```

Cell 4 — the FP/FN curve that sets the outdoor threshold:
```python
import csv

golden = Path("data/golden")
rows = list(csv.DictReader((golden / "manifest.csv").open()))
scored = [(model.analyze((golden / "images" / r["filename"]).read_bytes())[0], r["label"])
          for r in rows]

thresholds = [i / 100 for i in range(30, 96, 5)]
fp_rates, fn_rates = [], []
for t in thresholds:
    skies = [s for s, l in scored if l == "sky"]
    frauds = [s for s, l in scored if l == "fraud"]
    fp_rates.append(sum(1 for s in skies if s < t) / len(skies))
    fn_rates.append(sum(1 for s in frauds if s >= t) / len(frauds))

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(thresholds, fp_rates, marker="o", label="false positive (honest sky rejected)")
ax.plot(thresholds, fn_rates, marker="s", label="false negative (fraud accepted)")
ax.axhline(0.02, linestyle="--", color="grey", label="go-live bar: FP <= 2%")
ax.set_xlabel("outdoor_score threshold")
ax.set_ylabel("rate")
ax.legend()
plt.tight_layout()
plt.savefig("training/threshold_curve.png", dpi=120)

for t, fp, fn in zip(thresholds, fp_rates, fn_rates):
    print(f"{t:.2f}  FP {fp:.1%}  FN {fn:.1%}")
```

The lowest threshold whose FP is at or below 2% is the value to put in the backend's `skydex.vision.outdoor-min` and in `OUTDOOR_THRESHOLD` in `tests/test_accuracy.py`. Update both.

- [ ] **Step 10: Re-run the whole suite**

```bash
.venv/bin/pytest -v
.venv/bin/pytest tests/test_accuracy.py -v -s -m slow
```

Expected: fast suite green; the accuracy test still meets the bar with the probe loaded. If the probe made things worse, delete `data/probe.npz` and ship zero-shot — the fallback exists precisely so that is a one-command decision.

- [ ] **Step 11: Commit**

`data/probe.npz` is already gitignored by the `*.npz` rule from Task 1. Keep it that way: it is a build artefact reproducible from `train_probe.py`, and binary blobs in git age badly. Ask the user first, then:

```bash
git add app/probe.py app/model.py tests/test_probe.py training/
git commit -m "feat: linear probe for the phenomenon head"
```

---

### Task 7: Container and compose

**Files:**
- Create: `skydex-vision/Dockerfile`
- Create: `skydex-vision/.dockerignore`
- Create: `skydex-vision/docker-compose.yml`
- Create: `skydex-vision/README.md`

**Interfaces:**
- Consumes: everything above.
- Produces: `skydex-vision` reachable at `http://localhost:8000` and, from inside the compose network, `http://skydex-vision:8000`. The SkyDex integration plan's `skydex.vision.base-url` points at one of these.

- [ ] **Step 1: Write the Dockerfile**

`Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /srv

# CPU-only torch, from its own index. The default PyPI wheel pulls the CUDA
# runtime and takes this image past 5GB for a machine that has no NVIDIA GPU.
RUN pip install --no-cache-dir torch==2.4.1 --index-url https://download.pytorch.org/whl/cpu

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Bake the weights into the image. Without this the first request after every
# container start waits on a 350MB download, and a container that cannot reach
# the internet never becomes useful at all.
RUN python -c "import open_clip; open_clip.create_model_and_transforms('ViT-B-32', pretrained='laion2b_s34b_b79k')"

COPY app ./app

EXPOSE 8000

# One worker. The model is ~350MB of resident memory per process and this
# machine has 14GB total shared with Postgres and the JVM; a second worker buys
# throughput nobody needs and costs memory that is genuinely scarce.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

`.dockerignore`:
```
.venv/
data/train/
data/test/
data/golden/
training/
tests/
__pycache__/
.git/
```

- [ ] **Step 2: Build and smoke-test the image**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
docker build -t skydex-vision:latest .
docker run --rm -d -p 8000:8000 --name skydex-vision-smoke skydex-vision:latest
sleep 15
curl -s http://localhost:8000/health
curl -s -F "file=@data/golden/images/sky_clear_01.jpg" http://localhost:8000/v1/analyze | python3 -m json.tool
docker stop skydex-vision-smoke
```

Expected: `{"status":"ok"}`, then a full analysis body. Check the image size with `docker images skydex-vision` — anything over 3GB means torch came from the wrong index.

- [ ] **Step 3: Write the compose file**

`docker-compose.yml`:
```yaml
# Brings up the two containers the SkyDex backend talks to. The backend itself
# still runs on the host via `./gradlew bootRun` — it is the thing being
# developed, so it stays outside the compose network and reaches both of these
# through published ports.
services:
  skydex-db:
    image: postgres:16
    container_name: skydex-db
    environment:
      POSTGRES_DB: skydex
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - skydex-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10

  skydex-vision:
    build: .
    container_name: skydex-vision
    ports:
      - "8000:8000"
    volumes:
      # Mounted rather than baked so retraining does not need an image rebuild.
      # Absent on a fresh checkout, which is fine — the service falls back to
      # zero-shot and says so in its model name.
      - ./data:/srv/data:ro
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request; urllib.request.urlopen('http://localhost:8000/health')\""]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  skydex-db-data:
```

The existing `skydex-db` container was created by hand. Compose will refuse to create one with the same name — remove the old one first (`docker rm -f skydex-db`) and let compose own it. **Check with the user before removing it**: it holds the development database. If they want to keep it, run `docker exec skydex-db pg_dump -U postgres skydex > backup.sql` first.

- [ ] **Step 4: Bring the stack up**

```bash
cd ~/Documentos/workspace-becker/skydex-vision
docker compose up -d
docker compose ps
curl -s http://localhost:8000/health
```

Expected: both services healthy.

- [ ] **Step 5: Write the README**

`README.md`:
```markdown
# skydex-vision

Scores a photograph for the SkyDex backend: how much it looks like an outdoor
sky, and which of six weather groups it resembles. It returns numbers only —
every threshold and every verdict lives in the Kotlin backend.

## Run it

    docker compose up -d
    curl -F "file=@photo.jpg" http://localhost:8000/v1/analyze

## Develop

Python 3.11 (not 3.14 — PyTorch has no wheels for it):

    python3.11 -m venv .venv
    .venv/bin/pip install torch==2.4.1 --index-url https://download.pytorch.org/whl/cpu
    .venv/bin/pip install -r requirements-dev.txt
    .venv/bin/pytest                                  # fast suite
    .venv/bin/pytest tests/test_accuracy.py -s -m slow  # golden-set regression

## Retrain

1. Put the Kaggle sets under `data/train/` and `data/test/` (see `training/train_probe.py`)
2. `.venv/bin/python training/train_probe.py data/train data/probe.npz`
3. Run `training/train.ipynb` for accuracy, the confusion matrix and the threshold curve
4. Re-run the golden-set regression
5. `docker compose restart skydex-vision`

Without `data/probe.npz` the service runs zero-shot and reports
`clip-vit-b-32-zeroshot-v1` as its model name. With it, `...-probe-v1`.

## The contract

`POST /v1/analyze`, multipart field `file`:

```json
{
  "outdoor_score": 0.94,
  "phenomenon_scores": {
    "CLEAR": 0.02, "CLOUDY": 0.11, "FOG": 0.04,
    "RAIN": 0.62, "SNOW": 0.01, "STORM": 0.20
  },
  "model": "clip-vit-b-32-zeroshot-v1"
}
```

`400` means the bytes are not a readable image. There is no status for
"fraudulent" — this service does not have that opinion.

## Design

`docs/superpowers/specs/2026-08-16-ai-photo-validation-design.md`
```

- [ ] **Step 6: Commit**

Ask the user first, then:

```bash
git add Dockerfile .dockerignore docker-compose.yml README.md
git commit -m "feat: container, compose stack and README"
```

---

## Done when

- `.venv/bin/pytest` is green (fast suite)
- `.venv/bin/pytest tests/test_accuracy.py -s -m slow` meets the 2% false-positive bar
- `docker compose up -d` brings up both services healthy
- `curl -F "file=@sky.jpg" http://localhost:8000/v1/analyze` returns six groups summing to 1
- `training/train.ipynb` has produced `confusion_matrix.png` and `threshold_curve.png`
- The threshold read off the curve is written into `tests/test_accuracy.py`

The SkyDex backend is untouched by every task above. Proceed to
`2026-08-16-skydex-ai-validation-integration.md` only once this list is complete —
that plan's first task assumes a running service to point at.
