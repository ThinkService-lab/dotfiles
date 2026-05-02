# Global App Architecture — Agent Instructions

This file is the cross-tool agent instruction standard (`AGENTS.md`). It is read by Claude Code, Codex, Cursor, Antigravity, GitHub Copilot, Windsurf, and any other tool that respects the AGENTS.md open standard maintained by the Agentic AI Foundation.

You are building a cross-platform app (iOS, Android, Web) using the standard architecture defined below.
Read this file fully before writing any code, creating any file, or running any command.
This architecture is non-negotiable. Do not deviate from it unless a project-level `AGENTS.md` (or tool-specific equivalent like `CLAUDE.md` or `GEMINI.md`) explicitly overrides a section.

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
| UI Styling | NativeWind (Tailwind for React Native) |
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

- Use `pnpm` for JS-only dependencies. Use `npx expo install` for ANY package with native code (anything starting with `expo-`, `react-native-`, or `@react-native-`). Expo pins the version that matches your SDK — using pnpm/npm directly will install incompatible versions and silently break the native build.
- Always use TypeScript. Never plain JavaScript.
- Always use Expo Router for navigation. Never React Navigation standalone.
- Always use `expo-sqlite/localStorage/install` as the Supabase auth storage adapter (official Supabase recommendation). Do not use a raw `expo-secure-store` adapter — SecureStore has a 2KB limit that JWT tokens exceed, causing silent auth failures.
- Always enable Row-Level Security (RLS) on every Supabase table immediately after creating it.
- Always use migration files for schema changes (`supabase migration new`). Never edit the database directly in the Supabase dashboard.
- Always regenerate TypeScript types after every migration: `supabase gen types typescript --linked > src/types/database.ts`
- Never put `SUPABASE_SERVICE_ROLE_KEY` in the app bundle or any `EXPO_PUBLIC_*` variable.
- Never commit `.env.local`. Always verify it is in `.gitignore` before the first commit.
- Never use unicode bullet characters (`•`) in code — use proper list syntax.

---

## Agent Operating Rules

How the agent should behave when executing this manual:

- **Read this entire file before the first action** in a new project. Do not start writing code based on a partial read.
- **Confirm before destructive operations.** Always show a preview and ask for confirmation before: running `supabase db push`, dropping any table, deleting any file outside `node_modules`, force-pushing to git, or running `eas submit`.
- **Show the SQL diff before applying migrations.** Print the migration file content and ask "Apply this migration to the linked Supabase project? (y/n)" before running `supabase db push`.
- **Run the RLS audit query after every migration.** Verify `rowsecurity = true` for all new tables before continuing.
- **If a phase is ambiguous or context is missing, stop and ask.** Do not guess at the project name, bundle ID, Supabase ref, or any value the user has not provided.
- **State which phase you are in** at the start of each significant action. Example: "[Phase 6] Creating profiles table migration."
- **Never claim a phase is complete without verification.** After Phase 4 setup, run a test query. After Phase 5 auth, attempt a test login. After Phase 6 migration, query `pg_tables`.
- **If a command fails, stop and report.** Do not silently retry or work around it.

---

## Boot Sequence (First Action in Any New Project)

When the agent enters a project folder for the first time, run this checklist before doing anything else:

1. **Check for project-level instructions.** Read `./AGENTS.md`, `./CLAUDE.md`, or `./GEMINI.md` if any exist (in that priority order) — those instructions override anything in this global file.
2. **Detect project state.** Run `ls -la` to see if this is a fresh folder, an in-progress project, or an existing app. Adjust the starting phase accordingly.
3. **Verify toolchain.** Run the Phase 1 verification block (`node --version`, `pnpm --version`, `expo --version`, `eas --version`, `supabase --version`). Report missing tools before proceeding.
4. **Read `app.json` if it exists.** Confirm bundle ID, slug, and version match the project-level instruction file.
5. **Check `.gitignore` includes `.env.local`.** If it does not, fix it before any other action.

Only after all 5 steps are clean does the agent proceed to the requested phase.

---

## Phase 1 — Prerequisites

Before initialising a project, verify all tools are installed:

```bash
node --version        # must be LTS via nvm
pnpm --version
npx expo --version    # no global expo-cli — modern Expo uses npx
eas --version
supabase --version
```

If any are missing:

```bash
# Node via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install --lts && nvm use --lts

# EAS CLI (expo-cli is deprecated — modern usage is `npx create-expo-app` and `npx expo` directly)
npm install -g eas-cli

# Supabase CLI
npm install -g supabase

# pnpm
npm install -g pnpm
```

Required accounts: Apple Developer ($99/yr), Google Play Console ($25 one-time), Supabase, Expo, RevenueCat (if monetising), Sentry, PostHog, Vercel (if web). Verify all exist before Phase 2.

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
pnpm add nativewind && pnpm add -D tailwindcss@3.3.2
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

**Verify Phase 4:** From a screen, run `console.log(await supabase.auth.getSession())` — should return `{ data: { session: null }, error: null }` (no error). Failure here means the client is misconfigured.

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

**Verify Phase 6 (run before EVERY deployment):**

```sql
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';
```
Every row must show `rowsecurity = true`. If any row shows `false`, stop and add RLS before continuing.

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

## Phase 9 — Payments & Monetisation (OPTIONAL)

Skip this phase if your app is free with no in-app purchases or subscriptions. You can add payments later without architectural changes.

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

## Phase 10 — Push Notifications (OPTIONAL)

Skip this phase if your app does not need to send notifications. Most utility apps and v1 launches do not need push.

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

Init in `src/app/_layout.tsx`:

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
// In src/app/_layout.tsx
import { PostHogProvider } from 'posthog-react-native';
// Wrap root layout with <PostHogProvider apiKey={...} options={{ host: 'https://app.posthog.com' }}>
```

Docs:
- Sentry: https://docs.sentry.io/platforms/react-native/
- PostHog: https://posthog.com/docs/libraries/react-native

---

## Phase 12 — UI Styling (NativeWind)

NativeWind brings Tailwind class syntax to React Native. Same classes as web — `className="flex-1 bg-white p-4"` works on iOS, Android, and web.

**Install:**

```bash
pnpm add nativewind
pnpm add -D tailwindcss@3.3.2
npx tailwindcss init
```

**Configure `tailwind.config.js`:**

```javascript
module.exports = {
  content: ['./src/**/*.{js,jsx,ts,tsx}'],
  presets: [require('nativewind/preset')],
  theme: { extend: {} },
  plugins: [],
};
```

**Configure `babel.config.js`:**

```javascript
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [['babel-preset-expo', { jsxImportSource: 'nativewind' }], 'nativewind/babel'],
  };
};
```

**Create `src/global.css`:**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Import once at the top of `src/app/_layout.tsx`:

```typescript
import '../global.css';
```

**Usage:**

```typescript
import { View, Text } from 'react-native';

export function Card({ title }: { title: string }) {
  return (
    <View className="bg-white rounded-xl p-4 shadow">
      <Text className="text-lg font-semibold text-slate-900">{title}</Text>
    </View>
  );
}
```

**Verify Phase 12:** A `<View className="bg-red-500">` should render with a red background on all platforms. If it doesn't, the babel config is the most common cause.

Docs: https://www.nativewind.dev/getting-started/expo

---

## Phase 13 — Testing

Three layers, each with one tool. Don't add more.

| Layer | Tool | What it covers |
|---|---|---|
| Unit / component | Jest + React Native Testing Library | Pure functions, hooks, components in isolation |
| Integration | Jest + RNTL | Multiple components together, mocked Supabase |
| End-to-end | Maestro | Full user flows on simulator/device |

**Install:**

```bash
pnpm add -D jest jest-expo @testing-library/react-native @testing-library/jest-native
pnpm add -D @types/jest
```

**Configure `package.json`:**

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch"
  },
  "jest": {
    "preset": "jest-expo",
    "setupFilesAfterEach": ["@testing-library/jest-native/extend-expect"]
  }
}
```

**Test file convention:** Co-locate tests next to source as `<filename>.test.tsx`.

**Mock Supabase in tests:** Create `src/lib/__mocks__/supabase.ts` with stubbed methods. Never hit the real DB from tests.

**Maestro for E2E** (install separately, only when you have stable flows):

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
```

Write flows in `.maestro/<flow_name>.yaml`. Run with `maestro test .maestro/login.yaml`.

**Verify Phase 13:** `pnpm test` runs and passes with at least one assertion. Add a smoke test for the Home screen as the first test.

Docs:
- Jest Expo: https://docs.expo.dev/develop/unit-testing/
- React Native Testing Library: https://callstack.github.io/react-native-testing-library/
- Maestro: https://maestro.mobile.dev/

---

## Phase 14 — Git Workflow

**Branch strategy:**
- `main` — always deployable. Production builds come from here only.
- `dev` — integration branch. Feature branches merge here first.
- `feature/<short-name>` — one branch per feature. Squash-merge into `dev`.

**Commit format (Conventional Commits):**

```
<type>(<scope>): <subject>

<optional body>
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `build`.

Examples:
- `feat(auth): add magic link login`
- `fix(profile): handle null avatar URL`
- `chore(deps): bump expo to sdk 54`

**Required `.gitignore` entries:**

```
.env.local
.env*.local
node_modules/
ios/
android/
.expo/
dist/
*.log
google-service-account.json
```

**Commit policy:**
- Never commit `.env*` files. Verify before every commit: `git diff --cached --name-only | grep -i env` should return nothing.
- Never commit `google-service-account.json`.
- Never commit generated files: `ios/`, `android/`, `dist/`, `.expo/`.
- Migrations (`supabase/migrations/*.sql`) ARE committed.

**PR template (`.github/pull_request_template.md`):**

```markdown
## What
Brief description of the change.

## Why
The user-facing or technical reason.

## Verification
- [ ] Tested on iOS simulator
- [ ] Tested on Android emulator
- [ ] RLS audit query passes (if migrations included)
- [ ] No new console warnings
```

**Verify Phase 14:** `git status` shows no `.env*` or `node_modules` files. `cat .gitignore` includes all entries above.

---

## Phase 15 — EAS Build & CI/CD

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

## Phase 16 — Apple App Store Deployment

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

## Phase 17 — Google Play Store Deployment

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

## Phase 18 — Web Deployment (Vercel)

```bash
npx expo export --platform web
vercel
# Set output directory to: dist
```

Add all `EXPO_PUBLIC_*` variables to Vercel Dashboard → Project Settings → Environment Variables. Use `EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY` — not the legacy `ANON_KEY`.

> **Storage adapter on web:** No special handling needed. The `expo-sqlite/localStorage/install` polyfill used in Phase 4 works on both native and web — it falls back to the browser's native localStorage when running in a browser environment.

> **Native-only modules will not work on web.** `react-native-purchases` (RevenueCat) does not run on web — use RevenueCat Web Billing (Stripe) for web subscriptions, see Phase 9. `expo-notifications` push tokens do not work on web. Wrap any native-only initialisation in `if (Platform.OS !== 'web')`.

Docs: https://docs.expo.dev/workflow/web/

---

## Phase 19 — Post-Launch Checklist

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
