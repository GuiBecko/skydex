# SkyDex Open-Source Switch — Design

Date: 2026-08-27
Status: awaiting user review; all Phase 0 prerequisites resolved 2026-08-27

## Problem

SkyDex is three private repositories with no licence, two of them with no README, and a git
history that carries three signed JWTs, three private LAN addresses, a corporate email address
and a developer's home directory path. It also redistributes thirty third-party photographs
without naming their authors or licences.

The goal is to publish it as an open-source portfolio piece: a reader arrives, understands within
a minute that this is a competently built system, and can run the server side with one command if
they want to check. Nothing here is aimed at attracting outside contributors, at self-hosting by
third parties, or at operating a community.

That goal is narrow, and it is what keeps this spec small. What it does *not* remove is the legal
and security work: a public repository redistributing CC-BY images still owes attribution, and a
public repository containing a signature made by a live key is still a security exposure,
regardless of who the audience is.

## Goal and non-goals

**Goal.** Four public repositories — three existing ones plus an umbrella — that are legally
clean, free of leaked material, documented for a first-time reader, and runnable server-side via
`docker compose up`.

**Non-goals.** Explicitly out of scope, and each one is a decision rather than an omission:

- `CONTRIBUTING.md`, code of conduct, issue and PR templates. No outside contribution is sought.
- Published Docker images, deployment guides, versioned migrations. Nobody else is expected to
  operate an instance.
- Internationalisation. Every UI string is a Portuguese literal inside Compose and `strings.xml`
  is empty. That is a real limitation, and it stays: the app is Brazilian, the reader is looking
  at architecture, and extracting ~200 strings buys nothing for this goal.
- A signed APK release, and any publicly hosted instance. Both were considered and rejected — the
  APK needs a keystore, a real `applicationId` and a public backend; a hosted instance needs paid
  hosting for a service that holds 350MB of CLIP in memory, plus moderation of stranger-submitted
  photographs and LGPD exposure over their coordinates.
- Any change to application logic or features. Nothing about how a capture is validated, scored
  or displayed changes. What does change is configuration, packaging, documentation, history and
  identity: Phase 4 deletes one dependency that is never used, and Phase 5 renames the package,
  repoints the API-URL fallback and corrects the OSM tile referer. Those alter build identity and
  one outgoing HTTP header, not behaviour a user of the app would notice.

## Decisions

Each of these was chosen deliberately; the rejected alternative is recorded because the reasoning
is what makes it reviewable.

**Four repositories, not a monorepo.** The three existing repositories go public as they are, and
a new `GuiBecko/skydex` becomes the entry point that explains the system as a whole. A monorepo
would read better as a single showcase, but merging three histories costs more than the umbrella
README does, and the umbrella solves the actual problem — that a reader landing on the Android app
has no way to discover the vision service exists.

**MIT for all four.** Permissive, twenty lines, universally recognised, and compatible with
OpenCLIP (MIT) and the LAION-2B weights. Apache-2.0's `NOTICE` requirement is bureaucracy nobody
will maintain on a showcase project; AGPL-3.0 is disproportionate for a weather-photo game and
deters corporate readers.

**Sanitise in private, then flip visibility.** All work happens in the existing private
repositories, is verified against a clean clone, and only then becomes public. Rejected: starting
fresh with squashed history — the 122 commits are an asset here, because their messages record
reasoning ("refit the probe with balanced class weights", "store photo urls relative so a row can
never name a stale host"), and there is no secret in history that survives Phase 1. Also rejected:
publishing the umbrella first — a README pointing at three 404s is worse than nothing.

**Rewrite history rather than only rotating the secret.** The repositories have zero forks, zero
open pull requests and no collaborators, which makes rewriting cheap now and impossible after
publication. Commit messages, dates and topology are preserved; only SHAs change.

**Version `data/probe.npz`.** 25KB, and it is what makes the service behave as measured. Without
it a cloner gets the zero-shot fallback, which `data/golden/README.md` records as having once
refused an honest photograph of a clear blue sky. Publishing the worse configuration silently
would misrepresent the project.

**Publish design documents, remove agent instructions.** The four documents under
`docs/superpowers/` are genuine technical documentation and are the artefact that separates this
from a code dump. The `CLAUDE.md` files are operational instructions containing local paths and
account directives; they say nothing to a reader and are removed.

**Rename the Android package.** `com.example.skydex` is the Android Studio placeholder and a
technical reviewer reads it as "template project" instantly. New identity:
`io.github.guibecko.skydex` — reverse-domain convention for a GitHub-hosted project without its
own domain.

**Umbrella repository named `skydex`, lowercase.** Inconsistent with the `SkyDex-` prefix of the
others, and chosen anyway because lowercase is the GitHub norm for a project's canonical
repository. This is one line to change if the user prefers `SkyDex`.

## Security audit — findings of record

A full scan ran over the working tree and the complete history of all three repositories: 122
commits, every blob, twenty families of credential pattern, Shannon-entropy analysis over every
tracked file, EXIF inspection of every committed image, and commit-metadata review.

The scan is reproducible. It lives at `Atividades/switch-open-source/scan-secrets.sh`, takes a
repository path, and documents its own known-benign baseline (test fixtures such as
`"senha-nova"` and `"secret123"`, and the self-describing
`integration-test-secret-do-not-use-in-production`) in its header. Phase 1 step 9 and Phase 6
step 1 are that script re-run against the same three repositories; anything outside the
documented baseline blocks publication.

### Confirmed exposures

**1 — Three signed JWTs in `SkyDex---frontend` history. Severity: high.**

Present in four commits between 2026-07-28 and 2026-08-02, in `HomePage.kt`, `NearEvents.kt` and
`Registers.kt`. All three files were later refactored away, so nothing appears in `HEAD`; the
tokens exist only in history. Payloads carry `sub: guilherme2becker@gmail.com` and
`sub: email2.fake@gmail.com`.

Two facts establish the severity. All three tokens are expired (2026-07-28, 2026-08-01,
2026-08-02), so they cannot be replayed. But the HMAC was recomputed against the
`TOKEN_JWT_SECRET` currently in `SkyDex-backend/.env`, and **both tested signatures verify** — the
live key is the key that signed them. Publishing therefore hands any reader a known-plaintext
and valid-signature pair, which is offline brute-force material with no rate limit and no audit
trail. A recovered secret forges new, unexpired tokens for any subject, because `SecurityFilter`
validates only signature and `exp`. The secret is 28 characters but matches `[A-Za-z0-9 _-]+`; if
it is a memorable phrase rather than random, its real entropy is far below what its length
suggests.

**2 — Corporate email address. Severity: medium.**

In content at `ProfileScreen.kt:644` (a `@Preview` fixture, present in `HEAD`), and in the commit
metadata of 36 commits authored as `Becker <<work-address>>` — 4 in the backend, 32 in
the frontend. The remaining 86 commits use `guilherme2becker@gmail.com`. This contradicts the
workspace's own `CLAUDE.md` directive to commit as the GuiBecko account, so it was a stale
`git config`.

Publishing a personal project with a third of its commits attributed to an employer's domain
creates a visible link between the two, which is why it was raised. The user has since confirmed
there is no intellectual-property clause of that kind and that the work was done outside working
hours, so the exposure is cosmetic. Phase 1 rewrites the authorship regardless: the tool is
already open, and a public history that does not mention a former employer's domain is simply
tidier.

**3 — Private LAN addresses and a home directory path. Severity: low.**

| Value | Location | In `HEAD`? |
|---|---|---|
| `<lan-address>` | `SkyDex-backend`, `WeatherEventDtosTest.kt:33,35` (test fixture) | yes |
| `<lan-address>` | `SkyDex---frontend`, `app/build.gradle.kts:43` (live fallback) | yes |
| `<lan-address>` | `SkyDex---frontend`, `app/build.gradle.kts:37` (comment only) | yes |
| `<workspace>/...` (63×) | `skydex-vision`, `training/train.ipynb` (output cells) | yes |

All three addresses are RFC1918 — private and not routable from the internet. They grant no
access; what they disclose is the shape of a home network. The directory path discloses a username
and layout, captured from venv warnings in notebook execution output.

### Verified absent

These categories were searched for and not found. Recording them matters as much as recording the
findings, because it is what makes the audit falsifiable.

- **No API keys of any kind** — AWS access keys and secrets, GCP service-account JSON, Google API
  keys, GitHub PATs, OpenAI, Anthropic, Stripe, Slack, Twilio, SendGrid. No pattern matched any
  blob in any of the three histories. Consistent with the architecture: Open-Meteo and OSM tiles
  need no key.
- **No private keys, certificates or keystores.**
- **No hardcoded database password.** `application.properties` uses only `${VAR:default}`, whose
  defaults are local-development `postgres/postgres`.
- **`.env` and `local.properties` were never committed** — confirmed with `--diff-filter=A` across
  all 122 commits, not merely by inspecting the current tree.
- **No `.sql` file tracked anywhere.** The dump `skydex-backup-20260816-202123.sql` (6 bcrypt
  hashes, one real email address) sits loose in the workspace root, which is not a git repository.
  It cannot be published from where it is; it must not be moved inside a repository.
- **`.idea/` is tracked** in both Kotlin repositories (10 and 13 files), which is noise rather
  than leakage: `dataSources.xml` holds only `jdbc:postgresql://localhost:5432/skydex`, with no
  credential and no remote host. The two files that would carry passwords and local paths —
  `dataSources.local.xml` and `workspace.xml` — are not tracked.
- **Emails in file content** are otherwise synthetic fixtures (`alice@`, `bob@`, `pilot@`,
  `admin@skydex.com`).
- **Entropy analysis** surfaced nothing real: long Java/Kotlin identifiers, JPEG Huffman tables,
  and `integration-test-secret-do-not-use-in-production`, which documents itself.
- **EXIF on all 30 committed photographs**: no GPS, no camera make or model, no author, including
  the four screenshots captured on this machine. The Pillow normalisation documented in
  `data/golden/README.md` stripped everything.

## Target topology

```
GuiBecko/skydex                 NEW — the entry point
├── README.md                   what it is, screenshots, architecture, how to run
├── docker-compose.yml          postgres + backend + vision
├── .env.example
├── LICENSE                     MIT
└── docs/
    ├── 2026-08-07-skydex-mvp.md
    ├── 2026-08-16-skydex-vision-service.md
    ├── 2026-08-16-skydex-ai-validation-integration.md
    ├── 2026-08-16-ai-photo-validation-design.md
    └── 2026-08-27-skydex-open-source-design.md   (this document)

GuiBecko/SkyDex-backend         Kotlin, Spring Boot 3.2, 26 test files, Testcontainers
GuiBecko/SkyDex-vision          FastAPI, OpenCLIP ViT-B/32, 45 fast tests + 2 golden-set
GuiBecko/SkyDex-android         renamed from SkyDex---frontend; Jetpack Compose, 100 .kt files
```

Each repository keeps a README about itself — what it does, how to run it, how to test it — and
links the umbrella on its first line. Architecture is explained in exactly one place.

`SkyDex---frontend` is renamed to `SkyDex-android`: the triple hyphen is a typing accident visible
in the URL, and "android" describes the artefact better than "frontend". GitHub preserves a
redirect from the old name.

## Phase 0 — Prerequisites

Status as of 2026-08-27, after the user's decisions.

1. **`git-filter-repo` — DONE.** Version 2.47.0, installed via `pipx` (the machine has no `pip`)
   at `~/.local/bin/git-filter-repo`, and resolving as `git filter-repo`. The native
   `filter-branch` fallback is slow and error-prone and will not be used.
2. **Pending branch — merge it, do not discard.** `feat/friends-unfriend-and-invite-badge` has 2
   commits unique to it in the backend and 3 in the frontend, unmerged into `master`. The user
   waved the decision off; merging is taken as the reading, because those commits carry real
   features (either party can unfriend, and the invite badge counts) and discarding them would
   silently delete shipped work, while merging is reversible. This becomes Phase 1 step 3 rather
   than a blocking prerequisite. **If the intent was to discard, say so before Phase 1 runs.**
3. **Screenshots — deferred by the user, and no longer blocking.** There is no emulator and no
   `adb` on this machine, so capturing them needs the user running the app against a live backend.
   The umbrella README therefore ships with its screenshot section written but empty, carrying one
   line naming what belongs there. Consequence, stated plainly: the repositories go public without
   app imagery, which is the single largest weakness of the result for a portfolio reader, since a
   visual product judged only by its source loses most of its first impression. Adding them later
   is one commit to the umbrella and changes nothing else.
4. **Employer IP policy — RESOLVED, 2026-08-27.** The user confirms there is no such clause and
   that the work was done outside working hours. The 36 commits are rewritten to the personal
   address as planned, which now settles a cosmetic inconsistency rather than a risk.

Additionally, every `git push --force` requires explicit approval, once per repository. History is
never force-pushed on the implementer's judgment.

## Phase 1 — Security remediation

Ordered; step 6 depends on 1 through 5.

1. **Rotate `TOKEN_JWT_SECRET`** in `SkyDex-backend/.env` — the only `.env` among the three
   repositories — to 64 random characters, removing the entropy question entirely. Required
   whether or not the code is ever published.
2. **Confirm `git-filter-repo` is installed** (Phase 0, item 1 — already done). Every step below needs it.
3. **Merge `feat/friends-unfriend-and-invite-badge` into `master`** in both Kotlin repositories,
   settling the branch topology before any rewrite touches it. Rewriting across an unsettled
   topology invites conflicts.
4. **Mirror-clone all three repositories** to `<backup-location>/`, outside the workspace.
   This is the safety net for the rewrite, not a formality.
5. **Rehearse the rewrite on the mirrors**, and run step 9's scanner against the rehearsal. A
   rewrite verified only after the real repositories have been touched has no way back.
6. **Rewrite history** with `git-filter-repo --replace-text`, substituting across all blobs:
   - the three JWTs → `Bearer <token>`
   - `<lan-address>`, `<lan-address>`, `<lan-address>` → `<host>`
   - `<work-address>` → `pilot@skydex.com`
   - `<workspace>` → `/workspace`
7. **Rewrite commit authorship** for the 36 corporate-email commits, unifying on
   `guilherme2becker@gmail.com`. The tool is already open; doing it separately costs a second
   rewrite.
8. **Delete `origin/feat/ai-photo-validation`** in both Kotlin repositories. It is already merged
   into `master`, so deletion is lossless, and a stale branch would still carry the old blobs.
9. **Re-run the audit scanner** over the rewritten histories. This is the step that proves the
   rewrite worked; without it the claim is an assertion.
10. **Run all three test suites.** The rewrite edits test files —
   `WeatherEventDtosTest.kt` asserts on the literal `<lan-address>` — so it can break tests.

## Phase 2 — Legal layer

1. **`LICENSE` (MIT)** in all four repositories, copyright in the user's name, year 2026.
2. **Attribute the 30 Commons photographs.** Extend `data/golden/manifest.csv` with `author` and
   `licence` columns, sourced from each file's Commons page, and add
   `data/golden/LICENSE-IMAGES.md` stating that the images are not MIT, that each follows the
   licence on its row, and — explicitly — that a CC-BY-SA image's share-alike obligation covers
   that image and its derivatives, not the Kotlin or Python source. Without that last sentence the
   notice reads as though the repository were contaminated.
3. **Document the probe's provenance** in `training/README.md`: both Kaggle datasets
   (`jehanbhathena/weather-dataset`, `pratik2901/multiclass-weather-dataset`), their licences, and
   the training date. Then commit `data/probe.npz`.
4. **`THIRD-PARTY.md` in the vision repository.** OpenCLIP is MIT and the `laion2b_s34b_b79k`
   weights are downloaded at build time rather than redistributed in the repository — but the
   Dockerfile bakes them into the image, so the resulting image contains LAION weights and the
   notice should say so.

**Two verification gates with defined failure paths.** Both licence checks happen *before* the
corresponding commit, and neither failure is resolved unilaterally:

- If any Commons photograph turns out to carry a licence incompatible with redistribution, it is
  replaced with an equivalent image and both golden-set suites are re-run — which changes the
  measured numbers recorded in `data/golden/README.md`. The user is consulted before any
  substitution.
- If either Kaggle dataset does not permit redistribution of derived work — `pratik2901`
  declaring no licence at all is the likeliest failure — `data/probe.npz` is not committed, and the
  user is consulted rather than the decision being reversed silently.

## Phase 3 — Documentation

Four READMEs plus the document migration.

- **`skydex`** — what the project is, in two paragraphs; app screenshots; an architecture diagram
  (Compose → Spring Boot → Postgres, with the branch to FastAPI/CLIP); one-command startup; links
  to the three repositories. Closes with the technical decisions worth reading: why photo
  validation has two stages, how the thresholds were derived and which are measured versus
  judged, why `open-in-view` is off.
- **`SkyDex-backend`** — endpoints, how to run, how to test. The 26 Testcontainers-backed test
  files are a strength and get named as one.
- **`SkyDex-vision`** — already good. Gains a licence note and an umbrella link.
- **`SkyDex-android`** — how to build, what to put in `local.properties`, screenshots.
- **Migrate** the four documents from `docs/superpowers/` into the umbrella's `docs/`, names
  preserved, plus this spec.

## Phase 4 — Runnability

The backend has neither a Dockerfile nor a compose file, so this is new work.

1. **`Dockerfile` in the backend** — multi-stage, Gradle build to a JRE 17 slim runtime.
   `bootJar` already works, so this is mechanical.
2. **`docker-compose.yml` in the umbrella** — `postgres:16`, backend, vision. Plain `postgres:16`
   suffices: `hibernate-spatial` is declared in `build.gradle.kts` but never used — there is no
   spatial import anywhere in `src/`, coordinates are plain `Double` columns, and distance is
   haversine in Kotlin (`TravelPlausibility`).
3. **Remove the `hibernate-spatial` dependency.** Dead, and it would send a reader looking for
   PostGIS that does not exist. This is the one code change in the spec, and it is in service of
   the compose file having a defensible answer to "which Postgres image".
4. **`.env.example`** with all ten variables the code reads — the six in the current `.env` plus
   `SERVER_PORT`, `VISION_OUTDOOR_MIN`, `VISION_EXPECTED_SCORE_MAX` and `VISION_TOP_SCORE_MIN`,
   which exist today only as defaults in `application.properties`. Safe values, one comment per
   line.
5. **Document the first-run cost.** The vision image bakes ~350MB of CLIP weights at build time. A
   `docker compose up` that appears frozen for five minutes is worse than one that warns.

The vision repository's existing `docker-compose.yml` is left in place; it still serves isolated
development of that service.

## Phase 5 — Code hygiene

1. **Rename the package** to `io.github.guibecko.skydex` — 100 `.kt` files, the package
   directories, `build.gradle.kts`, the manifest and test imports. Isolated commit, verified by the
   suite.
2. **Replace the API URL fallback** in `app/build.gradle.kts` with `http://10.0.2.2:3002`, the
   Android emulator's alias for the host, which works for anyone without editing anything. The
   existing comment is good and is rewritten to explain that, rather than narrating a subnet
   change.
3. **Fix `TILE_REFERER`.** It currently sends `https://skydex.app/` — a domain the author does not
   own — to `tile.openstreetmap.org`. The OSM tile usage policy requires valid identification. It
   becomes the repository URL, which is honest identification, and the README credits OSM under
   ODbL.
4. **Remove `CLAUDE.md`** from both repositories and add `CLAUDE.md`, `.claude/` and
   `.superpowers/` to each `.gitignore`. Also untrack `.idea/` — 23 files that are noise on a
   showcase repository.
5. **Clean `training/train.ipynb`.** Outputs with value stay (accuracy, confusion matrix,
   threshold curve); the venv warning cells carrying `<workspace>` go.

## Phase 6 — Verification and the switch

Nothing becomes public before all four pass, in order:

1. **Clean secret scan** across all three rewritten histories, using the same scanner as the
   audit.
2. **Green suites**: backend (26 files, Testcontainers), vision (45 fast plus both slow golden-set
   regressions), Android (30 test files).
3. **Clean-clone test**: clone all three into `/tmp`, follow the README from scratch, and prove
   `docker compose up` starts and answers. This is the only test that catches a lying README.
4. **All 30 Commons attribution rows filled.**

Then, and only then: `gh repo edit --visibility public` on the three, the rename of
`SkyDex---frontend` to `SkyDex-android`, and creation of the umbrella repository.

## Risks

- **The rewrite breaks tests.** `WeatherEventDtosTest.kt` asserts on a literal that the rewrite
  replaces. Mitigated by Phase 1 step 10 running before any push, and by the mirror backups.
- **Force-push loses work.** Mitigated by mirror clones taken before any ref is touched, by
  resolving the pending branch first, and by per-repository explicit approval.
- **A Commons licence forces an image swap**, which moves the measured margins the golden-set
  README quotes. Mitigated by consulting the user rather than substituting silently, and by
  re-running both golden suites so the README's numbers stay true to the set.
- **A Kaggle dataset forbids redistributing derived weights**, leaving the vision service in its
  zero-shot configuration. Mitigated by checking before committing the probe, and by the README
  stating the configuration honestly if that happens.
- **The clean-clone test fails late**, after visibility has been considered ready. Mitigated by
  ordering it before the switch rather than after.

## What this spec does not resolve

Named rather than hidden, because a reviewer should see them:

- Nothing about authorship or the pending branch: both were resolved on 2026-08-27 and are
  recorded in Phase 0.
- The umbrella repository's capitalisation (`skydex` versus `SkyDex`). Decided as lowercase; one
  line to change.
- The exact package name. Decided as `io.github.guibecko.skydex`; the user may prefer another.
