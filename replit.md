# SMS Manager

## Overview
Originally imported from GitHub as a loose collection of Flutter `.dart`
files (contacts + native Android SMS sending) with no `pubspec.yaml`/`lib/`
structure and no way to run on Replit (no Flutter SDK, no Android
emulator/SIM here). Per the user's direction, it was rebuilt as a web app
first, and is now prepared to be packaged as an offline Android APK via
Capacitor.

## Current state: Offline-first web app (active, runnable, Capacitor-ready)
A static frontend (HTML/CSS/JS) with a contacts list and CSV import, plus
sequential bulk "sending" with a live per-contact status. **All data and
logic now live entirely in the browser** (`localStorage`, client-side CSV
parsing) — there is no backend API and no server-side storage. This was a
deliberate architecture change so the exact same `public/` folder can be
bundled into a Capacitor Android app and work fully offline (no Node server
runs on the phone).

### Structure
```
server/
  index.js       # Dev-only static file server for Replit preview (no API, no logic)
public/
  index.html     # RTL Arabic UI
  style.css
  storage.js     # ContactsStore: localStorage-backed CRUD (replaces old server DB)
  csvParser.js   # Client-side CSV parsing (English/Arabic headers)
  smsBridge.js   # sendSms(): calls a native Capacitor plugin on Android, else "not connected"
  app.js         # UI wiring; bulk sequential send loop (3s gap between sends)
capacitor.config.json  # webDir: "public" — Capacitor packages this folder as-is
docs/BUILD_APK_AR.md   # Arabic step-by-step guide to build the APK outside Replit
```

### Why client-side storage instead of a server/database
Two reasons compound here: `better-sqlite3` needs a native build step
(node-gyp + Python) unavailable in this environment (see
`.agents/memory/native-npm-build-deps.md`), and — more importantly — a
Capacitor Android app doesn't run a Node backend at all, so any
server-dependent design would break "offline" entirely. `localStorage` in
the packaged app's webview solves both: it's per-device, works with zero
network, and needs no native build step to use.

### Running it (Replit preview)
The `Start application` workflow runs `npm run dev` (`node server/index.js`),
a plain static file server on port 5000 — used only to preview/test the UI
here; it plays no role in the packaged Android app.

### Bulk sequential sending (queue + 3s delay)
- `public/smsBridge.js` — `sendSms(contact, message)`: inside a browser
  (e.g. this Replit preview) always returns `NOT_CONNECTED` (no native SMS
  API exists there). Inside a Capacitor Android build, it looks for a native
  plugin registered as `window.Capacitor.Plugins.SmsSender` and calls
  `.send({ phone, message })`; see `docs/BUILD_APK_AR.md` for how to install
  or write that plugin.
- `public/app.js` (`sendBulkSequential`) drives the queue: select contacts
  via checkboxes ("تحديد الكل" to select all), write a message, click
  "بدء الإرسال المتتابع" — it calls `sendSms` for each selected contact **in
  sequence**, waiting 3 seconds after each result before the next, with live
  per-contact status (قيد الانتظار / جارٍ الإرسال / تم الإرسال / فشل).
- In this Replit preview every attempt shows "غير متصل" — expected, since
  there is no native plugin here. Only a real Android build can send real
  SMS.

### App icon
`resources/icon.png` (1024×1024, square-cropped from the user's uploaded
photo) is the source icon for Capacitor's icon generator
(`npx capacitor-assets generate --android`, see `docs/BUILD_APK_AR.md`).
The same image is already resized into `public/icons/` and wired up in
`index.html` as the favicon/apple-touch-icon for the Replit preview.

### Converting to an Android APK (offline)
Full bilingual-technical walkthrough in `docs/BUILD_APK_AR.md` (Arabic),
covering: installing Node/Android Studio/JDK locally, `npx cap add android`,
`npx cap sync`, installing or writing a native SMS-sending plugin, adding the
`SEND_SMS` permission, and building the APK in Android Studio. This must be
done outside Replit — there is no Android SDK/Gradle/emulator here, so the
APK itself cannot be produced or tested in this environment.

### Known gaps / next steps
- Real SMS sending only becomes possible once the user builds the Android
  APK and wires up a native plugin per `docs/BUILD_APK_AR.md` — cannot be
  verified inside Replit.
- No authentication — anyone using the device can see/add/delete contacts
  (acceptable for a personal single-device APK; would need rethinking for
  multi-user use).
- Contacts stored per-browser/per-device (`localStorage`); there is no sync
  between the Replit preview and an installed APK, or between two devices.

## Legacy: original Flutter code (not runnable here)
The original Flutter source was reorganized into a valid project structure
under `lib/` (`main.dart`, `models/`, `services/`, `views/`) with a
`pubspec.yaml`, but it cannot run in Replit: no Flutter SDK module exists
here, there are no generated `android/`/`ios/` platform folders, and its
dependencies (`sqflite`, `telephony`) require a real/emulated mobile device.
Kept as reference/history; the Capacitor-based web app above is the active
path toward an Android APK.

## User preferences
- Prefers building incrementally.
- Wants all communication in Arabic, and code comments/notes written in
  Arabic as well.
