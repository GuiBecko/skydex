# SkyDex AI Validation Integration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop asking the user which phenomenon they photographed. Let Open-Meteo decide what the weather was, and let a vision model decide whether the photo is consistent with it.

**Architecture:** The vision model runs at photo-upload time and its scores are cached on the `uploaded_photos` row. At capture time Open-Meteo supplies the phenomenon, and a pure Kotlin policy service compares that phenomenon's visual group against the cached scores. Both upstreams answer 503 when unavailable, which is cheap because neither failure reaches the point where a photo is spent.

**Tech Stack:** Kotlin 1.9, Spring Boot 3.2.4, JPA/Postgres, Testcontainers, JUnit 5, Mockito (backend); Jetpack Compose, Retrofit, kotlinx-coroutines-test (Android).

**Spec:** `docs/superpowers/specs/2026-08-16-ai-photo-validation-design.md`

**Prerequisite:** `docs/superpowers/plans/2026-08-16-skydex-vision-service.md` must be complete and `docker compose up -d` must have `skydex-vision` healthy on `http://localhost:8000`.

## Global Constraints

- Code and comments in **English**, despite the surrounding user-facing copy being Portuguese.
- Never commit or push without asking the user first.
- Backend tests: `JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test` from `SkyDex-backend/`. Needs Docker (Testcontainers).
- Android tests: `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest` from `SkyDex---frontend/`.
- Android fast compile check: `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin`.
- The six visual groups, exactly: `CLEAR`, `CLOUDY`, `FOG`, `RAIN`, `SNOW`, `STORM`.
- Starting thresholds — replace with the values `training/train.ipynb` produced:
  - `skydex.vision.outdoor-min=0.60`
  - `skydex.vision.expected-score-max=0.10`
  - `skydex.vision.top-score-min=0.70`
- **CLIP never runs in the backend test suite.** `VisionClient` is mocked everywhere.
- New database columns are all nullable, so `spring.jpa.hibernate.ddl-auto=update` handles them. No manual migration.
- The `phenomenon` field on `CreateWeatherEventRequest` becomes **accepted and ignored**, never removed. Removing it would 400 every capture from an app already installed.

---

## File Structure

**Backend — create:**

| File | Responsibility |
|---|---|
| `dto/VisionAnalysis.kt` | The wire shape of the vision service's response |
| `services/VisionClient.kt` | HTTP access to skydex-vision. Returns null on any failure |
| `domain/VisualGroup.kt` | The six groups and the `Phenomenon` to group mapping |
| `domain/UnconfirmedReason.kt` | Why a capture was not confirmed |
| `services/PhotoAuthenticityService.kt` | The contradiction matrix. Pure, no I/O |
| `services/PhotoAnalysisService.kt` | Runs the model at upload and persists the scores |

**Backend — modify:**

| File | Change |
|---|---|
| `models/UploadedPhoto.kt` | + four vision columns |
| `models/WeatherEvent.kt` | + `unconfirmedReason` |
| `errors/DomainExceptions.kt` | + `ServiceUnavailableException`, `UnprocessableContentException` |
| `controllers/GlobalExceptionHandler.kt` | + handlers mapping those two to 503 and 422 |
| `controllers/PhotoController.kt` | Analyse before storing; 422 and 503 |
| `dto/OpenMeteoResponse.kt` | + `is_day` on `HourlyData` |
| `services/OpenMeteoClient.kt` | Request `is_day` |
| `services/CaptureValidationService.kt` | Open-Meteo decides the phenomenon; throws on outage |
| `controllers/WeatherEventController.kt` | Drop the claim, pass the photo's scores |
| `dto/WeatherEventDtos.kt` | `phenomenon` optional in; `unconfirmedReason` out |
| `src/main/resources/application.properties` | + vision config block |

**Android — modify:**

| File | Change |
|---|---|
| `data/remote/dto/WeatherEventDto.kt` | Drop `phenomenon` from the request, add `unconfirmedReason` to the response |
| `ui/capture/CaptureScreen.kt` | Delete the species chips |
| `ui/capture/CaptureViewModel.kt` | Delete the phenomenon state; upload eagerly; stop deleting unconfirmed captures |
| `ui/capture/CaptureGateway.kt` | Drop `delete` |
| `data/repository/CaptureRepository.kt` | Drop the `CaptureGateway` override on `delete` |
| `ui/common/ErrorPresenter.kt` | + 422 and 503 branches |
| `ui/components/CaptureRewardOverlay.kt` | + reason copy on the unconfirmed branch |
| `ui/captures/MyCapturesScreen.kt` | + a not-confirmed badge on the row |

---

### Task 1: VisionClient

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/VisionAnalysis.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/config/VisionClientConfig.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/VisionClient.kt`
- Modify: `SkyDex-backend/src/main/resources/application.properties`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/VisionClientTest.kt`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `com.skydex.api.dto.VisionAnalysis(outdoorScore: Double, phenomenonScores: Map<String, Double>, model: String)`
  - `com.skydex.api.config.VisionClientConfig.visionRestClient(baseUrl: String): RestClient` — the `@Bean("visionRestClient")` carrying the timeouts
  - `com.skydex.api.services.VisionClient(restClient: RestClient)` with `.analyze(bytes: ByteArray, filename: String): VisionAnalysis?`

The `RestClient` is built by a `@Bean` rather than inside `VisionClient` for a reason that only shows up in the test: `MockRestServiceServer.bindTo(builder)` installs its mock request factory on the builder **immediately**, so a `.requestFactory(...)` call made afterwards inside the client would silently override it and the test would hit a real `localhost:8000`. Building the client outside and injecting it finished keeps the timeouts real in production and the mock intact in tests. `OpenMeteoClient` gets away with the inline shape only because its own test never mocks at that layer.

- [ ] **Step 1: Write the failing test**

`src/test/kotlin/com/skydex/api/service/VisionClientTest.kt`:
```kotlin
package com.skydex.api.service

import com.skydex.api.services.VisionClient
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Test
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.test.web.client.MockRestServiceServer
import org.springframework.test.web.client.match.MockRestRequestMatchers.method
import org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo
import org.springframework.test.web.client.response.MockRestResponseCreators.withServerError
import org.springframework.test.web.client.response.MockRestResponseCreators.withStatus
import org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess
import org.springframework.http.HttpMethod
import org.springframework.web.client.RestClient

class VisionClientTest {

    // Order matters. `bindTo` installs the mock request factory on the builder as soon as it is
    // called, so the client must be built AFTER it — and the client must not set a request factory
    // of its own, which is why the timeouts live in VisionClientConfig instead.
    private val builder = RestClient.builder().baseUrl(BASE_URL)
    private val server: MockRestServiceServer = MockRestServiceServer.bindTo(builder).build()
    private val client = VisionClient(builder.build())

    private val jpeg = byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte(), 0xD9.toByte())

    @Test
    fun `parses a successful analysis`() {
        server.expect(requestTo("$BASE_URL/v1/analyze"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(
                withSuccess(
                    """
                    {
                      "outdoor_score": 0.94,
                      "phenomenon_scores": {
                        "CLEAR": 0.02, "CLOUDY": 0.11, "FOG": 0.04,
                        "RAIN": 0.62, "SNOW": 0.01, "STORM": 0.20
                      },
                      "model": "clip-vit-b-32-zeroshot-v1"
                    }
                    """.trimIndent(),
                    MediaType.APPLICATION_JSON
                )
            )

        val analysis = client.analyze(jpeg, "storm.jpg")

        assertEquals(0.94, analysis!!.outdoorScore)
        assertEquals(0.62, analysis.phenomenonScores["RAIN"])
        assertEquals("clip-vit-b-32-zeroshot-v1", analysis.model)
        server.verify()
    }

    @Test
    fun `returns null on a server error rather than throwing`() {
        server.expect(requestTo("$BASE_URL/v1/analyze")).andRespond(withServerError())

        // Null, not an exception: the caller decides what an unavailable model means,
        // and every other upstream in this codebase (OpenMeteoClient) behaves the same way.
        assertNull(client.analyze(jpeg, "storm.jpg"))
    }

    @Test
    fun `returns null when the service rejects the image`() {
        server.expect(requestTo("$BASE_URL/v1/analyze"))
            .andRespond(withStatus(HttpStatus.BAD_REQUEST))

        assertNull(client.analyze(jpeg, "storm.jpg"))
    }

    @Test
    fun `returns null on a malformed body`() {
        server.expect(requestTo("$BASE_URL/v1/analyze"))
            .andRespond(withSuccess("not json at all", MediaType.APPLICATION_JSON))

        assertNull(client.analyze(jpeg, "storm.jpg"))
    }

    private companion object {
        const val BASE_URL = "http://vision.test"
    }
}
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
cd ~/Documentos/workspace-becker/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.VisionClientTest"
```

Expected: compilation failure — `Unresolved reference: VisionClient`.

- [ ] **Step 3: Write the DTO**

`src/main/kotlin/com/skydex/api/dto/VisionAnalysis.kt`:
```kotlin
package com.skydex.api.dto

import com.fasterxml.jackson.annotation.JsonIgnoreProperties
import com.fasterxml.jackson.annotation.JsonProperty

/**
 * What `skydex-vision` says about one photograph.
 *
 * Evidence, not a verdict. The service has no notion of a `Phenomenon`, a threshold or a
 * `ValidationStatus` — deciding what these numbers mean is [com.skydex.api.services
 * .PhotoAuthenticityService]'s job, and keeping the split there is what lets a threshold change
 * ship without touching Python.
 *
 * [phenomenonScores] is keyed by [com.skydex.api.domain.VisualGroup] name and sums to 1. It is
 * deliberately a plain `Map<String, Double>` rather than a map keyed by the enum: an unknown key
 * from a newer model must not fail deserialisation of an otherwise usable response.
 */
@JsonIgnoreProperties(ignoreUnknown = true)
data class VisionAnalysis(
    @JsonProperty("outdoor_score") val outdoorScore: Double,
    @JsonProperty("phenomenon_scores") val phenomenonScores: Map<String, Double>,
    val model: String
)
```

- [ ] **Step 4: Write the RestClient bean**

`src/main/kotlin/com/skydex/api/config/VisionClientConfig.kt`:
```kotlin
package com.skydex.api.config

import org.springframework.beans.factory.annotation.Value
import org.springframework.boot.web.client.ClientHttpRequestFactories
import org.springframework.boot.web.client.ClientHttpRequestFactorySettings
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.client.RestClient
import java.time.Duration

/**
 * The HTTP client `VisionClient` talks to `skydex-vision` through.
 *
 * ## Why this is a bean and not built inside the client
 *
 * `OpenMeteoClient` builds its own `RestClient` inline, and copying that shape here produces a test
 * that quietly does the wrong thing: `MockRestServiceServer.bindTo(builder)` installs its mock
 * request factory on the builder the moment it is called, so a `.requestFactory(...)` invoked
 * afterwards inside the client overrides the mock and the "unit" test reaches for a real
 * localhost:8000. Building the client here and injecting it finished keeps the timeouts real in
 * production and the mock intact in tests, with no test-only seam in the production class.
 *
 * ## Why explicit timeouts at all
 *
 * `RestClient.create(...)` inherits the JDK default of **no read timeout**. That default is the
 * dangerous one: this call runs on a Tomcat request thread, so an upstream that accepts a
 * connection and then stops answering parks one thread per upload for as long as the TCP connection
 * survives — and takes down every endpoint, including the ones that never touch this service, once
 * the pool is exhausted.
 */
@Configuration
class VisionClientConfig {

    @Bean("visionRestClient")
    fun visionRestClient(
        @Value("\${skydex.vision.base-url:http://localhost:8000}") baseUrl: String
    ): RestClient = RestClient.builder()
        .baseUrl(baseUrl)
        .requestFactory(
            ClientHttpRequestFactories.get(
                ClientHttpRequestFactorySettings.DEFAULTS
                    .withConnectTimeout(CONNECT_TIMEOUT)
                    .withReadTimeout(READ_TIMEOUT)
            )
        )
        .build()

    private companion object {
        val CONNECT_TIMEOUT: Duration = Duration.ofSeconds(3)

        /**
         * Ten seconds, against Open-Meteo's five. A warm CLIP forward pass on CPU is ~250ms, but
         * the first request after a container restart pays for lazily-built graph state and can
         * take several seconds. Failing those would 503 every upload in the minute after a deploy.
         */
        val READ_TIMEOUT: Duration = Duration.ofSeconds(10)
    }
}
```

- [ ] **Step 4b: Write the client**

`src/main/kotlin/com/skydex/api/services/VisionClient.kt`:
```kotlin
package com.skydex.api.services

import com.skydex.api.dto.VisionAnalysis
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.core.io.ByteArrayResource
import org.springframework.http.MediaType
import org.springframework.stereotype.Service
import org.springframework.util.LinkedMultiValueMap
import org.springframework.web.client.RestClient

/**
 * Thin HTTP access to `skydex-vision`. Interpretation of the scores lives in
 * [PhotoAuthenticityService]; nothing here knows what a phenomenon is.
 *
 * The failure contract is deliberately [OpenMeteoClient]'s: **every** failure comes back as null
 * rather than as an exception. The caller ([PhotoAnalysisService]) turns a null into a 503, at a
 * point where nothing has been written and no photo has been spent.
 *
 * Timeouts live in [com.skydex.api.config.VisionClientConfig] — see its KDoc for why they are not
 * configured here.
 */
@Service
class VisionClient(
    @Qualifier("visionRestClient") private val restClient: RestClient
) {

    private val log = LoggerFactory.getLogger(javaClass)

    /**
     * Scores [bytes], or null if the service could not answer.
     *
     * [filename] is sent as the multipart part's filename only. Nothing downstream reads it — the
     * service identifies the image from its content — but a multipart file part without one is
     * rejected by some servers, so it is worth passing the real name through.
     */
    fun analyze(bytes: ByteArray, filename: String): VisionAnalysis? =
        try {
            val body = LinkedMultiValueMap<String, Any>().apply {
                add("file", object : ByteArrayResource(bytes) {
                    override fun getFilename() = filename
                })
            }

            restClient.post()
                .uri("/v1/analyze")
                .contentType(MediaType.MULTIPART_FORM_DATA)
                .body(body)
                .retrieve()
                .body(VisionAnalysis::class.java)
        } catch (e: Exception) {
            log.warn("Vision analysis failed for {}", filename, e)
            null
        }
}
```

- [ ] **Step 5: Add the configuration block**

Append to `src/main/resources/application.properties`:
```properties
# --- Photo vision analysis -----------------------------------------------------------------
# The skydex-vision container (see skydex-vision/docker-compose.yml). Reachable on the host as
# localhost:8000; from inside the compose network it would be http://skydex-vision:8000.
skydex.vision.base-url=${VISION_BASE_URL:http://localhost:8000}

# Stage 1: below this outdoor score, POST /api/photos answers 422 and stores nothing.
# Read off the FP/FN curve in skydex-vision/training/train.ipynb — do not tune it to make a test
# pass. False positives (an honest sky rejected) must stay under 2%.
skydex.vision.outdoor-min=${VISION_OUTDOOR_MIN:0.60}

# Stage 2: a contradiction only blocks when the expected group scored below this AND the winning
# group scored above the one below. Both bars exist so an uncertain model never blocks anyone.
skydex.vision.expected-score-max=${VISION_EXPECTED_SCORE_MAX:0.10}
skydex.vision.top-score-min=${VISION_TOP_SCORE_MIN:0.70}
```

- [ ] **Step 6: Run the test and watch it pass**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.VisionClientTest"
```

Expected: 4 passed.

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add src/main/kotlin/com/skydex/api/dto/VisionAnalysis.kt \
        src/main/kotlin/com/skydex/api/config/VisionClientConfig.kt \
        src/main/kotlin/com/skydex/api/services/VisionClient.kt \
        src/main/resources/application.properties \
        src/test/kotlin/com/skydex/api/service/VisionClientTest.kt
git commit -m "feat: HTTP client for the skydex-vision service"
```

---

### Task 2: The visual groups and the contradiction matrix

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/VisualGroup.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/PhotoAuthenticityService.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/VisualGroupTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/PhotoAuthenticityServiceTest.kt`

**Interfaces:**
- Consumes: `com.skydex.api.domain.Phenomenon`.
- Produces:
  - `VisualGroup` — enum `CLEAR, CLOUDY, FOG, RAIN, SNOW, STORM`, with `VisualGroup.of(p: Phenomenon): VisualGroup` and `VisualGroup.fromNameOrNull(name: String): VisualGroup?`
  - `PhotoAuthenticityService.contradicts(expected: Phenomenon, scores: Map<String, Double>?, isDay: Boolean): Boolean`

This is the most important test in the backend. It is pure — no network, no database, no model — so all 36 matrix cells are pinned as a table.

- [ ] **Step 1: Write the failing group test**

`src/test/kotlin/com/skydex/api/domain/VisualGroupTest.kt`:
```kotlin
package com.skydex.api.domain

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Test

class VisualGroupTest {

    @Test
    fun `every phenomenon maps to a group`() {
        // An unmapped phenomenon would throw at capture time, in production, on whatever
        // rare species somebody finally photographed. The `when` in VisualGroup.of is
        // exhaustive, so this only fails if someone adds a phenomenon and a null branch.
        Phenomenon.entries.forEach { assertNotNull(VisualGroup.of(it)) }
    }

    @Test
    fun `the rain family collapses to one group`() {
        // Drizzle, rain and rain showers are the same photograph. Asking the model to
        // separate them would be asking it to guess.
        assertEquals(VisualGroup.RAIN, VisualGroup.of(Phenomenon.DRIZZLE))
        assertEquals(VisualGroup.RAIN, VisualGroup.of(Phenomenon.RAIN))
        assertEquals(VisualGroup.RAIN, VisualGroup.of(Phenomenon.RAIN_SHOWER))
    }

    @Test
    fun `both storms collapse to one group`() {
        assertEquals(VisualGroup.STORM, VisualGroup.of(Phenomenon.THUNDERSTORM))
        assertEquals(VisualGroup.STORM, VisualGroup.of(Phenomenon.HAILSTORM))
    }

    @Test
    fun `the distinctive phenomena keep their own group`() {
        assertEquals(VisualGroup.CLEAR, VisualGroup.of(Phenomenon.CLEAR_SKY))
        assertEquals(VisualGroup.CLOUDY, VisualGroup.of(Phenomenon.CLOUDS))
        assertEquals(VisualGroup.FOG, VisualGroup.of(Phenomenon.FOG))
        assertEquals(VisualGroup.SNOW, VisualGroup.of(Phenomenon.SNOW))
    }

    @Test
    fun `an unknown group name resolves to null rather than throwing`() {
        // A newer vision model could report a group this build has never heard of. That must
        // degrade to "no opinion", never to a 500 on the capture path.
        assertNull(VisualGroup.fromNameOrNull("TORNADO"))
        assertEquals(VisualGroup.STORM, VisualGroup.fromNameOrNull("storm"))
    }
}
```

- [ ] **Step 2: Write the failing matrix test**

`src/test/kotlin/com/skydex/api/service/PhotoAuthenticityServiceTest.kt`:
```kotlin
package com.skydex.api.service

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.VisualGroup
import com.skydex.api.services.PhotoAuthenticityService
import org.junit.jupiter.api.Assertions.assertFalse
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.DynamicTest
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.TestFactory

class PhotoAuthenticityServiceTest {

    private val service = PhotoAuthenticityService(
        expectedScoreMax = 0.10,
        topScoreMin = 0.70
    )

    /**
     * Scores in which [winner] takes [share] and the rest is split evenly. Any group not named
     * scores below the 0.10 floor whenever [share] is high enough, which is what makes a
     * contradiction "confident".
     */
    private fun confident(winner: VisualGroup, share: Double = 0.80): Map<String, Double> {
        val others = VisualGroup.entries.filter { it != winner }
        val rest = (1.0 - share) / others.size
        return (others.associate { it.name to rest } + (winner.name to share))
    }

    /**
     * The full 36-cell matrix from the spec. `true` means BLOCK.
     *
     * expected \ photo   CLEAR  CLOUDY  FOG   RAIN  SNOW  STORM
     * CLEAR               ok    BLOCK  BLOCK BLOCK BLOCK BLOCK
     * CLOUDY             BLOCK   ok     ok    ok   BLOCK  ok
     * FOG                BLOCK  BLOCK   ok   BLOCK BLOCK BLOCK
     * RAIN               BLOCK   ok     ok    ok   BLOCK  ok
     * SNOW               BLOCK  BLOCK  BLOCK BLOCK  ok   BLOCK
     * STORM              BLOCK   ok     ok    ok   BLOCK  ok
     */
    private val blocks: Map<VisualGroup, Set<VisualGroup>> = mapOf(
        VisualGroup.CLEAR to setOf(
            VisualGroup.CLOUDY, VisualGroup.FOG, VisualGroup.RAIN, VisualGroup.SNOW, VisualGroup.STORM
        ),
        VisualGroup.CLOUDY to setOf(VisualGroup.CLEAR, VisualGroup.SNOW),
        VisualGroup.FOG to setOf(
            VisualGroup.CLEAR, VisualGroup.CLOUDY, VisualGroup.RAIN, VisualGroup.SNOW, VisualGroup.STORM
        ),
        VisualGroup.RAIN to setOf(VisualGroup.CLEAR, VisualGroup.SNOW),
        VisualGroup.SNOW to setOf(
            VisualGroup.CLEAR, VisualGroup.CLOUDY, VisualGroup.FOG, VisualGroup.RAIN, VisualGroup.STORM
        ),
        VisualGroup.STORM to setOf(VisualGroup.CLEAR, VisualGroup.SNOW)
    )

    /** One representative phenomenon per group, for driving the matrix. */
    private val representative = mapOf(
        VisualGroup.CLEAR to Phenomenon.CLEAR_SKY,
        VisualGroup.CLOUDY to Phenomenon.CLOUDS,
        VisualGroup.FOG to Phenomenon.FOG,
        VisualGroup.RAIN to Phenomenon.RAIN,
        VisualGroup.SNOW to Phenomenon.SNOW,
        VisualGroup.STORM to Phenomenon.THUNDERSTORM
    )

    @TestFactory
    fun `the contradiction matrix, all thirty-six cells`(): List<DynamicTest> =
        VisualGroup.entries.flatMap { expected ->
            VisualGroup.entries.map { photo ->
                val shouldBlock = photo in blocks.getValue(expected)
                DynamicTest.dynamicTest("expected $expected, photo says $photo -> ${if (shouldBlock) "BLOCK" else "ok"}") {
                    val result = service.contradicts(
                        expected = representative.getValue(expected),
                        scores = confident(photo),
                        isDay = true
                    )
                    if (shouldBlock) assertTrue(result) else assertFalse(result)
                }
            }
        }

    @Test
    fun `never blocks at night`() {
        // At night nobody can tell an overcast sky from a clear one, model included.
        assertFalse(
            service.contradicts(Phenomenon.THUNDERSTORM, confident(VisualGroup.CLEAR), isDay = false)
        )
    }

    @Test
    fun `never blocks when the winning group is not confident enough`() {
        // CLEAR wins, but only at 0.55 — under the 0.70 bar. An uncertain model gets no vote.
        val scores = mapOf(
            "CLEAR" to 0.55, "CLOUDY" to 0.30, "FOG" to 0.05,
            "RAIN" to 0.04, "SNOW" to 0.03, "STORM" to 0.03
        )

        assertFalse(service.contradicts(Phenomenon.THUNDERSTORM, scores, isDay = true))
    }

    @Test
    fun `never blocks when the expected group still scored above the floor`() {
        // CLEAR wins at 0.75, but STORM held 0.15 — above the 0.10 floor. The model saw
        // something storm-like, so it is not contradicting, only disagreeing about emphasis.
        val scores = mapOf(
            "CLEAR" to 0.75, "CLOUDY" to 0.04, "FOG" to 0.02,
            "RAIN" to 0.02, "SNOW" to 0.02, "STORM" to 0.15
        )

        assertFalse(service.contradicts(Phenomenon.THUNDERSTORM, scores, isDay = true))
    }

    @Test
    fun `never blocks when there is no analysis`() {
        // A photo uploaded before the analysis shipped, or one whose analysis was lost. A check
        // that did not run must not punish the photo it never looked at.
        assertFalse(service.contradicts(Phenomenon.THUNDERSTORM, scores = null, isDay = true))
    }

    @Test
    fun `never blocks when the analysis is empty`() {
        assertFalse(service.contradicts(Phenomenon.THUNDERSTORM, scores = emptyMap(), isDay = true))
    }

    @Test
    fun `never blocks when the winning group is one this build does not know`() {
        // A newer model reporting a group this build has no matrix row for. No opinion, no block.
        val scores = mapOf("TORNADO" to 0.90, "CLEAR" to 0.05, "STORM" to 0.05)

        assertFalse(service.contradicts(Phenomenon.THUNDERSTORM, scores, isDay = true))
    }

    @Test
    fun `every phenomenon in a group behaves like its group`() {
        // Drizzle must block a CLEAR photo exactly as RAIN does; the matrix is defined on
        // groups, and this is what proves the collapse is actually being applied.
        listOf(Phenomenon.DRIZZLE, Phenomenon.RAIN, Phenomenon.RAIN_SHOWER).forEach {
            assertTrue(service.contradicts(it, confident(VisualGroup.CLEAR), isDay = true), "$it")
        }
        listOf(Phenomenon.THUNDERSTORM, Phenomenon.HAILSTORM).forEach {
            assertTrue(service.contradicts(it, confident(VisualGroup.CLEAR), isDay = true), "$it")
        }
    }
}
```

- [ ] **Step 3: Run both tests and watch them fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.VisualGroupTest" --tests "com.skydex.api.service.PhotoAuthenticityServiceTest"
```

Expected: compilation failure — `Unresolved reference: VisualGroup`.

- [ ] **Step 4: Write VisualGroup**

`src/main/kotlin/com/skydex/api/domain/VisualGroup.kt`:
```kotlin
package com.skydex.api.domain

/**
 * The six classes a photograph can actually be sorted into, as opposed to the nine [Phenomenon]
 * values Open-Meteo's weather codes resolve to.
 *
 * The collapse is not a simplification for convenience. Drizzle, rain and a rain shower produce the
 * same image — raindrops are near-invisible to a phone camera and what shows is grey sky — and a
 * hailstorm photographs exactly like a thunderstorm, because the hail is on the ground and the
 * frame is pointed up. Asking a model to separate them would be asking it to guess, and then acting
 * on the guess.
 *
 * These names are a contract with `skydex-vision`: they are the keys of
 * [com.skydex.api.dto.VisionAnalysis.phenomenonScores]. Renaming one here without renaming it in
 * `app/prompts.py` silently turns every score for that group into "unknown group", which
 * [com.skydex.api.services.PhotoAuthenticityService] treats as no opinion — so the failure is a
 * quiet loss of enforcement, not an error anybody sees.
 */
enum class VisualGroup {
    CLEAR,
    CLOUDY,
    FOG,
    RAIN,
    SNOW,
    STORM;

    companion object {
        fun of(phenomenon: Phenomenon): VisualGroup = when (phenomenon) {
            Phenomenon.CLEAR_SKY -> CLEAR
            Phenomenon.CLOUDS -> CLOUDY
            Phenomenon.FOG -> FOG
            Phenomenon.DRIZZLE, Phenomenon.RAIN, Phenomenon.RAIN_SHOWER -> RAIN
            Phenomenon.SNOW -> SNOW
            Phenomenon.THUNDERSTORM, Phenomenon.HAILSTORM -> STORM
        }

        /** Null for a group name this build does not know — see the class KDoc. */
        fun fromNameOrNull(name: String): VisualGroup? =
            entries.firstOrNull { it.name.equals(name, ignoreCase = true) }
    }
}
```

- [ ] **Step 5: Write PhotoAuthenticityService**

`src/main/kotlin/com/skydex/api/services/PhotoAuthenticityService.kt`:
```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.VisualGroup
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service

/**
 * Stage 2 of photo validation: does the photograph contradict the weather Open-Meteo recorded?
 *
 * Pure — no network, no database, no model. Everything it needs arrives as arguments, which is what
 * lets `PhotoAuthenticityServiceTest` pin all thirty-six matrix cells in milliseconds.
 *
 * ## What it is NOT for
 *
 * It does not decide what the phenomenon is. Open-Meteo does that, and by the time this is called
 * the phenomenon is already settled. This answers one narrower question: is the photograph
 * incompatible with it?
 *
 * ## Innocent until confidently proven otherwise
 *
 * Three separate gates have to agree before this returns true, and each one exists to protect an
 * honest user from a model that is merely uncertain:
 *
 * 1. **Night skips the whole check.** No human can separate overcast from clear in the dark either.
 * 2. **The pair must be in the matrix.** A CLOUDY sky scoring as RAIN is not a contradiction —
 *    rain falls out of grey cloud and the photographs are the same.
 * 3. **The model must be confident in both directions.** The expected group has to have scored
 *    below a floor AND the winning group above a ceiling. A model that gives the expected group
 *    even a modest score has seen something consistent with it, and gets no vote.
 *
 * Anything it cannot evaluate — no analysis, an empty analysis, a group name from a newer model —
 * returns false. A check that did not run must never cost a user their capture.
 */
@Service
class PhotoAuthenticityService(
    @Value("\${skydex.vision.expected-score-max:0.10}") private val expectedScoreMax: Double,
    @Value("\${skydex.vision.top-score-min:0.70}") private val topScoreMin: Double
) {

    /**
     * True when [scores] confidently contradict [expected] and the pair is one that cannot be
     * reconciled.
     *
     * @param expected the phenomenon Open-Meteo recorded for the capture's place and time.
     * @param scores the cached `phenomenon_scores` from the photo's upload, or null if none.
     * @param isDay whether Open-Meteo reported daylight at that slot.
     */
    fun contradicts(expected: Phenomenon, scores: Map<String, Double>?, isDay: Boolean): Boolean {
        if (!isDay) return false
        if (scores.isNullOrEmpty()) return false

        val top = scores.maxByOrNull { it.value } ?: return false
        val observed = VisualGroup.fromNameOrNull(top.key) ?: return false

        val expectedGroup = VisualGroup.of(expected)
        if (observed in RECONCILABLE.getValue(expectedGroup)) return false

        val expectedScore = scores[expectedGroup.name] ?: 0.0
        return expectedScore < expectedScoreMax && top.value > topScoreMin
    }

    private companion object {

        /**
         * For each group Open-Meteo can report, the groups a photograph may plausibly score as.
         * The complement of this is the BLOCK matrix in the design document.
         *
         * The rule behind the shape: **when the weather is a distinctive phenomenon (CLEAR, FOG,
         * SNOW) the photograph must show it; when the weather is ordinary (CLOUDY, RAIN, STORM) the
         * photograph may look like anything in that neighbourhood.**
         *
         * - CLEAR admits only CLEAR. If the sky is clear, a photograph of it shows a clear sky.
         *   There is no "I photographed the cloudy part" — that would readmit every recycled
         *   storm photo taken on a sunny day.
         * - CLOUDY admits FOG (dense low cloud reads as mist), RAIN and STORM (a heavy overcast
         *   reads as both). It refuses CLEAR and SNOW.
         * - FOG admits only FOG. Fog is unmistakable and is the class CLIP is most reliable on.
         * - RAIN admits CLOUDY (the drops do not show; the grey sky does), FOG (heavy rain kills
         *   visibility) and STORM.
         * - SNOW admits only SNOW. Snow is EPIC rarity — the highest-value fraud target — and snow
         *   on the ground is among the easiest things in this list for a model to see.
         * - STORM shares RAIN's neighbourhood. Lightning is rarely in frame; a storm photographs
         *   as a dark sky or as rain.
         *
         * The asymmetry is intentional: CLOUDY admits FOG but FOG does not admit CLOUDY. Being
         * generous about what an ordinary sky may look like costs nothing; being generous about
         * what a distinctive phenomenon may look like gives away the whole check.
         */
        val RECONCILABLE: Map<VisualGroup, Set<VisualGroup>> = mapOf(
            VisualGroup.CLEAR to setOf(VisualGroup.CLEAR),
            VisualGroup.CLOUDY to setOf(
                VisualGroup.CLOUDY, VisualGroup.FOG, VisualGroup.RAIN, VisualGroup.STORM
            ),
            VisualGroup.FOG to setOf(VisualGroup.FOG),
            VisualGroup.RAIN to setOf(
                VisualGroup.RAIN, VisualGroup.CLOUDY, VisualGroup.FOG, VisualGroup.STORM
            ),
            VisualGroup.SNOW to setOf(VisualGroup.SNOW),
            VisualGroup.STORM to setOf(
                VisualGroup.STORM, VisualGroup.CLOUDY, VisualGroup.FOG, VisualGroup.RAIN
            )
        )
    }
}
```

- [ ] **Step 6: Run both tests and watch them pass**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.VisualGroupTest" --tests "com.skydex.api.service.PhotoAuthenticityServiceTest"
```

Expected: 5 passed in `VisualGroupTest`, 43 in `PhotoAuthenticityServiceTest` (36 dynamic + 7 named).

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add src/main/kotlin/com/skydex/api/domain/VisualGroup.kt \
        src/main/kotlin/com/skydex/api/services/PhotoAuthenticityService.kt \
        src/test/kotlin/com/skydex/api/domain/VisualGroupTest.kt \
        src/test/kotlin/com/skydex/api/service/PhotoAuthenticityServiceTest.kt
git commit -m "feat: visual groups and the photo contradiction matrix"
```

---

### Task 3: Analysis at upload time

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/UploadedPhoto.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/errors/DomainExceptions.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/GlobalExceptionHandler.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/PhotoAnalysisService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/PhotoController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt`

**Interfaces:**
- Consumes: `VisionClient.analyze`, `VisionAnalysis`.
- Produces:
  - `UploadedPhoto.visionOutdoorScore: Double?`, `.visionScores: String?` (JSON), `.visionModel: String?`, `.visionAnalyzedAt: Instant?`
  - `ServiceUnavailableException`, `UnprocessableContentException`
  - `PhotoAnalysisService.analyze(bytes: ByteArray, filename: String): VisionAnalysis` — throws on outage or rejection
  - `PhotoAnalysisService.serialise(analysis: VisionAnalysis): String`
  - `PhotoAnalysisService.deserialise(json: String?): Map<String, Double>?`

- [ ] **Step 1: Write the failing test**

Append to `src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt`. The class needs a mocked `VisionClient`, so add these to the imports and the class body:

```kotlin
// add to the imports at the top of the file
import com.skydex.api.dto.VisionAnalysis
import com.skydex.api.services.VisionClient
import org.mockito.kotlin.any
import org.springframework.boot.test.mock.mockito.MockBean
import org.mockito.Mockito.`when`
import org.junit.jupiter.api.BeforeEach
```

```kotlin
    // --- vision analysis at upload -----------------------------------------------------------
    //
    // CLIP never runs in this suite. The real model is exercised by the golden-set regression in
    // skydex-vision; what these tests pin is the wiring: what gets stored, and which status each
    // failure produces.

    @MockBean
    private lateinit var vision: VisionClient

    private fun analysis(outdoor: Double, top: String = "RAIN") = VisionAnalysis(
        outdoorScore = outdoor,
        phenomenonScores = mapOf(
            "CLEAR" to 0.04, "CLOUDY" to 0.04, "FOG" to 0.04,
            "RAIN" to 0.04, "SNOW" to 0.04, "STORM" to 0.04
        ) + (top to 0.80),
        model = "clip-vit-b-32-zeroshot-v1"
    )

    @BeforeEach
    fun stubVision() {
        `when`(vision.analyze(any(), any())).thenReturn(analysis(outdoor = 0.94))
    }

    @Test
    fun `stores the vision scores alongside the photo`() {
        val user = persistUser(email = "analysed@skydex.com")
        val part = MockMultipartFile("file", "storm.jpg", MediaType.IMAGE_JPEG_VALUE, jpegBytes)

        val body = mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isCreated)
            .andReturn().response.contentAsString

        val filename = objectMapper.readTree(body).get("photoUrl").asText().substringAfterLast('/')
        val stored = uploadedPhotoRepository.findByFilename(filename)!!

        assertEquals(0.94, stored.visionOutdoorScore)
        assertEquals("clip-vit-b-32-zeroshot-v1", stored.visionModel)
        assertNotNull(stored.visionAnalyzedAt)
        // Stored as JSON text rather than as a JSONB column: the map is read back whole, never
        // queried into, so a column type that needs a Hibernate dialect extension buys nothing.
        assertTrue(stored.visionScores!!.contains("\"RAIN\":0.8"), stored.visionScores!!)
    }

    @Test
    fun `refuses a photo the model does not think is the sky`() {
        `when`(vision.analyze(any(), any())).thenReturn(analysis(outdoor = 0.12))
        val user = persistUser(email = "notsky@skydex.com")
        val part = MockMultipartFile("file", "wall.jpg", MediaType.IMAGE_JPEG_VALUE, jpegBytes)

        mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isUnprocessableEntity)

        // Nothing was written. Rejecting at upload rather than at capture is what keeps junk out
        // of the database entirely, and what lets the user find out before typing a title.
        assertEquals(0, uploadedPhotoRepository.count())
    }

    @Test
    fun `answers 503 when the vision service cannot be reached`() {
        `when`(vision.analyze(any(), any())).thenReturn(null)
        val user = persistUser(email = "visiondown@skydex.com")
        val part = MockMultipartFile("file", "storm.jpg", MediaType.IMAGE_JPEG_VALUE, jpegBytes)

        mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isServiceUnavailable)

        // A 503 must cost the user nothing: no row, no file, nothing to clean up, and a retry
        // that behaves exactly like a first attempt.
        assertEquals(0, uploadedPhotoRepository.count())
    }
}
```

Note the closing brace — it replaces the existing final `}` of the class.

Add the Mockito-Kotlin dependency to `build.gradle.kts` so `any()` works with non-null Kotlin parameters:
```kotlin
	testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.PhotoControllerTest"
```

Expected: compilation failure — `Unresolved reference: visionOutdoorScore`.

- [ ] **Step 3: Add the columns**

In `src/main/kotlin/com/skydex/api/models/UploadedPhoto.kt`, add these four parameters after `consumedAt`, and extend the class KDoc:

```kotlin
    @Column(name = "consumed_at")
    var consumedAt: Instant? = null,

    /**
     * What `skydex-vision` said about this photograph when it was uploaded, cached so the model
     * never runs twice for one image.
     *
     * All four are nullable and all four are written together or not at all. Null means the photo
     * was uploaded before analysis existed, and `PhotoAuthenticityService` reads that as "no
     * opinion" rather than as a failed check — a photo must never be punished for a check that did
     * not run when it was taken.
     *
     * The window in which that can happen is bounded by [PhotoProvenanceService.MAX_AGE]: thirty
     * minutes after this ships, no unanalysed photo is citable by any capture.
     */
    @Column(name = "vision_outdoor_score")
    var visionOutdoorScore: Double? = null,

    /**
     * `phenomenon_scores` as JSON text, keyed by [com.skydex.api.domain.VisualGroup] name.
     *
     * Text and not JSONB: the map is read back whole and never queried into, so a column type
     * needing a Hibernate dialect extension would buy nothing and cost a dependency.
     */
    @Column(name = "vision_scores", columnDefinition = "TEXT")
    var visionScores: String? = null,

    /**
     * The model name that produced the scores, e.g. `clip-vit-b-32-probe-v1`.
     *
     * Recorded because a retrained model changes what these numbers mean. Without it, a capture
     * scored by a model that has since been replaced is indistinguishable from one scored by the
     * current one, and there is no way to re-examine a disputed verdict.
     */
    @Column(name = "vision_model", length = 64)
    var visionModel: String? = null,

    @Column(name = "vision_analyzed_at")
    var visionAnalyzedAt: Instant? = null
)
```

- [ ] **Step 4: Add the two exceptions**

Append to `src/main/kotlin/com/skydex/api/errors/DomainExceptions.kt`:
```kotlin
/**
 * Maps to **503**. An upstream this request cannot proceed without did not answer.
 *
 * Two callers, both on the capture path: `PhotoAnalysisService` when `skydex-vision` is
 * unreachable, and `CaptureValidationService` when Open-Meteo is. Both are genuinely retryable and
 * both cost the caller nothing — no photo is spent and no row is written before either can be
 * raised — so the client can simply try again with the same photo.
 *
 * Distinct from a 500 on purpose: 500 means we are broken, 503 means come back in a moment. The
 * Android client shows different copy for each, and telling a user to retry a genuine fault is
 * worse than useless.
 */
class ServiceUnavailableException(message: String) : RuntimeException(message)

/**
 * Maps to **422**. The request is well-formed and the caller is entitled to make it, but the
 * content itself cannot be accepted.
 *
 * One caller: a photo the vision model does not believe is the sky. This is deliberately not a 400
 * — a 400 in this API means "this request is malformed", and the Android client's error presenter
 * reads 400 as "re-check what you typed". Nothing was typed wrong here; the picture is the problem.
 */
class UnprocessableContentException(message: String) : RuntimeException(message)
```

- [ ] **Step 5: Map them to statuses**

In `src/main/kotlin/com/skydex/api/controllers/GlobalExceptionHandler.kt`, add the imports:
```kotlin
import com.skydex.api.errors.ServiceUnavailableException
import com.skydex.api.errors.UnprocessableContentException
```

and add these two handlers next to `handleBadRequest`:
```kotlin
    /**
     * Explicit, not left to the catch-all: `ServiceUnavailableException` does not implement
     * Spring's `ErrorResponse`, so `handleUnexpected` would report an upstream outage as a 500 —
     * telling the client we are broken when the honest answer is "try again shortly".
     */
    @ExceptionHandler(ServiceUnavailableException::class)
    fun handleServiceUnavailable(e: ServiceUnavailableException): ResponseEntity<ErrorResponse> {
        log.warn("An upstream this request needs is unavailable: {}", e.message)
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(ErrorResponse(e.message ?: "Service temporarily unavailable"))
    }

    @ExceptionHandler(UnprocessableContentException::class)
    fun handleUnprocessableContent(e: UnprocessableContentException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(ErrorResponse(e.message ?: "Content could not be accepted"))
```

- [ ] **Step 6: Write PhotoAnalysisService**

`src/main/kotlin/com/skydex/api/services/PhotoAnalysisService.kt`:
```kotlin
package com.skydex.api.services

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import com.skydex.api.dto.VisionAnalysis
import com.skydex.api.errors.ServiceUnavailableException
import com.skydex.api.errors.UnprocessableContentException
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service

/**
 * Stage 1 of photo validation, plus the persistence format for stage 2's inputs.
 *
 * ## Why this runs at upload and not at capture
 *
 * Three reasons, all load-bearing:
 *
 * - **The latency disappears.** The client fires the upload the moment the shutter closes and the
 *   user then spends several seconds typing a title, so the quarter-second forward pass is free.
 * - **The result is cached** on the `uploaded_photos` row, so the model runs exactly once per
 *   photograph however many times a capture is retried against it.
 * - **An outage costs nothing.** This point in the flow is before any photo row exists, before any
 *   photo is spent, and before any Open-Meteo call is paid for. A 503 here is a clean retry.
 *
 * ## Why a rejection is 422 and not 400
 *
 * The upload is well-formed and the caller is entitled to make it. What is wrong is the picture.
 * The Android error presenter reads a 400 as "re-check what you typed", which is the wrong
 * instruction for someone who needs to point the camera somewhere else.
 */
@Service
class PhotoAnalysisService(
    private val vision: VisionClient,
    private val objectMapper: ObjectMapper,
    @Value("\${skydex.vision.outdoor-min:0.60}") private val outdoorMin: Double
) {

    /**
     * Scores [bytes], or refuses the upload.
     *
     * @throws ServiceUnavailableException the model could not be reached, or rejected the bytes.
     *   Both are the server's problem from the caller's side: they uploaded a JPEG that
     *   `PhotoStorageService` already verified the magic bytes of, so a model that will not read it
     *   is a model that is misbehaving.
     * @throws UnprocessableContentException the model does not believe this is an outdoor sky.
     */
    fun analyze(bytes: ByteArray, filename: String): VisionAnalysis {
        val analysis = vision.analyze(bytes, filename)
            ?: throw ServiceUnavailableException("Photo analysis is unavailable right now")

        if (analysis.outdoorScore < outdoorMin) {
            log.info(
                "Rejecting an upload: outdoor score {} is below {} (model {})",
                analysis.outdoorScore, outdoorMin, analysis.model
            )
            throw UnprocessableContentException("This photo does not look like the sky")
        }

        return analysis
    }

    fun serialise(analysis: VisionAnalysis): String =
        objectMapper.writeValueAsString(analysis.phenomenonScores)

    /**
     * The stored scores, or null when there are none to read.
     *
     * Unparseable JSON also yields null rather than throwing. This is read on the capture path, and
     * a stored blob that cannot be parsed is a bug in something that already happened — failing the
     * user's capture over it would punish them for it twice.
     */
    fun deserialise(json: String?): Map<String, Double>? {
        if (json.isNullOrBlank()) return null
        return try {
            objectMapper.readValue<Map<String, Double>>(json)
        } catch (e: Exception) {
            log.warn("Stored vision scores could not be parsed; treating as no analysis", e)
            null
        }
    }

    private companion object {
        val log = LoggerFactory.getLogger(PhotoAnalysisService::class.java)
    }
}
```

- [ ] **Step 7: Wire it into the controller**

Replace the body of `src/main/kotlin/com/skydex/api/controllers/PhotoController.kt`:
```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.PhotoUploadResponse
import com.skydex.api.models.UploadedPhoto
import com.skydex.api.models.User
import com.skydex.api.repositories.UploadedPhotoRepository
import com.skydex.api.services.PhotoAnalysisService
import com.skydex.api.services.PhotoStorageService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.multipart.MultipartFile
import java.time.Instant

@RestController
@RequestMapping("/api/photos")
class PhotoController(
    private val photos: PhotoStorageService,
    private val uploadedPhotos: UploadedPhotoRepository,
    private val analysis: PhotoAnalysisService
) {

    /**
     * [currentUser] is load-bearing: it is recorded as the [UploadedPhoto] row's uploader, which is
     * what `PhotoProvenanceService.verify` later checks before `POST /api/events` may cite the
     * returned path.
     *
     * ## Order of operations
     *
     * Analysis runs **before** anything is written to disk or to the database, and that order is
     * the whole reason a rejected or unanalysable photo costs nothing. A 422 or a 503 raised here
     * leaves no file, no row, and nothing for a cleanup sweep to find later.
     *
     * The bytes are read once, into `file.bytes`, and handed to both the analyser and the storage
     * service. Reading the multipart stream twice would fail on the second read.
     */
    @PostMapping
    fun upload(
        @AuthenticationPrincipal currentUser: User,
        @RequestParam("file") file: MultipartFile
    ): ResponseEntity<PhotoUploadResponse> {
        val bytes = file.bytes
        val filename = file.originalFilename ?: "upload"

        // Throws 422 (not the sky) or 503 (model unreachable). Nothing is persisted yet.
        val scored = analysis.analyze(bytes, filename)

        val url = photos.store(bytes, file.originalFilename, file.contentType)
        uploadedPhotos.save(
            UploadedPhoto(
                filename = url.substringAfterLast('/'),
                uploaderId = currentUser.id!!,
                visionOutdoorScore = scored.outdoorScore,
                visionScores = analysis.serialise(scored),
                visionModel = scored.model,
                visionAnalyzedAt = Instant.now()
            )
        )
        return ResponseEntity.status(HttpStatus.CREATED).body(PhotoUploadResponse(url))
    }
}
```

`PhotoStorageService.store` validates the magic bytes and can still throw `BadUploadException` (400) for a file that is not really a JPEG. That check now runs *after* the model, which means a mislabelled file pays for an analysis it did not need. Accepted: it is a rare case, the cost is one CPU forward pass, and the alternative — storing before analysing — is what makes a 503 leave orphans behind.

- [ ] **Step 8: Run the tests and watch them pass**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.PhotoControllerTest"
```

Expected: all pass, including the three new ones. The pre-existing tests in this class now go through the `@BeforeEach` stub, which returns a high outdoor score, so they behave as before.

- [ ] **Step 9: Commit**

Ask the user first, then:

```bash
git add build.gradle.kts \
        src/main/kotlin/com/skydex/api/models/UploadedPhoto.kt \
        src/main/kotlin/com/skydex/api/errors/DomainExceptions.kt \
        src/main/kotlin/com/skydex/api/controllers/GlobalExceptionHandler.kt \
        src/main/kotlin/com/skydex/api/services/PhotoAnalysisService.kt \
        src/main/kotlin/com/skydex/api/controllers/PhotoController.kt \
        src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt
git commit -m "feat: analyse photos at upload time and refuse non-sky images"
```

---

### Task 4: Daylight from Open-Meteo

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/OpenMeteoResponse.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/OpenMeteoClient.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/OpenMeteoClientTest.kt`

**Interfaces:**
- Consumes: nothing.
- Produces: `HourlyData.isDay: List<Int?>` — Open-Meteo's `is_day`, 1 for daylight and 0 for night. Defaults to `emptyList()` so every existing `HourlyData(...)` construction in the test suite keeps compiling.

- [ ] **Step 1: Write the failing test**

Add to `src/test/kotlin/com/skydex/api/service/OpenMeteoClientTest.kt`:
```kotlin
    @Test
    fun `requests the is_day series and parses it`() {
        // Night is the one condition under which the photo check is skipped entirely, so the
        // capture path cannot work without this series. Asking for it costs nothing: the call
        // to Open-Meteo is already being made for the weather code.
        server.expect(requestTo(org.hamcrest.Matchers.containsString("is_day")))
            .andRespond(
                withSuccess(
                    """
                    {
                      "latitude": -30.0, "longitude": -51.0,
                      "hourly": {
                        "time": ["2026-08-16T14:00"],
                        "temperature_2m": [21.0],
                        "weather_code": [95],
                        "is_day": [1]
                      }
                    }
                    """.trimIndent(),
                    MediaType.APPLICATION_JSON
                )
            )

        val response = client.fetchHourlyForecast(-30.0, -51.0)

        assertEquals(listOf(1), response!!.hourly!!.isDay)
    }

    @Test
    fun `tolerates a response with no is_day series`() {
        // A cached or proxied response from before this field was requested. Missing daylight
        // information must not fail the parse — the caller defaults to treating it as day.
        server.expect(org.springframework.test.web.client.match.MockRestRequestMatchers.anything())
            .andRespond(
                withSuccess(
                    """
                    {
                      "latitude": -30.0, "longitude": -51.0,
                      "hourly": {
                        "time": ["2026-08-16T14:00"],
                        "temperature_2m": [21.0],
                        "weather_code": [95]
                      }
                    }
                    """.trimIndent(),
                    MediaType.APPLICATION_JSON
                )
            )

        assertEquals(emptyList<Int?>(), client.fetchHourlyForecast(-30.0, -51.0)!!.hourly!!.isDay)
    }
```

Match the existing imports and the way `server` and `client` are built in that file — read it before writing, and reuse its fixtures rather than adding new ones.

- [ ] **Step 2: Run the test and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.OpenMeteoClientTest"
```

Expected: `Unresolved reference: isDay`.

- [ ] **Step 3: Add the field**

In `src/main/kotlin/com/skydex/api/dto/OpenMeteoResponse.kt`:
```kotlin
data class HourlyData(
    val time: List<String>,
    @JsonProperty("temperature_2m") val temperatureCelsius: List<Double?>,
    @JsonProperty("weather_code") val weatherCode: List<Int?>,
    /**
     * Open-Meteo's `is_day`: 1 during daylight at that location, 0 at night.
     *
     * Defaulted to empty, which matters twice. It keeps every existing `HourlyData(...)` in the
     * test suite compiling, and it makes a response that predates this field — a cached one, a
     * proxied one — parse rather than fail. Callers read a missing entry as daylight, which is the
     * permissive choice: night only ever *skips* the photo check, so defaulting to day keeps the
     * check running rather than silently disabling it.
     */
    @JsonProperty("is_day") val isDay: List<Int?> = emptyList()
)
```

- [ ] **Step 4: Request it**

In `src/main/kotlin/com/skydex/api/services/OpenMeteoClient.kt`, change the `hourly=` list:
```kotlin
                .uri(
                    "/v1/forecast?latitude={lat}&longitude={lon}" +
                        "&hourly=temperature_2m,weather_code,is_day&timezone=UTC&past_days=1&forecast_days=2",
                    latitude,
                    longitude
                )
```

- [ ] **Step 5: Run the tests and watch them pass**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.OpenMeteoClientTest"
```

Expected: all pass.

- [ ] **Step 6: Commit**

Ask the user first, then:

```bash
git add src/main/kotlin/com/skydex/api/dto/OpenMeteoResponse.kt \
        src/main/kotlin/com/skydex/api/services/OpenMeteoClient.kt \
        src/test/kotlin/com/skydex/api/service/OpenMeteoClientTest.kt
git commit -m "feat: request daylight from Open-Meteo"
```

---

### Task 5: Open-Meteo decides the phenomenon

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/UnconfirmedReason.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/WeatherEvent.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/CaptureValidationService.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt`

**Interfaces:**
- Consumes: `PhotoAuthenticityService.contradicts`, `OpenMeteoClient.fetchHourlyForecast`, `VisualGroup`, `ServiceUnavailableException`.
- Produces:
  - `UnconfirmedReason` — enum `PHOTO_CONTRADICTS_WEATHER, IMPLAUSIBLE_TRAVEL, MOCK_LOCATION`
  - `WeatherEvent.unconfirmedReason: UnconfirmedReason?`
  - `ValidationResult(phenomenon: Phenomenon, status: ValidationStatus, observedWeatherCode: Int, xpAwarded: Int, unconfirmedReason: UnconfirmedReason?)`
  - `CaptureValidationService.validate(latitude, longitude, capturedAt, previous, locationIsMock, photoScores): ValidationResult` — throws `ServiceUnavailableException`

- [ ] **Step 1: Write the reason enum**

`src/main/kotlin/com/skydex/api/domain/UnconfirmedReason.kt`:
```kotlin
package com.skydex.api.domain

/**
 * Why a capture was stored as [ValidationStatus.UNCONFIRMED].
 *
 * Three values, and the one that is missing is worth naming. Before the vision model, the commonest
 * cause was "the phenomenon you claimed is not the one Open-Meteo recorded" — and that cause can no
 * longer occur, because the user no longer makes a claim. An upstream failure or a capture outside
 * the forecast window, which used to land in the same bucket, are now a 503 instead: without
 * Open-Meteo there is no phenomenon, and `weather_events.phenomenon` is NOT NULL.
 *
 * Null on a row means the reason was not recorded — either it predates this enum, or the capture is
 * CONFIRMED. Both read correctly as "nothing to explain".
 */
enum class UnconfirmedReason {
    /** The photograph confidently shows something the recorded weather rules out. */
    PHOTO_CONTRADICTS_WEATHER,

    /** The position is not one the author could have reached since their previous capture. */
    IMPLAUSIBLE_TRAVEL,

    /** The client reported the fix as coming from a mock location provider. */
    MOCK_LOCATION
}
```

- [ ] **Step 2: Add the column**

In `src/main/kotlin/com/skydex/api/models/WeatherEvent.kt`, add the import and the field before `userId`:
```kotlin
import com.skydex.api.domain.UnconfirmedReason
```
```kotlin
    /**
     * Why [validationStatus] is UNCONFIRMED, or null when it is CONFIRMED or when the row predates
     * this column. Nullable and never back-filled: a guessed reason on a historical row would be
     * worse than an absent one.
     */
    @Enumerated(EnumType.STRING)
    @Column(name = "unconfirmed_reason", length = 32)
    var unconfirmedReason: UnconfirmedReason? = null,

    @Column(name = "user_id", nullable = false)
    var userId: UUID
```

- [ ] **Step 3: Write the failing test**

Replace `src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt` entirely:
```kotlin
package com.skydex.api.service

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.UnconfirmedReason
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.dto.HourlyData
import com.skydex.api.dto.OpenMeteoResponse
import com.skydex.api.errors.ServiceUnavailableException
import com.skydex.api.services.CaptureValidationService
import com.skydex.api.services.LastKnownPosition
import com.skydex.api.services.OpenMeteoClient
import com.skydex.api.services.PhotoAuthenticityService
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.mockito.Mockito.mock
import org.mockito.Mockito.`when`
import java.time.Instant

class CaptureValidationServiceTest {

    private val client = mock(OpenMeteoClient::class.java)
    private val authenticity = PhotoAuthenticityService(expectedScoreMax = 0.10, topScoreMin = 0.70)
    private val service = CaptureValidationService(client, authenticity)

    private val at = Instant.parse("2026-08-16T14:10:00Z")

    private fun forecast(
        code: Int?,
        isDay: Int? = 1,
        slot: String = "2026-08-16T14:00"
    ) = OpenMeteoResponse(
        latitude = -30.0,
        longitude = -51.0,
        hourly = HourlyData(
            time = listOf(slot),
            temperatureCelsius = listOf(20.0),
            weatherCode = listOf(code),
            isDay = listOfNotNull(isDay)
        )
    )

    /** Scores in which [winner] takes 0.80 and everything else splits the remainder. */
    private fun photoSaying(winner: String) = mapOf(
        "CLEAR" to 0.04, "CLOUDY" to 0.04, "FOG" to 0.04,
        "RAIN" to 0.04, "SNOW" to 0.04, "STORM" to 0.04
    ) + (winner to 0.80)

    private fun validate(
        code: Int? = 95,
        isDay: Int? = 1,
        previous: LastKnownPosition? = null,
        locationIsMock: Boolean = false,
        photoScores: Map<String, Double>? = photoSaying("STORM")
    ) = service.validate(
        latitude = -30.0,
        longitude = -51.0,
        capturedAt = at,
        previous = previous,
        locationIsMock = locationIsMock,
        photoScores = photoScores
    )

    // --- the happy path -----------------------------------------------------------------------

    @Test
    fun `takes the phenomenon from open-meteo and confirms a consistent photo`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        val result = validate()

        assertEquals(Phenomenon.THUNDERSTORM, result.phenomenon)
        assertEquals(ValidationStatus.CONFIRMED, result.status)
        assertEquals(95, result.observedWeatherCode)
        assertEquals(Phenomenon.THUNDERSTORM.rarity.xp, result.xpAwarded)
        assertNull(result.unconfirmedReason)
    }

    @Test
    fun `confirms even when the photo scored a neighbouring group`() {
        // Open-Meteo says thunderstorm, the photo reads as rain. Lightning is rarely in frame,
        // so this is the ordinary case, not a contradiction.
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        val result = validate(photoScores = photoSaying("RAIN"))

        assertEquals(ValidationStatus.CONFIRMED, result.status)
    }

    // --- the three ways to be unconfirmed -----------------------------------------------------

    @Test
    fun `stores the phenomenon but awards nothing when the photo contradicts the weather`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        val result = validate(photoScores = photoSaying("CLEAR"))

        // The phenomenon is still Open-Meteo's — the weather really was a thunderstorm, and
        // blanking that to make the verdict tidier would erase a true fact about the row.
        assertEquals(Phenomenon.THUNDERSTORM, result.phenomenon)
        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(0, result.xpAwarded)
        assertEquals(UnconfirmedReason.PHOTO_CONTRADICTS_WEATHER, result.unconfirmedReason)
    }

    @Test
    fun `reports a mocked location`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        val result = validate(locationIsMock = true)

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(UnconfirmedReason.MOCK_LOCATION, result.unconfirmedReason)
        // The phenomenon is still recorded: the weather is a public fact independent of who
        // claims to have been standing in it.
        assertEquals(Phenomenon.THUNDERSTORM, result.phenomenon)
    }

    @Test
    fun `reports an implausible journey`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        val result = validate(
            previous = LastKnownPosition(-3.1, -60.0, at.minusSeconds(60))
        )

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(UnconfirmedReason.IMPLAUSIBLE_TRAVEL, result.unconfirmedReason)
    }

    // --- night and missing analysis -----------------------------------------------------------

    @Test
    fun `does not hold a photo against a night capture`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95, isDay = 0))

        val result = validate(photoScores = photoSaying("CLEAR"))

        assertEquals(ValidationStatus.CONFIRMED, result.status)
    }

    @Test
    fun `treats a missing is_day series as daylight`() {
        // A response from before is_day was requested. Defaulting to night would silently
        // disable the photo check for every capture.
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95, isDay = null))

        val result = validate(photoScores = photoSaying("CLEAR"))

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
    }

    @Test
    fun `confirms a photo that was never analysed`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 95))

        assertEquals(ValidationStatus.CONFIRMED, validate(photoScores = null).status)
    }

    // --- 503 -----------------------------------------------------------------------------------

    @Test
    fun `refuses the capture when open-meteo cannot be reached`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(null)

        // Not UNCONFIRMED: without Open-Meteo there is no phenomenon, and the column is NOT NULL.
        // The caller turns this into a 503, before any photo is spent, so the retry is free.
        assertThrows<ServiceUnavailableException> { validate() }
    }

    @Test
    fun `refuses a capture outside the forecast window`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0))
            .thenReturn(forecast(code = 95, slot = "2026-08-10T14:00"))

        assertThrows<ServiceUnavailableException> { validate() }
    }

    @Test
    fun `refuses a slot with no weather code`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = null))

        assertThrows<ServiceUnavailableException> { validate() }
    }

    @Test
    fun `refuses a weather code no species covers`() {
        // Every code Open-Meteo documents maps to a Phenomenon, so this is an upstream anomaly
        // rather than a user problem — and there is no row that can honestly be written for it.
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(forecast(code = 4))

        assertThrows<ServiceUnavailableException> { validate() }
    }
}
```

- [ ] **Step 4: Run the test and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.CaptureValidationServiceTest"
```

Expected: compilation failure — the constructor takes one argument and `validate` has a different signature.

- [ ] **Step 5: Rewrite the service**

Replace `src/main/kotlin/com/skydex/api/services/CaptureValidationService.kt` entirely:
```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.UnconfirmedReason
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.errors.ServiceUnavailableException
import org.springframework.stereotype.Service
import java.time.Duration
import java.time.Instant
import java.time.LocalDateTime
import java.time.ZoneOffset
import java.time.format.DateTimeParseException
import kotlin.math.abs

data class ValidationResult(
    val phenomenon: Phenomenon,
    val status: ValidationStatus,
    val observedWeatherCode: Int,
    val xpAwarded: Int,
    val unconfirmedReason: UnconfirmedReason?
)

/**
 * Decides what a capture *is* and whether it earns XP.
 *
 * ## What changed, and why it is not the same service it was
 *
 * This used to check a phenomenon the **user** claimed against Open-Meteo's record. It no longer
 * does, because the user no longer claims one: Open-Meteo's weather code IS the phenomenon. That
 * turns a check that could fail softly into a lookup that cannot fail at all — `weather_events
 * .phenomenon` is NOT NULL, so a missing answer is not an UNCONFIRMED capture, it is no capture.
 * Every path that used to return "unconfirmed, we could not tell" now throws
 * [ServiceUnavailableException] and the controller answers 503.
 *
 * That is cheap and it is the reason this ordering is safe: `PhotoProvenanceService.consume` runs
 * inside `CaptureCommitService.commit`, which is reached only *after* this returns. A capture that
 * dies here has spent nothing, so the client retries with the same photo for the remainder of its
 * thirty-minute `MAX_AGE`.
 *
 * ## The Open-Meteo call is no longer optional
 *
 * The previous version ran the position checks first, so an implausible capture cost no upstream
 * request. It cannot any more: even an implausible capture needs a phenomenon to be stored under.
 * The saving is gone and the ordering is inverted — weather first, verdict second.
 *
 * ## `locationIsMock` is still worth exactly what the client's honesty is worth
 *
 * It is the CLIENT's report that Android flagged the fix as coming from a mock provider, so it
 * stops a casual mock-GPS app installed alongside our unmodified client and nothing more. It earns
 * its place because casual mock-GPS is what most cheating actually looks like and it costs one
 * boolean. Do not write code elsewhere that treats it as proof of anything.
 *
 * [previous] is read without synchronisation, so the travel verdict reached here is provisional and
 * this is NOT the place that enforces it. `CaptureCommitService.commit` re-checks travel against the
 * trail re-read under a row lock and can downgrade a CONFIRMED result on the way out.
 */
@Service
class CaptureValidationService(
    private val openMeteoClient: OpenMeteoClient,
    private val authenticity: PhotoAuthenticityService
) {

    /**
     * The phenomenon Open-Meteo recorded for this place and time, and whether the capture earns XP.
     *
     * @param photoScores the cached `phenomenon_scores` from the photo's upload, or null for a
     *   photo that was never analysed. Null skips stage 2 rather than failing it.
     * @throws ServiceUnavailableException Open-Meteo did not answer, answered with no usable slot
     *   near [capturedAt], or answered with a code no [Phenomenon] covers.
     */
    fun validate(
        latitude: Double,
        longitude: Double,
        capturedAt: Instant,
        previous: LastKnownPosition?,
        locationIsMock: Boolean,
        photoScores: Map<String, Double>?
    ): ValidationResult {
        val hourly = openMeteoClient.fetchHourlyForecast(latitude, longitude)?.hourly
            ?: throw ServiceUnavailableException("The weather service is unavailable right now")

        var nearestIndex = -1
        var nearestDistance = Long.MAX_VALUE

        val slots = minOf(hourly.time.size, hourly.weatherCode.size)
        for (i in 0 until slots) {
            val slotInstant = parseSlot(hourly.time[i]) ?: continue
            val distance = abs(Duration.between(slotInstant, capturedAt).toMillis())
            if (distance < nearestDistance) {
                nearestDistance = distance
                nearestIndex = i
            }
        }

        if (nearestIndex < 0 || nearestDistance > MAX_SKEW.toMillis()) {
            throw ServiceUnavailableException("The weather service is unavailable right now")
        }

        val observedCode = hourly.weatherCode[nearestIndex]
            ?: throw ServiceUnavailableException("The weather service is unavailable right now")

        // Every code Open-Meteo documents maps to a species, so a null here is an upstream anomaly
        // and not something the user did. There is no honest row to write without a phenomenon.
        val phenomenon = Phenomenon.fromWeatherCode(observedCode)
            ?: throw ServiceUnavailableException("The weather service is unavailable right now")

        // Absent means daylight. Night only ever *skips* the photo check, so defaulting the other
        // way would silently disable stage 2 for every capture the moment the field went missing.
        val isDay = hourly.isDay.getOrNull(nearestIndex)?.let { it == 1 } ?: true

        val reason = when {
            locationIsMock -> UnconfirmedReason.MOCK_LOCATION
            !TravelPlausibility.isReachable(previous, latitude, longitude, capturedAt) ->
                UnconfirmedReason.IMPLAUSIBLE_TRAVEL
            authenticity.contradicts(phenomenon, photoScores, isDay) ->
                UnconfirmedReason.PHOTO_CONTRADICTS_WEATHER
            else -> null
        }

        return ValidationResult(
            phenomenon = phenomenon,
            status = if (reason == null) ValidationStatus.CONFIRMED else ValidationStatus.UNCONFIRMED,
            observedWeatherCode = observedCode,
            xpAwarded = if (reason == null) phenomenon.rarity.xp else 0,
            unconfirmedReason = reason
        )
    }

    /** Open-Meteo returns "2026-08-16T14:00" with no offset; we requested timezone=UTC. */
    private fun parseSlot(raw: String): Instant? = try {
        LocalDateTime.parse(raw).toInstant(ZoneOffset.UTC)
    } catch (e: DateTimeParseException) {
        null
    }

    private companion object {
        /**
         * Covers hourly granularity plus slack for a truncated or gap-ridden upstream response —
         * NOT phone clock skew. There is no phone clock in this path: `capturedAt` is stamped by
         * the server before this is ever called, so a client's clock can never influence it.
         */
        val MAX_SKEW: Duration = Duration.ofMinutes(90)
    }
}
```

- [ ] **Step 6: Update the ValidationStatus documentation**

The KDoc on `src/main/kotlin/com/skydex/api/domain/ValidationStatus.kt` now describes a design that no longer exists. Replace its body:
```kotlin
package com.skydex.api.domain

/**
 * How much the server was able to check about a capture.
 *
 * CONFIRMED means exactly four things, and it is worth reading them as the complete list:
 * - Open-Meteo has a record for that place and time, and it is the phenomenon the capture is
 *   stored under — the user no longer claims one, so this is a lookup rather than a check;
 * - the photograph does not confidently contradict that weather, and `skydex-vision` believed it
 *   was an outdoor sky when it was uploaded;
 * - the capture cites a photo this same user uploaded minutes earlier and has not spent before;
 * - the position is one the user could have reached from their previous capture, and the client
 *   did not report the fix as coming from a mock provider.
 *
 * What CONFIRMED still does NOT mean is that the user was physically there. The weather is a public
 * fact anyone can look up; the coordinates and the mock flag are assertions by the client; and the
 * photograph, though genuinely the caller's own upload and genuinely consistent with the sky, is
 * only ever proof that somebody photographed a sky like that one. Closing that last gap needs
 * device attestation, which this server does not have and cannot fake. Do not build anything that
 * treats CONFIRMED as presence.
 *
 * UNCONFIRMED is not an accusation either. It means no XP was awarded, and
 * [UnconfirmedReason] says which of three things happened. An unreachable upstream is no longer
 * one of them: without Open-Meteo there is no phenomenon at all, so that case is a 503 and no row
 * is written.
 */
enum class ValidationStatus { CONFIRMED, UNCONFIRMED }
```

- [ ] **Step 7: Run the test and watch it pass**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.CaptureValidationServiceTest"
```

Expected: 13 passed. The full suite will not compile yet — `WeatherEventController` still calls the old signature. That is Task 6.

- [ ] **Step 8: Commit**

Ask the user first, then:

```bash
git add src/main/kotlin/com/skydex/api/domain/UnconfirmedReason.kt \
        src/main/kotlin/com/skydex/api/domain/ValidationStatus.kt \
        src/main/kotlin/com/skydex/api/models/WeatherEvent.kt \
        src/main/kotlin/com/skydex/api/services/CaptureValidationService.kt \
        src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt
git commit -m "feat: Open-Meteo decides the phenomenon and the photo can contradict it"
```

---

### Task 6: Wire the capture endpoint

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`

**Interfaces:**
- Consumes: everything from Tasks 1-5.
- Produces:
  - `CreateWeatherEventRequest.phenomenon: String?` — accepted and ignored
  - `WeatherEventResponse.unconfirmedReason: String?`

- [ ] **Step 1: Write the failing tests**

Add to `src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`. Read the file first and reuse its existing fixtures — the helper that persists a user, the helper that uploads a photo, and however it stubs Open-Meteo. Add a `@MockBean VisionClient` in the same shape as `PhotoControllerTest` (Task 3, step 1) so uploads succeed.

```kotlin
    @Test
    fun `takes the phenomenon from open-meteo and ignores what the client sent`() {
        // The shipped Android app still sends `phenomenon`. It must be accepted and ignored, not
        // rejected: making it a 400 would break every capture from an app already installed.
        stubForecast(code = 95)   // thunderstorm
        val user = persistUser(email = "ignored@skydex.com")
        val photoUrl = uploadPhotoFor(user)

        val body = mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {"title":"t","description":"d","photoUrl":"$photoUrl",
                     "latitude":-30.0,"longitude":-51.0,
                     "phenomenon":"CLEAR_SKY","locationIsMock":false}
                    """.trimIndent()
                )
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.phenomenon").value("THUNDERSTORM"))
            .andReturn().response.contentAsString

        assertEquals("THUNDERSTORM", objectMapper.readTree(body).get("phenomenon").asText())
    }

    @Test
    fun `accepts a request with no phenomenon field at all`() {
        stubForecast(code = 95)
        val user = persistUser(email = "nophenomenon@skydex.com")
        val photoUrl = uploadPhotoFor(user)

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {"title":"t","description":"d","photoUrl":"$photoUrl",
                     "latitude":-30.0,"longitude":-51.0,"locationIsMock":false}
                    """.trimIndent()
                )
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.phenomenon").value("THUNDERSTORM"))
    }

    @Test
    fun `answers 503 without spending the photo when open-meteo is down`() {
        stubForecastUnavailable()
        val user = persistUser(email = "meteodown@skydex.com")
        val photoUrl = uploadPhotoFor(user)

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {"title":"t","description":"d","photoUrl":"$photoUrl",
                     "latitude":-30.0,"longitude":-51.0,"locationIsMock":false}
                    """.trimIndent()
                )
        )
            .andExpect(status().isServiceUnavailable)

        // The whole retry story depends on this. `consume` runs inside CaptureCommitService,
        // which is never reached, so the photo is still citable.
        val stored = uploadedPhotoRepository.findByFilename(photoUrl.substringAfterLast('/'))!!
        assertNull(stored.consumedAt)
        assertEquals(0, weatherEventRepository.count())
    }

    @Test
    fun `retrying after a 503 succeeds with the same photo`() {
        val user = persistUser(email = "retry@skydex.com")
        val photoUrl = uploadPhotoFor(user)
        val payload = """
            {"title":"t","description":"d","photoUrl":"$photoUrl",
             "latitude":-30.0,"longitude":-51.0,"locationIsMock":false}
        """.trimIndent()

        stubForecastUnavailable()
        mockMvc.perform(
            post("/api/events").header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON).content(payload)
        ).andExpect(status().isServiceUnavailable)

        stubForecast(code = 95)
        mockMvc.perform(
            post("/api/events").header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON).content(payload)
        ).andExpect(status().isCreated)
    }

    @Test
    fun `reports why a capture was not confirmed`() {
        stubForecast(code = 95)
        val user = persistUser(email = "mocked@skydex.com")
        val photoUrl = uploadPhotoFor(user)

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {"title":"t","description":"d","photoUrl":"$photoUrl",
                     "latitude":-30.0,"longitude":-51.0,"locationIsMock":true}
                    """.trimIndent()
                )
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.validationStatus").value("UNCONFIRMED"))
            .andExpect(jsonPath("$.unconfirmedReason").value("MOCK_LOCATION"))
            .andExpect(jsonPath("$.xpAwarded").value(0))
    }

    @Test
    fun `a confirmed capture reports no reason`() {
        stubForecast(code = 95)
        val user = persistUser(email = "confirmed@skydex.com")
        val photoUrl = uploadPhotoFor(user)

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    """
                    {"title":"t","description":"d","photoUrl":"$photoUrl",
                     "latitude":-30.0,"longitude":-51.0,"locationIsMock":false}
                    """.trimIndent()
                )
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.validationStatus").value("CONFIRMED"))
            .andExpect(jsonPath("$.unconfirmedReason").doesNotExist())
    }
```

Add these three private helpers to the class. `stubForecast` and `stubForecastUnavailable` drive a `@MockBean OpenMeteoClient`; if the file already stubs Open-Meteo some other way, adapt these to that mechanism rather than introducing a second one.

```kotlin
    @MockBean
    private lateinit var openMeteo: OpenMeteoClient

    @MockBean
    private lateinit var vision: VisionClient

    @BeforeEach
    fun stubVision() {
        // Uploads have to succeed for any of these tests to reach the capture endpoint. A high
        // outdoor score and a STORM-leaning photo means the default fixture is consistent with
        // the thunderstorm `stubForecast(95)` reports, so nothing is blocked by stage 2 unless a
        // test deliberately asks for it.
        `when`(vision.analyze(any(), any())).thenReturn(
            VisionAnalysis(
                outdoorScore = 0.94,
                phenomenonScores = mapOf(
                    "CLEAR" to 0.04, "CLOUDY" to 0.04, "FOG" to 0.04,
                    "RAIN" to 0.04, "SNOW" to 0.04, "STORM" to 0.80
                ),
                model = "clip-vit-b-32-zeroshot-v1"
            )
        )
    }

    /** Open-Meteo reporting [code] for the hour the capture will be stamped in. */
    private fun stubForecast(code: Int) {
        // Truncated to the hour so the slot is always within CaptureValidationService's 90-minute
        // window of `Instant.now()`, whatever minute the suite happens to run at.
        val slot = LocalDateTime.ofInstant(Instant.now(), ZoneOffset.UTC)
            .truncatedTo(ChronoUnit.HOURS)
            .toString()

        `when`(openMeteo.fetchHourlyForecast(anyDouble(), anyDouble())).thenReturn(
            OpenMeteoResponse(
                latitude = -30.0,
                longitude = -51.0,
                hourly = HourlyData(
                    time = listOf(slot),
                    temperatureCelsius = listOf(21.0),
                    weatherCode = listOf(code),
                    isDay = listOf(1)
                )
            )
        )
    }

    private fun stubForecastUnavailable() {
        `when`(openMeteo.fetchHourlyForecast(anyDouble(), anyDouble())).thenReturn(null)
    }

    /** Uploads a minimal JPEG as [user] and returns the relative path the server assigned. */
    private fun uploadPhotoFor(user: User): String {
        val part = MockMultipartFile(
            "file", "sky.jpg", MediaType.IMAGE_JPEG_VALUE,
            byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte(), 0xD9.toByte())
        )
        val body = mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isCreated)
            .andReturn().response.contentAsString
        return objectMapper.readTree(body).get("photoUrl").asText()
    }
```

Imports these need, on top of whatever the file already has:
```kotlin
import com.skydex.api.dto.HourlyData
import com.skydex.api.dto.OpenMeteoResponse
import com.skydex.api.dto.VisionAnalysis
import com.skydex.api.services.OpenMeteoClient
import com.skydex.api.services.VisionClient
import org.junit.jupiter.api.BeforeEach
import org.mockito.ArgumentMatchers.anyDouble
import org.mockito.Mockito.`when`
import org.mockito.kotlin.any
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.mock.web.MockMultipartFile
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart
import java.time.Instant
import java.time.LocalDateTime
import java.time.ZoneOffset
import java.time.temporal.ChronoUnit
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.WeatherEventControllerTest"
```

Expected: compilation failure.

- [ ] **Step 3: Make the request field optional and add the response field**

In `src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`, replace the `phenomenon` property:
```kotlin
    /**
     * **Accepted and ignored.** Kept in the DTO purely so an already-installed client that still
     * sends it is not answered with a 400.
     *
     * The user no longer chooses a phenomenon: Open-Meteo's weather code for the capture's place
     * and time decides it, and `CaptureValidationService` reads it from there. Removing this field
     * would make every capture from the shipped app fail; validating it would make them fail more
     * politely. Neither is better than ignoring it.
     *
     * This is the same accepted-and-ignored shape `capturedAt`, `latitude` and `longitude` already
     * have on the update handler, and it is pinned by `WeatherEventControllerTest`.
     */
    val phenomenon: String? = null,
```

Remove the now-unused `@field:NotBlank` on it, and drop the `NotBlank` import only if nothing else in the file uses it (`title` and `description` do, so it stays).

Add to `WeatherEventResponse`, with the import `com.fasterxml.jackson.annotation.JsonInclude`:
```kotlin
    val validationStatus: String,
    /**
     * Why [validationStatus] is UNCONFIRMED, or null. Serialised as the enum name so the client
     * can branch on it; the Portuguese copy for each reason lives in the app, not here.
     *
     * Omitted from the body entirely when null, rather than sent as an explicit `null`. The
     * inclusion rule is on this property alone and NOT on `spring.jackson.default-property-
     * inclusion`: that setting is global, and turning it on would silently drop
     * `observedWeatherCode` and every other nullable field from every endpoint in the API — a
     * change to responses nobody asked to change, some of which existing tests assert on.
     */
    @field:JsonInclude(JsonInclude.Include.NON_NULL)
    val unconfirmedReason: String?,
    val xpAwarded: Int
```

and in `from`:
```kotlin
            validationStatus = event.validationStatus.name,
            unconfirmedReason = event.unconfirmedReason?.name,
            xpAwarded = event.xpAwarded
```

- [ ] **Step 4: Rewire the controller**

In `src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`, replace the `create` method's body up to the `captureCommit.commit` call, and add the `PhotoAnalysisService` constructor parameter:

```kotlin
@RestController
@RequestMapping("/api/events")
class WeatherEventController(
    private val events: WeatherEventRepository,
    private val validation: CaptureValidationService,
    private val photoProvenance: PhotoProvenanceService,
    private val photoAnalysis: PhotoAnalysisService,
    private val captureCommit: CaptureCommitService,
    @Value("\${skydex.photos.public-base-url}") private val publicBaseUrl: String,
    private val badges: BadgeService
) {

    @PostMapping
    fun create(
        @AuthenticationPrincipal currentUser: User,
        @Valid @RequestBody request: CreateWeatherEventRequest
    ): ResponseEntity<WeatherEventResponse> {
        // `request.phenomenon` is deliberately not read. See its KDoc: it is accepted so an
        // already-installed client is not 400'd, and ignored because Open-Meteo decides now.

        // One stamp, used for BOTH the validation and the stored row. Reading Instant.now()
        // twice could straddle an hour boundary and validate against a slot the capture is then
        // not recorded in — rare, but an unreproducible "why is this UNCONFIRMED" bug.
        val capturedAt = Instant.now()

        // Checked before validation, on the same stamp: a photo that is not the caller's own,
        // already spent, or expired costs no Open-Meteo call at all. This is a read; the photo is
        // not spent until the commit below.
        val photo = photoProvenance.verify(request.photoUrl, currentUser.id!!, capturedAt)

        // Throws ServiceUnavailableException -> 503 when Open-Meteo cannot answer. Raised here,
        // before `commit`, so the photo is still unspent and the client's retry is free.
        val result = validation.validate(
            latitude = request.latitude,
            longitude = request.longitude,
            capturedAt = capturedAt,
            previous = currentUser.lastKnownPosition(),
            locationIsMock = request.locationIsMock,
            // Cached at upload. Null for a photo predating analysis, which
            // PhotoAuthenticityService reads as "no opinion" rather than as a failed check.
            photoScores = photoAnalysis.deserialise(photo.visionScores)
        )

        val saved = captureCommit.commit(
            WeatherEvent(
                id = null,
                title = request.title,
                description = request.description,
                // Rebuilt from the row that was verified, not echoed from the request.
                photoUrl = "/api/photos/${photo.filename}",
                capturedAt = capturedAt,
                latitude = request.latitude,
                longitude = request.longitude,
                phenomenon = result.phenomenon,
                validationStatus = result.status,
                observedWeatherCode = result.observedWeatherCode,
                xpAwarded = result.xpAwarded,
                unconfirmedReason = result.unconfirmedReason,
                userId = currentUser.id!!
            ),
            photoId = photo.id!!,
            now = capturedAt
        )
```

Leave the badge-sync block and the response construction exactly as they are.

- [ ] **Step 5: Record the reason on a late downgrade**

`CaptureCommitService.commit` can downgrade a CONFIRMED capture when the travel re-check under the row lock disagrees. It must now record why. In `src/main/kotlin/com/skydex/api/services/CaptureCommitService.kt`, add the import and extend that branch:

```kotlin
import com.skydex.api.domain.UnconfirmedReason
```
```kotlin
        if (!reachable) {
            // observedWeatherCode is deliberately left alone. The weather really was observed and
            // really is what the row records; what failed is presence, not the reading, and
            // blanking the code would erase a true fact to make the verdict look tidier.
            event.validationStatus = ValidationStatus.UNCONFIRMED
            event.xpAwarded = 0
            // Overwrites whatever the provisional verdict recorded, and should: this check is the
            // authoritative one, and a row that says PHOTO_CONTRADICTS_WEATHER when what actually
            // sank it was an impossible journey sends the user to fix the wrong thing.
            event.unconfirmedReason = UnconfirmedReason.IMPLAUSIBLE_TRAVEL
        }
```

- [ ] **Step 6: Run the whole backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Other test classes construct `WeatherEvent(...)` and `CreateWeatherEventRequest(...)`; both new fields are defaulted, so they keep compiling. `SkyDexControllerTest`, `ProfileControllerTest` and `FeedControllerTest` all filter on CONFIRMED and are unaffected.

Three kinds of pre-existing failure are expected here, and all three are the tests being right about the old behaviour rather than the new code being wrong:

1. **Tests asserting the stored phenomenon is the one they sent.** They now get whatever `stubForecast` reports. Change the assertion to the phenomenon the stubbed weather code maps to — that is the feature.
2. **Tests asserting a 400 for an unknown phenomenon.** That case no longer exists; the field is ignored. Delete them, and delete the `"unknown phenomenon"` branch from the Android `ErrorPresenter` in Task 9 if it is still there.
3. **Tests asserting UNCONFIRMED when Open-Meteo is unavailable.** That is a 503 now. Rewrite the expectation; `answers 503 without spending the photo` above is the replacement.

If `WeatherEventDtosTest` asserts the exact set of response fields, add `unconfirmedReason` to its expectations.

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt \
        src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt \
        src/main/kotlin/com/skydex/api/services/CaptureCommitService.kt \
        src/main/resources/application.properties \
        src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt
git commit -m "feat: captures take their phenomenon from Open-Meteo"
```

- [ ] **Step 8: Manual smoke test against the real stack**

```bash
cd ~/Documentos/workspace-becker/skydex-vision && docker compose up -d
cd ~/Documentos/workspace-becker/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun
```

In another terminal — register, upload a real sky photo, create a capture with no phenomenon:
```bash
TOKEN=$(curl -s -X POST localhost:3002/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Smoke","email":"smoke@skydex.com","password":"smoke1234"}' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])')

PHOTO=$(curl -s -X POST localhost:3002/api/photos \
  -H "Authorization: Bearer $TOKEN" -F "file=@/path/to/a/real/sky.jpg" \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["photoUrl"])')

curl -s -X POST localhost:3002/api/events -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"title\":\"smoke\",\"description\":\"smoke\",\"photoUrl\":\"$PHOTO\",\"latitude\":-30.03,\"longitude\":-51.22,\"locationIsMock\":false}" \
  | python3 -m json.tool
```

Expected: a capture whose `phenomenon` matches what the sky is actually doing in Porto Alegre right now. Then upload a screenshot and confirm the upload answers 422.

---

### Task 7: Remove the phenomenon selector from the app

**Files:**
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/WeatherEventDto.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureScreen.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`

**Interfaces:**
- Consumes: the backend from Task 6.
- Produces: `CreateWeatherEventRequest(title, description, photoUrl, latitude, longitude, locationIsMock)` — no `phenomenon`. `WeatherEventResponse.unconfirmedReason: String?`.

- [ ] **Step 1: Write the failing test**

In `CaptureViewModelTest.kt`, delete the test asserting that a missing phenomenon blocks submission (search for `MissingPhenomenon` or `Falta escolher o fenômeno`), remove every `viewModel.onPhenomenonSelected(...)` call, and add:

```kotlin
    @Test
    fun `the create request carries no phenomenon`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios sobre a cidade")
        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()
        viewModel.submit()
        advanceUntilIdle()

        // Nothing in the request names a species. The server reads the weather itself, and a
        // field the client fills in is a field a modified client can lie in.
        val sent = gateway.created.single()
        assertEquals("Tempestade", sent.title)
        assertEquals(-30.0346, sent.latitude, 0.0)
    }

    @Test
    fun `submits without the user ever choosing a species`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Céu")
        viewModel.onDescriptionChanged("Sem nuvem nenhuma")
        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()
        viewModel.submit()
        advanceUntilIdle()

        assertTrue(viewModel.state.value.saved)
        assertNull(viewModel.state.value.errorMessage)
    }
```

- [ ] **Step 2: Run the tests and watch them fail**

```bash
cd ~/Documentos/workspace-becker/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*CaptureViewModelTest*"
```

Expected: compilation failure or a `MissingPhenomenon` error state.

- [ ] **Step 3: Update the DTOs**

In `data/remote/dto/WeatherEventDto.kt`, delete the `phenomenon` property from `CreateWeatherEventRequest`, leaving:
```kotlin
data class CreateWeatherEventRequest(
    val title: String,
    val description: String,
    val photoUrl: String,
    val latitude: Double,
    val longitude: Double,
    // No `capturedAt`. The server stamps the capture time; a client that could name the hour
    // could backdate to a past storm and farm the rare badges.
    //
    // No `phenomenon` either, as of the AI validation work, and for a related reason: the server
    // reads the weather from Open-Meteo itself. A species the client names is a species a
    // modified client can name falsely. The backend still accepts the field from older installs
    // and ignores it; this build simply does not send it.
    /**
     * Whether the platform reported this fix as coming from a mock provider. Client-asserted, so
     * it stops a casual mock-GPS app and not a modified client.
     *
     * No default here, deliberately. The server defaults it to `false`, which means a client that
     * omits it disables the check for every real user with nothing failing anywhere.
     */
    val locationIsMock: Boolean
)
```

Add to `WeatherEventResponse`:
```kotlin
    val validationStatus: String,
    /**
     * Enum name — `PHOTO_CONTRADICTS_WEATHER`, `IMPLAUSIBLE_TRAVEL`, `MOCK_LOCATION` — or null.
     *
     * Nullable and absent from the body on a confirmed capture, so this must stay a `String?`.
     * The Portuguese sentence for each value lives in `CaptureRewardOverlay`, not here: this
     * carries the signal, the UI owns the words.
     */
    val unconfirmedReason: String? = null,
    val xpAwarded: Int
```

- [ ] **Step 4: Strip the ViewModel**

In `ui/capture/CaptureViewModel.kt`:
- delete `val phenomenon: String? = null` from `CaptureUiState`
- delete the `MissingPhenomenon` value
- delete `fun onPhenomenonSelected(...)`
- delete the `current.phenomenon == null -> MissingPhenomenon` branch from `submit`'s `when`
- delete `phenomenon = current.phenomenon!!,` from the `CreateWeatherEventRequest` construction
- delete `phenomenon = null,` from `startNewCapture`

- [ ] **Step 5: Strip the screen**

In `ui/capture/CaptureScreen.kt`:
- delete the `SPECIES` list and its KDoc
- delete the `onPhenomenonSelected` parameter from `CaptureContent` and from the call site in the screen composable
- delete the `Text("Qual fenômeno?")` heading, the `FlowRow` and the `FilterChip` block, together with the now-unused `FlowRow` and `FilterChip` imports
- delete `phenomenon = "THUNDERSTORM"` from `previewState` and `onPhenomenonSelected = {}` from `CapturePreviewHost`

In its place, put one line telling the user what now happens. Insert where the heading was:
```kotlin
        Text(
            text = "O SkyDex identifica o fenômeno sozinho, comparando sua foto com o clima real do lugar.",
            style = MaterialTheme.typography.bodyMedium,
            color = SkyDexPalette.colors.textSecondary
        )
```

Without this the screen simply loses a section and the user is never told the rule changed — the reveal at the end would read as the app having decided something arbitrary.

- [ ] **Step 6: Run the tests and the compile check**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*CaptureViewModelTest*"
```

Expected: both green.

- [ ] **Step 7: Commit**

Ask the user first, then:

```bash
git add app/src/main/java/com/example/skydex/data/remote/dto/WeatherEventDto.kt \
        app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt \
        app/src/main/java/com/example/skydex/ui/capture/CaptureScreen.kt \
        app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt
git commit -m "feat: the app no longer asks which phenomenon was photographed"
```

---

### Task 8: Upload the photo as soon as it is taken

**Files:**
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`

**Interfaces:**
- Consumes: `CaptureGateway.uploadPhoto`.
- Produces: no new public API. `onPhotoTaken` now starts the upload; `submit` awaits it.

- [ ] **Step 1: Write the failing tests**

Add to `CaptureViewModelTest.kt`:
```kotlin
    @Test
    fun `uploads the photo as soon as it is taken`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()

        // Before submit is ever called. The user spends the next several seconds typing, and the
        // model's forward pass happens inside that time instead of after it.
        assertEquals(1, gateway.uploads)
        assertNotNull(viewModel.state.value.uploadedPhotoUrl)
    }

    @Test
    fun `submit reuses the eager upload instead of sending a second copy`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()
        viewModel.onTitleChanged("t")
        viewModel.onDescriptionChanged("d")
        viewModel.submit()
        advanceUntilIdle()

        assertEquals(1, gateway.uploads)
        assertTrue(viewModel.state.value.saved)
    }

    @Test
    fun `submit waits for an upload still in flight`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val gate = CompletableDeferred<Unit>()
        gateway.uploadGate = gate
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onPhotoTaken(jpeg())
        viewModel.onTitleChanged("t")
        viewModel.onDescriptionChanged("d")
        viewModel.submit()
        advanceUntilIdle()

        // Still waiting on the upload, so nothing has been created and the button is still busy.
        assertTrue(viewModel.state.value.submitting)
        assertTrue(gateway.created.isEmpty())

        gate.complete(Unit)
        advanceUntilIdle()

        assertTrue(viewModel.state.value.saved)
        assertEquals(1, gateway.uploads)
    }

    @Test
    fun `an eager upload that fails surfaces its message without blocking a retake`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        gateway.uploadResult = Result.failure(
            HttpException(Response.error<Any>(422, """{"error":"This photo does not look like the sky"}"""
                .toResponseBody("application/json".toMediaType())))
        )
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()

        assertNotNull(viewModel.state.value.errorMessage)
        assertNull(viewModel.state.value.uploadedPhotoUrl)
        // Not submitting, not saved — the user can simply take another photo.
        assertFalse(viewModel.state.value.submitting)
    }

    @Test
    fun `a retake abandons the previous upload`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val gate = CompletableDeferred<Unit>()
        gateway.uploadGate = gate
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        val first = jpeg("first.jpg")
        viewModel.onPhotoTaken(first)
        val second = jpeg("second.jpg")
        viewModel.onPhotoTaken(second)
        gate.complete(Unit)
        advanceUntilIdle()

        // The path from the abandoned upload must never be cached against the photo on screen:
        // saving it would file the capture under an image the user cannot see any more.
        assertEquals(second, viewModel.state.value.photoFile)
    }
```

Extend `FakeCaptureGateway` in the same file with:
```kotlin
        var uploads = 0
        var uploadResult: Result<String>? = null
        var uploadGate: CompletableDeferred<Unit>? = null
```
and make `uploadPhoto` increment `uploads`, await `uploadGate` if set, and return `uploadResult` when set. Follow the fake's existing style — read it before editing.

- [ ] **Step 2: Run the tests and watch them fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*CaptureViewModelTest*"
```

Expected: `uploads` is 0 after `onPhotoTaken`.

- [ ] **Step 3: Implement the eager upload**

In `ui/capture/CaptureViewModel.kt`, add the imports:
```kotlin
import kotlinx.coroutines.CancellationException
import kotlinx.coroutines.Deferred
import kotlinx.coroutines.async
```

Add the holder and the field just below `profileBefore`:
```kotlin
    /**
     * An upload started by [onPhotoTaken], paired with the file it is carrying.
     *
     * The file is held alongside the job and not inferred from state, because state moves: a
     * retake replaces [CaptureUiState.photoFile] while this job is still in flight, and matching
     * on identity is what stops [submit] adopting a path that belongs to a photo the user has
     * already replaced.
     */
    private class PendingUpload(val file: File, val job: Deferred<Result<String>>)

    private var pendingUpload: PendingUpload? = null
```

Replace `onPhotoTaken`:
```kotlin
    /**
     * Records the new photo and starts uploading it immediately.
     *
     * ## Why the upload does not wait for Save
     *
     * The backend runs the vision model inside `POST /api/photos`, and a photo it does not believe
     * is the sky comes back 422. Uploading at Save time would put that rejection *after* the user
     * had written a title and a description — the moment they can least afford to be told to
     * start over. Firing here puts the round-trip inside the seconds they spend typing, so the
     * answer is usually already in by the time they reach the button.
     *
     * The failure is shown but nothing else happens: no navigation, no blocked form, no retry
     * loop. The recovery is "take another photo", which the screen already offers.
     */
    fun onPhotoTaken(file: File) {
        // The previous job is not cancelled. It is already in flight, cancelling buys nothing the
        // identity check below does not, and a cancelled `async` whose result is never awaited
        // reports as an unhandled failure in some coroutine configurations.
        _state.update { it.copy(photoFile = file, uploadedPhotoUrl = null, errorMessage = null) }

        val job = viewModelScope.async { captures.uploadPhoto(file) }
        pendingUpload = PendingUpload(file, job)

        viewModelScope.launch {
            val result = try {
                job.await()
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                Result.failure(e)
            }

            // Every write below is conditional on this still being the photo on screen. Between
            // the launch and here sits a whole network round-trip the user can retake during.
            result
                .onSuccess { url ->
                    _state.update { if (it.photoFile == file) it.copy(uploadedPhotoUrl = url) else it }
                }
                .onFailure { failure ->
                    logWarning(LOG_TAG, "Eager photo upload failed", failure)
                    _state.update {
                        if (it.photoFile == file) {
                            it.copy(errorMessage = failure.toUiMessage(ErrorContext.PHOTO_UPLOAD))
                        } else {
                            it
                        }
                    }
                }
        }
    }
```

In `submit`, replace the block that begins `val photoUrl = current.uploadedPhotoUrl ?: run {`:
```kotlin
            // Three ways to have a photo path here, in order of preference: one the eager upload
            // already finished, one it is still working on, or — only if there is no job at all,
            // which a normal flow cannot produce — a fresh upload started right now.
            val photoUrl = current.uploadedPhotoUrl ?: run {
                val pending = pendingUpload?.takeIf { it.file == photoFile }
                val uploaded = try {
                    pending?.job?.await() ?: captures.uploadPhoto(photoFile)
                } catch (e: CancellationException) {
                    throw e
                } catch (e: Exception) {
                    Result.failure(e)
                }

                uploaded.exceptionOrNull()?.let { failure ->
                    _state.update {
                        it.copy(
                            submitting = false,
                            errorMessage = failure.toUiMessage(ErrorContext.PHOTO_UPLOAD)
                        )
                    }
                    return@launch
                }

                // Conditional, because this write lands after a suspension the user can act
                // during: caching a path that belongs to a replaced file would file the next
                // attempt under a picture the user cannot see any more.
                uploaded.getOrThrow().also { url ->
                    _state.update { if (it.photoFile == photoFile) it.copy(uploadedPhotoUrl = url) else it }
                }
            }
```

Clear the holder in `startNewCapture`, next to the state reset:
```kotlin
    fun startNewCapture() {
        pendingUpload = null
        _state.update {
            it.copy(
                title = "",
                description = "",
                photoFile = null,
                uploadedPhotoUrl = null,
                submitting = false,
                saved = false,
                reward = null,
                errorMessage = null
            )
        }
    }
```

Add the tag constant to the class's companion, or create one if there is none:
```kotlin
    private companion object {
        const val LOG_TAG = "CaptureViewModel"
        const val CONFIRMED_STATUS = "CONFIRMED"
    }
```
If `CONFIRMED_STATUS` already exists elsewhere in the file, leave it where it is and add only `LOG_TAG`.

- [ ] **Step 4: Run the tests and watch them pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*CaptureViewModelTest*"
```

Expected: green, including every pre-existing test in the class.

- [ ] **Step 5: Commit**

Ask the user first, then:

```bash
git add app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt \
        app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt
git commit -m "feat: upload the photo the moment it is taken"
```

---

### Task 9: Copy for 422 and 503

**Files:**
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/common/ErrorPresenter.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/common/ErrorPresenterTest.kt`

**Interfaces:**
- Consumes: `ErrorContext`.
- Produces: no new API. `toUiMessage` gains two status branches.

- [ ] **Step 1: Write the failing test**

Add to `ErrorPresenterTest.kt` (create it in that package if the file does not exist, following the conventions of the nearest existing test):
```kotlin
    private fun httpError(code: Int, body: String = """{"error":"x"}"""): Throwable =
        HttpException(Response.error<Any>(code, body.toResponseBody("application/json".toMediaType())))

    @Test
    fun `a 422 on upload tells the user to point the camera at the sky`() {
        val message = httpError(422).toUiMessage(ErrorContext.PHOTO_UPLOAD)

        assertTrue(message.body.contains("céu"), message.body)
        // Not the 5xx copy: this is not an outage and telling the user "não é você" would send
        // them to wait for a fix that is never coming.
        assertFalse(message.title.contains("fora do ar"))
    }

    @Test
    fun `a 503 on upload promises that waiting helps`() {
        val message = httpError(503).toUiMessage(ErrorContext.PHOTO_UPLOAD)

        assertEquals("Tentar de novo", message.actionLabel)
        assertTrue(message.body.contains("instantes"), message.body)
    }

    @Test
    fun `a 503 on save says the photo is safe`() {
        // This is the whole point of the copy: the backend does not spend the photo on a 503, so
        // the honest instruction is "press Save again", not "take another photo".
        val message = httpError(503).toUiMessage(ErrorContext.CAPTURE_SAVE)

        assertTrue(message.body.contains("foto"), message.body)
        assertEquals("Tentar de novo", message.actionLabel)
    }

    @Test
    fun `a 500 is still reported as an outage, not as a 503`() {
        val message = httpError(500).toUiMessage(ErrorContext.CAPTURE_SAVE)

        assertTrue(message.title.contains("fora do ar"), message.title)
    }
}
```

- [ ] **Step 2: Run the test and watch it fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*ErrorPresenterTest*"
```

Expected: the 422 case falls through to the generic upload copy and the 503 case reads as an outage.

- [ ] **Step 3: Add the branches**

In `ui/common/ErrorPresenter.kt`, inside the `when` in `toUiMessage`, add the two branches **above** `status >= 500`:
```kotlin
        status == 413 -> PhotoTooLarge
        status == 422 -> unprocessable(context)
        // Before `status >= 500`, and the order is the whole point: 503 is not an outage report,
        // it is "an upstream we need is briefly unavailable, and nothing you did was lost".
        status == 503 -> unavailable(context)
        status == 400 -> badRequest(context, backend)
        status >= 500 -> ServerDown
```

Add the two functions next to `badRequest`:
```kotlin
/**
 * 422 — the request was fine, the content was not.
 *
 * One endpoint produces it: `POST /api/photos`, when the vision model does not believe the picture
 * is an outdoor sky. Distinct from a 400 on purpose, because the instructions differ. A 400 means
 * re-check what you typed; this means point the camera somewhere else.
 */
private fun unprocessable(context: ErrorContext): UiMessage = when (context) {
    ErrorContext.PHOTO_UPLOAD -> UiMessage(
        title = "Essa foto não parece o céu",
        body = "Aponte a câmera para cima e tire outra foto do fenômeno.",
        tone = Tone.NOTICE
    )

    else -> UiMessage(
        title = "Não deu para aceitar isso",
        body = "Confira o que você enviou e tente de novo.",
        tone = Tone.NOTICE
    )
}

/**
 * 503 — an upstream the server needs did not answer.
 *
 * Two sources, and the copy differs because what survives differs. On upload nothing was written
 * at all. On save, the backend raises the 503 *before* the photo is spent — `PhotoProvenanceService
 * .consume` runs inside the commit transaction, which is never reached — so the photo is still
 * citable and pressing Save again is genuinely all that is needed. Saying "tire outra foto" here
 * would send the user to redo work that was never lost.
 */
private fun unavailable(context: ErrorContext): UiMessage = when (context) {
    ErrorContext.PHOTO_UPLOAD -> UiMessage(
        title = "Não conseguimos analisar a foto agora",
        body = "Tente de novo em instantes.",
        tone = Tone.NOTICE,
        actionLabel = RETRY
    )

    ErrorContext.CAPTURE_SAVE -> UiMessage(
        title = "Não conseguimos conferir o clima agora",
        body = "Sua foto está salva. Toque em salvar de novo em instantes.",
        tone = Tone.NOTICE,
        actionLabel = RETRY
    )

    else -> UiMessage(
        title = "Esse serviço está indisponível agora",
        body = "Tente de novo em instantes.",
        tone = Tone.NOTICE,
        actionLabel = RETRY
    )
}
```

- [ ] **Step 4: Run the tests and watch them pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*ErrorPresenterTest*"
```

Expected: green.

- [ ] **Step 5: Commit**

Ask the user first, then:

```bash
git add app/src/main/java/com/example/skydex/ui/common/ErrorPresenter.kt \
        app/src/test/java/com/example/skydex/ui/common/ErrorPresenterTest.kt
git commit -m "feat: copy for a rejected photo and an unavailable upstream"
```

---

### Task 10: Keep unconfirmed captures and say why

**Files:**
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureGateway.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/CaptureRepository.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/components/CaptureRewardOverlay.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/captures/MyCapturesScreen.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/components/CaptureRewardTest.kt`

**Interfaces:**
- Consumes: `WeatherEventResponse.unconfirmedReason`.
- Produces: `CaptureReward.unconfirmedReason: String?`; `reasonCopyFor(reason: String?): String` in `CaptureRewardOverlay.kt`.

- [ ] **Step 1: Write the failing tests**

In `CaptureViewModelTest.kt`, delete the tests asserting that an unconfirmed capture is deleted (search for `discard` or `delete`), and add:
```kotlin
    @Test
    fun `keeps an unconfirmed capture instead of deleting it`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        gateway.createResult = Result.success(
            aCaptureResponse(validationStatus = "UNCONFIRMED", unconfirmedReason = "PHOTO_CONTRADICTS_WEATHER")
        )
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("t")
        viewModel.onDescriptionChanged("d")
        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()
        viewModel.submit()
        advanceUntilIdle()

        // The model may be the one that is wrong. Silently destroying the user's photograph
        // because of it is the wrong answer, and there is nothing they could do about it.
        assertEquals(0, gateway.deletes)
        assertTrue(viewModel.state.value.saved)
    }

    @Test
    fun `carries the reason into the reward overlay`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        gateway.createResult = Result.success(
            aCaptureResponse(validationStatus = "UNCONFIRMED", unconfirmedReason = "MOCK_LOCATION")
        )
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0, -51.0) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("t")
        viewModel.onDescriptionChanged("d")
        viewModel.onPhotoTaken(jpeg())
        advanceUntilIdle()
        viewModel.submit()
        advanceUntilIdle()

        val reward = viewModel.state.value.reward!!
        assertFalse(reward.confirmed)
        assertEquals("MOCK_LOCATION", reward.unconfirmedReason)
    }
```

Add a `aCaptureResponse(validationStatus: String, unconfirmedReason: String? = null)` helper to the test file if there is not already one, and a `deletes` counter to `FakeCaptureGateway` — keep the counter for exactly as long as it takes this test to prove it stays at zero, then remove `delete` from the fake along with the interface method in step 3.

In `CaptureRewardTest.kt` (or wherever `rewardSignals` is tested), add:
```kotlin
    @Test
    fun `each reason gets its own sentence`() {
        assertTrue(reasonCopyFor("PHOTO_CONTRADICTS_WEATHER").contains("foto"))
        assertTrue(reasonCopyFor("MOCK_LOCATION").contains("localização"))
        assertTrue(reasonCopyFor("IMPLAUSIBLE_TRAVEL").contains("distante"))
    }

    @Test
    fun `an unknown or absent reason still says something useful`() {
        // A reason from a newer backend must not render as an empty line or as the enum name.
        assertTrue(reasonCopyFor(null).isNotBlank())
        assertTrue(reasonCopyFor("SOMETHING_NEW").isNotBlank())
        assertFalse(reasonCopyFor("SOMETHING_NEW").contains("SOMETHING_NEW"))
    }
```

- [ ] **Step 2: Run the tests and watch them fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "*CaptureViewModelTest*" --tests "*CaptureRewardTest*"
```

Expected: `deletes` is 1, and `reasonCopyFor` does not exist.

- [ ] **Step 3: Remove the auto-delete**

In `ui/capture/CaptureViewModel.kt`:
- delete the whole `discardUnconfirmed` function and its KDoc
- delete the `if (!reward.confirmed) discardUnconfirmed(event.id)` line from the `onSuccess` block
- keep the `logWarning` constructor parameter — Task 8 gave it a new caller. Update its KDoc: it now documents the eager upload's failure path rather than the discard's.

In `ui/capture/CaptureGateway.kt`, delete the `delete` method and its KDoc.

In `data/repository/CaptureRepository.kt`, drop `override` from `delete` and keep the method — it is still the API surface for a future manual delete, and its Retrofit-204 comment is worth preserving:
```kotlin
    /**
     * `DELETE api/events/{id}`.
     *
     * Currently uncalled. It lost its only caller when the capture screen stopped destroying
     * unconfirmed captures behind the user's back — an unconfirmed capture is now kept and
     * explained rather than deleted. Kept because the endpoint exists and a user-driven delete on
     * Meus Registros is the obvious next use.
     *
     * `deleteCapture` returns the raw [retrofit2.Response], not `Unit`: Retrofit 2.9.0 throws
     * KotlinNullPointerException trying to map the backend's empty 204 body onto a non-null `Unit`
     * return type. That means an unsuccessful status arrives here as a perfectly normal value, so
     * [resultOf] alone would report a 403 or a 404 as a *successful* delete.
     */
    suspend fun delete(id: String): Result<Unit> = resultOf {
        val response = api.deleteCapture(id)
        if (!response.isSuccessful) throw HttpException(response)
    }
```

- [ ] **Step 4: Carry the reason to the overlay**

In `ui/components/CaptureRewardOverlay.kt`, add to `CaptureReward`:
```kotlin
data class CaptureReward(
    val phenomenonName: String,
    val rarity: String,
    val confirmed: Boolean,
    val xpAwarded: Int,
    /**
     * The backend's `unconfirmedReason` enum name, or null.
     *
     * Held as the raw name rather than as a sealed type on purpose: a backend that adds a fourth
     * reason must not crash a client that has only three, and [reasonCopyFor] turns an unknown
     * value into a sentence rather than into an exception.
     */
    val unconfirmedReason: String? = null,
    val bonus: CaptureRewardBonus? = null
)
```

Add the copy function next to `rewardSignals`, and for the same reason it is a pure function: the JVM test source set has no Compose runtime, so the mapping has to be assertable without rendering.
```kotlin
/**
 * The sentence explaining an unconfirmed capture.
 *
 * Each reason gets its own, because each implies a different next action — one says the photograph
 * did not match, one says the phone's position was not believed, one says the journey was not
 * possible. A single "não foi confirmado" would leave the user with nothing to do differently.
 *
 * The fallback is deliberately vague rather than technical: an unknown reason means a newer backend
 * than this build, and printing the enum name would put `PHOTO_CONTRADICTS_WEATHER` on a
 * Portuguese screen.
 */
fun reasonCopyFor(reason: String?): String = when (reason) {
    "PHOTO_CONTRADICTS_WEATHER" ->
        "Sua foto não combinou com o clima registrado nesse lugar e horário. " +
            "A captura fica guardada, mas sem XP."

    "MOCK_LOCATION" ->
        "O aparelho informou que a localização veio de um simulador. " +
            "Desative o app de localização falsa e tente de novo."

    "IMPLAUSIBLE_TRAVEL" ->
        "Esse ponto ficou distante demais da sua captura anterior para o tempo que passou."

    else ->
        "Não conseguimos confirmar essa captura. Ela fica guardada, mas sem XP."
}
```

In the overlay's unconfirmed branch, render `reasonCopyFor(reward.unconfirmedReason)` where the generic sentence is today.

- [ ] **Step 5: Populate it**

In `CaptureViewModel.rewardFrom`:
```kotlin
    private fun rewardFrom(event: WeatherEventResponse) = CaptureReward(
        phenomenonName = event.phenomenonName,
        rarity = event.rarity,
        confirmed = event.validationStatus.equals(CONFIRMED_STATUS, ignoreCase = true),
        xpAwarded = event.xpAwarded,
        unconfirmedReason = event.unconfirmedReason
    )
```

- [ ] **Step 6: Mark unconfirmed rows in the list**

In `ui/captures/MyCapturesScreen.kt`, find where a capture row is composed and add a badge when the status is not confirmed. Match the file's existing spacing, colour and typography conventions rather than importing new ones:
```kotlin
            if (!capture.validationStatus.equals("CONFIRMED", ignoreCase = true)) {
                // Unconfirmed captures are kept now instead of being deleted behind the user's
                // back, so the list has to say which ones they are. Without this the user sees a
                // capture worth no XP sitting among ones that are, with nothing to explain it.
                Text(
                    text = "Não confirmada",
                    style = MaterialTheme.typography.labelSmall,
                    color = SkyDexPalette.colors.textSecondary
                )
            }
```

- [ ] **Step 7: Run the full Android suite**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest
```

Expected: green.

- [ ] **Step 8: Run the full backend suite one more time**

```bash
cd ~/Documentos/workspace-becker/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: green.

- [ ] **Step 9: End-to-end on the device**

```bash
cd ~/Documentos/workspace-becker/skydex-vision && docker compose up -d
cd ~/Documentos/workspace-becker/SkyDex-backend && JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun &
cd ~/Documentos/workspace-becker/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:installDebug
```

With the phone connected, walk these four:
1. Photograph the sky → the capture screen shows no species chips → Save → the overlay reveals a phenomenon that matches the weather outside
2. Photograph a wall → the "não parece o céu" message appears **while you are still typing**, not after Save
3. `docker compose stop skydex-vision` → photograph the sky → the upload reports the 503 copy
4. `docker compose start skydex-vision`, photograph the sky, and check Meus Registros shows any unconfirmed capture with its badge rather than having deleted it

- [ ] **Step 10: Commit**

Ask the user first, then:

```bash
git add app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt \
        app/src/main/java/com/example/skydex/ui/capture/CaptureGateway.kt \
        app/src/main/java/com/example/skydex/data/repository/CaptureRepository.kt \
        app/src/main/java/com/example/skydex/ui/components/CaptureRewardOverlay.kt \
        app/src/main/java/com/example/skydex/ui/captures/MyCapturesScreen.kt \
        app/src/test/java/com/example/skydex/
git commit -m "feat: keep unconfirmed captures and explain why"
```

---

## Follow-ups, deliberately out of scope

Named here so they are decisions rather than omissions.

1. **`BadgeService:66` counts UNCONFIRMED captures.** That count has been artificially near-zero because the client deleted the rows; it will now count for real. Read what that badge is supposed to mean and decide whether the threshold still makes sense. This plan does not change it, because changing a badge's meaning is a product call and not a side effect of a validation change.
2. **"Discordo dessa análise."** Now that a reason is visible and a capture survives, there is a natural place for a dispute action. Deliberately not built: it needs a moderation story that does not exist yet.
3. **Orphaned JPEG cleanup** (pre-existing backlog item #13). Unchanged by this work — a 422 or 503 at upload leaves nothing behind, since analysis now runs before anything is written.
4. **Phase 5 of the spec — retraining on real captures.** Needs production traffic. The label comes from Open-Meteo and UNCONFIRMED captures are included, which is what keeps the model from reinforcing its own errors.

## Done when

- `JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test` is green in `SkyDex-backend`
- `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest` is green in `SkyDex---frontend`
- The four device walkthroughs in Task 10 step 9 all behave as written
- `skydex-vision`'s golden-set regression still meets the 2% false-positive bar at whatever `skydex.vision.outdoor-min` is configured to
