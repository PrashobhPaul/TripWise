# TripWise

Auto-book your commute, and share live location with the people *you* choose.
One codebase → two installable outputs: an installable **PWA** (works today) and a
sideloadable **APK** (adds native background geofencing). No Play Store.

## What ships how

| Need | PWA | APK (Capacitor) |
|---|---|---|
| Install on any phone (you, Divya, Amma, friends) | Add to Home Screen | Sideload the .apk |
| Notifications when app is closed | Web Push (Android; iOS 16.4+ if installed) | FCM |
| Light / dark theme | ✅ | ✅ |
| Live location share + revoke | ✅ | ✅ |
| **Background geofence auto-trigger** (no touching the phone) | ❌ (web can't) | ✅ |

So: everyone runs the PWA to *receive* trips and share their own; the phone that needs
the hands-free commute trigger runs the APK.

## Ship the PWA (instant, no build tools)
```bash
npm install
npm run build      # outputs dist/
```
Deploy `dist/` to GitHub Pages / Vercel / Netlify. On each device open the URL →
browser menu → **Add to Home Screen**. Done.

## Ship the APK (sideload, native geofence)
Push this repo to GitHub. The included workflow `.github/workflows/build-apk.yml`
builds it on every push to `main` — open the run → download the **tripwise-apk**
artifact → sideload it. That's your phone-only pipeline; no local Android setup.

For a signed release build, add a keystore + the signing secrets and switch the
Gradle task to `assembleRelease`.

## Configure

Set these (repo secrets for CI, `.env` for local `npm run dev`):
```
VITE_SUPABASE_URL=...
VITE_SUPABASE_ANON_KEY=...
VITE_VAPID_PUBLIC_KEY=...   # PWA web push only
```

**Backend:** run `supabase/schema.sql`, then
`supabase functions deploy tripwise --no-verify-jwt` and set its secrets
(see comments in `supabase/functions/tripwise/index.ts`).

**Push:**
- APK → add `google-services.json` under `android/app/` after `npx cap add android`, and turn on FCM in your Firebase project.
- PWA → generate VAPID keys, set `VITE_VAPID_PUBLIC_KEY`, and have the Edge Function send Web Push with the private key.
- `profiles.fcm_token` (native) and `profiles.web_push` (web) store the per-device targets so the backend fans out selectively to the recipients you pick per route/journey.

## Theme
Toggle (sun/moon) in the top bar. Persists per device — your deep-navy, their bright/light. Defaults to dark.

## Structure
```
src/App.jsx            app UI (themed)
src/styles.js          dark (navy) + light palettes
src/theme.js           theme persistence
src/push.js            FCM (native) + Web Push (PWA) registration
src/supabase.js        client
src/send/whatsapp.ts   deeplink + optional DIY auto-send (non-app contacts)
supabase/schema.sql    multi-tenant schema + 3-layer consent RLS
supabase/functions/    trip lifecycle + live feed Edge Function
capacitor.config.ts    APK config
.github/workflows/     APK build CI
```

## What's real vs stubbed in this drop
- Real: full themed UI, consent-model schema + RLS, Edge Function, send layer, push registration, PWA install + service-worker push, APK build pipeline.
- To wire on-device: the native geofence plugin actions (`@capacitor-community/background-geolocation`) and the NotificationListener that reads Uber's confirmation. These only run in the APK on a real device — hooks are in place; the handlers are the next build step.

---

## Native geofencing (the "trigger without opening the app" piece)

`src/geofence.js` registers your five geofences and, on a transition, (1) POSTs the
event to Supabase so the backend FCM-pushes the recipients you picked, and (2) raises
a tap-to-book Uber notification.

**Plugin:** `@transistorsoft/capacitor-background-geolocation` — reliable native
geofencing; free for the debug/sideload APK you build here (a license is only needed
for a Play Store *release*). Free alternative: `cordova-plugin-geofence`.

**Extra env:**
```
VITE_TRIPWISE_FN=https://<project-ref>.supabase.co/functions/v1/tripwise
VITE_TRIPWISE_SECRET=<same MACRODROID_SECRET you set on the Edge Function>
```

**Android permissions** (Manifest, after `npx cap add android`):
`ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `FOREGROUND_SERVICE`,
`FOREGROUND_SERVICE_LOCATION`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`.
On first run, grant Location = **Allow all the time / Precise** and allow notifications.

**Reality check for trial 1 (especially on Samsung):**
- Android won't let the app foreground Uber from the background — the geofence raises a
  notification; you tap it to open Uber. Booking is one tap by design. The recipient
  push is the fully-automatic part.
- Samsung aggressively kills background services. Set the app to **Unrestricted** battery,
  turn off **"Manage app if unused"**, and (Device care → Background usage limits) make
  sure the app isn't put to Sleep. Expect a round or two of on-device tuning — background
  geofencing is never perfect on the first install on any OEM.

---

## Offline-capable map (South India: TG, AP, KA, TN, KL)

`src/LiveMap.jsx` renders a hosted PMTiles basemap and plots a journey's coordinates
(marker + amber trail) live from Supabase Realtime. Only coordinates stream between
users — the map is served by range requests, not bundled.

**1. Build the extract** (pulls only South India from the Protomaps planet — check
current Protomaps docs for the planet build URL):
```
pmtiles extract <PLANET_URL> south-india.pmtiles --bbox=74.0,8.0,84.9,19.95 --maxzoom=14
```
Cap at z14 for road-level (z13 ~halves the size; fine for movement tracking).

**2. Host it** on Cloudflare R2 (free: 10 GB storage, no egress). Make the object
public (or front it with a Worker). Enable CORS + HTTP range requests.

**3. Point the app at it:**
```
VITE_PMTILES_URL=https://<r2-public-domain>/south-india.pmtiles
```

The app fetches only the tiles in view and caches them, so any area you've actually
travelled becomes available offline afterwards. Per-trip "download this corridor for
offline" (e.g. Hyderabad–Thrissur) is a clean V2 on the same file.

---

## Map coverage strategy (major cities + corridors)

Two layers, both from the same hosted `south-india.pmtiles`:

1. **Live map (default):** hosted at z14, range-requested. Users only ever download
   the tiles along the route they actually travel, cached as they go — so "just the
   corridors and cities you use" happens automatically, no pre-carving needed.
2. **Offline packs (`src/packs.js`):** the major metro cities (z15) and intercity
   corridors (z12) users can pre-download for true offline use — metro interiors and
   highway dead zones where fetch-as-you-move fails. City packs are a bbox; corridor
   packs are waypoints the pack manager buffers (~4 km) and tiles along.

Note: pre-caching a metro city fixes the *map* offline, not GPS. Hyderabad & Kochi
metros are elevated (GPS tracks); Chennai & Bengaluru have underground stretches where
the position pauses until it resurfaces.

TODO (next build): the pack download manager that warms the tile cache for a pack's
bbox/corridor from the hosted PMTiles.

---

## Offline pack download manager (`src/packManager.js`)

Slices a smaller, self-contained set of tiles for a pack out of the hosted
`south-india.pmtiles` and stores them on **Capacitor Filesystem** (survives offline
reliably; not evicted like the WebView cache).

- **City packs** → tiles for the bbox, z0..15.
- **Corridor packs** → tiles for a ~4 km buffer *along* the waypoints (the road ribbon,
  not the empty rectangle around a diagonal route).
- Tiles are **decompressed once at download** (Protomaps ships gzip MVT) and stored raw,
  so the offline serve path is a plain filesystem read — no per-tile gunzip at runtime.

`LiveMap` registers a single `tw://` protocol that resolves **installed pack tiles first,
hosted PMTiles second** — so online and offline share one code path and the same map
style. Nothing to swap when you go offline; a downloaded corridor just works.

API: `downloadPack(pack, onProgress)`, `listInstalledPacks()`, `deletePack(id)`.
Wire these to a "Downloads" screen listing `CITY_PACKS` + `CORRIDOR_PACKS` from `packs.js`.

**Verify on device:** tile decompression (gzip → raw MVT) and Filesystem base64 round-trip
are the two things to confirm on real hardware — they can't be exercised in a web preview.
