# TikTok Live Recorder (Android)

A small Android app that records a TikTok user's live stream to an MP4 file
in shared storage. You type in a username, tap **Start Recording**, and the
app uses [yt-dlp](https://github.com/yt-dlp/yt-dlp) (via the
[`youtubedl-android`](https://github.com/yausername/youtubedl-android)
wrapper, which bundles both yt-dlp and ffmpeg for Android) to pull down the
live stream in the background, saving the finished file to
`Movies/TikTokRecordings` where it shows up in the Gallery / Files app.

## ⚠️ Please use responsibly

Only record streams you have the right to record (e.g. your own streams, or
with the streamer's permission), and follow TikTok's Terms of Service and
the laws that apply to you. This project is provided for personal,
non-commercial archival use — it is not affiliated with or endorsed by
TikTok, and TikTok's web/API structure can change at any time, which may
break the app until yt-dlp is updated.

## How it works

- `MainActivity` — simple UI: a username field, Start/Stop buttons, a status line.
- `RecordingService` — a foreground `Service` that builds the URL
  `https://www.tiktok.com/@<username>/live`, hands it to yt-dlp with
  `-f best -o <file>`, and streams progress back to the UI.
- `MediaStoreHelper` — once recording stops (stream ends, or the user taps
  Stop), copies the mp4 from the app's private cache into the public
  `Movies/TikTokRecordings` folder via `MediaStore` (Android 10+) or a
  direct file copy (Android 9 and below).
- `App` — initializes the yt-dlp and ffmpeg binaries once at app startup.

## Building locally

1. Open the project folder in Android Studio (Hedgehog or newer).
2. Let Gradle sync — it will pull `youtubedl-android` from JitPack.
3. Run on a device/emulator running Android 8.0 (API 26) or higher.

If you want a `gradlew` wrapper for the command line / Android Studio to
regenerate automatically, open the project once in Android Studio or run:

```bash
gradle wrapper --gradle-version 8.7
```

## Building the APK with GitHub Actions

Two workflows are included under `.github/workflows/`:

- **`android-build.yml`** — runs on every push/PR to `main`. Builds both a
  debug and a release APK and uploads them as workflow artifacts (visible
  under the Actions run's "Artifacts" section).
- **`release.yml`** — runs when you push a tag like `v1.0.0`. Builds the
  release APK and attaches it to a new GitHub Release automatically.

Both workflows use `gradle/actions/setup-gradle` to install Gradle on the
runner, so you don't need to commit the `gradle-wrapper.jar` binary for CI
to work.

### Getting a properly signed release APK ("ready to publish")

By default the release build is **unsigned** (fine for testing/sideloading,
not for the Play Store). To have CI produce a signed APK you can publish:

1. Generate a keystore if you don't already have one:
   ```bash
   keytool -genkey -v -keystore release.keystore -alias upload \
     -keyalg RSA -keysize 2048 -validity 10000
   ```
2. Base64-encode it: `base64 -w0 release.keystore > keystore.b64`
3. In your GitHub repo, go to **Settings → Secrets and variables → Actions**
   and add:
   - `KEYSTORE_BASE64` — contents of `keystore.b64`
   - `KEYSTORE_PASSWORD` — your keystore password
   - `KEY_ALIAS` — e.g. `upload`
   - `KEY_PASSWORD` — your key password
4. Push a tag (`git tag v1.0.0 && git push --tags`) — the `release.yml`
   workflow will decode the keystore, build a signed release APK, and attach
   it to a new GitHub Release.

Without those secrets, the workflows still build successfully and produce an
unsigned release APK (and a debug APK, which is always auto-signed with the
debug key) as artifacts.

### Before publishing to the Play Store

- Change `applicationId`/`namespace` in `app/build.gradle` away from
  `com.example.tiktokliverecorder`.
- Replace the placeholder adaptive-icon vector art in
  `app/src/main/res/drawable/ic_launcher_*.xml` with real branding.
- Google Play's policies restrict apps that circumvent platform terms of
  service or facilitate unauthorized downloading of others' content —
  review the Play Developer Program Policies before submitting a tool like
  this; many stream-recording utilities are distributed as sideloaded APKs
  (e.g. via GitHub Releases) rather than through the Play Store for this
  reason.

## Known limitations

- TikTok frequently changes its site/CDN; if recording stops working, try
  bumping the `youtubedl-android` library version (it ships updated yt-dlp
  builds) in `app/build.gradle`.
- Aggressive battery-optimization on some OEM phones (Xiaomi, Huawei,
  Samsung, etc.) can kill long-running background services — ask the user
  to disable battery optimization for the app if recordings cut off early.
- The app currently records whatever is live when you tap Start; it does
  not poll/wait for a user to go live in the future (you'd add a polling
  loop in `RecordingService` for that).
