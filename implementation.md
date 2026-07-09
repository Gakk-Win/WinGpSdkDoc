# Win GP SDK — Implementation & Integration Guide

The **Win GP SDK** (`com.gakk.winsdk:mygp`) is an embeddable quiz/games widget for
Android, shipped as an AAR and consumed by the MyGP host app. It provides a
self-contained Jetpack Compose widget (and a widget-less full-screen entry point)
that manages its own UI, state, and networking, while delegating **authentication**
and **analytics/event handling** to the host app.

> This document reflects the current public API (SDK **1.4.1**). For the
> version-by-version history, see [changelog.md](./changelog.md).

---

## 1. Requirements

| Component      | Version                     |
| -------------- | --------------------------- |
| Android SDK    | Min 21 (`compileSdk 35`)    |
| Compose BOM    | 2024.09.00                  |
| Kotlin         | 2.1.0                       |
| Gradle Plugin  | 8.13.0                      |
| Java target    | 11                          |

The SDK is Kotlin-only and built on Compose Material 3.

---

## 2. Dependency Setup

### 2.1 Declare the Artifactory repository

Add the Win Artifactory repo to your project's `settings.gradle.kts` (or root
`build.gradle.kts` `dependencyResolutionManagement` block):

```kotlin
repositories {
    maven("https://jfrog.deenislamic.com/artifactory/win") {
        credentials {
            username = ""                       // Keep it empty
            password = "<token provided by Win>" // access token supplied by the Win team
        }
    }
}
```

### 2.2 Add the dependency

In your app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.gakk.winsdk:mygp:<version>")
    // See changelog.md for the latest version.
}
```

**Published coordinates:** `groupId = com.gakk.winsdk`, `artifactId = mygp`.

---

## 3. Architecture Overview

The SDK follows **MVVM + Repository**, uses **Jetpack Compose** for UI, and uses
**no dependency-injection framework** (manual construction / lazy init).

### 3.1 Gradle modules

- **`winSdk`** — the library itself (Android Library plugin). Published as the AAR.
- **`app`** — a demo/test harness that depends on `winSdk` via `project(":winSdk")`
  and demonstrates every integration path (see `app/.../MainActivity.kt`).

### 3.2 Public API surface

All host-facing types live in `winSdk/.../publicapi/`. Everything else in the SDK
is `internal`.

| Type                        | Kind                | Purpose                                                                 |
| --------------------------- | ------------------- | ----------------------------------------------------------------------- |
| `WinQuizWidget`             | `@Composable`       | The primary embeddable widget entry point.                              |
| `WinSdk`                    | `object`            | Widget-less entry point: `init()`, `openPlatform()`, `events`.          |
| `WinSdkController`          | `interface`         | Host → SDK commands + SDK → host `events` stream.                       |
| `rememberWinSdkController`  | `@Composable`       | Factory for a `WinSdkController`, scoped by an optional `cardKey`.       |
| `TokenProvider`             | `interface`         | Host-implemented JWT supply/refresh. The SDK never handles auth itself. |
| `WinAccessToken`            | `data class`        | Wrapper for a raw JWT string.                                           |
| `WinEvent`                  | `sealed class`      | Events emitted to the host (game started, game over, errors, etc.).     |
| `WinLanguage`               | `enum`              | UI language: `Bengali` / `English`.                                     |
| `WinWidgetDefaults`         | `object`            | Default `Shape` and `Language` for the widget.                          |

### 3.3 Internal flow (for maintainers)

1. `WinQuizWidget` → `WinQuizWidgetHost` → `WinWidgetViewModel` (created per
   controller key).
2. `WinWidgetViewModel` drives UI via `StateFlow<WidgetState>` (Loading, Games,
   GamePreview, GameResult, Error).
3. Widget data (banners, games list, quiz categories) is fetched via
   `WinWidgetRepository` → `WinApiService` (Retrofit).
4. Game-specific logic is handled by `GameController` implementations:
   `QuizUiController`, `ImageMatchingController`, `TicTacToeController`,
   `FillBlank`. Each talks to `GamesRepository` → `GamesApiService`.
5. **Events:** ViewModel → `WinSdkEventBus` (singleton `SharedFlow`) → filtered by
   `cardKey` → `DefaultWinSdkController.events` → host.
6. **Commands:** `DefaultWinSdkController.appCommands` → ViewModel observes and
   reacts.

### 3.4 Networking

- A single Retrofit instance via `ApiClient` (double-checked-locking singleton).
- `WinInterceptor` adds auth headers; `TokenAuthenticator` handles `401` refresh
  using the host's `TokenProvider`.
- `TokenCache` holds the current token in-memory; cleared on ViewModel
  `onCleared()`.

### 3.5 Local storage

- `LocalPref` (SharedPreferences) tracks per-user, per-game free-play counts, keyed
  by the MSISDN extracted from the JWT payload.

### 3.6 Native vs. non-native games

Games whose feature IDs are **not** in `Constants.nativeGameFeatureIds` open in a
full-screen, WebView-based `GameActivity`. `GameActivity` is self-sufficient: it
receives a plain target URL, fetches the token (from `TokenCache` or
`WinSdk.tokenProvider`), builds the SSO URL itself (`UrlUtils.buildSSOUrl`), and
shows a spinner plus an error view (with retry) on failure. Both the widget path
(`WinWidgetViewModel`) and the direct path (`WinSdk.openPlatform`) launch it the
same way.

---

## 4. Integration Path A — The Compose Widget

### 4.1 Entry point signature

```kotlin
@Composable
fun WinQuizWidget(
    modifier: Modifier = Modifier,
    shape: Shape = WinWidgetDefaults.Shape,            // RoundedCornerShape(12.dp)
    language: WinLanguage = WinWidgetDefaults.Language, // WinLanguage.Bengali
    tokenProvider: TokenProvider,
    controller: WinSdkController = rememberWinSdkController(),
    viewModelStoreOwner: ViewModelStoreOwner? = null,
)
```

### 4.2 Minimal usage

```kotlin
@Composable
fun MyQuizScreen() {
    val controller = rememberWinSdkController(cardKey = "home_card")
    val tokenProvider = remember { MyTokenProvider() }

    WinQuizWidget(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        tokenProvider = tokenProvider,
        controller = controller,
    )
}
```

The widget handles its own logic and rendering internally.

### 4.3 Parameters

- **`shape`** — clips the widget's card. Defaults to `RoundedCornerShape(12.dp)`;
  pass e.g. `RectangleShape` for square corners.
- **`language`** — pass the host app's currently selected language. Changing it
  recomposes the widget with the new locale (localizes SDK strings, picks the
  matching API title field, switches digit script, and forwards `lang` to WebView
  games).
- **`viewModelStoreOwner`** — by default the widget uses the nearest
  `LocalViewModelStoreOwner` (hosting Activity/Fragment). Pass your own (e.g. a
  nav-graph back-stack entry) to share or manually scope the widget's state.
- **`controller`** — see [§6](#6-the-winsdkcontroller).

---

## 5. Integration Path B — Widget-less (`WinSdk.openPlatform`)

Use this to navigate straight into the full-screen Win platform from any
Activity/Fragment without embedding the Compose widget.

### 5.1 Register the token provider once

Call `WinSdk.init(...)` before `openPlatform()` — ideally in
`Application.onCreate`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        WinSdk.init(MyTokenProvider())
    }
}
```

> If a `WinQuizWidget` is currently composed it already registers a provider, so
> `openPlatform()` may succeed without `init()`. That fallback is timing-dependent
> — always call `init()` if you use the direct path.

### 5.2 Open the platform

```kotlin
// Opens the platform home:
WinSdk.openPlatform(context)

// Deep-link into a specific game/feature:
WinSdk.openPlatform(context, targetUrl = "https://…")
```

- `context` — any `Context`. If it is not an `Activity`, the screen is launched in
  a new task (`FLAG_ACTIVITY_NEW_TASK`).
- Throws `IllegalStateException` if no `TokenProvider` has been registered.

### 5.3 Observe direct-launch events

Events from games launched via `openPlatform` are delivered on `WinSdk.events`
(filtered by the reserved `Constants.DIRECT_LAUNCH_KEY`), **not** on any
controller:

```kotlin
LaunchedEffect(Unit) {
    WinSdk.events.collect { event: WinEvent -> handle(event) }
}
```

---

## 6. The `WinSdkController`

The controller is the two-way channel between host and widget.

```kotlin
interface WinSdkController {
    val key: String?
    val events: Flow<WinEvent>

    fun resetPlayLimit()   // Resets the player's free-play limits.
    fun showGames()        // Navigates the widget back to its games home.
}
```

Create one with `rememberWinSdkController(cardKey)`. The optional **`cardKey`**
scopes independent widget instances — each key gets its own ViewModel and its own
filtered event stream, so multiple widgets on one screen do not cross-talk.

```kotlin
val controller = rememberWinSdkController(cardKey = "card1")
// ...
Button(onClick = { controller.resetPlayLimit() }) { Text("Reset Game Limit") }
Button(onClick = { controller.showGames() }) { Text("Show Games") }

LaunchedEffect(controller) {
    controller.events.collect { event -> handle(event) }
}
```

---

## 7. `TokenProvider` — Authentication

The SDK requests an authentication token whenever it needs to hit a secure
endpoint. The host supplies a `TokenProvider`; the SDK never handles auth directly.

```kotlin
interface TokenProvider {
    suspend fun getToken(): Result<WinAccessToken>
    suspend fun refreshToken(): Result<WinAccessToken>
}
```

Example:

```kotlin
class MyTokenProvider : TokenProvider {
    override suspend fun getToken(): Result<WinAccessToken> {
        val token = AuthRepository.getAccessToken() // secure storage or Win SSO API
        return if (token != null) {
            Result.success(WinAccessToken(token))
        } else {
            Result.failure(Exception("No token available"))
        }
    }

    override suspend fun refreshToken(): Result<WinAccessToken> {
        // Called by TokenAuthenticator on a 401. Return a freshly minted token.
        return getToken()
    }
}
```

Behavior:

- Tokens are cached in the ViewModel scope (`TokenCache`) and requested lazily —
  only when the user starts playing a game.
- `getToken()` returning `Result.failure(...)` surfaces the `Throwable`'s message
  to the user with a **retry** button, and the SDK emits `WinEvent.Failure`.
- `refreshToken()` is invoked by `TokenAuthenticator` when a request returns `401`.
- Tokens are **never persisted** by the SDK.

---

## 8. Language Support

```kotlin
enum class WinLanguage(internal val tag: String) {
    Bengali("bn"),
    English("en"),
}
```

Pass the host's selected language into `WinQuizWidget(language = …)`. Switching at
runtime recomposes the widget and drives: local string resources, the dynamic title
field chosen from API responses (`featureTitle` vs. `featureTitleEn`), the digit
script used in result counts, and the `lang` query parameter on the SSO URL for
WebView games. The default is `WinLanguage.Bengali`.

---

## 9. Event Model (`WinEvent`)

`WinEvent` is a sealed class; handle it exhaustively in a `when`. Events reach the
host via `controller.events` (widget path) or `WinSdk.events` (direct path).

| Event                    | Fired when… | Key properties |
| ------------------------ | ----------- | -------------- |
| `WidgetContentsLoaded`   | The widget's contents finished loading. | — |
| `WinOpened`              | The user navigates to the WIN website via the SDK's WebView. | `source` (banner/icon/see_all/result), `sourceDetails?` |
| `Failure`                | An error/exception occurs in the SDK. | `exception: Throwable` |
| `PlayGame`               | The user starts playing a game. | `featureId: Int?`, `featureTitle` |
| `GameOver`               | The user finishes a game. | `featureId?`, `featureTitle`, `correctAnswers`, `totalQuestions`, `status: GameStatus?` (WIN/LOSE/DRAW) |
| `CloseIconClicked`       | The user closes a game/feature. | `featureId?`, `featureTitle`, `fromResult`, `correctAnswers?`, `totalQuestions?` |
| `LimitReached`           | The user taps a game but their free-play limit is reached. | `featureId`, `featureTitle` |
| `ItemClicked`            | The user taps a banner/shortcut item. | `featureTitle`, `source` (banner/shortcut) |

> For `CloseIconClicked`, `correctAnswers`/`totalQuestions`/`featureId` are present
> only when the user quits mid-game; they are null when closing from a result
> screen (`fromResult = true`).

### 9.1 Analytics hook example

```kotlin
controller.events.collect { event: WinEvent ->
    when (event) {
        is WinEvent.PlayGame -> Firebase.logEvent("game_started", null)
        is WinEvent.WinOpened -> Firebase.logEvent(
            "win_opened",
            bundleOf("source" to event.source, "sourceDetails" to event.sourceDetails),
        )
        is WinEvent.GameOver -> Firebase.logEvent(
            "game_over",
            bundleOf("title" to event.featureTitle, "score" to event.correctAnswers),
        )
        else -> {}
    }
}
```

---

## 10. Layout & Sizing

By default the widget fills the available width. **Do not set a fixed height in
`dp`** — it may break the internal layout. Prefer responsive width modifiers:

```kotlin
WinQuizWidget(
    modifier = Modifier
        .padding(16.dp)
        .fillMaxWidth(),
    tokenProvider = tokenProvider,
    controller = controller,
)
```

---

## 11. Security Notes

- Tokens are **never persisted** by the SDK — only cached in-memory for the
  ViewModel's lifetime.
- If `getToken()`/`refreshToken()` returns `Result.failure` or a null token, the
  SDK emits `WinEvent.Failure` and shows an error message with a retry option.
- All SDK internals are `internal`; only `publicapi/` types are exposed.

---

## 12. Build & Publish (maintainers)

```bash
./gradlew assembleDebug              # Debug APK (app) + AAR (winSdk)
./gradlew :winSdk:assembleRelease    # Release AAR only
./gradlew test                       # Unit tests (all modules)
./gradlew connectedAndroidTest       # Instrumentation tests
./gradlew publish                    # Publish AAR to JFrog
```

Publishing requires the `WIN_JFROG_USERNAME` / `WIN_JFROG_PASSWORD` environment
variables. The SDK version lives in `gradle/libs.versions.toml` under the `winSdk`
key — bump it there before publishing, and record the change in
[changelog.md](./changelog.md).

---

## 13. Support & Contact

For integration help, contact the SDK maintainers at **ahsan@cloud7bd.com**.
