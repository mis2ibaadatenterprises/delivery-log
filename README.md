# Delivery Log

A delivery-team app: each delivery person uploads a **stock photo** (required) and an
optional **bill photo**; the app grabs **GPS at the moment the photo is added** and stores it
as a tap-to-open Google Maps link. One **master** account sees every person's uploads and
profiles in one feed.

It installs on a phone like a normal app (home-screen icon, no browser bars) and
**updates itself** — no reinstall, no store.

---

## Already done for you

- Firebase project **delivery-log** (`delivery-log-e6c8f`), Mumbai region, Spark (free) plan
- Email/Password sign-in enabled
- Firestore created, security rules published
- Web app registered and its config already baked into `config.js`
- Master account email set to **kanik958@gmail.com** — sign up with that email in the app and it becomes the master

## Photos are stored in Firestore, not Cloud Storage

Firebase now requires a card on file (Blaze plan) before Cloud Storage can be switched on,
so each photo is compressed in the phone (max 1280px, under ~700 KB) and saved as its own
Firestore document in the `photos` collection. The app works exactly the same.

**Limit:** Firestore's free tier is 1 GB, so roughly **2,000–2,500 photos**. To switch to real
Cloud Storage later, upgrade the project to Blaze, publish `storage.rules`, and the original
Storage code is in git history.

## Finish hosting (GitHub Pages)

1. Create a **public** repo named `delivery-log`.
2. Upload every file in this folder to the repo root.
3. **Settings → Pages → Source: Deploy from a branch → `main` / `(root)` → Save.**
4. Wait ~1 minute. Your URL will be `https://<username>.github.io/delivery-log/`.
5. **Important:** in Firebase → Authentication → Settings → **Authorized domains → Add domain**,
   add `<username>.github.io`. Sign-in fails until you do.
6. Open the URL on a phone → Chrome menu → **Add to Home screen**.

> The repo is public, so `config.js` is visible. That is fine and normal — the Firebase
> `apiKey` identifies the project, it does not grant access. The Firestore rules are what
> protect the data.

## First accounts

- Sign up with **kanik958@gmail.com** → master dashboard (sees everyone).
- Each delivery person signs up with their own email → delivery view (sees only their own).

---

## How auto-update works

Two layers, both already wired:

1. **Service worker** (`sw.js`) — app code is fetched network-first, so a new deploy is
   live on the next open. If a new build arrives while the app is open, a *"A new version
   is ready — Reload"* bar appears. It also re-checks every 30 minutes and whenever the app
   returns to the foreground.
2. **Version notice** — the app reads a Firestore document `config/app`:

   ```json
   { "version": "1.1.0", "apkUrl": "https://…/delivery-1.1.0.apk" }
   ```

   If `version` differs from the built-in one, every phone shows an update bar. Add
   `apkUrl` only if you later ship a real APK — the bar then links to the download.

**To release a new version:** bump `APP_VERSION` near the top of the `<script type="module">`
in `index.html`, bump `CACHE` in `sw.js` to match, `firebase deploy`, and (optionally) set
`config/app.version` to the same string.

---

## Turning this into a real `.apk`

The project is already a valid installable PWA, so wrapping is mechanical. Easiest first:

**PWABuilder (no tools to install, ~5 minutes)**
1. Go to <https://www.pwabuilder.com>, paste your hosted URL.
2. **Package for stores → Android → Download package.**
3. Inside you get a signed `.apk` / `.aab` plus `signing.keystore` — **keep that keystore
   file forever**; every future update must be signed with the same one.
4. Side-load the `.apk` (or upload the `.aab` to Play Console).

The wrapper is a Trusted Web Activity: it loads your hosted URL, so **the app still
auto-updates without a new APK** for anything inside the web app. You only rebuild the APK
when the icon, name or permissions change.

**Capacitor (if you want native code later)**
```bash
npm init -y && npm i @capacitor/core @capacitor/cli @capacitor/android
npx cap init "Delivery Log" com.yourcompany.delivery --web-dir=.
npx cap add android
npx cap open android      # Android Studio → Build → Generate Signed Bundle / APK
```
Add to `android/app/src/main/AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.INTERNET"/>
```

---

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app — UI, logic, Firebase calls |
| `config.js` | Your Firebase keys and master email (the only file you edit) |
| `sw.js` | Service worker: offline shell + auto-update |
| `manifest.json` | Makes it installable |
| `firestore.rules` | Who may read/write which records (already published) |
| `storage.rules` | Kept for if you ever upgrade to Cloud Storage |
| `icon-*.png` | App icons (192, 512, maskable) |

## Data shape

```
users/{uid}    { name, phone, email, vehicle, role: "master" | "delivery", createdAt }
entries/{id}   { uid, userName, userEmail, shop, notes,
                 stockPhoto, billPhoto, lat, lng, accuracy, mapUrl, createdAt }
photos/{id}    { uid, data (JPEG data URL), createdAt }
config/app     { version, apkUrl }
```

## Keeping inside the free tier

- The master feed loads only the **last 30 days**, so the cost of opening it stays flat no
  matter how much history builds up. Opening a person from the People tab fetches that
  person's full history on demand.
- **Profile → Free up space** (master only) removes photos older than 1, 3 or 6 months.
  Entries, GPS and notes are always kept — only the pictures go. It counts first and asks
  you to tap again before deleting anything.
- Free limits that matter: **1 GB stored** (~2,500 photos, never resets), 50,000 reads/day
  and 20,000 writes/day (reset daily, ~12:30pm IST). One upload costs 3 writes.

## Notes and limits

- Photos are resized to 1280px / JPEG 0.72 in the browser, and compressed further if a
  photo would still exceed ~700 KB.
- If a delivery person denies location permission, the upload still saves and is clearly
  marked **no location** — so you can see who is skipping it.
- Deleting uploads is master-only; delivery persons cannot remove their own records.
- The Firebase web `apiKey` is not a secret — it identifies the project. The security
  rules in step 3 are what actually protect the data.
