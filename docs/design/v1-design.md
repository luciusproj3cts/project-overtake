# OVERTAKE: Version 1 Design

**Status:** Approved by founder (2026-11-21)
**Date:** 2026-11-21
**Scope:** First public Android release (target: ~3 months from start of build)

Markers used in this document:
- **[Decided]**: decided by the founder in planning.
- **[Claude's call]**: proposed by Claude; the founder can override.
- **[Open]**: not decided yet; must be settled before or during the build.

---

## 1. What OVERTAKE is and why

OVERTAKE is "Strava for car users": a mobile app that records a user's car trips, shows their driving stats, ranks them on leaderboards, and lets them share good-looking trip cards to social media. Direct competitors are Enroad and Wheelz. It also has a community forum for car enthusiasts.

**Why now:** the team wants to validate demand with a real product before committing to company registration, monetisation, or investment. Version 1 must be good enough that people keep using it, and must collect the data needed to make later decisions.

**Success for v1:** a stable Android app on Google Play in Malaysia that records trips reliably across common phone brands, with analytics in place to measure retention and feature use. The iOS launch is gated on a monthly-active-user (MAU) target, still to be set.

## 2. Goals, non-goals, anti-goals

### Goals (v1)
1. **Trip tracking:** record car trips and show stats: distance, duration, average speed, top speed. **[Decided]**
2. **Leaderboards:** rank users by total distance and by number of trips. **[Decided]**
3. **Shareable trip cards:** an image summary of a trip, shared via Android's share menu. **[Decided]**
4. **Community page:** a posts/forum feature. **[Decided]**
5. **Analytics:** user data must be analysable, either through analytics platforms or directly. **[Decided, PRD]**

### Non-goals (v1; may come later)
- Themes and driver personality badges: after launch. **[Decided]**
- Automatic trip detection: v1 is manual start only; the code must allow auto-detect to be added later. **[Decided]**
- iOS app: after the Android MAU target is reached. **[Decided]**
- SMS / phone-number sign-in: dropped. **[Decided]**
- Monetisation: to be decided later. **[Decided]**
- Virtual garage, hardware track leaderboard (RaceBox/Dragy): dropped. **[Decided]**
- Countries outside Malaysia. **[Decided]**
- Car-meet/events listings, clubs/groups, a trip-card feed. **[Decided: not chosen for v1]**

### Anti-goals (failure even if it "works")
- **Leaderboards that reward speed.** Leaderboards must never rank by speed. **[Decided]** Top speed may appear on a user's own trip card. **[Decided]**
- **Unreliable recording:** trips that silently stop recording, or stats that are obviously wrong (e.g. 400 km/h spikes). This destroys trust in the core feature.
- **Battery drain** so heavy that users uninstall.
- **Leaking where users live or work** through in-app data (trips and routes are private to their owner). Shared trip cards show the full route line by founder's choice; see Risks.
- **Running costs above RM0/month** before launch. **[Decided: free tiers only]**

## 3. Constraints

| Constraint | Detail |
| --- | --- |
| Builder | Founder (no coding background) working with Claude Code; an external reviewer will review the code later. Code must be conventional and well documented. |
| Timeline | First Android release in ~3 months. |
| Budget | Free tiers only for now. |
| Region | Malaysia only: km and km/h, English UI, Malaysian time zone (Asia/Kuala_Lumpur), Malaysia's PDPA privacy law. |
| Store | Google Play. New personal developer account → **mandatory closed test with 12+ testers for 14 consecutive days** before public release. |
| Founder rules (CLAUDE.md) | Plan and get approval before executing; ask before committing; separate branches for testing; max 3 fix attempts before escalating; priorities: quality, security, performance/battery/GPS & data accuracy. |

## 4. Approaches considered

| | A. Expo (React Native) | B. Flutter | C. Native Kotlin |
| --- | --- | --- | --- |
| Codebase | TypeScript; Android + later iOS | Dart; Android + later iOS | Android only |
| GPS/battery control | Good (Android's fused location provider via a foreground service; custom native module possible) | Good | Best |
| iOS later | Mostly reused | Mostly reused | Full rewrite in Swift |
| Fit with existing plan | Matches the planned Expo/EAS → Play Console workflow | Different toolchain | Different toolchain |
| Speed to v1 | Fastest | Similar | Slowest |

**Chosen: A. Expo + Supabase. [Decided]**
Reasoning: iOS is planned (rules out C); the founder's workflow already assumes Expo/EAS (B offers no real gain for the switch); GPS reliability across phone brands is the main risk and is equally hard in A and B.

**Backend alternatives:** Supabase (Postgres) vs Firebase (Firestore). Supabase chosen because leaderboards and analytics need a SQL database, and standard Postgres connects to almost any analytics tool. **[Decided]**

## 5. Tech stack

| Layer | Choice | Free-tier notes |
| --- | --- | --- |
| Mobile app | Expo (React Native, TypeScript), development builds (not Expo Go, because background location needs native config) | Free |
| Location | `expo-location` + `expo-task-manager`, Android foreground service (type: location), high-accuracy mode | Free |
| Local storage (on phone) | SQLite (`expo-sqlite`) for in-progress trips and the upload queue | Free |
| Backend | Supabase: Postgres database, Auth, Storage (photos), Row Level Security, Edge Functions | Free: 500 MB DB, 1 GB file storage, 50K monthly active users; pauses after ~7 days without DB activity |
| Sign-in | Supabase Auth: Google sign-in + email/password. Apple sign-in added with iOS. **[Decided]** | Free |
| Maps | Google Maps SDK for Android (`react-native-maps`) | Mobile native maps listed at USD 0. Billing card required on Google Cloud. **Budget alert at USD 1 + caps on paid APIs.** **[Decided]** |
| Place names (in-app trip history only) | Android's on-device geocoder (via `expo-location`), not Google's paid Geocoding API | Free |
| Trip card image | Route drawn as a line with `react-native-svg` (no map tiles), card captured on the phone with `react-native-view-shot`, shared via Android share sheet | Free; no Google Maps usage |
| Usage analytics | PostHog (React Native SDK), EU or US cloud **[Claude's call]** | Free: 1M events/month. Session replay off. |
| Crash reporting | Sentry (React Native SDK) | Free: developer tier |
| Builds/releases | EAS Build → `.aab` → Google Play Console | Free tier (queued builds) |
| Code hosting | GitHub (`luciusproj3cts/project-overtake`) | Free |

## 6. Feature design

### 6.1 Trip tracking (core)

**Starting/stopping [Decided: manual]**
- User taps **Start drive**. A persistent "Recording trip" notification appears (Android foreground service). Recording continues with the screen off or while using other apps (Waze, Spotify).
- User taps **Stop** (in the app or on the notification).
- Only "While using the app" location permission is required. A clear explanation screen ("prominent disclosure") is shown before the permission prompt, as Google Play requires.
- **Forgotten stop:** if the car is stationary for 15 minutes, the trip auto-ends at the moment movement stopped, and the user is notified. **[Claude's call]**
- **Battery-optimisation guidance:** on first use, detect the phone brand and, if it is known for killing background apps (Xiaomi, Oppo, Vivo, Huawei, Samsung), show a one-time guide to exempt OVERTAKE. **[Claude's call]**
- Code is structured so a future "auto-detect" trigger can start the same recording service. **[Decided]**

**GPS sampling & accuracy [Claude's call]**
- Request location roughly every 1 second, high accuracy.
- Discard fixes with reported accuracy worse than ~25 m.
- **Speed** uses the speed the GPS chip reports (Doppler-based; more accurate than calculating from position jumps).
- **Top speed** = highest speed held across at least 3 consecutive good-accuracy fixes, to ignore one-off GPS spikes.
- **Distance** = sum of distances between filtered fixes, ignoring jitter while stationary.
- **Average speed** = distance ÷ *moving* time (time stopped in traffic excluded). Total duration is shown separately.
- **Plausibility checks:** trips with sustained speeds above 300 km/h, or teleport-like jumps, are flagged and excluded from leaderboards.

**Data flow & storage (free-tier critical) [Claude's call]**
1. During a trip, raw GPS points are saved only on the phone (SQLite). Nothing needs internet.
2. When the trip ends, the phone calculates stats and simplifies the route (removes redundant points, ~5 m tolerance) into a compact encoded line of a few KB.
3. The phone uploads the trip summary + simplified route to Supabase. If offline, it queues and retries later.
4. Raw points are deleted from the phone after a successful upload, but kept for 7 days for debugging during closed testing.
- Why: 1 hour of raw GPS ≈ 3,600 points. Storing raw points on the server would fill the free 500 MB database within weeks. A simplified route keeps each trip to a few KB.

**Trip record fields:** id, user, car, start/end time, duration, moving time, distance, average speed, top speed, simplified route, start/end area names (shown only in the owner's in-app trip history **[Claude's call]**), qualifies-for-leaderboard flag, created/updated timestamps.

**Trip history:** list of the user's trips (newest first) with a map + stats detail screen. Users can delete their own trips.

### 6.2 Leaderboards

- **Two boards [Decided]:** *Distance* (total km) and *Trips* (number of trips).
- **Two time windows [Decided]:** *This week* (resets Monday 00:00 Malaysian time) and *All-time*.
- **Who is ranked [Decided]:** everyone in Malaysia on one board.
- **Qualifying trips [Claude's call]:** at least 1 km, passed plausibility checks, recorded via the app (no manual entry). This stops gaming the Trips board with tiny trips.
- **Display [Claude's call]:** top 100 + the user's own rank and value, shown even if outside the top 100.
- **Computation [Claude's call]:** calculated in the database (Postgres views/functions), refreshed on a schedule (e.g. every 15 minutes) so it stays fast as users grow.
- **Integrity [Claude's call]:** for v1, stats are calculated on the phone and accepted by the server after plausibility checks. Stronger server-side verification can be added if cheating appears.

### 6.3 Shareable trip cards

- Generated on the phone from a finished trip; shared via Android's share menu (Instagram, WhatsApp, TikTok, etc.) or saved to gallery.
- **Route line, not a map [Decided]:** like a Strava story, the card shows only the route as a line on a 2D plane, with no map, streets or labels behind it. The line is drawn from the stored simplified route, scaled and centred to fit the card.
- **Full route line [Decided]:** the line is not trimmed at the start/end.
- **No place names on the card [Decided].**
- **Content [Decided]:** route line, stats including top speed, the car's photo.
- **Content details [Claude's call]:** distance, duration, average speed, top speed, date, car name + photo, OVERTAKE branding.
- **Map usage:** Google Maps is used only inside the app (live drive screen, the owner's trip detail screen), never on the shareable card.
- **Visual design [Decided: Claude proposes, founder approves]:** before any UI is built, Claude proposes a simple design system (colours, Google Fonts, core components) with mockups of key screens, including the trip card. The founder picks/adjusts.

### 6.4 Community (posts / forum) [Decided: posts/forum]

**v1 shape [Decided]:**
- A single feed of posts (newest first). No categories/tags in v1.
- A post has text + up to 4 photos (compressed on the phone).
- Comments on posts. Likes on posts.
- Users can edit/delete their own posts and comments. **[Claude's call]**

**Required by Google Play for user-generated content:**
- Users accept terms that define objectionable content before posting.
- **Report** a post, comment or user. **Block** a user (their content disappears for you).
- A way for the team to review reports and hide content. v1: done through the Supabase dashboard (no custom admin app). **[Claude's call]**
- Rate limiting to slow spam (e.g. max posts per hour). **[Claude's call]**

### 6.5 Profiles & cars

- **Profile:** display name, unique username, profile photo (optional), stats summary (total km, total trips).
- **Visibility [Decided: public profile, private trips]:** other users (e.g. tapping a name on the leaderboard or forum) see display name, username, profile photo, cars, and total km/trips. Individual trips and routes are private to the owner; they only leave the app when the user shares a trip card.
- **Cars [Decided]:** users can add **multiple cars**, each with photos. Each trip is assigned to a car (default car pre-selected at Start). Each user can have a max of 3 cars.
- **Car fields [Claude's call]:** make, model, year (optional), nickname (optional), photo(s).
- **Car photo appears on [Decided]:** profile page and trip cards.
- **Photo handling [Claude's call]:** resized/compressed on the phone (~1280 px, ~200–300 KB) before upload, to fit the 1 GB free storage.

### 6.6 Accounts, privacy & compliance

- Sign-in: Google + email/password. **[Decided]**
- **In-app account deletion** and a web link to request deletion: required by Google Play. Deleting an account deletes trips, cars, photos and posts. **[Claude's call: posts deleted, not anonymised]**
- **Privacy policy** (hosted web page) covering location data, analytics, crash reporting, PDPA rights. Required by Google Play. Founder to review; Claude drafts.
- **Google Play forms:** Data Safety form, foreground-service (location) declaration, location permission justification, content rating, UGC policy.
- **Data protection:** Row Level Security on every table. Users can only read/write their own private data; public data (leaderboard entries, posts, public profile fields) is exposed deliberately.
- Secrets never committed: `.env` is git-ignored; EAS secrets are used for builds.

### 6.7 Analytics

- **PostHog** records app usage events, for example: `signed_up`, `trip_started`, `trip_completed`, `trip_auto_ended`, `trip_card_shared`, `leaderboard_viewed`, `post_created`, `car_added`. Used for retention curves, funnels, feature usage. **[Claude's call: event list]**
- **Direct data analysis:** Supabase is standard Postgres, so it can later be connected to Metabase, Looker Studio, or Google Sheets for custom reports, satisfying the PRD requirement. No extra work needed in v1.
- **Sentry** for crashes and performance.
- No personal data (emails, exact locations) is sent to PostHog. **[Claude's call]**

## 7. App structure

**Bottom tabs [Claude's call]:**
1. **Drive:** big Start/Stop button, live distance/time while recording, car selector.
2. **Trips:** history list → trip detail → share card.
3. **Leaderboard:** Distance / Trips × This week / All-time.
4. **Community:** feed → post → comments; create post.
5. **Profile:** stats, cars, settings, sign out, delete account.

**Repository layout [Claude's call]:**
```
/app            Expo mobile app (TypeScript)
/supabase       database migrations, Row Level Security policies, edge functions
/docs           design docs, privacy policy draft, store listing text
CLAUDE.md       instructions for Claude Code
```

**Android app ID (package name) [Open]:** permanent once uploaded to Google Play. Not needed to start building; must be decided before the first upload to Play (closed testing, ~week 10). Options discussed: `com.bamboo.overtake`, `com.overtake.app`, or based on a domain the team owns.

**Environments [Claude's call]:** one Supabase project for development now; a second ("production") created before the closed test. The free plan allows 2 projects.

## 8. Testing & quality

- **Trip calculations** (distance, speeds, filtering, route simplification, privacy trimming) are pure functions with automated tests, using recorded GPS samples, including deliberately bad data (spikes, tunnels, traffic jams).
- **Database security tests** confirm users can't read or modify each other's private data.
- **Real-device testing:** founder's Android phone first; during closed testing, cover at least Samsung, Xiaomi and one other brand popular in Malaysia (Oppo/Vivo/Huawei).
- Reviewer-friendly: TypeScript strict mode, linting, consistent formatting, comments explaining *why* for GPS/stat logic.

## 9. Milestones (~12 weeks) [Claude's call]

| Weeks | Milestone |
| --- | --- |
| 1–2 | Design system proposal + key-screen mockups (founder approval); project setup, Supabase, sign-in, profiles, cars + photos |
| 3–5 | Trip recording, stats, trip history. Heavy real-world driving tests |
| 6 | Trip cards + sharing |
| 7 | Leaderboards |
| 8–9 | Community forum + report/block/moderation |
| 10 | Polish, privacy policy, account deletion, Play Store forms, production Supabase project |
| 10–12 | **Closed test: 12+ testers, 14 consecutive days** → fixes → apply for production → public release |

Founder actions with lead time: decide the Android app ID before ~week 10; create the Google Play developer account (identity verification takes days); recruit 12+ testers by ~week 9; create a Google Cloud account with a billing card for Maps.

## 10. Risks

| Risk | Mitigation |
| --- | --- |
| Phone brands kill the recording service | Foreground service, battery-optimisation guide, multi-brand testing, local storage so partial trips aren't lost |
| GPS errors produce wrong stats (especially top speed) | Accuracy filtering, Doppler speed, sustained-speed rule, plausibility checks, automated tests with bad data |
| Free database fills up | Simplified routes only on server; compressed photos; monitor usage |
| Closed-test requirement delays launch | Recruit testers early; start closed test by week 10 |
| Forum abuse / Google Play UGC rejection | Terms, report/block, moderation via dashboard, rate limits |
| Top speed on shareable cards encourages speeding (legal/reputational) | Accepted by founder; leaderboards never use speed |
| Full, untrimmed route line on shared cards could let people who know the area recognise a user's home/work from the line's shape | Accepted by founder; no map or place names on the card reduces it. Trimming can be added later as a setting. |
| 3-month timeline slips | Side features already deferred; forum is the most cuttable scope if needed |

## 11. Left to the builder

Library versions, exact file structure inside `/app`, UI component choices, exact filtering thresholds (tuned with real data), database index design, and the leaderboard refresh mechanism, all within the decisions above.
