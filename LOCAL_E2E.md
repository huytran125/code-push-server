# CodePush Server — Local End-to-End Guide

Run the CodePush server **entirely on your machine** — no Azure cloud account, no OAuth
provider — and drive it with the CLI and a React Native app.

This guide covers the full loop:

```
Azurite (local storage)  ◀──  CodePush Server  ◀──  CLI (release updates)
                                      ▲
                                      └──────────  React Native app (checks for updates)
```

---

## Contents

1. [Prerequisites](#1-prerequisites)
2. [How auth/storage work locally](#2-how-authstorage-work-locally)
3. [Configure `api/.env`](#3-configure-apienv)
4. [Build](#4-build)
5. [Run the server](#5-run-the-server)
6. [Set up & log in the CLI (no OAuth)](#6-set-up--log-in-the-cli-no-oauth)
7. [Create an app & release an update](#7-create-an-app--release-an-update)
8. [Verify the update is downloadable](#8-verify-the-update-is-downloadable)
9. [Configure the React Native mobile app](#9-configure-the-react-native-mobile-app)
10. [Troubleshooting](#10-troubleshooting)
11. [Known limitations (local only)](#11-known-limitations-local-only)
12. [Teardown](#12-teardown)

---

## 1. Prerequisites

| Tool | Notes |
|------|-------|
| Node.js | v18+ (the project targets 18; newer majors boot fine) |
| npm | v9+ |
| Azurite | The Azure Storage **emulator** — runs locally, needs no Azure account. Used via `npx`, no install required. |

No Docker, no Azure subscription, no GitHub/Microsoft OAuth app needed.

---

## 2. How auth/storage work locally

Two facts shape the local setup:

- **The server needs a storage backend.** Locally we use **Azurite** (the emulator),
  selected via `EMULATED=true`. Data persists in `api/azurite-data/`.
- **Every management/CLI call needs a bearer *access key*.** Normally you get one by
  logging in through OAuth in a browser. To skip OAuth, this repo includes two
  **dev-only, env-gated hooks** in [`api/script/default-server.ts`](api/script/default-server.ts)
  (both off by default — production is unaffected):

  | Env var | Effect |
  |---------|--------|
  | `USE_JSON_STORAGE=true` | Use a zero-dependency in-memory backend instead of Azure/Azurite. Great for quick API testing; **uploads can't be downloaded** (see [limitations](#11-known-limitations-local-only)). |
  | `SEED_LOCAL_KEY=<key>` | On boot, create a `dev@local.test` account + this fixed access key, so the CLI can `login --accessKey <key>` without OAuth. |

> **Two ways to run locally:**
> - **Azurite (recommended)** — uploads are real and downloadable. Use this for the full loop incl. mobile.
> - **JsonStorage (`USE_JSON_STORAGE=true`)** — instant, no emulator, but the package blob layer is broken, so it's only for exercising the API/CLI, not for actually downloading bundles.
>
> The rest of this guide uses **Azurite**.

---

## 3. Configure `api/.env`

Copy the example and set these values:

```bash
cd api
cp .env.example .env   # if you don't already have a .env
```

```ini
# --- storage: use the local emulator ---
EMULATED=true

# --- auth: skip OAuth, seed a local access key on boot ---
DEBUG_DISABLE_AUTH=false
SEED_LOCAL_KEY=localdevaccesskey1234567890

# --- server ---
SERVER_URL=http://localhost:3000

# Leave OAuth + Redis EMPTY:
# GITHUB_CLIENT_ID=        (empty)
# MICROSOFT_CLIENT_ID=     (empty)
# REDIS_HOST=              (empty)
```

> ⚠️ `DEBUG_DISABLE_AUTH` **must be `false`**. When it's `true` the `/authenticated`
> route the CLI checks isn't mounted, and the access-key login fails.
>
> ⚠️ The access key must be 10–100 chars, `[A-Za-z0-9_-]` only.
> `localdevaccesskey1234567890` is fine.

---

## 4. Build

```bash
cd api
npm install
npm run build      # compiles TS -> ./bin and copies the EJS views
```

(Only needed again if you change server source.)

---

## 5. Run the server

**Terminal 1 — Azurite** (leave running):

```bash
cd api
npx azurite --silent --location ./azurite-data
```

Listens on `10000` (blob) / `10001` (queue) / `10002` (table).

**Terminal 2 — the server** (from `api/`, leave running):

```bash
npm run start:env
```

You should see:

```
[seed] Local access key ready for dev@local.test: localdevaccesskey1234567890
No REDIS_HOST or REDIS_PORT environment variable configured.
API host listening at http://localhost:3000
```

Sanity check (use `/`, **not** `/health` — see [troubleshooting](#10-troubleshooting)):

```bash
curl http://localhost:3000/
# -> Welcome to the CodePush REST API!
```

---

## 6. Set up & log in the CLI (no OAuth)

**Build & link the CLI** (once):

```bash
cd cli
npm install
npm run build
npm link            # makes the `code-push-standalone` command available
```

**Log in with the seeded key** (once per machine, or after `logout`):

```bash
code-push-standalone login http://localhost:3000 --accessKey localdevaccesskey1234567890
# -> Successfully logged-in. Your session file was written to ~/.code-push.config
```

No browser, no OAuth. The session is saved to `~/.code-push.config`.

---

## 7. Create an app & release an update

```bash
# Create an app (auto-creates Staging + Production deployments)
code-push-standalone app add MyApp-Android

# See the deployment keys (you'll need the key for the mobile app)
code-push-standalone deployment ls MyApp-Android -k
```

Release a plain folder of assets (generic command):

```bash
code-push-standalone release MyApp-Android ./path/to/bundle 1.0.0
#                            <appName>      <contents>       <targetBinaryVersion>
```

Or, for a real React Native bundle (runs the RN bundler):

```bash
code-push-standalone release-react MyApp-Android android
code-push-standalone release-react MyApp-iOS     ios
```

`targetBinaryVersion` (`1.0.0` above) must match the **native app version**
(`versionName` on Android / `CFBundleShortVersionString` on iOS) the update targets.

---

## 8. Verify the update is downloadable

Simulate what a client device does — an update check:

```bash
STAGING_KEY=$(code-push-standalone deployment ls MyApp-Android -k | grep -i staging | grep -oE '[A-Za-z0-9_-]{28,}')

curl "http://localhost:3000/updateCheck?deploymentKey=$STAGING_KEY&appVersion=1.0.0&clientUniqueId=test1"
```

You'll get JSON with `"isAvailable":true` and a real `downloadURL` like
`http://127.0.0.1:10000/devstoreaccount1/storagev2/<blobId>`. Fetch it to confirm the
bundle is actually served:

```bash
curl -s -o /tmp/update.zip -w "%{http_code} %{size_download} bytes\n" "<downloadURL from above>"
# -> 200 <N> bytes
```

If you get `200` and a non-zero size, the full upload → download loop works.

---

## 9. Configure the React Native mobile app

Your app uses [`react-native-code-push`](https://github.com/microsoft/react-native-code-push).
Two things must be set: the **server URL** (point it at your local server) and the
**deployment key** (from step 7).

### 9.1 Install the client SDK

```bash
# in your React Native project
npm install react-native-code-push
# iOS
cd ios && pod install && cd ..
```

### 9.2 Wire it up in JS

```jsx
import codePush from "react-native-code-push";

const codePushOptions = {
  // Check on resume; for local testing you can also call codePush.sync() manually.
  checkFrequency: codePush.CheckFrequency.ON_APP_RESUME,
  installMode: codePush.InstallMode.ON_NEXT_RESUME,
};

function App() {
  // ... your app ...
}

export default codePush(codePushOptions)(App);
```

Force a check during testing (shows a dialog, installs immediately):

```js
codePush.sync({
  updateDialog: true,
  installMode: codePush.InstallMode.IMMEDIATE,
});
```

### 9.3 Pick the right server host for your target

`localhost` on a device/emulator means *the device itself*, not your Mac. Use:

| Target | Server URL |
|--------|-----------|
| **iOS simulator** | `http://localhost:3000` |
| **Android emulator** | `http://10.0.2.2:3000` |
| **Physical device** (same Wi-Fi) | `http://<your-Mac-LAN-IP>:3000` |

Find your LAN IP on macOS: `ipconfig getifaddr en0`.

### 9.4 Android config

`android/app/src/main/res/values/strings.xml`:

```xml
<resources>
    <string name="app_name">MyApp</string>
    <!-- deployment key from: code-push-standalone deployment ls MyApp-Android -k -->
    <string moduleConfig="true" name="CodePushDeploymentKey">YOUR_STAGING_KEY</string>
    <!-- local server (emulator host shown) -->
    <string moduleConfig="true" name="CodePushServerUrl">http://10.0.2.2:3000</string>
</resources>
```

**Allow cleartext HTTP** (Android blocks it by default on API 28+). Create
`android/app/src/main/res/xml/network_security_config.xml`:

```xml
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
        <domain includeSubdomains="true">localhost</domain>
        <!-- add your LAN IP here for physical devices, e.g. 192.168.1.50 -->
    </domain-config>
</network-security-config>
```

Reference it in `android/app/src/main/AndroidManifest.xml` on the `<application>` tag:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ... >
```

### 9.5 iOS config

`ios/<App>/Info.plist`:

```xml
<key>CodePushDeploymentKey</key>
<string>YOUR_STAGING_KEY</string>
<key>CodePushServerURL</key>
<string>http://localhost:3000</string>
```

**Allow cleartext HTTP** via an App Transport Security exception in the same `Info.plist`:

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

> Note the key casing differs by platform: Android uses `CodePushServerUrl`,
> iOS uses `CodePushServerURL`.

### 9.6 ⚠️ Download reachability caveat

The server returns a `downloadURL` pointing at the Azurite blob endpoint:
`http://127.0.0.1:10000/...`. That host is hardcoded by the emulator
(`UseDevelopmentStorage=true`), so:

| Target | Update check | Bundle download |
|--------|:---:|:---:|
| **iOS simulator** | ✅ | ✅ (shares the Mac's loopback) |
| **Android emulator** | ✅ | ❌ `127.0.0.1` = the emulator itself |
| **Physical device** | ✅ | ❌ `127.0.0.1` unreachable |

**For a full OTA test today, use the iOS simulator.** To make Android/devices work you
need to make blob URLs use a reachable host (a code change to rewrite the URL to your
LAN IP, or moving storage to MinIO/S3 with presigned URLs).

---

## 10. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| CLI `login` returns **401** | `DEBUG_DISABLE_AUTH` is `true` → set it to `false` and restart. The `/authenticated` route only mounts when it's `false`. |
| `GET /health` returns **500** | The health route fails when Redis isn't configured. **Expected locally** — use `GET /` to check liveness instead. |
| Server ignores your Azure creds / uses emulator unexpectedly | `EMULATED` is truthy for *any* non-empty string (even `"false"`). Set exactly `EMULATED=true` for local, and **remove/blank it** for real Azure. |
| Server crashes reading `./certs/cert.key` | `HTTPS` is truthy for any non-empty string. Leave `HTTPS` **empty** (not `false`) for local HTTP. |
| `release` works but client can't download | You're on `USE_JSON_STORAGE=true` (broken blob URLs) — switch to Azurite, or it's the `127.0.0.1` mobile caveat above. |
| CLI says "already logged in" when switching servers | Run `code-push-standalone logout` first, then `login` to the new URL. |

---

## 11. Known limitations (local only)

- **`USE_JSON_STORAGE=true`** stores blobs in memory as strings and returns a malformed
  download URL — fine for API/CLI testing, **not** for downloading bundles. Use Azurite
  for anything involving real packages.
- **Azurite download URLs are `127.0.0.1`** — only reachable from the host / iOS simulator
  (see §9.6).
- The seed hooks (`USE_JSON_STORAGE`, `SEED_LOCAL_KEY`) and the open CLI access key are
  **for local development only**. Never set them in a deployed environment.
- `/health` depends on Redis; don't wire it as a health probe unless Redis is running.

---

## 12. Teardown

```bash
# stop the server and Azurite with Ctrl-C in their terminals
code-push-standalone logout          # clears ~/.code-push.config
rm -rf api/azurite-data              # wipe local storage data (optional)
```

To start fresh next time, just repeat from [step 5](#5-run-the-server) — the
`SEED_LOCAL_KEY` hook recreates the account/key on every boot, and Azurite reloads
`azurite-data/` if you kept it.
