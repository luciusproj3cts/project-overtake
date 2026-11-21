# OVERTAKE Design System (v1)

**Status:** Approved by founder (2026-11-21)
**Direction:** C · Bold / brutalist (chosen 2026-11-21)
**Reference mockups:** https://claude.ai/artifact/MNgWoMi8ybqf7ViMRSmKSd (row C)

Loud, poster-like and high-contrast: heavy type, hard edges, thick black borders, and one electric accent. Every screen should look like it could be printed as a street poster.

All values below become **code tokens** (one shared theme file in the app), so the look can be tuned in one place.

---

## 1. Colour

| Token | Hex | Use |
| --- | --- | --- |
| `paper` | `#ECEAE3` | App background |
| `ink` | `#0A0A0A` | Text, borders, dark blocks |
| `accent` | `#D4FF1F` | Lime: primary buttons, active tab, highlights, the route line on cards |
| `muted` | `#5A5A55` | Secondary text (meets 4.5:1 contrast on `paper`) |
| `paper-deep` | `#CFCCC2` | Placeholder / image backgrounds |
| `danger` | `#B3261E` | Errors and destructive actions (Delete account, Report) |
| `success` | `#1B6E31` | Confirmations, e.g. "Trip saved" |

**Rules**
- Lime is **always paired with black text**, never white. On black blocks, lime may be used as the text colour.
- Only one accent: no second highlight colour.
- No gradients, no transparency effects, no blurred shadows.
- **Dark mode:** not in v1. The app is light (`paper`) by default; black `ink` blocks give contrast. **[Open: revisit after launch]**

## 2. Typography

| Role | Font | Notes |
| --- | --- | --- |
| Display (headings, big numbers, buttons) | **DX Brutalist** (Fontshare) | Free for commercial use; the licence file is checked and stored in the repo when the font is added. Fallback while testing: Archivo Black (Google Fonts). |
| Body, labels, numbers in tables | **Space Mono** (Google Fonts) | Regular 400, Bold 700 |

**Type scale (in density-independent pixels, dp)**

| Token | Font | Size / line height | Example |
| --- | --- | --- | --- |
| `hero` | Display | 120 / 114 | Live distance on the Drive screen |
| `h1` | Display | 44 / 42 | Screen titles ("RANKS") |
| `h2` | Display | 26 / 28 | Section titles, big buttons |
| `stat` | Display | 24–38 / 1.0× | Stat blocks |
| `body` | Space Mono 400 | 14 / 20 | Post text, descriptions |
| `label` | Space Mono 700 | 11–13 / 16 | Field labels, tab labels, "DIST/KM" |

**Rules**
- Headings, labels and buttons are **UPPERCASE**. Body text (posts, comments, descriptions) stays in normal case for readability.
- Units go in labels ("DIST/KM"), not after numbers, to keep numbers big and clean.
- Text scales with the user's Android font-size setting; layouts must not break at 1.3×.

## 3. Shape & spacing

| Token | Value |
| --- | --- |
| Border (main) | 3 px solid `ink` |
| Border (dividers, list rows) | 2 px solid `ink` |
| Corner radius | **0**: everything is square |
| Hard shadow (primary buttons, "You" row) | 5–6 px offset right and down, solid `ink`, no blur |
| Spacing scale | 4 · 8 · 12 · 16 · 24 · 32 |
| Screen side margin | 16 |
| Minimum touch target | 44 × 44 (bottom tabs 56 tall) |

## 4. Core components

| Component | Spec |
| --- | --- |
| **Primary button** | Lime fill, black display text (uppercase), 3 px black border, hard shadow; pressed: shadow disappears and the button shifts 5 px (feels "pushed in"). Height 56–72. |
| **Secondary button** | Transparent, black border, black text. |
| **Segmented control** (Distance/Trips, This week/All-time) | Full-width row split by black borders; selected = black fill + lime text, or lime fill + black text. |
| **Stat block** | Label (Space Mono bold, small) above a big display number, in a bordered grid cell. Top speed cell is inverted (black fill, lime text). |
| **List row** (leaderboard, trips) | 2 px bottom border; rank in display font; #1 row highlighted lime. |
| **Pinned "You" row** | Lime fill, black border, hard shadow; shows rank, value, and distance to the next rank. |
| **Bottom tabs** | 5 equal cells with black dividers; text labels (DRIVE · TRIPS · RANKS · COMMUNITY · PROFILE); active tab = lime fill. |
| **Recording bar** | Black strip with lime "● REC" and the car selector. |
| **Text input** | Paper background, 3 px black border, Space Mono; focus = border thickens to 4 px and the label gets a lime background. Error = `danger` border + message. |
| **Photo placeholder** | `paper-deep` block with an uppercase label. |
| **Icons** | Simple square-capped line icons (2 px), used sparingly. Tabs use text labels. |

**Tab names [Decided]:** DRIVE · TRIPS · RANKS · COMMUNITY · PROFILE. Labels are 11 px Space Mono bold so the longer names fit five equal tabs.

## 5. Trip card (shareable)

- Size: 1080 × 1920 px (Instagram/TikTok story), rendered on the phone.
- Background: lime (`accent`).
- Top: stacked "OVER / TAKE" wordmark + date.
- Centre: **black panel with the route line in lime**, square line caps, square start (filled) and end (outlined) markers; full route, no map, no place names.
- Stats grid: DIST/KM · TIME/MIN · AVG/KMH · TOP/KMH (top speed cell inverted).
- Bottom: car photo + car name + @username in a bordered strip.

## 6. Motion

Minimal and snappy: 120–180 ms, no bounce. The only "signature" motion is the primary button press (shadow collapse). Live numbers on the Drive screen update without animation, so they are readable at a glance.

## 7. Accessibility

- Text contrast ≥ 4.5:1 (all token pairs above pass).
- Never rely on lime vs. paper alone to show state: selected items also change fill to black or gain a border.
- Every icon-only button has a spoken label for screen readers.
