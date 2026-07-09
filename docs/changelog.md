# Changelog

All notable changes to the **Win GP SDK** (`com.gakk.winsdk:mygp`) are documented
here. The SDK version is defined in `gradle/libs.versions.toml` under the `winSdk`
key. This project follows [Semantic Versioning](https://semver.org/) and
[Conventional Commits](https://www.conventionalcommits.org/).

---

## 1.4.1 — current

- **fix:** Handle WebView safe-area insets correctly across Android versions
  (`WinInterceptor` now reports `Build.VERSION.RELEASE`).
- **build:** Add the Foojay toolchain resolver plugin for reproducible JDK
  provisioning.
- **chore:** Refresh the demo JWT in `TestTokenProvider`; bump SDK version to
  1.4.1.

## 1.4.0

- **feat:** Widget-less entry point — `WinSdk.openPlatform(context, targetUrl?)`
  launches the full-screen `GameActivity` directly from any Activity/Fragment,
  with events delivered on `WinSdk.events` (filtered by the reserved
  `DIRECT_LAUNCH_KEY`). `WinSdk.init(tokenProvider)` registers the provider
  globally.
- **feat:** Bengali & English language support — new `language: WinLanguage`
  parameter on `WinQuizWidget`; drives local strings, dynamic API title fields
  (`featureTitle` vs. `featureTitleEn`), digit script in result counts, and the
  `lang` SSO query parameter for WebView games. Runtime switching supported.
- **feat:** Internal SDK analytics reporting pipeline.
- **feat:** Game-quit tracking — `CloseIconClicked` refactored to carry
  `featureId`, `correctAnswers`, and `totalQuestions` when the user quits mid-game.
- **feat:** Added a click event to the "Others" button.
- **fix:** Stabilize widget content height; simplify `GamesView` layout.
- **fix:** Keep the TicTacToe board square and centered on large/foldable screens.
- **fix:** `WinGameIcon` uses `TextAutoSize` to resolve word-breaking in the
  more-games bottom sheet.
- **refactor:** Changed the game progress bar to horizontal in `GameActivity`.
- **chore:** Point the analytics endpoint at the default base URL; remove the
  unused `Language` enum.

## 1.3.3

- **feat:** Customizable `shape` support on `WinQuizWidget` (defaults to
  `RoundedCornerShape(12.dp)`).
- **feat:** Handle landscape orientation for custom views in `GameActivity` (via a
  `setLandscape` JS interface).
- **refactor:** Removed the legacy `WinEventCallback` in favor of the
  `WinSdkController.events` flow; updated `ResultView` styles and adopted
  `WinTextStyles` across UI components.
- **docs:** Comprehensive README integration guide and tech-stack details.
- **fix:** Updated the "No internet" text.

## 1.3.2

- **fix:** Disabled zoom in the WebView.
- **chore:** Version bump to 1.3.2.

## 1.3.1

- **refactor:** Added fade and scale animations to `WinQuizView` state transitions.
- **chore:** Version bump to 1.3.1.

## 1.3.0

- **feat:** Version bump to 1.3.0 (integration-branch release rollup).

## Earlier releases (1.1.0 – 1.2.6)

Incremental releases prior to 1.3.0 covered ongoing UI refinements, ProGuard rule
updates (1.1.2), dependency bumps (including Lottie 6.6.6), and routine version
bumps. See `git log` for the full commit-level detail.

---

### Release process (maintainers)

1. Update the `winSdk` version in `gradle/libs.versions.toml`.
2. Add a new section here summarizing the changes since the last release.
3. Publish with `./gradlew publish` (requires `WIN_JFROG_USERNAME` /
   `WIN_JFROG_PASSWORD`).
