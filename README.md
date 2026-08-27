# SkyDex

A gamified camera app for capturing meteorological events and sharing them with
friends. Photograph a storm, a rainbow, fog — the app verifies you actually did,
and it goes in your dex.

The interesting problem is that last part. Nothing stops a user photographing a
wall, or a screenshot of a storm, as long as a storm happens to be in the
forecast. SkyDex looks at the photograph.

## Screenshots

_Not yet captured — the app's capture, dex, feed and map screens belong here._

## How it fits together

    Android app  ──HTTP──▶  Spring Boot API  ──JDBC──▶  Postgres
     (Compose)                    │
                                  ├──HTTP──▶  skydex-vision  (FastAPI + CLIP)
                                  └──HTTP──▶  Open-Meteo     (weather record)

| Repository | What it is |
|---|---|
| [SkyDex-backend](https://github.com/GuiBecko/SkyDex-backend) | Kotlin, Spring Boot 3.2. Auth, captures, dex, friends, feed. |
| [SkyDex-android](https://github.com/GuiBecko/SkyDex-android) | Jetpack Compose client. |
| [SkyDex-vision](https://github.com/GuiBecko/SkyDex-vision) | FastAPI service scoring a photograph against CLIP. Returns numbers only. |

## Validating a capture

A capture is a photograph, coordinates, a timestamp and a claimed phenomenon.
Two stages must pass before anything is stored.

**Stage 1 — is this outdoors?** skydex-vision scores the photograph against
CLIP prompts for sky versus a catalogue of frauds: indoor rooms, faces, blank
walls, screens, screenshots. Below 0.60 the upload is refused with 422 and
nothing is written — no row, no file.

That threshold is measured, not guessed. On the project's 30-image golden set
the worst fraud scores 0.0547 and the weakest genuine sky 0.7831, so 0.60 sits
in the middle of a 0.73-wide empty band with no example on either side of it.

**Stage 2 — does the photograph agree with the weather?** The claimed phenomenon
is checked against Open-Meteo's hourly record for those coordinates, and against
which of six visual groups the photograph resembles. A contradiction only blocks
when the model is confident both ways: the expected group scored below 0.10 *and*
some other group scored above 0.70.

Those two numbers are **not** measured. There is no labelled set of
photo-versus-weather contradiction pairs to derive them from, so they are
deliberately conservative starting values, chosen to let an honest upload
through rather than block one. The repository says so where they are defined
rather than presenting them as tuned.

## Design decisions worth reading

- **The vision service returns numbers, never verdicts.** Every threshold and
  every decision lives in the Kotlin backend. The Python service cannot refuse
  an upload; it can only report what it saw.
- **The phenomenon head is trained; the outdoor head is not.** The fraud
  catalogue is a moving target that prompts express better than a fixed
  training set, so the outdoor score stays zero-shot. Only the six-way weather
  classification uses a trained linear probe.
- **A vision outage costs nothing.** The call happens before any row exists and
  before any photo is written, so a failure degrades to "not validated" instead
  of stranding half a capture.
- **Photo URLs are stored relative.** The public base URL is composed in on the
  way out, so re-pointing the deployment never orphans a stored row.
- **`open-in-view` is off.** The entities reference each other by plain UUID
  columns rather than JPA associations, so there are no lazy loads for it to
  strand — and leaving it on would have made dev diverge from the test
  environment.

Longer write-ups are in [`docs/`](docs/), including the design document for the
photo-validation work and the plans the three services were built from.

## Running the server side

Requires Docker. The compose file expects the repositories checked out as
siblings:

    git clone https://github.com/GuiBecko/skydex
    git clone https://github.com/GuiBecko/SkyDex-backend
    git clone https://github.com/GuiBecko/SkyDex-vision
    cd skydex
    cp .env.example .env        # then set TOKEN_JWT_SECRET
    docker compose up

**The first build takes several minutes.** The vision image bakes ~350MB of
CLIP weights in, so that no container start waits on a download. It is not
stuck.

Then `curl http://localhost:3002/actuator/health` should answer
`{"status":"UP"}`.

For the app, see
[SkyDex-android](https://github.com/GuiBecko/SkyDex-android) — it needs Android
Studio, and one line in `local.properties`.

## Licence

MIT — see `LICENSE`. Two exceptions live in the vision repository: the
golden-set photographs are third-party works under their own licences, and the
trained probe derives from two Kaggle datasets. Both are documented there.
