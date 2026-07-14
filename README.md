# ShortTube — Standalone Android App

## What changed

1. **Netlify is gone.** `netlify/functions/` and `netlify.toml` are deleted.
   The YouTube calls that used to run on Netlify's servers now run
   directly from the app in `js/video-api.js`. `js/feed.js` and
   `index.html` call `ShortTubeVideoAPI` instead of `fetch('/api/feed')` /
   `fetch('/api/related')`.
1a. **Dailymotion has been removed entirely.** The Dailymotion API key,
   all `fetchDailymotion*`/`mapDailymotionItem` functions, the source
   label/CSS for "Dailymotion", and every Dailymotion mention in the
   in-app Terms/Privacy/About/Legal pages are gone. The feed is now
   YouTube + Appwrite (community uploads) only.

1b. **Unity Ads added, Google/GPT removed entirely** (`js/ads-unity.js`,
   Game ID `800084807`). `js/ads.js` (the old GPT/Google Ad Manager
   system) has been deleted along with every call to it — Unity Ads is
   now the only ad network in the app.
   - **Interstitial** (`Interstitial_Android`): a real full-screen ad
     via `@openanime/capacitor-plugin-unityads` — add it with
     `npm install @openanime/capacitor-plugin-unityads && npx cap sync android`
     (already listed in `package.json`). Fires in **both** feeds:
     every `INTERSTITIAL_EVERY_N_VIDEOS` (default 3, set to 1 for
     "every single video") videos rendered in Home, and every
     `INTERSTITIAL_EVERY_N_VIDEOS` Shorts actually scrolled/swiped to
     — both share one counter in `js/ads-unity.js` so pacing feels the
     same in either feed.
   - **Banner** (`Banner_Android`): the plugin above doesn't implement
     banner yet, so the reserved slot (`#unityBannerSlot`) is, per your
     choice, either fully hidden or a plain static "Ad space" box — no
     ad network involved at all. Toggle with `BANNER_PLACEHOLDER_MODE`
     ('hidden' or 'placeholder') at the top of `js/ads-unity.js`. Fill
     in `showBanner()`'s `loadAds()` call once real Unity banner
     support exists (or you move to Unity's newer LevelPlay mediation
     plugin, which supports banner today under a separate App
     Key/Ad Unit dashboard).
   - **Set `TEST_MODE: false`** in `js/ads-unity.js` before you submit
     anywhere — `true` only ever serves Unity's test creative.
2. **Appwrite is untouched.** `js/appwrite-config.js`, `js/auth.js`,
   `js/upload.js`, `js/profile.js`, `js/social.js` still talk to Appwrite
   exactly as before.
3. **Web assets moved into `www/`** (`index.html`, `js/`, `logo.png`,
   `splash.png`) — this is the folder Capacitor packages into the app.
4. Added `capacitor.config.json`, an updated `package.json`, and
   `.github/workflows/build-apk.yml` — a cloud pipeline that turns this
   repo into a real `.apk` without needing a terminal on your phone.

### ⚠️ Read this before you publish

The YouTube key in `video-api.js` now ships inside the APK and can be
extracted by decompiling it — there's no way around this once there's
no server to hide it behind. At minimum, before you upload anywhere
(including Amazon Appstore):
1. **Rotate the key.** The one in this repo has been exposed in files
   before and must be treated as burned — generate a new one in
   [Google Cloud Console](https://console.cloud.google.com/apis/credentials).
2. **Restrict it to the YouTube Data API v3 only** (API restrictions),
   so even if extracted it can't be reused against other Google APIs.
3. **Add an Android app restriction** (package name `com.shorttube.app`
   + your release SHA-1 signing fingerprint) once you have a signing
   keystore for the release build.
4. **Set a daily quota cap** on the key in Cloud Console → Quotas, so
   a leaked key can't run up a surprise bill or get your whole GCP
   project suspended.
5. **Check Cloud Console traffic metrics** every so often after
   launch — a sudden spike unrelated to your real users is the usual
   sign the key is being reused elsewhere; rotate again if you see it.
6. Never add an Appwrite **API key** (server secret) anywhere in this
   app — only the public Endpoint + Project ID, which is all it uses.

## Building the APK from your phone (no terminal needed)

You need a **GitHub account** and the **GitHub mobile app** (or GitHub in
your phone's browser). The actual build runs on GitHub's servers.

1. Create a new GitHub repo and upload this whole folder to it (see
   editors below for the easiest way to do this from a phone).
2. Open the repo → **Actions** tab → **Build ShortTube APK** →
   **Run workflow**. (It also runs automatically on every push to `main`.)
3. Wait a few minutes for the green checkmark, then open that run →
   scroll to **Artifacts** → download `shorttube-debug-apk`. Unzip it
   to get `app-debug.apk`.
4. Transfer that `.apk` to your phone (or download it directly on the
   phone if you ran the workflow from the phone's browser) and tap it
   to install — you'll need to allow "install unknown apps" for
   whichever app you downloaded it with.

This produces a **debug APK** (unsigned, fine for testing/personal
installs). Publishing to the Play Store requires a signed release
build — a separate step involving a keystore, which is worth tackling
once the debug build is working end to end.

## Mobile-friendly tools for editing this code

- **GitHub mobile app** — browse the repo, edit small files, and commit
  directly from your phone. Good for quick tweaks (e.g. a color, a key).
- **github.dev** — press `.` on any GitHub repo page (or replace
  `github.com` with `github.dev` in the URL) in your phone's browser to
  get a full VS Code editor, no install needed. Best for editing
  `index.html`/`js/*.js` comfortably.
- **Working Copy** (iOS) / **Acode** or **Spck Editor** (Android) — apps
  that combine a code editor with git, if you want something more
  capable than the browser.
- To upload the initial folder (including binary files like
  `logo.png`), the GitHub web UI's drag-and-drop upload (via github.dev
  or github.com in your phone browser) is the simplest path — no git
  commands required.

If this workflow ever feels like too much friction, a lower-effort
alternative for getting *a* working APK quickly is a no-code
WebView-wrapper service (e.g. Median.co, Appilix) that takes a hosted
URL or zip and hands back an APK directly — useful for a fast test
build, though the Capacitor/GitHub Actions route above gives you more
control and is the standard path if you want to keep improving this
into a real Play Store app.
