# Win GP SDK

The **Win GP SDK** (`com.gakk.winsdk:mygp`) is an embeddable quiz/games widget for
Android, shipped as an AAR and consumed by the MyGP host app. It provides a
self-contained Jetpack Compose widget — and a widget-less full-screen entry
point — that manages its own UI, state, and networking, while delegating
**authentication** and **analytics/event handling** to the host app.

<div class="grid cards" markdown>

- :material-book-open-variant: **[Implementation Guide](implementation.md)**

    Requirements, dependency setup, both integration paths, the event model, and
    security notes.

- :material-history: **[Changelog](changelog.md)**

    Version-by-version history of the SDK.

</div>

## Quick start

Add the Win Artifactory repo and the dependency, then drop the widget in:

```kotlin
// settings.gradle.kts
repositories {
    maven("https://jfrog.deenislamic.com/artifactory/win") {
        credentials {
            username = ""                        // Keep it empty
            password = "<token provided by Win>"  // access token supplied by the Win team
        }
    }
}

// app/build.gradle.kts
dependencies {
    implementation("com.gakk.winsdk:mygp:<version>")
}
```

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

See the **[Implementation Guide](implementation.md)** for the full walkthrough.

## Support

For integration help, contact the SDK maintainers at **ahsan@cloud7bd.com**.
