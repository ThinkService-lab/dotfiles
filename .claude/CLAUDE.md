# Global App Architecture — Agent Instructions

You are building a cross-platform app (iOS, Android, Web) using the standard architecture defined below.
Read this file fully before writing any code, creating any file, or running any command.
This architecture is non-negotiable. Do not deviate from it unless the per-project CLAUDE.md explicitly overrides a section.

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | React Native + Expo (TypeScript) |
| Routing | Expo Router (file-based) |
| Backend / Database | Supabase (Postgres) |
| Auth | Supabase Auth |
| Server Logic | Supabase Edge Functions (Deno/TypeScript) |
| Storage | Supabase Storage (optional — only if your app handles file/image uploads) |
| State — server | TanStack React Query |
| State — local | Zustand |
| Payments — mobile | RevenueCat |
| Payments — web | Stripe |
| Push Notifications | Expo Notifications (APNs + FCM) |
| Crash Reporting | Sentry |
| Analytics | PostHog |
| Build + CI/CD | EAS Build + EAS Submit + GitHub Actions |
| Web Hosting | Vercel |
| Package Manager | pnpm (never npm or yarn) |

---

## Absolute Rules

These rules apply to every file, every command, every decision.

- Always use `pnpm`. Never `npm install` or `yarn`.
- Always use TypeScript. Never plain JavaScript.
- Always use Expo Router for navigation. Never React Navigation standalone.
- Always use `expo-sqlite/localStorage/install` as the Supabase auth storage adapter (official Supabase recommendation). Do not use a raw `expo-secure-store` adapter — SecureStore has a 2KB limit that JWT tokens exceed, causing silent auth failures.
- Always enable Row-Level Security (RLS) on every Supabase table immediately after creating it.
- Always use migration files for schema changes (`supabase migration new`). Never edit the database directly in the Supabase dashboard.
- Always regenerate TypeScript types after every migration: `supabase gen types typescript --linked > src/types/database.ts`
- Never put `SUPABASE_SERVICE_ROLE_KEY` in the app bundle or any `EXPO_PUBLIC_*` variable.
- Never commit `.env.local`. Always verify it is in `.gitignore` before the first commit.
- Never use unicode bullet characters (`•`) in code — use proper list syntax.
- Never use `\n` inside docx generation — use separate elements.

---

## Phase 1 — Prerequisites

Before initialising a project, verify all tools are installed:

```bash
node --version        # must be LTS via nvm
pnpm --version
expo --version
eas --version
supabase --version
```

If any are missing:

```bash
# Node via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install --lts && nvm use --lts

# Expo + EAS
npm install -g expo-cli eas-cli

# Supabase CLI
npm install -g supabase

# pnpm
npm install -g pnpm
```

Required accounts (verify they exist before starting):
- Apple Developer Program — https://developer.apple.com/programs/ ($99/year)
- Google Play Console — https://play.google.com/console ($25 one-time)
- Supabase — https://supabase.com
- Expo / EAS — https://expo.dev
- RevenueCat — https://www.revenuecat.com
- Sentry — https://sentry.io
- PostHog — https://posthog.com
- Vercel — https://vercel.com

---

## Phase 2 — Project Initialisation

```bash
npx create-expo-app@latest <AppName> --template
# Select: Blank (TypeScript)

cd <AppName>
eas init
```

Update `app.json` immediately. Set these fields before any other work:
- `expo.name`
- `expo.slug`
- `expo.scheme`
- `expo.ios.bundleIdentifier` — format: `com.yourcompany.appname`
- `expo.android.package` — same value as bundleIdentifier

> **WARNING:** The bundle identifier cannot be changed after submission to the App Store. Set it correctly now.

Install all standard dependencies:

```bash
pnpm add react-native-url-polyfill
# Use npx expo install for native packages — Expo pins the correct compatible version
npx expo install @supabase/supabase-js expo-sqlite expo-secure-store expo-file-system expo-image-picker
pnpm add expo-router expo-constants expo-linking expo-status-bar
pnpm add @tanstack/react-query zustand
pnpm add react-native-purchases
pnpm add expo-notifications expo-device
pnpm add @sentry/react-native
pnpm add -D typescript @types/react @types/react-native
```

---

## Phase 3 — Folder Structure

Always use the `src/app` layout — not `app/` at the root. Both are supported by Expo Router, but `src/app` is the officially recommended pattern (Expo Engineering blog, Sep 2025) because it cleanly separates app source code from config files at the root.

Reference: https://expo.dev/blog/expo-app-folder-structure-best-practices

```
<AppName>/
├── src/
│   ├── app/                        ← Expo Router entry point (every file = a route)
│   │   ├── (auth)/                 ← Route group: excluded from URL
│   │   │   ├── _layout.tsx
│   │   │   ├── login.tsx
│   │   │   ├── signup.tsx
│   │   │   └── forgot-password.tsx
│   │   ├── (app)/                  ← Route group: protected screens
│   │   │   ├── _layout.tsx         ← Auth guard lives here
│   │   │   ├── index.tsx           ← Home (matches /)
│   │   │   └── profile.tsx
│   │   ├── api/                    ← Expo Router API routes (server-side, +api suffix)
│   │   │   └── webhook+api.ts
│   │   ├── _layout.tsx             ← Root layout (providers, fonts, Sentry, RevenueCat)
│   │   └── +not-found.tsx
│   ├── screens/                    ← Screen components (route files import from here)
│   │   ├── home/
│   │   │   ├── components/         ← Screen-specific sub-components
│   │   │   └── index.tsx
│   │   └── profile/
│   │       └── index.tsx
│   ├── components/                 ← Shared, reusable UI components
│   ├── hooks/                      ← Custom React hooks
│   ├── lib/                        ← Third-party client setup
│   │   ├── supabase.ts
│   │   ├── auth.tsx
│   │   └── sentry.ts
│   ├── server/                     ← Server-only utilities (used by api/ routes)
│   │   └── db.ts
│   ├── services/                   ← Data fetching and API call functions
│   ├── stores/                     ← Zustand global state
│   ├── types/
│   │   └── database.ts             ← Auto-generated by Supabase CLI, never hand-edit
│   └── utils/                      ← Pure utility functions (date formatters, etc.)
├── assets/                         ← Images, fonts, icons
├── supabase/
│   └── migrations/                 ← All schema changes go here, never edit DB directly
├── .env.local                      ← Never commit
├── .gitignore                      ← Verify .env.local is listed before first commit
├── app.json
├── eas.json
└── tsconfig.json
```

**Route files are thin wrappers.** Keep logic out of `src/app/`. A route file should only handle URL params and render a screen:

```typescript
// src/app/(app)/index.tsx
import { Home } from '@/screens/home';
export default function HomeRoute() {
  return <Home />;
}
```

**Platform-specific files.** When a component needs different implementations per platform, use file extensions — Metro picks the right one automatically:

```
src/components/bar-chart.tsx        ← default (native)
src/components/bar-chart.web.tsx    ← web override
```

Import as normal: `import { BarChart } from '@/components/bar-chart'`

Set path aliases in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

---

## Phase 4 — Supabase Project Setup

```bash
supabase login
supabase link --project-ref <PROJECT_REF>
supabase init
```

Create `src/lib/supabase.ts`:

```typescript
import 'react-native-url-polyfill/auto';
import 'expo-sqlite/localStorage/install';
import { createClient } from '@supabase/supabase-js';
import { AppState } from 'react-native';

export const supabase = createClient(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
  {
    auth: {
      storage: localStorage,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);

// Required: refresh token when app returns to foreground
AppState.addEventListener('change', (state) => {
  if (state === 'active') {
    supabase.auth.startAutoRefresh();
  } else {
    supabase.auth.stopAutoRefresh();
  }
});
```

> **Key name change:** Supabase has deprecated `ANON_KEY` in favour of a new `PUBLISHABLE_KEY` (`sb_publishable_xxx`). Use `EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY` in your `.env.local`. The legacy `ANON_KEY` still works during transition but will be removed. Get the new key from: Supabase Dashboard → Project Settings → API Keys → Publishable key.

> **On `expo-sqlite/localStorage/install`:** This is the recommended storage adapter from Supabase's official Expo quickstart (2025). It uses Expo's localStorage polyfill backed by SQLite — suitable for most apps.

> **If you need encrypted session storage:** Do NOT use a simple `expo-secure-store` adapter directly. SecureStore has a 2KB value size limit and JWT tokens exceed it, causing silent auth failures. Use the `LargeSecureStore` class from the official Supabase tutorial which combines AES encryption (`aes-js`) with AsyncStorage. Install: `npx expo install expo-secure-store aes-js react-native-get-random-values`. Reference: https://supabase.com/docs/guides/getting-started/tutorials/with-expo-react-native

Docs: https://supabase.com/docs/guides/getting-started/quickstarts/expo-react-native

---

## Phase 5 — Authentication

Enable providers in Supabase Dashboard → Authentication → Providers:
- Email (default on)
- Google OAuth
- Apple OAuth

> **Apple Rule:** If the iOS app offers ANY third-party sign-in (Google, Facebook, etc.), Sign in with Apple is MANDATORY. Apple will reject the app without it. Reference: App Store Review Guidelines 4.8.

Create `src/lib/auth.tsx` — AuthProvider with session management and `useAuth` hook.
Wrap the root `app/_layout.tsx` with `<AuthProvider>`.

In `app/(app)/_layout.tsx` — redirect unauthenticated users to `/(auth)/login`.

Docs: https://supabase.com/docs/guides/auth

---

## Phase 6 — Database Schema & Row-Level Security

Always create tables via migrations:

```bash
supabase migration new <migration_name>
# Write SQL in the generated file
supabase db push
```

Every new table must immediately have RLS enabled:

```sql
ALTER TABLE public.<table_name> ENABLE ROW LEVEL SECURITY;
```

Then add explicit policies. A table with RLS enabled but no policies denies all access by default.

**RLS rules from official Supabase docs:**
- Never use `FOR ALL`. Write separate policies for SELECT, INSERT, UPDATE, DELETE.
- Always add `TO authenticated` — never rely on `auth.uid()` alone to exclude the `anon` role.
- Wrap auth functions in `(select auth.uid())` — this caches per query, not per row (major performance gain on large tables).
- SELECT / UPDATE use `USING`. INSERT uses `WITH CHECK`. UPDATE uses both.

Always create the profiles table first:

```sql
CREATE TABLE public.profiles (
  id          UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  email       TEXT UNIQUE NOT NULL,
  full_name   TEXT,
  avatar_url  TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- SELECT: users can read their own profile only
CREATE POLICY "Users can view own profile"
  ON public.profiles
  FOR SELECT
  TO authenticated
  USING ((select auth.uid()) = id);

-- UPDATE: users can update their own profile only
CREATE POLICY "Users can update own profile"
  ON public.profiles
  FOR UPDATE
  TO authenticated
  USING ((select auth.uid()) = id)
  WITH CHECK ((select auth.uid()) = id);

-- INSERT is handled by the trigger below — no client INSERT policy needed.
-- If you ever allow client-side inserts, add:
-- CREATE POLICY "Users can insert own profile"
--   ON public.profiles FOR INSERT TO authenticated
--   WITH CHECK ((select auth.uid()) = id);

-- Auto-create profile on signup (runs with SECURITY DEFINER to bypass RLS)
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name)
  VALUES (NEW.id, NEW.email, NEW.raw_user_meta_data->>'full_name');
  RETURN NEW;
END;
$$;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

After every migration, regenerate types:

```bash
supabase gen types typescript --linked > src/types/database.ts
```

Audit query — run before every deployment:

```sql
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';
```
Every row must show `rowsecurity = true`.

Docs:
- RLS guide: https://supabase.com/docs/guides/database/postgres/row-level-security
- RLS performance: https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv
- Securing your API: https://supabase.com/docs/guides/api/securing-your-api

---

## Phase 7 — Storage (OPTIONAL)

**Only implement this phase if your app requires file or image uploads** (e.g. profile photos, document uploads, user-generated media). Most apps do not need Storage in v1. Skip this phase if your app only stores structured data in the database.

On the Supabase free tier: 1 GB file storage included. Storage egress counts toward your 10 GB bandwidth limit.

If your app does need Storage:

Create buckets in Supabase Dashboard → Storage:
- `avatars` — public read (profile images)
- `documents` — private (user documents)

Storage buckets use RLS policies on the `storage.objects` table. Apply the same rules as database RLS: separate policies per operation, `TO authenticated`, and `(select auth.uid())`.

```sql
-- Allow authenticated users to upload to their own folder in the avatars bucket
CREATE POLICY "Users can upload own avatar"
  ON storage.objects
  FOR INSERT
  TO authenticated
  WITH CHECK (
    bucket_id = 'avatars'
    AND (select auth.uid())::text = (storage.foldername(name))[1]
  );

-- Allow public read on the avatars bucket (no auth required)
CREATE POLICY "Public avatar read"
  ON storage.objects
  FOR SELECT
  USING (bucket_id = 'avatars');

-- For upsert (overwrite), also add SELECT and UPDATE policies
CREATE POLICY "Users can update own avatar"
  ON storage.objects
  FOR UPDATE
  TO authenticated
  USING ((select auth.uid())::text = (storage.foldername(name))[1])
  WITH CHECK (bucket_id = 'avatars');
```

Docs: https://supabase.com/docs/guides/storage/security/access-control

---

## Phase 8 — Environment Variables & Secrets

`.env.local` structure:

```
EXPO_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...
EXPO_PUBLIC_REVENUECAT_IOS_KEY=appl_...
EXPO_PUBLIC_REVENUECAT_ANDROID_KEY=goog_...
EXPO_PUBLIC_SENTRY_DSN=https://...@....ingest.sentry.io/...
EXPO_PUBLIC_POSTHOG_KEY=phc_...
```

> **Key naming:** Supabase has deprecated the old `ANON_KEY` in favour of `PUBLISHABLE_KEY` (format: `sb_publishable_xxx`). Get it from Supabase Dashboard → Project Settings → API Keys. The legacy `anon` key still works during the transition period but will be removed.

`EXPO_PUBLIC_*` = safe for client bundle (public keys only).
Non-prefixed = server/CI only. Never expose in the app.

Add secrets to EAS (never to any committed file):

```bash
eas secret:create --scope project --name SUPABASE_SERVICE_ROLE_KEY --value <key>
```

> **WARNING:** `SUPABASE_SERVICE_ROLE_KEY` bypasses RLS entirely. It belongs only in Edge Functions and CI/CD pipelines. Never in the app bundle.

Docs: https://docs.expo.dev/build-reference/variables/

---

## Phase 9 — Payments & Monetisation

**Install (must use `npx expo install`, not pnpm — native module version pinning required):**

```bash
npx expo install react-native-purchases react-native-purchases-ui
```

> **CRITICAL — Expo Go does not support RevenueCat.** `react-native-purchases` contains native code that Expo Go cannot run. You must use a development build (EAS Build with `developmentClient: true`) to test any RevenueCat code. Hot reloading without a full build will throw: `Invariant Violation: new NativeEventEmitter() requires a non-null argument`.

Initialise in `src/app/_layout.tsx` before any screen renders:

```typescript
import Purchases, { LOG_LEVEL } from 'react-native-purchases';
import { Platform } from 'react-native';
import { useEffect } from 'react';

// Inside your root layout component:
useEffect(() => {
  Purchases.setLogLevel(__DEV__ ? LOG_LEVEL.DEBUG : LOG_LEVEL.ERROR);
  Purchases.configure({
    apiKey: Platform.OS === 'ios'
      ? process.env.EXPO_PUBLIC_REVENUECAT_IOS_KEY!
      : process.env.EXPO_PUBLIC_REVENUECAT_ANDROID_KEY!,
  });
}, []);
```

**Web subscriptions:** RevenueCat now supports web via Web Billing (uses Stripe as processor). Configure a Web Billing App in the RevenueCat dashboard separately from iOS/Android. Use the same entitlements across all three platforms so subscription status is unified.

> **Apple Rule:** In-app digital purchases on iOS MUST use StoreKit (via RevenueCat). Stripe or external payment links for digital goods will result in rejection. Reference: App Store Review Guidelines 3.1.1.

Create matching products in all three places — they must align:
1. App Store Connect → Monetisation → Subscriptions
2. Google Play Console → Monetise → Subscriptions
3. RevenueCat Dashboard → Products (link to App Store and Play Store products)

Entitlements in RevenueCat are the shared abstraction across platforms. Link all platform products to the same entitlement (e.g. `premium`). Check entitlement status in your app, not individual product IDs.

Docs:
- Expo install: https://www.revenuecat.com/docs/getting-started/installation/expo
- Configuring SDK: https://www.revenuecat.com/docs/getting-started/configuring-sdk
- Web Billing: https://www.revenuecat.com/docs/web/web-billing

---

## Phase 10 — Push Notifications

Add to `app.json` plugins:

```json
["expo-notifications", {
  "icon": "./assets/notification-icon.png",
  "color": "#1F4E79",
  "defaultChannel": "default"
}]
```

Register for push on app load. Save the Expo Push Token to `profiles.push_token` in Supabase.
Send notifications from Edge Functions via `https://exp.host/--/api/v2/push/send`.

> Push will not work on simulators. Always test on a physical device.

Docs: https://docs.expo.dev/versions/latest/sdk/notifications/

---

## Phase 11 — Error Monitoring & Analytics

**Sentry:**

```bash
pnpm add @sentry/react-native
npx @sentry/wizard@latest -i reactNative
```

Init in `app/_layout.tsx`:

```typescript
import * as Sentry from '@sentry/react-native';
Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  enabled: !__DEV__,
});
```

**PostHog:**

```typescript
import { PostHogProvider } from 'posthog-react-native';
// Wrap root layout with <PostHogProvider apiKey={...} options={{ host: 'https://app.posthog.com' }}>
```

Docs:
- Sentry: https://docs.sentry.io/platforms/react-native/
- PostHog: https://posthog.com/docs/libraries/react-native

---

## Phase 12 — EAS Build & CI/CD

`eas.json`:

```json
{
  "cli": { "version": ">= 5.0.0" },
  "build": {
    "development": { "developmentClient": true, "distribution": "internal" },
    "preview": { "distribution": "internal", "android": { "buildType": "apk" } },
    "production": { "autoIncrement": true }
  },
  "submit": {
    "production": {
      "ios": { "appleId": "<email>", "ascAppId": "<app-id>" },
      "android": { "serviceAccountKeyPath": "./google-service-account.json" }
    }
  }
}
```

Build commands:

```bash
# Development
eas build --profile development --platform ios
eas build --profile development --platform android

# Internal testing
eas build --profile preview --platform all

# Production
eas build --profile production --platform all

# OTA update (JS changes only, no new store submission needed)
eas update --branch production --message '<description>'
```

Docs: https://docs.expo.dev/build/introduction/

---

## Phase 13 — Apple App Store Deployment

Pre-submission checklist (verify every item):
- [ ] `bundleIdentifier` in `app.json` matches App Store Connect
- [ ] App icon: 1024x1024px PNG, no transparency, no rounded corners
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) declares all API usage
- [ ] App Privacy Labels completed in App Store Connect
- [ ] All `NSUsageDescription` keys present in `app.json` for permissions used
- [ ] Sign in with Apple implemented if any other social login is offered
- [ ] In-App Purchase products created in App Store Connect AND RevenueCat
- [ ] Age rating set
- [ ] Support URL and Privacy Policy URL filled in
- [ ] Screenshots: iPhone 6.9" required; iPad 13" required if tablet supported

Submit:

```bash
eas build --profile production --platform ios
eas submit --platform ios
```

Docs:
- App Store Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- App Store Connect: https://developer.apple.com/help/app-store-connect/
- Privacy Manifest: https://developer.apple.com/documentation/bundleresources/privacy_manifest_files

---

## Phase 14 — Google Play Store Deployment

Pre-submission checklist:
- [ ] `android.package` in `app.json` matches Play Console
- [ ] Adaptive icon foreground: 1024x1024px transparent PNG
- [ ] Data Safety section completed in Play Console
- [ ] Content rating questionnaire completed
- [ ] Privacy Policy URL added to store listing
- [ ] In-App Products created in Play Console AND RevenueCat
- [ ] Screenshots: phone (min 2); tablets if supported

Submit:

```bash
eas build --profile production --platform android
eas submit --platform android
```

Docs:
- Play Console Help: https://support.google.com/googleplay/android-developer/
- Data Safety: https://support.google.com/googleplay/android-developer/answer/10787469
- EAS Submit Android: https://docs.expo.dev/submit/android/

---

## Phase 15 — Web Deployment (Vercel)

```bash
npx expo export --platform web
vercel
# Set output directory to: dist
```

Add all `EXPO_PUBLIC_*` variables to Vercel Dashboard → Project Settings → Environment Variables. Use `EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY` — not the legacy `ANON_KEY`.

> **Warning:** `expo-secure-store` does not work on web. Use a conditional storage adapter in `src/lib/supabase.ts`:

```typescript
const storage = Platform.OS === 'web'
  ? { getItem: (k) => Promise.resolve(localStorage.getItem(k)),
      setItem: (k, v) => Promise.resolve(localStorage.setItem(k, v)),
      removeItem: (k) => Promise.resolve(localStorage.removeItem(k)) }
  : ExpoSecureStoreAdapter;
```

Docs: https://docs.expo.dev/workflow/web/

---

## Phase 16 — Post-Launch Checklist

Security:
- [ ] All tables have RLS enabled (run audit query from Phase 6)
- [ ] No `service_role` string anywhere in app source code
- [ ] `.env.local` is in `.gitignore`
- [ ] Unauthenticated navigation tested — all protected routes redirect correctly

Monitoring:
- [ ] Sentry receives a test error
- [ ] PostHog receives a test event
- [ ] Supabase monitoring dashboard is reviewed

Performance:
- [ ] Hermes engine is enabled (default in Expo SDK 50+)
- [ ] All database queries are paginated — no unbounded result sets
- [ ] Images are compressed; WebP used where possible

Backup:
- [ ] Supabase daily backups confirmed (Pro plan) or manual export scheduled (Free plan)
- [ ] A restore has been tested at least once

---

## Minimum Cost to Launch

This section covers the real minimum costs to go from zero to a live app on both stores. All figures are verified from official pricing pages.

### What is free (genuinely $0)

| Service | Free Tier |
|---|---|
| Supabase | 500 MB DB, 1 GB storage, 50K MAUs, 10 GB bandwidth, 500K Edge Function calls/month. **2 active projects max. Projects pause after 7 days of inactivity.** No backups, no SLA. |
| EAS Build | 30 builds/month (max 15 iOS), lower priority queue. 1,000 EAS Update MAUs, 100 GB update bandwidth. |
| RevenueCat | Free up to $2,500 MRR. No build or user limit below that. |
| Vercel | Hobby tier covers most indie apps. 100 GB bandwidth/month. |
| Sentry | 5,000 errors/month. |
| PostHog | 1 million events/month. |

### Unavoidable costs to ship to both stores

| Cost | Amount | Notes |
|---|---|---|
| Apple Developer Program | **$99/year** | Required to submit to the App Store. Renews annually. |
| Google Play Console | **$25 one-time** | Required to submit to the Play Store. Never renews. |
| **Year 1 minimum** | **$124** | Apple $99 + Google $25 |
| **Year 2+ minimum** | **$99/year** | Apple renewal only |

### When to upgrade (trigger points)

**Supabase → Pro ($25/month):**
- Database approaches 400–450 MB (hitting 500 MB limit causes read-only mode)
- Approaching 50,000 MAUs
- You need daily backups (free tier has none)
- You need the project to stay online without manual unpausing

**EAS → Starter ($19/month):**
- You need more than 30 builds/month
- You need priority build queue (free builds can queue for 30–60+ minutes)
- You need more than 1,000 EAS Update MAUs

### Realistic cost at early traction (~5,000 users)

| Service | Monthly Cost |
|---|---|
| Supabase | $0 (well within free tier) |
| EAS | $0 or $19 depending on build frequency |
| RevenueCat | $0 (below $2.5k MRR) |
| Vercel | $0 |
| Sentry + PostHog | $0 |
| Apple (amortised) | ~$8/month |
| **Total** | **$8–$27/month** |

### Realistic cost at growth (~50,000 users)

| Service | Monthly Cost |
|---|---|
| Supabase Pro | $25 |
| EAS Starter | $19 |
| RevenueCat | $0 if under $2.5k MRR, then % of revenue |
| Vercel Pro | $20 |
| Apple (amortised) | ~$8 |
| **Total** | **~$72/month** |

> **On the free Supabase tier:** Commercial use is explicitly allowed. The main risk is the 7-day inactivity pause — this is not a problem for apps with real users. Monitor your DB size in the Supabase dashboard. Set up an alert before you hit 450 MB.

---

## Documentation Reference

| Resource | URL |
|---|---|
| Expo Docs | https://docs.expo.dev |
| Expo Router | https://docs.expo.dev/router/introduction/ |
| EAS Build | https://docs.expo.dev/build/introduction/ |
| EAS Submit | https://docs.expo.dev/submit/introduction/ |
| EAS Update | https://docs.expo.dev/eas-update/introduction/ |
| Supabase Docs | https://supabase.com/docs |
| Supabase Auth | https://supabase.com/docs/guides/auth |
| Supabase JS Client | https://supabase.com/docs/reference/javascript/introduction |
| Row Level Security | https://supabase.com/docs/guides/database/postgres/row-level-security |
| Supabase Storage | https://supabase.com/docs/guides/storage |
| Supabase Edge Functions | https://supabase.com/docs/guides/functions |
| Supabase CLI | https://supabase.com/docs/reference/cli/introduction |
| RevenueCat Expo | https://www.revenuecat.com/docs/getting-started/installation/expo |
| Apple Review Guidelines | https://developer.apple.com/app-store/review/guidelines/ |
| Apple App Store Connect | https://developer.apple.com/help/app-store-connect/ |
| Apple Privacy Manifest | https://developer.apple.com/documentation/bundleresources/privacy_manifest_files |
| Apple HIG | https://developer.apple.com/design/human-interface-guidelines/ |
| Google Play Console | https://support.google.com/googleplay/android-developer/ |
| Google Data Safety | https://support.google.com/googleplay/android-developer/answer/10787469 |
| Sentry React Native | https://docs.sentry.io/platforms/react-native/ |
| PostHog React Native | https://posthog.com/docs/libraries/react-native |
| Vercel Docs | https://vercel.com/docs |
| GitHub Actions | https://docs.github.com/en/actions |
