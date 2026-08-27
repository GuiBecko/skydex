# SkyDex MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the current pre-MVP SkyDex prototype into a shippable MVP where a user photographs a real meteorological phenomenon with their phone camera, the backend validates it against live Open-Meteo data for that GPS position and time, the validated capture is added to the user's SkyDex collection for XP, unlocks achievement badges on their profile, and appears in a feed shared with accepted friends.

**Architecture:** Kotlin/Spring Boot 3.2.4 REST API backed by PostgreSQL (PostGIS image), JWT bearer auth, talking to a single-module Jetpack Compose Android app over Retrofit/Gson. The backend moves from returning JPA entities directly to a DTO boundary (which is also what stops the current password-hash leak), and gains two pure catalogs that are the single source of truth for progression: `Phenomenon` (species, rarity, XP) and `Achievement` (badge unlock rules). The Android app moves from `MainActivity`-held string navigation state to Navigation Compose + per-screen ViewModels over a thin repository layer, with the JWT persisted in DataStore and injected by an OkHttp interceptor.

**Tech Stack:** Kotlin 1.9.23 / Java 17 / Spring Boot 3.2.4 / Spring Data JPA / Spring Security / auth0 java-jwt 4.4.0 / PostgreSQL 17 (postgis/postgis image) / Testcontainers · Kotlin 2.2.10 / AGP 9.3.0 / Jetpack Compose (BOM 2026.02.01) / Material 3 / Retrofit 2.9.0 + Gson / Coil 2.6.0 / DataStore Preferences / Navigation Compose / Google Play Services Location

---

## Scope: what is and is not in this MVP

**In scope** — the core loop must work end to end:

1. Register, log in, and *stay* logged in across app restarts.
2. See which phenomena are happening right now at the device's real GPS position.
3. Photograph a phenomenon with the device camera and upload the photo.
4. Backend validates the claimed phenomenon against Open-Meteo for that position and time.
5. Confirmed captures award XP and unlock a species in the SkyDex collection.
6. Milestones unlock achievement badges (one-to-many on the user) shown on a profile screen.
7. Send/accept friend requests and see friends' captures in a shared feed.
8. Backend no longer leaks password hashes and no longer lets any logged-in user delete everyone's data.

**Explicitly out of scope** (do not build these; they are the post-MVP backlog):
Flyway migrations, S3/object-storage photo hosting, in-app CameraX viewfinder, push notifications, comments/likes, leaderboards, avatars, offline mode, map view, capture editing, refresh tokens, rate limiting.

---

## Global Constraints

These apply to **every** task. Each task's requirements implicitly include this section.

- **Commits are pre-authorized for this execution; pushes are not.** The user granted blanket approval on 2026-08-07 for the commits in this plan, so every "Commit" step runs `git commit` directly with the message given. **Never run `git push`, never open a PR, and never merge** — publishing stays the user's decision alone. This overrides `CLAUDE.md`'s "never commit by yourself" for this plan only; outside it, ask first.
- **Work happens directly on `master` in both repos.** The user was told `master` is the default branch in `SkyDex-backend` and `SkyDex---frontend` and chose it deliberately. Do not create, switch, rebase, or delete branches.
- **All new code, identifiers, comments, log messages and API paths in English.** The existing codebase is partly Portuguese; this plan renames the parts it touches. Do not introduce new Portuguese identifiers. User-facing UI copy stays in Portuguese (pt-BR) — that is product language, not code.
- **Backend JDK is 17.** `JAVA_HOME=$HOME/.jdks/ms-17.0.20`. Backend `sourceCompatibility` stays `JavaVersion.VERSION_17`.
- **Android Gradle JDK is 21.** `JAVA_HOME=$HOME/android-studio/jbr`. The Android toolchain is pinned to 21 in `gradle/gradle-daemon-jvm.properties`.
- **Backend Kotlin is 1.9.23, Spring Boot 3.2.4.** Do not bump these during the MVP; version bumps are their own change.
- **Android `minSdk` is 26, `targetSdk`/`compileSdk` 36.** Every API used must be available on API 26 (`java.time` is, via desugaring-free API 26 support).
- **Dev database** is the Docker container named `skydex-db` (`postgis/postgis`, `POSTGRES_DB=skydex`, user `guilherme_becker`, port 5432). Start it with `docker start skydex-db` before running the app.
- **Secrets live in `SkyDex-backend/.env`** (already gitignored). Never hardcode `TOKEN_JWT_SECRET`, DB credentials, or paths to them into tracked files.
- **`spring.jpa.hibernate.ddl-auto` stays `update` in dev and `create-drop` in test.** No migration tool this MVP. Task 2 includes a one-time destructive dev-DB reset; pre-MVP data is disposable. Adding Flyway is the first post-MVP item.
- **Test commands** (run from the respective repo root):
  - Backend: `JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test`
  - Android unit: `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest`
  - Android compile check: `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin`
  - Backend tests need Docker running (Testcontainers). Android runs on a phone connected over USB (`~/Android/Sdk/platform-tools/adb devices` must list it).
- **Follow TDD.** Every task writes the failing test first, watches it fail, then implements. Do not write implementation before a red test.

---

## Baseline: verified current state

Confirmed by inspection and by compiling both repos on 2026-08-07:

- `SkyDex-backend` `./gradlew compileKotlin` → **BUILD SUCCESSFUL** (3 warnings: unused `eventos`/`users` vars, an always-true null check).
- `SkyDex---frontend` `./gradlew :app:compileDebugKotlin` → **BUILD SUCCESSFUL**.
- `./gradlew` in **both** repos lacks the executable bit — `./gradlew` fails with "Permissão negada". Task 1 fixes this.
- Backend tests are `@SpringBootTest` against the **real dev database** with `deleteAll()` in `@BeforeEach`. They cannot run without the dev Postgres up, and when they do run they destroy dev data. Task 1 fixes this.
- `GET /api/users` returns `User` JPA entities, so every response includes the BCrypt `password` hash. Task 2 fixes this.
- `DELETE /api/eventos` and `DELETE /api/users` (no id) wipe every row and are reachable by any authenticated user. `PUT`/`DELETE /api/eventos/{id}` have no ownership check. Task 3 fixes this.
- There is no camera, no photo storage, no geolocation, no phenomenon catalog, no XP, and no friends. `urlFoto` is a URL the user types by hand and `NearEvents.kt` hardcodes `lat = -23.55, lon = -46.63`.
- The Android app holds the JWT in `remember { mutableStateOf("") }` inside `MainActivity`, so the session dies on process death and on rotation. Tasks 4 and 5 fix this.

---

## File Structure

### Backend — `SkyDex-backend/src/main/kotlin/com/skydex/api/`

| File | Responsibility | Task |
|---|---|---|
| `SkyDexApplication.kt` | Spring Boot entry point (unchanged) | — |
| `config/SecurityConfig.kt` | Filter chain, public routes, stateless sessions, 401 entry point | 3 |
| `config/WebConfig.kt` | Static resource handler serving uploaded photos | 7 |
| `security/SecurityFilter.kt` | Bearer token → authenticated principal; swallows invalid tokens | 3 |
| `security/TokenService.kt` | JWT issue/verify (unchanged behaviour, English comments) | 2 |
| `models/User.kt` | `users` entity — English fields, unique email | 2 |
| `models/WeatherEvent.kt` | `weather_events` entity (renamed from `EventoMetereologico`) | 2, 6, 12 |
| `models/Friendship.kt` | `friendships` entity | 15 |
| `domain/Phenomenon.kt` | Species catalog enum: display name, rarity, alert level, WMO codes | 11 |
| `domain/Rarity.kt` | Rarity tiers and their XP values | 11 |
| `domain/ValidationStatus.kt` | `CONFIRMED` / `UNCONFIRMED` | 12 |
| `domain/Levels.kt` | Pure `levelFor(xp)` / `xpToNextLevel(xp)` functions | 13 |
| `domain/Achievement.kt` | Badge catalog: names, descriptions, unlock rules | 18 |
| `models/UserBadge.kt` | `user_badges` entity — the many side of user-to-badges | 18 |
| `repositories/UserBadgeRepository.kt` | Badge queries | 18 |
| `dto/ProfileDtos.kt` | `BadgeResponse`, `ProfileResponse` | 18 |
| `services/BadgeService.kt` | Idempotent badge awarding | 18 |
| `services/ProfileService.kt` | Profile assembly (identity + progression + badges) | 18 |
| `repositories/UserRepository.kt` | User queries | 2 |
| `repositories/WeatherEventRepository.kt` | Event queries, XP sum, feed page | 2, 13, 16 |
| `repositories/FriendshipRepository.kt` | Friendship queries | 15 |
| `dto/AuthDtos.kt` | `LoginRequest`, `RegisterRequest`, `LoginResponse` | 2 |
| `dto/UserDtos.kt` | `UserResponse`, `UpdateProfileRequest` | 2 |
| `dto/WeatherEventDtos.kt` | `CreateWeatherEventRequest`, `WeatherEventResponse` | 2, 6, 13 |
| `dto/NearbyDtos.kt` | `NearbyPhenomenonResponse` | 2, 11 |
| `dto/PhotoDtos.kt` | `PhotoUploadResponse` | 7 |
| `dto/SkyDexDtos.kt` | `SkyDexEntryResponse`, `SkyDexResponse` | 13 |
| `dto/FriendDtos.kt` | `FriendRequestBody`, `FriendResponse`, `FriendRequestResponse` | 15 |
| `dto/ErrorResponse.kt` | Uniform `{ "error": "..." }` body | 2 |
| `controllers/AuthController.kt` | `/auth/register`, `/auth/login` | 2 |
| `controllers/UserController.kt` | `/api/users/me` read/update/delete | 3 |
| `controllers/WeatherEventController.kt` | `/api/events` create, mine, get, update, delete | 2, 3, 6, 12 |
| `controllers/WeatherController.kt` | `/api/weather/nearby` | 2, 11 |
| `controllers/PhotoController.kt` | `POST /api/photos` multipart upload | 7 |
| `controllers/SkyDexController.kt` | `GET /api/skydex` | 13 |
| `controllers/FriendController.kt` | `/api/friends*` | 15 |
| `controllers/FeedController.kt` | `GET /api/feed` | 16 |
| `controllers/GlobalExceptionHandler.kt` | Validation/domain exceptions → `ErrorResponse` | 3 |
| `errors/DomainExceptions.kt` | `NotFoundException`, `ForbiddenException`, `ConflictException` | 3 |
| `services/OpenMeteoClient.kt` | Raw Open-Meteo HTTP access (renamed from `OpenMeteoService`) | 2, 12 |
| `services/NearbyPhenomenaService.kt` | Next-24h phenomena for a coordinate | 11 |
| `services/CaptureValidationService.kt` | Validate a claimed phenomenon against observed data | 12, 12c |
| `models/UploadedPhoto.kt` | `uploaded_photos` — binds a stored photo to its uploader, single-use | 12b |
| `repositories/UploadedPhotoRepository.kt` | Photo lookup and the atomic conditional consume | 12b |
| `services/PhotoProvenanceService.kt` | Read-only `verify` + atomic `consume` of a cited photo | 12b |
| `services/CaptureCommitService.kt` | The one transaction: spend the photo, save the capture, update the trail | 12b, 12c |
| `services/PhotoStorageService.kt` | Write/serve uploaded photo bytes | 7 |
| `services/SkyDexService.kt` | Collection + XP + level assembly | 13 |
| `services/FeedService.kt` | Feed assembly across self + friends | 16 |

### Backend — `SkyDex-backend/src/test/kotlin/com/skydex/api/`

| File | Responsibility | Task |
|---|---|---|
| `support/TestcontainersConfiguration.kt` | Throwaway PostGIS container wired via `@ServiceConnection` | 1 |
| `support/IntegrationTestBase.kt` | Shared `@SpringBootTest` + MockMvc + profile + cleanup | 1 |
| `support/TestFixtures.kt` | `persistUser`, `authHeaderFor`, `persistEvent` helpers | 1, 6 |
| `controller/AuthControllerTest.kt` | renamed from `AuthController.kt` | 1 |
| `controller/WeatherEventControllerTest.kt` | renamed from `eventoController.kt` | 1 |
| `controller/UserControllerTest.kt` | renamed from `userController.kt` | 1 |
| `controller/PhotoControllerTest.kt` | Upload validation and round-trip | 7 |
| `controller/SkyDexControllerTest.kt` | Collection/XP/level responses | 13 |
| `controller/FriendControllerTest.kt` | Request/accept/list | 15 |
| `controller/FeedControllerTest.kt` | Visibility rules | 16 |
| `domain/PhenomenonTest.kt` | Code→species mapping is total and unambiguous | 11 |
| `domain/LevelsTest.kt` | Level boundaries | 13 |
| `domain/AchievementTest.kt` | Unlock rules and catalog integrity | 18 |
| `controller/ProfileControllerTest.kt` | Profile payload, awarding, idempotency, isolation | 18 |
| `service/CaptureValidationServiceTest.kt` | Confirm/reject logic against a stubbed client | 12 |

### Android — `SkyDex---frontend/app/src/main/java/com/example/skydex/`

The current layout puts screens under `ui/theme/pages/`, which nests pages inside the theme package. Task 5 moves them. Target layout:

| File | Responsibility | Task |
|---|---|---|
| `SkyDexApplication.kt` | `Application` subclass owning `ServiceLocator` | 4 |
| `ServiceLocator.kt` | Manual DI: session store, api, repositories | 4, 5 |
| `MainActivity.kt` | Sets theme, hosts `SkyDexNavHost` | 5 |
| `data/session/SessionStore.kt` | DataStore-backed JWT + userId | 4 |
| `data/remote/SkyDexApi.kt` | Retrofit interface (no manual auth headers) | 4, 6, 7, 13, 15, 16 |
| `data/remote/AuthInterceptor.kt` | Attaches `Authorization: Bearer …` | 4 |
| `data/remote/ApiFactory.kt` | Builds OkHttp + Retrofit (replaces `RetroFitClient.kt`) | 4 |
| `data/remote/dto/AuthDto.kt` | Login/register wire types | 4 |
| `data/remote/dto/WeatherEventDto.kt` | Capture wire types | 4, 6, 13 |
| `data/remote/dto/PhotoDto.kt` | Upload response | 7 |
| `data/remote/dto/SkyDexDto.kt` | Collection wire types | 13 |
| `data/remote/dto/FriendDto.kt` | Social wire types | 15 |
| `data/repository/AuthRepository.kt` | Login/register/logout + session writes | 4 |
| `data/repository/CaptureRepository.kt` | Nearby, upload, create, list mine | 5, 6, 7 |
| `data/repository/SkyDexRepository.kt` | Collection fetch | 13 |
| `data/repository/SocialRepository.kt` | Friends + feed | 15, 16 |
| `ui/navigation/Routes.kt` | Route constants | 5 |
| `ui/navigation/SkyDexNavHost.kt` | `NavHost` + auth gate | 5 |
| `ui/components/AppBottomBar.kt` | Bottom bar (extracted from `HomePage.kt`) | 5 |
| `ui/auth/LoginScreen.kt` / `LoginViewModel.kt` | Login | 4, 5 |
| `ui/auth/RegisterScreen.kt` / `RegisterViewModel.kt` | Register | 5 |
| `ui/home/HomeScreen.kt` / `HomeViewModel.kt` | Nearby phenomena + capture entry point | 5, 10 |
| `ui/capture/CaptureScreen.kt` / `CaptureViewModel.kt` | Camera + GPS + upload + submit | 8, 9, 10 |
| `ui/captures/MyCapturesScreen.kt` / `MyCapturesViewModel.kt` | Own captures (from `Registers.kt`) | 5 |
| `ui/skydex/SkyDexScreen.kt` / `SkyDexViewModel.kt` | Species grid | 14 |
| `ui/feed/FeedScreen.kt` / `FeedViewModel.kt` | Friends feed | 17 |
| `ui/friends/FriendsScreen.kt` / `FriendsViewModel.kt` | Requests + friend list | 17 |
| `ui/profile/ProfileScreen.kt` / `ProfileViewModel.kt` | Identity, level, stats, badge shelf, logout | 19 |
| `data/remote/dto/ProfileDto.kt` | Profile + badge wire types | 19 |
| `data/repository/ProfileRepository.kt` | Profile fetch | 19 |
| `util/DeviceLocation.kt` | Fused location one-shot | 9 |
| `util/PhotoCaptureFiles.kt` | FileProvider temp-file plumbing | 8 |
| `ui/theme/*` | Existing theme files, unchanged | — |

Deleted along the way: `RetroFitClient.kt` (→ `ApiFactory.kt`, task 4), `ui/theme/pages/HomePage.kt`, `Login.kt`, `Signin.kt`, `NearEvents.kt`, `Registers.kt` (→ `ui/*`, task 5), `dto/AuthDTO.kt`, `dto/EventoDTO.kt` (→ `data/remote/dto/*`, task 4).

---

# Phase 1 — Foundation and Hardening

Nothing else in this plan is safe to build until the test suite can run without destroying the dev database and the API stops handing out password hashes. Phase 1 changes no user-visible behaviour except that some endpoints disappear.

---

### Task 1: Runnable, isolated backend test suite

Right now `./gradlew test` cannot run (no executable bit on the wrapper) and, if it could, it would `deleteAll()` the developer's real database. This task makes the existing three test classes run against a throwaway container.

**Files:**
- Modify: `SkyDex-backend/gradlew` (permissions only)
- Modify: `SkyDex---frontend/gradlew` (permissions only)
- Modify: `SkyDex-backend/build.gradle.kts:22-42` (test dependencies)
- Modify: `SkyDex-backend/src/main/resources/application.properties`
- Create: `SkyDex-backend/src/test/resources/application-test.properties`
- Create: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestcontainersConfiguration.kt`
- Create: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt`
- Rename: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/AuthController.kt` → `AuthControllerTest.kt`
- Rename: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/eventoController.kt` → `WeatherEventControllerTest.kt`
- Rename: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/userController.kt` → `UserControllerTest.kt`
- Modify: `SkyDex-backend/CLAUDE.md`
- Modify: `SkyDex---frontend/CLAUDE.md`

**Interfaces:**
- Consumes: nothing (first task).
- Produces:
  - `com.skydex.api.support.TestcontainersConfiguration` — a `@TestConfiguration` exposing `postgresContainer(): PostgreSQLContainer<*>`.
  - `com.skydex.api.support.IntegrationTestBase` — `abstract class` that every backend integration test extends. Subclasses get `protected lateinit var mockMvc: MockMvc` and `protected lateinit var objectMapper: ObjectMapper` already injected, plus a `@BeforeEach` that truncates all tables.

- [ ] **Step 1: Make both Gradle wrappers executable**

```bash
cd <workspace>
chmod +x SkyDex-backend/gradlew SkyDex---frontend/gradlew
git -C SkyDex-backend update-index --chmod=+x gradlew
git -C SkyDex---frontend update-index --chmod=+x gradlew
```

Verify: `cd SkyDex-backend && ./gradlew --version` prints the Gradle version instead of "Permissão negada".

- [ ] **Step 2: Add Testcontainers test dependencies**

In `SkyDex-backend/build.gradle.kts`, inside the `dependencies { }` block, immediately after the existing `testImplementation("org.springframework.boot:spring-boot-starter-test")` line, add:

```kotlin
	testImplementation("org.springframework.boot:spring-boot-testcontainers")
	testImplementation("org.testcontainers:junit-jupiter")
	testImplementation("org.testcontainers:postgresql")
```

Spring Boot 3.2.4's dependency management already pins the Testcontainers BOM, so no versions are needed.

- [ ] **Step 3: Give the main datasource properties safe defaults**

`SkyDex-backend/src/main/resources/application.properties` currently fails to resolve its placeholders when `.env` is absent (for example in CI). Replace the first three lines:

```properties
spring.datasource.url=${URL_POSTGRES:jdbc:postgresql://localhost:5432/skydex}
spring.datasource.username=${USUARIO_POSTGRES:postgres}
spring.datasource.password=${SENHA_POSTGRES:postgres}
```

Leave the remaining lines (`ddl-auto`, `show-sql`, `server.port`) exactly as they are.

- [ ] **Step 4: Add the test profile properties**

Create `SkyDex-backend/src/test/resources/application-test.properties`:

```properties
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=false
spring.jpa.open-in-view=false

# Only ever used by tests. The real secret lives in .env and is never committed.
TOKEN_JWT_SECRET=integration-test-secret-do-not-use-in-production
```

- [ ] **Step 5: Write the Testcontainers configuration**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestcontainersConfiguration.kt`:

```kotlin
package com.skydex.api.support

import org.springframework.boot.test.context.TestConfiguration
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.springframework.context.annotation.Bean
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.utility.DockerImageName

/**
 * Starts a throwaway PostGIS database for the test run. The PostGIS image is used instead of
 * plain postgres so that tests exercise the same dialect Hibernate picks in development
 * (hibernate-spatial is on the classpath).
 */
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    fun postgresContainer(): PostgreSQLContainer<*> =
        PostgreSQLContainer(
            DockerImageName.parse("postgis/postgis:17-3.5")
                .asCompatibleSubstituteFor("postgres")
        )
}
```

- [ ] **Step 6: Write the shared integration test base**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt`:

```kotlin
package com.skydex.api.support

import com.fasterxml.jackson.databind.ObjectMapper
import com.skydex.api.repositories.EventoRepository
import com.skydex.api.repositories.UserRepository
import org.junit.jupiter.api.BeforeEach
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.annotation.Import
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.web.servlet.MockMvc

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Import(TestcontainersConfiguration::class)
abstract class IntegrationTestBase {

    @Autowired
    protected lateinit var mockMvc: MockMvc

    @Autowired
    protected lateinit var objectMapper: ObjectMapper

    @Autowired
    protected lateinit var userRepository: UserRepository

    @Autowired
    protected lateinit var eventoRepository: EventoRepository

    @BeforeEach
    fun clearDatabase() {
        eventoRepository.deleteAll()
        userRepository.deleteAll()
    }
}
```

Note: `EventoRepository` is the current (Portuguese) name. Task 2 renames it to `WeatherEventRepository` and this file is updated then.

- [ ] **Step 7: Rename the three test files and point them at the base class**

```bash
cd <workspace>/SkyDex-backend/src/test/kotlin/com/skydex/api/controller
git mv AuthController.kt AuthControllerTest.kt
git mv eventoController.kt WeatherEventControllerTest.kt
git mv userController.kt UserControllerTest.kt
```

In each renamed file, delete the class-level annotations `@SpringBootTest` and `@AutoConfigureMockMvc`, delete the `@Autowired mockMvc`, `objectMapper`, and repository fields that `IntegrationTestBase` now provides, delete the `deleteAll()` calls from `@BeforeEach` (keep any fixture creation), and make the class extend the base. For example `AuthControllerTest.kt` becomes:

```kotlin
package com.skydex.api.controller

import com.skydex.api.controllers.LoginRequest
import com.skydex.api.controllers.RegisterRequest
import com.skydex.api.models.User
import com.skydex.api.support.IntegrationTestBase
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.http.MediaType
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.LocalDateTime
import java.util.UUID

class AuthControllerTest : IntegrationTestBase() {

    @Autowired
    private lateinit var passwordEncoder: PasswordEncoder

    @Test
    fun `registers a new user and returns 200`() {
        val request = RegisterRequest(
            nome = "Dev SkyDex",
            email = "dev@skydex.com",
            password = "super-safe-password"
        )

        mockMvc.perform(
            post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.mensagem").value("Usuário Registrado com sucesso"))
    }

    @Test
    fun `rejects registration with an email that already exists`() {
        userRepository.save(
            User(
                id = UUID.randomUUID(),
                nome = "Existing User",
                email = "conflict@skydex.com",
                password = passwordEncoder.encode("any-password"),
                dataEntrada = LocalDateTime.now()
            )
        )

        val request = RegisterRequest(
            nome = "Impostor",
            email = "conflict@skydex.com",
            password = "another-password"
        )

        mockMvc.perform(
            post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("Usuário com esse email já cadastrado"))
    }

    @Test
    fun `logs in with valid credentials and returns a token`() {
        val plainPassword = "my-secret-password"
        userRepository.save(
            User(
                id = UUID.randomUUID(),
                nome = "SkyDex Admin",
                email = "admin@skydex.com",
                password = passwordEncoder.encode(plainPassword),
                dataEntrada = LocalDateTime.now()
            )
        )

        mockMvc.perform(
            post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(LoginRequest("admin@skydex.com", plainPassword)))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.tokenGerado").exists())
    }

    @Test
    fun `rejects login with the wrong password`() {
        userRepository.save(
            User(
                id = UUID.randomUUID(),
                nome = "SkyDex Admin",
                email = "admin@skydex.com",
                password = passwordEncoder.encode("right-password"),
                dataEntrada = LocalDateTime.now()
            )
        )

        mockMvc.perform(
            post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(LoginRequest("admin@skydex.com", "wrong-password")))
        )
            .andExpect(status().isUnauthorized)
            .andExpect(jsonPath("$.error").value("email ou senha inválidos"))
    }

    @Test
    fun `rejects login for an unknown email`() {
        mockMvc.perform(
            post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(LoginRequest("ghost@skydex.com", "whatever")))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("Usuário não encontrado"))
    }
}
```

Apply the same transformation to `WeatherEventControllerTest.kt` (class `WeatherEventControllerTest : IntegrationTestBase()`, keep its `usuarioTesteId` setup and its `SecurityContextHolder` wiring, remove the two `deleteAll()` lines) and `UserControllerTest.kt` (class `UserControllerTest : IntegrationTestBase()`, keep the `@MockBean openMeteoService`, remove `repository.deleteAll()` from `@BeforeEach`, and rename its `private lateinit var repository` usages to the inherited `userRepository`).

- [ ] **Step 8: Run the suite and watch it go green against a container**

```bash
cd <workspace>/SkyDex-backend
docker ps >/dev/null || { echo "start Docker first"; exit 1; }
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: `BUILD SUCCESSFUL`, all tests in `AuthControllerTest`, `WeatherEventControllerTest`, `UserControllerTest` pass. The first run pulls `postgis/postgis:17-3.5` and takes a few minutes.

If a test fails on the `users.email` column, it is a pre-existing test-data problem, not a container problem — the container starts from an empty schema, so any test that assumed leftover dev rows must be given its own fixture.

- [ ] **Step 9: Prove the dev database is untouched**

```bash
docker start skydex-db
sleep 3
docker exec skydex-db psql -U guilherme_becker -d skydex -c '\dt'
```

Expected: whatever tables existed before still exist. `./gradlew test` no longer touches them.

- [ ] **Step 10: Fill in the empty command placeholders in both CLAUDE.md files**

Replace the `## Useful commands` section of `SkyDex-backend/CLAUDE.md` with:

```markdown
## Useful commands

- `JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test` -> Running Tests (needs Docker; uses Testcontainers, never touches the dev DB)
- `JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.AuthControllerTest"` -> Running one test class
- `docker start skydex-db && JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun` -> Running Server on :8080
```

Replace the `## Useful Commands` section of `SkyDex---frontend/CLAUDE.md` with:

```markdown
## Useful Commands

- `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest` -> Running Tests
- `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin` -> Fast compile check
- `JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:installDebug` -> Running The Application on the connected phone
```

- [ ] **Step 11: Commit**

Stage and propose:

```bash
cd <workspace>/SkyDex-backend
git add gradlew build.gradle.kts src/main/resources/application.properties src/test
git commit -m "test: isolate integration tests with Testcontainers"
```

```bash
cd <workspace>/SkyDex---frontend
git add gradlew CLAUDE.md
git commit -m "chore: make gradle wrapper executable and document commands"
```

Run both commits directly — they are pre-authorized. Do not push.

---

### Task 2: English domain model and a DTO boundary

`GET /api/users` currently serialises `User` entities, which means every response contains the BCrypt hash under `"password"`. The fix is a DTO boundary, and since every response type is being rewritten anyway, this is the cheapest moment to move the domain to English.

**Files:**
- Rename: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/EventoMetereologico.kt` → `models/WeatherEvent.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/User.kt`
- Rename: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/EventoRepository.kt` → `repositories/WeatherEventRepository.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/UserRepository.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/AuthDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/UserDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/NearbyDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/ErrorResponse.kt`
- Delete: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/EventoProximo.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/OpenMeteoResponse.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/AuthController.kt`
- Rename: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/EventoController.kt` → `controllers/WeatherEventController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/UserController.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherController.kt`
- Rename: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/OpenMeteoService.kt` → `services/OpenMeteoClient.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/security/TokenService.kt` (comments only)
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt`
- Create: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestFixtures.kt`
- Modify: all three test classes from Task 1

**Interfaces:**
- Consumes: `IntegrationTestBase` from Task 1.
- Produces (later tasks depend on these exact signatures):
  - `models.User(id: UUID?, name: String, email: String, passwordHash: String, joinedAt: Instant)`. `passwordHash` is a private property, so it is set only through the constructor and read through the `UserDetails` method `getPassword()`. Nothing in the MVP changes a password after registration — see the post-MVP backlog.
  - `models.WeatherEvent(id: UUID?, title: String, description: String, photoUrl: String, capturedAt: Instant, userId: UUID)`. **Task 6 appends `latitude: Double, longitude: Double`. Task 12 appends `phenomenon: Phenomenon, validationStatus: ValidationStatus, observedWeatherCode: Int?, xpAwarded: Int`.**
  - `repositories.WeatherEventRepository : JpaRepository<WeatherEvent, UUID>` with `fun findByUserIdOrderByCapturedAtDesc(userId: UUID): List<WeatherEvent>`.
  - `dto.UserResponse(id: UUID, name: String, email: String, joinedAt: Instant)` with `UserResponse.from(user: User)`.
  - `dto.LoginResponse(token: String, userId: UUID, name: String)`.
  - `dto.CreateWeatherEventRequest(title: String, description: String, photoUrl: String)`. **Task 6 appends `latitude: Double, longitude: Double` — and deliberately *not* `capturedAt`: the server stamps the capture time, so no client can backdate a capture to yesterday's storm. Task 7 constrains `photoUrl` to `^/api/photos/[A-Za-z0-9._-]+\.(jpg|png)$`, so it carries the relative path the upload endpoint returned, never an absolute URL. Task 12 appends `phenomenon: String`.**
  - `dto.WeatherEventResponse(id: UUID, title: String, description: String, photoUrl: String, capturedAt: Instant, userId: UUID, authorName: String)` with `WeatherEventResponse.from(event: WeatherEvent, author: User)`.
    > **Signature changed in Task 7's fix round.** `from` now takes a required third parameter, `baseUrl: String`, and composes the absolute photo URL from the relative path stored on the row. Callers inject `@Value("\${skydex.photos.public-base-url}")`. The text below describes the two-argument form Tasks 2 and 3 actually shipped; Tasks 12, 16 and 18 use the three-argument form. **Task 6 appends `latitude, longitude`. Task 13 appends `phenomenon: String, phenomenonName: String, rarity: String, validationStatus: String, xpAwarded: Int`.**
  - `dto.NearbyPhenomenonResponse(phenomenon: String, time: String, temperatureCelsius: Double?, alertLevel: String)`. **Task 11 appends `phenomenonName: String, rarity: String`.**
  - `dto.ErrorResponse(error: String)`.
  - `services.OpenMeteoClient.fetchHourlyForecast(latitude: Double, longitude: Double): OpenMeteoResponse?`
  - API paths after this task: `POST /auth/register`, `POST /auth/login`, `GET|PUT|DELETE /api/users/{id}`, `GET /api/users/{id}/events`, `GET /api/weather/nearby?lat=&lon=`, `POST|GET /api/events`, `GET|PUT|DELETE /api/events/{id}`.

- [ ] **Step 1: Write the failing test for the password leak**

Add to `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/UserControllerTest.kt`:

```kotlin
    @Test
    fun `user responses never expose the password hash`() {
        val user = persistUser(name = "Leak Check", email = "leak@skydex.com", password = "plain-text-secret")

        mockMvc.perform(
            get("/api/users/{id}", user.id!!)
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.name").value("Leak Check"))
            .andExpect(jsonPath("$.email").value("leak@skydex.com"))
            .andExpect(jsonPath("$.password").doesNotExist())
            .andExpect(jsonPath("$.passwordHash").doesNotExist())
            .andExpect(jsonPath("$.authorities").doesNotExist())
            .andExpect(jsonPath("$.enabled").doesNotExist())
    }
```

- [ ] **Step 2: Write the test fixtures the new test needs**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestFixtures.kt`:

```kotlin
package com.skydex.api.support

import com.skydex.api.models.User
import com.skydex.api.models.WeatherEvent
import java.time.Instant
import java.util.UUID

/**
 * Fixture helpers shared by every integration test. They live as extension functions on the
 * test base so they can reach the injected repositories.
 */
fun IntegrationTestBase.persistUser(
    name: String = "Test Pilot",
    email: String = "pilot@skydex.com",
    password: String = "test-password"
): User = userRepository.save(
    User(
        id = null,
        name = name,
        email = email,
        passwordHash = passwordEncoder.encode(password),
        joinedAt = Instant.now()
    )
)

fun IntegrationTestBase.persistEvent(
    owner: User,
    title: String = "Aurora",
    description: String = "Green lights in the night sky",
    photoUrl: String = "http://localhost:8080/api/photos/test.jpg",
    capturedAt: Instant = Instant.now()
): WeatherEvent = weatherEventRepository.save(
    WeatherEvent(
        id = null,
        title = title,
        description = description,
        photoUrl = photoUrl,
        capturedAt = capturedAt,
        userId = owner.id!!
    )
)

/** Builds a ready-to-send `Authorization` header value for the given user. */
fun IntegrationTestBase.authHeaderFor(user: User): String =
    "Bearer " + tokenService.generateToken(user)
```

- [ ] **Step 3: Extend the test base so the fixtures compile**

Replace the body of `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt` with:

```kotlin
package com.skydex.api.support

import com.fasterxml.jackson.databind.ObjectMapper
import com.skydex.api.repositories.UserRepository
import com.skydex.api.repositories.WeatherEventRepository
import com.skydex.api.security.TokenService
import org.junit.jupiter.api.BeforeEach
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.annotation.Import
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.web.servlet.MockMvc

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Import(TestcontainersConfiguration::class)
abstract class IntegrationTestBase {

    @Autowired
    protected lateinit var mockMvc: MockMvc

    @Autowired
    protected lateinit var objectMapper: ObjectMapper

    @Autowired
    internal lateinit var userRepository: UserRepository

    @Autowired
    internal lateinit var weatherEventRepository: WeatherEventRepository

    @Autowired
    internal lateinit var passwordEncoder: PasswordEncoder

    @Autowired
    internal lateinit var tokenService: TokenService

    @BeforeEach
    fun clearDatabase() {
        weatherEventRepository.deleteAll()
        userRepository.deleteAll()
    }
}
```

The repositories are `internal` rather than `protected` so the extension functions in `TestFixtures.kt` can reach them.

- [ ] **Step 4: Run the test and confirm it fails to compile**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.UserControllerTest"
```

Expected: compilation failure — `Unresolved reference: WeatherEventRepository`, `Unresolved reference: WeatherEvent`, `No value passed for parameter 'name'`. That is the red state; the rest of this task makes it green.

- [ ] **Step 5: Rewrite the User entity in English**

Replace `SkyDex-backend/src/main/kotlin/com/skydex/api/models/User.kt` entirely:

```kotlin
package com.skydex.api.models

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.userdetails.UserDetails
import java.time.Instant
import java.util.UUID

@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    var id: UUID? = null,

    @Column(nullable = false)
    var name: String = "",

    @Column(nullable = false, unique = true)
    var email: String = "",

    @Column(name = "password_hash", nullable = false)
    private var passwordHash: String = "",

    @Column(name = "joined_at", nullable = false)
    var joinedAt: Instant = Instant.now()
) : UserDetails {

    override fun getAuthorities(): MutableCollection<out GrantedAuthority> =
        mutableListOf(SimpleGrantedAuthority("ROLE_USER"))

    override fun getPassword(): String = passwordHash

    override fun getUsername(): String = email

    override fun isAccountNonExpired(): Boolean = true
    override fun isAccountNonLocked(): Boolean = true
    override fun isCredentialsNonExpired(): Boolean = true
    override fun isEnabled(): Boolean = true
}
```

Two things to notice. `passwordHash` is `private var` with no mutator, so the only way a hash enters the entity is the constructor — the old `definirSenha` method is gone and nothing in the MVP changes a password after registration. And the hash is reachable only through `getPassword()`, the `UserDetails` contract method, which is exactly the accessor Jackson would have serialised; the DTO boundary in the next steps is what actually keeps it out of responses.

`private` on a primary-constructor property applies to the generated property, not to the constructor parameter, so `User(passwordHash = …)` still works as a named argument from `AuthController` and the test fixtures.

- [ ] **Step 6: Rewrite the event entity in English**

```bash
cd <workspace>/SkyDex-backend/src/main/kotlin/com/skydex/api
git mv models/EventoMetereologico.kt models/WeatherEvent.kt
```

Replace its contents:

```kotlin
package com.skydex.api.models

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.time.Instant
import java.util.UUID

@Entity
@Table(name = "weather_events")
class WeatherEvent(
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    var id: UUID? = null,

    @Column(nullable = false)
    var title: String = "",

    @Column(columnDefinition = "TEXT", nullable = false)
    var description: String = "",

    @Column(name = "photo_url", nullable = false)
    var photoUrl: String = "",

    @Column(name = "captured_at", nullable = false)
    var capturedAt: Instant = Instant.now(),

    @Column(name = "user_id", nullable = false)
    var userId: UUID
)
```

- [ ] **Step 7: Rename the repository**

```bash
git mv repositories/EventoRepository.kt repositories/WeatherEventRepository.kt
```

```kotlin
package com.skydex.api.repositories

import com.skydex.api.models.WeatherEvent
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface WeatherEventRepository : JpaRepository<WeatherEvent, UUID> {
    fun findByUserIdOrderByCapturedAtDesc(userId: UUID): List<WeatherEvent>
}
```

And leave `UserRepository.kt` as-is apart from adding the `@Repository` annotation for consistency:

```kotlin
package com.skydex.api.repositories

import com.skydex.api.models.User
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface UserRepository : JpaRepository<User, UUID> {
    fun findByEmail(email: String): User?
}
```

- [ ] **Step 8: Create the DTOs**

`dto/ErrorResponse.kt`:

```kotlin
package com.skydex.api.dto

data class ErrorResponse(val error: String)
```

`dto/AuthDtos.kt`:

```kotlin
package com.skydex.api.dto

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Pattern
import jakarta.validation.constraints.Size
import java.util.UUID

data class LoginRequest(
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email format is invalid")
    val email: String,

    @field:NotBlank(message = "Password is required")
    val password: String
)

data class RegisterRequest(
    @field:NotBlank(message = "Name is required")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email format is invalid")
    val email: String,

    @field:NotBlank(message = "Password is required")
    @field:Size(min = 8, message = "Password must be at least 8 characters long")
    val password: String
)

data class LoginResponse(
    val token: String,
    val userId: UUID,
    val name: String
)
```

`dto/UserDtos.kt`:

```kotlin
package com.skydex.api.dto

import com.skydex.api.models.User
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import java.time.Instant
import java.util.UUID

data class UserResponse(
    val id: UUID,
    val name: String,
    val email: String,
    val joinedAt: Instant
) {
    companion object {
        fun from(user: User) = UserResponse(
            id = user.id!!,
            name = user.name,
            email = user.email,
            joinedAt = user.joinedAt
        )
    }
}

data class UpdateProfileRequest(
    @field:NotBlank(message = "Name is required")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email format is invalid")
    val email: String
)
```

`dto/WeatherEventDtos.kt`:

```kotlin
package com.skydex.api.dto

import com.skydex.api.models.User
import com.skydex.api.models.WeatherEvent
import jakarta.validation.constraints.NotBlank
import java.time.Instant
import java.util.UUID

data class CreateWeatherEventRequest(
    @field:NotBlank(message = "Title is required")
    val title: String,

    @field:NotBlank(message = "Description is required")
    val description: String,

    @field:NotBlank(message = "Photo URL is required")
    val photoUrl: String
)

data class WeatherEventResponse(
    val id: UUID,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: Instant,
    val userId: UUID,
    val authorName: String
) {
    companion object {
        fun from(event: WeatherEvent, author: User) = WeatherEventResponse(
            id = event.id!!,
            title = event.title,
            description = event.description,
            photoUrl = event.photoUrl,
            capturedAt = event.capturedAt,
            userId = event.userId,
            authorName = author.name
        )
    }
}
```

`dto/NearbyDtos.kt`:

```kotlin
package com.skydex.api.dto

data class NearbyPhenomenonResponse(
    val phenomenon: String,
    val time: String,
    val temperatureCelsius: Double?,
    val alertLevel: String
)
```

Delete `dto/EventoProximo.kt`:

```bash
git rm src/main/kotlin/com/skydex/api/dto/EventoProximo.kt
```

Rename the fields in `dto/OpenMeteoResponse.kt` to English while keeping the Open-Meteo wire names via `@JsonProperty`:

```kotlin
package com.skydex.api.dto

import com.fasterxml.jackson.annotation.JsonProperty

data class OpenMeteoResponse(
    val latitude: Double,
    val longitude: Double,
    val hourly: HourlyData?
)

data class HourlyData(
    val time: List<String>,
    @JsonProperty("temperature_2m") val temperatureCelsius: List<Double?>,
    @JsonProperty("weather_code") val weatherCode: List<Int?>
)
```

- [ ] **Step 9: Rewrite `OpenMeteoService` as a thin client**

```bash
git mv services/OpenMeteoService.kt services/OpenMeteoClient.kt
```

```kotlin
package com.skydex.api.services

import com.skydex.api.dto.OpenMeteoResponse
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import org.springframework.web.client.RestClient

/**
 * Thin HTTP access to the Open-Meteo forecast API. Interpretation of the response
 * (which phenomenon a weather code means) lives in the services that consume this.
 *
 * Times are requested in UTC so they can be compared against capture instants directly.
 */
@Service
class OpenMeteoClient {

    private val log = LoggerFactory.getLogger(javaClass)
    private val restClient = RestClient.create("https://api.open-meteo.com")

    fun fetchHourlyForecast(latitude: Double, longitude: Double): OpenMeteoResponse? =
        try {
            restClient.get()
                .uri(
                    "/v1/forecast?latitude={lat}&longitude={lon}" +
                        "&hourly=temperature_2m,weather_code&timezone=UTC&past_days=1&forecast_days=2",
                    latitude,
                    longitude
                )
                .retrieve()
                .body(OpenMeteoResponse::class.java)
        } catch (e: Exception) {
            log.warn("Open-Meteo request failed for lat={} lon={}", latitude, longitude, e)
            null
        }
}
```

- [ ] **Step 10: Move the "nearby phenomena" logic into its own controller**

Create `controllers/WeatherController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.NearbyPhenomenonResponse
import com.skydex.api.services.OpenMeteoClient
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/weather")
class WeatherController(private val openMeteoClient: OpenMeteoClient) {

    @GetMapping("/nearby")
    fun nearby(
        @RequestParam lat: Double,
        @RequestParam lon: Double
    ): ResponseEntity<List<NearbyPhenomenonResponse>> {
        val hourly = openMeteoClient.fetchHourlyForecast(lat, lon)?.hourly
            ?: return ResponseEntity.ok(emptyList())

        // OpenMeteoClient asks for past_days=1, so these arrays BEGIN YESTERDAY at 00:00 UTC
        // and run 72 slots. Select by timestamp, never by array index: an index window here
        // would make an endpoint named "nearby" report the past.
        val now = Instant.now()
        val results = mutableListOf<NearbyPhenomenonResponse>()
        val slots = minOf(hourly.time.size, hourly.weatherCode.size, hourly.temperatureCelsius.size)
        for (i in 0 until slots) {
            if (results.size >= FORECAST_HOURS) break
            val slotTime = parseUtcSlot(hourly.time[i]) ?: continue
            if (slotTime.isBefore(now)) continue
            val code = hourly.weatherCode[i] ?: continue
            val alertLevel = when (code) {
                45, 48 -> "Interessante"
                65, 80, 81, 82 -> "Atenção"
                71, 73, 75, 77, 85, 86 -> "Interessante"
                95 -> "Perigo"
                96, 99 -> "Perigo Extremo!"
                else -> "Tranquilo"
            }
            val name = when (code) {
                0, 1 -> "Céu Limpo"
                2, 3 -> "Nublado"
                45, 48 -> "Nevoeiro Intenso"
                51, 53, 55, 56, 57 -> "Garoa"
                61, 63, 65, 66, 67 -> "Chuva"
                71, 73, 75, 77, 85, 86 -> "Neve"
                80, 81, 82 -> "Pancada de Chuva"
                95 -> "Tempestade com Trovões"
                96, 99 -> "Tempestade Severa com Granizo"
                else -> continue
            }
            results.add(
                NearbyPhenomenonResponse(
                    phenomenon = name,
                    time = hourly.time[i],
                    temperatureCelsius = hourly.temperatureCelsius[i],
                    alertLevel = alertLevel
                )
            )
        }
        return ResponseEntity.ok(results)
    }

    /** Open-Meteo returns "2026-08-07T14:00" with no offset; the client requested timezone=UTC. */
    private fun parseUtcSlot(raw: String): Instant? = try {
        LocalDateTime.parse(raw).toInstant(ZoneOffset.UTC)
    } catch (e: DateTimeParseException) {
        null
    }

    private companion object {
        const val FORECAST_HOURS = 24
    }
}
```

Add the imports `java.time.Instant`, `java.time.LocalDateTime`, `java.time.ZoneOffset`, `java.time.format.DateTimeParseException`.

Task 11 replaces both `when` blocks with the `Phenomenon` catalog. They are written out longhand here so this task compiles and passes on its own.

**The forecast window is verified behaviour, not a guess.** A live call to the exact URL `OpenMeteoClient` builds returns 72 hourly slots whose index 0 is *yesterday* 00:00 UTC and whose index 24 is today 00:00 UTC. Slicing `0..23` therefore returns the 24 hours that already elapsed. Any test for this endpoint must build its slot timestamps **relative to `Instant.now()`** — a hardcoded date string will fall into the past and make the test rot silently.

- [ ] **Step 11: Rewrite `AuthController`**

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.ErrorResponse
import com.skydex.api.dto.LoginRequest
import com.skydex.api.dto.LoginResponse
import com.skydex.api.dto.RegisterRequest
import com.skydex.api.dto.UserResponse
import com.skydex.api.models.User
import com.skydex.api.repositories.UserRepository
import com.skydex.api.security.TokenService
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.time.Instant

@RestController
@RequestMapping("/auth")
class AuthController(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val tokenService: TokenService
) {

    @PostMapping("/register")
    fun register(@Valid @RequestBody request: RegisterRequest): ResponseEntity<Any> {
        if (userRepository.findByEmail(request.email) != null) {
            return ResponseEntity.badRequest().body(ErrorResponse("Email already registered"))
        }

        val user = userRepository.save(
            User(
                id = null,
                name = request.name,
                email = request.email,
                passwordHash = passwordEncoder.encode(request.password),
                joinedAt = Instant.now()
            )
        )
        return ResponseEntity.status(HttpStatus.CREATED).body(UserResponse.from(user))
    }

    @PostMapping("/login")
    fun login(@Valid @RequestBody request: LoginRequest): ResponseEntity<Any> {
        val user = userRepository.findByEmail(request.email)
            ?: return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(ErrorResponse("Invalid email or password"))

        if (!passwordEncoder.matches(request.password, user.password)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(ErrorResponse("Invalid email or password"))
        }

        return ResponseEntity.ok(
            LoginResponse(
                token = tokenService.generateToken(user),
                userId = user.id!!,
                name = user.name
            )
        )
    }
}
```

Note two deliberate behaviour changes: register now returns **201** with a `UserResponse`, and an unknown email now returns **401** with the same message as a wrong password, so the endpoint no longer confirms which emails exist.

Because `User.passwordHash` is private, read it through `user.password` — that is the `UserDetails.getPassword()` accessor.

- [ ] **Step 12: Rewrite the event controller**

```bash
git mv controllers/EventoController.kt controllers/WeatherEventController.kt
```

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.CreateWeatherEventRequest
import com.skydex.api.dto.WeatherEventResponse
import com.skydex.api.models.User
import com.skydex.api.models.WeatherEvent
import com.skydex.api.repositories.UserRepository
import com.skydex.api.repositories.WeatherEventRepository
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.PutMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.time.Instant
import java.util.UUID

@RestController
@RequestMapping("/api/events")
class WeatherEventController(
    private val events: WeatherEventRepository,
    private val users: UserRepository
) {

    @PostMapping
    fun create(
        @AuthenticationPrincipal currentUser: User,
        @Valid @RequestBody request: CreateWeatherEventRequest
    ): ResponseEntity<WeatherEventResponse> {
        val saved = events.save(
            WeatherEvent(
                id = null,
                title = request.title,
                description = request.description,
                photoUrl = request.photoUrl,
                capturedAt = Instant.now(),
                userId = currentUser.id!!
            )
        )
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(WeatherEventResponse.from(saved, currentUser))
    }

    @GetMapping("/mine")
    fun listMine(@AuthenticationPrincipal currentUser: User): ResponseEntity<List<WeatherEventResponse>> {
        val mine = events.findByUserIdOrderByCapturedAtDesc(currentUser.id!!)
        return ResponseEntity.ok(mine.map { WeatherEventResponse.from(it, currentUser) })
    }

    @GetMapping("/{id}")
    fun getById(@PathVariable id: UUID): ResponseEntity<WeatherEventResponse> {
        val event = events.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        val author = users.findById(event.userId).orElse(null) ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(WeatherEventResponse.from(event, author))
    }

    @PutMapping("/{id}")
    fun update(
        @AuthenticationPrincipal currentUser: User,
        @PathVariable id: UUID,
        @Valid @RequestBody request: CreateWeatherEventRequest
    ): ResponseEntity<WeatherEventResponse> {
        val event = events.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        val author = users.findById(event.userId).orElse(null) ?: return ResponseEntity.notFound().build()
        event.title = request.title
        event.description = request.description
        event.photoUrl = request.photoUrl
        // The author is looked up from the event, never taken from the caller: until Task 3 adds
        // the ownership check these can differ, and a response that names the caller as author of
        // someone else's capture is internally inconsistent with the userId in the same body.
        return ResponseEntity.ok(WeatherEventResponse.from(events.save(event), author))
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: UUID): ResponseEntity<Void> {
        val event = events.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        events.delete(event)
        return ResponseEntity.noContent().build()
    }
}
```

The ownership checks that `update` and `delete` still lack are Task 3. `GET /api/events` (list everything) and `DELETE /api/events` (wipe everything) are gone.

- [ ] **Step 13: Rewrite the user controller**

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.ErrorResponse
import com.skydex.api.dto.UpdateProfileRequest
import com.skydex.api.dto.UserResponse
import com.skydex.api.dto.WeatherEventResponse
import com.skydex.api.models.User
import com.skydex.api.repositories.UserRepository
import com.skydex.api.repositories.WeatherEventRepository
import jakarta.validation.Valid
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PutMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.util.UUID

@RestController
@RequestMapping("/api/users")
class UserController(
    private val users: UserRepository,
    private val events: WeatherEventRepository
) {

    @GetMapping("/{id}")
    fun getById(@PathVariable id: UUID): ResponseEntity<UserResponse> {
        val user = users.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(UserResponse.from(user))
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: UUID,
        @Valid @RequestBody request: UpdateProfileRequest
    ): ResponseEntity<UserResponse> {
        val user = users.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        user.name = request.name
        user.email = request.email
        return ResponseEntity.ok(UserResponse.from(users.save(user)))
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: UUID): ResponseEntity<Void> {
        val user = users.findById(id).orElse(null) ?: return ResponseEntity.notFound().build()
        users.delete(user)
        return ResponseEntity.noContent().build()
    }

    @GetMapping("/{id}/events")
    fun listEvents(@PathVariable id: UUID): ResponseEntity<Any> {
        val user = users.findById(id).orElse(null)
            ?: return ResponseEntity.status(404).body(ErrorResponse("User not found"))
        val list = events.findByUserIdOrderByCapturedAtDesc(id)
        return ResponseEntity.ok(list.map { WeatherEventResponse.from(it, user) })
    }
}
```

Gone: `POST /api/users` (duplicate of register, with no duplicate-email check), `GET /api/users` (listed every user with their hash), `DELETE /api/users` (wiped every user), and `GET /api/users/{id}/eventosProximos` (moved to `WeatherController`). Task 3 replaces the `{id}` paths with `/me`.

Note the behaviour change on `GET /{id}/events`: an empty list is now **200 with `[]`**, not 404.

- [ ] **Step 14: Translate the Portuguese comments in `TokenService.kt`**

Keep the logic byte-for-byte identical; replace only the trailing comments:

```kotlin
    fun generateToken(user: User): String {
        val algorithm = Algorithm.HMAC256(secret)
        return JWT.create()
            .withIssuer("skydex-api")
            .withSubject(user.email)
            .withExpiresAt(generateExpirationDate())
            .sign(algorithm)
    }

    fun validateToken(token: String): String {
        val algorithm = Algorithm.HMAC256(secret)
        return JWT.require(algorithm)
            .withIssuer("skydex-api")
            .build()
            .verify(token)
            .subject
    }

    private fun generateExpirationDate(): Instant =
        LocalDateTime.now().plusHours(2).toInstant(ZoneOffset.of("-03:00"))
```

- [ ] **Step 15: Update the three test classes to the new names and paths**

Mechanical rename pass across `src/test`:

- `EventoMetereologico(...)` → `WeatherEvent(...)`, and `titulo`/`descricao`/`urlFoto`/`dataHoraRegistro` → `title`/`description`/`photoUrl`/`capturedAt` (now an `Instant`; use `Instant.now()`).
- `User(nome = …, dataEntrada = …)` → `User(name = …, passwordHash = …, joinedAt = Instant.now())`, or just call `persistUser(...)`.
- `EventoRequest` → `CreateWeatherEventRequest`, dropping its `userId` argument.
- `UserRequest` → `UpdateProfileRequest(name = …, email = …)`.
- `/api/eventos` → `/api/events`; `/api/users/{id}/eventos` → `/api/users/{id}/events`; `/api/users/{id}/eventosProximos` → `/api/weather/nearby`.
- `jsonPath("$.nome")` → `jsonPath("$.name")`; `$.titulo` → `$.title`; `$.urlFoto` → `$.photoUrl`.
- The `@MockBean openMeteoService: OpenMeteoService` in `UserControllerTest` becomes `@MockBean openMeteoClient: OpenMeteoClient`, and its stub becomes `when(openMeteoClient.fetchHourlyForecast(lat, lon)).thenReturn(null)`.
- Delete the two tests that exercised removed endpoints: "deve criar um novo usuario…" (POST `/api/users`) and "deve listar todos os usuarios…" (GET `/api/users`).
- Move the nearby-phenomena tests out of `UserControllerTest` into a new `controller/WeatherControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.dto.HourlyData
import com.skydex.api.dto.OpenMeteoResponse
import com.skydex.api.services.OpenMeteoClient
import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Test
import org.mockito.Mockito.`when`
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

class WeatherControllerTest : IntegrationTestBase() {

    @MockBean
    private lateinit var openMeteoClient: OpenMeteoClient

    @Test
    fun `maps a thunderstorm weather code to a danger-level phenomenon`() {
        val user = persistUser(email = "weather@skydex.com")
        `when`(openMeteoClient.fetchHourlyForecast(-23.55, -46.63)).thenReturn(
            OpenMeteoResponse(
                latitude = -23.55,
                longitude = -46.63,
                hourly = HourlyData(
                    time = listOf("2026-08-07T12:00"),
                    temperatureCelsius = listOf(21.5),
                    weatherCode = listOf(95)
                )
            )
        )

        mockMvc.perform(
            get("/api/weather/nearby")
                .param("lat", "-23.55")
                .param("lon", "-46.63")
                .header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$[0].phenomenon").value("Tempestade com Trovões"))
            .andExpect(jsonPath("$[0].alertLevel").value("Perigo"))
            .andExpect(jsonPath("$[0].temperatureCelsius").value(21.5))
    }

    @Test
    fun `returns an empty list when Open-Meteo is unreachable`() {
        val user = persistUser(email = "offline@skydex.com")
        `when`(openMeteoClient.fetchHourlyForecast(1.0, 2.0)).thenReturn(null)

        mockMvc.perform(
            get("/api/weather/nearby")
                .param("lat", "1.0")
                .param("lon", "2.0")
                .header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(0))
    }
}
```

Note the behaviour change: an Open-Meteo failure is now 200-with-empty-list rather than 500. A transient upstream outage should not read as a client-visible server error, and the app renders "nenhum evento" either way.

- [ ] **Step 16: Reset the dev database**

Renaming `eventos_catalogados` → `weather_events` and `nome` → `name` under `ddl-auto=update` creates new empty tables and orphans the old ones. Pre-MVP data is disposable, so drop and recreate:

```bash
docker start skydex-db
sleep 3
docker exec skydex-db psql -U guilherme_becker -d skydex -c \
  'DROP TABLE IF EXISTS eventos_catalogados, weather_events, users CASCADE;'
```

Dropping only the application tables — rather than the whole database — leaves the PostGIS extensions and the `tiger` schema intact, so nothing has to be re-created afterwards. Hibernate rebuilds `users` and `weather_events` from the new entity mappings on the next start.

The user pre-authorized this reset on 2026-08-07, having been told it destroys the 3 users and 7 events then present in the dev database. Run it without asking again. Verify afterwards:

```bash
docker exec skydex-db psql -U guilherme_becker -d skydex -c '\dt public.*'
```

Expected: `users` and `eventos_catalogados` are gone; `spatial_ref_sys` remains.

- [ ] **Step 17: Run the full backend suite**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, including `user responses never expose the password hash`.

- [ ] **Step 18: Commit**

```bash
git add -A src
git commit -m "refactor!: english domain model and DTO response boundary

Entities are no longer serialized directly, which stops GET /api/users
handing out BCrypt hashes. Renames EventoMetereologico to WeatherEvent
and the Portuguese field and route names to English."
```

---

### Task 3: Authorization, ownership, and honest error responses

After Task 2, any authenticated user can still edit or delete any other user's captures and profile, an invalid JWT produces a 500, and an anonymous request gets 403 instead of 401.

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/errors/DomainExceptions.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/GlobalExceptionHandler.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/security/SecurityFilter.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/config/SecurityConfig.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/UserController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/UserControllerTest.kt`

**Interfaces:**
- Consumes: `IntegrationTestBase`, `persistUser`, `persistEvent`, `authHeaderFor` (Task 1/2); `UserResponse`, `WeatherEventResponse`, `ErrorResponse` (Task 2).
- Produces:
  - `errors.NotFoundException(message: String)`, `errors.ForbiddenException(message: String)`, `errors.ConflictException(message: String)` — all extend `RuntimeException`. Later tasks throw these instead of building `ResponseEntity` error bodies by hand.
  - `GlobalExceptionHandler` maps them to 404/403/409 and `MethodArgumentNotValidException` to 400, all with an `ErrorResponse` body.
  - Routes change to `GET|PUT|DELETE /api/users/me` and `GET /api/users/me/events`. `GET /api/users/{id}` and `GET /api/users/{id}/events` are removed.
    > **Superseded by Task 4, Step 0.** `GET /api/users/me/events` duplicates `GET /api/events/mine` from Task 2 — same repository call, same mapping, same ordering — and its test asserts only `$.length()`, which `findAll()` would satisfy identically. Task 4 deletes it and keeps `/api/events/mine`, which lives on the right resource and carries the stronger ownership-scoping test. Left in place here because Task 3 was already executed; this note records why the endpoint exists in that commit and not afterwards.
  - Every non-`/auth/**` request without a valid bearer token returns **401** with `{"error":"Authentication required"}`.

- [ ] **Step 1: Write the failing ownership and auth tests**

Append to `WeatherEventControllerTest.kt`:

```kotlin
    @Test
    fun `refuses to update an event owned by another user`() {
        val owner = persistUser(email = "owner@skydex.com")
        val intruder = persistUser(email = "intruder@skydex.com")
        val event = persistEvent(owner, title = "Owner's storm")

        val payload = CreateWeatherEventRequest(
            title = "Hijacked",
            description = "Not mine",
            photoUrl = "http://localhost:8080/api/photos/x.jpg"
        )

        mockMvc.perform(
            put("/api/events/{id}", event.id!!)
                .header("Authorization", authHeaderFor(intruder))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isForbidden)
            .andExpect(jsonPath("$.error").value("You can only modify your own captures"))

        val unchanged = weatherEventRepository.findById(event.id!!).orElseThrow()
        assertEquals("Owner's storm", unchanged.title)
    }

    @Test
    fun `refuses to delete an event owned by another user`() {
        val owner = persistUser(email = "owner2@skydex.com")
        val intruder = persistUser(email = "intruder2@skydex.com")
        val event = persistEvent(owner)

        mockMvc.perform(
            delete("/api/events/{id}", event.id!!)
                .header("Authorization", authHeaderFor(intruder))
        )
            .andExpect(status().isForbidden)

        assertTrue(weatherEventRepository.existsById(event.id!!))
    }

    @Test
    fun `returns 401 rather than 500 for a malformed token`() {
        mockMvc.perform(
            get("/api/events/mine")
                .header("Authorization", "Bearer not-a-real-jwt")
        )
            .andExpect(status().isUnauthorized)
            .andExpect(jsonPath("$.error").value("Authentication required"))
    }

    @Test
    fun `returns 401 when no token is supplied`() {
        mockMvc.perform(get("/api/events/mine"))
            .andExpect(status().isUnauthorized)
    }
```

Add the imports the new tests need: `com.skydex.api.dto.CreateWeatherEventRequest`, `com.skydex.api.support.persistEvent`, `com.skydex.api.support.persistUser`, `com.skydex.api.support.authHeaderFor`, `org.junit.jupiter.api.Assertions.assertEquals`, `org.junit.jupiter.api.Assertions.assertTrue`.

Also delete the `SecurityContextHolder.getContext().authentication = auth` line from that class's `@BeforeEach` — from here on, tests authenticate with real bearer headers so they exercise the real filter chain.

Append to `UserControllerTest.kt`:

```kotlin
    @Test
    fun `me returns the authenticated user`() {
        val user = persistUser(name = "Self", email = "self@skydex.com")

        mockMvc.perform(
            get("/api/users/me").header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.id").value(user.id!!.toString()))
            .andExpect(jsonPath("$.name").value("Self"))
            .andExpect(jsonPath("$.password").doesNotExist())
    }

    @Test
    fun `rejects a profile update that would take another user's email`() {
        persistUser(email = "taken@skydex.com")
        val user = persistUser(email = "mover@skydex.com")

        val payload = UpdateProfileRequest(name = "Mover", email = "taken@skydex.com")

        mockMvc.perform(
            put("/api/users/me")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isConflict)
            .andExpect(jsonPath("$.error").value("Email already registered"))
    }
```

- [ ] **Step 2: Run the tests and watch them fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.WeatherEventControllerTest" --tests "com.skydex.api.controller.UserControllerTest"
```

Expected: `refuses to update an event owned by another user` fails with `Status expected:<403> but was:<200>`; `returns 401 rather than 500 for a malformed token` fails with a 500; `me returns the authenticated user` fails with a 404.

- [ ] **Step 3: Add the domain exceptions**

Create `errors/DomainExceptions.kt`:

```kotlin
package com.skydex.api.errors

class NotFoundException(message: String) : RuntimeException(message)

class ForbiddenException(message: String) : RuntimeException(message)

class ConflictException(message: String) : RuntimeException(message)
```

- [ ] **Step 4: Add the exception handler**

Create `controllers/GlobalExceptionHandler.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.ErrorResponse
import com.skydex.api.errors.ConflictException
import com.skydex.api.errors.ForbiddenException
import com.skydex.api.errors.NotFoundException
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.multipart.MaxUploadSizeExceededException

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException::class)
    fun handleNotFound(e: NotFoundException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.NOT_FOUND).body(ErrorResponse(e.message ?: "Not found"))

    @ExceptionHandler(ForbiddenException::class)
    fun handleForbidden(e: ForbiddenException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.FORBIDDEN).body(ErrorResponse(e.message ?: "Forbidden"))

    @ExceptionHandler(ConflictException::class)
    fun handleConflict(e: ConflictException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.CONFLICT).body(ErrorResponse(e.message ?: "Conflict"))

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val message = e.bindingResult.fieldErrors
            .joinToString("; ") { "${it.field}: ${it.defaultMessage}" }
            .ifBlank { "Invalid request" }
        return ResponseEntity.badRequest().body(ErrorResponse(message))
    }

    @ExceptionHandler(MaxUploadSizeExceededException::class)
    fun handleTooLarge(e: MaxUploadSizeExceededException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE).body(ErrorResponse("Photo is too large"))
}
```

The `MaxUploadSizeExceededException` handler is unused until Task 7; it is here so the handler is written once.

- [ ] **Step 5: Make the security filter swallow invalid tokens**

Replace `security/SecurityFilter.kt`:

```kotlin
package com.skydex.api.security

import com.auth0.jwt.exceptions.JWTVerificationException
import com.skydex.api.repositories.UserRepository
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

/**
 * Reads the bearer token from each request and, if it is valid, marks the caller as
 * authenticated. An invalid or expired token leaves the context anonymous; the entry point
 * configured in SecurityConfig turns that into a 401.
 */
@Component
class SecurityFilter(
    private val tokenService: TokenService,
    private val userRepository: UserRepository
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val token = recoverToken(request)

        if (token != null) {
            val email = try {
                tokenService.validateToken(token)
            } catch (e: JWTVerificationException) {
                null
            }

            if (email != null) {
                val user = userRepository.findByEmail(email)
                if (user != null) {
                    SecurityContextHolder.getContext().authentication =
                        UsernamePasswordAuthenticationToken(user, null, user.authorities)
                }
            }
        }

        filterChain.doFilter(request, response)
    }

    private fun recoverToken(request: HttpServletRequest): String? {
        val authHeader = request.getHeader("Authorization") ?: return null
        if (!authHeader.startsWith("Bearer ")) return null
        return authHeader.removePrefix("Bearer ").trim().ifBlank { null }
    }
}
```

Two fixes beyond the try/catch: the token is authenticated with `user.authorities` instead of `emptyList()`, and `removePrefix` replaces `replace("Bearer ", "")` so a token containing that substring is not mangled.

- [ ] **Step 6: Make anonymous requests return 401**

Replace `config/SecurityConfig.kt`:

```kotlin
package com.skydex.api.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.skydex.api.dto.ErrorResponse
import com.skydex.api.security.SecurityFilter
import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.http.MediaType
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.web.AuthenticationEntryPoint
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter

@Configuration
class SecurityConfig(
    private val securityFilter: SecurityFilter,
    private val objectMapper: ObjectMapper
) {

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()

    /** Returns a JSON 401 instead of Spring's default 403 for unauthenticated requests. */
    @Bean
    fun authenticationEntryPoint(): AuthenticationEntryPoint =
        AuthenticationEntryPoint { _, response, _ ->
            response.status = HttpServletResponse.SC_UNAUTHORIZED
            response.contentType = MediaType.APPLICATION_JSON_VALUE
            response.characterEncoding = "UTF-8"
            response.writer.write(objectMapper.writeValueAsString(ErrorResponse("Authentication required")))
        }

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth.requestMatchers("/auth/**").permitAll()
                auth.requestMatchers("/error").permitAll()
                auth.requestMatchers(HttpMethod.GET, "/api/photos/**").permitAll()
                auth.anyRequest().authenticated()
            }
            .exceptionHandling { it.authenticationEntryPoint(authenticationEntryPoint()) }
            .addFilterBefore(securityFilter, UsernamePasswordAuthenticationFilter::class.java)

        return http.build()
    }
}
```

`GET /api/photos/**` is public so Coil can render capture photos without attaching a header. Filenames are random UUIDs; making photo reads authenticated is a post-MVP hardening item.

- [ ] **Step 7: Enforce ownership on events**

In `WeatherEventController.kt`, replace `update` and `delete`, and switch the not-found paths to exceptions:

```kotlin
    @GetMapping("/{id}")
    fun getById(@PathVariable id: UUID): WeatherEventResponse {
        val event = events.findById(id).orElseThrow { NotFoundException("Capture not found") }
        // Author comes from the event, never from the caller: this endpoint has no ownership
        // restriction, so the two genuinely differ here. The test below pins that.
        val author = users.findById(event.userId).orElseThrow { NotFoundException("Capture not found") }
        return WeatherEventResponse.from(event, author)
    }

    @PutMapping("/{id}")
    fun update(
        @AuthenticationPrincipal currentUser: User,
        @PathVariable id: UUID,
        @Valid @RequestBody request: CreateWeatherEventRequest
    ): WeatherEventResponse {
        val event = events.findById(id).orElseThrow { NotFoundException("Capture not found") }
        if (event.userId != currentUser.id) {
            throw ForbiddenException("You can only modify your own captures")
        }
        event.title = request.title
        event.description = request.description
        event.photoUrl = request.photoUrl
        // Safe to pass currentUser as the author ONLY because the guard above proves
        // currentUser.id == event.userId. If that guard is ever relaxed — a moderator edit, a
        // shared album — this must go back to looking the author up from event.userId, or the
        // response will attribute the capture to whoever edited it.
        return WeatherEventResponse.from(events.save(event), currentUser)
    }

    @DeleteMapping("/{id}")
    fun delete(
        @AuthenticationPrincipal currentUser: User,
        @PathVariable id: UUID
    ): ResponseEntity<Void> {
        val event = events.findById(id).orElseThrow { NotFoundException("Capture not found") }
        if (event.userId != currentUser.id) {
            throw ForbiddenException("You can only modify your own captures")
        }
        events.delete(event)
        return ResponseEntity.noContent().build()
    }
```

Add imports `com.skydex.api.errors.ForbiddenException` and `com.skydex.api.errors.NotFoundException`.

- [ ] **Step 8: Replace the user `{id}` routes with `/me`**

Replace the body of `UserController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.UpdateProfileRequest
import com.skydex.api.dto.UserResponse
import com.skydex.api.dto.WeatherEventResponse
import com.skydex.api.errors.ConflictException
import com.skydex.api.models.User
import com.skydex.api.repositories.UserRepository
import com.skydex.api.repositories.WeatherEventRepository
import jakarta.validation.Valid
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PutMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/users")
class UserController(
    private val users: UserRepository,
    private val events: WeatherEventRepository
) {

    @GetMapping("/me")
    fun me(@AuthenticationPrincipal currentUser: User): UserResponse =
        UserResponse.from(currentUser)

    @PutMapping("/me")
    fun updateMe(
        @AuthenticationPrincipal currentUser: User,
        @Valid @RequestBody request: UpdateProfileRequest
    ): UserResponse {
        val existing = users.findByEmail(request.email)
        if (existing != null && existing.id != currentUser.id) {
            throw ConflictException("Email already registered")
        }
        currentUser.name = request.name
        currentUser.email = request.email
        return UserResponse.from(users.save(currentUser))
    }

    @DeleteMapping("/me")
    fun deleteMe(@AuthenticationPrincipal currentUser: User): ResponseEntity<Void> {
        events.findByUserIdOrderByCapturedAtDesc(currentUser.id!!).forEach { events.delete(it) }
        users.delete(currentUser)
        return ResponseEntity.noContent().build()
    }

    // SUPERSEDED — Task 4, Step 0 deletes this handler. It duplicates GET /api/events/mine
    // (Task 2) verbatim: same finder, same mapping, same ordering. Kept in this text because
    // Task 3 shipped it; do not reintroduce it in later tasks.
    @GetMapping("/me/events")
    fun myEvents(@AuthenticationPrincipal currentUser: User): List<WeatherEventResponse> =
        events.findByUserIdOrderByCapturedAtDesc(currentUser.id!!)
            .map { WeatherEventResponse.from(it, currentUser) }
}
```

Update any test still calling `/api/users/{id}` to use `/api/users/me` with the matching bearer header.

- [ ] **Step 9: Run the tests and confirm green**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS. The ownership tests now get 403, the malformed-token test gets 401, `/api/users/me` returns the caller.

- [ ] **Step 10: Manually confirm the destructive endpoints are gone**

```bash
docker start skydex-db && sleep 3
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun &
sleep 25
curl -s -o /dev/null -w '%{http_code}\n' -X DELETE http://localhost:8080/api/events
curl -s -o /dev/null -w '%{http_code}\n' -X DELETE http://localhost:8080/api/users
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/api/users
```

Expected: `401` for all three (no token), and after logging in and retrying with a valid token, `405` or `404` — never `200`. Stop the server afterwards.

- [ ] **Step 11: Commit**

```bash
git add -A src
git commit -m "fix!: enforce capture ownership and return 401 for bad tokens

Adds ownership checks to PUT/DELETE /api/events/{id}, replaces the
user {id} routes with /me, swallows invalid JWTs instead of 500ing,
and centralises error bodies in a RestControllerAdvice."
```

---

### Task 4: Android session persistence and automatic auth headers

The JWT currently lives in a `remember { mutableStateOf("") }` inside `MainActivity`, so the user is logged out by every process death, and every API call has to hand-thread a `token` string down through composables. This task makes the session durable and makes the header automatic.

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/UserController.kt` (Step 0)
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/UserControllerTest.kt` (Step 0)
- Modify: `SkyDex---frontend/gradle/libs.versions.toml`
- Modify: `SkyDex---frontend/app/build.gradle.kts`
- Modify: `SkyDex---frontend/app/src/main/AndroidManifest.xml`
- Modify: `SkyDex---frontend/app/src/main/res/xml/backup_rules.xml`
- Modify: `SkyDex---frontend/app/src/main/res/xml/data_extraction_rules.xml`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/SkyDexApplication.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ServiceLocator.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/session/SessionStore.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/AuthInterceptor.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/ApiFactory.kt`
- Move: `.../SkyDexApi.kt` → `.../data/remote/SkyDexApi.kt`
- Move: `.../dto/AuthDTO.kt` → `.../data/remote/dto/AuthDto.kt`
- Move: `.../dto/EventoDTO.kt` → `.../data/remote/dto/WeatherEventDto.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/ResultOf.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/AuthRepository.kt`
- Delete: `SkyDex---frontend/app/src/main/java/com/example/skydex/RetroFitClient.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/data/remote/AuthInterceptorTest.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/data/remote/SkyDexApiTest.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/data/session/SessionStoreTest.kt`

**Interfaces:**
- Consumes: backend routes from Tasks 2 and 3.
- Produces:
  - `data.session.Session(token: String, userId: String)`
  - `data.session.SessionStore(dataStore: DataStore<Preferences>)`, plus a `constructor(context: Context)` the app uses. With `val session: Flow<Session?>`, `suspend fun save(token: String, userId: String)`, `suspend fun clear()`, `fun blockingToken(): String?`
  - `data.repository.resultOf(block)` — the cancellation-safe `runCatching` every repository in this plan uses
  - `data.remote.AuthInterceptor(tokenProvider: () -> String?)` — an OkHttp `Interceptor`
  - `data.remote.ApiFactory.create(baseUrl: String, interceptor: AuthInterceptor): SkyDexApi`
  - `ServiceLocator` — object with `fun init(app: Application)`, `val sessionStore: SessionStore`, `val api: SkyDexApi`, `val authRepository: AuthRepository`. Tasks 5–17 add repositories here.
  - `data.repository.AuthRepository(api, sessionStore)` with `suspend fun login(email: String, password: String): Result<Unit>`, `suspend fun register(name: String, email: String, password: String): Result<Unit>`, `suspend fun logout()`, `val session: Flow<Session?>`
  - `data.remote.dto.LoginRequest(email, password)`, `LoginResponse(token: String, userId: String, name: String)`, `RegisterRequest(name, email, password)`
  - `data.remote.dto.CreateWeatherEventRequest(title, description, photoUrl)` and `WeatherEventResponse(id: String, title: String, description: String, photoUrl: String, capturedAt: String, userId: String, authorName: String)`. **Gson has no `Instant`/`UUID` adapter registered, so every timestamp and id is a `String` on the Android side.** Task 6 appends `latitude: Double, longitude: Double`; Task 13 appends `phenomenon`, `phenomenonName`, `rarity`, `validationStatus`, `xpAwarded`.
  - `SkyDexApi` — no method takes an `@Header("Authorization")` parameter any more.

- [ ] **Step 0: Delete the duplicated my-events endpoint (backend)**

Task 3 shipped `GET /api/users/me/events`, which is a verbatim duplicate of `GET /api/events/mine` from Task 2: same finder call, same `WeatherEventResponse.from` mapping, same ordering. Two routes returning identical bytes is two places to fix every future change to the shape of a capture list — and Tasks 6 and 13 both widen that shape.

Keep `/api/events/mine`. It sits on the resource it returns, and its test asserts the *ownership scoping* (a second user's event must not appear), while the `/me/events` test asserts only `$.length()`, which a `findAll()` regression would satisfy identically.

In `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/UserController.kt`, delete the `myEvents` handler and its `@GetMapping("/me/events")`. If `WeatherEventRepository` and `WeatherEventResponse` are then unused in that file, drop those imports and the constructor parameter too — but check first: `deleteMe` still uses the repository, so the constructor parameter stays.

In `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/UserControllerTest.kt`, delete the `lists the authenticated user's events and returns 200` test. Do not port its assertions to `/api/events/mine` — that endpoint's own test in `WeatherEventControllerTest` is already strictly stronger.

Run the backend suite and confirm it stays green at 26 (27 minus the deleted test):

```bash
cd SkyDex-backend && JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Commit this separately from the Android work — it is a backend change and belongs in its own commit in the other repository.

- [ ] **Step 1: Write the failing interceptor test**

Create `SkyDex---frontend/app/src/test/java/com/example/skydex/data/remote/AuthInterceptorTest.kt`:

```kotlin
package com.example.skydex.data.remote

import okhttp3.Interceptor
import okhttp3.Protocol
import okhttp3.Request
import okhttp3.Response
import okhttp3.ResponseBody.Companion.toResponseBody
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNull
import org.junit.Test

class AuthInterceptorTest {

    private fun runWith(token: String?): Request {
        var seen: Request? = null
        val interceptor = AuthInterceptor { token }
        val chain = object : Interceptor.Chain {
            private val original = Request.Builder().url("http://localhost/api/events/mine").build()
            override fun request(): Request = original
            override fun proceed(request: Request): Response {
                seen = request
                return Response.Builder()
                    .request(request)
                    .protocol(Protocol.HTTP_1_1)
                    .code(200)
                    .message("OK")
                    .body("".toResponseBody(null))
                    .build()
            }
            override fun connection() = null
            override fun call() = throw UnsupportedOperationException()
            override fun connectTimeoutMillis() = 0
            override fun withConnectTimeout(timeout: Int, unit: java.util.concurrent.TimeUnit) = this
            override fun readTimeoutMillis() = 0
            override fun withReadTimeout(timeout: Int, unit: java.util.concurrent.TimeUnit) = this
            override fun writeTimeoutMillis() = 0
            override fun withWriteTimeout(timeout: Int, unit: java.util.concurrent.TimeUnit) = this
        }
        interceptor.intercept(chain)
        return seen!!
    }

    @Test
    fun `attaches a bearer header when a token is available`() {
        assertEquals("Bearer abc123", runWith("abc123").header("Authorization"))
    }

    @Test
    fun `sends no authorization header when logged out`() {
        assertNull(runWith(null).header("Authorization"))
    }

    @Test
    fun `does not double-prefix a token that already says Bearer`() {
        assertEquals("Bearer abc123", runWith("Bearer abc123").header("Authorization"))
    }
}
```

- [ ] **Step 2: Run it and watch it fail**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest
```

Expected: compilation failure — `Unresolved reference: AuthInterceptor`.

- [ ] **Step 3: Add the new dependencies**

In `gradle/libs.versions.toml`, add under `[versions]`:

```toml
datastore = "1.1.1"
navigationCompose = "2.8.5"
coroutinesTest = "1.8.1"
```

and under `[libraries]`:

```toml
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutinesTest" }
```

In `app/build.gradle.kts`, inside `dependencies { }`:

```kotlin
    implementation(libs.androidx.datastore.preferences)
    implementation(libs.androidx.navigation.compose)
    testImplementation(libs.kotlinx.coroutines.test)
```

If Gradle cannot resolve one of these exact versions, bump it to the newest stable release the resolver offers rather than downgrading anything else.

- [ ] **Step 4: Write the interceptor**

Create `app/src/main/java/com/example/skydex/data/remote/AuthInterceptor.kt`:

```kotlin
package com.example.skydex.data.remote

import okhttp3.Interceptor
import okhttp3.Response

/**
 * Attaches the stored JWT to every outgoing request. The token is read lazily through
 * [tokenProvider] so a login or logout takes effect on the very next call.
 */
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider()?.takeIf { it.isNotBlank() }
            ?: return chain.proceed(chain.request())

        val value = if (token.startsWith("Bearer ")) token else "Bearer $token"
        val authorized = chain.request().newBuilder()
            .header("Authorization", value)
            .build()
        return chain.proceed(authorized)
    }
}
```

- [ ] **Step 5: Run the interceptor test and confirm it passes**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.data.remote.AuthInterceptorTest"
```

Expected: 3 tests, all PASS.

- [ ] **Step 6: Write the session store**

Create `app/src/main/java/com/example/skydex/data/session/SessionStore.kt`:

```kotlin
package com.example.skydex.data.session

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.runBlocking

data class Session(val token: String, val userId: String)

private val Context.sessionDataStore by preferencesDataStore(name = "skydex_session")

/**
 * Takes the [DataStore] rather than the [Context] so the persistence round trip is testable on
 * the JVM: a test can hand it `PreferenceDataStoreFactory.create { tempFile }` and assert that a
 * saved session reads back. Constructing it from a `Context` — what the app does — is the
 * secondary constructor below. Session survival is the entire point of this task, and with no
 * device available it would otherwise be verified by nothing at all.
 */
class SessionStore(private val dataStore: DataStore<Preferences>) {

    constructor(context: Context) : this(context.applicationContext.sessionDataStore)

    private object Keys {
        val TOKEN = stringPreferencesKey("token")
        val USER_ID = stringPreferencesKey("user_id")
    }

    val session: Flow<Session?> = dataStore.data.map { prefs ->
        val token = prefs[Keys.TOKEN]
        val userId = prefs[Keys.USER_ID]
        if (token.isNullOrBlank() || userId.isNullOrBlank()) null else Session(token, userId)
    }

    suspend fun save(token: String, userId: String) {
        dataStore.edit { prefs ->
            prefs[Keys.TOKEN] = token
            prefs[Keys.USER_ID] = userId
        }
    }

    suspend fun clear() {
        dataStore.edit { it.clear() }
    }

    /**
     * Read the token from OkHttp's blocking interceptor thread. Never call this from the
     * main thread — OkHttp interceptors always run on a background dispatcher.
     *
     * Nothing may escape this method. OkHttp's `AsyncCall.execute` reports a throwing
     * interceptor to the callback and then **rethrows on its own pool thread**, where it reaches
     * the default uncaught-exception handler and kills the process. A missing header producing a
     * clean 401 is strictly better than a crash, so every failure — a corrupt DataStore file, an
     * uninitialised ServiceLocator — degrades to `null` here.
     *
     * `runCatching` is correct in this one place, unlike in the repositories: `runBlocking`
     * starts its own event loop, so there is no caller coroutine whose cancellation could be
     * swallowed. See [resultOf] for why the repositories must not use it.
     */
    fun blockingToken(): String? = runCatching { runBlocking { session.first()?.token } }.getOrNull()
}
```

- [ ] **Step 7: Move and rewrite the wire DTOs**

```bash
cd <workspace>/SkyDex---frontend/app/src/main/java/com/example/skydex
mkdir -p data/remote/dto data/repository data/session
git mv dto/AuthDTO.kt data/remote/dto/AuthDto.kt
git mv dto/EventoDTO.kt data/remote/dto/WeatherEventDto.kt
git mv SkyDexApi.kt data/remote/SkyDexApi.kt
git rm RetroFitClient.kt
rmdir dto
```

`data/remote/dto/AuthDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class LoginRequest(
    val email: String,
    val password: String
)

data class LoginResponse(
    val token: String,
    val userId: String,
    val name: String
)

data class RegisterRequest(
    val name: String,
    val email: String,
    val password: String
)

data class UserResponse(
    val id: String,
    val name: String,
    val email: String,
    val joinedAt: String
)

data class ErrorResponse(
    val error: String
)
```

`data/remote/dto/WeatherEventDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class CreateWeatherEventRequest(
    val title: String,
    val description: String,
    val photoUrl: String
)

data class WeatherEventResponse(
    val id: String,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: String,
    val userId: String,
    val authorName: String
)

data class NearbyPhenomenonResponse(
    val phenomenon: String,
    val time: String,
    val temperatureCelsius: Double?,
    val alertLevel: String
)
```

`NearbyPhenomenonResponse` moves here out of `NearEvents.kt`, where it currently lives as `EventoProximoDTO` inside a UI file.

- [ ] **Step 8: Rewrite the API interface without manual headers**

`data/remote/SkyDexApi.kt`:

```kotlin
package com.example.skydex.data.remote

import com.example.skydex.data.remote.dto.CreateWeatherEventRequest
import com.example.skydex.data.remote.dto.LoginRequest
import com.example.skydex.data.remote.dto.LoginResponse
import com.example.skydex.data.remote.dto.NearbyPhenomenonResponse
import com.example.skydex.data.remote.dto.RegisterRequest
import com.example.skydex.data.remote.dto.UserResponse
import com.example.skydex.data.remote.dto.WeatherEventResponse
import retrofit2.http.Body
import retrofit2.http.DELETE
import retrofit2.http.GET
import retrofit2.http.POST
import retrofit2.http.Path
import retrofit2.http.Query

interface SkyDexApi {

    @POST("auth/login")
    suspend fun login(@Body request: LoginRequest): LoginResponse

    @POST("auth/register")
    suspend fun register(@Body request: RegisterRequest): UserResponse

    @GET("api/users/me")
    suspend fun me(): UserResponse

    @GET("api/events/mine")
    suspend fun myCaptures(): List<WeatherEventResponse>

    @POST("api/events")
    suspend fun createCapture(@Body request: CreateWeatherEventRequest): WeatherEventResponse

    @DELETE("api/events/{id}")
    suspend fun deleteCapture(@Path("id") id: String)

    @GET("api/weather/nearby")
    suspend fun nearbyPhenomena(
        @Query("lat") latitude: Double,
        @Query("lon") longitude: Double
    ): List<NearbyPhenomenonResponse>
}
```

Leading slashes are dropped from the paths so they resolve relative to `BASE_URL`. `BASE_URL` in `local.properties` has no trailing slash, so also make sure `ApiFactory` appends one (next step).

- [ ] **Step 9: Write the Retrofit factory**

Create `app/src/main/java/com/example/skydex/data/remote/ApiFactory.kt`:

```kotlin
package com.example.skydex.data.remote

import com.example.skydex.BuildConfig
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.util.concurrent.TimeUnit

object ApiFactory {

    /**
     * Takes the base URL instead of reading [BuildConfig] internally so a test can point it at a
     * fake host and actually execute this function. Reading the constant here would make the
     * whole factory — including the trailing-slash fix-up below, without which Retrofit throws
     * `IllegalArgumentException: baseUrl must end in /` — unreachable from any JVM test.
     */
    fun create(baseUrl: String, interceptor: AuthInterceptor): SkyDexApi {
        val logging = HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
            else HttpLoggingInterceptor.Level.NONE
            // BODY logging prints request headers, and the bearer token is a live credential:
            // without this it lands in `adb logcat` and in every captured bug report. Redaction
            // still shows the header was present, which is what device debugging actually needs.
            redactHeader("Authorization")
        }

        val client = OkHttpClient.Builder()
            .addInterceptor(interceptor)
            .addInterceptor(logging)
            .connectTimeout(15, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            // connect/read timeouts do NOT cover time spent inside an application interceptor,
            // and AuthInterceptor blocks on a DataStore read. Without a call timeout, a wedged
            // disk read hangs the request forever.
            .callTimeout(45, TimeUnit.SECONDS)
            .build()

        return Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL.trimEnd('/') + "/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(SkyDexApi::class.java)
    }
}
```

The auth interceptor is added *before* the logging interceptor so the log shows the header that was actually sent.

- [ ] **Step 10: Write the auth repository**

First create `app/src/main/java/com/example/skydex/data/repository/ResultOf.kt`. Every repository in this plan — Tasks 4, 5, 7, 13, 15, 16 and 18 — wraps its network calls in this helper instead of `runCatching`:

```kotlin
package com.example.skydex.data.repository

import kotlin.coroutines.cancellation.CancellationException

/**
 * Like [runCatching], but never swallows coroutine cancellation.
 *
 * `runCatching` catches [Throwable], which includes the [CancellationException] the coroutine
 * machinery throws to unwind a cancelled job. Repositories are called from composable scopes that
 * cancel whenever the user navigates away, so a plain `runCatching` would convert an ordinary
 * cancellation into `Result.failure` — breaking structured concurrency and, worse, surfacing
 * "invalid credentials" to a user who simply left the screen. Rethrowing keeps cancellation a
 * control-flow signal rather than an error.
 */
inline fun <T> resultOf(block: () -> T): Result<T> = try {
    Result.success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: Throwable) {
    Result.failure(e)
}
```

It lives in package `com.example.skydex.data.repository`, so the repositories in that package call it without an import.

Then create `app/src/main/java/com/example/skydex/data/repository/AuthRepository.kt`:

```kotlin
package com.example.skydex.data.repository

import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.remote.dto.LoginRequest
import com.example.skydex.data.remote.dto.RegisterRequest
import com.example.skydex.data.session.Session
import com.example.skydex.data.session.SessionStore
import kotlinx.coroutines.flow.Flow

class AuthRepository(
    private val api: SkyDexApi,
    private val sessionStore: SessionStore
) {

    val session: Flow<Session?> = sessionStore.session

    suspend fun login(email: String, password: String): Result<Unit> = resultOf {
        val response = api.login(LoginRequest(email.trim(), password))
        sessionStore.save(response.token, response.userId)
    }

    suspend fun register(name: String, email: String, password: String): Result<Unit> = resultOf {
        api.register(RegisterRequest(name.trim(), email.trim(), password))
    }

    suspend fun logout() {
        sessionStore.clear()
    }
}
```

- [ ] **Step 11: Wire it all up in an Application subclass**

Create `app/src/main/java/com/example/skydex/ServiceLocator.kt`:

```kotlin
package com.example.skydex

import android.content.Context
import com.example.skydex.BuildConfig
import com.example.skydex.data.remote.ApiFactory
import com.example.skydex.data.remote.AuthInterceptor
import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.repository.AuthRepository
import com.example.skydex.data.session.SessionStore

/**
 * Hand-rolled dependency graph. The app is small enough that a DI framework would cost more
 * than it saves; if this grows past a dozen entries, replace it with Hilt.
 */
object ServiceLocator {

    private lateinit var appContext: Context

    fun init(context: Context) {
        appContext = context.applicationContext
    }

    /**
     * A bare `UninitializedPropertyAccessException` from a `lateinit` gives no hint about what
     * went wrong. `SkyDexApplication.onCreate` covers every UI path, but a ContentProvider —
     * `androidx.startup`, or any library that installs one — runs *before* it, so this is
     * reachable. Say so plainly rather than leaving a mystery crash.
     */
    private fun requireContext(): Context {
        check(::appContext.isInitialized) {
            "ServiceLocator.init() was never called — is SkyDexApplication registered in the manifest?"
        }
        return appContext
    }

    val sessionStore: SessionStore by lazy { SessionStore(requireContext()) }

    val api: SkyDexApi by lazy {
        ApiFactory.create(BuildConfig.BASE_URL, AuthInterceptor { sessionStore.blockingToken() })
    }

    val authRepository: AuthRepository by lazy { AuthRepository(api, sessionStore) }
}
```

Create `app/src/main/java/com/example/skydex/SkyDexApplication.kt`:

```kotlin
package com.example.skydex

import android.app.Application

class SkyDexApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        ServiceLocator.init(this)
    }
}
```

In `app/src/main/AndroidManifest.xml`, add `android:name=".SkyDexApplication"` as the first attribute of `<application>`:

```xml
    <application
        android:name=".SkyDexApplication"
        android:allowBackup="true"
```

Then keep the session token out of Android's backup system. `allowBackup="true"` stays — later tasks will have local data worth restoring — but the DataStore file now holds a live bearer credential, and both rule files that would control this are stock scaffolding with every rule commented out. Under the defaults, the token is copied to cloud backup and to device-to-device transfer.

`minSdk` is 26 and `targetSdk` is 36, so **both** files apply: `backup_rules.xml` governs API 26–30, `data_extraction_rules.xml` governs 31+.

Replace the body of `app/src/main/res/xml/backup_rules.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <!-- Holds the session JWT. Nothing under datastore/ should leave the device. -->
    <exclude domain="file" path="datastore/" />
</full-backup-content>
```

and `app/src/main/res/xml/data_extraction_rules.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="file" path="datastore/" />
    </cloud-backup>
    <device-transfer>
        <exclude domain="file" path="datastore/" />
    </device-transfer>
</data-extraction-rules>
```

The `<device-transfer>` block is not optional and not redundant. An exclusion placed only under `<cloud-backup>` — or at the root — still lets the token ride along in a phone-to-phone transfer, which is the half that gets forgotten.

`preferencesDataStore(name = "skydex_session")` writes to `filesDir/datastore/skydex_session.preferences_pb`, and `domain="file"` maps to `getFilesDir()`, so the pair above is correct. Excluding the whole `datastore/` directory rather than the one file means a future DataStore is private by default; narrow it if one ever genuinely needs backing up.

**This cannot be verified without hardware.** Malformed rule files are silently ignored, so inspection proves nothing. When a device is available, confirm with `adb shell bmgr backupnow com.example.skydex` followed by `adb shell dumpsys backup`, and record the result.

- [ ] **Step 12: Point the existing screens at the new API object**

The five files under `ui/theme/pages/` still call `RetrofitClient.api.…` with a `token` argument. Task 5 moves and rewrites them properly; for now make them compile by:

- replacing `import com.example.skydex.RetrofitClient` with `import com.example.skydex.ServiceLocator`,
- replacing `RetrofitClient.api` with `ServiceLocator.api`,
- dropping the `token` argument from every call site,
- in `Login.kt`, replacing the `onLoginSuccess(formatoToken, resposta.userId)` body with a call through `ServiceLocator.authRepository.login(email, password)` and deleting `seTokenJaTemBearer`,
- in `NearEvents.kt`, deleting the local `EventoProximoDTO` declaration and importing `com.example.skydex.data.remote.dto.NearbyPhenomenonResponse` in its place (rename the field accesses: `fenomeno` → `phenomenon`, `horario` → `time`, `temperatura` → `temperatureCelsius`, `nivelAlerta` → `alertLevel`),
- in `Registers.kt`, importing `com.example.skydex.data.remote.dto.WeatherEventResponse` and renaming the field accesses (`urlFoto` → `photoUrl`, `titulo` → `title`, `descricao` → `description`, `dataHoraRegistro` → `capturedAt`), and switching the call to `ServiceLocator.api.myCaptures()`,
- in `HomePage.kt`, importing `com.example.skydex.data.remote.dto.CreateWeatherEventRequest` and dropping its `userId` argument.

- [ ] **Step 13: Compile and run all Android tests**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, `AuthInterceptorTest` green.

- [ ] **Step 14: Verify session persistence on the phone**

```bash
docker start skydex-db && sleep 3
(cd ../SkyDex-backend && JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun) &
sleep 25
~/Android/Sdk/platform-tools/adb devices          # the phone must be listed
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:installDebug
```

Set `API_BASE_URL` in `local.properties` to your machine's LAN address first (`hostname -I` gives it) — the phone cannot reach `localhost`.

On the phone: register, log in, force-stop the app from Android settings, reopen it. **Expected: the app opens straight into the app shell, not the login screen.** (The auto-login *routing* lands in Task 5; for now confirm via Logcat that requests still carry `Authorization: Bearer …` after the restart.)

- [ ] **Step 15: Commit**

```bash
git add -A app
git commit -m "feat: persist the session in DataStore and inject auth headers

Replaces the in-memory token held by MainActivity with a DataStore-backed
SessionStore and an OkHttp interceptor, and introduces a ServiceLocator
so screens stop constructing their own Retrofit calls."
```

---

### Task 5: Navigation Compose, ViewModels, and a real package layout

`MainActivity` currently owns a `telaAtual` string, a nested `when`, and two `Scaffold`s; screens live under `ui/theme/pages/`; every screen does its own `LaunchedEffect` + `try/catch` and loses its state on rotation. Three more screens are coming in Phases 3 and 4, so this is the moment to fix the structure.

**Files:**
- Create: `.../ui/navigation/Routes.kt`
- Create: `.../ui/navigation/SkyDexNavHost.kt`
- Create: `.../ui/components/AppBottomBar.kt`
- Create: `.../ui/auth/LoginScreen.kt`, `.../ui/auth/LoginViewModel.kt`
- Create: `.../ui/auth/RegisterScreen.kt`, `.../ui/auth/RegisterViewModel.kt`
- Create: `.../ui/home/HomeScreen.kt`, `.../ui/home/HomeViewModel.kt`
- Create: `.../ui/captures/MyCapturesScreen.kt`, `.../ui/captures/MyCapturesViewModel.kt`
- Create: `.../data/repository/CaptureRepository.kt`
- Modify: `.../MainActivity.kt`
- Modify: `.../ServiceLocator.kt`
- Delete: `.../ui/theme/pages/HomePage.kt`, `Login.kt`, `Signin.kt`, `NearEvents.kt`, `Registers.kt`
- Test: `.../app/src/test/java/com/example/skydex/ui/auth/LoginViewModelTest.kt`
- Test: `.../app/src/test/java/com/example/skydex/ui/home/HomeViewModelTest.kt`

**Interfaces:**
- Consumes: `ServiceLocator`, `AuthRepository`, `SessionStore`, `Session`, `SkyDexApi` (Task 4).
- Produces:
  - `ui.navigation.Routes` — object with `const val LOGIN = "login"`, `REGISTER = "register"`, `HOME = "home"`, `NEARBY = "nearby"`, `MY_CAPTURES = "my_captures"`. Tasks 10/14/17 add `CAPTURE`, `SKYDEX`, `FEED`, `FRIENDS`.
  - `ui.navigation.SkyDexNavHost(session: Session?, onLoggedOut: () -> Unit)` — composable owning the `NavHost`.
  - `ui.components.AppBottomBar(currentRoute: String, onNavigate: (String) -> Unit)`.
  - `data.repository.CaptureRepository(api)` with `suspend fun myCaptures(): Result<List<WeatherEventResponse>>`, `suspend fun nearby(latitude: Double, longitude: Double): Result<List<NearbyPhenomenonResponse>>`, `suspend fun create(request: CreateWeatherEventRequest): Result<WeatherEventResponse>`, `suspend fun delete(id: String): Result<Unit>`. **Task 7 adds `suspend fun uploadPhoto(file: File): Result<String>`.**
  - `ui.common.UiState<T>` — sealed interface with `Loading`, `Success(data: T)`, `Error(message: String)`. Every screen ViewModel in Tasks 5–17 exposes `StateFlow<UiState<…>>`.
  - `ServiceLocator.captureRepository`.

- [ ] **Step 1: Write the failing ViewModel tests**

Create `app/src/test/java/com/example/skydex/ui/auth/LoginViewModelTest.kt`:

```kotlin
package com.example.skydex.ui.auth

import com.example.skydex.data.remote.dto.LoginRequest
import com.example.skydex.data.remote.dto.LoginResponse
import com.example.skydex.data.repository.AuthRepository
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class LoginViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    @Test
    fun `successful login flips the state to LoggedIn`() = runTest(dispatcher) {
        val repository = FakeAuthRepository(result = Result.success(Unit))
        val viewModel = LoginViewModel(repository)

        viewModel.onEmailChanged("pilot@skydex.com")
        viewModel.onPasswordChanged("super-safe-password")
        viewModel.submit()
        advanceUntilIdle()

        assertTrue(viewModel.state.value.loggedIn)
        assertEquals(null, viewModel.state.value.errorMessage)
    }

    @Test
    fun `a failed login surfaces a message and stays logged out`() = runTest(dispatcher) {
        val repository = FakeAuthRepository(result = Result.failure(IOException("boom")))
        val viewModel = LoginViewModel(repository)

        viewModel.onEmailChanged("pilot@skydex.com")
        viewModel.onPasswordChanged("wrong")
        viewModel.submit()
        advanceUntilIdle()

        assertEquals(false, viewModel.state.value.loggedIn)
        assertEquals("Credenciais inválidas ou servidor indisponível.", viewModel.state.value.errorMessage)
    }

    @Test
    fun `submitting with a blank field does not call the repository`() = runTest(dispatcher) {
        val repository = FakeAuthRepository(result = Result.success(Unit))
        val viewModel = LoginViewModel(repository)

        viewModel.onEmailChanged("pilot@skydex.com")
        viewModel.submit()
        advanceUntilIdle()

        assertEquals(0, repository.loginCalls)
        assertEquals("Preencha e-mail e senha.", viewModel.state.value.errorMessage)
    }
}
```

Create the fake in the same test source set, `app/src/test/java/com/example/skydex/data/repository/FakeAuthRepository.kt`:

```kotlin
package com.example.skydex.ui.auth

import com.example.skydex.data.repository.AuthRepository

/**
 * AuthRepository is a final class, so the fake subclasses nothing — it re-implements the
 * two methods LoginViewModel touches behind the AuthGateway interface.
 */
class FakeAuthRepository(private val result: Result<Unit>) : AuthGateway {
    var loginCalls = 0
        private set

    override suspend fun login(email: String, password: String): Result<Unit> {
        loginCalls++
        return result
    }

    override suspend fun register(name: String, email: String, password: String): Result<Unit> = result

    override suspend fun logout() = Unit
}
```

Introducing `AuthGateway` is what makes the ViewModel testable without Mockito; the next step defines it.

- [ ] **Step 2: Run the tests and watch them fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.auth.*"
```

Expected: compilation failure — `Unresolved reference: LoginViewModel`, `Unresolved reference: AuthGateway`.

- [ ] **Step 3: Extract the gateway interface and the shared UI state**

Create `app/src/main/java/com/example/skydex/ui/auth/AuthGateway.kt`:

```kotlin
package com.example.skydex.ui.auth

interface AuthGateway {
    suspend fun login(email: String, password: String): Result<Unit>
    suspend fun register(name: String, email: String, password: String): Result<Unit>
    suspend fun logout()
}
```

Make `AuthRepository` implement it — change its declaration in `data/repository/AuthRepository.kt` to:

```kotlin
class AuthRepository(
    private val api: SkyDexApi,
    private val sessionStore: SessionStore
) : AuthGateway {
```

and prefix `login`, `register` and `logout` with `override`.

Create `app/src/main/java/com/example/skydex/ui/common/UiState.kt`:

```kotlin
package com.example.skydex.ui.common

sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}
```

- [ ] **Step 4: Write the login ViewModel and screen**

`app/src/main/java/com/example/skydex/ui/auth/LoginViewModel.kt`:

```kotlin
package com.example.skydex.ui.auth

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val submitting: Boolean = false,
    val loggedIn: Boolean = false,
    val errorMessage: String? = null
)

class LoginViewModel(private val auth: AuthGateway) : ViewModel() {

    private val _state = MutableStateFlow(LoginUiState())
    val state: StateFlow<LoginUiState> = _state.asStateFlow()

    fun onEmailChanged(value: String) = _state.update { it.copy(email = value, errorMessage = null) }

    fun onPasswordChanged(value: String) = _state.update { it.copy(password = value, errorMessage = null) }

    fun submit() {
        val current = _state.value
        if (current.email.isBlank() || current.password.isBlank()) {
            _state.update { it.copy(errorMessage = "Preencha e-mail e senha.") }
            return
        }

        _state.update { it.copy(submitting = true, errorMessage = null) }
        viewModelScope.launch {
            val result = auth.login(current.email, current.password)
            _state.update {
                if (result.isSuccess) {
                    it.copy(submitting = false, loggedIn = true)
                } else {
                    it.copy(
                        submitting = false,
                        errorMessage = "Credenciais inválidas ou servidor indisponível."
                    )
                }
            }
        }
    }
}
```

`app/src/main/java/com/example/skydex/ui/auth/LoginScreen.kt` — the same visual design as the current `Login.kt`, driven by the ViewModel:

```kotlin
package com.example.skydex.ui.auth

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.collectAsState
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

@Composable
fun LoginScreen(
    viewModel: LoginViewModel,
    onLoggedIn: () -> Unit,
    onNavigateToRegister: () -> Unit,
    modifier: Modifier = Modifier
) {
    val state by viewModel.state.collectAsState()

    LaunchedEffect(state.loggedIn) {
        if (state.loggedIn) onLoggedIn()
    }

    Column(
        modifier = modifier.fillMaxSize().padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("SkyDex", fontSize = 36.sp, fontWeight = FontWeight.ExtraBold, color = Color(0xFF0284C7))
        Text(
            "Seu radar meteorológico pessoal",
            color = Color.Gray,
            modifier = Modifier.padding(bottom = 32.dp)
        )

        OutlinedTextField(
            value = state.email,
            onValueChange = viewModel::onEmailChanged,
            label = { Text("E-mail") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        Spacer(Modifier.height(16.dp))

        OutlinedTextField(
            value = state.password,
            onValueChange = viewModel::onPasswordChanged,
            label = { Text("Senha") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        Spacer(Modifier.height(24.dp))

        state.errorMessage?.let {
            Text(it, color = Color.Red, modifier = Modifier.padding(bottom = 16.dp))
        }

        Button(
            onClick = viewModel::submit,
            modifier = Modifier.fillMaxWidth().height(50.dp),
            enabled = !state.submitting
        ) {
            Text(if (state.submitting) "Entrando..." else "Entrar", fontSize = 16.sp)
        }

        Spacer(Modifier.height(16.dp))

        TextButton(onClick = onNavigateToRegister) {
            Text("Não tem uma conta? Registre-se")
        }
    }
}
```

- [ ] **Step 5: Run the login tests and confirm they pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.auth.*"
```

Expected: 3 tests PASS.

- [ ] **Step 6: Write the capture repository**

Create `app/src/main/java/com/example/skydex/data/repository/CaptureRepository.kt`:

```kotlin
package com.example.skydex.data.repository

import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.remote.dto.CreateWeatherEventRequest
import com.example.skydex.data.remote.dto.NearbyPhenomenonResponse
import com.example.skydex.data.remote.dto.WeatherEventResponse
import retrofit2.HttpException

class CaptureRepository(private val api: SkyDexApi) {

    suspend fun myCaptures(): Result<List<WeatherEventResponse>> =
        resultOf { api.myCaptures() }

    suspend fun nearby(latitude: Double, longitude: Double): Result<List<NearbyPhenomenonResponse>> =
        resultOf { api.nearbyPhenomena(latitude, longitude) }

    suspend fun create(request: CreateWeatherEventRequest): Result<WeatherEventResponse> =
        resultOf { api.createCapture(request) }

    // `deleteCapture` returns the raw Response, not Unit: Retrofit 2.9.0 throws
    // KotlinNullPointerException trying to map the backend's empty 204 body onto a non-null
    // `Unit` return type (verified by probe in Task 4). That means an unsuccessful status
    // arrives here as a perfectly normal value, so `resultOf` alone would report a 403 or a
    // 404 as a SUCCESSFUL delete. Convert it to a failure explicitly.
    suspend fun delete(id: String): Result<Unit> = resultOf {
        val response = api.deleteCapture(id)
        if (!response.isSuccessful) throw HttpException(response)
    }
}
```

Register it in `ServiceLocator`:

```kotlin
    val captureRepository: CaptureRepository by lazy { CaptureRepository(api) }
```

- [ ] **Step 7: Write the home and my-captures ViewModels and screens**

`ui/home/HomeViewModel.kt` — for now the home screen shows the nearby list at a fixed coordinate; Task 9 replaces the constant with real GPS:

```kotlin
package com.example.skydex.ui.home

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.NearbyPhenomenonResponse
import com.example.skydex.data.repository.CaptureRepository
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class HomeViewModel(private val captures: CaptureRepository) : ViewModel() {

    private val _state = MutableStateFlow<UiState<List<NearbyPhenomenonResponse>>>(UiState.Loading)
    val state: StateFlow<UiState<List<NearbyPhenomenonResponse>>> = _state.asStateFlow()

    fun load(latitude: Double, longitude: Double) {
        _state.value = UiState.Loading
        viewModelScope.launch {
            captures.nearby(latitude, longitude)
                .onSuccess { _state.value = UiState.Success(it) }
                .onFailure { _state.value = UiState.Error("Não foi possível carregar os eventos próximos.") }
        }
    }
}
```

`ui/captures/MyCapturesViewModel.kt`:

```kotlin
package com.example.skydex.ui.captures

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.data.repository.CaptureRepository
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class MyCapturesViewModel(private val captures: CaptureRepository) : ViewModel() {

    private val _state = MutableStateFlow<UiState<List<WeatherEventResponse>>>(UiState.Loading)
    val state: StateFlow<UiState<List<WeatherEventResponse>>> = _state.asStateFlow()

    init { refresh() }

    fun refresh() {
        _state.value = UiState.Loading
        viewModelScope.launch {
            captures.myCaptures()
                .onSuccess { _state.value = UiState.Success(it) }
                .onFailure { _state.value = UiState.Error("Não foi possível carregar seus registros.") }
        }
    }
}
```

For the screens: move the `LazyColumn` bodies of the current `NearEvents.kt` and `Registers.kt` into `ui/home/HomeScreen.kt` and `ui/captures/MyCapturesScreen.kt` verbatim, replacing their `LaunchedEffect` + local `remember` state with `val state by viewModel.state.collectAsState()` and a `when (state) { is UiState.Loading -> …; is UiState.Error -> …; is UiState.Success -> … }`. Keep `EventoCard`/`RegistroCard` as `PhenomenonCard`/`CaptureCard` in the same files, with field names updated to `phenomenon`/`time`/`temperatureCelsius`/`alertLevel` and `title`/`description`/`photoUrl`/`capturedAt`.

Do the same for `Signin.kt` → `ui/auth/RegisterScreen.kt` + `RegisterViewModel.kt`, mirroring `LoginViewModel` exactly but with a `name` field, a `registered: Boolean` flag instead of `loggedIn`, and the messages "Preencha todos os campos." and "Não foi possível registrar. O e-mail já existe?".

- [ ] **Step 8: Extract the bottom bar**

Create `ui/components/AppBottomBar.kt` with the `FooterSection` body from `HomePage.kt`, renamed and generalised:

```kotlin
package com.example.skydex.ui.components

import androidx.compose.animation.animateColorAsState
import androidx.compose.animation.core.animateDpAsState
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.navigationBarsPadding
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Dataset
import androidx.compose.material.icons.filled.Home
import androidx.compose.material.icons.filled.WbSunny
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.unit.dp
import com.example.skydex.ui.navigation.Routes

private data class BarItem(val route: String, val icon: ImageVector, val label: String)

@Composable
fun AppBottomBar(currentRoute: String, onNavigate: (String) -> Unit) {
    val items = listOf(
        BarItem(Routes.NEARBY, Icons.Default.WbSunny, "Eventos Próximos"),
        BarItem(Routes.HOME, Icons.Default.Home, "Início"),
        BarItem(Routes.MY_CAPTURES, Icons.Default.Dataset, "Meus Registros")
    )

    Row(
        modifier = Modifier
            .fillMaxWidth()
            .background(Color.White)
            .navigationBarsPadding()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        items.forEach { item ->
            val selected = currentRoute == item.route
            val tint by animateColorAsState(
                if (selected) Color(0xFF0284C7) else Color.Gray,
                label = "tint-${item.route}"
            )
            val size by animateDpAsState(
                if (selected) 36.dp else 28.dp,
                label = "size-${item.route}"
            )
            Column(
                modifier = Modifier.weight(1f),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                IconButton(onClick = { onNavigate(item.route) }) {
                    Icon(
                        modifier = Modifier.size(size),
                        imageVector = item.icon,
                        contentDescription = item.label,
                        tint = tint
                    )
                }
            }
        }
    }
}
```

Tasks 14 and 17 add `Routes.SKYDEX` and `Routes.FEED` entries to this `items` list.

- [ ] **Step 9: Write the routes and the NavHost**

`ui/navigation/Routes.kt`:

```kotlin
package com.example.skydex.ui.navigation

object Routes {
    const val LOGIN = "login"
    const val REGISTER = "register"
    const val HOME = "home"
    const val NEARBY = "nearby"
    const val MY_CAPTURES = "my_captures"
}
```

`ui/navigation/SkyDexNavHost.kt`:

```kotlin
package com.example.skydex.ui.navigation

import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Scaffold
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import com.example.skydex.ServiceLocator
import com.example.skydex.data.session.Session
import com.example.skydex.ui.auth.LoginScreen
import com.example.skydex.ui.auth.LoginViewModel
import com.example.skydex.ui.auth.RegisterScreen
import com.example.skydex.ui.auth.RegisterViewModel
import com.example.skydex.ui.captures.MyCapturesScreen
import com.example.skydex.ui.captures.MyCapturesViewModel
import com.example.skydex.ui.components.AppBottomBar
import com.example.skydex.ui.home.HomeScreen
import com.example.skydex.ui.home.HomeViewModel

private val BAR_ROUTES = setOf(Routes.HOME, Routes.NEARBY, Routes.MY_CAPTURES)

@Composable
fun SkyDexNavHost(session: Session?) {
    val navController = rememberNavController()
    val backStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = backStackEntry?.destination?.route ?: Routes.LOGIN
    val startDestination = remember { if (session == null) Routes.LOGIN else Routes.HOME }

    Scaffold(
        modifier = Modifier.fillMaxSize(),
        bottomBar = {
            if (currentRoute in BAR_ROUTES) {
                AppBottomBar(currentRoute) { route ->
                    navController.navigate(route) {
                        popUpTo(Routes.HOME) { saveState = true }
                        launchSingleTop = true
                        restoreState = true
                    }
                }
            }
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            // `remember`ed: [session] decides where to START, once. Everything after that is
            // explicit navigation. Letting this expression re-evaluate would repin the graph
            // mid-session and tear the back stack down underneath the user.
            startDestination = startDestination,
            modifier = Modifier.padding(innerPadding)
        ) {
            composable(Routes.LOGIN) {
                val vm: LoginViewModel = viewModel { LoginViewModel(ServiceLocator.authRepository) }
                LoginScreen(
                    viewModel = vm,
                    onLoggedIn = {
                        navController.navigate(Routes.HOME) {
                            popUpTo(Routes.LOGIN) { inclusive = true }
                        }
                    },
                    onNavigateToRegister = { navController.navigate(Routes.REGISTER) }
                )
            }

            composable(Routes.REGISTER) {
                val vm: RegisterViewModel = viewModel { RegisterViewModel(ServiceLocator.authRepository) }
                RegisterScreen(
                    viewModel = vm,
                    onRegistered = { navController.popBackStack() },
                    onNavigateToLogin = { navController.popBackStack() }
                )
            }

            composable(Routes.HOME) {
                val vm: HomeViewModel = viewModel { HomeViewModel(ServiceLocator.captureRepository) }
                HomeScreen(viewModel = vm)
            }

            composable(Routes.NEARBY) {
                val vm: HomeViewModel = viewModel { HomeViewModel(ServiceLocator.captureRepository) }
                HomeScreen(viewModel = vm)
            }

            composable(Routes.MY_CAPTURES) {
                val vm: MyCapturesViewModel = viewModel { MyCapturesViewModel(ServiceLocator.captureRepository) }
                MyCapturesScreen(viewModel = vm)
            }
        }
    }
}
```

`Routes.HOME` and `Routes.NEARBY` render the same screen for now; Task 10 gives `HOME` its own dashboard with the capture button and leaves `NEARBY` as the phenomena list.

- [ ] **Step 10: Shrink MainActivity to a host**

Replace `MainActivity.kt`:

```kotlin
package com.example.skydex

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import com.example.skydex.ui.navigation.SkyDexNavHost
import com.example.skydex.ui.theme.SkyDexTheme

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            SkyDexTheme {
                val snapshot by produceState<SessionSnapshot?>(initialValue = null) {
                    ServiceLocator.sessionStore.session.collect { value = SessionSnapshot(it) }
                }

                // Draw nothing until DataStore has answered. `NavHost` fixes its start
                // destination the first time it is composed, so composing it against a
                // not-yet-read session would strand a logged-in user on the login screen.
                snapshot?.let { SkyDexNavHost(session = it.session) }
            }
        }
    }
}

/**
 * Distinguishes "DataStore has not answered yet" (no snapshot) from "there is no stored session"
 * (a snapshot holding `null`). Collecting the session flow straight into state collapses the two
 * into the same `null`, and the difference is exactly what decides the start destination.
 */
private data class SessionSnapshot(val session: Session?)
```

The extra wrapper is not ceremony. `collectAsState(initial = null)` reports `null` both before DataStore has answered and when there is genuinely no session, so on every cold start the graph would be built with `startDestination = LOGIN` and reaching Home would depend on `NavController.setGraph` tearing down and rebuilding the back stack when the value later changed — undocumented behaviour that discards ViewModels and flashes the login screen at a user who is already logged in.

- [ ] **Step 11: Delete the old page files**

```bash
cd <workspace>/SkyDex---frontend/app/src/main/java/com/example/skydex
git rm ui/theme/pages/HomePage.kt ui/theme/pages/Login.kt ui/theme/pages/Signin.kt \
       ui/theme/pages/NearEvents.kt ui/theme/pages/Registers.kt
```

- [ ] **Step 12: Compile and run all Android tests**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`; `LoginViewModelTest` and `AuthInterceptorTest` green.

- [ ] **Step 13: Verify auto-login on the phone**

With the backend running and the phone connected:

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:installDebug
```

Log in, force-stop the app, reopen it. **Expected: it lands on Home, not on Login.** Rotate the phone on the "Meus Registros" tab — the list must not re-fetch or flash a spinner.

- [ ] **Step 14: Commit**

```bash
git add -A app
git commit -m "refactor: navigation compose, viewmodels and a real package layout

Replaces MainActivity's string-based screen state with a NavHost, moves
screens out of ui/theme/pages into feature packages, and puts every
screen behind a ViewModel over a repository so state survives rotation."
```

---

# Phase 2 — The Capture Loop

This is the phase that makes SkyDex a camera app. Today a "capture" is a title, a description, and a URL the user types in by hand. After this phase it is a photograph taken with the phone camera, stamped with the phone's GPS position and the moment it was taken.

---

### Task 6: Geolocation and capture time on weather events

The backend cannot validate a phenomenon it has no coordinates for, and the feed cannot say where anything happened. This task adds `latitude` and `longitude` to the capture model, and makes `capturedAt` a server-side stamp.

`capturedAt` decides whether a photo earns XP and a rare badge: Task 12 validates the capture against the real weather *at that instant*. If the client names the hour, anyone can look up yesterday's thunderstorm, backdate a submission, and farm the rarest achievements without ever seeing a storm — and `MAX_SKEW` does not stop that, because it only bounds the distance to the nearest forecast slot, and the forecast window is ~72h wide. So the server stamps it with `Instant.now()` at creation and the request DTO carries no such field. The MVP capture flow photographs and submits in one go, so nothing is lost.

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/WeatherEvent.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestFixtures.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/WeatherEventDto.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/captures/MyCapturesScreen.kt` (preview fixtures)
- Modify: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/captures/MyCapturesViewModelTest.kt` (fixtures)

  `WeatherEventResponse` gains two non-nullable fields with no defaults, so every construction of it
  has to be updated — including the `@Preview` fixtures and the test fixtures, which are easy to
  forget because nothing references them until the Kotlin compiler does.

**Interfaces:**
- Consumes: `WeatherEvent`, `CreateWeatherEventRequest`, `WeatherEventResponse` (Task 2); `NotFoundException`, `ForbiddenException` (Task 3).
- Produces:
  - `WeatherEvent` gains `var latitude: Double` and `var longitude: Double` (columns `latitude`, `longitude`, both `nullable = false`), appended after `capturedAt` in the constructor.
  - `CreateWeatherEventRequest` gains `latitude: Double`, `longitude: Double`, appended after `photoUrl`. It deliberately has **no** `capturedAt` — the server stamps it. **Task 12 appends `phenomenon: String`.**
  - `WeatherEventResponse` gains `latitude: Double`, `longitude: Double`, appended after `capturedAt`. **Task 13 appends five more fields.**
  - `persistEvent(...)` gains `latitude: Double = -23.55`, `longitude: Double = -46.63` parameters.
  - Android `CreateWeatherEventRequest` gains `latitude: Double`, `longitude: Double` (no `capturedAt` — the server owns it); `WeatherEventResponse` gains `latitude: Double`, `longitude: Double`.

- [ ] **Step 1: Write the failing tests**

Append to `WeatherEventControllerTest.kt`:

```kotlin
    @Test
    fun `stores the coordinates and stamps the capture time on the server`() {
        val user = persistUser(email = "geo@skydex.com")
        val before = Instant.now()

        val payload = CreateWeatherEventRequest(
            title = "Tempestade",
            description = "Raios sobre o bairro",
            photoUrl = "http://localhost:8080/api/photos/storm.jpg",
            latitude = -30.0346,
            longitude = -51.2177
        )

        val body = mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.latitude").value(-30.0346))
            .andExpect(jsonPath("$.longitude").value(-51.2177))
            .andExpect(jsonPath("$.authorName").value("Test Pilot"))
            .andReturn().response.contentAsString

        // The client never sends a capture time, so the only thing that can be asserted is that
        // the server chose one, and chose it now. This is the whole anti-backdating property:
        // if a `capturedAt` field is ever added back to the request, this test still passes but
        // stops meaning anything — so the request DTO having no such field is what protects it.
        val stamped = Instant.parse(objectMapper.readTree(body).get("capturedAt").asText())
        assert(!stamped.isBefore(before.truncatedTo(ChronoUnit.MILLIS)))
        assert(!stamped.isAfter(Instant.now()))
    }

    @Test
    fun `rejects a latitude outside the valid range`() {
        val user = persistUser(email = "badgeo@skydex.com")

        val payload = CreateWeatherEventRequest(
            title = "Impossible",
            description = "Off the planet",
            photoUrl = "http://localhost:8080/api/photos/x.jpg",
            latitude = 120.0,
            longitude = 0.0
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("latitude: must be between -90 and 90"))
    }

    @Test
    fun `ignores coordinates supplied on update`() {
        val user = persistUser(email = "pinmover@skydex.com")
        val event = persistEvent(owner = user, latitude = -30.0346, longitude = -51.2177)

        // The mirror image of backdating. Task 12 scores a capture against the weather at an
        // instant AND a place, so a movable pin is a movable verdict: create now, look up where a
        // storm is happening at this frozen instant, PUT the coordinates there, collect the badge.
        val payload = CreateWeatherEventRequest(
            title = "Editado",
            description = "So o texto mudou",
            photoUrl = "http://localhost:8080/api/photos/x.jpg",
            latitude = 35.6762,
            longitude = 139.6503
        )

        mockMvc.perform(
            put("/api/events/{id}", event.id!!)
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.title").value("Editado"))
            .andExpect(jsonPath("$.latitude").value(-30.0346))
            .andExpect(jsonPath("$.longitude").value(-51.2177))
    }

    @Test
    fun `accepts a longitude that is only valid on the longitude scale`() {
        val user = persistUser(email = "meridian@skydex.com")

        // 120 deg E is a real place (western Australia) and the ONLY value that catches the bug
        // this pair of tests exists for: latitude's bounds copy-pasted onto longitude. A rejection
        // test cannot catch it — 200 is outside both +-90 and +-180, and the message string is a
        // literal rather than something derived from the annotation, so it reads "-180 and 180"
        // either way. Acceptance is what discriminates.
        val payload = CreateWeatherEventRequest(
            title = "Deserto",
            description = "Ceu limpo a perder de vista",
            photoUrl = "http://localhost:8080/api/photos/x.jpg",
            latitude = -25.0,
            longitude = 120.0
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.longitude").value(120.0))
    }

    @Test
    fun `rejects a longitude outside the valid range`() {
        val user = persistUser(email = "offmap@skydex.com")

        val payload = CreateWeatherEventRequest(
            title = "Impossible",
            description = "Off the planet",
            photoUrl = "http://localhost:8080/api/photos/x.jpg",
            latitude = 0.0,
            longitude = 200.0
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("longitude: must be between -180 and 180"))
    }

    @Test
    fun `rejects a photo url pointing at a foreign host`() {
        val user = persistUser(email = "pixel@skydex.com")

        // This app shows one user's captures to their friends, so photoUrl is not merely the
        // author's own business: every friend who opens the feed fetches whatever it names. An
        // unconstrained string is a tracking pixel any user can plant in everyone else's client.
        val payload = CreateWeatherEventRequest(
            title = "Parece uma foto",
            description = "Mas nao esta no nosso servidor",
            photoUrl = "http://attacker.example/pixel.jpg",
            latitude = 0.0,
            longitude = 0.0
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("photoUrl: Photo URL must be a path returned by POST /api/photos"))
    }

    @Test
    fun `rejects a photo url that escapes the photos path`() {
        val user = persistUser(email = "traversal@skydex.com")

        // Same origin, wrong endpoint. Excluding `/` from the filename is what stops this, and it
        // is the half that a "must start with /api/photos" check would miss.
        val payload = CreateWeatherEventRequest(
            title = "Ainda parece",
            description = "Mas sobe um nivel",
            photoUrl = "/api/photos/../../api/users/me",
            latitude = 0.0,
            longitude = 0.0
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isBadRequest)
    }

    @Test
    fun `ignores a capture time supplied by the client`() {
        val user = persistUser(email = "liar@skydex.com")

        // Hand-built JSON, because the DTO has no `capturedAt` field to set. A gamified app pays
        // XP and rare badges for matching real weather, so a client that could name the hour
        // could look up yesterday's thunderstorm and farm "Pé de Raio" without seeing a storm.
        // Jackson ignores unknown properties by default; this pins that the value is discarded
        // rather than honoured.
        val backdated = """
            {"title":"Ontem","description":"Faz de conta","photoUrl":"http://localhost:8080/api/photos/x.jpg",
             "latitude":0.0,"longitude":0.0,"capturedAt":"2020-01-01T00:00:00Z"}
        """.trimIndent()

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(backdated)
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.capturedAt").value(org.hamcrest.Matchers.not("2020-01-01T00:00:00Z")))
    }
```

Add the imports `java.time.Instant` and `java.time.temporal.ChronoUnit`.

- [ ] **Step 2: Run and watch them fail**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.WeatherEventControllerTest"
```

Expected: compilation failure — `No value passed for parameter 'latitude'`.

- [ ] **Step 3: Add the columns to the entity**

In `models/WeatherEvent.kt`, append two properties after `capturedAt` and before `userId`:

```kotlin
    @Column(nullable = false)
    var latitude: Double = 0.0,

    @Column(nullable = false)
    var longitude: Double = 0.0,
```

Coordinates are stored as plain doubles, not a PostGIS `geography` column. Nothing in the MVP does a radius query; when one is needed, that is the moment to introduce the spatial type (`hibernate-spatial` is already on the classpath).

- [ ] **Step 4: Extend the request and response DTOs**

Replace `dto/WeatherEventDtos.kt`:

```kotlin
package com.skydex.api.dto

import com.skydex.api.models.User
import com.skydex.api.models.WeatherEvent
import jakarta.validation.constraints.DecimalMax
import jakarta.validation.constraints.DecimalMin
import jakarta.validation.constraints.NotBlank
import java.time.Instant
import java.util.UUID

data class CreateWeatherEventRequest(
    @field:NotBlank(message = "Title is required")
    val title: String,

    @field:NotBlank(message = "Description is required")
    val description: String,

    // Constrained to a path this server itself issued, not merely non-blank. `photoUrl` is
    // client-supplied, persisted verbatim, and then rendered by every friend who opens the feed —
    // so an unconstrained string lets any user plant `http://attacker.example/pixel.jpg` and
    // collect the IP and view time of everyone who scrolls past it. Requiring a relative
    // /api/photos/ path makes a foreign host unrepresentable rather than merely discouraged, and
    // excluding `/` from the filename blocks traversal into other endpoints on our own origin.
    @field:Pattern(
        regexp = "^/api/photos/[A-Za-z0-9._-]+\\.(jpg|png)$",
        message = "Photo URL must be a path returned by POST /api/photos"
    )
    val photoUrl: String,

    @field:DecimalMin(value = "-90.0", message = "must be between -90 and 90")
    @field:DecimalMax(value = "90.0", message = "must be between -90 and 90")
    val latitude: Double,

    @field:DecimalMin(value = "-180.0", message = "must be between -180 and 180")
    @field:DecimalMax(value = "180.0", message = "must be between -180 and 180")
    val longitude: Double
)

data class WeatherEventResponse(
    val id: UUID,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: Instant,
    val latitude: Double,
    val longitude: Double,
    val userId: UUID,
    val authorName: String
) {
    companion object {
        fun from(event: WeatherEvent, author: User) = WeatherEventResponse(
            id = event.id!!,
            title = event.title,
            description = event.description,
            photoUrl = event.photoUrl,
            capturedAt = event.capturedAt,
            latitude = event.latitude,
            longitude = event.longitude,
            userId = event.userId,
            authorName = author.name
        )
    }
}
```

Note there is no `@PastOrPresent` anywhere here, and no clock-skew problem to tune: the capture time is never parsed from a request, so a phone with a fast clock cannot trip a validator that does not exist. Task 12's `MAX_SKEW` still applies, but it bounds the distance from the server's own stamp to the nearest forecast slot — a different question entirely.

- [ ] **Step 5: Pass the new fields through the controller**

In `WeatherEventController.create`, replace the `WeatherEvent(...)` construction:

```kotlin
        val saved = events.save(
            WeatherEvent(
                id = null,
                title = request.title,
                description = request.description,
                photoUrl = request.photoUrl,
                // Server-stamped, never client-supplied: see this task's opening note.
                capturedAt = Instant.now(),
                latitude = request.latitude,
                longitude = request.longitude,
                userId = currentUser.id!!
            )
        )
```

`update` copies neither the coordinates nor the capture time — it edits the description of a capture, not the capture:

```kotlin
        event.title = request.title
        event.description = request.description
        event.photoUrl = request.photoUrl
        // Neither capturedAt NOR the coordinates are updated here, and for one reason: Task 12
        // scores a capture against the real weather at an instant AND a place. Freezing the time
        // while leaving the pin editable just moves the cheat one axis over — create a capture
        // now, find where a storm is happening at that frozen instant, PUT the coordinates there,
        // collect the rare badge. Nothing legitimate is lost: the coordinates are client-supplied
        // at creation because the server cannot see the phone, and letting them move afterwards
        // only buys a second, better-informed attempt.
        // `CreateWeatherEventRequest` still carries latitude/longitude, and this handler silently
        // ignores them — the same shape as capturedAt, and pinned by the test below.
```

- [ ] **Step 6: Extend the fixture**

In `support/TestFixtures.kt`, add the two parameters to `persistEvent` and pass them through:

```kotlin
fun IntegrationTestBase.persistEvent(
    owner: User,
    title: String = "Aurora",
    description: String = "Green lights in the night sky",
    photoUrl: String = "http://localhost:8080/api/photos/test.jpg",
    capturedAt: Instant = Instant.now(),
    latitude: Double = -23.55,
    longitude: Double = -46.63
): WeatherEvent = weatherEventRepository.save(
    WeatherEvent(
        id = null,
        title = title,
        description = description,
        photoUrl = photoUrl,
        capturedAt = capturedAt,
        latitude = latitude,
        longitude = longitude,
        userId = owner.id!!
    )
)
```

Update every existing `CreateWeatherEventRequest(...)` in the test sources to pass `latitude` and `longitude`. Do **not** add `capturedAt` — the request DTO has no such field, by design.

- [ ] **Step 7: Run the tests and confirm green**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, including the three new tests.

- [ ] **Step 8: Mirror the change in the Android wire DTOs**

In `data/remote/dto/WeatherEventDto.kt`:

```kotlin
data class CreateWeatherEventRequest(
    val title: String,
    val description: String,
    val photoUrl: String,
    val latitude: Double,
    val longitude: Double
    // No `capturedAt`. The server stamps the capture time; a client that could name the hour
    // could backdate to a past storm and farm the rare badges. `WeatherEventResponse` below
    // still reads it back as a String, since Gson has no Instant adapter.
)

data class WeatherEventResponse(
    val id: String,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: String,
    val latitude: Double,
    val longitude: Double,
    val userId: String,
    val authorName: String
)
```

- [ ] **Step 9: Compile the Android app**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin
```

Expected: failures at every `CreateWeatherEventRequest(...)` call site. Task 10 supplies real coordinates; until then, keep the app compiling by passing `latitude = 0.0, longitude = 0.0` at the one remaining call site and marking it `// TODO(task-10): replace with the device's real position`.

- [ ] **Step 10: Commit**

```bash
cd <workspace>/SkyDex-backend
git add -A src && git commit -m "feat: record where and when a capture was taken"
cd ../SkyDex---frontend
git add -A app && git commit -m "feat: send coordinates and capture time with a new capture"
```

---

### Task 7: Photo upload and serving

`photoUrl` is currently whatever string the user typed. This task gives the backend somewhere to put a real JPEG and a URL that Coil can load.

**Files:**
- Modify: `SkyDex-backend/src/main/resources/application.properties`
- Modify: `SkyDex-backend/src/test/resources/application-test.properties`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/PhotoStorageService.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/PhotoController.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/PhotoDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/config/WebConfig.kt`
- Modify: `SkyDex-backend/.env`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/SkyDexApi.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/PhotoDto.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/CaptureRepository.kt`

**Interfaces:**
- Consumes: `ErrorResponse` (Task 2); `GlobalExceptionHandler`, `SecurityConfig`'s public `GET /api/photos/**` rule (Task 3).
- Produces:
  - `services.PhotoStorageService(storageDirectory: String)` with `fun store(bytes: ByteArray, originalFilename: String?, contentType: String?): String` returning a **relative** `/api/photos/<uuid>.<ext>`, and `fun directory(): Path`. It deliberately does **not** know the public base URL — `WeatherEventResponse.from` takes it as a required third parameter and composes the absolute URL when serialising, so an absolute host never reaches the persistence path.
  - `dto.PhotoUploadResponse(photoUrl: String)`.
  - `POST /api/photos` — multipart, part name `file`, authenticated, returns 201 with `PhotoUploadResponse`. Accepts only `image/jpeg` and `image/png`, max 8 MB.
  - `GET /api/photos/{filename}` — public, served by `WebConfig`'s resource handler.
  - Android `SkyDexApi.uploadPhoto(file: MultipartBody.Part): PhotoUploadResponse` and `CaptureRepository.uploadPhoto(file: File): Result<String>` (returns the URL).
  - New env vars: `PHOTO_STORAGE_DIR`, `PUBLIC_BASE_URL`.

- [ ] **Step 1: Write the failing upload tests**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.springframework.http.MediaType
import org.springframework.mock.web.MockMultipartFile
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

class PhotoControllerTest : IntegrationTestBase() {

    /** Smallest valid JPEG: SOI marker, EOI marker. Enough to prove bytes round-trip. */
    private val jpegBytes = byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte(), 0xD9.toByte())

    @Test
    fun `stores an uploaded jpeg and returns a fetchable url`() {
        val user = persistUser(email = "photographer@skydex.com")
        val part = MockMultipartFile("file", "storm.jpg", MediaType.IMAGE_JPEG_VALUE, jpegBytes)

        val body = mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.photoUrl").exists())
            .andReturn().response.contentAsString

        val photoUrl = objectMapper.readTree(body).get("photoUrl").asText()
        assertTrue(photoUrl.contains("/api/photos/"), "expected a /api/photos/ URL, got $photoUrl")
        assertTrue(photoUrl.endsWith(".jpg"), "expected the extension to be preserved, got $photoUrl")

        // The stored file is reachable without authentication so Coil can render it.
        val filename = photoUrl.substringAfterLast('/')
        mockMvc.perform(get("/api/photos/{filename}", filename))
            .andExpect(status().isOk)
    }

    @Test
    fun `rejects a non-image upload`() {
        val user = persistUser(email = "spammer@skydex.com")
        val part = MockMultipartFile("file", "payload.sh", "application/x-sh", "rm -rf /".toByteArray())

        mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("Only JPEG and PNG images are accepted"))
    }

    @Test
    fun `rejects a non-image disguised as a jpeg`() {
        val user = persistUser(email = "liar@skydex.com")

        // The test above only rejects an attacker who is honest about what they are sending.
        // Content-Type is a client-written multipart header, so the interesting case is the one
        // that lies: script bytes under an image label. If this passes, the file is stored and
        // served under a .jpg name.
        val part = MockMultipartFile(
            "file", "storm.jpg", MediaType.IMAGE_JPEG_VALUE, "<html><script>alert(1)</script>".toByteArray()
        )

        mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("File is not a JPEG image"))
    }

    @Test
    fun `rejects an empty upload`() {
        val user = persistUser(email = "empty@skydex.com")
        val part = MockMultipartFile("file", "empty.jpg", MediaType.IMAGE_JPEG_VALUE, ByteArray(0))

        mockMvc.perform(
            multipart("/api/photos").file(part).header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("Photo is empty"))
    }

    @Test
    fun `refuses an anonymous upload`() {
        val part = MockMultipartFile("file", "storm.jpg", MediaType.IMAGE_JPEG_VALUE, jpegBytes)

        mockMvc.perform(multipart("/api/photos").file(part))
            .andExpect(status().isUnauthorized)
    }
}
```

- [ ] **Step 2: Run and watch them fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.PhotoControllerTest"
```

Expected: all five fail with 404 (no such endpoint), except `refuses an anonymous upload` which may already pass — that is fine, it is a regression guard.

- [ ] **Step 3: Add the storage configuration**

Append to `SkyDex-backend/src/main/resources/application.properties`:

```properties
# Where uploaded capture photos are written, and the base URL clients use to read them back.
skydex.photos.directory=${PHOTO_STORAGE_DIR:./data/photos}
skydex.photos.public-base-url=${PUBLIC_BASE_URL:http://localhost:8080}

spring.servlet.multipart.max-file-size=8MB
spring.servlet.multipart.max-request-size=8MB
```

Append to `SkyDex-backend/.env` (untracked; tell the user to set `PUBLIC_BASE_URL` to their machine's LAN address so the phone can load photos):

```
PHOTO_STORAGE_DIR=./data/photos
PUBLIC_BASE_URL=http://<lan-address>:8080
```

Append to `SkyDex-backend/.gitignore`:

```
data/photos/
```

Append to `SkyDex-backend/src/test/resources/application-test.properties`:

```properties
skydex.photos.directory=${java.io.tmpdir}/skydex-test-photos
skydex.photos.public-base-url=http://localhost:8080
```

- [ ] **Step 4: Write the storage service**

Create `services/PhotoStorageService.kt`:

```kotlin
package com.skydex.api.services

import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.Paths
import java.util.Locale
import java.util.UUID

@Service
class PhotoStorageService(
    @Value("\${skydex.photos.directory}") private val storageDirectory: String
) {

    private val allowedContentTypes = setOf("image/jpeg", "image/png")

    /** JPEG files begin FF D8 FF; PNG files begin 89 50 4E 47 0D 0A 1A 0A. */
    private val JPEG_MAGIC = byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte())
    private val PNG_MAGIC = byteArrayOf(
        0x89.toByte(), 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A
    )

    private fun ByteArray.startsWith(prefix: ByteArray): Boolean =
        size >= prefix.size && prefix.indices.all { this[it] == prefix[it] }

    private val root: Path by lazy {
        Paths.get(storageDirectory).toAbsolutePath().normalize().also { Files.createDirectories(it) }
    }

    /**
     * Writes the bytes under a freshly generated name and returns the URL clients should use.
     * The original filename is never reused — only its extension, and only after validation —
     * so a caller cannot influence the path that gets written.
     */
    fun store(bytes: ByteArray, originalFilename: String?, contentType: String?): String {
        if (bytes.isEmpty()) throw BadUploadException("Photo is empty")
        if (contentType?.lowercase(Locale.ROOT) !in allowedContentTypes) {
            throw BadUploadException("Only JPEG and PNG images are accepted")
        }

        // The Content-Type above is a multipart header — the client writes it, so on its own it
        // proves nothing. Without this check anyone can POST a shell script, an HTML page or a zip
        // labelled `image/jpeg` and have it stored and served under a .jpg name. Verifying the
        // leading bytes is what turns the claim into something checked.
        val declared = contentType!!.lowercase(Locale.ROOT)
        val extension = when (declared) {
            "image/png" -> {
                if (!bytes.startsWith(PNG_MAGIC)) throw BadUploadException("File is not a PNG image")
                "png"
            }
            else -> {
                if (!bytes.startsWith(JPEG_MAGIC)) throw BadUploadException("File is not a JPEG image")
                "jpg"
            }
        }
        val filename = "${UUID.randomUUID()}.$extension"
        Files.write(root.resolve(filename), bytes)

        // RELATIVE, deliberately. This string is handed back by the client and persisted in
        // weather_events.photo_url, and a row is immutable once written — so baking an absolute
        // host into it means a new DHCP lease, a different laptop, or a real deployment leaves
        // every historical capture pointing at an address that no longer serves those bytes,
        // with the JPEGs sitting intact and unreachable on disk. Config can be re-pointed;
        // written rows cannot. The base URL is a display concern, so it is applied at the
        // read path instead: `WeatherEventResponse.from` takes a baseUrl parameter and composes
        // the absolute URL when serialising. See the note under that class.
        return "/api/photos/$filename"
    }

    // No `resolve(filename)` helper here on purpose. An earlier draft had one that normalised and
    // checked containment, but nothing called it: the controller uses `store`, and `WebConfig`
    // serves through Spring's own ResourceHttpRequestHandler, whose PathResourceResolver already
    // performs that containment check — on top of `isInvalidPath` rejecting `..` and
    // StrictHttpFirewall rejecting encoded slashes before routing. A tested-but-unwired method that
    // reads like a security guard is worse than no method: the next reader trusts it. Backlog #13
    // (deleting a photo when its capture is deleted) is the caller that will justify reintroducing
    // it, with a status code that is not 409.

    fun directory(): Path = root
}

class BadUploadException(message: String) : RuntimeException(message)
```

Then add the response-boundary composer. Put it next to `WeatherEventResponse` in `dto/WeatherEventDtos.kt`:

```kotlin
fun from(event: WeatherEvent, author: User, baseUrl: String) = WeatherEventResponse(
    // ... existing field mapping ...
    photoUrl = if (event.photoUrl.startsWith("/")) baseUrl.trimEnd('/') + event.photoUrl
               else event.photoUrl,
    // ...
)
```

`baseUrl` is a **required third parameter**, not an optional post-processing step.

**One sharp edge this creates, and it needs saying before Task 10 or 13 hits it.** The API is now asymmetric on `photoUrl`: every event response returns it **absolute**, and `PUT /api/events/{id}` rejects that exact value, because the request DTO's `@Pattern` accepts only a relative `/api/photos/...` path. So a read-modify-write edit flow — fetch a capture, change its title, send it back unchanged otherwise — gets a 400 on the field it did not touch. No client does this today (`SkyDexApi` has no update method), which is the only reason it is not already a bug. Whoever builds an edit screen must strip the host before re-submitting, or the request DTO needs a separate, looser shape for updates.

An earlier draft of this plan applied the composition at the controller boundary instead, via a `withAbsolutePhotoUrl` extension, reasoning that it avoided threading a base URL through Tasks 12, 13, 16 and 18. That reasoning was wrong, and the Task 7 implementer said so rather than implementing it quietly: `WeatherEventResponse` has no access to configuration, so **every class that returns one must inject `publicBaseUrl` under either design**. The churn is identical. The only thing the two designs differ on is what happens when someone forgets — a silently relative URL, or a compile error.

There is already a case in this plan where the boundary does not exist: Task 16's `FeedService.feed()` calls `from` itself and returns finished responses, so its controller has no mapping step to hang an extension on. Making `baseUrl` required means that case cannot be missed.

The `startsWith("/")` guard keeps it idempotent and leaves an already-absolute value alone.

Every caller injects `@Value("\${skydex.photos.public-base-url}") private val publicBaseUrl: String` and passes it through — `WeatherEventController` (four sites), and later `FeedService`, plus whatever Tasks 12, 13 and 18 add.

`PhotoUploadResponse.photoUrl` stays **relative** — the client persists exactly what it was given, and only the read path composes.

Register the new exception in `controllers/GlobalExceptionHandler.kt`:

```kotlin
    @ExceptionHandler(com.skydex.api.services.BadUploadException::class)
    fun handleBadUpload(e: com.skydex.api.services.BadUploadException): ResponseEntity<ErrorResponse> =
        ResponseEntity.badRequest().body(ErrorResponse(e.message ?: "Invalid upload"))
```

- [ ] **Step 5: Write the upload endpoint and the response DTO**

Create `dto/PhotoDtos.kt`:

```kotlin
package com.skydex.api.dto

data class PhotoUploadResponse(val photoUrl: String)
```

Create `controllers/PhotoController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.PhotoUploadResponse
import com.skydex.api.models.User
import com.skydex.api.services.PhotoStorageService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/api/photos")
class PhotoController(private val photos: PhotoStorageService) {

    @PostMapping
    fun upload(
        @AuthenticationPrincipal currentUser: User,
        @RequestParam("file") file: MultipartFile
    ): ResponseEntity<PhotoUploadResponse> {
        val url = photos.store(file.bytes, file.originalFilename, file.contentType)
        return ResponseEntity.status(HttpStatus.CREATED).body(PhotoUploadResponse(url))
    }
}
```

`currentUser` is unused in the body but keeps the endpoint authenticated and makes the ownership requirement visible; do not drop the parameter.

- [ ] **Step 6: Serve the stored files**

Create `config/WebConfig.kt`:

```kotlin
package com.skydex.api.config

import com.skydex.api.services.PhotoStorageService
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

/**
 * Serves uploaded capture photos straight off disk. `GET /api/photos/{filename}` is permitted
 * anonymously in SecurityConfig so the Android image loader can fetch them.
 *
 * (The handler pattern is spelled out in code below rather than here: Kotlin block comments nest,
 * so a literal `/`+`**` inside a KDoc opens a comment that is never closed.)
 */
@Configuration
class WebConfig(private val photos: PhotoStorageService) : WebMvcConfigurer {

    override fun addResourceHandlers(registry: ResourceHandlerRegistry) {
        registry
            .addResourceHandler("/api/photos/**")
            .addResourceLocations(photos.directory().toUri().toString())
    }
}
```

Then make `X-Content-Type-Options: nosniff` explicit in `config/SecurityConfig.kt`, inside the existing `http { }` chain:

```kotlin
            .headers { headers -> headers.contentTypeOptions { } }
```

**Be honest about what this line does: nothing, today.** Spring Security already sends that header by default, and removing this line leaves the suite green — verified by deleting it and re-running. It is kept, and paired with an assertion in `PhotoControllerTest` that the served photo carries the header, so the behaviour is *pinned* rather than inherited. The distinction matters because these files are attacker-supplied and served from the API's own origin: if a later task reconfigures `headers { }` and silently drops the default, a test should fail rather than a browser start sniffing. Do not read the byte check as the whole defence, either. It stops a plain script or HTML page wearing a `.jpg` name, which is the common case. It does **not** stop a polyglot — a file that is a structurally valid JPEG *and* something else, since a JPEG comment segment can carry an arbitrary payload after a valid header. For that case `nosniff` is the operative control and the byte check only makes the attack expensive. The two are a pair; neither is merely a net under the other.

- [ ] **Step 7: Run the tests and confirm green**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.PhotoControllerTest"
```

Expected: 5 tests PASS.

If `stores an uploaded jpeg…` fails on the `GET` with a 404, the resource handler and the `POST` mapping are colliding on `/api/photos`. Confirm the `POST` handler is matched first (it is, because `@PostMapping` on a controller beats a resource handler for the same path) and that the file really landed in `photos.directory()`.

- [ ] **Step 8: Run the whole backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS.

- [ ] **Step 9: Add the Android upload plumbing**

Create `data/remote/dto/PhotoDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class PhotoUploadResponse(val photoUrl: String)
```

Add to `data/remote/SkyDexApi.kt`:

```kotlin
    @Multipart
    @POST("api/photos")
    suspend fun uploadPhoto(@Part file: MultipartBody.Part): PhotoUploadResponse
```

with imports `okhttp3.MultipartBody`, `retrofit2.http.Multipart`, `retrofit2.http.Part`, and `com.example.skydex.data.remote.dto.PhotoUploadResponse`.

Add to `data/repository/CaptureRepository.kt`:

```kotlin
    /** Uploads a local JPEG and returns the public URL the backend assigned to it. */
    suspend fun uploadPhoto(file: File): Result<String> = resultOf {
        val body = file.asRequestBody("image/jpeg".toMediaType())
        val part = MultipartBody.Part.createFormData("file", file.name, body)
        api.uploadPhoto(part).photoUrl
    }
```

with imports `java.io.File`, `okhttp3.MediaType.Companion.toMediaType`, `okhttp3.MultipartBody`, `okhttp3.RequestBody.Companion.asRequestBody`.

- [ ] **Step 10: Compile the Android app**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 11: Verify an end-to-end upload with curl**

```bash
cd <workspace>/SkyDex-backend
docker start skydex-db && sleep 3
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun &
sleep 25

curl -s -X POST http://localhost:8080/auth/register -H 'Content-Type: application/json' \
  -d '{"name":"Uploader","email":"upload@skydex.com","password":"super-safe-password"}'

TOKEN=$(curl -s -X POST http://localhost:8080/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"upload@skydex.com","password":"super-safe-password"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

printf '\xff\xd8\xff\xd9' > /tmp/tiny.jpg
URL=$(curl -s -X POST http://localhost:8080/api/photos -H "Authorization: Bearer $TOKEN" \
  -F 'file=@/tmp/tiny.jpg;type=image/jpeg' | sed -E 's/.*"photoUrl":"([^"]+)".*/\1/')
echo "stored at: $URL"
curl -s -o /dev/null -w 'fetch status: %{http_code}\n' "$URL"
```

Expected: `stored at: http://…/api/photos/<uuid>.jpg` and `fetch status: 200`. Stop the server afterwards.

- [ ] **Step 12: Commit**

```bash
cd <workspace>/SkyDex-backend
git add -A src .gitignore && git commit -m "feat: upload and serve capture photos from local storage"
cd ../SkyDex---frontend
git add -A app && git commit -m "feat: add multipart photo upload to the capture repository"
```

---

### Task 8: Take a photo with the device camera

**Files:**
- Modify: `SkyDex---frontend/app/src/main/AndroidManifest.xml`
- Create: `SkyDex---frontend/app/src/main/res/xml/file_paths.xml`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/util/PhotoCaptureFiles.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/util/PhotoCaptureFilesTest.kt`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `util.PhotoCaptureFiles` — object with `fun newCaptureFile(context: Context): File` (creates `<cacheDir>/captures/<uuid>.jpg`, parent directories included) and `fun uriFor(context: Context, file: File): Uri` (wraps it in the `FileProvider` authority `${applicationId}.fileprovider`).
  - A `FileProvider` declared in the manifest under authority `com.example.skydex.fileprovider`.
  - `<queries>` for `android.media.action.IMAGE_CAPTURE` so `ActivityResultContracts.TakePicture` can resolve the system camera on API 30+.

This task deliberately uses the **system camera app** via `ActivityResultContracts.TakePicture` rather than an in-app CameraX viewfinder. It needs no `CAMERA` permission (the system app owns the hardware), no preview surface, and no lifecycle binding. An in-app viewfinder is a post-MVP polish item.

- [ ] **Step 1: Write the failing test**

Create `app/src/test/java/com/example/skydex/util/PhotoCaptureFilesTest.kt`:

```kotlin
package com.example.skydex.util

import org.junit.Assert.assertEquals
import org.junit.Assert.assertNotEquals
import org.junit.Assert.assertTrue
import org.junit.Rule
import org.junit.Test
import org.junit.rules.TemporaryFolder

class PhotoCaptureFilesTest {

    @get:Rule
    val tempFolder = TemporaryFolder()

    @Test
    fun `creates the captures directory and a jpg file inside it`() {
        val file = PhotoCaptureFiles.newCaptureFileIn(tempFolder.root)

        assertTrue("captures directory should exist", file.parentFile!!.isDirectory)
        assertEquals("captures", file.parentFile!!.name)
        assertTrue("expected a .jpg name, got ${file.name}", file.name.endsWith(".jpg"))
    }

    @Test
    fun `never returns the same name twice`() {
        val first = PhotoCaptureFiles.newCaptureFileIn(tempFolder.root)
        val second = PhotoCaptureFiles.newCaptureFileIn(tempFolder.root)

        assertNotEquals(first.name, second.name)
    }
}
```

The test targets `newCaptureFileIn(baseDir: File)`, the pure part; `newCaptureFile(context)` is the one-line Android wrapper around it. Splitting it this way is what makes the file-naming logic testable on the JVM without Robolectric.

- [ ] **Step 2: Run it and watch it fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.util.PhotoCaptureFilesTest"
```

Expected: `Unresolved reference: PhotoCaptureFiles`.

- [ ] **Step 3: Write the helper**

Create `app/src/main/java/com/example/skydex/util/PhotoCaptureFiles.kt`:

```kotlin
package com.example.skydex.util

import android.content.Context
import android.net.Uri
import androidx.core.content.FileProvider
import java.io.File
import java.util.UUID

object PhotoCaptureFiles {

    /** Pure part: allocates a unique JPEG path under `<baseDir>/captures/`. */
    fun newCaptureFileIn(baseDir: File): File {
        val directory = File(baseDir, "captures").apply { mkdirs() }
        return File(directory, "${UUID.randomUUID()}.jpg")
    }

    fun newCaptureFile(context: Context): File = newCaptureFileIn(context.cacheDir)

    fun uriFor(context: Context, file: File): Uri =
        FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
}
```

`androidx.core.content.FileProvider` comes from `androidx.core:core-ktx`, already a dependency.

- [ ] **Step 4: Run the test and confirm it passes**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.util.PhotoCaptureFilesTest"
```

Expected: 2 tests PASS.

- [ ] **Step 5: Declare the FileProvider and the camera query**

Create `app/src/main/res/xml/file_paths.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="captures" path="captures/" />
</paths>
```

In `app/src/main/AndroidManifest.xml`, add a `<queries>` block immediately after `<uses-permission android:name="android.permission.INTERNET" />`:

```xml
    <queries>
        <intent>
            <action android:name="android.media.action.IMAGE_CAPTURE" />
        </intent>
    </queries>
```

and add this `<provider>` inside `<application>`, after the `<activity>` element:

```xml
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
```

Do **not** add `<uses-permission android:name="android.permission.CAMERA" />`. Declaring it would make `ACTION_IMAGE_CAPTURE` require a runtime grant that the app does not otherwise need.

- [ ] **Step 6: Compile and run the Android tests**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 7: Commit**

```bash
git add -A app
git commit -m "feat: file provider plumbing for system-camera photo capture"
```

---

### Task 9: Read the device's real position

`NearEvents` hardcodes São Paulo. This task gets the phone's actual coordinates.

**Files:**
- Modify: `SkyDex---frontend/gradle/libs.versions.toml`
- Modify: `SkyDex---frontend/app/build.gradle.kts`
- Modify: `SkyDex---frontend/app/src/main/AndroidManifest.xml`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/util/DeviceLocation.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/util/CoordinatesTest.kt`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `util.Coordinates(latitude: Double, longitude: Double)` — a data class, plus `fun Coordinates.isPlausible(): Boolean` (latitude in [-90, 90], longitude in [-180, 180], and not exactly (0.0, 0.0), which is what a failed fix looks like).
  - `util.DeviceLocation(context: Context)` with `suspend fun current(): Coordinates?` — returns null when permission is missing or no fix is available.
  - `util.LOCATION_PERMISSIONS: Array<String>` — `ACCESS_FINE_LOCATION` and `ACCESS_COARSE_LOCATION`.
  - Manifest gains both location permissions.

- [ ] **Step 1: Write the failing test for the pure part**

Create `app/src/test/java/com/example/skydex/util/CoordinatesTest.kt`:

```kotlin
package com.example.skydex.util

import org.junit.Assert.assertFalse
import org.junit.Assert.assertTrue
import org.junit.Test

class CoordinatesTest {

    @Test
    fun `accepts a real position`() {
        assertTrue(Coordinates(-30.0346, -51.2177).isPlausible())
    }

    @Test
    fun `rejects null island, which is what a failed fix looks like`() {
        assertFalse(Coordinates(0.0, 0.0).isPlausible())
    }

    @Test
    fun `rejects out-of-range values`() {
        assertFalse(Coordinates(91.0, 0.0).isPlausible())
        assertFalse(Coordinates(0.0, 181.0).isPlausible())
    }
}
```

- [ ] **Step 2: Run it and watch it fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.util.CoordinatesTest"
```

Expected: `Unresolved reference: Coordinates`.

- [ ] **Step 3: Add the Play Services Location dependency**

In `gradle/libs.versions.toml`, under `[versions]`:

```toml
playServicesLocation = "21.3.0"
```

under `[libraries]`:

```toml
play-services-location = { group = "com.google.android.gms", name = "play-services-location", version.ref = "playServicesLocation" }
```

In `app/build.gradle.kts`, inside `dependencies { }`:

```kotlin
    implementation(libs.play.services.location)
```

- [ ] **Step 4: Declare the permissions**

In `app/src/main/AndroidManifest.xml`, after the INTERNET permission:

```xml
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

- [ ] **Step 5: Write the location helper**

Create `app/src/main/java/com/example/skydex/util/DeviceLocation.kt`:

```kotlin
package com.example.skydex.util

import android.Manifest
import android.annotation.SuppressLint
import android.content.Context
import android.content.pm.PackageManager
import androidx.core.content.ContextCompat
import com.google.android.gms.location.CurrentLocationRequest
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume

data class Coordinates(val latitude: Double, val longitude: Double)

/** A fix at exactly (0, 0) is what the fused provider reports when it has nothing real. */
fun Coordinates.isPlausible(): Boolean =
    latitude in -90.0..90.0 &&
        longitude in -180.0..180.0 &&
        !(latitude == 0.0 && longitude == 0.0)

val LOCATION_PERMISSIONS = arrayOf(
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.ACCESS_COARSE_LOCATION
)

class DeviceLocation(private val context: Context) {

    fun hasPermission(): Boolean = LOCATION_PERMISSIONS.any {
        ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
    }

    /**
     * One-shot position request. Returns null when the permission is missing or the provider
     * cannot produce a fix — callers must handle that rather than substituting a default.
     */
    @SuppressLint("MissingPermission") // guarded by hasPermission() above
    suspend fun current(): Coordinates? {
        if (!hasPermission()) return null

        val client = LocationServices.getFusedLocationProviderClient(context)
        val request = CurrentLocationRequest.Builder()
            .setPriority(Priority.PRIORITY_BALANCED_POWER_ACCURACY)
            .setMaxUpdateAgeMillis(60_000)
            .setDurationMillis(15_000)
            .build()

        return suspendCancellableCoroutine { continuation ->
            client.getCurrentLocation(request, null)
                .addOnSuccessListener { location ->
                    val coordinates = location?.let { Coordinates(it.latitude, it.longitude) }
                    continuation.resume(coordinates?.takeIf { it.isPlausible() })
                }
                .addOnFailureListener { continuation.resume(null) }
        }
    }
}
```

Register it in `ServiceLocator`:

```kotlin
    val deviceLocation: DeviceLocation by lazy { DeviceLocation(appContext) }
```

with the import `com.example.skydex.util.DeviceLocation`.

- [ ] **Step 6: Run the test and confirm it passes**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.util.CoordinatesTest"
```

Expected: 3 tests PASS.

- [ ] **Step 7: Compile the whole app**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin
```

Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 8: Commit**

```bash
git add -A app
git commit -m "feat: read the device position via fused location provider"
```

---

### Task 10: The capture screen — camera, GPS, upload, submit

This task assembles Tasks 6 through 9 into the screen that is the reason the app exists.

**Files:**
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureGateway.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureScreen.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/CaptureRepository.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/Routes.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/SkyDexNavHost.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/home/HomeScreen.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/home/HomeViewModel.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`

**Interfaces:**
- Consumes: `CaptureRepository` incl. `uploadPhoto` (Tasks 5, 7); `Coordinates`, `DeviceLocation` (Task 9); `PhotoCaptureFiles` (Task 8); `CreateWeatherEventRequest` with coordinates (Task 6); `UiState` (Task 5).
- Produces:
  - `ui.capture.CaptureGateway` — the two-method slice of `CaptureRepository` this screen needs
  - `ui.capture.CaptureUiState(title: String, description: String, photoFile: File?, coordinates: Coordinates?, locating: Boolean, submitting: Boolean, saved: Boolean, errorMessage: String?)`
  - `ui.capture.CaptureViewModel(captures: CaptureRepository, locationProvider: suspend () -> Coordinates?)` with `onTitleChanged`, `onDescriptionChanged`, `onPhotoTaken(file: File)`, `refreshLocation()`, `submit()`
  - `Routes.CAPTURE = "capture"`
  - `HomeViewModel` gains `fun loadForCurrentPosition()` and its state becomes `UiState<HomeData>` where `data class HomeData(coordinates: Coordinates?, phenomena: List<NearbyPhenomenonResponse>)`.

Injecting the location as a `suspend () -> Coordinates?` lambda rather than a `DeviceLocation` keeps the ViewModel testable on the JVM — the real wiring passes `ServiceLocator.deviceLocation::current`.

- [ ] **Step 1: Write the failing ViewModel tests**

Create `app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`:

```kotlin
package com.example.skydex.ui.capture

import com.example.skydex.data.remote.dto.CreateWeatherEventRequest
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.util.Coordinates
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNotNull
import org.junit.Assert.assertNull
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Rule
import org.junit.Test
import org.junit.rules.TemporaryFolder
import java.io.File
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class CaptureViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @get:Rule
    val tempFolder = TemporaryFolder()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    private fun jpeg(): File = tempFolder.newFile("photo.jpg").apply { writeBytes(byteArrayOf(1, 2, 3)) }

    @Test
    fun `a complete capture uploads the photo then creates the event`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios sobre o bairro")
        viewModel.onPhotoTaken(jpeg())
        viewModel.submit()
        advanceUntilIdle()

        assertTrue(viewModel.state.value.saved)
        assertNull(viewModel.state.value.errorMessage)
        assertEquals(1, gateway.uploadedFiles.size)

        val sent = gateway.createdRequests.single()
        assertEquals("Tempestade", sent.title)
        // Relative, and it must be exactly what uploadPhoto returned. Task 7 constrains this
        // field server-side to `^/api/photos/[A-Za-z0-9._-]+\\.(jpg|png)$`, so a screen that
        // invented or rewrote the URL would be rejected at the API rather than here.
        assertEquals("/api/photos/uploaded.jpg", sent.photoUrl)
        assertEquals(-30.0346, sent.latitude, 0.00001)
        // No capturedAt assertion: the request carries no such field. The server stamps the
        // capture time (Task 6), which is what stops a client backdating to yesterday's storm.
    }

    @Test
    fun `refuses to submit without a photo`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios")
        viewModel.submit()
        advanceUntilIdle()

        assertEquals("Tire uma foto do fenômeno antes de salvar.", viewModel.state.value.errorMessage)
        assertEquals(0, gateway.createdRequests.size)
    }

    @Test
    fun `refuses to submit without a position`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { null }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios")
        viewModel.onPhotoTaken(jpeg())
        viewModel.submit()
        advanceUntilIdle()

        assertEquals(
            "Não foi possível obter sua localização. Ative o GPS e tente de novo.",
            viewModel.state.value.errorMessage
        )
        assertEquals(0, gateway.createdRequests.size)
    }

    @Test
    fun `does not create the event when the upload fails`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway(uploadResult = Result.failure(IOException("no network")))
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios")
        viewModel.onPhotoTaken(jpeg())
        viewModel.submit()
        advanceUntilIdle()

        assertEquals(0, gateway.createdRequests.size)
        assertNotNull(viewModel.state.value.errorMessage)
        assertEquals(false, viewModel.state.value.saved)
    }
}

class FakeCaptureGateway(
    // A relative path, because that is what the real endpoint returns. A fake that hands back
    // an absolute CDN URL would let the ViewModel pass a test it fails against the server,
    // which rejects anything outside `^/api/photos/...` (Task 7).
    private val uploadResult: Result<String> = Result.success("/api/photos/uploaded.jpg")
) : CaptureGateway {

    val uploadedFiles = mutableListOf<File>()
    val createdRequests = mutableListOf<CreateWeatherEventRequest>()

    override suspend fun uploadPhoto(file: File): Result<String> {
        uploadedFiles += file
        return uploadResult
    }

    override suspend fun create(request: CreateWeatherEventRequest): Result<WeatherEventResponse> {
        createdRequests += request
        return Result.success(
            WeatherEventResponse(
                id = "00000000-0000-0000-0000-000000000001",
                title = request.title,
                description = request.description,
                photoUrl = request.photoUrl,
                capturedAt = "2026-08-07T17:42:10Z",   // server-stamped; the fake just picks one
                latitude = request.latitude,
                longitude = request.longitude,
                userId = "00000000-0000-0000-0000-000000000002",
                authorName = "Test Pilot"
            )
        )
    }
}
```

- [ ] **Step 2: Run and watch them fail**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.capture.*"
```

Expected: `Unresolved reference: CaptureViewModel`, `Unresolved reference: CaptureGateway`.

- [ ] **Step 3: Extract the gateway and make the repository implement it**

Create `app/src/main/java/com/example/skydex/ui/capture/CaptureGateway.kt`:

```kotlin
package com.example.skydex.ui.capture

import com.example.skydex.data.remote.dto.CreateWeatherEventRequest
import com.example.skydex.data.remote.dto.WeatherEventResponse
import java.io.File

interface CaptureGateway {
    suspend fun uploadPhoto(file: File): Result<String>
    suspend fun create(request: CreateWeatherEventRequest): Result<WeatherEventResponse>
}
```

In `data/repository/CaptureRepository.kt`, change the declaration to `class CaptureRepository(private val api: SkyDexApi) : CaptureGateway {` and prefix `uploadPhoto` and `create` with `override`.

- [ ] **Step 4: Write the ViewModel**

Create `app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`:

```kotlin
package com.example.skydex.ui.capture

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.CreateWeatherEventRequest
import com.example.skydex.util.Coordinates
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import java.io.File

data class CaptureUiState(
    val title: String = "",
    val description: String = "",
    val photoFile: File? = null,
    val coordinates: Coordinates? = null,
    val locating: Boolean = false,
    val submitting: Boolean = false,
    val saved: Boolean = false,
    val errorMessage: String? = null
)

class CaptureViewModel(
    private val captures: CaptureGateway,
    private val locationProvider: suspend () -> Coordinates?
) : ViewModel() {

    private val _state = MutableStateFlow(CaptureUiState())
    val state: StateFlow<CaptureUiState> = _state.asStateFlow()

    fun onTitleChanged(value: String) = _state.update { it.copy(title = value, errorMessage = null) }

    fun onDescriptionChanged(value: String) =
        _state.update { it.copy(description = value, errorMessage = null) }

    fun onPhotoTaken(file: File) = _state.update { it.copy(photoFile = file, errorMessage = null) }

    fun refreshLocation() {
        _state.update { it.copy(locating = true) }
        viewModelScope.launch {
            val coordinates = locationProvider()
            _state.update { it.copy(locating = false, coordinates = coordinates) }
        }
    }

    fun submit() {
        val current = _state.value

        val error = when {
            current.title.isBlank() || current.description.isBlank() ->
                "Preencha o título e a descrição."
            current.photoFile == null ->
                "Tire uma foto do fenômeno antes de salvar."
            current.coordinates == null ->
                "Não foi possível obter sua localização. Ative o GPS e tente de novo."
            else -> null
        }
        if (error != null) {
            _state.update { it.copy(errorMessage = error) }
            return
        }

        val photoFile = current.photoFile!!
        val coordinates = current.coordinates!!

        _state.update { it.copy(submitting = true, errorMessage = null) }
        viewModelScope.launch {
            val uploaded = captures.uploadPhoto(photoFile)
            if (uploaded.isFailure) {
                _state.update {
                    it.copy(submitting = false, errorMessage = "Falha ao enviar a foto. Tente de novo.")
                }
                return@launch
            }

            val request = CreateWeatherEventRequest(
                title = current.title,
                description = current.description,
                photoUrl = uploaded.getOrThrow(),
                latitude = coordinates.latitude,
                longitude = coordinates.longitude
            )

            captures.create(request)
                .onSuccess { _state.update { s -> s.copy(submitting = false, saved = true) } }
                .onFailure {
                    _state.update { s ->
                        s.copy(submitting = false, errorMessage = "Falha ao salvar o registro. Tente de novo.")
                    }
                }
        }
    }
}
```

Note what this ViewModel does **not** send: there is no `capturedAt`. Task 6 moved that to the server, so the client has no clock in this flow at all — which is exactly why `java.time` is absent from the imports.

`uploaded.getOrThrow()` is passed through untouched. Rewriting it — prefixing a host, normalising a slash — would produce a value the server's `@Pattern` rejects (Task 7). The path the upload returned is the only path the create call may carry.

- [ ] **Step 5: Run the ViewModel tests and confirm they pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.capture.*"
```

Expected: 4 tests PASS.

- [ ] **Step 6: Write the capture screen**

Create `app/src/main/java/com/example/skydex/ui/capture/CaptureScreen.kt`:

```kotlin
package com.example.skydex.ui.capture

import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.PhotoCamera
import androidx.compose.material3.Button
import androidx.compose.material3.Icon
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import coil.compose.AsyncImage
import com.example.skydex.util.LOCATION_PERMISSIONS
import com.example.skydex.util.PhotoCaptureFiles
import java.io.File

@Composable
fun CaptureScreen(
    viewModel: CaptureViewModel,
    onSaved: () -> Unit,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    val state by viewModel.state.collectAsState()

    // Held outside the ViewModel because TakePicture only reports success/failure, not the URI.
    var pendingFile by remember { mutableStateOf<File?>(null) }

    val takePicture = rememberLauncherForActivityResult(ActivityResultContracts.TakePicture()) { ok ->
        val file = pendingFile
        if (ok && file != null) viewModel.onPhotoTaken(file)
        pendingFile = null
    }

    val requestLocation = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { viewModel.refreshLocation() }

    LaunchedEffect(Unit) { requestLocation.launch(LOCATION_PERMISSIONS) }

    // The launcher's result map tells you WHICH outcome you got. Use it: a denied permission and a
    // missing fix both leave `state.coordinates` null, but they are not the same dead end. Android
    // will not re-prompt after a denial, so a screen that only says "não foi possível obter sua
    // localização" strands the user inside the app's core flow with no way out — their only path is
    // Settings, and nothing tells them that. `DeviceLocation.hasPermission()` is public precisely so
    // this distinction is available; the copy for the two cases must differ.

    LaunchedEffect(state.saved) { if (state.saved) onSaved() }

    Column(
        modifier = modifier
            .fillMaxSize()
            .background(Color(0xFFF3F4F6))
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text("Novo Registro", fontSize = 28.sp, fontWeight = FontWeight.Bold, color = Color.Black)

        if (state.photoFile != null) {
            AsyncImage(
                model = state.photoFile,
                contentDescription = "Foto do fenômeno",
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(220.dp)
                    .background(Color.LightGray, RoundedCornerShape(12.dp))
            )
        }

        OutlinedButton(
            onClick = {
                val file = PhotoCaptureFiles.newCaptureFile(context)
                pendingFile = file
                takePicture.launch(PhotoCaptureFiles.uriFor(context, file))
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.PhotoCamera, contentDescription = null)
            Spacer(Modifier.height(0.dp))
            Text(
                if (state.photoFile == null) "  Tirar Foto" else "  Tirar Outra Foto",
                fontWeight = FontWeight.Bold
            )
        }

        OutlinedTextField(
            value = state.title,
            onValueChange = viewModel::onTitleChanged,
            label = { Text("Título") },
            singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )

        OutlinedTextField(
            value = state.description,
            onValueChange = viewModel::onDescriptionChanged,
            label = { Text("Descrição") },
            maxLines = 4,
            modifier = Modifier.fillMaxWidth().height(110.dp)
        )

        Text(
            text = when {
                state.locating -> "Obtendo sua localização..."
                state.coordinates != null ->
                    "Localização: %.4f, %.4f".format(
                        state.coordinates!!.latitude,
                        state.coordinates!!.longitude
                    )
                else -> "Localização indisponível — ative o GPS."
            },
            color = if (state.coordinates != null) Color.Gray else Color(0xFFB91C1C),
            fontSize = 13.sp
        )

        state.errorMessage?.let {
            Text(it, color = Color(0xFFB91C1C), fontSize = 14.sp)
        }

        Button(
            onClick = viewModel::submit,
            enabled = !state.submitting,
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) {
            Text(if (state.submitting) "Salvando..." else "Salvar Registro", fontSize = 16.sp)
        }

        Spacer(Modifier.height(8.dp))
    }
}
```

Coil renders a `java.io.File` model directly, so no bitmap decoding is needed for the preview.

- [ ] **Step 7: Give Home a real position and a capture button**

Replace `ui/home/HomeViewModel.kt`:

```kotlin
package com.example.skydex.ui.home

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.NearbyPhenomenonResponse
import com.example.skydex.data.repository.CaptureRepository
import com.example.skydex.ui.common.UiState
import com.example.skydex.util.Coordinates
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

data class HomeData(
    val coordinates: Coordinates?,
    val phenomena: List<NearbyPhenomenonResponse>
)

class HomeViewModel(
    private val captures: CaptureRepository,
    private val locationProvider: suspend () -> Coordinates?
) : ViewModel() {

    private val _state = MutableStateFlow<UiState<HomeData>>(UiState.Loading)
    val state: StateFlow<UiState<HomeData>> = _state.asStateFlow()

    fun loadForCurrentPosition() {
        _state.value = UiState.Loading
        viewModelScope.launch {
            val coordinates = locationProvider()
            if (coordinates == null) {
                _state.value = UiState.Error("Ative o GPS para ver os fenômenos da sua região.")
                return@launch
            }
            captures.nearby(coordinates.latitude, coordinates.longitude)
                .onSuccess { _state.value = UiState.Success(HomeData(coordinates, it)) }
                .onFailure {
                    _state.value = UiState.Error("Não foi possível carregar os eventos próximos.")
                }
        }
    }
}
```

In `ui/home/HomeScreen.kt`, add a `onStartCapture: () -> Unit` parameter, request `LOCATION_PERMISSIONS` in a `LaunchedEffect(Unit)` the same way `CaptureScreen` does and call `viewModel.loadForCurrentPosition()` from its callback, read `UiState<HomeData>` instead of `UiState<List<…>>` (the list is `data.phenomena`), and put the "Registrar Novo Evento" button at the top, wired to `onStartCapture`.

> **Correction, recorded after Task 10 shipped.** An earlier draft of this line said "styled like the old `CardAcaoPrincipal`". That identifier exists nowhere — not in this repo, not in any prior commit, not elsewhere in this plan. It was a phantom reference of mine and the implementer rightly designed the button itself: a full-width `Card` in the app's existing accent `Color(0xFF0284C7)` with a camera icon and bold white label.

- [ ] **Step 8: Add the route**

In `ui/navigation/Routes.kt`:

```kotlin
    const val CAPTURE = "capture"
```

In `ui/navigation/SkyDexNavHost.kt`, update the `HOME` and `NEARBY` destinations to build the ViewModel with the location provider and pass the capture callback, and add the new destination:

```kotlin
            composable(Routes.HOME) {
                val vm: HomeViewModel = viewModel {
                    HomeViewModel(
                        ServiceLocator.captureRepository,
                        ServiceLocator.deviceLocation::current
                    )
                }
                HomeScreen(
                    viewModel = vm,
                    onStartCapture = { navController.navigate(Routes.CAPTURE) }
                )
            }

            composable(Routes.NEARBY) {
                val vm: HomeViewModel = viewModel {
                    HomeViewModel(
                        ServiceLocator.captureRepository,
                        ServiceLocator.deviceLocation::current
                    )
                }
                HomeScreen(
                    viewModel = vm,
                    onStartCapture = { navController.navigate(Routes.CAPTURE) }
                )
            }

            composable(Routes.CAPTURE) {
                val vm: CaptureViewModel = viewModel {
                    CaptureViewModel(
                        ServiceLocator.captureRepository,
                        ServiceLocator.deviceLocation::current
                    )
                }
                CaptureScreen(
                    viewModel = vm,
                    onSaved = {
                        navController.navigate(Routes.MY_CAPTURES) {
                            popUpTo(Routes.CAPTURE) { inclusive = true }
                        }
                    }
                )
            }
```

with imports for `CaptureScreen` and `CaptureViewModel`.

- [ ] **Step 9: Compile and run every Android test**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, all suites green.

- [ ] **Step 10: Walk the whole loop on a real phone**

```bash
cd <workspace>/SkyDex-backend
docker start skydex-db && sleep 3
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun &
sleep 25
cd ../SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:installDebug
```

Make sure `API_BASE_URL` in `local.properties` and `PUBLIC_BASE_URL` in the backend `.env` both point at your machine's LAN IP (`hostname -I`).

On the phone: log in → Home → grant location → "Registrar Novo Evento" → take a photo with the system camera → fill in the title and description → Save. **Expected: the app lands on "Meus Registros" and the new entry shows the photo you just took.** Confirm on the server:

```bash
ls -la <workspace>/SkyDex-backend/data/photos/
docker exec skydex-db psql -U guilherme_becker -d skydex \
  -c 'SELECT title, latitude, longitude, captured_at FROM weather_events ORDER BY captured_at DESC LIMIT 3;'
```

Expected: a real JPEG on disk and a row with non-zero coordinates.

- [ ] **Step 11: Commit**

```bash
git add -A app
git commit -m "feat: capture screen with camera, GPS and photo upload

Completes the core loop: photograph a phenomenon, stamp it with the
device position and time, upload the JPEG, and create the capture."
```

---

# Phase 3 — Gamification

The name "SkyDex" promises a Pokédex of weather. This phase delivers it: a catalog of phenomenon species with rarity tiers, a backend check that the phenomenon you claimed was actually happening where and when you say it was, and a collection screen that fills in as you capture.

The validation step is the whole point. Without it, "capture a hailstorm" is a text field. With it, the app knows.

---

### Task 11: The phenomenon catalog

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/Rarity.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/Phenomenon.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/NearbyPhenomenaService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/NearbyDtos.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/PhenomenonTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherControllerTest.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/WeatherEventDto.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/home/HomeScreen.kt` (Step 11)

**Interfaces:**
- Consumes: `OpenMeteoClient.fetchHourlyForecast` (Task 2); `NearbyPhenomenonResponse` (Task 2).
- Produces:
  - `domain.Rarity` — enum `COMMON(10)`, `UNCOMMON(25)`, `RARE(60)`, `EPIC(150)`, `LEGENDARY(400)`, property `val xp: Int`.
  - `domain.Phenomenon` — enum with `val displayName: String`, `val rarity: Rarity`, `val alertLevel: String`, `val weatherCodes: Set<Int>`, and `companion object { fun fromWeatherCode(code: Int): Phenomenon? }`. Nine entries, listed below.
  - `services.NearbyPhenomenaService(openMeteoClient)` with `fun forCoordinates(latitude: Double, longitude: Double): List<NearbyPhenomenonResponse>`.
  - `NearbyPhenomenonResponse(phenomenon: String, phenomenonName: String, rarity: String, time: String, temperatureCelsius: Double?, alertLevel: String)` — `phenomenon` is now the enum **name** (`THUNDERSTORM`), `phenomenonName` the display name (`Tempestade com Trovões`).
  - Android `NearbyPhenomenonResponse` gains `phenomenonName: String` and `rarity: String`.

- [ ] **Step 1: Write the failing catalog test**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/PhenomenonTest.kt`:

```kotlin
package com.skydex.api.domain

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Test

class PhenomenonTest {

    @Test
    fun `maps the WMO codes this app cares about`() {
        assertEquals(Phenomenon.CLEAR_SKY, Phenomenon.fromWeatherCode(0))
        assertEquals(Phenomenon.CLOUDS, Phenomenon.fromWeatherCode(3))
        assertEquals(Phenomenon.FOG, Phenomenon.fromWeatherCode(45))
        assertEquals(Phenomenon.RAIN, Phenomenon.fromWeatherCode(65))
        assertEquals(Phenomenon.RAIN_SHOWER, Phenomenon.fromWeatherCode(82))
        assertEquals(Phenomenon.SNOW, Phenomenon.fromWeatherCode(75))
        assertEquals(Phenomenon.THUNDERSTORM, Phenomenon.fromWeatherCode(95))
        assertEquals(Phenomenon.HAILSTORM, Phenomenon.fromWeatherCode(99))
    }

    @Test
    fun `returns null for a code outside the catalog`() {
        assertNull(Phenomenon.fromWeatherCode(4))
        assertNull(Phenomenon.fromWeatherCode(-1))
    }

    @Test
    fun `no weather code belongs to two species`() {
        val seen = mutableMapOf<Int, Phenomenon>()
        Phenomenon.entries.forEach { phenomenon ->
            phenomenon.weatherCodes.forEach { code ->
                val previous = seen.put(code, phenomenon)
                if (previous != null) {
                    throw AssertionError("code $code claimed by both $previous and $phenomenon")
                }
            }
        }
    }

    @Test
    fun `rarity tiers award increasing xp`() {
        val xpByTier = Rarity.entries.map { it.xp }
        assertEquals(xpByTier.sorted(), xpByTier, "Rarity entries must be declared cheapest first")
    }

    @Test
    fun `every species has a non-empty display name and at least one code`() {
        Phenomenon.entries.forEach {
            assert(it.displayName.isNotBlank()) { "$it has no display name" }
            assert(it.weatherCodes.isNotEmpty()) { "$it has no weather codes" }
        }
    }
}
```

- [ ] **Step 2: Run and watch it fail**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.PhenomenonTest"
```

Expected: `Unresolved reference: Phenomenon`.

- [ ] **Step 3: Write the rarity tiers**

Create `domain/Rarity.kt`:

```kotlin
package com.skydex.api.domain

/**
 * How hard a phenomenon is to catch, and what a confirmed capture of it is worth.
 * Declared cheapest first — PhenomenonTest enforces that ordering.
 */
enum class Rarity(val xp: Int) {
    COMMON(10),
    UNCOMMON(25),
    RARE(60),
    EPIC(150),
    LEGENDARY(400)
}
```

- [ ] **Step 4: Write the catalog**

Create `domain/Phenomenon.kt`:

```kotlin
package com.skydex.api.domain

/**
 * The SkyDex species list. Each entry owns a disjoint set of WMO weather codes
 * (https://open-meteo.com/en/docs — "Weather variable documentation"), which is how a
 * capture claim gets checked against what the sky was actually doing.
 *
 * Rarity is tuned for Brazil: snow is an expedition, hail with a thunderstorm is the trophy.
 */
enum class Phenomenon(
    val displayName: String,
    val rarity: Rarity,
    val alertLevel: String,
    val weatherCodes: Set<Int>
) {
    CLEAR_SKY("Céu Limpo", Rarity.COMMON, "Tranquilo", setOf(0, 1)),
    CLOUDS("Nublado", Rarity.COMMON, "Tranquilo", setOf(2, 3)),
    FOG("Nevoeiro Intenso", Rarity.UNCOMMON, "Interessante", setOf(45, 48)),
    DRIZZLE("Garoa", Rarity.COMMON, "Tranquilo", setOf(51, 53, 55, 56, 57)),
    RAIN("Chuva", Rarity.COMMON, "Atenção", setOf(61, 63, 65, 66, 67)),
    RAIN_SHOWER("Pancada de Chuva", Rarity.UNCOMMON, "Atenção", setOf(80, 81, 82)),
    SNOW("Neve", Rarity.EPIC, "Interessante", setOf(71, 73, 75, 77, 85, 86)),
    THUNDERSTORM("Tempestade com Trovões", Rarity.RARE, "Perigo", setOf(95)),
    HAILSTORM("Tempestade Severa com Granizo", Rarity.LEGENDARY, "Perigo Extremo!", setOf(96, 99));

    companion object {
        private val byWeatherCode: Map<Int, Phenomenon> =
            entries.flatMap { phenomenon -> phenomenon.weatherCodes.map { it to phenomenon } }.toMap()

        fun fromWeatherCode(code: Int): Phenomenon? = byWeatherCode[code]

        fun fromNameOrNull(name: String): Phenomenon? =
            entries.firstOrNull { it.name.equals(name, ignoreCase = true) }
    }
}
```

`displayName` and `alertLevel` are user-facing Portuguese copy, which is why they are not in English — see the Global Constraints.

- [ ] **Step 5: Run the catalog test and confirm it passes**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.PhenomenonTest"
```

Expected: 5 tests PASS.

- [ ] **Step 6: Write the failing nearby-service test**

`controller/WeatherControllerTest.kt` currently holds **three** tests, not two. Task 2's fix round
built a `slotAt(hoursFromNow: Long)` helper in that file that formats a slot relative to
`Instant.now()`. **Keep that helper and use it.** Do not write a literal date anywhere in this file:
a hardcoded `2026-08-07T12:00` drifts into the past and the test silently starts asserting an empty
list — the STANDING RULE restated under Step 9 below.

Handle all three existing tests:

1. `maps a thunderstorm weather code to a danger-level phenomenon` → replace with the catalog
   version below (renamed to `…to its catalog entry`).
2. `returns an empty list when Open-Meteo is unreachable` → unchanged, keep as is.
3. `never reports a slot that has already elapsed` → **KEEP, but it breaks under the new contract**
   and must be updated: it asserts `$[0].phenomenon` is `"Chuva"`, which is now the value of
   `phenomenonName`. Change that assertion to `$[0].phenomenon` = `"RAIN"` and add
   `$[0].phenomenonName` = `"Chuva"`. Do not delete it — it is a Task 2 regression test for the
   past-slot filter.

Then add the two new cases below (catalog gap-skipping, and the current-hour boundary).

```kotlin
    @Test
    fun `maps a thunderstorm weather code to its catalog entry`() {
        val user = persistUser(email = "weather@skydex.com")
        `when`(openMeteoClient.fetchHourlyForecast(-23.55, -46.63)).thenReturn(
            OpenMeteoResponse(
                latitude = -23.55,
                longitude = -46.63,
                hourly = HourlyData(
                    time = listOf(slotAt(1)),
                    temperatureCelsius = listOf(21.5),
                    weatherCode = listOf(95)
                )
            )
        )

        mockMvc.perform(
            get("/api/weather/nearby")
                .param("lat", "-23.55")
                .param("lon", "-46.63")
                .header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$[0].phenomenon").value("THUNDERSTORM"))
            .andExpect(jsonPath("$[0].phenomenonName").value("Tempestade com Trovões"))
            .andExpect(jsonPath("$[0].rarity").value("RARE"))
            .andExpect(jsonPath("$[0].alertLevel").value("Perigo"))
            .andExpect(jsonPath("$[0].temperatureCelsius").value(21.5))
    }

    @Test
    fun `skips hours whose weather code is not in the catalog`() {
        val user = persistUser(email = "gaps@skydex.com")
        `when`(openMeteoClient.fetchHourlyForecast(1.0, 2.0)).thenReturn(
            OpenMeteoResponse(
                latitude = 1.0,
                longitude = 2.0,
                hourly = HourlyData(
                    time = listOf(slotAt(1), slotAt(2), slotAt(3)),
                    temperatureCelsius = listOf(20.0, 21.0, 22.0),
                    weatherCode = listOf(4, null, 45)
                )
            )
        )

        mockMvc.perform(
            get("/api/weather/nearby")
                .param("lat", "1.0")
                .param("lon", "2.0")
                .header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].phenomenon").value("FOG"))
    }

    // `returns an empty list when Open-Meteo is unreachable` already exists unchanged — leave it.

    // The whole reason this service truncates the window to the hour. Under the OLD controller's
    // bare `Instant.now()` the slot covering the current hour is in the past and gets skipped;
    // under `truncatedTo(HOURS)` it is included. That difference is not cosmetic: the capture
    // screen renders this list while Task 12's validator scores the photo against the slot nearest
    // its timestamp — the same current-hour slot. Without this test the truncation is revertible
    // in silence, and the symptom would surface much later as a spurious UNCONFIRMED.
    // `slotAt(0)` truncates to the current hour, which is the discriminating value.
    @Test
    fun `includes the slot covering the current hour`() {
        val user = persistUser(email = "current-hour@skydex.com")
        `when`(openMeteoClient.fetchHourlyForecast(5.0, 6.0)).thenReturn(
            OpenMeteoResponse(
                latitude = 5.0,
                longitude = 6.0,
                hourly = HourlyData(
                    time = listOf(slotAt(0)),
                    temperatureCelsius = listOf(23.0),
                    weatherCode = listOf(95)
                )
            )
        )

        mockMvc.perform(
            get("/api/weather/nearby")
                .param("lat", "5.0")
                .param("lon", "6.0")
                .header("Authorization", authHeaderFor(user))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].phenomenon").value("THUNDERSTORM"))
    }
```

**Prove this last test discriminates rather than assuming it.** After Step 9 is green, temporarily
change `truncatedTo(ChronoUnit.HOURS)` back to a bare `Instant.now()`, re-run
`WeatherControllerTest`, and confirm `includes the slot covering the current hour` is the test that
fails. Restore, re-confirm green, and put both the RED and the GREEN output in your report. A test
claimed to discriminate but never observed doing so is exactly what this step exists to prevent.
(Note the one honest limit: if the suite happens to run in the first seconds of a wall-clock hour,
`slotAt(0)` and `Instant.now()` are close enough that the mutant may pass. If the mutation shows no
failure, re-run it rather than concluding the test is weak.)

- [ ] **Step 7: Run and watch the new assertions fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.WeatherControllerTest"
```

Expected: `No value at JSON path "$[0].phenomenonName"`.

- [ ] **Step 8: Extend the response DTO**

Replace `dto/NearbyDtos.kt`:

```kotlin
package com.skydex.api.dto

import com.skydex.api.domain.Phenomenon

data class NearbyPhenomenonResponse(
    val phenomenon: String,
    val phenomenonName: String,
    val rarity: String,
    val time: String,
    val temperatureCelsius: Double?,
    val alertLevel: String
) {
    companion object {
        fun from(phenomenon: Phenomenon, time: String, temperatureCelsius: Double?) =
            NearbyPhenomenonResponse(
                phenomenon = phenomenon.name,
                phenomenonName = phenomenon.displayName,
                rarity = phenomenon.rarity.name,
                time = time,
                temperatureCelsius = temperatureCelsius,
                alertLevel = phenomenon.alertLevel
            )
    }
}
```

- [ ] **Step 9: Move the mapping into a service**

Create `services/NearbyPhenomenaService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Phenomenon
import com.skydex.api.dto.NearbyPhenomenonResponse
import org.springframework.stereotype.Service

@Service
class NearbyPhenomenaService(private val openMeteoClient: OpenMeteoClient) {

    /**
     * The next [FORECAST_HOURS] hourly slots that map to a species in the catalog.
     *
     * OpenMeteoClient asks for past_days=1, so the arrays BEGIN YESTERDAY at 00:00 UTC and run
     * 72 slots. Slots are therefore selected by timestamp, never by array index — slicing
     * `0 until 24` would make this service report the 24 hours that already elapsed. Verified
     * against the live API: index 0 is yesterday 00:00 UTC, index 24 is today 00:00 UTC.
     */
    fun forCoordinates(latitude: Double, longitude: Double): List<NearbyPhenomenonResponse> {
        val hourly = openMeteoClient.fetchHourlyForecast(latitude, longitude)?.hourly
            ?: return emptyList()

        // Truncated to the hour so the slot COVERING the current hour is included, not skipped.
        // This matters beyond cosmetics: the capture screen shows this list, and Task 12's
        // validator scores a capture against the hourly slot nearest its timestamp — which is
        // that same current-hour slot. Comparing against a bare Instant.now() would offer the
        // user next hour's phenomenon while validating their photo against this hour's.
        val windowStart = Instant.now().truncatedTo(ChronoUnit.HOURS)
        val slots = minOf(hourly.time.size, hourly.weatherCode.size, hourly.temperatureCelsius.size)
        val results = mutableListOf<NearbyPhenomenonResponse>()

        for (i in 0 until slots) {
            if (results.size >= FORECAST_HOURS) break
            val slotTime = parseUtcSlot(hourly.time[i]) ?: continue
            if (slotTime.isBefore(windowStart)) continue

            val code = hourly.weatherCode[i] ?: continue
            val phenomenon = Phenomenon.fromWeatherCode(code) ?: continue
            results.add(
                NearbyPhenomenonResponse.from(phenomenon, hourly.time[i], hourly.temperatureCelsius[i])
            )
        }
        return results
    }

    /** Open-Meteo returns "2026-08-07T14:00" with no offset; the client requested timezone=UTC. */
    private fun parseUtcSlot(raw: String): Instant? = try {
        LocalDateTime.parse(raw).toInstant(ZoneOffset.UTC)
    } catch (e: DateTimeParseException) {
        null
    }

    private companion object {
        const val FORECAST_HOURS = 24
    }
}
```

Imports: `java.time.Instant`, `java.time.LocalDateTime`, `java.time.ZoneOffset`, `java.time.temporal.ChronoUnit`, `java.time.format.DateTimeParseException`.

**Every test for this service must build its slot timestamps relative to `Instant.now()`** — for example `LocalDateTime.ofInstant(Instant.now().plus(1, ChronoUnit.HOURS), ZoneOffset.UTC).withMinute(0).withSecond(0).withNano(0).toString()`. A hardcoded date string drifts into the past and the test starts asserting an empty list without anyone noticing.

Replace `controllers/WeatherController.kt` with the thin version:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.NearbyPhenomenonResponse
import com.skydex.api.services.NearbyPhenomenaService
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/weather")
class WeatherController(private val nearbyPhenomena: NearbyPhenomenaService) {

    @GetMapping("/nearby")
    fun nearby(
        @RequestParam lat: Double,
        @RequestParam lon: Double
    ): List<NearbyPhenomenonResponse> = nearbyPhenomena.forCoordinates(lat, lon)
}
```

- [ ] **Step 10: Run the full backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS.

- [ ] **Step 11: Mirror the DTO on Android**

In `data/remote/dto/WeatherEventDto.kt`:

```kotlin
data class NearbyPhenomenonResponse(
    /** Enum name, e.g. "THUNDERSTORM" — stable identifier for the species. */
    val phenomenon: String,
    /** Display copy, e.g. "Tempestade com Trovões". */
    val phenomenonName: String,
    val rarity: String,
    val time: String,
    val temperatureCelsius: Double?,
    val alertLevel: String
)
```

In `ui/home/HomeScreen.kt`, change the card to render `phenomenonName` as the title and add a small rarity chip next to the alert-level chip, colouring it by rarity:

```kotlin
private fun rarityColor(rarity: String): Color = when (rarity) {
    "LEGENDARY" -> Color(0xFFF59E0B)
    "EPIC" -> Color(0xFF8B5CF6)
    "RARE" -> Color(0xFF3B82F6)
    "UNCOMMON" -> Color(0xFF10B981)
    else -> Color(0xFF6B7280)
}
```

`PhenomenonCard` currently renders `phenomenon.phenomenon` as the title. That field now holds the
enum name (`THUNDERSTORM`), so leaving it would print an identifier at the user. Switch the title to
`phenomenonName`.

**`NearbyPhenomenonResponse` gains two non-nullable fields with no defaults, so every construction
site must be updated — including the ones nothing references until the compiler does.** Task 6 was
bitten by exactly this. There are three files beyond the DTO:

- `ui/home/HomeScreen.kt` — the `previewPhenomena` fixtures (three positional constructions). Give
  them real enum names and rarities so the `@Preview`s exercise the rarity chip rather than
  defaulting past it.
- `app/src/test/java/com/example/skydex/ui/home/HomeViewModelTest.kt` — the `storm` fixture.
- `app/src/test/java/com/example/skydex/data/repository/CaptureRepositoryTest.kt` — a positional
  construction at roughly line 58.

- [ ] **Step 11b: While you are in `HomeScreen.kt`, close its permission dead end**

This is not about the phenomenon catalog. It is here because this step already opens the file, and
doing it now costs a few lines instead of a whole review round later.

A re-review of Task 10 found that `HomeScreen`'s permission callback **discards the `results` map**.
So a user who denied location permanently taps "Tentar novamente", the contract returns instantly
with a denial, `loadForCurrentPosition()` finds no permission, and the screen prints *"Ative o GPS
para ver os fenômenos da sua região."* — sending them hunting for a switch that was never the
problem, next to a button that is a guaranteed no-op they can tap forever.

`CaptureScreen` already solves exactly this, and is the pattern to copy: read the denial out of the
result map into a `rememberSaveable` flag, give the two cases different copy, and offer
`Settings.ACTION_APPLICATION_DETAILS_SETTINGS` for the denial — the only real way out once Android
has stopped prompting. `DeviceLocation.hasPermission()` is public for this.

Also fix the comment at `HomeScreen.kt:68-73`, which currently overclaims: routing retry through the
launcher rescues a user who denied *and then fixed it in Settings*, and does nothing for one who has
not. Say that.

- [ ] **Step 12: Compile and test the Android app**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 13: Commit**

```bash
cd <workspace>/SkyDex-backend
git add -A src && git commit -m "feat: phenomenon catalog with rarity tiers"
cd ../SkyDex---frontend
git add -A app && git commit -m "feat: show phenomenon species and rarity on the nearby list"
```

---

### Task 12: Validate a capture against what the sky was actually doing

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/ValidationStatus.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/CaptureValidationService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/WeatherEvent.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestFixtures.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`

**Interfaces:**
- Consumes: `Phenomenon`, `Rarity` (Task 11); `OpenMeteoClient`, `HourlyData` (Task 2); `WeatherEvent` with coordinates (Task 6).
- Produces:
  - `domain.ValidationStatus` — enum `CONFIRMED`, `UNCONFIRMED`.
  - `services.CaptureValidationService(openMeteoClient)` with
    `fun validate(claimed: Phenomenon, latitude: Double, longitude: Double, capturedAt: Instant): ValidationResult`
    where `data class ValidationResult(val status: ValidationStatus, val observedWeatherCode: Int?, val xpAwarded: Int)`.
  - `WeatherEvent` gains `var phenomenon: Phenomenon` (`@Enumerated(EnumType.STRING)`), `var validationStatus: ValidationStatus` (`@Enumerated(EnumType.STRING)`), `var observedWeatherCode: Int?`, `var xpAwarded: Int` — appended after `longitude`, before `userId`.
  - `CreateWeatherEventRequest` gains `phenomenon: String` (`@field:NotBlank`), appended last.
  - `persistEvent` gains `phenomenon: Phenomenon = Phenomenon.RAIN`, `validationStatus: ValidationStatus = ValidationStatus.CONFIRMED`, `xpAwarded: Int = Phenomenon.RAIN.rarity.xp`.
  - `POST /api/events` returns **400** with `{"error":"Unknown phenomenon: <value>"}` when the claimed species is not in the catalog.

Validation rules, stated plainly so the implementer does not have to infer them:

1. Ask Open-Meteo for the hourly forecast at the capture's coordinates (`past_days=1&forecast_days=2&timezone=UTC` — already what `OpenMeteoClient` requests).
2. Find the hourly slot whose timestamp is nearest to `capturedAt`. Slot times come back as `2026-08-07T14:00` with no zone; parse with `LocalDateTime.parse` and convert with `ZoneOffset.UTC`.
3. If the nearest slot is more than **90 minutes** away, give up: `UNCONFIRMED`, 0 XP. That window covers phone clock skew and the hourly granularity of the data without letting someone stamp a capture onto yesterday's storm.
4. Map the slot's weather code through `Phenomenon.fromWeatherCode`. If it equals the claimed species: `CONFIRMED`, award `claimed.rarity.xp`. Otherwise `UNCONFIRMED`, 0 XP.
5. If Open-Meteo is unreachable or returns no hourly data: `UNCONFIRMED`, 0 XP. **A capture is never rejected for this** — the photo is still the user's, it just does not score. Making an upstream outage lose the user's work would be the wrong trade.

- [ ] **Step 1: Write the failing validation-service tests**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt`:

```kotlin
package com.skydex.api.service

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.dto.HourlyData
import com.skydex.api.dto.OpenMeteoResponse
import com.skydex.api.services.CaptureValidationService
import com.skydex.api.services.OpenMeteoClient
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.mockito.Mockito.mock
import org.mockito.Mockito.`when`
import java.time.Instant

class CaptureValidationServiceTest {

    private val client = mock(OpenMeteoClient::class.java)
    private val service = CaptureValidationService(client)

    private fun forecast(vararg slots: Pair<String, Int?>) = OpenMeteoResponse(
        latitude = -30.0,
        longitude = -51.0,
        hourly = HourlyData(
            time = slots.map { it.first },
            temperatureCelsius = slots.map { 20.0 },
            weatherCode = slots.map { it.second }
        )
    )

    @Test
    fun `confirms a claim that matches the observed code and awards its rarity xp`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(
            forecast("2026-08-07T14:00" to 95, "2026-08-07T15:00" to 3)
        )

        val result = service.validate(
            claimed = Phenomenon.THUNDERSTORM,
            latitude = -30.0,
            longitude = -51.0,
            capturedAt = Instant.parse("2026-08-07T14:10:00Z")
        )

        assertEquals(ValidationStatus.CONFIRMED, result.status)
        assertEquals(95, result.observedWeatherCode)
        assertEquals(Phenomenon.THUNDERSTORM.rarity.xp, result.xpAwarded)
    }

    @Test
    fun `does not confirm a claim that contradicts the observed code`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(
            forecast("2026-08-07T14:00" to 0)
        )

        val result = service.validate(
            claimed = Phenomenon.HAILSTORM,
            latitude = -30.0,
            longitude = -51.0,
            capturedAt = Instant.parse("2026-08-07T14:10:00Z")
        )

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(0, result.observedWeatherCode)
        assertEquals(0, result.xpAwarded)
    }

    @Test
    fun `picks the nearest hourly slot, not the first one`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(
            forecast(
                "2026-08-07T12:00" to 0,
                "2026-08-07T13:00" to 0,
                "2026-08-07T14:00" to 95
            )
        )

        val result = service.validate(
            claimed = Phenomenon.THUNDERSTORM,
            latitude = -30.0,
            longitude = -51.0,
            capturedAt = Instant.parse("2026-08-07T13:50:00Z")
        )

        assertEquals(ValidationStatus.CONFIRMED, result.status)
    }

    @Test
    fun `refuses to confirm when the nearest slot is more than 90 minutes away`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(
            forecast("2026-08-07T14:00" to 95)
        )

        val result = service.validate(
            claimed = Phenomenon.THUNDERSTORM,
            latitude = -30.0,
            longitude = -51.0,
            capturedAt = Instant.parse("2026-08-07T18:00:00Z")
        )

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(0, result.xpAwarded)
    }

    @Test
    fun `treats an upstream outage as unconfirmed rather than an error`() {
        `when`(client.fetchHourlyForecast(-30.0, -51.0)).thenReturn(null)

        val result = service.validate(
            claimed = Phenomenon.THUNDERSTORM,
            latitude = -30.0,
            longitude = -51.0,
            capturedAt = Instant.parse("2026-08-07T14:10:00Z")
        )

        assertEquals(ValidationStatus.UNCONFIRMED, result.status)
        assertEquals(null, result.observedWeatherCode)
        assertEquals(0, result.xpAwarded)
    }
}
```

This is a plain unit test with no Spring context — it does not extend `IntegrationTestBase` and needs no container.

- [ ] **Step 2: Run and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.CaptureValidationServiceTest"
```

Expected: `Unresolved reference: CaptureValidationService`.

- [ ] **Step 3: Write the validation status enum**

Create `domain/ValidationStatus.kt`:

```kotlin
package com.skydex.api.domain

/**
 * Whether Open-Meteo's record for the capture's place and time agrees with what the user
 * claimed to have photographed. UNCONFIRMED is not an accusation — an upstream outage or a
 * capture outside the forecast window lands here too. It only means: no XP awarded.
 */
enum class ValidationStatus { CONFIRMED, UNCONFIRMED }
```

- [ ] **Step 4: Write the validation service**

Create `services/CaptureValidationService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import org.springframework.stereotype.Service
import java.time.Duration
import java.time.Instant
import java.time.LocalDateTime
import java.time.ZoneOffset
import java.time.format.DateTimeParseException
import kotlin.math.abs

data class ValidationResult(
    val status: ValidationStatus,
    val observedWeatherCode: Int?,
    val xpAwarded: Int
)

@Service
class CaptureValidationService(private val openMeteoClient: OpenMeteoClient) {

    /**
     * Checks a capture claim against Open-Meteo's hourly record for that place and time.
     * Never throws: an unreachable upstream or a capture outside the forecast window comes
     * back UNCONFIRMED with zero XP, so a user never loses a photo to someone else's outage.
     */
    fun validate(
        claimed: Phenomenon,
        latitude: Double,
        longitude: Double,
        capturedAt: Instant
    ): ValidationResult {
        val hourly = openMeteoClient.fetchHourlyForecast(latitude, longitude)?.hourly
            ?: return unconfirmed(null)

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
            return unconfirmed(null)
        }

        val observedCode = hourly.weatherCode[nearestIndex] ?: return unconfirmed(null)
        val observed = Phenomenon.fromWeatherCode(observedCode)

        return if (observed == claimed) {
            ValidationResult(ValidationStatus.CONFIRMED, observedCode, claimed.rarity.xp)
        } else {
            ValidationResult(ValidationStatus.UNCONFIRMED, observedCode, 0)
        }
    }

    /** Open-Meteo returns "2026-08-07T14:00" with no offset; we requested timezone=UTC. */
    private fun parseSlot(raw: String): Instant? = try {
        LocalDateTime.parse(raw).toInstant(ZoneOffset.UTC)
    } catch (e: DateTimeParseException) {
        null
    }

    private fun unconfirmed(observedCode: Int?) =
        ValidationResult(ValidationStatus.UNCONFIRMED, observedCode, 0)

    private companion object {
        /** Covers hourly granularity plus a plausible amount of phone clock skew. */
        val MAX_SKEW: Duration = Duration.ofMinutes(90)
    }
}
```

- [ ] **Step 5: Run the service test and confirm it passes**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.service.CaptureValidationServiceTest"
```

Expected: 5 tests PASS.

- [ ] **Step 6: Write the failing controller test**

Append to `controller/WeatherEventControllerTest.kt`:

```kotlin
    /**
     * Open-Meteo's label for the hour the server is in right now, e.g. "2026-08-07T14:00".
     *
     * The capture time is stamped server-side with `Instant.now()` (Task 6), so a forecast slot
     * hard-coded to a fixed date would drift out of `MAX_SKEW` and every one of these tests would
     * start failing on a wall clock the author never ran. Deriving the slot from the same clock
     * the server reads keeps them honest instead of merely green.
     */
    private fun currentSlotLabel(): String =
        LocalDateTime.ofInstant(Instant.now().truncatedTo(ChronoUnit.HOURS), ZoneOffset.UTC).toString()

    @Test
    fun `confirms a capture whose claim matches the observed weather and awards xp`() {
        val user = persistUser(email = "hunter@skydex.com")

        `when`(openMeteoClient.fetchHourlyForecast(-30.0346, -51.2177)).thenReturn(
            OpenMeteoResponse(
                latitude = -30.0346,
                longitude = -51.2177,
                hourly = HourlyData(
                    time = listOf(currentSlotLabel()),
                    temperatureCelsius = listOf(19.0),
                    weatherCode = listOf(95)
                )
            )
        )

        val payload = CreateWeatherEventRequest(
            title = "Tempestade",
            description = "Raios sobre o bairro",
            photoUrl = "/api/photos/storm.jpg",
            latitude = -30.0346,
            longitude = -51.2177,
            phenomenon = "THUNDERSTORM"
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.phenomenon").value("THUNDERSTORM"))
            .andExpect(jsonPath("$.validationStatus").value("CONFIRMED"))
            .andExpect(jsonPath("$.xpAwarded").value(60))
    }

    @Test
    fun `saves the capture but awards no xp when the claim is contradicted`() {
        val user = persistUser(email = "optimist@skydex.com")

        `when`(openMeteoClient.fetchHourlyForecast(-30.0346, -51.2177)).thenReturn(
            OpenMeteoResponse(
                latitude = -30.0346,
                longitude = -51.2177,
                hourly = HourlyData(
                    time = listOf(currentSlotLabel()),
                    temperatureCelsius = listOf(28.0),
                    weatherCode = listOf(0)
                )
            )
        )

        val payload = CreateWeatherEventRequest(
            title = "Granizo (eu juro)",
            description = "Pedras do tamanho de bolas de golfe",
            photoUrl = "/api/photos/hail.jpg",
            latitude = -30.0346,
            longitude = -51.2177,
            phenomenon = "HAILSTORM"
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.validationStatus").value("UNCONFIRMED"))
            .andExpect(jsonPath("$.xpAwarded").value(0))
    }

    @Test
    fun `rejects a phenomenon that is not in the catalog`() {
        val user = persistUser(email = "inventor@skydex.com")

        val payload = CreateWeatherEventRequest(
            title = "Chuva de sapos",
            description = "Aconteceu mesmo",
            photoUrl = "/api/photos/frogs.jpg",
            latitude = -30.0346,
            longitude = -51.2177,
            phenomenon = "FROG_RAIN"
        )

        mockMvc.perform(
            post("/api/events")
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("Unknown phenomenon: FROG_RAIN"))
    }
```

Add to the class a `@MockBean private lateinit var openMeteoClient: OpenMeteoClient` and the imports `com.skydex.api.dto.HourlyData`, `com.skydex.api.dto.OpenMeteoResponse`, `com.skydex.api.services.OpenMeteoClient`, `org.mockito.Mockito.when`, `org.springframework.boot.test.mock.mockito.MockBean`.

Every other `CreateWeatherEventRequest(...)` in the test sources now needs `phenomenon = "RAIN"` (or another catalog name). At the time of writing there are **12 construction sites, all in `WeatherEventControllerTest.kt`** — the compiler will name any you miss, but the count is here so a partial pass is obvious. Because `openMeteoClient` is mocked and returns `null` by default, those captures come back `UNCONFIRMED` with 0 XP, which is what the older assertions expect.

- [ ] **Step 7: Run and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.WeatherEventControllerTest"
```

Expected: `No value passed for parameter 'phenomenon'`.

- [ ] **Step 8: Extend the entity**

In `models/WeatherEvent.kt`, add these four properties after `longitude` and before `userId`, plus the imports `com.skydex.api.domain.Phenomenon`, `com.skydex.api.domain.ValidationStatus`, `jakarta.persistence.Enumerated`, `jakarta.persistence.EnumType`:

```kotlin
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    var phenomenon: Phenomenon = Phenomenon.CLOUDS,

    @Enumerated(EnumType.STRING)
    @Column(name = "validation_status", nullable = false, length = 16)
    var validationStatus: ValidationStatus = ValidationStatus.UNCONFIRMED,

    @Column(name = "observed_weather_code")
    var observedWeatherCode: Int? = null,

    @Column(name = "xp_awarded", nullable = false)
    var xpAwarded: Int = 0,
```

Enums are persisted as strings, not ordinals — reordering the catalog must never silently reinterpret stored rows.

- [ ] **Step 9: Extend the request DTO**

In `dto/WeatherEventDtos.kt`, append to `CreateWeatherEventRequest`:

```kotlin
    @field:NotBlank(message = "Phenomenon is required")
    val phenomenon: String
```

**The response DTO gains three fields in this task too** — the ones this task's logic produces. Append them to `WeatherEventResponse` after `authorName`, and to its `from` mapper:

```kotlin
    val phenomenon: String,          // = event.phenomenon.name
    val validationStatus: String,    // = event.validationStatus.name
    val xpAwarded: Int               // = event.xpAwarded
```

`from(...)` is the single construction site for this DTO (all nine call sites go through it), so widening it costs no other edits.

Task 13 appends the two remaining, purely presentational fields — `phenomenonName` (`event.phenomenon.displayName`) and `rarity` (`event.phenomenon.rarity.name`) — when the collection screen needs them. The split is deliberate: a task asserts on the contract it creates. Step 6's controller tests above check `$.phenomenon`, `$.validationStatus` and `$.xpAwarded`, and they cannot pass unless those three land here.

**This deliberately breaks the Android capture flow for two tasks — a sequencing decision, recorded here so it is visible rather than discovered.** `phenomenon` becomes required here, but Android's `CreateWeatherEventRequest` does not gain the field until Task 14, which is where the species picker UI lives — there is nowhere sensible to put a value before then. In the window between, `POST /api/events` from the app omits the key, jackson-module-kotlin refuses to bind a non-null `String` from an absent property, and the request fails with a 400 carrying Spring's default error body rather than this app's `ErrorResponse` envelope (there is no `HttpMessageNotReadableException` handler; see the backlog).

The window is safe only because nothing installs the app inside it: the device-walkthrough steps sit in Task 10 and again in Task 14, never in 12 or 13. If you reorder these tasks, that stops being true.

- [ ] **Step 10: Wire validation into event creation**

In `WeatherEventController`, inject the service and rewrite `create`:

```kotlin
@RestController
@RequestMapping("/api/events")
class WeatherEventController(
    private val events: WeatherEventRepository,
    private val users: UserRepository,
    private val validation: CaptureValidationService
) {

    @PostMapping
    fun create(
        @AuthenticationPrincipal currentUser: User,
        @Valid @RequestBody request: CreateWeatherEventRequest
    ): ResponseEntity<WeatherEventResponse> {
        val claimed = Phenomenon.fromNameOrNull(request.phenomenon)
            ?: throw BadUploadException("Unknown phenomenon: ${request.phenomenon}")

        // One stamp, used for BOTH the validation and the stored row. Reading Instant.now()
        // twice could straddle an hour boundary and validate against a slot the capture is
        // then not recorded in — rare, but it would be an unreproducible "why is this
        // UNCONFIRMED" bug. The client never supplies this; see Task 6.
        val capturedAt = Instant.now()

        val result = validation.validate(
            claimed = claimed,
            latitude = request.latitude,
            longitude = request.longitude,
            capturedAt = capturedAt
        )

        val saved = events.save(
            WeatherEvent(
                id = null,
                title = request.title,
                description = request.description,
                photoUrl = request.photoUrl,
                capturedAt = capturedAt,
                latitude = request.latitude,
                longitude = request.longitude,
                phenomenon = claimed,
                validationStatus = result.status,
                observedWeatherCode = result.observedWeatherCode,
                xpAwarded = result.xpAwarded,
                userId = currentUser.id!!
            )
        )
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(WeatherEventResponse.from(saved, currentUser, publicBaseUrl))
    }
```

Add the imports `com.skydex.api.domain.Phenomenon`, `com.skydex.api.services.CaptureValidationService`, `com.skydex.api.services.BadUploadException`, and in the controller `java.time.Instant` for the server-side stamp.

The test class additionally needs `java.time.Instant`, `java.time.LocalDateTime`, `java.time.ZoneOffset` and `java.time.temporal.ChronoUnit` for `currentSlotLabel()`.

`BadUploadException` already maps to a 400 with an `ErrorResponse` body via `GlobalExceptionHandler` (Task 7, step 4). If its name reads oddly for a non-upload case, rename it to `BadRequestException` here and update the handler and `PhotoStorageService` together — but do not leave two exceptions that mean the same thing.

In `update`, do **not** re-run validation: editing a title must not re-roll XP. Leave `phenomenon`, `validationStatus`, `observedWeatherCode` and `xpAwarded` untouched there.

Pin that with a test, or it is revertible in silence — the same lesson Task 6 learned about the coordinates, one layer up. This one matters more: a re-roll on edit turns `PUT` into a free retry button, so a contradicted capture could be edited over and over until the forecast happened to agree.

```kotlin
    @Test
    fun `editing a capture does not re-roll its validation or xp`() {
        val user = persistUser(email = "rerooler@skydex.com")
        // UNCONFIRMED / 0 XP passed EXPLICITLY. `persistEvent` defaults to CONFIRMED with RAIN's
        // 10 XP (Step 11), which would both contradict the assertions below and destroy the
        // point of the test: starting from CONFIRMED, a handler that re-validated would leave it
        // CONFIRMED and the test would pass while proving nothing.
        val event = persistEvent(
            owner = user,
            latitude = -30.0346,
            longitude = -51.2177,
            validationStatus = ValidationStatus.UNCONFIRMED,
            xpAwarded = 0
        )

        // The capture is stored UNCONFIRMED with 0 XP. Make the forecast agree
        // now, so a handler that re-validated on edit would flip it to CONFIRMED and pay out.
        `when`(openMeteoClient.fetchHourlyForecast(-30.0346, -51.2177)).thenReturn(
            OpenMeteoResponse(
                latitude = -30.0346,
                longitude = -51.2177,
                hourly = HourlyData(
                    time = listOf(currentSlotLabel()),
                    temperatureCelsius = listOf(19.0),
                    weatherCode = listOf(95)
                )
            )
        )

        val payload = CreateWeatherEventRequest(
            title = "Titulo novo",
            description = "So o texto mudou",
            photoUrl = "/api/photos/x.jpg",
            latitude = -30.0346,
            longitude = -51.2177,
            phenomenon = "THUNDERSTORM"
        )

        mockMvc.perform(
            put("/api/events/{id}", event.id!!)
                .header("Authorization", authHeaderFor(user))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(payload))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.title").value("Titulo novo"))
            .andExpect(jsonPath("$.validationStatus").value("UNCONFIRMED"))
            .andExpect(jsonPath("$.xpAwarded").value(0))
    }
```

The title assertion is not decoration: without it the test would also pass against a handler that ignored the request body entirely, which is a different bug wearing the same green.

- [ ] **Step 11: Extend the fixture**

In `support/TestFixtures.kt`, add the parameters and pass them through:

```kotlin
fun IntegrationTestBase.persistEvent(
    owner: User,
    title: String = "Aurora",
    description: String = "Green lights in the night sky",
    photoUrl: String = "/api/photos/test.jpg",
    capturedAt: Instant = Instant.now(),
    latitude: Double = -23.55,
    longitude: Double = -46.63,
    phenomenon: Phenomenon = Phenomenon.RAIN,
    validationStatus: ValidationStatus = ValidationStatus.CONFIRMED,
    xpAwarded: Int = Phenomenon.RAIN.rarity.xp
): WeatherEvent = weatherEventRepository.save(
    WeatherEvent(
        id = null,
        title = title,
        description = description,
        photoUrl = photoUrl,
        capturedAt = capturedAt,
        latitude = latitude,
        longitude = longitude,
        phenomenon = phenomenon,
        validationStatus = validationStatus,
        observedWeatherCode = phenomenon.weatherCodes.first(),
        xpAwarded = xpAwarded,
        userId = owner.id!!
    )
)
```

- [ ] **Step 12: Reset the dev database and run everything**

The new non-nullable columns cannot be added to a table that already has rows under `ddl-auto=update`. Drop the application tables exactly as in Task 2, step 16 — this reset is pre-authorized too:

```bash
docker start skydex-db && sleep 3
docker exec skydex-db psql -U guilherme_becker -d skydex -c \
  'DROP TABLE IF EXISTS weather_events, users CASCADE;'
```

Then:

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS.

- [ ] **Step 13: Commit**

```bash
git add -A src
git commit -m "feat: validate captures against Open-Meteo and award rarity xp

A capture now claims a species from the catalog; the backend checks it
against the hourly record for that place and time and only awards XP
when the two agree. An upstream outage leaves the capture unconfirmed
rather than rejecting it."
```

---

## Presence-proof (Tasks 12b and 12c) — why these exist

Task 12's review established that `CONFIRMED` is forgeable. Both defences Task 12 built — the 90-minute window and the update freeze — guard the **time** axis, which the server already controls because it stamps `capturedAt` itself. The axis the client actually controls is the **coordinates**, and nothing guards it. `GET /api/weather/nearby` accepts arbitrary `lat`/`lon`, so the API hands out the answer key: query any point on Earth, read the phenomenon happening there, POST a claim that matches. Photos are not bound to an uploader either, so the claim can cite any existing photo path — including another user's.

The user decided on 2026-08-09 to close this now rather than accept it as an MVP limitation. What the server can honestly check is "was that weather happening there then", never "was the user there", so the answer is layered:

- **Task 12b — photo provenance.** Pure server-side. A capture must cite a photo *you* uploaded, recently, and not already spent.
- **Task 12c — travel plausibility and mock-location reporting.** Pure server-side for the plausibility half. Kills global storm-chasing, which is the exploit that matters for the rare species.

**Not built, and the user must decide separately: Google Play Integrity.** It is the only true attestation, and the only thing that would make any client-supplied GPS trustworthy. It needs a Google Play Console app listing, a Cloud project, server-side token verification, and it breaks debug builds and the USB-connected-phone loop this project develops on. Do not attempt it inside these tasks.

**Rejected on merit, not cost: EXIF geotag verification.** The system camera frequently omits GPS unless it holds location permission, so it would reject honest users at a high rate, while EXIF is trivially writable by anyone deliberately cheating. Do not revive it without new evidence.

---

### Task 12b: A capture must cite a photo you actually took

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/UploadedPhoto.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/UploadedPhotoRepository.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/PhotoProvenanceService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/PhotoController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/PhotoControllerTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/TestFixtures.kt`

**Interfaces:**
- `models.UploadedPhoto` — `id: UUID?`, `filename: String` (unique, the bare name with no path), `uploaderId: UUID`, `uploadedAt: Instant`, `consumedAt: Instant?`.
- `repositories.UploadedPhotoRepository.findByFilename(filename: String): UploadedPhoto?`
- `services.PhotoProvenanceService(photos)` with `fun claim(photoUrl: String, uploaderId: UUID, now: Instant): UploadedPhoto` — throws `BadUploadException` when the photo is unknown, not the caller's, already spent, or older than `MAX_AGE`; otherwise stamps `consumedAt` and returns the row.
- `POST /api/photos` records an `UploadedPhoto` bound to the caller.
- `POST /api/events` calls `claim(...)` before validating, so a bad photo costs no Open-Meteo call.

Rules:

1. `MAX_AGE` is **30 minutes**. The real flow uploads the photo and creates the capture seconds apart; 30 minutes is generous for a slow network without leaving a stock image usable tomorrow.
2. A photo is **single-use**. `consumedAt` is stamped on the capture that spends it.
3. Failures return **400** with the `ErrorResponse` envelope, via the existing `BadUploadException`.
4. **Unknown photo and someone-else's photo return the SAME message** — `"Photo is not available for this capture"`. Distinguishing them would turn the endpoint into an oracle for which filenames exist. Already-spent and too-old are the caller's own photos, so they get their own messages and leak nothing: `"This photo has already been used for a capture"` and `"Photo has expired; take a new one"`.
5. `photoUrl` becomes **frozen on update**, joining `capturedAt`, the coordinates and the score. Otherwise a capture could be scored with a real photo and then have the evidence swapped afterwards. `CreateWeatherEventRequest` still carries the field and `update` ignores it — the same accepted-and-ignored shape as the coordinates.

- [ ] **Step 1: Write the failing provenance tests first**

In `WeatherEventControllerTest.kt`, cover: a capture citing a photo the caller uploaded seconds ago succeeds (201) and marks the row spent; citing another user's photo returns 400 `"Photo is not available for this capture"`; citing a filename no `UploadedPhoto` row knows returns the same 400 and the same message; citing an already-spent photo returns 400 `"This photo has already been used for a capture"`; citing a photo stamped 31 minutes ago returns 400 `"Photo has expired; take a new one"`; and `PUT` cannot change `photoUrl`.

In `PhotoControllerTest.kt`, cover that a successful upload writes an `UploadedPhoto` row whose `uploaderId` is the caller and whose `filename` matches the returned path's last segment.

**Build the age fixtures relative to `Instant.now()`**, never as literal dates — the standing rule in this plan, and the one that has bitten it most.

The existing `WeatherEventControllerTest` tests all cite `/api/photos/*.jpg` paths that no `UploadedPhoto` row backs, so **they will all start failing**. That is expected and is the point of the red step. Give `TestFixtures.kt` a `persistUploadedPhoto(owner: User, filename: String = "${UUID.randomUUID()}.jpg", uploadedAt: Instant = Instant.now()): UploadedPhoto` helper and route the existing tests through it. Count them before you start so a partial pass is visible; note that at least one builds its request as a **hand-written JSON string** rather than through the DTO, so the compiler will not point at it.

- [ ] **Step 2: Entity, repository, service, wiring**

`UploadedPhoto` uses `@Column(unique = true)` on `filename`. Derive the filename from `photoUrl` with `substringAfterLast('/')` — the `@Pattern` on the request already guarantees the `/api/photos/<name>` shape, so no traversal is reachable here, but do not skip the check that the row exists.

`PhotoProvenanceService.claim` takes `now` as a parameter rather than reading the clock itself, so the age rule is testable without sleeping. `WeatherEventController.create` passes the same `capturedAt` stamp it already reads once.

- [ ] **Step 3: Mutation-probe the two guarantees that are only worth having if they hold**

Delete the ownership check and confirm the another-user's-photo test is what fails. Delete the `consumedAt` stamp and confirm the already-spent test is what fails. Restore both, re-run green, and put the RED and GREEN in your report.

- [ ] **Step 4: Full suite, then commit**

```bash
docker start skydex-db && sleep 3
docker exec skydex-db psql -U guilherme_becker -d skydex -c \
  'DROP TABLE IF EXISTS uploaded_photos, weather_events, users CASCADE;'
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
git add -A src && git commit -m "feat: a capture must cite a fresh photo the caller uploaded"
```

**Known consequence, do not fix here:** unconsumed photos now accumulate as rows as well as files. Backlog item 13 already owns photo cleanup; this task adds a row to the same leak, it does not create a new class of one.

---

### Task 12c: A capture must be somewhere the user could plausibly be

**Files:**
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/CaptureValidationService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/User.kt` (the movement trail)
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/CaptureCommitService.kt` (write the trail in the SAME transaction as the capture — this service was created by Task 12b and already owns that boundary; do not open a second one)
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/service/CaptureValidationServiceTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/WeatherEventControllerTest.kt`

**Interfaces:**
- `User` gains `lastCaptureLatitude: Double?`, `lastCaptureLongitude: Double?`, `lastCaptureAt: Instant?` — the user's movement trail, written on every successful capture.
- `CaptureValidationService.validate(...)` gains `previous: LastKnownPosition?` (a small value type holding the three fields above) and `locationIsMock: Boolean`.

**Do NOT derive the previous position from the user's capture rows** — the obvious `WeatherEventRepository.findFirstByUserIdOrderByCapturedAtDesc(...)` is wrong here, and it took a second look to see why. `DELETE /api/events/{id}` already exists and is unrestricted for the owner (`WeatherEventController.kt:156`), so a trail made of capture rows is a trail the cheater can erase: capture in Tokyo, delete it, and the next capture has nothing implausible to compare against. The trail must outlive the captures, so it lives on the `users` row and is never deleted. Write it inside the same transaction that saves the capture, so a capture and its trail update cannot diverge.
- `CreateWeatherEventRequest` gains `val locationIsMock: Boolean = false` — **defaulted, so it does not break the Android client**, which does not send it until Task 14.

Rules:

1. **Travel plausibility.** If the caller has a recorded previous position, compute the great-circle distance to it (haversine, Earth radius 6371 km) and the elapsed time. If the implied speed exceeds **MAX_SPEED_KMH = 900** (a little above airliner cruise), the capture cannot be `CONFIRMED`: store it `UNCONFIRMED` with 0 XP. A capture is still never *rejected* for this — same principle as an upstream outage. Guard the zero-elapsed-time case: two captures at the same instant in different places are implausible by definition, and dividing by zero must not throw.

   **Write the trail on every capture, confirmed or not.** If only confirmed captures updated it, a cheater could park the trail by making a deliberately-unconfirmed capture somewhere and then jumping. The trail records where the client *claimed* to be, which is exactly the sequence being tested for coherence.

   **State the limit honestly in the KDoc rather than overselling the check.** This bounds the *rate* of long-distance hopping; it does not prevent it. Porto Alegre to Tokyo is roughly 18,500 km, so at 900 km/h the gate is satisfied by waiting about 20 hours — one intercontinental hop per day still passes. It also does nothing against someone faking positions a few kilometres apart. What it kills is the cheap version of the exploit: scanning the globe for rare phenomena and collecting several in an afternoon.
1b. **The trail must never leave the server.** It is deliberately more sensitive than the capture coordinates it derives from: those disappear when a user deletes a capture, and the trail is specifically designed not to. So a user who deletes a capture to erase a location still has that location on their `users` row, and it must never be serialisable.

   `UserResponse.from` maps fields explicitly (`UserDtos.kt`), so the Task 2 DTO boundary already prevents an accidental leak — this is a property to *pin*, not a hole to close. Add a test in the same spirit as Task 2's password-hash test: fetch `GET /api/users/me` after a capture and assert the response body contains neither `lastCaptureLatitude` nor the latitude value itself. Without it, a future task that serialises `User` directly, or swaps the explicit mapper for reflection, would leak a location the user believes they deleted.

2. **Mock location.** If `locationIsMock` is true, the capture cannot be `CONFIRMED`. Be honest in the KDoc that this is **client-asserted and therefore only stops a casual mock-GPS app, not a modified client** — it is worth having because most casual cheating is the former, and it becomes trustworthy only if Play Integrity is ever added.
3. Both checks run **before** the Open-Meteo call where possible, so a rejected capture costs no upstream request.

Tests to add, all with slot times relative to `Instant.now()`: a second capture 10 km away an hour later still confirms; a second capture on another continent minutes later is UNCONFIRMED with 0 XP even though the weather there matches; two captures at the same instant in different places do not throw; a first-ever capture (no recorded position) confirms normally; `locationIsMock = true` is UNCONFIRMED even when everything else agrees; and — the one that pins the design decision above — **deleting the previous capture does not clear the trail**: capture, delete it via `DELETE /api/events/{id}`, then capture implausibly far away and assert it is still UNCONFIRMED. Without that test the trail could be moved back onto capture rows by a later refactor and nothing would notice.

Mutation-probe the speed gate: raise `MAX_SPEED_KMH` to something absurd and confirm the another-continent test is what fails.

Update `ValidationStatus`'s KDoc to state plainly what `CONFIRMED` now means and what it still does not: the claim matched the record for that place and time, the photo was freshly taken by this user, and the position was reachable — but the server still cannot prove the user was physically present, and will not be able to without device attestation.

Commit: `feat: reject captures from implausible positions and mocked locations`

---

### Task 13: The SkyDex collection, XP and levels

- [ ] **Step 0: Turn off Open Session In View — its own commit, before anything else in this task**

Unrelated to the SkyDex collection; it is here because it is a one-line change that has been decided and needs a review gate, and this is the next task through.

`src/main/resources/application.properties` does not set `spring.jpa.open-in-view`, so dev and production run Spring Boot's default of `true` while `src/test/resources/application-test.properties` sets `false`. Tests therefore exercise a different persistence-context lifetime from the one the app ships with — the divergence that let a latent fail-open in Task 12c's locked read reach a re-review instead of a test failure.

Add `spring.jpa.open-in-view=false` to `src/main/resources/application.properties`, with a comment saying why. This is safe here specifically because there are no lazy associations to strand: the three entities reference each other by plain `UUID` columns, not `@ManyToOne`/`@OneToMany`. **Verify that yourself with a grep before making the change** rather than taking this paragraph's word for it — if you find a lazy association, stop and report instead of proceeding.

Leave `CaptureTrailOpenInViewTest` alone: it pins `open-in-view=true` explicitly via `@TestPropertySource`, precisely so the OSIV-on path stays covered after this change.

Commit on its own: `chore: disable open-in-view so dev matches test`

Then run the full suite before starting Step 1, so any fallout from this change is attributed to it and not to the collection work.

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/Levels.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/SkyDexDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/SkyDexService.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/SkyDexController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/WeatherEventRepository.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/WeatherEventDtos.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/LevelsTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/SkyDexControllerTest.kt`

**Interfaces:**
- Consumes: `Phenomenon`, `Rarity` (Task 11); `ValidationStatus`, `WeatherEvent` with XP (Task 12).
- Produces:
  - `domain.levelFor(totalXp: Int): Int` — `1 + floor(sqrt(totalXp / 100))`, clamped so negative input returns 1. Boundaries: 0→1, 99→1, 100→2, 399→2, 400→3, 900→4.
  - `WeatherEventRepository` gains `fun findByUserIdAndValidationStatus(userId: UUID, status: ValidationStatus): List<WeatherEvent>` and `@Query("SELECT COALESCE(SUM(e.xpAwarded), 0) FROM WeatherEvent e WHERE e.userId = :userId") fun totalXpForUser(@Param("userId") userId: UUID): Int`.
  - `dto.SkyDexEntryResponse(phenomenon: String, displayName: String, rarity: String, xpPerCapture: Int, captured: Boolean, captureCount: Int, firstCapturedAt: Instant?)`
  - `dto.SkyDexResponse(level: Int, totalXp: Int, xpToNextLevel: Int, capturedSpecies: Int, totalSpecies: Int, entries: List<SkyDexEntryResponse>)`
  - `services.SkyDexService(events)` with `fun forUser(userId: UUID): SkyDexResponse`
  - `GET /api/skydex` — authenticated, returns `SkyDexResponse` for the caller.
  - `WeatherEventResponse` gains **only** `phenomenonName: String` and `rarity: String`. Task 12 already added `phenomenon`, `validationStatus` and `xpAwarded`; see Step 10, which carries the full resulting shape. Do not re-add or reorder those three.

- [ ] **Step 1: Write the failing level test**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/LevelsTest.kt`:

```kotlin
package com.skydex.api.domain

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class LevelsTest {

    @Test
    fun `a brand new account is level 1`() {
        assertEquals(1, levelFor(0))
        assertEquals(1, levelFor(99))
    }

    @Test
    fun `each level costs quadratically more xp`() {
        assertEquals(2, levelFor(100))
        assertEquals(2, levelFor(399))
        assertEquals(3, levelFor(400))
        assertEquals(4, levelFor(900))
        assertEquals(5, levelFor(1600))
    }

    @Test
    fun `negative xp cannot drop below level 1`() {
        assertEquals(1, levelFor(-50))
    }

    @Test
    fun `reports how much xp the next level still needs`() {
        assertEquals(100, xpToNextLevel(0))
        assertEquals(1, xpToNextLevel(99))
        assertEquals(300, xpToNextLevel(100))
    }
}
```

- [ ] **Step 2: Run and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.LevelsTest"
```

Expected: `Unresolved reference: levelFor`.

- [ ] **Step 3: Write the level maths**

Create `domain/Levels.kt`:

```kotlin
package com.skydex.api.domain

import kotlin.math.floor
import kotlin.math.sqrt

private const val XP_PER_LEVEL_UNIT = 100

/**
 * Level curve: level N starts at (N-1)^2 * 100 XP. Level 2 at 100, level 3 at 400,
 * level 4 at 900. Quadratic so the early levels arrive fast and the later ones mean something.
 */
fun levelFor(totalXp: Int): Int {
    if (totalXp <= 0) return 1
    return 1 + floor(sqrt(totalXp.toDouble() / XP_PER_LEVEL_UNIT)).toInt()
}

/** XP still needed to reach the next level. */
fun xpToNextLevel(totalXp: Int): Int {
    val safeXp = maxOf(totalXp, 0)
    val nextLevel = levelFor(safeXp) + 1
    val threshold = (nextLevel - 1) * (nextLevel - 1) * XP_PER_LEVEL_UNIT
    return threshold - safeXp
}
```

- [ ] **Step 4: Run the level test and confirm it passes**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.LevelsTest"
```

Expected: 4 tests PASS.

- [ ] **Step 5: Write the failing collection test**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/SkyDexControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistEvent
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Test
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.Instant

class SkyDexControllerTest : IntegrationTestBase() {

    @Test
    fun `an empty collection lists every species as uncaptured`() {
        val user = persistUser(email = "rookie@skydex.com")

        mockMvc.perform(get("/api/skydex").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.level").value(1))
            .andExpect(jsonPath("$.totalXp").value(0))
            .andExpect(jsonPath("$.capturedSpecies").value(0))
            .andExpect(jsonPath("$.totalSpecies").value(Phenomenon.entries.size))
            .andExpect(jsonPath("$.entries.length()").value(Phenomenon.entries.size))
            .andExpect(jsonPath("$.entries[?(@.captured == true)]").isEmpty())
    }

    @Test
    fun `counts confirmed captures per species and sums their xp`() {
        val user = persistUser(email = "veteran@skydex.com")
        val earlier = Instant.parse("2026-08-01T10:00:00Z")
        val later = Instant.parse("2026-08-05T10:00:00Z")

        persistEvent(
            user, title = "Storm 1", capturedAt = later,
            phenomenon = Phenomenon.THUNDERSTORM,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.THUNDERSTORM.rarity.xp
        )
        persistEvent(
            user, title = "Storm 2", capturedAt = earlier,
            phenomenon = Phenomenon.THUNDERSTORM,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.THUNDERSTORM.rarity.xp
        )
        persistEvent(
            user, title = "Fog", capturedAt = later,
            phenomenon = Phenomenon.FOG,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.FOG.rarity.xp
        )

        mockMvc.perform(get("/api/skydex").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.totalXp").value(60 + 60 + 25))
            .andExpect(jsonPath("$.level").value(2))
            .andExpect(jsonPath("$.capturedSpecies").value(2))
            .andExpect(jsonPath("$.entries[?(@.phenomenon == 'THUNDERSTORM')].captureCount").value(2))
            .andExpect(
                jsonPath("$.entries[?(@.phenomenon == 'THUNDERSTORM')].firstCapturedAt")
                    .value("2026-08-01T10:00:00Z")
            )
    }

    @Test
    fun `unconfirmed captures do not unlock a species`() {
        val user = persistUser(email = "liar@skydex.com")
        persistEvent(
            user, title = "Alleged hail",
            phenomenon = Phenomenon.HAILSTORM,
            validationStatus = ValidationStatus.UNCONFIRMED,
            xpAwarded = 0
        )

        mockMvc.perform(get("/api/skydex").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.capturedSpecies").value(0))
            .andExpect(jsonPath("$.totalXp").value(0))
            .andExpect(jsonPath("$.entries[?(@.phenomenon == 'HAILSTORM')].captured").value(false))
    }

    @Test
    fun `one user's captures never appear in another user's collection`() {
        val mine = persistUser(email = "mine@skydex.com")
        val theirs = persistUser(email = "theirs@skydex.com")
        persistEvent(
            theirs, phenomenon = Phenomenon.SNOW,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.SNOW.rarity.xp
        )

        mockMvc.perform(get("/api/skydex").header("Authorization", authHeaderFor(mine)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.capturedSpecies").value(0))
            .andExpect(jsonPath("$.totalXp").value(0))
    }
}
```

- [ ] **Step 6: Run and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.SkyDexControllerTest"
```

Expected: all four fail with 404.

- [ ] **Step 7: Extend the repository**

Replace `repositories/WeatherEventRepository.kt`:

```kotlin
package com.skydex.api.repositories

import com.skydex.api.domain.ValidationStatus
import com.skydex.api.models.WeatherEvent
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface WeatherEventRepository : JpaRepository<WeatherEvent, UUID> {

    fun findByUserIdOrderByCapturedAtDesc(userId: UUID): List<WeatherEvent>

    fun findByUserIdAndValidationStatus(
        userId: UUID,
        validationStatus: ValidationStatus
    ): List<WeatherEvent>

    @Query("SELECT COALESCE(SUM(e.xpAwarded), 0) FROM WeatherEvent e WHERE e.userId = :userId")
    fun totalXpForUser(@Param("userId") userId: UUID): Int
}
```

- [ ] **Step 8: Write the collection DTOs**

Create `dto/SkyDexDtos.kt`:

```kotlin
package com.skydex.api.dto

import java.time.Instant

data class SkyDexEntryResponse(
    val phenomenon: String,
    val displayName: String,
    val rarity: String,
    val xpPerCapture: Int,
    val captured: Boolean,
    val captureCount: Int,
    val firstCapturedAt: Instant?
)

data class SkyDexResponse(
    val level: Int,
    val totalXp: Int,
    val xpToNextLevel: Int,
    val capturedSpecies: Int,
    val totalSpecies: Int,
    val entries: List<SkyDexEntryResponse>
)
```

- [ ] **Step 9: Write the collection service and controller**

Create `services/SkyDexService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.domain.levelFor
import com.skydex.api.domain.xpToNextLevel
import com.skydex.api.dto.SkyDexEntryResponse
import com.skydex.api.dto.SkyDexResponse
import com.skydex.api.repositories.WeatherEventRepository
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class SkyDexService(private val events: WeatherEventRepository) {

    /**
     * Builds the full species list every time, marking which ones the user has confirmed.
     * Nothing is denormalised onto the user row, so the collection can never drift out of
     * sync with the captures it is derived from.
     */
    fun forUser(userId: UUID): SkyDexResponse {
        val confirmed = events.findByUserIdAndValidationStatus(userId, ValidationStatus.CONFIRMED)
        val bySpecies = confirmed.groupBy { it.phenomenon }

        val entries = Phenomenon.entries.map { species ->
            val captures = bySpecies[species].orEmpty()
            SkyDexEntryResponse(
                phenomenon = species.name,
                displayName = species.displayName,
                rarity = species.rarity.name,
                xpPerCapture = species.rarity.xp,
                captured = captures.isNotEmpty(),
                captureCount = captures.size,
                firstCapturedAt = captures.minOfOrNull { it.capturedAt }
            )
        }

        val totalXp = events.totalXpForUser(userId)

        return SkyDexResponse(
            level = levelFor(totalXp),
            totalXp = totalXp,
            xpToNextLevel = xpToNextLevel(totalXp),
            capturedSpecies = entries.count { it.captured },
            totalSpecies = entries.size,
            entries = entries
        )
    }
}
```

Create `controllers/SkyDexController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.SkyDexResponse
import com.skydex.api.models.User
import com.skydex.api.services.SkyDexService
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/skydex")
class SkyDexController(private val skyDex: SkyDexService) {

    @GetMapping
    fun mine(@AuthenticationPrincipal currentUser: User): SkyDexResponse =
        skyDex.forUser(currentUser.id!!)
}
```

- [ ] **Step 10: Extend the event response DTO**

**Only TWO fields are new here.** Task 12 already added `phenomenon`, `validationStatus` and `xpAwarded` — the three its own logic produces and its own tests assert on. This step appends the two purely presentational ones, `phenomenonName` and `rarity`, which exist for the collection screen. The full shape afterwards is below; add only the two that are missing, and do not re-add or reorder the three that are already there.

In `dto/WeatherEventDtos.kt`, extend `WeatherEventResponse` and its mapper to:

```kotlin
data class WeatherEventResponse(
    val id: UUID,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: Instant,
    val latitude: Double,
    val longitude: Double,
    val userId: UUID,
    val authorName: String,
    val phenomenon: String,        // Task 12
    val phenomenonName: String,    // new here
    val rarity: String,            // new here
    val validationStatus: String,  // Task 12
    val xpAwarded: Int             // Task 12
) {
    companion object {
        fun from(event: WeatherEvent, author: User, baseUrl: String) = WeatherEventResponse(
            id = event.id!!,
            title = event.title,
            description = event.description,
            photoUrl = event.photoUrl,
            capturedAt = event.capturedAt,
            latitude = event.latitude,
            longitude = event.longitude,
            userId = event.userId,
            authorName = author.name,
            phenomenon = event.phenomenon.name,
            phenomenonName = event.phenomenon.displayName,
            rarity = event.phenomenon.rarity.name,
            validationStatus = event.validationStatus.name,
            xpAwarded = event.xpAwarded
        )
    }
}
```

- [ ] **Step 11: Run the full backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, all suites.

- [ ] **Step 12: Commit**

```bash
git add -A src
git commit -m "feat: skydex collection endpoint with xp and levels

GET /api/skydex returns every species with capture counts derived from
confirmed captures, plus the caller's total XP and level."
```

---

### Task 14: The SkyDex screen

**Files:**
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/SkyDexDto.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/SkyDexRepository.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/skydex/SkyDexViewModel.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/skydex/SkyDexScreen.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/SkyDexApi.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/WeatherEventDto.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ServiceLocator.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/Routes.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/SkyDexNavHost.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/components/AppBottomBar.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureViewModel.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/capture/CaptureScreen.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/skydex/SkyDexViewModelTest.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`

**Interfaces:**
- Consumes: `GET /api/skydex` (Task 13); `UiState` (Task 5); `CaptureGateway` (Task 10).
- Produces:
  - Android `SkyDexEntryResponse(phenomenon: String, displayName: String, rarity: String, xpPerCapture: Int, captured: Boolean, captureCount: Int, firstCapturedAt: String?)` and `SkyDexResponse(level: Int, totalXp: Int, xpToNextLevel: Int, capturedSpecies: Int, totalSpecies: Int, entries: List<SkyDexEntryResponse>)`
  - `SkyDexApi.skyDex(): SkyDexResponse`
  - `data.repository.SkyDexRepository(api)` with `suspend fun collection(): Result<SkyDexResponse>`, implementing `ui.skydex.SkyDexGateway`
  - `ui.skydex.SkyDexViewModel(gateway)` exposing `StateFlow<UiState<SkyDexResponse>>` and `fun refresh()`
  - `Routes.SKYDEX = "skydex"`
  - Android `CreateWeatherEventRequest` gains `phenomenon: String` **and `locationIsMock: Boolean`**; `WeatherEventResponse` gains `phenomenon`, `phenomenonName`, `rarity`, `validationStatus`, `xpAwarded`
  - `Coordinates` gains `isMock: Boolean = false` (defaulted; 21 existing two-arg call sites), sourced in `DeviceLocation.current()` from the platform fix
  - `CaptureUiState` gains `phenomenon: String?` and `CaptureViewModel` gains `fun onPhenomenonSelected(name: String)`; `submit()` refuses with "Escolha qual fenômeno você registrou." when it is null

- [ ] **Step 1: Write the failing tests**

Create `app/src/test/java/com/example/skydex/ui/skydex/SkyDexViewModelTest.kt`:

```kotlin
package com.example.skydex.ui.skydex

import com.example.skydex.data.remote.dto.SkyDexEntryResponse
import com.example.skydex.data.remote.dto.SkyDexResponse
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class SkyDexViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    private val sample = SkyDexResponse(
        level = 2,
        totalXp = 145,
        xpToNextLevel = 255,
        capturedSpecies = 2,
        totalSpecies = 9,
        entries = listOf(
            SkyDexEntryResponse("THUNDERSTORM", "Tempestade com Trovões", "RARE", 60, true, 2, "2026-08-01T10:00:00Z"),
            SkyDexEntryResponse("SNOW", "Neve", "EPIC", 150, false, 0, null)
        )
    )

    @Test
    fun `loads the collection on construction`() = runTest(dispatcher) {
        val viewModel = SkyDexViewModel(FakeSkyDexGateway(Result.success(sample)))
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Success)
        assertEquals(2, (state as UiState.Success).data.level)
        assertEquals(9, state.data.totalSpecies)
    }

    @Test
    fun `surfaces a message when the collection cannot be loaded`() = runTest(dispatcher) {
        val viewModel = SkyDexViewModel(FakeSkyDexGateway(Result.failure(IOException("offline"))))
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Error)
        assertEquals("Não foi possível carregar sua coleção.", (state as UiState.Error).message)
    }
}

class FakeSkyDexGateway(private val result: Result<SkyDexResponse>) : SkyDexGateway {
    override suspend fun collection(): Result<SkyDexResponse> = result
}
```

Append to `app/src/test/java/com/example/skydex/ui/capture/CaptureViewModelTest.kt`:

```kotlin
    @Test
    fun `refuses to submit without choosing a phenomenon`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios")
        viewModel.onPhotoTaken(jpeg())
        viewModel.submit()
        advanceUntilIdle()

        assertEquals("Escolha qual fenômeno você registrou.", viewModel.state.value.errorMessage)
        assertEquals(0, gateway.createdRequests.size)
    }

    @Test
    fun `sends the chosen phenomenon with the capture`() = runTest(dispatcher) {
        val gateway = FakeCaptureGateway()
        val viewModel = CaptureViewModel(gateway) { Coordinates(-30.0346, -51.2177) }

        viewModel.refreshLocation()
        advanceUntilIdle()
        viewModel.onTitleChanged("Tempestade")
        viewModel.onDescriptionChanged("Raios")
        viewModel.onPhenomenonSelected("THUNDERSTORM")
        viewModel.onPhotoTaken(jpeg())
        viewModel.submit()
        advanceUntilIdle()

        assertEquals("THUNDERSTORM", gateway.createdRequests.single().phenomenon)
    }
```

**Existing `CaptureViewModelTest` cases need updating, and the count is a rule rather than a number.** The file holds 15 tests today, grown across Task 10's fix rounds and Task 11 — an earlier draft of this plan said "four", which is stale; do not trust it. The rule: **every test whose `submit()` is expected to get past the guards needs `viewModel.onPhenomenonSelected("THUNDERSTORM")` before it**, and the guard tests must NOT get it.

Explicitly excluded — adding the call to any of these destroys the coverage they exist for:
- `refuses to submit without a title` and `refuses to submit without a description` — assert a guard that fires before phenomenon
- `refuses to submit without a photo` and `refuses to submit without a position` — same, and these are the two the guard ordering below is designed to protect
- `the initial location request is claimed once per view model` and `refreshLocation still works after the initial request was claimed` — never call `submit()`

Order the guards: title/description → photo → position → phenomenon. That order is load-bearing: appending phenomenon **last** is precisely what keeps the four "refuses to submit without X" tests asserting the guard they were written for. Report in your report which tests you touched and which you left alone.

- [ ] **Step 2: Run and watch them fail**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.*"
```

Expected: `Unresolved reference: SkyDexViewModel`, `Unresolved reference: onPhenomenonSelected`.

- [ ] **Step 3: Add the wire DTOs and the API method**

Create `data/remote/dto/SkyDexDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class SkyDexEntryResponse(
    val phenomenon: String,
    val displayName: String,
    val rarity: String,
    val xpPerCapture: Int,
    val captured: Boolean,
    val captureCount: Int,
    val firstCapturedAt: String?
)

data class SkyDexResponse(
    val level: Int,
    val totalXp: Int,
    val xpToNextLevel: Int,
    val capturedSpecies: Int,
    val totalSpecies: Int,
    val entries: List<SkyDexEntryResponse>
)
```

**Wire `locationIsMock`, or Task 12c's mock-location check is dead code.** The server accepts the field with a default of `false`, so omitting it compiles and passes — and silently disables the check for every real user, which is the worst of both worlds: the cost of the feature with none of the benefit.

Carry it on `Coordinates`, **defaulted** — `data class Coordinates(val latitude: Double, val longitude: Double, val isMock: Boolean = false)`. The default is not laziness: there are 21 two-argument construction sites across `CoordinatesTest`, `HomeViewModelTest`, `CaptureViewModelTest` and `HomeScreen`'s preview data, and none of them is about mocking. Only `DeviceLocation.current()` and the one new mocked-position test pass the third argument.

Source it from the platform fix in `DeviceLocation.current()`:

```kotlin
val isMock = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    location.isMock                 // API 31+
} else {
    @Suppress("DEPRECATION")
    location.isFromMockProvider     // API 18-30; minSdk here is 26
}
```

Then pass `coordinates.isMock` through `CaptureViewModel.submit()` into the request. Add a ViewModel test that a mocked position (`Coordinates(-30.0346, -51.2177, isMock = true)`) produces `locationIsMock = true` on the created request. The required field makes *omitting* it a compile error, but nothing makes passing the wrong value one: hardcoding `locationIsMock = false` at the call site compiles, passes every other test, and silently disables the feature. That test is the only thing standing between here and that.

Be honest in the KDoc, matching what Task 12c's service already says: this is client-asserted, so it stops a casual mock-GPS app and not a modified client.

In `data/remote/dto/WeatherEventDto.kt`, extend both capture types:

```kotlin
data class CreateWeatherEventRequest(
    val title: String,
    val description: String,
    val photoUrl: String,
    val latitude: Double,
    val longitude: Double,
    /** Species name from the backend catalog, e.g. "THUNDERSTORM". */
    val phenomenon: String,
    /**
     * Whether the platform reported this fix as coming from a mock provider. Client-asserted, so
     * it stops a casual mock-GPS app and not a modified client — see `CaptureValidationService`.
     *
     * No default here, deliberately. The server defaults it to `false`, which means a client that
     * omits it disables the check for every real user with nothing failing anywhere. Making it
     * required turns "someone forgot to pass it" into a compile error.
     */
    val locationIsMock: Boolean
)

data class WeatherEventResponse(
    val id: String,
    val title: String,
    val description: String,
    val photoUrl: String,
    val capturedAt: String,
    val latitude: Double,
    val longitude: Double,
    val userId: String,
    val authorName: String,
    val phenomenon: String,
    val phenomenonName: String,
    val rarity: String,
    val validationStatus: String,
    val xpAwarded: Int
)
```

Both extensions break every existing construction site, and the compiler will point at each one. You do not need to hunt for them, but here they are so nothing is a surprise — all five `WeatherEventResponse(` sites are fixture or preview data and take literals (`"THUNDERSTORM"`, `"Tempestade com Trovões"`, `"RARE"`, `"CONFIRMED"`, `60`):

- `ui/captures/MyCapturesScreen.kt` — two preview samples
- `ui/captures/MyCapturesViewModelTest.kt`, `ui/capture/CaptureViewModelTest.kt`, `data/repository/CaptureRepositoryTest.kt` — one fixture each

and the two `CreateWeatherEventRequest(` sites are `ui/capture/CaptureViewModel.kt` (production — this is where the real values are wired) and `data/repository/CaptureRepositoryTest.kt` (fixture).

`MyCapturesScreen` is not being asked to *display* the new fields in this task; it only has to compile.

Add to `data/remote/SkyDexApi.kt`:

```kotlin
    @GET("api/skydex")
    suspend fun skyDex(): SkyDexResponse
```

- [ ] **Step 4: Add the gateway, repository and ViewModel**

Create `app/src/main/java/com/example/skydex/ui/skydex/SkyDexGateway.kt`:

```kotlin
package com.example.skydex.ui.skydex

import com.example.skydex.data.remote.dto.SkyDexResponse

interface SkyDexGateway {
    suspend fun collection(): Result<SkyDexResponse>
}
```

Create `app/src/main/java/com/example/skydex/data/repository/SkyDexRepository.kt`:

```kotlin
package com.example.skydex.data.repository

import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.remote.dto.SkyDexResponse
import com.example.skydex.ui.skydex.SkyDexGateway

class SkyDexRepository(private val api: SkyDexApi) : SkyDexGateway {
    override suspend fun collection(): Result<SkyDexResponse> = resultOf { api.skyDex() }
}
```

Register it in `ServiceLocator`:

```kotlin
    val skyDexRepository: SkyDexRepository by lazy { SkyDexRepository(api) }
```

Create `app/src/main/java/com/example/skydex/ui/skydex/SkyDexViewModel.kt`:

```kotlin
package com.example.skydex.ui.skydex

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.SkyDexResponse
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class SkyDexViewModel(private val gateway: SkyDexGateway) : ViewModel() {

    private val _state = MutableStateFlow<UiState<SkyDexResponse>>(UiState.Loading)
    val state: StateFlow<UiState<SkyDexResponse>> = _state.asStateFlow()

    init { refresh() }

    fun refresh() {
        _state.value = UiState.Loading
        viewModelScope.launch {
            gateway.collection()
                .onSuccess { _state.value = UiState.Success(it) }
                .onFailure { _state.value = UiState.Error("Não foi possível carregar sua coleção.") }
        }
    }
}
```

- [ ] **Step 5: Add phenomenon selection to the capture flow**

In `CaptureViewModel.kt`, add `val phenomenon: String? = null` to `CaptureUiState`, add

```kotlin
    fun onPhenomenonSelected(name: String) =
        _state.update { it.copy(phenomenon = name, errorMessage = null) }
```

extend the guard chain (order matters — the tests depend on it):

```kotlin
        val error = when {
            current.title.isBlank() || current.description.isBlank() ->
                "Preencha o título e a descrição."
            current.photoFile == null ->
                "Tire uma foto do fenômeno antes de salvar."
            current.coordinates == null ->
                "Não foi possível obter sua localização. Ative o GPS e tente de novo."
            current.phenomenon == null ->
                "Escolha qual fenômeno você registrou."
            else -> null
        }
```

and pass it in the request:

```kotlin
                phenomenon = current.phenomenon!!
```

In `CaptureScreen.kt`, add a species picker above the title field. It uses a hardcoded list because the catalog is a backend enum with no discovery endpoint in the MVP — the names must match `Phenomenon` exactly:

```kotlin
private val SPECIES = listOf(
    "CLEAR_SKY" to "Céu Limpo",
    "CLOUDS" to "Nublado",
    "FOG" to "Nevoeiro Intenso",
    "DRIZZLE" to "Garoa",
    "RAIN" to "Chuva",
    "RAIN_SHOWER" to "Pancada de Chuva",
    "SNOW" to "Neve",
    "THUNDERSTORM" to "Tempestade com Trovões",
    "HAILSTORM" to "Tempestade Severa com Granizo"
)
```

rendered as a horizontally scrolling `Row` of `FilterChip`s:

```kotlin
        Text("Qual fenômeno?", fontWeight = FontWeight.Bold, fontSize = 15.sp)
        Row(
            modifier = Modifier.fillMaxWidth().horizontalScroll(rememberScrollState()),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            SPECIES.forEach { (name, label) ->
                FilterChip(
                    selected = state.phenomenon == name,
                    onClick = { viewModel.onPhenomenonSelected(name) },
                    label = { Text(label) }
                )
            }
        }
```

with imports `androidx.compose.foundation.horizontalScroll`, `androidx.compose.foundation.rememberScrollState`, `androidx.compose.material3.FilterChip`.

If this list drifts from the backend enum, a capture fails with "Unknown phenomenon". Exposing `GET /api/phenomena` and driving the chips from it is a small post-MVP follow-up worth doing.

- [ ] **Step 6: Run the ViewModel tests and confirm they pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest
```

Expected: every suite green, including the two new `SkyDexViewModelTest` cases and the two new `CaptureViewModelTest` cases.

- [ ] **Step 7: Write the SkyDex screen**

Create `app/src/main/java/com/example/skydex/ui/skydex/SkyDexScreen.kt`:

```kotlin
package com.example.skydex.ui.skydex

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.example.skydex.data.remote.dto.SkyDexEntryResponse
import com.example.skydex.data.remote.dto.SkyDexResponse
import com.example.skydex.ui.common.UiState

@Composable
fun SkyDexScreen(viewModel: SkyDexViewModel, modifier: Modifier = Modifier) {
    val state by viewModel.state.collectAsState()

    Box(
        modifier = modifier.fillMaxSize().background(Color(0xFFF3F4F6)).padding(16.dp)
    ) {
        when (val current = state) {
            is UiState.Loading -> CircularProgressIndicator(
                color = Color(0xFF0284C7),
                modifier = Modifier.align(Alignment.Center)
            )

            is UiState.Error -> Text(
                text = current.message,
                color = Color(0xFFB91C1C),
                textAlign = TextAlign.Center,
                modifier = Modifier.align(Alignment.Center)
            )

            is UiState.Success -> CollectionGrid(current.data)
        }
    }
}

@Composable
private fun CollectionGrid(data: SkyDexResponse) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        verticalArrangement = Arrangement.spacedBy(12.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        item(span = { androidx.compose.foundation.lazy.grid.GridItemSpan(2) }) {
            Column {
                Text("Meu SkyDex", fontSize = 28.sp, fontWeight = FontWeight.Bold, color = Color.Black)
                Spacer(Modifier.height(4.dp))
                Text(
                    "Nível ${data.level} · ${data.totalXp} XP · " +
                        "${data.capturedSpecies}/${data.totalSpecies} espécies",
                    color = Color.Gray,
                    fontSize = 14.sp
                )
                Spacer(Modifier.height(8.dp))
                LinearProgressIndicator(
                    progress = {
                        val span = data.totalXp + data.xpToNextLevel
                        if (span <= 0) 0f else data.totalXp.toFloat() / span
                    },
                    modifier = Modifier.fillMaxWidth().height(6.dp),
                    color = Color(0xFF0284C7)
                )
                Spacer(Modifier.height(4.dp))
                Text(
                    "Faltam ${data.xpToNextLevel} XP para o nível ${data.level + 1}",
                    color = Color.Gray,
                    fontSize = 12.sp
                )
                Spacer(Modifier.height(12.dp))
            }
        }

        items(data.entries) { entry -> SpeciesCard(entry) }
    }
}

@Composable
private fun SpeciesCard(entry: SkyDexEntryResponse) {
    val accent = rarityColor(entry.rarity)

    Card(
        colors = CardDefaults.cardColors(
            containerColor = if (entry.captured) Color.White else Color(0xFFE5E7EB)
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = if (entry.captured) 4.dp else 0.dp),
        shape = RoundedCornerShape(12.dp),
        modifier = Modifier.fillMaxWidth().height(130.dp)
    ) {
        Column(modifier = Modifier.padding(12.dp)) {
            Text(
                text = entry.rarity,
                color = if (entry.captured) accent else Color.Gray,
                fontSize = 10.sp,
                fontWeight = FontWeight.Bold
            )
            Spacer(Modifier.height(6.dp))
            Text(
                text = if (entry.captured) entry.displayName else "???",
                fontWeight = FontWeight.Bold,
                fontSize = 16.sp,
                color = if (entry.captured) Color(0xFF1F2937) else Color.Gray
            )
            Spacer(Modifier.height(6.dp))
            Text(
                text = if (entry.captured) {
                    "${entry.captureCount} registro${if (entry.captureCount == 1) "" else "s"}"
                } else {
                    "${entry.xpPerCapture} XP ao capturar"
                },
                color = Color.Gray,
                fontSize = 12.sp
            )
        }
    }
}

internal fun rarityColor(rarity: String): Color = when (rarity) {
    "LEGENDARY" -> Color(0xFFF59E0B)
    "EPIC" -> Color(0xFF8B5CF6)
    "RARE" -> Color(0xFF3B82F6)
    "UNCOMMON" -> Color(0xFF10B981)
    else -> Color(0xFF6B7280)
}
```

`ui/home/HomeScreen.kt` defines its own private `rarityColor` from Task 11 — delete that copy and import this one instead so the two screens cannot disagree about what "EPIC" looks like.

- [ ] **Step 8: Add the route and the bottom-bar tab**

In `ui/navigation/Routes.kt`:

```kotlin
    const val SKYDEX = "skydex"
```

In `ui/navigation/SkyDexNavHost.kt`, add `Routes.SKYDEX` to `BAR_ROUTES` and add the destination:

```kotlin
            composable(Routes.SKYDEX) {
                val vm: SkyDexViewModel = viewModel { SkyDexViewModel(ServiceLocator.skyDexRepository) }
                SkyDexScreen(viewModel = vm)
            }
```

In `ui/components/AppBottomBar.kt`, add to `items` between Home and Meus Registros:

```kotlin
        BarItem(Routes.SKYDEX, Icons.Default.CatchingPokemon, "SkyDex"),
```

with the import `androidx.compose.material.icons.filled.CatchingPokemon` (present in `material-icons-extended`, already a dependency).

- [ ] **Step 9: Compile and run every Android test**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, all suites green.

- [ ] **Step 10: Verify gamification end to end on the phone**

Start the backend and install the app as in Task 10, step 10. Then:

1. Open Home and note which phenomenon Open-Meteo reports for the current hour at your position.
2. Capture it: pick that species in the chips, take a photo, save.
3. Open the SkyDex tab. **Expected: that species is unlocked, XP has gone up by its rarity value, and the level bar has moved.**
4. Capture again, but pick a species that is *not* what the sky is doing (`SNOW`, unless you are somewhere remarkable).
5. Reopen the SkyDex tab. **Expected: XP unchanged and that species still locked** — the capture is saved under "Meus Registros" but unconfirmed.

Cross-check the backend:

```bash
docker exec skydex-db psql -U guilherme_becker -d skydex \
  -c 'SELECT phenomenon, validation_status, observed_weather_code, xp_awarded FROM weather_events ORDER BY captured_at DESC LIMIT 5;'
```

- [ ] **Step 11: Commit**

```bash
git add -A app
git commit -m "feat: skydex collection screen and phenomenon selection

Captures now claim a species from the catalog, and the SkyDex tab shows
which ones are unlocked along with XP and level progress."
```

---

# Phase 4 — Social

"Sharing with your friends" is in the one-line description of the product and does not exist yet. This phase adds friend requests and a shared feed. It is deliberately last: everything before it is usable solo, so if the MVP has to ship early, it ships without this and still works.

---

### Task 15: Friendships

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/Friendship.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/FriendshipRepository.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/FriendDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/FriendshipService.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/FriendController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/errors/DomainExceptions.kt` (add `BadRequestException`)
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/GlobalExceptionHandler.kt` (map it to 400)
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/FriendControllerTest.kt`

**Interfaces:**
- Consumes: `User`, `UserRepository` (Task 2); `NotFoundException`, `ForbiddenException`, `ConflictException` (Task 3); `GlobalExceptionHandler`, `ErrorResponse` (Task 3).
- Produces:
  - `errors.BadRequestException` mapped to 400 by `GlobalExceptionHandler` — new here; see Step 6.
  - `models.FriendshipStatus` — enum `PENDING`, `ACCEPTED`.
  - `models.Friendship(id: UUID?, requesterId: UUID, addresseeId: UUID, status: FriendshipStatus, createdAt: Instant)`, table `friendships`, unique on `(requester_id, addressee_id)`.
  - `repositories.FriendshipRepository` with `findByRequesterIdAndAddresseeId`, `findByAddresseeIdAndStatus`, and `findAllByUserAndStatus(userId, status)`.
  - `dto.FriendRequestBody(email: String)`, `dto.FriendRequestResponse(id: UUID, requesterId: UUID, requesterName: String, requesterEmail: String, createdAt: Instant)`, `dto.FriendResponse(userId: UUID, name: String, email: String, friendsSince: Instant)`.
  - `services.FriendshipService(friendships, users)` with `fun request(requester: User, email: String): FriendRequestResponse`, `fun incoming(user: User): List<FriendRequestResponse>`, `fun accept(user: User, requestId: UUID): FriendResponse`, `fun decline(user: User, requestId: UUID)`, `fun friends(user: User): List<FriendResponse>`, and **`fun friendIds(userId: UUID): List<UUID>`** — Task 16 depends on that last one.
  - Routes: `POST /api/friends/requests`, `GET /api/friends/requests`, `POST /api/friends/requests/{id}/accept`, `DELETE /api/friends/requests/{id}`, `GET /api/friends`.
  - `IntegrationTestBase` gains an injected `friendshipRepository` and clears it in `@BeforeEach`.

- [ ] **Step 1: Write the failing tests**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/FriendControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.dto.FriendRequestBody
import com.skydex.api.models.FriendshipStatus
import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.util.UUID

class FriendControllerTest : IntegrationTestBase() {

    private fun requestBody(email: String) = objectMapper.writeValueAsString(FriendRequestBody(email))

    @Test
    fun `sends a friend request and lists it for the recipient`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        val bob = persistUser(name = "Bob", email = "bob@skydex.com")

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.requesterName").value("Alice"))

        mockMvc.perform(get("/api/friends/requests").header("Authorization", authHeaderFor(bob)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].requesterEmail").value("alice@skydex.com"))

        // The sender does not see their own outgoing request in the incoming list.
        mockMvc.perform(get("/api/friends/requests").header("Authorization", authHeaderFor(alice)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(0))
    }

    @Test
    fun `accepting a request makes both users friends`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        val bob = persistUser(name = "Bob", email = "bob@skydex.com")

        val created = mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andReturn().response.contentAsString
        val requestId = objectMapper.readTree(created).get("id").asText()

        mockMvc.perform(
            post("/api/friends/requests/{id}/accept", requestId)
                .header("Authorization", authHeaderFor(bob))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.name").value("Alice"))

        mockMvc.perform(get("/api/friends").header("Authorization", authHeaderFor(bob)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].email").value("alice@skydex.com"))

        // Friendship is symmetric: Alice sees Bob too.
        mockMvc.perform(get("/api/friends").header("Authorization", authHeaderFor(alice)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].email").value("bob@skydex.com"))

        assertEquals(
            FriendshipStatus.ACCEPTED,
            friendshipRepository.findById(UUID.fromString(requestId)).orElseThrow().status
        )
    }

    @Test
    fun `only the recipient can accept a request`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        persistUser(name = "Bob", email = "bob@skydex.com")
        val mallory = persistUser(name = "Mallory", email = "mallory@skydex.com")

        val created = mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andReturn().response.contentAsString
        val requestId = objectMapper.readTree(created).get("id").asText()

        mockMvc.perform(
            post("/api/friends/requests/{id}/accept", requestId)
                .header("Authorization", authHeaderFor(mallory))
        )
            .andExpect(status().isForbidden)
            .andExpect(jsonPath("$.error").value("This request was not sent to you"))
    }

    @Test
    fun `refuses to befriend yourself`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("alice@skydex.com"))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.error").value("You cannot add yourself"))
    }

    @Test
    fun `refuses a duplicate request in either direction`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        val bob = persistUser(name = "Bob", email = "bob@skydex.com")

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andExpect(status().isCreated)

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        )
            .andExpect(status().isConflict)
            .andExpect(jsonPath("$.error").value("You already have a pending or accepted request with this user"))

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(bob))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("alice@skydex.com"))
        )
            .andExpect(status().isConflict)
    }

    @Test
    fun `returns 404 for an unknown email`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")

        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("nobody@skydex.com"))
        )
            .andExpect(status().isNotFound)
            .andExpect(jsonPath("$.error").value("No user with that email"))
    }

    @Test
    fun `declining a request removes it from the recipient's list`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        val bob = persistUser(name = "Bob", email = "bob@skydex.com")

        val created = mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andReturn().response.contentAsString
        val requestId = objectMapper.readTree(created).get("id").asText()

        mockMvc.perform(
            delete("/api/friends/requests/{id}", requestId)
                .header("Authorization", authHeaderFor(bob))
        ).andExpect(status().isNoContent)

        mockMvc.perform(get("/api/friends/requests").header("Authorization", authHeaderFor(bob)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(0))

        // Declining frees the pair: Alice can ask again rather than being locked out by the
        // duplicate check forever.
        mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andExpect(status().isCreated)
    }

    @Test
    fun `a stranger cannot decline someone else's request`() {
        val alice = persistUser(name = "Alice", email = "alice@skydex.com")
        persistUser(name = "Bob", email = "bob@skydex.com")
        val mallory = persistUser(name = "Mallory", email = "mallory@skydex.com")

        val created = mockMvc.perform(
            post("/api/friends/requests")
                .header("Authorization", authHeaderFor(alice))
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody("bob@skydex.com"))
        ).andReturn().response.contentAsString
        val requestId = objectMapper.readTree(created).get("id").asText()

        mockMvc.perform(
            delete("/api/friends/requests/{id}", requestId)
                .header("Authorization", authHeaderFor(mallory))
        )
            .andExpect(status().isForbidden)
            .andExpect(jsonPath("$.error").value("This request is not yours"))
    }
}
```

Note the extra import these two need: `org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete`.

These two tests exist because `DELETE /api/friends/requests/{id}` is the one route in this task that **destroys** data, and the draft above shipped it with no coverage at all. It is also more powerful than its name suggests: `decline` deletes the row whatever its status, so the same endpoint an addressee uses to refuse a request is what either party uses to unfriend. That is a reasonable MVP choice, but an uncovered destructive endpoint authorised by a two-branch `if` is not — the second test is what pins that `if`.

- [ ] **Step 2: Run and watch it fail**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.FriendControllerTest"
```

Expected: `Unresolved reference: FriendRequestBody`, `Unresolved reference: friendshipRepository`.

- [ ] **Step 3: Write the entity**

Create `models/Friendship.kt`:

```kotlin
package com.skydex.api.models

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.EnumType
import jakarta.persistence.Enumerated
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table
import jakarta.persistence.UniqueConstraint
import java.time.Instant
import java.util.UUID

enum class FriendshipStatus { PENDING, ACCEPTED }

/**
 * One row per relationship, stored in the direction it was requested. Both users see the
 * friendship once it is ACCEPTED — see FriendshipService.friends.
 */
@Entity
@Table(
    name = "friendships",
    uniqueConstraints = [UniqueConstraint(columnNames = ["requester_id", "addressee_id"])]
)
class Friendship(
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    var id: UUID? = null,

    @Column(name = "requester_id", nullable = false)
    var requesterId: UUID,

    @Column(name = "addressee_id", nullable = false)
    var addresseeId: UUID,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    var status: FriendshipStatus = FriendshipStatus.PENDING,

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()
)
```

- [ ] **Step 4: Write the repository**

Create `repositories/FriendshipRepository.kt`:

```kotlin
package com.skydex.api.repositories

import com.skydex.api.models.Friendship
import com.skydex.api.models.FriendshipStatus
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface FriendshipRepository : JpaRepository<Friendship, UUID> {

    fun findByRequesterIdAndAddresseeId(requesterId: UUID, addresseeId: UUID): Friendship?

    fun findByAddresseeIdAndStatus(addresseeId: UUID, status: FriendshipStatus): List<Friendship>

    @Query(
        "SELECT f FROM Friendship f " +
            "WHERE f.status = :status AND (f.requesterId = :userId OR f.addresseeId = :userId)"
    )
    fun findAllByUserAndStatus(
        @Param("userId") userId: UUID,
        @Param("status") status: FriendshipStatus
    ): List<Friendship>
}
```

- [ ] **Step 5: Write the DTOs**

Create `dto/FriendDtos.kt`:

```kotlin
package com.skydex.api.dto

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import java.time.Instant
import java.util.UUID

data class FriendRequestBody(
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email format is invalid")
    val email: String
)

data class FriendRequestResponse(
    val id: UUID,
    val requesterId: UUID,
    val requesterName: String,
    val requesterEmail: String,
    val createdAt: Instant
)

data class FriendResponse(
    val userId: UUID,
    val name: String,
    val email: String,
    val friendsSince: Instant
)
```

- [ ] **Step 6: Write the service**

Create `services/FriendshipService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.dto.FriendRequestResponse
import com.skydex.api.dto.FriendResponse
import com.skydex.api.errors.ConflictException
import com.skydex.api.errors.ForbiddenException
import com.skydex.api.errors.NotFoundException
import com.skydex.api.models.Friendship
import com.skydex.api.models.FriendshipStatus
import com.skydex.api.models.User
import com.skydex.api.repositories.FriendshipRepository
import com.skydex.api.repositories.UserRepository
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class FriendshipService(
    private val friendships: FriendshipRepository,
    private val users: UserRepository
) {

    fun request(requester: User, email: String): FriendRequestResponse {
        val addressee = users.findByEmail(email.trim())
            ?: throw NotFoundException("No user with that email")

        if (addressee.id == requester.id) throw BadRequestException("You cannot add yourself")

        val existing = friendships.findByRequesterIdAndAddresseeId(requester.id!!, addressee.id!!)
            ?: friendships.findByRequesterIdAndAddresseeId(addressee.id!!, requester.id!!)
        if (existing != null) {
            throw ConflictException("You already have a pending or accepted request with this user")
        }

        val saved = friendships.save(
            Friendship(
                id = null,
                requesterId = requester.id!!,
                addresseeId = addressee.id!!,
                status = FriendshipStatus.PENDING
            )
        )
        return toRequestResponse(saved, requester)
    }

    fun incoming(user: User): List<FriendRequestResponse> =
        friendships.findByAddresseeIdAndStatus(user.id!!, FriendshipStatus.PENDING)
            .mapNotNull { friendship ->
                val requester = users.findById(friendship.requesterId).orElse(null)
                requester?.let { toRequestResponse(friendship, it) }
            }

    fun accept(user: User, requestId: UUID): FriendResponse {
        val friendship = friendships.findById(requestId).orElseThrow {
            NotFoundException("Friend request not found")
        }
        if (friendship.addresseeId != user.id) {
            throw ForbiddenException("This request was not sent to you")
        }

        friendship.status = FriendshipStatus.ACCEPTED
        val saved = friendships.save(friendship)

        val requester = users.findById(saved.requesterId).orElseThrow {
            NotFoundException("Friend request not found")
        }
        return FriendResponse(requester.id!!, requester.name, requester.email, saved.createdAt)
    }

    fun decline(user: User, requestId: UUID) {
        val friendship = friendships.findById(requestId).orElseThrow {
            NotFoundException("Friend request not found")
        }
        if (friendship.addresseeId != user.id && friendship.requesterId != user.id) {
            throw ForbiddenException("This request is not yours")
        }
        friendships.delete(friendship)
    }

    fun friends(user: User): List<FriendResponse> =
        friendships.findAllByUserAndStatus(user.id!!, FriendshipStatus.ACCEPTED)
            .mapNotNull { friendship ->
                val otherId = otherSide(friendship, user.id!!)
                users.findById(otherId).orElse(null)?.let {
                    FriendResponse(it.id!!, it.name, it.email, friendship.createdAt)
                }
            }

    /** Ids of everyone [userId] is actually friends with. Used to scope the feed. */
    fun friendIds(userId: UUID): List<UUID> =
        friendships.findAllByUserAndStatus(userId, FriendshipStatus.ACCEPTED)
            .map { otherSide(it, userId) }

    private fun otherSide(friendship: Friendship, userId: UUID): UUID =
        if (friendship.requesterId == userId) friendship.addresseeId else friendship.requesterId

    private fun toRequestResponse(friendship: Friendship, requester: User) = FriendRequestResponse(
        id = friendship.id!!,
        requesterId = requester.id!!,
        requesterName = requester.name,
        requesterEmail = requester.email,
        createdAt = friendship.createdAt
    )
}
```

**`BadRequestException` does not exist yet — add it, and do not reach for `BadUploadException` instead.** An earlier draft of this plan said to reuse `BadUploadException` for this 400. That was written before Task 12b, and it has aged badly: `BadUploadException` now lives in `services/PhotoStorageService.kt` and its name is accurate *there* — it means "this photo cannot be accepted". Throwing it to reject a self-addressed friend request would mean a friendship rule importing a photo-storage exception, which reads as a mistake to everyone who meets it later. It also happens to be in the same Kotlin package, so it compiles with no import and nothing would stop you.

Two small additions instead. In `errors/DomainExceptions.kt`, beside the three that are already there:

```kotlin
class BadRequestException(message: String) : RuntimeException(message)
```

and in `controllers/GlobalExceptionHandler.kt`, beside the existing handlers:

```kotlin
    @ExceptionHandler(BadRequestException::class)
    fun handleBadRequest(e: BadRequestException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.BAD_REQUEST).body(ErrorResponse(e.message ?: "Bad request"))
```

Leave `BadUploadException` and its handler exactly as they are — migrating the photo paths onto the new type is not part of this task, and touching Tasks 12b/12c's code here would put unreviewed changes in a friendship commit. Two 400-mapped exceptions coexisting is the intended end state for the MVP; note it in the new class's KDoc so the duplication reads as deliberate.

**Two limitations to write down rather than engineer around.** Both are accepted for the MVP; the point is that they appear in the KDoc instead of being discovered later.

1. *The duplicate check is not atomic.* `request` reads both directions and then inserts, and the unique constraint only covers `(requester_id, addressee_id)` — so it cannot stop the mirrored pair. If Alice and Bob request each other at the same instant, both reads miss, both inserts succeed, and the pair now has two ACCEPTED-able rows; `friends()` would then list the same person twice. Guard the symptom, not the race: make `friends()` and `friendIds` return one entry per person — `.distinctBy { it.userId }` and `.distinct()` respectively. Do **not** add locking or a normalised `(low_id, high_id)` column ordering for this; the window is two humans acting within milliseconds of each other, the damage is a duplicated list row, and the feed is unaffected because `IN (:ids)` does not care about repeats. Say all of that in the KDoc so the `distinct` reads as a decision rather than as superstition.

2. *`friendsSince` is the moment the request was **sent**, not accepted.* There is no `accepted_at` column and the MVP does not need one. Since the field name claims otherwise, say so explicitly in `FriendResponse`'s KDoc. Do not add the column, and do not rename the field — Task 17's UI binds to this name.

- [ ] **Step 7: Write the controller**

Create `controllers/FriendController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.FriendRequestBody
import com.skydex.api.dto.FriendRequestResponse
import com.skydex.api.dto.FriendResponse
import com.skydex.api.models.User
import com.skydex.api.services.FriendshipService
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.util.UUID

@RestController
@RequestMapping("/api/friends")
class FriendController(private val friendships: FriendshipService) {

    @PostMapping("/requests")
    fun sendRequest(
        @AuthenticationPrincipal currentUser: User,
        @Valid @RequestBody body: FriendRequestBody
    ): ResponseEntity<FriendRequestResponse> =
        ResponseEntity.status(HttpStatus.CREATED).body(friendships.request(currentUser, body.email))

    @GetMapping("/requests")
    fun incomingRequests(@AuthenticationPrincipal currentUser: User): List<FriendRequestResponse> =
        friendships.incoming(currentUser)

    @PostMapping("/requests/{id}/accept")
    fun acceptRequest(
        @AuthenticationPrincipal currentUser: User,
        @PathVariable id: UUID
    ): FriendResponse = friendships.accept(currentUser, id)

    @DeleteMapping("/requests/{id}")
    fun declineRequest(
        @AuthenticationPrincipal currentUser: User,
        @PathVariable id: UUID
    ): ResponseEntity<Void> {
        friendships.decline(currentUser, id)
        return ResponseEntity.noContent().build()
    }

    @GetMapping
    fun myFriends(@AuthenticationPrincipal currentUser: User): List<FriendResponse> =
        friendships.friends(currentUser)
}
```

- [ ] **Step 8: Let the test base clean up friendships**

In `support/IntegrationTestBase.kt`, add the injected repository:

```kotlin
    @Autowired
    internal lateinit var friendshipRepository: FriendshipRepository
```

and add **one line** to the existing `clearDatabase`:

```kotlin
    @BeforeEach
    fun clearDatabase() {
        friendshipRepository.deleteAll()   // <-- the only new line
        weatherEventRepository.deleteAll()
        uploadedPhotoRepository.deleteAll()
        userRepository.deleteAll()
    }
```

**Add the line; do not retype the method.** An earlier draft of this plan reproduced `clearDatabase` with only three calls, because it was written before Task 12b introduced `uploaded_photos`. Transcribing that draft would have silently dropped `uploadedPhotoRepository.deleteAll()` and leaked photo rows between tests — and since `uploaded_photos.filename` is UNIQUE and photos are single-use, the failures would surface in *other* test classes, intermittently, depending on execution order. Whatever the method looks like when you open the file is the truth; you are inserting one line into it.

Ordering carries no database meaning here: `friendships` references users by plain UUID columns with no foreign key, so nothing constrains the delete order. It goes first only to read consistently with the rest.

with the import `com.skydex.api.repositories.FriendshipRepository`.

- [ ] **Step 9: Run the tests and confirm green**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, including all eight `FriendControllerTest` cases.

- [ ] **Step 10: Commit**

```bash
git add -A src
git commit -m "feat: friend requests and symmetric friendships"
```

---

### Task 16: The friends feed

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/FeedService.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/FeedController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/WeatherEventRepository.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/FeedControllerTest.kt`

**Interfaces:**
- Consumes: `FriendshipService.friendIds` (Task 15); `WeatherEventResponse` (Tasks 2/6/13); `WeatherEventRepository` (Task 13).
- Produces:
  - `WeatherEventRepository` gains `fun findByUserIdInOrderByCapturedAtDesc(userIds: Collection<UUID>, pageable: Pageable): List<WeatherEvent>`.
  - `services.FeedService(events, users, friendships)` with `fun forUser(user: User, page: Int, size: Int): List<WeatherEventResponse>`.
  - `GET /api/feed?page=0&size=20` — authenticated, newest first, own captures plus accepted friends' captures. `size` is clamped to 1..50.

- [ ] **Step 1: Write the failing tests**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/FeedControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.models.Friendship
import com.skydex.api.models.FriendshipStatus
import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistEvent
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Test
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.Instant

class FeedControllerTest : IntegrationTestBase() {

    @Test
    fun `shows my captures and my friends captures, newest first`() {
        val me = persistUser(name = "Me", email = "me@skydex.com")
        val friend = persistUser(name = "Friend", email = "friend@skydex.com")
        friendshipRepository.save(
            Friendship(
                id = null,
                requesterId = me.id!!,
                addresseeId = friend.id!!,
                status = FriendshipStatus.ACCEPTED
            )
        )

        persistEvent(me, title = "Mine (older)", capturedAt = Instant.parse("2026-08-01T10:00:00Z"))
        persistEvent(friend, title = "Theirs (newer)", capturedAt = Instant.parse("2026-08-05T10:00:00Z"))

        mockMvc.perform(get("/api/feed").header("Authorization", authHeaderFor(me)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].title").value("Theirs (newer)"))
            .andExpect(jsonPath("$[0].authorName").value("Friend"))
            .andExpect(jsonPath("$[1].title").value("Mine (older)"))
            .andExpect(jsonPath("$[1].authorName").value("Me"))
    }

    @Test
    fun `never shows captures from strangers`() {
        val me = persistUser(name = "Me", email = "me@skydex.com")
        val stranger = persistUser(name = "Stranger", email = "stranger@skydex.com")
        persistEvent(stranger, title = "Not for you")

        mockMvc.perform(get("/api/feed").header("Authorization", authHeaderFor(me)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(0))
    }

    @Test
    fun `a pending request does not grant feed access`() {
        val me = persistUser(name = "Me", email = "me@skydex.com")
        val pending = persistUser(name = "Pending", email = "pending@skydex.com")
        friendshipRepository.save(
            Friendship(
                id = null,
                requesterId = me.id!!,
                addresseeId = pending.id!!,
                status = FriendshipStatus.PENDING
            )
        )
        persistEvent(pending, title = "Still private")

        mockMvc.perform(get("/api/feed").header("Authorization", authHeaderFor(me)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(0))
    }

    @Test
    fun `paginates`() {
        val me = persistUser(name = "Me", email = "me@skydex.com")
        repeat(5) { i ->
            persistEvent(
                me,
                title = "Capture $i",
                capturedAt = Instant.parse("2026-08-0${i + 1}T10:00:00Z")
            )
        }

        mockMvc.perform(
            get("/api/feed").param("page", "0").param("size", "2")
                .header("Authorization", authHeaderFor(me))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].title").value("Capture 4"))

        mockMvc.perform(
            get("/api/feed").param("page", "2").param("size", "2")
                .header("Authorization", authHeaderFor(me))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))
            .andExpect(jsonPath("$[0].title").value("Capture 0"))
    }

    @Test
    fun `clamps an absurd page size instead of trusting the caller`() {
        val me = persistUser(name = "Me", email = "me@skydex.com")
        repeat(3) { i ->
            persistEvent(
                me,
                title = "Capture $i",
                capturedAt = Instant.parse("2026-08-0${i + 1}T10:00:00Z")
            )
        }

        // Above the ceiling: served, clamped to 50, so all three rows come back.
        mockMvc.perform(
            get("/api/feed").param("size", "10000").header("Authorization", authHeaderFor(me))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(3))

        // Below the floor: PageRequest rejects size 0 with IllegalArgumentException, so an
        // unclamped value here is a 500, not merely an odd response.
        mockMvc.perform(
            get("/api/feed").param("size", "0").header("Authorization", authHeaderFor(me))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(1))

        // Negative page, same reason.
        mockMvc.perform(
            get("/api/feed").param("page", "-1").header("Authorization", authHeaderFor(me))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.length()").value(3))
    }

    @Test
    fun `the feed is not public`() {
        mockMvc.perform(get("/api/feed")).andExpect(status().isUnauthorized)
    }
}
```

The clamp test earns its place: `size` and `page` arrive straight from the query string, and `PageRequest.of` throws `IllegalArgumentException` on `size < 1` or `page < 0`. Without `coerceIn`/`maxOf` those become uncaught 500s from an unauthenticated-looking URL — and with them, nothing else in the suite would notice if a later edit dropped the clamp.

- [ ] **Step 2: Run and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.FeedControllerTest"
```

Expected: **five** fail with 404 — every test that sends a token.

`the feed is not public` is the exception and will pass already, because `SecurityConfig` ends in `anyRequest().authenticated()`, so an unauthenticated request is rejected at the filter chain before routing ever discovers there is no `/api/feed`. That is not a broken test and there is nothing to fix: it guards against a future `permitAll` being added for this path, which is a real risk precisely because it would not break anything else. Note it in your report rather than treating a green test as a failed RED phase.

- [ ] **Step 3: Add the paged query**

Add to `repositories/WeatherEventRepository.kt`:

```kotlin
    fun findByUserIdInOrderByCapturedAtDesc(
        userIds: Collection<UUID>,
        pageable: Pageable
    ): List<WeatherEvent>
```

with the import `org.springframework.data.domain.Pageable`.

- [ ] **Step 4: Write the feed service**

Create `services/FeedService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.dto.WeatherEventResponse
import com.skydex.api.models.User
import com.skydex.api.repositories.UserRepository
import com.skydex.api.repositories.WeatherEventRepository
import org.springframework.beans.factory.annotation.Value
import org.springframework.data.domain.PageRequest
import org.springframework.stereotype.Service

@Service
class FeedService(
    private val events: WeatherEventRepository,
    private val users: UserRepository,
    private val friendships: FriendshipService,
    /**
     * Injected here rather than at a controller, because this service returns finished
     * `WeatherEventResponse`s and there is no mapping step in `FeedController` to hang it on —
     * which is exactly the case `WeatherEventResponse.from`'s KDoc describes when it explains why
     * `baseUrl` is a required parameter. The property is `skydex.photos.public-base-url`, the same
     * one `WeatherEventController` reads.
     */
    @Value("\${skydex.photos.public-base-url}") private val publicBaseUrl: String
) {

    /**
     * The caller's own captures plus those of accepted friends, newest first.
     * Unconfirmed captures are included — the response carries validationStatus so the
     * client can badge them, and hiding a friend's honest miss would be unfriendly.
     */
    fun forUser(user: User, page: Int, size: Int): List<WeatherEventResponse> {
        val visibleAuthors = friendships.friendIds(user.id!!) + user.id!!

        val pageRequest = PageRequest.of(maxOf(page, 0), size.coerceIn(1, MAX_PAGE_SIZE))
        val captures = events.findByUserIdInOrderByCapturedAtDesc(visibleAuthors, pageRequest)

        // One query for every author on the page, rather than one per capture.
        val authorsById = users.findAllById(captures.map { it.userId }.distinct())
            .associateBy { it.id }

        return captures.mapNotNull { capture ->
            authorsById[capture.userId]?.let { WeatherEventResponse.from(capture, it, publicBaseUrl) }
        }
    }

    private companion object {
        const val MAX_PAGE_SIZE = 50
    }
}
```

- [ ] **Step 5: Write the controller**

Create `controllers/FeedController.kt`:

```kotlin
package com.skydex.api.controllers

import com.skydex.api.dto.WeatherEventResponse
import com.skydex.api.models.User
import com.skydex.api.services.FeedService
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/feed")
class FeedController(private val feed: FeedService) {

    @GetMapping
    fun myFeed(
        @AuthenticationPrincipal currentUser: User,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): List<WeatherEventResponse> = feed.forUser(currentUser, page, size)
}
```

- [ ] **Step 6: Run the full backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, every suite.

- [ ] **Step 7: Commit**

```bash
git add -A src
git commit -m "feat: friends feed scoped to accepted friendships"
```

---

### Task 17: The friends and feed screens

**Files:**
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/FriendDto.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/SocialRepository.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/feed/FeedViewModel.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/feed/FeedScreen.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/friends/FriendsViewModel.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/friends/FriendsScreen.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/social/SocialGateway.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/SkyDexApi.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ServiceLocator.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/Routes.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/SkyDexNavHost.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/components/AppBottomBar.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/home/HomeScreen.kt` (new `onOpenMyCaptures` callback — see Step 9)
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/friends/FriendsViewModelTest.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/feed/FeedViewModelTest.kt`

**Interfaces:**
- Consumes: friend and feed routes (Tasks 15, 16); `WeatherEventResponse` (Task 14); `UiState` (Task 5).
- Produces:
  - Android `FriendRequestBody(email: String)`, `FriendRequestResponse(id: String, requesterId: String, requesterName: String, requesterEmail: String, createdAt: String)`, `FriendResponse(userId: String, name: String, email: String, friendsSince: String)`
  - `SkyDexApi.sendFriendRequest`, `incomingFriendRequests`, `acceptFriendRequest`, `declineFriendRequest`, `friends`, `feed`
  - `ui.social.SocialGateway` — interface with the six operations above returning `Result<…>`
  - `data.repository.SocialRepository(api) : SocialGateway`
  - `ui.friends.FriendsUiState(email: String, friends: List<FriendResponse>, requests: List<FriendRequestResponse>, loading: Boolean, message: String?)`
  - `ui.friends.FriendsViewModel(gateway)` with `onEmailChanged`, `sendRequest`, `accept(id)`, `decline(id)`, `refresh`
  - `ui.feed.FeedViewModel(gateway)` exposing `StateFlow<UiState<List<WeatherEventResponse>>>` and `fun refresh()`
  - `Routes.FEED = "feed"`, `Routes.FRIENDS = "friends"`

- [ ] **Step 1: Write the failing tests**

Create `app/src/test/java/com/example/skydex/ui/friends/FriendsViewModelTest.kt`:

```kotlin
package com.example.skydex.ui.friends

import com.example.skydex.data.remote.dto.FriendRequestResponse
import com.example.skydex.data.remote.dto.FriendResponse
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.ui.social.SocialGateway
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Before
import org.junit.Test
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class FriendsViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    @Test
    fun `loads friends and pending requests on construction`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway(
            friends = listOf(FriendResponse("u1", "Alice", "alice@skydex.com", "2026-08-01T10:00:00Z")),
            requests = listOf(
                FriendRequestResponse("r1", "u2", "Bob", "bob@skydex.com", "2026-08-02T10:00:00Z")
            )
        )
        val viewModel = FriendsViewModel(gateway)
        advanceUntilIdle()

        assertEquals(1, viewModel.state.value.friends.size)
        assertEquals("Bob", viewModel.state.value.requests.single().requesterName)
    }

    @Test
    fun `sending a request clears the field and reports success`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway()
        val viewModel = FriendsViewModel(gateway)
        advanceUntilIdle()

        viewModel.onEmailChanged("bob@skydex.com")
        viewModel.sendRequest()
        advanceUntilIdle()

        assertEquals(listOf("bob@skydex.com"), gateway.sentTo)
        assertEquals("", viewModel.state.value.email)
        assertEquals("Convite enviado!", viewModel.state.value.message)
    }

    @Test
    fun `refuses to send a request with a blank email`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway()
        val viewModel = FriendsViewModel(gateway)
        advanceUntilIdle()

        viewModel.sendRequest()
        advanceUntilIdle()

        assertEquals(0, gateway.sentTo.size)
        assertEquals("Digite o e-mail do seu amigo.", viewModel.state.value.message)
    }

    @Test
    fun `reports a failed request without clearing the field`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway(sendResult = Result.failure(IOException("nope")))
        val viewModel = FriendsViewModel(gateway)
        advanceUntilIdle()

        viewModel.onEmailChanged("ghost@skydex.com")
        viewModel.sendRequest()
        advanceUntilIdle()

        assertEquals("ghost@skydex.com", viewModel.state.value.email)
        assertEquals("Não foi possível enviar o convite.", viewModel.state.value.message)
    }

    @Test
    fun `accepting a request reloads the lists`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway(
            requests = listOf(
                FriendRequestResponse("r1", "u2", "Bob", "bob@skydex.com", "2026-08-02T10:00:00Z")
            )
        )
        val viewModel = FriendsViewModel(gateway)
        advanceUntilIdle()

        gateway.friends = listOf(FriendResponse("u2", "Bob", "bob@skydex.com", "2026-08-02T10:00:00Z"))
        gateway.requests = emptyList()
        viewModel.accept("r1")
        advanceUntilIdle()

        assertEquals(listOf("r1"), gateway.accepted)
        assertEquals(1, viewModel.state.value.friends.size)
        assertEquals(0, viewModel.state.value.requests.size)
    }
}
```

Two test classes now share the fake, so it gets its own file rather than being reached across packages. Create `app/src/test/java/com/example/skydex/ui/social/FakeSocialGateway.kt` (package `com.example.skydex.ui.social`) with the class below, and import it from both test files:

```kotlin
class FakeSocialGateway(
    var friends: List<FriendResponse> = emptyList(),
    var requests: List<FriendRequestResponse> = emptyList(),
    private val sendResult: Result<Unit> = Result.success(Unit),
    var feedResult: Result<List<WeatherEventResponse>> = Result.success(emptyList())
) : SocialGateway {

    val sentTo = mutableListOf<String>()
    val accepted = mutableListOf<String>()

    /** Every `(page, size)` pair the ViewModel asked for, in order. */
    val feedCalls = mutableListOf<Pair<Int, Int>>()

    override suspend fun sendRequest(email: String): Result<Unit> {
        if (sendResult.isSuccess) sentTo += email
        return sendResult
    }

    override suspend fun incomingRequests(): Result<List<FriendRequestResponse>> = Result.success(requests)

    override suspend fun accept(requestId: String): Result<Unit> {
        accepted += requestId
        return Result.success(Unit)
    }

    override suspend fun decline(requestId: String): Result<Unit> = Result.success(Unit)

    override suspend fun friends(): Result<List<FriendResponse>> = Result.success(friends)

    override suspend fun feed(page: Int, size: Int): Result<List<WeatherEventResponse>> {
        feedCalls += page to size
        return feedResult
    }
}
```

Also create `app/src/test/java/com/example/skydex/ui/feed/FeedViewModelTest.kt`. `FeedViewModel` is the same shape as Task 14's `SkyDexViewModel`, which is tested; the feed additionally deserves it because it is the only screen that renders **other people's** data, so "renders exactly what the gateway returned" is the property worth pinning:

```kotlin
package com.example.skydex.ui.feed

import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.ui.common.UiState
import com.example.skydex.ui.social.FakeSocialGateway
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class FeedViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    private val capture = WeatherEventResponse(
        id = "e1",
        title = "Tempestade",
        description = "Raios sobre o lago",
        photoUrl = "http://localhost:8080/api/photos/a.jpg",
        capturedAt = "2026-08-05T10:00:00Z",
        latitude = -30.0346,
        longitude = -51.2177,
        userId = "u1",
        authorName = "Amiga",
        phenomenon = "THUNDERSTORM",
        phenomenonName = "Tempestade com Trovões",
        rarity = "RARE",
        validationStatus = "CONFIRMED",
        xpAwarded = 60
    )

    @Test
    fun `loads the feed on construction`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway(feedResult = Result.success(listOf(capture)))
        val viewModel = FeedViewModel(gateway)
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Success)
        assertEquals(listOf(capture), (state as UiState.Success).data)
        assertEquals(listOf(0 to 20), gateway.feedCalls)
    }

    @Test
    fun `surfaces a message when the feed cannot be loaded`() = runTest(dispatcher) {
        val gateway = FakeSocialGateway(feedResult = Result.failure(IOException("offline")))
        val viewModel = FeedViewModel(gateway)
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Error)
        assertEquals("Não foi possível carregar o feed.", (state as UiState.Error).message)
    }
}
```

`assertEquals(listOf(0 to 20), gateway.feedCalls)` is the part that matters beyond the happy path: `page` and `size` are the two arguments the backend clamps, and this is the only place the client's side of that contract is written down.

- [ ] **Step 2: Run and watch it fail**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.friends.*"
```

Expected: `Unresolved reference: FriendsViewModel`, `Unresolved reference: SocialGateway`.

- [ ] **Step 3: Add the wire DTOs and API methods**

Create `data/remote/dto/FriendDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class FriendRequestBody(val email: String)

data class FriendRequestResponse(
    val id: String,
    val requesterId: String,
    val requesterName: String,
    val requesterEmail: String,
    val createdAt: String
)

data class FriendResponse(
    val userId: String,
    val name: String,
    val email: String,
    val friendsSince: String
)
```

Add to `data/remote/SkyDexApi.kt`:

```kotlin
    @POST("api/friends/requests")
    suspend fun sendFriendRequest(@Body body: FriendRequestBody): FriendRequestResponse

    @GET("api/friends/requests")
    suspend fun incomingFriendRequests(): List<FriendRequestResponse>

    @POST("api/friends/requests/{id}/accept")
    suspend fun acceptFriendRequest(@Path("id") id: String): FriendResponse

    @DELETE("api/friends/requests/{id}")
    suspend fun declineFriendRequest(@Path("id") id: String)

    @GET("api/friends")
    suspend fun friends(): List<FriendResponse>

    @GET("api/feed")
    suspend fun feed(@Query("page") page: Int, @Query("size") size: Int): List<WeatherEventResponse>
```

with the matching DTO imports.

- [ ] **Step 4: Add the gateway and repository**

Create `app/src/main/java/com/example/skydex/ui/social/SocialGateway.kt`:

```kotlin
package com.example.skydex.ui.social

import com.example.skydex.data.remote.dto.FriendRequestResponse
import com.example.skydex.data.remote.dto.FriendResponse
import com.example.skydex.data.remote.dto.WeatherEventResponse

interface SocialGateway {
    suspend fun sendRequest(email: String): Result<Unit>
    suspend fun incomingRequests(): Result<List<FriendRequestResponse>>
    suspend fun accept(requestId: String): Result<Unit>
    suspend fun decline(requestId: String): Result<Unit>
    suspend fun friends(): Result<List<FriendResponse>>
    suspend fun feed(page: Int, size: Int): Result<List<WeatherEventResponse>>
}
```

Create `app/src/main/java/com/example/skydex/data/repository/SocialRepository.kt`:

```kotlin
package com.example.skydex.data.repository

import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.remote.dto.FriendRequestBody
import com.example.skydex.data.remote.dto.FriendRequestResponse
import com.example.skydex.data.remote.dto.FriendResponse
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.ui.social.SocialGateway

class SocialRepository(private val api: SkyDexApi) : SocialGateway {

    override suspend fun sendRequest(email: String): Result<Unit> =
        resultOf { api.sendFriendRequest(FriendRequestBody(email.trim())) }.map { }

    override suspend fun incomingRequests(): Result<List<FriendRequestResponse>> =
        resultOf { api.incomingFriendRequests() }

    override suspend fun accept(requestId: String): Result<Unit> =
        resultOf { api.acceptFriendRequest(requestId) }.map { }

    override suspend fun decline(requestId: String): Result<Unit> =
        resultOf { api.declineFriendRequest(requestId) }

    override suspend fun friends(): Result<List<FriendResponse>> =
        resultOf { api.friends() }

    override suspend fun feed(page: Int, size: Int): Result<List<WeatherEventResponse>> =
        resultOf { api.feed(page, size) }
}
```

Register it in `ServiceLocator`:

```kotlin
    val socialRepository: SocialRepository by lazy { SocialRepository(api) }
```

- [ ] **Step 5: Write the friends ViewModel**

Create `app/src/main/java/com/example/skydex/ui/friends/FriendsViewModel.kt`:

```kotlin
package com.example.skydex.ui.friends

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.FriendRequestResponse
import com.example.skydex.data.remote.dto.FriendResponse
import com.example.skydex.ui.social.SocialGateway
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

data class FriendsUiState(
    val email: String = "",
    val friends: List<FriendResponse> = emptyList(),
    val requests: List<FriendRequestResponse> = emptyList(),
    val loading: Boolean = false,
    val message: String? = null
)

class FriendsViewModel(private val social: SocialGateway) : ViewModel() {

    private val _state = MutableStateFlow(FriendsUiState())
    val state: StateFlow<FriendsUiState> = _state.asStateFlow()

    init { refresh() }

    fun onEmailChanged(value: String) = _state.update { it.copy(email = value, message = null) }

    fun refresh() {
        _state.update { it.copy(loading = true) }
        viewModelScope.launch {
            val friends = social.friends().getOrDefault(emptyList())
            val requests = social.incomingRequests().getOrDefault(emptyList())
            _state.update { it.copy(loading = false, friends = friends, requests = requests) }
        }
    }

    fun sendRequest() {
        val email = _state.value.email.trim()
        if (email.isBlank()) {
            _state.update { it.copy(message = "Digite o e-mail do seu amigo.") }
            return
        }

        viewModelScope.launch {
            social.sendRequest(email)
                .onSuccess {
                    _state.update { it.copy(email = "", message = "Convite enviado!") }
                    refresh()
                }
                .onFailure {
                    _state.update { it.copy(message = "Não foi possível enviar o convite.") }
                }
        }
    }

    fun accept(requestId: String) {
        viewModelScope.launch {
            social.accept(requestId)
                .onSuccess { refresh() }
                .onFailure { _state.update { it.copy(message = "Não foi possível aceitar o convite.") } }
        }
    }

    fun decline(requestId: String) {
        viewModelScope.launch {
            social.decline(requestId)
                .onSuccess { refresh() }
                .onFailure { _state.update { it.copy(message = "Não foi possível recusar o convite.") } }
        }
    }
}
```

- [ ] **Step 6: Run the friends tests and confirm they pass**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.friends.*"
```

Expected: 5 tests PASS.

- [ ] **Step 7: Write the feed ViewModel**

Create `app/src/main/java/com/example/skydex/ui/feed/FeedViewModel.kt`:

```kotlin
package com.example.skydex.ui.feed

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.ui.common.UiState
import com.example.skydex.ui.social.SocialGateway
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class FeedViewModel(private val social: SocialGateway) : ViewModel() {

    private val _state = MutableStateFlow<UiState<List<WeatherEventResponse>>>(UiState.Loading)
    val state: StateFlow<UiState<List<WeatherEventResponse>>> = _state.asStateFlow()

    init { refresh() }

    fun refresh() {
        _state.value = UiState.Loading
        viewModelScope.launch {
            social.feed(page = 0, size = 20)
                .onSuccess { _state.value = UiState.Success(it) }
                .onFailure { _state.value = UiState.Error("Não foi possível carregar o feed.") }
        }
    }
}
```

- [ ] **Step 8: Write the two screens**

Create `app/src/main/java/com/example/skydex/ui/feed/FeedScreen.kt`. It is a `LazyColumn` over the feed, reusing the visual language of `MyCapturesScreen`'s card but with the author's name and a validation badge:

```kotlin
package com.example.skydex.ui.feed

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import coil.compose.AsyncImage
import com.example.skydex.data.remote.dto.WeatherEventResponse
import com.example.skydex.ui.common.UiState
import com.example.skydex.ui.skydex.rarityColor

@Composable
fun FeedScreen(viewModel: FeedViewModel, modifier: Modifier = Modifier) {
    val state by viewModel.state.collectAsState()

    Box(modifier = modifier.fillMaxSize().background(Color(0xFFF3F4F6)).padding(16.dp)) {
        when (val current = state) {
            is UiState.Loading -> CircularProgressIndicator(
                color = Color(0xFF0284C7),
                modifier = Modifier.align(Alignment.Center)
            )

            is UiState.Error -> Text(
                current.message,
                color = Color(0xFFB91C1C),
                textAlign = TextAlign.Center,
                modifier = Modifier.align(Alignment.Center)
            )

            is UiState.Success -> if (current.data.isEmpty()) {
                Text(
                    "Nada por aqui ainda. Adicione amigos para ver os registros deles!",
                    color = Color.Gray,
                    textAlign = TextAlign.Center,
                    modifier = Modifier.align(Alignment.Center)
                )
            } else {
                LazyColumn(verticalArrangement = Arrangement.spacedBy(16.dp)) {
                    item {
                        Text("Feed", fontSize = 28.sp, fontWeight = FontWeight.Bold, color = Color.Black)
                    }
                    items(current.data) { capture -> FeedCard(capture) }
                }
            }
        }
    }
}

@Composable
private fun FeedCard(capture: WeatherEventResponse) {
    Card(
        colors = CardDefaults.cardColors(containerColor = Color.White),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp),
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(capture.authorName, fontWeight = FontWeight.Bold, fontSize = 14.sp, color = Color(0xFF0284C7))
            Spacer(Modifier.height(8.dp))

            AsyncImage(
                model = capture.photoUrl,
                contentDescription = capture.title,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxWidth().height(180.dp).background(Color.LightGray)
            )

            Spacer(Modifier.height(12.dp))
            Text(capture.title, fontWeight = FontWeight.Bold, fontSize = 18.sp)
            Spacer(Modifier.height(4.dp))
            Text(capture.description, color = Color.Gray, fontSize = 14.sp)
            Spacer(Modifier.height(8.dp))

            Row(horizontalArrangement = Arrangement.spacedBy(6.dp)) {
                Badge(capture.phenomenonName, rarityColor(capture.rarity))
                if (capture.validationStatus == "CONFIRMED") {
                    Badge("CONFIRMADO +${capture.xpAwarded} XP", Color(0xFF10B981))
                } else {
                    Badge("NÃO CONFIRMADO", Color(0xFF6B7280))
                }
            }
        }
    }
}

@Composable
private fun Badge(text: String, color: Color) {
    Surface(color = color.copy(alpha = 0.12f), shape = MaterialTheme.shapes.small) {
        Text(
            text = text.uppercase(),
            color = color,
            fontSize = 10.sp,
            fontWeight = FontWeight.Bold,
            modifier = Modifier.padding(horizontal = 6.dp, vertical = 3.dp)
        )
    }
}
```

Create `app/src/main/java/com/example/skydex/ui/friends/FriendsScreen.kt` with:
- a title "Amigos";
- an `OutlinedTextField` bound to `state.email` plus a "Convidar" `Button` calling `viewModel::sendRequest`;
- `state.message` rendered below it in grey when it is `"Convite enviado!"` and in `Color(0xFFB91C1C)` otherwise;
- a "Convites recebidos" section listing `state.requests`, each row showing `requesterName` and `requesterEmail` with an "Aceitar" `Button` (`viewModel.accept(it.id)`) and a "Recusar" `TextButton` (`viewModel.decline(it.id)`);
- a "Meus amigos" section listing `state.friends` by `name` and `email`, with the empty-state text "Você ainda não tem amigos no SkyDex."

Use the same `Card` + `Column(padding = 16.dp)` styling as `FeedCard` so the two screens read as one app.

- [ ] **Step 9: Add the routes and tabs**

In `ui/navigation/Routes.kt`:

```kotlin
    const val FEED = "feed"
    const val FRIENDS = "friends"
```

In `ui/navigation/SkyDexNavHost.kt`, add both to `BAR_ROUTES` and add the destinations:

```kotlin
            composable(Routes.FEED) {
                val vm: FeedViewModel = viewModel { FeedViewModel(ServiceLocator.socialRepository) }
                FeedScreen(viewModel = vm)
            }

            composable(Routes.FRIENDS) {
                val vm: FriendsViewModel = viewModel { FriendsViewModel(ServiceLocator.socialRepository) }
                FriendsScreen(viewModel = vm)
            }
```

In `ui/components/AppBottomBar.kt`, the final tab list is five entries:

```kotlin
    val items = listOf(
        BarItem(Routes.NEARBY, Icons.Default.WbSunny, "Eventos Próximos"),
        BarItem(Routes.FEED, Icons.Default.DynamicFeed, "Feed"),
        BarItem(Routes.HOME, Icons.Default.Home, "Início"),
        BarItem(Routes.SKYDEX, Icons.Default.CatchingPokemon, "SkyDex"),
        BarItem(Routes.FRIENDS, Icons.Default.People, "Amigos")
    )
```

with the imports `androidx.compose.material.icons.filled.DynamicFeed` and `androidx.compose.material.icons.filled.People`.

`Routes.MY_CAPTURES` drops off the bottom bar at five tabs, and replacing its entry point is **part of this step, not a follow-up**. Dropping the tab and stopping there leaves "Meus Registros" registered in the graph, listed in `BAR_ROUTES`, and reachable by nothing — the state `Routes.NEARBY` is in right now, which `SkyDexNavHost.kt` marks with the comment *"Registered but unreachable: AppBottomBar still ships no NEARBY tab."* This step is what finally deletes that comment by giving NEARBY a tab; do not create the same situation for `MY_CAPTURES` on the way past. Nothing in the suite tests navigation, so nothing would fail.

Three edits, and the first two files are **not** in this task's file list above — add them:

1. `ui/home/HomeScreen.kt` — take a second navigation callback beside the existing `onStartCapture`:

   ```kotlin
   fun HomeScreen(
       viewModel: HomeViewModel,
       onStartCapture: () -> Unit,
       onOpenMyCaptures: () -> Unit,
       modifier: Modifier = Modifier
   )
   ```

   and render a `TextButton(onClick = onOpenMyCaptures) { Text("Meus Registros") }` near the existing capture affordance. Follow the file's existing `TextButton` styling rather than inventing new colors.

2. `ui/navigation/SkyDexNavHost.kt` — `HomeScreen(...)` is constructed at **two** call sites, once for `Routes.HOME` and once for `Routes.NEARBY`. Both need the new argument, or the build breaks on the one you miss:

   ```kotlin
   onOpenMyCaptures = { navController.navigate(Routes.MY_CAPTURES) }
   ```

3. Keep `Routes.MY_CAPTURES` in `BAR_ROUTES` so the bar stays visible while that screen is open, and leave its `composable(Routes.MY_CAPTURES)` destination exactly as it is.

Also update `HomeScreen`'s `@Preview`, which calls `HomeScreen(...)` and will not compile without the new parameter — pass `{}`.

**This bar layout is superseded by Task 19,** which replaces the `FRIENDS` tab with `PROFILE` and moves both `FRIENDS` and `MY_CAPTURES` under it. Build the five tabs as written here — Task 19 needs `FriendsScreen` to exist before it can relocate the entry point — but do not invest in polishing this arrangement.

- [ ] **Step 10: Compile and run every Android test**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, every suite green.

- [ ] **Step 11: Verify the social loop with two accounts**

Start the backend and install the app. You need two accounts; the second can be driven with curl.

```bash
curl -s -X POST http://localhost:8080/auth/register -H 'Content-Type: application/json' \
  -d '{"name":"Amigo Teste","email":"amigo@skydex.com","password":"super-safe-password"}'
```

On the phone, logged in as your own account:
1. Amigos tab → type `amigo@skydex.com` → Convidar. **Expected: "Convite enviado!"**
2. Accept it as the other user:

```bash
FRIEND_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"amigo@skydex.com","password":"super-safe-password"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')
REQ_ID=$(curl -s http://localhost:8080/api/friends/requests -H "Authorization: Bearer $FRIEND_TOKEN" \
  | sed -E 's/.*"id":"([^"]+)".*/\1/')
curl -s -X POST "http://localhost:8080/api/friends/requests/$REQ_ID/accept" \
  -H "Authorization: Bearer $FRIEND_TOKEN"
```

3. Pull to refresh the Amigos tab. **Expected: "Amigo Teste" appears under "Meus amigos".**
4. Open the Feed tab. **Expected: your own captures are listed with your name and the correct confirmed/unconfirmed badge.**
5. Log out and back in as a third, unrelated account. **Expected: an empty feed** — the visibility scoping holds.

- [ ] **Step 12: Commit**

```bash
git add -A app
git commit -m "feat: friends and feed screens

Completes the social loop: invite by email, accept requests, and see
your friends' confirmed captures in a shared feed."
```

---

# Phase 5 — Achievements and Profile

Badges are the second half of the gamification promise: the SkyDex says *what* you have caught, the badge shelf says *who that makes you*. This phase adds an achievement catalog, a one-to-many badge relation on the user, and the profile screen that shows them off.

It also fixes a structural problem left over from Task 17. Five bottom-bar tabs (Nearby, Feed, Home, SkyDex, Friends) already crowded the bar and pushed `MY_CAPTURES` off it. **Task 19's bar layout supersedes Task 17's:** Profile replaces Friends as the fifth tab, and Profile becomes the home for the three things that are actually "about me" — badges, Meus Registros, and Amigos.

---

### Task 18: Achievements, badges and the profile endpoint

**Files:**
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/domain/Achievement.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/models/UserBadge.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/UserBadgeRepository.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/dto/ProfileDtos.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/BadgeService.kt`
- Create: `SkyDex-backend/src/main/kotlin/com/skydex/api/services/ProfileService.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/repositories/WeatherEventRepository.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/UserController.kt`
- Modify: `SkyDex-backend/src/main/kotlin/com/skydex/api/controllers/WeatherEventController.kt`
- Modify: `SkyDex-backend/src/test/kotlin/com/skydex/api/support/IntegrationTestBase.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/AchievementTest.kt`
- Test: `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/ProfileControllerTest.kt`

**Interfaces:**
- Consumes: `Phenomenon` (Task 11); `ValidationStatus`, `WeatherEvent` with XP (Task 12); `SkyDexService.forUser`, `levelFor`, `xpToNextLevel`, `WeatherEventRepository` (Task 13); `FriendshipService.friendIds` (Task 15); `User`, `UserResponse` (Task 2).
- Produces:
  - `domain.AchievementContext(confirmedCaptures: Int, unconfirmedCaptures: Int, distinctSpecies: Int, totalSpecies: Int, speciesCounts: Map<Phenomenon, Int>, friends: Int)` — everything an achievement rule is allowed to look at.
  - `domain.Achievement` — enum with `val displayName: String`, `val description: String`, and `fun isEarnedBy(context: AchievementContext): Boolean`. Thirteen entries, listed in step 3.
  - `models.UserBadge(id: UUID?, userId: UUID, achievement: Achievement, unlockedAt: Instant)`, table `user_badges`, unique on `(user_id, achievement)`. This is the one-to-many side: one user, many badge rows.
  - `repositories.UserBadgeRepository` with `fun findByUserId(userId: UUID): List<UserBadge>`.
  - `services.BadgeService(badges, events, friendships, skyDex)` with `fun syncFor(user: User): List<UserBadge>` — idempotent; awards any newly earned badge and returns the user's full badge list.
  - `dto.BadgeResponse(achievement: String, displayName: String, description: String, unlocked: Boolean, unlockedAt: Instant?)`
  - `dto.ProfileResponse(user: UserResponse, level: Int, totalXp: Int, xpToNextLevel: Int, confirmedCaptures: Int, totalCaptures: Int, capturedSpecies: Int, totalSpecies: Int, friends: Int, unlockedBadges: Int, totalBadges: Int, badges: List<BadgeResponse>)`
  - `services.ProfileService(users, events, friendships, skyDex, badges)` with `fun forUser(user: User): ProfileResponse`.
  - `WeatherEventRepository` gains `fun countByUserId(userId: UUID): Long` and `fun countByUserIdAndValidationStatus(userId: UUID, validationStatus: ValidationStatus): Long`.
  - `GET /api/users/me/profile` — authenticated, returns `ProfileResponse` with **every** achievement, locked ones included, so the profile doubles as a goal list.
  - `IntegrationTestBase` gains an injected `userBadgeRepository`, cleared in `@BeforeEach`.

Design decisions worth stating up front, so the implementer does not have to re-derive them:

- **The rule lives on the enum, the award lives in a row.** `Achievement` is a pure catalog with a predicate; `UserBadge` records that a specific user crossed a specific threshold at a specific moment. That keeps `unlockedAt` truthful and makes the catalog freely reorderable.
- **`syncFor` is idempotent and safe to call often.** It is called from `POST /api/events` (right after a capture is saved) and from `GET /api/users/me/profile`. Calling it twice awards nothing twice.
- **`POST /api/events` does not change its response shape.** Badges are persisted silently there; the user sees them on the Profile, where Task 19 marks anything unlocked in the last 24 h as new. Returning freshly unlocked badges from the capture response would mean changing `WeatherEventResponse`, which four earlier tasks assert against — a "badge desbloqueado!" toast at capture time is in the post-MVP list instead.
- **A badge is never revoked.** If a rule's input later drops below the threshold (a capture is deleted, a friend is removed), the row stays. Achievements record that something happened, not that it is still true.

- [ ] **Step 1: Write the failing achievement-catalog test**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/domain/AchievementTest.kt`:

```kotlin
package com.skydex.api.domain

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertFalse
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class AchievementTest {

    private fun context(
        confirmed: Int = 0,
        unconfirmed: Int = 0,
        distinctSpecies: Int = 0,
        speciesCounts: Map<Phenomenon, Int> = emptyMap(),
        friends: Int = 0
    ) = AchievementContext(
        confirmedCaptures = confirmed,
        unconfirmedCaptures = unconfirmed,
        distinctSpecies = distinctSpecies,
        totalSpecies = Phenomenon.entries.size,
        speciesCounts = speciesCounts,
        friends = friends
    )

    @Test
    fun `a brand new account has earned nothing`() {
        val earned = Achievement.entries.filter { it.isEarnedBy(context()) }
        assertEquals(emptyList<Achievement>(), earned)
    }

    @Test
    fun `three confirmed captures unlocks the three-capture badge but not the ten`() {
        val ctx = context(confirmed = 3)

        assertTrue(Achievement.FIRST_CAPTURE.isEarnedBy(ctx))
        assertTrue(Achievement.THREE_CAPTURES.isEarnedBy(ctx))
        assertFalse(Achievement.TEN_CAPTURES.isEarnedBy(ctx))
        assertFalse(Achievement.FIFTY_CAPTURES.isEarnedBy(ctx))
    }

    @Test
    fun `capture-count badges are cumulative thresholds`() {
        val ctx = context(confirmed = 50)
        assertTrue(Achievement.FIRST_CAPTURE.isEarnedBy(ctx))
        assertTrue(Achievement.THREE_CAPTURES.isEarnedBy(ctx))
        assertTrue(Achievement.TEN_CAPTURES.isEarnedBy(ctx))
        assertTrue(Achievement.FIFTY_CAPTURES.isEarnedBy(ctx))
    }

    @Test
    fun `species badges depend on how many distinct species are collected`() {
        assertFalse(Achievement.FIVE_SPECIES.isEarnedBy(context(distinctSpecies = 4)))
        assertTrue(Achievement.FIVE_SPECIES.isEarnedBy(context(distinctSpecies = 5)))

        assertFalse(Achievement.ALL_SPECIES.isEarnedBy(context(distinctSpecies = Phenomenon.entries.size - 1)))
        assertTrue(Achievement.ALL_SPECIES.isEarnedBy(context(distinctSpecies = Phenomenon.entries.size)))
    }

    @Test
    fun `species-specific badges require that exact species`() {
        val stormOnly = context(confirmed = 1, speciesCounts = mapOf(Phenomenon.THUNDERSTORM to 1))

        assertTrue(Achievement.THUNDER_CHASER.isEarnedBy(stormOnly))
        assertFalse(Achievement.HAIL_SURVIVOR.isEarnedBy(stormOnly))
        assertFalse(Achievement.SNOW_SEEKER.isEarnedBy(stormOnly))
        assertFalse(Achievement.OBVIOUS_PHOTOGRAPHER.isEarnedBy(stormOnly))
    }

    @Test
    fun `the optimist badge rewards being wrong five times`() {
        assertFalse(Achievement.WEATHER_OPTIMIST.isEarnedBy(context(unconfirmed = 4)))
        assertTrue(Achievement.WEATHER_OPTIMIST.isEarnedBy(context(unconfirmed = 5)))
    }

    @Test
    fun `the network badge needs three friends`() {
        assertFalse(Achievement.WEATHER_NETWORK.isEarnedBy(context(friends = 2)))
        assertTrue(Achievement.WEATHER_NETWORK.isEarnedBy(context(friends = 3)))
    }

    @Test
    fun `every achievement has a name and a description`() {
        Achievement.entries.forEach {
            assert(it.displayName.isNotBlank()) { "$it has no display name" }
            assert(it.description.isNotBlank()) { "$it has no description" }
        }
    }

    @Test
    fun `no two achievements share a display name`() {
        val names = Achievement.entries.map { it.displayName }
        assertEquals(names.size, names.distinct().size, "duplicate badge names: $names")
    }
}
```

- [ ] **Step 2: Run it and watch it fail**

```bash
cd <workspace>/SkyDex-backend
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.AchievementTest"
```

Expected: `Unresolved reference: Achievement`, `Unresolved reference: AchievementContext`.

- [ ] **Step 3: Write the achievement catalog**

Create `domain/Achievement.kt`:

```kotlin
package com.skydex.api.domain

/**
 * Everything an achievement rule is allowed to look at. Passing one immutable snapshot keeps
 * the rules pure and testable — they never reach into a repository.
 */
data class AchievementContext(
    val confirmedCaptures: Int,
    val unconfirmedCaptures: Int,
    val distinctSpecies: Int,
    val totalSpecies: Int,
    val speciesCounts: Map<Phenomenon, Int>,
    val friends: Int
)

/**
 * The badge shelf. Each entry owns its own unlock rule, so adding a badge is one line here
 * and nothing else — BadgeService iterates the catalog and never knows what the rules are.
 *
 * Names and descriptions are user-facing pt-BR copy, per the Global Constraints.
 */
enum class Achievement(
    val displayName: String,
    val description: String,
    private val rule: (AchievementContext) -> Boolean
) {
    FIRST_CAPTURE(
        "Molhou o Dedo",
        "Você apontou a câmera pro céu e deu certo. Uma vez.",
        { it.confirmedCaptures >= 1 }
    ),
    THREE_CAPTURES(
        "Caçador de Nuvem",
        "Três registros confirmados. Já dá pra puxar assunto no elevador.",
        { it.confirmedCaptures >= 3 }
    ),
    TEN_CAPTURES(
        "Meteorologista de Varanda",
        "Dez confirmados. Os vizinhos já perguntam se vai chover.",
        { it.confirmedCaptures >= 10 }
    ),
    FIFTY_CAPTURES(
        "Homem do Tempo",
        "Cinquenta confirmados. A emissora local que se cuide.",
        { it.confirmedCaptures >= 50 }
    ),
    FIVE_SPECIES(
        "Colecionador de Céu",
        "Cinco espécies diferentes no SkyDex. Tá ficando sério.",
        { it.distinctSpecies >= 5 }
    ),
    ALL_SPECIES(
        "Doutor Tempestade",
        "Todas as espécies capturadas. O céu não tem mais segredos pra você.",
        { it.totalSpecies > 0 && it.distinctSpecies >= it.totalSpecies }
    ),
    THUNDER_CHASER(
        "Pé de Raio",
        "Você ficou do lado de fora durante uma tempestade. Por uma foto.",
        { (it.speciesCounts[Phenomenon.THUNDERSTORM] ?: 0) >= 1 }
    ),
    HAIL_SURVIVOR(
        "Sobrevivente do Granizo",
        "Granizo confirmado. Esperamos que o carro esteja bem.",
        { (it.speciesCounts[Phenomenon.HAILSTORM] ?: 0) >= 1 }
    ),
    SNOW_SEEKER(
        "Viu Neve de Verdade",
        "Neve confirmada. Isso aqui não acontece todo dia.",
        { (it.speciesCounts[Phenomenon.SNOW] ?: 0) >= 1 }
    ),
    FOG_WALKER(
        "Perdido na Neblina",
        "Nevoeiro registrado. Presumimos que você achou o caminho de volta.",
        { (it.speciesCounts[Phenomenon.FOG] ?: 0) >= 1 }
    ),
    OBVIOUS_PHOTOGRAPHER(
        "Fotógrafo do Óbvio",
        "Você registrou um céu limpo e o SkyDex confirmou. Corajoso.",
        { (it.speciesCounts[Phenomenon.CLEAR_SKY] ?: 0) >= 1 }
    ),
    WEATHER_OPTIMIST(
        "Otimista Climático",
        "Cinco palpites que o Open-Meteo não confirmou. A esperança é a última que morre.",
        { it.unconfirmedCaptures >= 5 }
    ),
    WEATHER_NETWORK(
        "Rede de Estações",
        "Três amigos no SkyDex. Agora é uma operação.",
        { it.friends >= 3 }
    );

    fun isEarnedBy(context: AchievementContext): Boolean = rule(context)
}
```

Note `OBVIOUS_PHOTOGRAPHER` is genuinely earnable — `CLEAR_SKY` is a real catalog species worth 10 XP, so photographing an empty sky is a valid (if unglamorous) capture.

- [ ] **Step 4: Run the catalog test and confirm it passes**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.domain.AchievementTest"
```

Expected: 9 tests PASS.

- [ ] **Step 5: Write the failing profile test**

Create `SkyDex-backend/src/test/kotlin/com/skydex/api/controller/ProfileControllerTest.kt`:

```kotlin
package com.skydex.api.controller

import com.skydex.api.domain.Achievement
import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.models.Friendship
import com.skydex.api.models.FriendshipStatus
import com.skydex.api.support.IntegrationTestBase
import com.skydex.api.support.authHeaderFor
import com.skydex.api.support.persistEvent
import com.skydex.api.support.persistUser
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

class ProfileControllerTest : IntegrationTestBase() {

    @Test
    fun `a new profile lists every badge as locked`() {
        val user = persistUser(name = "Rookie", email = "rookie@skydex.com")

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.user.name").value("Rookie"))
            .andExpect(jsonPath("$.user.password").doesNotExist())
            .andExpect(jsonPath("$.level").value(1))
            .andExpect(jsonPath("$.totalXp").value(0))
            .andExpect(jsonPath("$.confirmedCaptures").value(0))
            .andExpect(jsonPath("$.friends").value(0))
            .andExpect(jsonPath("$.unlockedBadges").value(0))
            .andExpect(jsonPath("$.totalBadges").value(Achievement.entries.size))
            .andExpect(jsonPath("$.badges.length()").value(Achievement.entries.size))
            .andExpect(jsonPath("$.badges[?(@.unlocked == true)]").isEmpty())
    }

    @Test
    fun `three confirmed captures unlock the first two capture badges`() {
        val user = persistUser(email = "hunter@skydex.com")
        repeat(3) { i ->
            persistEvent(
                user,
                title = "Chuva $i",
                phenomenon = Phenomenon.RAIN,
                validationStatus = ValidationStatus.CONFIRMED,
                xpAwarded = Phenomenon.RAIN.rarity.xp
            )
        }

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.confirmedCaptures").value(3))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'FIRST_CAPTURE')].unlocked").value(true))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'THREE_CAPTURES')].unlocked").value(true))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'TEN_CAPTURES')].unlocked").value(false))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'THREE_CAPTURES')].displayName").value("Caçador de Nuvem"))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'FIRST_CAPTURE')].unlockedAt").exists())
    }

    @Test
    fun `unconfirmed captures do not unlock capture badges but do unlock the optimist`() {
        val user = persistUser(email = "optimist@skydex.com")
        repeat(5) { i ->
            persistEvent(
                user,
                title = "Granizo imaginário $i",
                phenomenon = Phenomenon.HAILSTORM,
                validationStatus = ValidationStatus.UNCONFIRMED,
                xpAwarded = 0
            )
        }

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.confirmedCaptures").value(0))
            .andExpect(jsonPath("$.totalCaptures").value(5))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'FIRST_CAPTURE')].unlocked").value(false))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'HAIL_SURVIVOR')].unlocked").value(false))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'WEATHER_OPTIMIST')].unlocked").value(true))
    }

    @Test
    fun `a species-specific badge unlocks only for that species`() {
        val user = persistUser(email = "chaser@skydex.com")
        persistEvent(
            user,
            phenomenon = Phenomenon.THUNDERSTORM,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.THUNDERSTORM.rarity.xp
        )

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.badges[?(@.achievement == 'THUNDER_CHASER')].unlocked").value(true))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'SNOW_SEEKER')].unlocked").value(false))
    }

    @Test
    fun `friend count feeds the network badge`() {
        val user = persistUser(email = "social@skydex.com")
        repeat(3) { i ->
            val friend = persistUser(email = "friend$i@skydex.com")
            friendshipRepository.save(
                Friendship(
                    id = null,
                    requesterId = user.id!!,
                    addresseeId = friend.id!!,
                    status = FriendshipStatus.ACCEPTED
                )
            )
        }

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.friends").value(3))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'WEATHER_NETWORK')].unlocked").value(true))
    }

    @Test
    fun `awarding is idempotent across repeated reads`() {
        val user = persistUser(email = "repeat@skydex.com")
        persistEvent(
            user,
            phenomenon = Phenomenon.RAIN,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.RAIN.rarity.xp
        )

        repeat(3) {
            mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(user)))
                .andExpect(status().isOk)
                .andExpect(jsonPath("$.unlockedBadges").value(1))
        }

        assertEquals(1, userBadgeRepository.findByUserId(user.id!!).size)
    }

    @Test
    fun `one user's badges never leak into another's profile`() {
        val mine = persistUser(email = "mine@skydex.com")
        val theirs = persistUser(email = "theirs@skydex.com")
        persistEvent(
            theirs,
            phenomenon = Phenomenon.SNOW,
            validationStatus = ValidationStatus.CONFIRMED,
            xpAwarded = Phenomenon.SNOW.rarity.xp
        )

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(theirs)))
            .andExpect(status().isOk)

        mockMvc.perform(get("/api/users/me/profile").header("Authorization", authHeaderFor(mine)))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.unlockedBadges").value(0))
            .andExpect(jsonPath("$.badges[?(@.achievement == 'SNOW_SEEKER')].unlocked").value(false))
    }

    @Test
    fun `refuses an anonymous request`() {
        mockMvc.perform(get("/api/users/me/profile"))
            .andExpect(status().isUnauthorized)
    }
}
```

- [ ] **Step 6: Run it and watch it fail**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test --tests "com.skydex.api.controller.ProfileControllerTest"
```

Expected: `Unresolved reference: userBadgeRepository`, and every request 404s.

- [ ] **Step 7: Write the badge entity and repository**

Create `models/UserBadge.kt`:

```kotlin
package com.skydex.api.models

import com.skydex.api.domain.Achievement
import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.EnumType
import jakarta.persistence.Enumerated
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table
import jakarta.persistence.UniqueConstraint
import java.time.Instant

/**
 * One row per badge a user has unlocked — the many side of user-to-badges. The unique
 * constraint is what makes BadgeService.syncFor safe to call on every capture and every
 * profile read: a second award simply cannot be inserted.
 */
@Entity
@Table(
    name = "user_badges",
    uniqueConstraints = [UniqueConstraint(columnNames = ["user_id", "achievement"])]
)
class UserBadge(
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    var id: java.util.UUID? = null,

    @Column(name = "user_id", nullable = false)
    var userId: java.util.UUID,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 48)
    var achievement: Achievement,

    @Column(name = "unlocked_at", nullable = false)
    var unlockedAt: Instant = Instant.now()
)
```

Create `repositories/UserBadgeRepository.kt`:

```kotlin
package com.skydex.api.repositories

import com.skydex.api.models.UserBadge
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.util.UUID

@Repository
interface UserBadgeRepository : JpaRepository<UserBadge, UUID> {
    fun findByUserId(userId: UUID): List<UserBadge>
}
```

- [ ] **Step 8: Add the capture counts to the event repository**

Add to `repositories/WeatherEventRepository.kt`:

```kotlin
    fun countByUserId(userId: UUID): Long

    fun countByUserIdAndValidationStatus(userId: UUID, validationStatus: ValidationStatus): Long
```

- [ ] **Step 9: Write the badge service**

Create `services/BadgeService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Achievement
import com.skydex.api.domain.AchievementContext
import com.skydex.api.domain.Phenomenon
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.models.User
import com.skydex.api.models.UserBadge
import com.skydex.api.repositories.UserBadgeRepository
import com.skydex.api.repositories.WeatherEventRepository
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class BadgeService(
    private val badges: UserBadgeRepository,
    private val events: WeatherEventRepository,
    private val friendships: FriendshipService
) {

    /**
     * Awards every achievement the user now qualifies for and returns their full badge list.
     * Idempotent: already-held badges are skipped, so this is safe to call after each capture
     * and on every profile read. Badges are never revoked — they record that something
     * happened, not that it is still true.
     */
    fun syncFor(user: User): List<UserBadge> {
        val userId = user.id!!
        val alreadyHeld = badges.findByUserId(userId)
        val heldAchievements = alreadyHeld.map { it.achievement }.toSet()

        val context = contextFor(userId)
        val newlyEarned = Achievement.entries
            .filter { it !in heldAchievements && it.isEarnedBy(context) }
            .map { UserBadge(id = null, userId = userId, achievement = it) }

        if (newlyEarned.isEmpty()) return alreadyHeld

        return try {
            alreadyHeld + badges.saveAll(newlyEarned)
        } catch (e: DataIntegrityViolationException) {
            // Another request for the same user crossed the same threshold at the same moment and
            // inserted first. The unique constraint on (user_id, achievement) is what stops the
            // duplicate row, but it stops it by THROWING — and this call sits after the capture
            // has already been committed, so letting it propagate would answer a successful
            // capture with a 500. Re-read instead: the badge exists, which is the outcome we
            // wanted, and the loser of the race simply reports the winner's row.
            //
            // This is not a swallowed error. It is the standard idempotent-insert recovery, and
            // it is narrow: only this exception, only after a constraint that exists precisely to
            // make the duplicate impossible. Anything else still propagates.
            badges.findByUserId(userId)
        }
    }

    fun badgesFor(userId: UUID): List<UserBadge> = badges.findByUserId(userId)

    private fun contextFor(userId: UUID): AchievementContext {
        val confirmed = events.findByUserIdAndValidationStatus(userId, ValidationStatus.CONFIRMED)
        val speciesCounts = confirmed.groupingBy { it.phenomenon }.eachCount()

        return AchievementContext(
            confirmedCaptures = confirmed.size,
            unconfirmedCaptures = events
                .countByUserIdAndValidationStatus(userId, ValidationStatus.UNCONFIRMED)
                .toInt(),
            distinctSpecies = speciesCounts.size,
            totalSpecies = Phenomenon.entries.size,
            speciesCounts = speciesCounts,
            friends = friendships.friendIds(userId).size
        )
    }
}
```

- [ ] **Step 10: Write the profile DTOs and service**

Create `dto/ProfileDtos.kt`:

```kotlin
package com.skydex.api.dto

import com.skydex.api.domain.Achievement
import com.skydex.api.models.UserBadge
import java.time.Instant

data class BadgeResponse(
    val achievement: String,
    val displayName: String,
    val description: String,
    val unlocked: Boolean,
    val unlockedAt: Instant?
) {
    companion object {
        fun from(achievement: Achievement, badge: UserBadge?) = BadgeResponse(
            achievement = achievement.name,
            displayName = achievement.displayName,
            description = achievement.description,
            unlocked = badge != null,
            unlockedAt = badge?.unlockedAt
        )
    }
}

data class ProfileResponse(
    val user: UserResponse,
    val level: Int,
    val totalXp: Int,
    val xpToNextLevel: Int,
    val confirmedCaptures: Int,
    val totalCaptures: Int,
    val capturedSpecies: Int,
    val totalSpecies: Int,
    val friends: Int,
    val unlockedBadges: Int,
    val totalBadges: Int,
    val badges: List<BadgeResponse>
)
```

Create `services/ProfileService.kt`:

```kotlin
package com.skydex.api.services

import com.skydex.api.domain.Achievement
import com.skydex.api.domain.ValidationStatus
import com.skydex.api.dto.BadgeResponse
import com.skydex.api.dto.ProfileResponse
import com.skydex.api.dto.UserResponse
import com.skydex.api.models.User
import com.skydex.api.repositories.WeatherEventRepository
import org.springframework.stereotype.Service

@Service
class ProfileService(
    private val events: WeatherEventRepository,
    private val friendships: FriendshipService,
    private val skyDex: SkyDexService,
    private val badges: BadgeService
) {

    fun forUser(user: User): ProfileResponse {
        val userId = user.id!!

        // Award anything newly earned before reporting, so the profile is never stale.
        val held = badges.syncFor(user).associateBy { it.achievement }

        // Level, XP and species progress come from SkyDexService so the two screens can
        // never disagree about what level the user is.
        val collection = skyDex.forUser(userId)

        val badgeResponses = Achievement.entries.map { BadgeResponse.from(it, held[it]) }

        return ProfileResponse(
            user = UserResponse.from(user),
            level = collection.level,
            totalXp = collection.totalXp,
            xpToNextLevel = collection.xpToNextLevel,
            confirmedCaptures = events
                .countByUserIdAndValidationStatus(userId, ValidationStatus.CONFIRMED)
                .toInt(),
            totalCaptures = events.countByUserId(userId).toInt(),
            capturedSpecies = collection.capturedSpecies,
            totalSpecies = collection.totalSpecies,
            friends = friendships.friendIds(userId).size,
            unlockedBadges = badgeResponses.count { it.unlocked },
            totalBadges = badgeResponses.size,
            badges = badgeResponses
        )
    }
}
```

- [ ] **Step 11: Expose the endpoint and sync on capture**

In `controllers/UserController.kt`, inject `ProfileService` and add the endpoint:

```kotlin
class UserController(
    private val users: UserRepository,
    private val events: WeatherEventRepository,
    private val profiles: ProfileService
) {

    @GetMapping("/me/profile")
    fun myProfile(@AuthenticationPrincipal currentUser: User): ProfileResponse =
        profiles.forUser(currentUser)
```

with the imports `com.skydex.api.dto.ProfileResponse` and `com.skydex.api.services.ProfileService`.

In `controllers/WeatherEventController.kt`, inject `BadgeService` and call it after the capture is saved, immediately before building the response.

**Add one constructor parameter. Do not retype the constructor.** An earlier draft of this plan showed it as `(events, users, validation, badges)`, which is what it looked like before Tasks 12b, 12c and 13. It now also takes `photoProvenance: PhotoProvenanceService`, `captureCommit: CaptureCommitService` and the `@Value`-injected `publicBaseUrl` — transcribing the old four-parameter version would delete the entire photo-provenance and travel-plausibility wiring. That one will not compile, so it cannot ship silently, but do not waste a cycle discovering it. Append:

```kotlin
    private val badges: BadgeService
```

**Where the call goes, and why it is outside the transaction.** The save no longer happens in the controller: `captureCommit.commit(...)` performs the photo spend, the insert and the movement-trail update as one transaction, holding a `PESSIMISTIC_WRITE` lock on the user's row. Put `badges.syncFor(currentUser)` *after* that call returns, on the line the draft indicates. Do **not** move it inside `CaptureCommitService.commit`: `syncFor` runs several count queries and a `saveAll`, and doing that while holding the user-row lock would extend a lock every one of that user's captures contends for, to buy nothing — a badge is not required to be atomic with the capture that earned it, because `syncFor` is idempotent and the next capture or profile read re-awards it.

The cost of that choice is that a `syncFor` failure surfaces as a 500 on a capture that really was saved. Which brings us to the race below.

```kotlin
        // Persist any badge this capture just unlocked. The response shape is unchanged —
        // the user sees new badges on the Profile screen, which marks recent ones as new.
        badges.syncFor(currentUser)

        return ResponseEntity.status(HttpStatus.CREATED)
            .body(WeatherEventResponse.from(saved, currentUser, publicBaseUrl))
```

with the import `com.skydex.api.services.BadgeService`.

- [ ] **Step 12: Let the test base clean up badges**

In `support/IntegrationTestBase.kt`, add the repository:

```kotlin
    @Autowired
    internal lateinit var userBadgeRepository: UserBadgeRepository
```

and add **one line** to the existing `clearDatabase`, which already has four calls and becomes five:

```kotlin
    @BeforeEach
    fun clearDatabase() {
        userBadgeRepository.deleteAll()     // <-- the only new line
        friendshipRepository.deleteAll()
        weatherEventRepository.deleteAll()
        uploadedPhotoRepository.deleteAll()
        userRepository.deleteAll()
    }
```

**Add the line; do not retype the method.** An earlier draft of this plan reproduced `clearDatabase` without `uploadedPhotoRepository.deleteAll()`, because it was written before Task 12b introduced `uploaded_photos`. Transcribing that draft would leak photo rows between tests, and since `uploaded_photos.filename` is UNIQUE and photos are single-use, the failures would appear in *other* test classes, intermittently, depending on execution order. Task 15 hit this exact block and the line was rescued there; it is still the trap. Whatever the method looks like when you open the file is the truth.

Deletion order carries no database meaning — none of these tables has a foreign key to another, they reference each other by plain UUID columns.

with the import `com.skydex.api.repositories.UserBadgeRepository`.

- [ ] **Step 13: Run the full backend suite**

```bash
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test
```

Expected: PASS, including all 8 `ProfileControllerTest` cases and the 9 `AchievementTest` cases.

`user_badges` is a brand-new table with no existing rows, so `ddl-auto=update` creates it cleanly — no dev-database reset needed for this task.

- [ ] **Step 14: Verify against a running server**

```bash
docker start skydex-db && sleep 3
JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew bootRun &
sleep 25

TOKEN=$(curl -s -X POST http://localhost:8080/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"upload@skydex.com","password":"super-safe-password"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

curl -s http://localhost:8080/api/users/me/profile -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected: a `badges` array of 13 entries, `totalBadges: 13`, and `unlocked: true` on whichever ones your existing captures have earned. Stop the server afterwards.

- [ ] **Step 15: Commit**

```bash
git add -A src
git commit -m "feat: achievement badges and the user profile endpoint

Adds a one-to-many user-to-badges relation driven by a pure Achievement
catalog, and GET /api/users/me/profile returning level, capture counts
and every badge with its unlock state."
```

---

### Task 19: The profile screen with the badge shelf

**Files:**
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/dto/ProfileDto.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/profile/ProfileGateway.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/repository/ProfileRepository.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/profile/ProfileViewModel.kt`
- Create: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/profile/ProfileScreen.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/data/remote/SkyDexApi.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ServiceLocator.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/Routes.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/navigation/SkyDexNavHost.kt`
- Modify: `SkyDex---frontend/app/src/main/java/com/example/skydex/ui/components/AppBottomBar.kt`
- Test: `SkyDex---frontend/app/src/test/java/com/example/skydex/ui/profile/ProfileViewModelTest.kt`

**Interfaces:**
- Consumes: `GET /api/users/me/profile` (Task 18); `UiState` (Task 5); `AuthRepository.logout` (Task 4); `rarityColor` (Task 14); `Routes.MY_CAPTURES` (Task 5), `Routes.FRIENDS` (Task 17).
- Produces:
  - Android `BadgeResponse(achievement: String, displayName: String, description: String, unlocked: Boolean, unlockedAt: String?)`, `UserSummary(id: String, name: String, email: String, joinedAt: String)`, `ProfileResponse(user: UserSummary, level: Int, totalXp: Int, xpToNextLevel: Int, confirmedCaptures: Int, totalCaptures: Int, capturedSpecies: Int, totalSpecies: Int, friends: Int, unlockedBadges: Int, totalBadges: Int, badges: List<BadgeResponse>)`
  - `SkyDexApi.profile(): ProfileResponse`
  - `ui.profile.ProfileGateway` — `suspend fun profile(): Result<ProfileResponse>`
  - `data.repository.ProfileRepository(api) : ProfileGateway`
  - `ui.profile.ProfileViewModel(gateway, onLogout: suspend () -> Unit)` exposing `StateFlow<UiState<ProfileResponse>>`, `fun refresh()`, `fun logout()`
  - `Routes.PROFILE = "profile"`
  - **Final bottom bar (supersedes Task 17):** `NEARBY`, `FEED`, `HOME`, `SKYDEX`, `PROFILE`. `FRIENDS` and `MY_CAPTURES` stay registered destinations, reached from Profile, and stay in `BAR_ROUTES` so the bar remains visible on them.

- [ ] **Step 1: Write the failing ViewModel test**

Create `app/src/test/java/com/example/skydex/ui/profile/ProfileViewModelTest.kt`:

```kotlin
package com.example.skydex.ui.profile

import com.example.skydex.data.remote.dto.BadgeResponse
import com.example.skydex.data.remote.dto.ProfileResponse
import com.example.skydex.data.remote.dto.UserSummary
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import java.io.IOException

@OptIn(ExperimentalCoroutinesApi::class)
class ProfileViewModelTest {

    private val dispatcher = StandardTestDispatcher()

    @Before fun setUp() = Dispatchers.setMain(dispatcher)
    @After fun tearDown() = Dispatchers.resetMain()

    private val sample = ProfileResponse(
        user = UserSummary("u1", "Becker", "becker@skydex.com", "2026-07-01T10:00:00Z"),
        level = 2,
        totalXp = 145,
        xpToNextLevel = 255,
        confirmedCaptures = 4,
        totalCaptures = 6,
        capturedSpecies = 2,
        totalSpecies = 9,
        friends = 1,
        unlockedBadges = 2,
        totalBadges = 13,
        badges = listOf(
            BadgeResponse("FIRST_CAPTURE", "Molhou o Dedo", "Uma vez.", true, "2026-08-01T10:00:00Z"),
            BadgeResponse("THREE_CAPTURES", "Caçador de Nuvem", "Três.", true, "2026-08-03T10:00:00Z"),
            BadgeResponse("TEN_CAPTURES", "Meteorologista de Varanda", "Dez.", false, null)
        )
    )

    @Test
    fun `loads the profile on construction`() = runTest(dispatcher) {
        val viewModel = ProfileViewModel(FakeProfileGateway(Result.success(sample))) {}
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Success)
        assertEquals(2, (state as UiState.Success).data.level)
        assertEquals(13, state.data.totalBadges)
        assertEquals(2, state.data.badges.count { it.unlocked })
    }

    @Test
    fun `surfaces a message when the profile cannot be loaded`() = runTest(dispatcher) {
        val viewModel = ProfileViewModel(FakeProfileGateway(Result.failure(IOException("offline")))) {}
        advanceUntilIdle()

        val state = viewModel.state.value
        assertTrue(state is UiState.Error)
        assertEquals("Não foi possível carregar seu perfil.", (state as UiState.Error).message)
    }

    @Test
    fun `logging out invokes the logout action`() = runTest(dispatcher) {
        var loggedOut = false
        val viewModel = ProfileViewModel(FakeProfileGateway(Result.success(sample))) { loggedOut = true }
        advanceUntilIdle()

        viewModel.logout()
        advanceUntilIdle()

        assertTrue(loggedOut)
    }
}

class FakeProfileGateway(private val result: Result<ProfileResponse>) : ProfileGateway {
    override suspend fun profile(): Result<ProfileResponse> = result
}
```

- [ ] **Step 2: Run it and watch it fail**

```bash
cd <workspace>/SkyDex---frontend
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.profile.*"
```

Expected: `Unresolved reference: ProfileViewModel`, `Unresolved reference: ProfileGateway`.

- [ ] **Step 3: Add the wire DTOs and the API method**

Create `data/remote/dto/ProfileDto.kt`:

```kotlin
package com.example.skydex.data.remote.dto

data class UserSummary(
    val id: String,
    val name: String,
    val email: String,
    val joinedAt: String
)

data class BadgeResponse(
    /** Enum name, e.g. "THREE_CAPTURES" — stable identifier for the achievement. */
    val achievement: String,
    val displayName: String,
    val description: String,
    val unlocked: Boolean,
    /** ISO-8601 instant, or null while the badge is still locked. */
    val unlockedAt: String?
)

data class ProfileResponse(
    val user: UserSummary,
    val level: Int,
    val totalXp: Int,
    val xpToNextLevel: Int,
    val confirmedCaptures: Int,
    val totalCaptures: Int,
    val capturedSpecies: Int,
    val totalSpecies: Int,
    val friends: Int,
    val unlockedBadges: Int,
    val totalBadges: Int,
    val badges: List<BadgeResponse>
)
```

Add to `data/remote/SkyDexApi.kt`:

```kotlin
    @GET("api/users/me/profile")
    suspend fun profile(): ProfileResponse
```

with the import `com.example.skydex.data.remote.dto.ProfileResponse`.

- [ ] **Step 4: Add the gateway, repository and ViewModel**

Create `app/src/main/java/com/example/skydex/ui/profile/ProfileGateway.kt`:

```kotlin
package com.example.skydex.ui.profile

import com.example.skydex.data.remote.dto.ProfileResponse

interface ProfileGateway {
    suspend fun profile(): Result<ProfileResponse>
}
```

Create `app/src/main/java/com/example/skydex/data/repository/ProfileRepository.kt`:

```kotlin
package com.example.skydex.data.repository

import com.example.skydex.data.remote.SkyDexApi
import com.example.skydex.data.remote.dto.ProfileResponse
import com.example.skydex.ui.profile.ProfileGateway

class ProfileRepository(private val api: SkyDexApi) : ProfileGateway {
    override suspend fun profile(): Result<ProfileResponse> = resultOf { api.profile() }
}
```

Register it in `ServiceLocator`:

```kotlin
    val profileRepository: ProfileRepository by lazy { ProfileRepository(api) }
```

Create `app/src/main/java/com/example/skydex/ui/profile/ProfileViewModel.kt`:

```kotlin
package com.example.skydex.ui.profile

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.skydex.data.remote.dto.ProfileResponse
import com.example.skydex.ui.common.UiState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class ProfileViewModel(
    private val gateway: ProfileGateway,
    private val onLogout: suspend () -> Unit
) : ViewModel() {

    private val _state = MutableStateFlow<UiState<ProfileResponse>>(UiState.Loading)
    val state: StateFlow<UiState<ProfileResponse>> = _state.asStateFlow()

    init { refresh() }

    fun refresh() {
        _state.value = UiState.Loading
        viewModelScope.launch {
            gateway.profile()
                .onSuccess { _state.value = UiState.Success(it) }
                .onFailure { _state.value = UiState.Error("Não foi possível carregar seu perfil.") }
        }
    }

    fun logout() {
        viewModelScope.launch { onLogout() }
    }
}
```

- [ ] **Step 5: Run the ViewModel test and confirm it passes**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest --tests "com.example.skydex.ui.profile.*"
```

Expected: 3 tests PASS.

- [ ] **Step 6: Write the profile screen**

Create `app/src/main/java/com/example/skydex/ui/profile/ProfileScreen.kt`:

```kotlin
package com.example.skydex.ui.profile

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.EmojiEvents
import androidx.compose.material.icons.filled.Lock
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.Icon
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.example.skydex.data.remote.dto.BadgeResponse
import com.example.skydex.data.remote.dto.ProfileResponse
import com.example.skydex.ui.common.UiState
import java.time.Duration
import java.time.Instant

@Composable
fun ProfileScreen(
    viewModel: ProfileViewModel,
    onOpenMyCaptures: () -> Unit,
    onOpenFriends: () -> Unit,
    onLoggedOut: () -> Unit,
    modifier: Modifier = Modifier
) {
    val state by viewModel.state.collectAsState()

    Box(modifier = modifier.fillMaxSize().background(Color(0xFFF3F4F6)).padding(16.dp)) {
        when (val current = state) {
            is UiState.Loading -> CircularProgressIndicator(
                color = Color(0xFF0284C7),
                modifier = Modifier.align(Alignment.Center)
            )

            is UiState.Error -> Column(
                modifier = Modifier.align(Alignment.Center),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(current.message, color = Color(0xFFB91C1C), textAlign = TextAlign.Center)
                Spacer(Modifier.height(12.dp))
                OutlinedButton(onClick = viewModel::refresh) { Text("Tentar de novo") }
            }

            is UiState.Success -> ProfileBody(
                profile = current.data,
                onOpenMyCaptures = onOpenMyCaptures,
                onOpenFriends = onOpenFriends,
                onLogout = {
                    viewModel.logout()
                    onLoggedOut()
                }
            )
        }
    }
}

@Composable
private fun ProfileBody(
    profile: ProfileResponse,
    onOpenMyCaptures: () -> Unit,
    onOpenFriends: () -> Unit,
    onLogout: () -> Unit
) {
    LazyColumn(verticalArrangement = Arrangement.spacedBy(16.dp)) {
        item { IdentityCard(profile) }
        item { StatsRow(profile, onOpenMyCaptures, onOpenFriends) }

        item {
            Text(
                "Conquistas  ${profile.unlockedBadges}/${profile.totalBadges}",
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold,
                color = Color(0xFF1F2937)
            )
        }

        // Unlocked badges first so the shelf leads with what the user actually earned.
        items(profile.badges.sortedByDescending { it.unlocked }) { badge -> BadgeRow(badge) }

        item {
            TextButton(onClick = onLogout, modifier = Modifier.fillMaxWidth()) {
                Text("Sair da conta", color = Color(0xFFB91C1C))
            }
        }
    }
}

@Composable
private fun IdentityCard(profile: ProfileResponse) {
    Card(
        colors = CardDefaults.cardColors(containerColor = Color(0xFF0284C7)),
        shape = RoundedCornerShape(16.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 8.dp),
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(modifier = Modifier.padding(20.dp)) {
            Text(profile.user.name, color = Color.White, fontSize = 24.sp, fontWeight = FontWeight.ExtraBold)
            Text(profile.user.email, color = Color.White.copy(alpha = 0.8f), fontSize = 13.sp)

            Spacer(Modifier.height(16.dp))

            Text(
                "Nível ${profile.level} · ${profile.totalXp} XP",
                color = Color.White,
                fontWeight = FontWeight.Bold,
                fontSize = 16.sp
            )
            Spacer(Modifier.height(8.dp))
            LinearProgressIndicator(
                progress = {
                    val span = profile.totalXp + profile.xpToNextLevel
                    if (span <= 0) 0f else profile.totalXp.toFloat() / span
                },
                modifier = Modifier.fillMaxWidth().height(6.dp),
                color = Color.White,
                trackColor = Color.White.copy(alpha = 0.3f)
            )
            Spacer(Modifier.height(4.dp))
            Text(
                "Faltam ${profile.xpToNextLevel} XP para o nível ${profile.level + 1}",
                color = Color.White.copy(alpha = 0.8f),
                fontSize = 12.sp
            )
        }
    }
}

@Composable
private fun StatsRow(
    profile: ProfileResponse,
    onOpenMyCaptures: () -> Unit,
    onOpenFriends: () -> Unit
) {
    Row(horizontalArrangement = Arrangement.spacedBy(12.dp), modifier = Modifier.fillMaxWidth()) {
        StatTile(
            value = "${profile.confirmedCaptures}",
            label = "confirmados",
            hint = "de ${profile.totalCaptures}",
            modifier = Modifier.weight(1f),
            onClick = onOpenMyCaptures
        )
        StatTile(
            value = "${profile.capturedSpecies}/${profile.totalSpecies}",
            label = "espécies",
            hint = "no SkyDex",
            modifier = Modifier.weight(1f),
            onClick = null
        )
        StatTile(
            value = "${profile.friends}",
            label = "amigos",
            hint = "ver todos",
            modifier = Modifier.weight(1f),
            onClick = onOpenFriends
        )
    }
}

@Composable
private fun StatTile(
    value: String,
    label: String,
    hint: String,
    modifier: Modifier = Modifier,
    onClick: (() -> Unit)?
) {
    Card(
        colors = CardDefaults.cardColors(containerColor = Color.White),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp),
        shape = RoundedCornerShape(12.dp),
        modifier = modifier
    ) {
        Column(
            modifier = Modifier.padding(12.dp).fillMaxWidth(),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(value, fontWeight = FontWeight.ExtraBold, fontSize = 20.sp, color = Color(0xFF1F2937))
            Text(label, fontSize = 12.sp, color = Color.Gray)
            if (onClick != null) {
                TextButton(onClick = onClick) {
                    Text(hint, fontSize = 11.sp, color = Color(0xFF0284C7))
                }
            } else {
                Text(hint, fontSize = 11.sp, color = Color.Gray)
            }
        }
    }
}

@Composable
private fun BadgeRow(badge: BadgeResponse) {
    val accent = if (badge.unlocked) Color(0xFFF59E0B) else Color(0xFF9CA3AF)

    Card(
        colors = CardDefaults.cardColors(
            containerColor = if (badge.unlocked) Color.White else Color(0xFFE5E7EB)
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = if (badge.unlocked) 3.dp else 0.dp),
        shape = RoundedCornerShape(12.dp),
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            modifier = Modifier.padding(14.dp).fillMaxWidth(),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Surface(color = accent.copy(alpha = 0.15f), shape = CircleShape) {
                Icon(
                    imageVector = if (badge.unlocked) Icons.Default.EmojiEvents else Icons.Default.Lock,
                    contentDescription = null,
                    tint = accent,
                    modifier = Modifier.padding(8.dp).size(24.dp)
                )
            }

            Spacer(Modifier.size(12.dp))

            Column(modifier = Modifier.weight(1f)) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Text(
                        text = badge.displayName,
                        fontWeight = FontWeight.Bold,
                        fontSize = 16.sp,
                        color = if (badge.unlocked) Color(0xFF1F2937) else Color.Gray
                    )
                    if (isRecent(badge.unlockedAt)) {
                        Spacer(Modifier.size(6.dp))
                        Surface(
                            color = Color(0xFF10B981).copy(alpha = 0.15f),
                            shape = MaterialTheme.shapes.small
                        ) {
                            Text(
                                "NOVO",
                                color = Color(0xFF10B981),
                                fontSize = 9.sp,
                                fontWeight = FontWeight.Bold,
                                modifier = Modifier.padding(horizontal = 5.dp, vertical = 2.dp)
                            )
                        }
                    }
                }
                Spacer(Modifier.size(2.dp))
                Text(
                    text = badge.description,
                    fontSize = 12.sp,
                    color = Color.Gray
                )
            }
        }
    }
}

/** A badge unlocked in the last day gets a NOVO marker — the payoff for the capture. */
private fun isRecent(unlockedAt: String?): Boolean {
    if (unlockedAt == null) return false
    return try {
        Duration.between(Instant.parse(unlockedAt), Instant.now()).toHours() < 24
    } catch (e: Exception) {
        false
    }
}
```

- [ ] **Step 7: Add the route and restructure the bottom bar**

In `ui/navigation/Routes.kt`:

```kotlin
    const val PROFILE = "profile"
```

In `ui/components/AppBottomBar.kt`, replace the `items` list with the final five-tab layout — this supersedes the list from Task 17:

```kotlin
    val items = listOf(
        BarItem(Routes.NEARBY, Icons.Default.WbSunny, "Eventos Próximos"),
        BarItem(Routes.FEED, Icons.Default.DynamicFeed, "Feed"),
        BarItem(Routes.HOME, Icons.Default.Home, "Início"),
        BarItem(Routes.SKYDEX, Icons.Default.CatchingPokemon, "SkyDex"),
        BarItem(Routes.PROFILE, Icons.Default.Person, "Perfil")
    )
```

with the import `androidx.compose.material.icons.filled.Person`. Drop the now-unused `People` import.

In `ui/navigation/SkyDexNavHost.kt`, add `Routes.PROFILE` to `BAR_ROUTES` (keeping `FRIENDS` and `MY_CAPTURES` in it so the bar stays visible when Profile navigates to them), and add the destination:

```kotlin
            composable(Routes.PROFILE) {
                val vm: ProfileViewModel = viewModel {
                    ProfileViewModel(
                        gateway = ServiceLocator.profileRepository,
                        onLogout = { ServiceLocator.authRepository.logout() }
                    )
                }
                ProfileScreen(
                    viewModel = vm,
                    onOpenMyCaptures = { navController.navigate(Routes.MY_CAPTURES) },
                    onOpenFriends = { navController.navigate(Routes.FRIENDS) },
                    onLoggedOut = {
                        navController.navigate(Routes.LOGIN) {
                            popUpTo(0) { inclusive = true }
                        }
                    }
                )
            }
```

with imports for `ProfileScreen` and `ProfileViewModel`. `popUpTo(0) { inclusive = true }` clears the whole back stack so the logged-out user cannot swipe back into the app.

**Update the KDoc on `SkyDexNavHost` while you are in the file.** It currently ends with *"There is no logout affordance in the app yet; this is the contract for the task that adds one."* This is that task. Leaving the sentence there leaves the file telling its next reader that a thing they can see in the code below does not exist. Rewrite it to say the Profile screen owns the logout affordance and points at this destination; keep the `popUpTo(0)` explanation, which is still exactly right. Task 17 had to delete an equivalently stale comment about `Routes.NEARBY` — comments that describe a future that has arrived are the ones that mislead hardest, because they read as current.

- [ ] **Step 8: Compile and run every Android test**

```bash
JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:compileDebugKotlin :app:testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, every suite green.

- [ ] **Step 9: Verify the badge loop on a real phone**

Start the backend and install the app as in Task 10, step 10.

1. Open the Perfil tab on a fresh account. **Expected: 0/13 conquistas, every badge greyed out with a lock icon, and the descriptions readable as goals.**
2. Make one confirmed capture. Return to Perfil. **Expected: "Molhou o Dedo" unlocked, with a green NOVO marker, and the counter reads 1/13.**
3. Make two more confirmed captures. **Expected: "Caçador de Nuvem" (3 captures) unlocks — this is the badge from the original request.**
4. Make five captures claiming a species the sky is not doing. **Expected: "Otimista Climático" unlocks and the confirmed count does not move.**
5. Tap the "amigos" tile. **Expected: it navigates to the Amigos screen and the bottom bar stays visible.**
6. Tap "Sair da conta". **Expected: it lands on Login, and the system back button does not return to the app.**

Cross-check the backend:

```bash
docker exec skydex-db psql -U guilherme_becker -d skydex \
  -c 'SELECT u.email, b.achievement, b.unlocked_at FROM user_badges b JOIN users u ON u.id = b.user_id ORDER BY b.unlocked_at;'
```

Expected: one row per unlocked badge per user, never a duplicate `(user_id, achievement)` pair.

- [ ] **Step 10: Commit**

```bash
git add -A app
git commit -m "feat: profile screen with the achievement badge shelf

Profile replaces Friends as the fifth bottom-bar tab and hosts identity,
level, stats, the badge shelf and logout. Meus Registros and Amigos are
now reached from here."
```

---

## Definition of Done

The MVP is done when all of the following hold. Verify them in one sitting, on a real phone, against a freshly reset database.

- [ ] `cd SkyDex-backend && JAVA_HOME=$HOME/.jdks/ms-17.0.20 ./gradlew test` → BUILD SUCCESSFUL, and the run never touched the dev database.
- [ ] `cd SkyDex---frontend && JAVA_HOME=$HOME/android-studio/jbr ./gradlew :app:testDebugUnitTest` → BUILD SUCCESSFUL.
- [ ] No API response anywhere contains a `password` or `passwordHash` field. Check with `curl -s .../api/users/me -H "Authorization: Bearer $TOKEN" | grep -i pass` returning nothing.
- [ ] `DELETE /api/events` and `DELETE /api/users` do not exist. `PUT`/`DELETE /api/events/{id}` return 403 for a non-owner.
- [ ] An expired or malformed token returns 401, not 500 and not 403.
- [ ] Register → log in → force-stop → reopen lands on Home, still authenticated.
- [ ] The Home screen shows phenomena for the phone's **actual** position, not for São Paulo.
- [ ] Capturing takes a real photograph, uploads it, and the photo renders in Meus Registros and in the Feed.
- [ ] Claiming the phenomenon Open-Meteo reports for your position and hour yields `CONFIRMED` and awards its rarity's XP; claiming a different one yields `UNCONFIRMED` and 0 XP, and the capture is still saved.
- [ ] The SkyDex tab shows all 9 species, unlocks only confirmed ones, and its level and XP match `GET /api/skydex`.
- [ ] Two accounts can become friends and each sees the other's captures in the feed; a third, unrelated account sees neither.
- [ ] A fresh account's Perfil tab shows 0/13 conquistas with every badge locked and its description readable as a goal.
- [ ] The third confirmed capture unlocks "Caçador de Nuvem", marked NOVO, and the counter moves to 3/13 or higher.
- [ ] Five contradicted claims unlock "Otimista Climático" while `confirmedCaptures` stays at 0.
- [ ] `SELECT user_id, achievement, count(*) FROM user_badges GROUP BY 1,2 HAVING count(*) > 1` returns no rows after repeated profile reads.
- [ ] "Sair da conta" lands on Login and the system back button does not re-enter the app.
- [ ] Rotating the phone on any tab does not re-fetch or lose state.

---

## Post-MVP backlog

Recorded here so the MVP tasks above stay honest about what they deliberately left out. Do not build these as part of this plan.

1. **Flyway migrations.** `ddl-auto=update` is fine while data is disposable; it is not fine once a real user has a capture. This is the first thing to do after the MVP ships, and it should happen before any external user installs the app.

   Task 6 already planted the first mine. `WeatherEvent.latitude`/`longitude` are `@Column(nullable = false)` with no `@ColumnDefault`, and Hibernate emits no DDL `DEFAULT` for a Kotlin property initializer — so on any database that already has rows, `alter table weather_events add column latitude float8 not null` fails with *"column contains null values"*. `SchemaUpdate` **logs that failure and lets the context start**, so the app boots looking healthy and then dies on the first read or write of `weather_events` with *"column does not exist"*. Our dev database has zero rows, which hides this completely. Note also that the obvious backfill — `DEFAULT 0` to match the entity initializers — parks every legacy capture on Null Island, which Task 12 would then cheerfully validate against the weather in the Gulf of Guinea. Pick a real backfill or make the columns nullable in the migration.
2. **Password change and password reset.** The MVP sets a password at registration and never lets it change: `User.passwordHash` has no mutator and `UpdateProfileRequest` carries only name and email. A change endpoint (`PUT /api/users/me/password`, verifying the current password) is small; a reset flow needs email delivery and is the larger piece.
3. **Refresh tokens.** The JWT expires after 2 hours with no renewal path, so a user is silently logged out mid-session. As a stopgap, add an OkHttp interceptor that clears the session and routes to Login on any 401, then do this properly.
4. **Object-storage photos.** Local disk means photos die with the container and cannot be served from more than one instance.
5. **`GET /api/phenomena`.** The species chips in `CaptureScreen` are a hardcoded copy of the backend enum. A discovery endpoint removes the drift risk noted in Task 14.
6. **"Badge desbloqueado!" at capture time.** Task 18 persists badges during `POST /api/events` but deliberately leaves the response shape alone, so the payoff only appears on the next Profile visit (marked NOVO). Returning the freshly unlocked badges from the capture response — or a small dedicated endpoint — would let the capture screen celebrate immediately.
7. **In-app CameraX viewfinder,** replacing the system-camera intent.
8. **Authenticated photo reads.** `GET /api/photos/**` is currently public with UUID filenames.
9. **Rate limiting** on `/auth/**` and `/api/photos`.
10. **Compose UI tests.** This plan tests ViewModels and pure logic only; the screens are verified by hand.
11. **Captures near me,** using the PostGIS geography column that `hibernate-spatial` is already on the classpath for.
12. **Push notifications** when a friend captures something legendary.
13. **Delete the photo file when its capture is deleted.** `DELETE /api/events/{id}` and account deletion both remove rows and leave the JPEG on disk forever, so storage grows monotonically and a "deleted" photo stays readable to anyone holding its URL. Not a correctness bug — nothing links to the orphan — but it is the same shape as the orphaned-events bug Task 3 fixed, one layer down. Doing it properly means deriving the filename from `photoUrl`, deleting inside the same transaction, and deciding what happens when the file is already gone.

14. **A title typed while a submit is in flight is silently dropped.** `CaptureViewModel.submit()` reads `val current = _state.value` once and uses `current.title` / `current.description` after the upload suspends, so text edited mid-flight reaches the screen but not the request. Confirmed by probe, and it predates the capture screen's fix rounds. The re-entrancy guard narrows the window rather than closing it. Fix by re-reading the state after the upload, or by disabling the text fields during submit.

15. **A retake that lands *during* `create` still files the capture under the old photo.** Task 10's second guard checks `photoFile` after the upload and before `create`, which is as late as it can check — a `create` cannot be un-sent. The remaining window is `create` alone, down from upload-plus-create. A post-`create` comparison could at least suppress navigation and tell the user what happened.

16. **Aborted submits create their own orphans.** When the pre-`create` guard aborts, the JPEG it already uploaded stays on the server and the conditional cache deliberately does not record it, so nothing will ever reuse it. Same sweep as item 13 — noted because the cache's comment reads as if it prevents all avoidable orphans, and it now also creates a small class of them.

17. **`Routes.NEARBY` renders the same screen as `HOME`, and now has its own tab pointing at it.** *(Updated after Task 17, which changed half of this entry: NEARBY is no longer unreachable — it got a bottom-bar tab, and the stale "Registered but unreachable" comment was deleted. What remains is worse in one way, because it is now user-visible: the bar shows "Eventos Próximos" and "Início" as two tabs onto byte-identical content — same `HomeViewModel`, same `HomeContent`.)* No task in this plan ever gave NEARBY a distinct screen. Either give it one — the phenomena list without the capture card — or delete the route, the composable, the tab and the `BAR_ROUTES` entry together. This is the highest-visibility item on this list: it is the only one a user meets by opening the app.

18. **Home never refreshes its weather after the first successful load.** The one-shot latch re-arms on `Error` but not on `Success`, so a `Success` list is loaded once per process while the user moves and the hour changes. Deliberate: reloading on every return costs a `getCurrentLocation` with a 15-second window plus a network call on every tab switch, for data that is hourly at source. The honest fix is time-based — reload on entry if the last load is older than N minutes — which needs an injected clock and a threshold nobody has picked. A `Success` carrying an *empty* list likewise has no manual refresh.

---

## Self-review notes

Recorded during the review pass over this plan, so the executor knows these were considered rather than missed.

- **Growing DTOs.** `WeatherEvent`, `CreateWeatherEventRequest` and `WeatherEventResponse` each gain fields in three separate tasks (2 → 6 → 12/13). Every task's **Interfaces → Produces** block states the shape at that point and names the task that extends it next. New fields are always appended, never inserted, so named-argument call sites keep compiling.
- **`BadUploadException` does double duty.** Task 7 introduces it for photo validation; Tasks 12 and 15 reuse it for "Unknown phenomenon" and "You cannot add yourself". Both of those tasks say explicitly: either accept the reuse or rename it to `BadRequestException` once, updating the handler and all three call sites together. What must not happen is two exception types that both mean 400.
- **`rarityColor` is defined twice on purpose, briefly.** Task 11 adds a private copy in `HomeScreen.kt` because `SkyDexScreen.kt` does not exist yet; Task 14 step 7 deletes that copy and imports the shared one.
- **Guard order in `CaptureViewModel.submit()` is load-bearing.** Task 10 establishes title/description → photo → position; Task 14 appends phenomenon last, precisely so the Task 10 tests keep asserting the guard they were written for.
- **Two destructive dev-database resets** are required, in Task 2 (renames) and Task 12 (new non-nullable columns). Both drop only the application tables, leaving the PostGIS extensions alone, and both were pre-authorized by the user on 2026-08-07 after being told what they destroy. If a third reset turns out to be needed, that is the signal to stop and bring Flyway forward instead of dropping tables a third time.
- **`persistEvent` grows twice** (Task 6 adds coordinates, Task 12 adds species and validation). Both times the parameters have defaults, so existing call sites are unaffected.
- **Tests that assert removed endpoints get deleted, not skipped.** Task 2 step 15 names the two (`POST /api/users`, `GET /api/users`).
- **The bottom bar is built twice.** Task 5 gives it three tabs, Task 17 five, Task 19 replaces the fifth. Task 17 says explicitly that Task 19 supersedes it, because `FriendsScreen` has to exist before Profile can host the link to it.
- **Badges are additive-only and never revoked.** Deleting a capture or removing a friend leaves the badge row in place. That is intentional: an achievement records that something happened, not that it is still true. Task 18 states it so nobody "fixes" it later.
- **`Achievement` has 13 entries, and three places encode that number** — `AchievementTest` derives it from `Achievement.entries.size`, `ProfileControllerTest` likewise, and only the manual verification steps and the Definition of Done write `13` literally. Adding a badge therefore touches no test assertions, just those prose counts.
