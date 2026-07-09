# Changelog

All notable changes to the **Win GP SDK** (`com.gakk.winsdk:mygp`) are documented in
this file. The SDK follows [Semantic Versioning](https://semver.org/).

---

### 1.4.1 — current

- **Fixed:** WebView safe-area insets are now handled correctly across Android versions.
- Toolchain and demo-app maintenance.

### 1.4.0

- **New — widget-less launcher:** `WinSdk.openPlatform(context, targetUrl?)` opens the
  full-screen Win platform directly from any Activity/Fragment. Register your provider
  once with `WinSdk.init(tokenProvider)` and observe results on `WinSdk.events`.
- **New — language support:** the `language: WinLanguage` parameter on `WinQuizWidget`
  switches the UI between **Bengali** and **English** (strings, API title fields, digit
  script, and WebView language), with runtime switching supported.
- **New — richer game-quit tracking:** `CloseIconClicked` now carries `featureId`,
  `correctAnswers`, and `totalQuestions` when the user quits mid-game.
- **New:** click event added to the "Others" button.
- **Fixed:** more stable widget content height and simplified games layout.
- **Fixed:** the TicTacToe board stays square and centered on large/foldable screens.
- **Fixed:** word-breaking in the more-games bottom sheet.

### 1.3.3

- **New:** customizable `shape` on `WinQuizWidget` (defaults to
  `RoundedCornerShape(12.dp)`).
- **New:** landscape orientation support for custom games.
- **Changed:** events are delivered through `WinSdkController.events`; the legacy event
  callback was removed.
- **Fixed:** updated the "No internet" copy.

### 1.3.2

- **Fixed:** disabled pinch-zoom in the WebView.

### 1.3.1

- **Changed:** added fade and scale animations to widget state transitions.

### 1.3.0

- Integration-branch release rollup.

### 1.1.0 – 1.2.6

- Incremental releases covering ongoing UI refinements, ProGuard rule updates, and
  dependency bumps (including Lottie 6.6.6).
