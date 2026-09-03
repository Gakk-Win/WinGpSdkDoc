# Win GP SDK — Partner Integration Guide

**SDK version:** `1.5.0`  
**Min Android SDK:** 21 (Android 5.0)  
**Kotlin:** 2.1+  
**Compose BOM:** 2024.09.00+

The **Win GP SDK** (`com.gakk.winsdk:mygp`) is an embeddable quiz/games widget for
Android. It ships as an AAR and gives you two ways to surface Win: a self-contained
**Jetpack Compose widget** you drop into a screen, or a **widget-less full-screen
launcher** you can trigger from anywhere. The SDK owns its own UI, state, and
networking — you only supply an **auth token** and, optionally, listen for **events**.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Add the Dependency](#2-add-the-dependency)
3. [Permissions & File Uploads](#3-permissions-file-uploads)
4. [Authentication — `TokenProvider`](#4-authentication-tokenprovider)
5. [Integration Path A — The Compose Widget](#5-integration-path-a-the-compose-widget)
6. [Integration Path B — Widget-less Launcher](#6-integration-path-b-widget-less-launcher)
7. [Controlling the Widget — `WinSdkController`](#7-controlling-the-widget-winsdkcontroller)
8. [Language Support](#8-language-support)
9. [Configuration Reference](#9-configuration-reference)
10. [Event Reference](#10-event-reference)
11. [Layout & Sizing](#11-layout-sizing)
12. [ProGuard / R8](#12-proguard-r8)
13. [Common Patterns](#13-common-patterns)
14. [FAQ](#14-faq)

---

## 1. Prerequisites

| Requirement           | Minimum                  |
|-----------------------|--------------------------|
| Android Gradle Plugin | 8.0+                     |
| Kotlin                | 2.1.0+                   |
| Jetpack Compose       | BOM 2024.09.00+          |
| `compileSdk`          | 35+                      |
| `minSdk`              | 21                       |
| Java target           | 11                       |

The SDK is Kotlin-only and built on Compose Material 3. It bundles its own
transitive dependencies (Retrofit, Coil, Lottie) — you do not need to declare any of
those yourself.

---

## 2. Add the Dependency

### 2a. Add the Maven repository

In your **project-level** `settings.gradle.kts` (or `build.gradle.kts`), add the
private repository inside `dependencyResolutionManagement`:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        // Win GP SDK repository
        maven {
            url = uri("https://jfrog.deenislamic.com/artifactory/win")
            credentials {
                username = "" // Keep it empty
                password = providers.gradleProperty("WIN_JFROG_TOKEN").orNull
                    ?: System.getenv("WIN_JFROG_TOKEN")
            }
        }
    }
}
```

> **Important:** The token is provided by the Gakk business team.

Store the credential in your project's `local.properties` (never commit this file):

```properties
# local.properties  — add to .gitignore
WIN_JFROG_TOKEN=the_token_provided_from_gakk
```

…or as a CI environment variable `WIN_JFROG_TOKEN`.

### 2b. Declare the dependency

In your **app module** `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.gakk.winsdk:mygp:1.5.0")
    // See the Changelog for the latest version.
}
```

Sync Gradle — the SDK is now available.

---

## 3. Permissions & File Uploads

**The SDK declares only `INTERNET`, and asks for no runtime permission at any point.**

The `INTERNET` permission is declared in the SDK's own manifest, which the manifest
merger folds into your app automatically.

> **No action required** — you do not need to add any permission to your
> `AndroidManifest.xml`, and you never have to run a permission request on the SDK's
> behalf.

### 3a. File uploads in web games

Web games that use `<input type="file">` are served by AndroidX activity-result
contracts, none of which requires a grant: the pickers return only the items the user
actually selected, and a camera capture is written into the SDK's own cache directory
rather than into shared storage.

**How an input is routed** — the input's own attributes decide, with no prompt of the
SDK's own in between:

| The input | Opens | Contract |
|-----------|-------|----------|
| `capture`, with an `accept` that allows images (or no `accept`) | Camera | `TakePicture` |
| `accept` listing only image and video types | [Photo picker][photo-picker] | `PickVisualMedia` |
| Everything else, including an input with no `accept` | Document picker | `GetContent` |

`multiple` switches the two picker rows to `PickMultipleVisualMedia` /
`GetMultipleContents`.

`accept` is read as MIME types, with `.png`-style extensions resolved for you and casing
ignored. A single concrete type (`image/gif`, `application/pdf`) becomes the picker's
filter as-is; several image and video types narrow the photo picker to images, video, or
both; several types of any other kind open the document picker on `*/*`.

> Recording video is not offered — a game asking for `video/*` with `capture` picks an
> existing file.

### 3b. What the SDK adds to your manifest

One `<provider>`, so a camera app has somewhere to write. It is merged in from the AAR —
**you do not declare it yourself**:

```xml
<provider
    android:name="com.gakk.winsdk.ui.WinSdkFileProvider"
    android:authorities="${applicationId}.winsdk.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/win_sdk_file_paths"/>
</provider>
```

The authority is derived from your `applicationId`, so it cannot collide with another
app. It is a `FileProvider` subclass rather than `androidx.core.content.FileProvider`
itself, so it will not collide with a provider your own app declares. Captured photos go
to `cacheDir/win_sdk_captures` and are cleared on the next capture an hour or more later.

### 3c. If your app declares `android.permission.CAMERA`

Android then requires that grant before *any* camera intent will start, including the
SDK's — this follows from the merged manifest, not from which API the SDK uses. The SDK
never requests the permission. If it is declared but not granted, a `capture` input falls
back to a picker.

[photo-picker]: https://developer.android.com/training/data-storage/shared/photo-picker

---

## 4. Authentication — `TokenProvider`

The SDK never handles auth itself. Instead, you implement a `TokenProvider` that hands
the SDK a valid JWT on demand. The SDK extracts the subscriber's MSISDN from the token
payload server-side.

```kotlin
interface TokenProvider {
    suspend fun getToken(): Result<WinAccessToken>
    suspend fun refreshToken(): Result<WinAccessToken>
}
```

### Example implementation

```kotlin
class MyTokenProvider : TokenProvider {

    override suspend fun getToken(): Result<WinAccessToken> {
        val token = authRepo.getAccessToken() // secure storage or your SSO API
        return if (token != null) {
            Result.success(WinAccessToken(token))
        } else {
            Result.failure(IllegalStateException("No token available"))
        }
    }

    override suspend fun refreshToken(): Result<WinAccessToken> {
        // Called automatically when a request returns 401.
        // Return a freshly minted token, or Result.failure to surface an error.
        return getToken()
    }
}
```

**Behavior**

- Tokens are requested **lazily** — only when the user actually starts a game — and
  cached in memory for the session.
- `getToken()` returning `Result.failure(...)` shows the `Throwable`'s message to the
  user with a **Retry** button, and emits [`WinEvent.Failure`](#10-event-reference).
- `refreshToken()` is invoked on a `401` before the SDK retries the request once.
- Tokens are **never persisted** by the SDK.

---

## 5. Integration Path A — The Compose Widget

Use `WinQuizWidget` when your host screen is written in Jetpack Compose. The widget
renders inline and manages its own state.

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

### Minimal example

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

That's it — the widget handles its own rendering and logic internally.

### Full example with events

```kotlin
@Composable
fun HomeScreen() {
    val controller = rememberWinSdkController(cardKey = "home_card")
    val tokenProvider = remember { MyTokenProvider() }

    // Observe what happens inside the widget
    LaunchedEffect(controller) {
        controller.events.collect { event ->
            when (event) {
                is WinEvent.PlayGame -> analytics.log("game_started", event.featureTitle)
                is WinEvent.GameOver -> analytics.log("game_over", event.featureTitle)
                is WinEvent.Failure  -> Log.e("Win", "SDK error", event.exception)
                else -> Unit
            }
        }
    }

    WinQuizWidget(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        language = WinLanguage.Bengali,
        tokenProvider = tokenProvider,
        controller = controller,
    )
}
```

---

## 6. Integration Path B — Widget-less Launcher

Use `WinSdk.openPlatform()` to jump straight into the full-screen Win platform from
**any** Activity or Fragment — no Compose widget required.

### Step 1 — register your token provider once

Call `WinSdk.init(...)` before `openPlatform()`, ideally in `Application.onCreate`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        WinSdk.init(MyTokenProvider())
    }
}
```

### Step 2 — open the platform

```kotlin
// Opens the platform home:
WinSdk.openPlatform(context)

// Deep-link straight into a specific game/feature:
WinSdk.openPlatform(context, targetUrl = "https://…")
```

- `context` — any `Context`. If it is **not** an `Activity`, the screen launches in a
  new task automatically.
- Throws `IllegalStateException` if no `TokenProvider` has been registered — always
  call `WinSdk.init()` first.

### Step 3 — observe direct-launch events

Events from `openPlatform` are delivered on the **global** `WinSdk.events` flow, not on
any controller:

```kotlin
LaunchedEffect(Unit) {
    WinSdk.events.collect { event ->
        when (event) {
            is WinEvent.PlayGame -> analytics.log("game_started", event.featureTitle)
            else -> Unit
        }
    }
}
```

> **Which path should I use?** Use the **widget** (Path A) to embed Win inside an
> existing screen with its banners and game list. Use the **launcher** (Path B) when
> you just want a button or deep-link that drops the user into the full Win platform.

---

## 7. Controlling the Widget — `WinSdkController`

The controller is the two-way channel between your app and a widget instance.

```kotlin
interface WinSdkController {
    val key: String?
    val events: Flow<WinEvent>

    fun resetPlayLimit()   // Reset the player's free-play limits.
    fun showGames()        // Navigate the widget back to its games home.
}
```

Create one with `rememberWinSdkController(cardKey)`:

```kotlin
val controller = rememberWinSdkController(cardKey = "card1")

Button(onClick = { controller.resetPlayLimit() }) { Text("Reset Game Limit") }
Button(onClick = { controller.showGames() })      { Text("Show Games") }
```

> **`cardKey` scopes independent widget instances.** Each key gets its own state and
> its own filtered event stream, so multiple widgets on one screen never cross-talk.
> Give each widget a distinct `cardKey`.

---

## 8. Language Support

```kotlin
enum class WinLanguage { Bengali, English }
```

Pass the host app's currently selected language into `WinQuizWidget(language = …)`.
Switching it at runtime recomposes the widget and drives: localized SDK strings, the
title field chosen from API responses, the digit script used in result counts, and the
language forwarded to WebView games. The default is `WinLanguage.Bengali`.

```kotlin
var language by rememberSaveable { mutableStateOf(WinLanguage.Bengali) }

WinQuizWidget(
    language = language,
    tokenProvider = tokenProvider,
    controller = controller,
)

Button(onClick = {
    language = if (language == WinLanguage.Bengali) WinLanguage.English else WinLanguage.Bengali
}) { Text("Toggle language") }
```

---

## 9. Configuration Reference

### `WinQuizWidget` parameters

| Parameter             | Type                   | Default                     | Description |
|-----------------------|------------------------|-----------------------------|-------------|
| `modifier`            | `Modifier`             | `Modifier`                  | Standard Compose modifier. Prefer `fillMaxWidth()`. |
| `shape`               | `Shape`                | `RoundedCornerShape(12.dp)` | Clips the widget's card. Pass `RectangleShape` for square corners. |
| `language`            | `WinLanguage`          | `WinLanguage.Bengali`       | UI language; see [§8](#8-language-support). |
| `tokenProvider`       | `TokenProvider`        | — (required)                | Supplies/refreshes the auth token; see [§4](#4-authentication-tokenprovider). |
| `controller`          | `WinSdkController`     | `rememberWinSdkController()`| Host ↔ widget channel; see [§7](#7-controlling-the-widget-winsdkcontroller). |
| `viewModelStoreOwner` | `ViewModelStoreOwner?` | `null`                      | Defaults to the nearest owner (Activity/Fragment). Override to scope state to e.g. a nav destination. |

### `WinWidgetDefaults`

```kotlin
object WinWidgetDefaults {
    val Shape: Shape = RoundedCornerShape(12.dp)
    val Language: WinLanguage = WinLanguage.Bengali
}
```

### `WinSdk.openPlatform` parameters

| Parameter   | Type      | Default | Description |
|-------------|-----------|---------|-------------|
| `context`   | `Context` | —       | Any context. Non-Activity contexts launch in a new task. |
| `targetUrl` | `String?` | `null`  | Optional deep link. `null` opens the platform home. |

---

## 10. Event Reference

`WinEvent` is a sealed class — handle it exhaustively in a `when`. Events arrive via
`controller.events` (widget path) or `WinSdk.events` (launcher path).

| Event                  | Payload | When fired |
|------------------------|---------|------------|
| `WidgetContentsLoaded` | — | The widget's contents finished loading. |
| `WinOpened`            | `source: String`, `sourceDetails: String?` | The user navigates to the Win website via the SDK's WebView. |
| `PlayGame`             | `featureId: Int?`, `featureTitle: String` | The user starts playing a game. |
| `GameOver`             | `featureId: Int?`, `featureTitle: String`, `correctAnswers: Int`, `totalQuestions: Int`, `status: GameStatus?` | The user finishes a game. `status` is `WIN` / `LOSE` / `DRAW`. |
| `CloseIconClicked`     | `featureId: Int?`, `featureTitle: String`, `fromResult: Boolean`, `correctAnswers: Int?`, `totalQuestions: Int?` | The user closes a game/feature. |
| `LimitReached`         | `featureId: Int`, `featureTitle: String` | The user taps a game but their free-play limit is reached. |
| `ItemClicked`          | `featureTitle: String`, `source: String` | The user taps a banner/shortcut item. |
| `Failure`              | `exception: Throwable` | An error occurs in the SDK. |

> For `CloseIconClicked`, the `featureId` / `correctAnswers` / `totalQuestions` fields
> are populated only when the user quits **mid-game**; they are `null` when closing from
> a result screen (`fromResult = true`).

```kotlin
controller.events.collect { event ->
    when (event) {
        is WinEvent.PlayGame -> { /* game started */ }
        is WinEvent.GameOver -> { /* event.correctAnswers / event.status */ }
        is WinEvent.LimitReached -> { /* free plays exhausted */ }
        is WinEvent.Failure -> { /* event.exception */ }
        else -> Unit
    }
}
```

---

## 11. Layout & Sizing

By default the widget fills the available width. **Do not set a fixed height in `dp`** —
it may break the internal layout. Prefer responsive width modifiers:

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

## 12. ProGuard / R8

The SDK ships a `consumer-rules.pro` file that is automatically merged into your app's
ProGuard configuration. **No manual rules are required.**

---

## 13. Common Patterns

### Send events to your analytics layer

```kotlin
controller.events.collect { event ->
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
        else -> Unit
    }
}
```

### Reset a user's free-play limit

```kotlin
Button(onClick = { controller.resetPlayLimit() }) { Text("Reset Game Limit") }
```

### Send the user back to the games home

```kotlin
Button(onClick = { controller.showGames() }) { Text("Show Games") }
```

### Add a "Play Win" button anywhere (no widget)

```kotlin
Button(onClick = { WinSdk.openPlatform(context) }) { Text("Play Win") }
```

### Show two independent widgets on one screen

```kotlin
val topController = rememberWinSdkController(cardKey = "top")
val bottomController = rememberWinSdkController(cardKey = "bottom")

WinQuizWidget(controller = topController, tokenProvider = tokenProvider)
WinQuizWidget(controller = bottomController, tokenProvider = tokenProvider)
```

---

## 14. FAQ

**Q: What do I pass as the token?**  
A: A valid JWT wrapped in `WinAccessToken(token)`, returned from your
`TokenProvider`. The SDK attaches it to secure API calls and extracts the subscriber's
MSISDN from the token payload.

**Q: Do I need to add the `INTERNET` permission?**  
A: No. The SDK declares it in its own manifest and the manifest merger adds it to your
app automatically.

**Q: Does the SDK ask for camera or storage permissions?**  
A: No. File uploads from web games go through AndroidX activity-result contracts, which
need no runtime grant — the pickers return only what the user selected, and a camera
capture lands in the SDK's own cache directory. See
[§3](#3-permissions-file-uploads).

**Q: A `capture` input opens a picker instead of the camera. Why?**  
A: Most often because your own app declares `android.permission.CAMERA` without holding
the grant — Android then blocks every camera intent, the SDK's included. It can also
happen when no camera app is installed. Either way the SDK falls back to a picker rather
than leaving the input stuck.

**Q: Do I need to add any ProGuard/R8 rules?**  
A: No. The SDK bundles its own `consumer-rules.pro`, which is merged into your build
automatically.

**Q: When should I use the widget vs. `WinSdk.openPlatform`?**  
A: Use the **widget** to embed Win (banners + game list) inside one of your screens.
Use **`openPlatform`** for a button or deep link that drops the user straight into the
full-screen Win platform.

**Q: Can I show more than one widget on a screen?**  
A: Yes. Give each `rememberWinSdkController` a distinct `cardKey` so their state and
event streams stay isolated.

**Q: Why aren't my direct-launch events arriving on `controller.events`?**  
A: Games launched via `WinSdk.openPlatform` emit on the global `WinSdk.events` flow, not
on any controller. Collect `WinSdk.events` for that path.

**Q: The widget looks squished / clipped. What's wrong?**  
A: You've likely set a fixed height in `dp`. Remove it and let the widget size itself;
constrain width only.

**Q: Where do I report bugs or request features?**  
A: Contact the Win GP SDK team at **ahsan@cloud7bd.com**.
