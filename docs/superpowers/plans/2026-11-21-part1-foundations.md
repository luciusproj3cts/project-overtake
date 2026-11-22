# Part 1 · Foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An installable Android development app where a user can sign up (email or Google), create a public profile, and add up to 3 cars with one photo each, styled in the approved design system, with the 5-tab shell ready for later parts.

**Architecture:** Expo (React Native, TypeScript) app in `/mobile` using Expo Router for screens. Supabase provides auth, a Postgres database (tables + Row Level Security, defined as SQL migrations in `/supabase/migrations`) and photo storage. Business rules (validation, routing decisions, photo sizing) live in small pure functions with unit tests; screens stay thin.

**Tech Stack:** Expo SDK (latest stable at setup) · Expo Router · expo-dev-client · EAS Build · TypeScript (strict) · Jest (`jest-expo`) + React Native Testing Library · Supabase JS v2 + Supabase CLI · `@react-native-google-signin/google-signin` · `expo-image-picker` + `expo-image-manipulator` · `@react-native-async-storage/async-storage` · `expo-font` · Sentry (`@sentry/react-native`) · PostHog (`posthog-react-native`)

**Execution method [Decided]:** Native (superpowers:executing-plans), run with the founder on their Windows PC; one whole-branch review at the end.

**Spec:** `docs/design/v1-design.md` (product design) and `docs/design/design-system.md` (visual design). Executors read both.

## Global Constraints

- Founder rules (`CLAUDE.md`): **commit routine work without asking; ask the founder before large changes** (e.g. switching away from Supabase); work on branch `feature/part1-foundations`; max 3 fix attempts per problem, then report the issue with an alternative; replies follow the founder's response format.
- Platform: Android only. Development build (`expo-dev-client`), never Expo Go.
- Temporary Android app ID: `com.overtake.dev` (final app ID is **[Open]**, needed before the first Play upload; the Google OAuth Android client must be recreated when it changes).
- Region: Malaysia, English UI, km and km/h, time zone `Asia/Kuala_Lumpur`.
- Budget: free tiers only. Supabase dev project in region **Southeast Asia (Singapore)**.
- Secrets: `.env*` files are git-ignored except `.env.example`. The app only reads `EXPO_PUBLIC_*` variables (Supabase URL + publishable key are safe to ship because every table has Row Level Security). The Supabase **secret** key exists only in `mobile/.env.test.local` for database tests and never in app code.
- Row Level Security enabled on every table and storage bucket.
- Visual design: only tokens from `mobile/src/theme/tokens.ts`, matching `docs/design/design-system.md` exactly (colours, 0 radius, 3 px/2 px ink borders, hard shadow, uppercase display text). Fonts: DX Brutalist (display) + Space Mono (body/labels); Archivo Black is the display fallback until the DX Brutalist licence file is added.
- Cars: max **3** per user, **1** photo per car. Photos: JPEG, longest side ≤ **1280 px**, ≤ **400 KB**.
- Username: 3–20 characters, lowercase `a–z`, `0–9`, `.` and `_`, unique (case-insensitive). Display name: 1–40 characters.
- Analytics never receives email, username, display name or location.
- Tabs: DRIVE · TRIPS · RANKS · COMMUNITY · PROFILE.

## Review Focus

1. **Photo upload fails mid-save** (signal drops): the car must still exist exactly once, show "ADD PHOTO" to retry, and never be duplicated by a double-tap. Test in Task 10.
2. **Two people claim the same username at the same moment:** the second gets a friendly "Username taken" message, not a crash or a raw database error. Test in Task 9.
3. **A 4th car added by bypassing the app** (direct API call): the database rejects it with `MAX_CARS_REACHED`. Test in Task 4.
4. **App reopened offline with a saved session:** it opens to the app (not a blank screen or a forced sign-out). Test in Task 5.
5. **Huge or rotated phone photos** (50 MP, portrait EXIF): uploaded upright, ≤ 1280 px and ≤ 400 KB. Test in Task 8.

---

### Task 0: Founder setup on the Windows PC (no code)

Claude walks the founder through each item live; each ends with a check.

- [ ] **Step 1:** Install Node.js LTS, Git for Windows, and the Claude desktop app (Claude Code). Check: `node -v` and `git --version` print versions.
- [ ] **Step 2:** Clone `luciusproj3cts/project-overtake` to a folder on the PC and open it in Claude Code. Check: `git status` shows branch `main`.
- [ ] **Step 3:** On the Android phone, enable Developer options → USB debugging. Check: phone is visible when plugged in (Claude confirms with `adb devices` once tools are installed in Task 1).
- [ ] **Step 4:** Create free accounts: Expo (expo.dev), Supabase, Sentry, PostHog (EU or US cloud). Check: founder can log in to each.
- [ ] **Step 5:** In Supabase, create project `overtake-dev`, region Southeast Asia (Singapore). Save the database password in a password manager. Check: project dashboard opens.
- [ ] **Step 6:** Create a Google Cloud project `overtake`, configure the OAuth consent screen (External, app name OVERTAKE). OAuth clients are created in Task 7. Check: consent screen saved.

### Task 1: App skeleton, tooling and first dev build

**Files:**
- Create: `mobile/` (Expo project: `package.json`, `app.config.ts`, `tsconfig.json`, `eas.json`, `.eslintrc.js`, `.prettierrc`, `jest.config.js`, `.env.example`, `app/_layout.tsx`, `app/index.tsx`)
- Create: `.gitignore` (root: `node_modules`, `.env*` except `.env.example`, `.expo`, `android/`, `ios/`, `*.keystore`)
- Test: `mobile/src/__tests__/smoke.test.tsx`

**Interfaces:**
- Produces: npm scripts `start`, `test`, `test:db`, `lint`, `typecheck`; EAS profiles `development` (dev client, internal APK) and `preview` (standalone APK, used later for Drive mode); `app.config.ts` reads `EXPO_PUBLIC_*` from env.

- [ ] **Step 1: Create the project** with `npx create-expo-app@latest mobile --template` (blank TypeScript), add Expo Router, `expo-dev-client`, ESLint, Prettier, `jest-expo`, `@testing-library/react-native`. Set `"strict": true` in `tsconfig.json`. App name `OVERTAKE`, slug `overtake`, Android package `com.overtake.dev`, orientation portrait.
- [ ] **Step 2: Write the smoke test**

```tsx
// renders the root index without crashing
render(<Index />);
expect(screen.getByText('OVERTAKE')).toBeTruthy();
```

- [ ] **Step 3: Run** `npm test`. Expected: FAIL (index does not render `OVERTAKE` yet).
- [ ] **Step 4: Implement** `app/index.tsx` showing the text `OVERTAKE`.
- [ ] **Step 5: Run** `npm test && npm run typecheck && npm run lint`. Expected: all pass, 0 errors.
- [ ] **Step 6: Build and install the dev client:** `npx eas-cli build -p android --profile development`, install the APK on the phone, run `npm start`, open the app. Expected: phone shows `OVERTAKE`; editing the text on the PC updates the phone within seconds.
- [ ] **Step 7: Commit** `chore: scaffold Expo app with tooling and dev build`.

### Task 2: Design tokens, fonts and core UI components

**Files:**
- Create: `mobile/src/theme/tokens.ts`, `mobile/src/theme/fonts.ts`, `mobile/assets/fonts/` (Space Mono 400/700 TTF, Archivo Black TTF, `LICENSES.md`)
- Create: `mobile/src/components/ui/Button.tsx`, `TextField.tsx`, `Screen.tsx`, `PhotoBox.tsx`
- Test: `mobile/src/theme/__tests__/tokens.test.ts`, `mobile/src/components/ui/__tests__/Button.test.tsx`, `TextField.test.tsx`

**Interfaces:**
- Produces:
  - `colors: { paper, ink, accent, muted, paperDeep, danger, success }` (hex values from design-system.md §1), `space: { xs:4, sm:8, md:12, lg:16, xl:24, xxl:32 }`, `border: { main:3, divider:2 }`, `radius: 0`, `shadow: { offset:6 }`, `type: { hero, h1, h2, stat, body, label }` (sizes from §2), `fontFamily: { display, mono, monoBold }`.
  - `contrastRatio(fg: string, bg: string): number`
  - `<Button label variant="primary"|"secondary"|"danger" onPress disabled loading />`: label rendered uppercase; pressed state removes the shadow and shifts 6 px.
  - `<TextField label value onChangeText error? secureTextEntry? autoCapitalize? />`
  - `<Screen title?>` (paper background, 16 px side margin, safe area)
  - `<PhotoBox uri? label onPress? size />` (`paperDeep` placeholder with uppercase label when no `uri`)
  - `useAppFonts(): boolean` (true when fonts are loaded)

- [ ] **Step 1: Write failing tests**

```ts
// tokens.test.ts
expect(contrastRatio(colors.ink, colors.paper)).toBeGreaterThanOrEqual(4.5);
expect(contrastRatio(colors.muted, colors.paper)).toBeGreaterThanOrEqual(4.5);
expect(contrastRatio(colors.ink, colors.accent)).toBeGreaterThanOrEqual(4.5);
expect(contrastRatio(colors.danger, colors.paper)).toBeGreaterThanOrEqual(4.5);
expect(contrastRatio(colors.success, colors.paper)).toBeGreaterThanOrEqual(4.5);
expect(radius).toBe(0);
```

```tsx
// Button.test.tsx
render(<Button label="Save car" onPress={fn} />);
expect(screen.getByText('SAVE CAR')).toBeTruthy();
fireEvent.press(screen.getByRole('button')); expect(fn).toHaveBeenCalledTimes(1);
render(<Button label="Save" onPress={fn2} disabled />); fireEvent.press(...); expect(fn2).not.toHaveBeenCalled();
render(<Button label="Save" onPress={fn3} loading />); // not pressable, shows activity indicator, accessibilityState.busy === true
```

```tsx
// TextField.test.tsx
render(<TextField label="Username" value="" onChangeText={f} error="Username taken" />);
expect(screen.getByText('USERNAME')).toBeTruthy();
expect(screen.getByText('Username taken')).toBeTruthy();
```

- [ ] **Step 2: Run** `npm test -- theme components`. Expected: FAIL (modules not found).
- [ ] **Step 3: Implement** tokens (exact hexes from design-system.md), `contrastRatio` (WCAG relative-luminance formula), fonts via `expo-font`, and the four components.
- [ ] **Step 4: Run** `npm test`. Expected: PASS.
- [ ] **Step 5: Check on phone:** temporary preview of all components on `app/index.tsx` matches the mockup style (square corners, thick borders, lime primary button with hard shadow). Founder confirms.
- [ ] **Step 6: Commit** `feat: design tokens, fonts and core UI components`.

### Task 3: Five-tab app shell

**Files:**
- Create: `mobile/app/(tabs)/_layout.tsx`, `drive.tsx`, `trips.tsx`, `ranks.tsx`, `community.tsx`, `profile.tsx`, `mobile/src/components/ui/TabBar.tsx`
- Test: `mobile/src/components/ui/__tests__/TabBar.test.tsx`

**Interfaces:**
- Produces: route group `(tabs)` with routes `drive` (initial), `trips`, `ranks`, `community`, `profile`; custom `<TabBar>` used by the tabs layout. Placeholder screens show their title (e.g. `DRIVE`) and "Coming in Part N".

- [ ] **Step 1: Write failing test**

```tsx
render(<TabBar state={{ index: 2, routes: fiveRoutes }} ... />);
['DRIVE','TRIPS','RANKS','COMMUNITY','PROFILE'].forEach(l => expect(screen.getByText(l)).toBeTruthy());
expect(screen.getByText('RANKS').parent).toHaveStyle({ backgroundColor: colors.accent }); // active tab
expect(screen.getAllByRole('tab')).toHaveLength(5); // each ≥ 56 px tall
```

- [ ] **Step 2: Run** the test. Expected: FAIL.
- [ ] **Step 3: Implement** the tab layout and `TabBar` (5 equal cells, 2 px ink dividers, 3 px top border, 11 px Space Mono bold labels, active cell lime).
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: all five tabs switch, labels fit without truncation.
- [ ] **Step 5: Commit** `feat: five-tab app shell`.

### Task 4: Database schema, security rules and database tests

**Files:**
- Create: `supabase/config.toml` (via `npx supabase init`), `supabase/migrations/20261121000001_profiles.sql`, `20261121000002_cars.sql`, `20261121000003_storage.sql`
- Create: `mobile/jest.db.config.js`, `mobile/tests/db/helpers.ts`, `mobile/.env.test.local` (git-ignored; Supabase URL, publishable key, secret key)
- Test: `mobile/tests/db/profiles.test.ts`, `mobile/tests/db/cars.test.ts`, `mobile/tests/db/storage.test.ts`

**Interfaces:**
- Produces (database):
  - `public.profiles(id uuid PK → auth.users ON DELETE CASCADE, username citext UNIQUE NOT NULL CHECK (username ~ '^[a-z0-9._]{3,20}$'), display_name text NOT NULL CHECK (char_length BETWEEN 1 AND 40), avatar_path text NULL, created_at timestamptz, updated_at timestamptz)`
  - `public.cars(id uuid PK DEFAULT gen_random_uuid(), owner_id uuid NOT NULL → profiles ON DELETE CASCADE, make text NOT NULL (1–40), model text NOT NULL (1–40), year int NULL CHECK (year BETWEEN 1950 AND 2100), nickname text NULL (≤30), photo_path text NULL, is_default boolean NOT NULL DEFAULT false, created_at, updated_at)` + partial unique index: one `is_default = true` per `owner_id`
  - Trigger `enforce_max_cars` (BEFORE INSERT on cars): raises exception with message `MAX_CARS_REACHED` when the owner already has 3 cars
  - Trigger `set_updated_at` on both tables
  - Function `public.is_username_available(candidate text) RETURNS boolean` (SECURITY DEFINER, case-insensitive)
  - RLS: profiles: SELECT for `authenticated`; INSERT/UPDATE only where `id = auth.uid()`; no DELETE (account deletion is Part 6). Cars: SELECT for `authenticated`; INSERT/UPDATE/DELETE only where `owner_id = auth.uid()`.
  - Storage buckets `car-photos` and `avatars`: public read; write/update/delete only for objects whose first path segment = `auth.uid()`; max file size 1 MB; allowed type `image/jpeg`.
- Produces (tests): `createTestUser(): Promise<{ client: SupabaseClient; userId: string }>` and `deleteTestUsers()` in `helpers.ts` (uses the secret key, only in tests).

- [ ] **Step 1: Link and write failing tests** (`npx supabase link` to `overtake-dev`)

```ts
// profiles.test.ts
A inserts own profile {username:'tester_a'} → succeeds
A inserts profile with id of B → error (RLS)
B inserts {username:'TESTER_A'} → error code '23505' (case-insensitive unique)
A updates B's display_name → 0 rows changed
username 'ab', 'Has Space', 'x'.repeat(21) → each rejected (check constraint)
is_username_available('Tester_A') → false; ('free_name') → true

// cars.test.ts
A inserts 3 cars → all succeed; 4th → error message contains 'MAX_CARS_REACHED'
A inserts car with owner_id = B → error (RLS)
B deletes A's car → 0 rows deleted
A sets two cars is_default = true → second update errors (unique index)
B can SELECT A's cars (public profile) → returns 3 rows

// storage.test.ts
A uploads `${A}/car1.jpg` to car-photos → ok; A uploads `${B}/x.jpg` → error
A uploads a 2 MB file → error; uploads 'image/png' → error
```

- [ ] **Step 2: Run** `npm run test:db`. Expected: FAIL (tables don't exist).
- [ ] **Step 3: Write the three migrations** to produce exactly the interfaces above; apply with `npx supabase db push`.
- [ ] **Step 4: Run** `npm run test:db`. Expected: PASS; test users cleaned up afterwards.
- [ ] **Step 5: Commit** `feat(db): profiles, cars, storage with RLS and tests`.

### Task 5: Supabase client, session and routing gate

**Files:**
- Create: `mobile/src/lib/env.ts`, `mobile/src/lib/supabase.ts`, `mobile/src/features/auth/useSession.ts`, `mobile/src/features/auth/resolveRoute.ts`, `mobile/src/features/profile/useMyProfile.ts`
- Modify: `mobile/app/_layout.tsx` (fonts + providers + gate), `mobile/app/index.tsx` (redirect)
- Test: `mobile/src/features/auth/__tests__/resolveRoute.test.ts`, `mobile/src/lib/__tests__/env.test.ts`

**Interfaces:**
- Consumes: `profiles` table (Task 4), `useAppFonts` (Task 2)
- Produces:
  - `env: { supabaseUrl: string; supabasePublishableKey: string; googleWebClientId: string; sentryDsn?: string; posthogKey?: string }`. `readEnv(source: Record<string, string | undefined>)` throws `Missing env: <NAME>` for required values
  - `supabase: SupabaseClient` (AsyncStorage session persistence, `autoRefreshToken: true`, `detectSessionInUrl: false`; auto-refresh paused when the app is in background)
  - `useSession(): { session: Session | null; loading: boolean }`
  - `useMyProfile(): { profile: Profile | null; loading: boolean; error: Error | null; refresh(): Promise<void> }`. Last fetched profile cached in AsyncStorage key `overtake.myProfile` and returned while offline
  - `type Profile = { id: string; username: string; displayName: string; avatarPath: string | null }`
  - `resolveRoute(s: { fontsLoaded: boolean; sessionLoading: boolean; session: Session | null; profileLoading: boolean; profile: Profile | null }): 'loading' | 'auth' | 'onboarding' | 'app'`

- [ ] **Step 1: Write failing tests**

```ts
resolveRoute({fontsLoaded:false, ...}) → 'loading'
resolveRoute({fontsLoaded:true, sessionLoading:false, session:null, ...}) → 'auth'
resolveRoute({..., session:S, profileLoading:false, profile:null}) → 'onboarding'
resolveRoute({..., session:S, profileLoading:false, profile:P}) → 'app'
resolveRoute({..., session:S, profileLoading:true, profile:P /* cached, offline */}) → 'app'   // Review Focus 4
readEnv({}) throws 'Missing env: EXPO_PUBLIC_SUPABASE_URL'
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** the modules; root layout redirects to `/(auth)/sign-in`, `/(onboarding)/profile-setup` or `/(tabs)/drive` based on `resolveRoute`; `'loading'` shows a paper screen with the wordmark.
- [ ] **Step 4: Run** `npm test`. Expected: PASS.
- [ ] **Step 5: Commit** `feat: supabase client, session and routing gate`.

### Task 6: Email sign-up, sign-in and password reset

**Files:**
- Create: `mobile/app/(auth)/_layout.tsx`, `sign-in.tsx`, `sign-up.tsx`, `forgot-password.tsx`
- Create: `mobile/src/features/auth/validation.ts`, `mobile/src/features/auth/authService.ts`, `mobile/src/features/auth/authErrors.ts`
- Test: `mobile/src/features/auth/__tests__/validation.test.ts`, `authErrors.test.ts`, `authService.test.ts`

**Interfaces:**
- Consumes: `supabase` (Task 5), `Button`/`TextField`/`Screen` (Task 2)
- Produces:
  - `validateEmail(v: string): string | null`, `validatePassword(v: string): string | null` (≥ 8 chars, at least one letter and one number)
  - `signUpWithEmail(email, password): Promise<AuthResult>`, `signInWithEmail(email, password): Promise<AuthResult>`, `sendPasswordReset(email): Promise<AuthResult>`, `signOut(): Promise<void>`
  - `type AuthResult = { ok: true } | { ok: false; message: string }`
  - `mapAuthError(e: { code?: string; message?: string }): string`

- [ ] **Step 1: Write failing tests**

```ts
validateEmail('a@b.co') → null; validateEmail('nope') → 'Enter a valid email'
validatePassword('short1') → 'At least 8 characters'; ('longpassword') → 'Include a number'; ('abc12345') → null
mapAuthError({code:'invalid_credentials'}) → 'Wrong email or password'
mapAuthError({code:'user_already_exists'}) → 'An account with this email already exists'
mapAuthError({code:'over_email_send_rate_limit'}) → 'Too many attempts. Try again in a few minutes'
mapAuthError({}) → 'Something went wrong. Try again'
signUpWithEmail with mocked supabase error → { ok:false, message: mapped }   // never throws
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** the service, validation and three screens (design-system inputs and buttons; sign-in links to sign-up and forgot-password; buttons show `loading` and can't be double-submitted).
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: sign up with the founder's own email (Supabase built-in email only delivers to team addresses; the sending service is **[Open]** until before the closed test), confirm, sign in, sign out.
- [ ] **Step 5: Commit** `feat(auth): email sign-up, sign-in and reset`.

### Task 7: Google sign-in

**Files:**
- Create: `mobile/src/features/auth/googleSignIn.ts`, Google button on `sign-in.tsx` and `sign-up.tsx`
- Modify: `mobile/app.config.ts` (Google Sign-In config plugin)
- Test: `mobile/src/features/auth/__tests__/googleSignIn.test.ts`

**Interfaces:**
- Consumes: `supabase`, `env.googleWebClientId`, `mapAuthError`
- Produces: `signInWithGoogle(): Promise<AuthResult | { ok: false; cancelled: true }>`

- [ ] **Step 1: Founder + Claude:** get the dev build's SHA-1 with `npx eas-cli credentials`; in Google Cloud create an **Android** OAuth client (package `com.overtake.dev` + SHA-1) and a **Web** OAuth client; in Supabase → Auth → Providers → Google, enable and paste the Web client ID/secret. Put the Web client ID in `mobile/.env` as `EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID`.
- [ ] **Step 2: Write failing tests** (GoogleSignin mocked)

```ts
user cancels → returns { ok:false, cancelled:true }; supabase.auth.signInWithIdToken not called
success with idToken 'tok' → signInWithIdToken called with { provider:'google', token:'tok' }; returns { ok:true }
Play Services missing → { ok:false, message:'Google sign-in is not available on this phone' }
```

- [ ] **Step 3: Run** the test. Expected: FAIL.
- [ ] **Step 4: Implement** `signInWithGoogle`; a cancelled sign-in shows no error.
- [ ] **Step 5: Rebuild the dev client** (native module added), `npm test`. Expected: PASS. On phone: Google sign-in lands on profile setup for a new account, on DRIVE for an existing one.
- [ ] **Step 6: Commit** `feat(auth): Google sign-in`.

### Task 8: Photo pipeline (pick, compress, upload)

**Files:**
- Create: `mobile/src/features/photos/preparePhoto.ts`, `mobile/src/features/photos/uploadPhoto.ts`, `mobile/src/features/photos/pickPhoto.ts`
- Test: `mobile/src/features/photos/__tests__/preparePhoto.test.ts`, `uploadPhoto.test.ts`

**Interfaces:**
- Consumes: `supabase`, buckets `car-photos` / `avatars` (Task 4)
- Produces:
  - `pickPhoto(source: 'camera' | 'library'): Promise<string | null>` (local URI; null if cancelled or permission denied)
  - `preparePhoto(uri: string): Promise<{ uri: string; width: number; height: number; bytes: number }>`: longest side ≤ 1280, JPEG; first pass quality 0.7, second pass 0.5 if > 400 KB; EXIF orientation applied by the manipulator before resizing
  - `uploadPhoto(bucket: 'car-photos' | 'avatars', userId: string, localUri: string): Promise<string>` returning the storage path `${userId}/${uuid}.jpg`
  - `deletePhoto(bucket, path): Promise<void>`
  - `publicPhotoUrl(bucket, path): string`

- [ ] **Step 1: Write failing tests** (image manipulator + file size mocked)

```ts
8000×6000 input → manipulate called with resize { width: 1280 }; output width 1280, height 960
3000×4000 (portrait) → resize { height: 1280 }; output 960×1280                                   // Review Focus 5
800×600 input → no resize; still re-encoded as JPEG 0.7
first pass 520 KB → second pass at 0.5 is called; result bytes ≤ 400 KB
uploadPhoto → storage.upload called with path matching /^USER\/[0-9a-f-]{36}\.jpg$/ and contentType 'image/jpeg'
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** the four functions (camera/library permission prompts explain why: "to add a photo of your car").
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: pick a large portrait photo → uploaded file opens upright in the Supabase dashboard, ≤ 400 KB.
- [ ] **Step 5: Commit** `feat(photos): pick, compress and upload pipeline`.

### Task 9: Profile setup and profile tab

**Files:**
- Create: `mobile/app/(onboarding)/_layout.tsx`, `profile-setup.tsx`, `mobile/app/profile/edit.tsx`
- Create: `mobile/src/features/profile/validation.ts`, `mobile/src/features/profile/profileService.ts`
- Modify: `mobile/app/(tabs)/profile.tsx` (avatar, display name, @username, TOTAL KM / TOTAL TRIPS stat blocks showing 0 until Part 2, car list from Task 10, SIGN OUT)
- Test: `mobile/src/features/profile/__tests__/validation.test.ts`, `profileService.test.ts`

**Interfaces:**
- Consumes: `supabase`, `useMyProfile`, `is_username_available` (Task 4), `pickPhoto`/`preparePhoto`/`uploadPhoto` (Task 8, for the optional avatar)
- Produces:
  - `normalizeUsername(v: string): string` (trim + lowercase)
  - `validateUsername(v: string): string | null`, `validateDisplayName(v: string): string | null`
  - `checkUsernameAvailable(v: string): Promise<boolean>` (debounced 400 ms in the screen)
  - `createProfile(input: { username: string; displayName: string }): Promise<{ ok: true; profile: Profile } | { ok: false; field?: 'username'; message: string }>`
  - `updateProfile(input: Partial<{ displayName: string; avatarPath: string | null }>): Promise<{ ok: true } | { ok: false; message: string }>` (username is not editable in v1 **[Decided]**)

- [ ] **Step 1: Write failing tests**

```ts
normalizeUsername('  Amir.Drives ') → 'amir.drives'
validateUsername('ab') → 'At least 3 characters'; ('a'.repeat(21)) → 'At most 20 characters'
validateUsername('amir drives') → 'Only letters, numbers, . and _'; ('amir_drives') → null
validateDisplayName('') → 'Enter a display name'; ('x'.repeat(41)) → 'At most 40 characters'
createProfile when supabase returns error code '23505' → { ok:false, field:'username', message:'Username taken' }   // Review Focus 2
createProfile success → profile with camelCase fields
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** the service and screens; setup screen shows live "AVAILABLE / TAKEN" under the username field; on success routes to DRIVE and fires `profile_completed` (Task 11).
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: new account → setup → DRIVE; PROFILE tab shows the entered details; edit display name persists after app restart.
- [ ] **Step 5: Commit** `feat(profile): profile setup, view and edit`.

### Task 10: Cars (add, edit, delete, default, max 3, one photo)

**Files:**
- Create: `mobile/app/cars/new.tsx`, `mobile/app/cars/[id].tsx`, `mobile/src/features/cars/validation.ts`, `mobile/src/features/cars/carService.ts`, `mobile/src/features/cars/useMyCars.ts`, `mobile/src/components/CarRow.tsx`
- Modify: `mobile/app/(tabs)/profile.tsx` (MY CARS section, ADD CAR button)
- Test: `mobile/src/features/cars/__tests__/validation.test.ts`, `carService.test.ts`

**Interfaces:**
- Consumes: `supabase`, cars table + `MAX_CARS_REACHED` (Task 4), `preparePhoto`/`uploadPhoto`/`deletePhoto` (Task 8), `useSession`
- Produces:
  - `type Car = { id: string; make: string; model: string; year: number | null; nickname: string | null; photoPath: string | null; isDefault: boolean }`
  - `MAX_CARS = 3`
  - `validateCar(input: { make: string; model: string; year?: string; nickname?: string }): Partial<Record<'make' | 'model' | 'year' | 'nickname', string>>` (empty object = valid)
  - `addCar(input, localPhotoUri?: string): Promise<{ ok: true; car: Car; photoFailed: boolean } | { ok: false; message: string }>`: inserts the row first (first car becomes default), then uploads the photo and sets `photo_path`; a failed upload returns `photoFailed: true` and keeps the car
  - `setCarPhoto(carId: string, localUri: string): Promise<{ ok: boolean; message?: string }>` (replaces and deletes the old file)
  - `updateCar(carId, input): Promise<{ ok: boolean; message?: string }>`
  - `setDefaultCar(carId): Promise<{ ok: boolean }>` (clears the previous default first)
  - `deleteCar(car: Car, remaining: Car[]): Promise<{ ok: boolean }>`: deletes the photo file and the row; if it was the default, `nextDefaultAfterDelete` becomes default
  - `nextDefaultAfterDelete(cars: Car[], deletedId: string): string | null` (oldest remaining car)
  - `useMyCars(): { cars: Car[]; loading: boolean; refresh(): Promise<void> }`

- [ ] **Step 1: Write failing tests**

```ts
validateCar({make:'', model:'GR86'}) → { make:'Enter the make' }
validateCar({make:'Toyota', model:'GR86', year:'1949'}) → { year:'Year must be 1950–2100' }
validateCar({make:'Toyota', model:'GR86', nickname:'x'.repeat(31)}) → { nickname:'At most 30 characters' }
validateCar({make:'Toyota', model:'GR86'}) → {}
addCar when DB returns message 'MAX_CARS_REACHED' → { ok:false, message:'You can have up to 3 cars' }
addCar with upload rejecting → { ok:true, photoFailed:true }; insert called exactly once                // Review Focus 1
nextDefaultAfterDelete([c1(oldest), c2, c3], c1.id) → c2.id; ([c1], c1.id) → null
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** the service and screens: ADD CAR disabled with "MAX 3 CARS" at 3 cars; SAVE disabled while saving (no double insert); car with no photo shows `PhotoBox` labelled "ADD PHOTO"; DELETE asks "Delete this car? This can't be undone." before deleting; default car marked "DEFAULT".
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: add 3 cars with photos, 4th blocked; switch default; edit; delete with confirmation; airplane mode during save → car saved with "ADD PHOTO", retry works once back online.
- [ ] **Step 5: Commit** `feat(cars): manage up to 3 cars with one photo each`.

### Task 11: Crash reporting and analytics

**Files:**
- Create: `mobile/src/lib/analytics.ts`, `mobile/src/lib/monitoring.ts`
- Modify: `mobile/app/_layout.tsx` (init both), `authService.ts`, `googleSignIn.ts`, `profileService.ts`, `carService.ts` (fire events)
- Test: `mobile/src/lib/__tests__/analytics.test.ts`

**Interfaces:**
- Consumes: `env.sentryDsn`, `env.posthogKey`
- Produces:
  - `initMonitoring(): void` (Sentry; no-op without DSN; no PII; 20% performance sampling)
  - `initAnalytics(): void`, `identify(userId: string): void` (Supabase user UUID only), `track(event: AnalyticsEvent, props?: Record<string, string | number | boolean>): void`
  - `type AnalyticsEvent = 'signed_up' | 'signed_in' | 'profile_completed' | 'car_added' | 'car_photo_failed'`
  - `sanitizeProps(props): props` removes keys `email`, `username`, `displayName`, `display_name`, `name`, `lat`, `lng`, `location`

- [ ] **Step 1: Write failing tests**

```ts
sanitizeProps({ method:'google', email:'a@b.co', username:'x' }) → { method:'google' }
track('signed_up', { method:'email', email:'a@b.co' }) → posthog.capture called with ('signed_up', { method:'email' })
initAnalytics() with no key → posthog never constructed; track is a no-op (no throw)
```

- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement** both modules (PostHog session replay off, autocapture off) and add `track` calls: `signed_up {method}`, `signed_in {method}`, `profile_completed`, `car_added {hasPhoto}`, `car_photo_failed`.
- [ ] **Step 4: Run** `npm test`. Expected: PASS. On phone: events appear in PostHog Live events without email/username; a test crash (temporary button, removed after) appears in Sentry.
- [ ] **Step 5: Commit** `feat: crash reporting and privacy-safe analytics`.

### Task 12: Part 1 acceptance

**Files:**
- Create: `docs/setup/windows-dev-setup.md` (what Task 0 and Task 1 set up, how to run desk mode)

- [ ] **Step 1: Run** `npm run typecheck && npm run lint && npm test && npm run test:db`. Expected: 0 errors, all tests pass.
- [ ] **Step 2: Build** `npx eas-cli build -p android --profile preview` and install. Expected: app works with the PC switched off.
- [ ] **Step 3: Founder walkthrough on the phone:** email sign-up → profile → 3 cars with photos → 4th blocked → sign out → Google sign-in → offline reopen opens to DRIVE. Each item ticked or reported.
- [ ] **Step 4: Commit** the setup doc; ask whether to open a pull request from `feature/part1-foundations` to `main` for the reviewer.
