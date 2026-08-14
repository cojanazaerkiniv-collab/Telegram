# UZGRAM — liquid glass fork notes

This fork rebrands Telegram for Android as **UZGRAM** and adds an iOS 26 style
liquid glass presentation layer on top of the existing UI. Only presentation
code is touched: the native SQLite storage layer, the MTProto serialization
engine and the networking stack are unchanged.

## Identity

| What | Value |
| --- | --- |
| `applicationId` | `uz.uzgram.messenger` (from `APP_PACKAGE` in `gradle.properties`) |
| App name | `UZGRAM` (`AppName` in every `values*/strings.xml`) |
| Version | `1.0.0-liquid`, version code `10001` |
| `minSdkVersion` | 26 |
| API credentials | `BuildVars.APP_ID` / `BuildVars.APP_HASH` |

The Java package (`org.telegram.*`) and the Gradle `namespace` are unchanged —
only the installed application id is relocated. Renaming the source package
would rewrite thousands of files with no user-visible benefit and would break
the JNI symbol names in `jni/`.

`LocaleController` now always resolves `AppName` / `AppNameBeta` from the
bundled resources: the cloud language pack is served by the upstream server and
still spells those keys "Telegram".

### Firebase / Huawei

The **application** modules' `google-services.json` files had their
`package_name` entries renamed to `uz.uzgram.messenger*`, so the Gradle plugin
matches the new application id.

`TMessagesProj/google-services.json` deliberately keeps the original
`org.telegram.messenger*` names. That module is an Android *library*, which has
no application id, so the plugin resolves it against the module's `namespace` —
still `org.telegram.messenger`. Renaming it there fails the build with
"No matching client found for package name 'org.telegram.messenger'". They still point at the upstream Firebase project, so **push
notifications, Maps and Google sign-in will not work until these files are
replaced with ones generated for a `uz.uzgram.messenger` project**.

## Building

`.github/workflows/build.yml` builds a signed APK on GitHub Actions. Open the
**Actions** tab, pick *Build UZGRAM APK*, hit **Run workflow**, choose `release`
or `debug`, and collect the APK from the run's Artifacts section.

No secrets are needed: `TMessagesProj/config/release.keystore` is committed and
its passwords are in `gradle.properties`, so the build signs itself. That also
means the APK is signed with a key that is public — fine for testing, not for
publishing. Generate your own keystore before you ship anything.

Expect a long run. The NDK compiles native code for four ABIs
(`armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`), which dominates the build; the
job allows three hours. To make CI iterate faster, trim `abiFilters` in the
`afat` flavor of `TMessagesProj_App/build.gradle` down to `arm64-v8a`.

Locally the equivalent is:

```
./gradlew :TMessagesProj_App:assembleAfatRelease
```

which needs JDK 17, SDK platform 35, build-tools 35.0.0, NDK 27.2.12479018 and
the SDK's CMake 3.10.2 (not a newer system cmake).

## Liquid glass layer

Everything lives in `org.telegram.ui.Components.LiquidGlass`: the palette, the
frosted plate and specular stroke drawing, hardware blur, immersive windows and
the spring physics. It has a master switch — `LiquidGlass.setEnabled(false)`
restores the stock look at runtime, and every helper becomes a no-op.

Wired into:

**Chrome and windows**

- `ActionBar` — refractive toolbar mask in `onDraw` (skipped where Telegram's
  own `glassMode` blur capsules already run).
- `ActionBarLayout` — `applyIos26WindowSystem(Window)` for edge-to-edge, the
  push/pop depth effect that sinks the covered screen to 0.94 scale, and the
  iOS deceleration curve on page transitions.
- `ActionBar` back button and `ActionBarMenuItem` — elastic press feedback.

**Modals**

- `AlertDialog` / `BottomSheet` — frosted plates with 24dp corners plus window
  blur-behind on Android 12+.
- `ActionBarPopupWindow` — context menus frosted along the popup background
  bounds, following the swipe-back scale and section gaps.
- `Bulletin` — toasts frosted into floating glass pills.

**Lists**

- `DialogCell` — chat list rows as detached glass plates, drawn inside the swipe
  translation so swipe actions and the archive pull keep working.
- `ProfileSearchCell` — search and contact rows as 20dp glass modules.
- `TextCell`, `TextSettingsCell`, `TextCheckCell` — settings rows as inset iOS
  grouped-table plates.
- Row dividers are suppressed wherever a plate is drawn, since the plate edge
  already separates adjacent rows.

**Chat**

- `ChatActivityEnterView` — the compose panel frosted with a specular hairline
  along the edge the message list passes under.
- `ChatActionCell` — frosted wash and specular edge following the service pill.
- `Theme.getColor` — translucent `key_chat_inBubble` and Apple blue
  `key_chat_outBubble`; bubble radius default raised to 18dp.

**Full-screen surfaces**

- `VoIPToggleButton` — call controls frosted as glass discs. These force the
  dark fill: the call screen sits on a dark blurred backdrop whatever the app
  theme is.
- `PreviewButtons`, `CaptionContainerView` — the story editor already blurs its
  own backdrop, so these take the specular edge only, rather than a second
  frosted layer on top of the first.

### What is not converted

- `EmojiView` (the sticker and emoji panel) and `GroupCallActivity`'s internals
  still use their stock backgrounds.
- `PhotoViewer` was left alone deliberately: Telegram 12.9 already routes its
  controls through `blur3`'s own frosted glass, so it is glass already, just not
  through this layer.
- Platform widgets the app hosts rather than draws — system pickers, the
  keyboard, Android share sheets — cannot be restyled from inside the app at
  all.

Note that Telegram 12.9 already ships its own glass system (`ui.Components.blur3`,
gated by `LiteMode.FLAG_LIQUID_GLASS`). This layer sits on top of it rather than
replacing it, so the two do not double-tint the same surface.

## SF Pro typography

Apple's SF Pro fonts are proprietary and are **not** committed here. To switch
the app over to them, drop these three files into
`TMessagesProj/src/main/assets/fonts/`:

```
SF-Pro-Display-Regular.ttf
SF-Pro-Display-Medium.ttf
SF-Pro-Display-Bold.ttf
```

`AndroidUtilities.getTypeface()` routes every asset font lookup through
`resolveTypefacePath()`, which redirects the upright Roboto faces to SF Pro when
all three files are present and silently falls back to the bundled Roboto faces
when they are not — so a checkout without them builds and renders exactly as
before. Italic, monospace and Merriweather faces stay on their original files
because no SF Pro counterpart is bundled.
