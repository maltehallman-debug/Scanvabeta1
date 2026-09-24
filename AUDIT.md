# Scanva: pre-launch audit

**Phase 1 (read-only).** No code was changed. Written 2026-09-24 on branch `audit/pre-launch`, starting from `claude/friendly-wozniak-or7i5l` @ `9faf8f6`.

> Legal items below are observations to discuss with the owner. They are not legal advice.

## What the project is (detected)

| | |
|---|---|
| **Repo contents** | Two static files: `index.html` (the marketing site plus the setup wizard) and `demo.html` (the app running on fake data). There is no `package.json`, build step, tests, linter or backend in the repo. |
| **Hosting** | GitHub Pages at `https://maltehallman-debug.github.io/Scanvabeta1/`. The live page is byte-identical to branch **`claude/friendly-wozniak-or7i5l`**, not `main` (md5 `bc6147b9…` in both). Every push to that branch goes live immediately. |
| **The actual product** | A Google Apps Script web app (`Code.gs` + `Index.html`). Its source is embedded base64-encoded in `index.html` (`<script type="text/plain" id="dataCodeGs">` and three siblings). It is installed into each user's own Google account, either by copy-paste or by "Quick install", which calls the Apps Script API from the browser using a Google OAuth token. |
| **Third parties** | Google (OAuth/GSI, Apps Script API, Sheets, Fonts, Gemini API in Pro), cdnjs (Leaflet), OpenStreetMap tiles, Strava API (Pro), Payhip (template sale, off-site). |
| **User data** | The site stores nothing itself: no forms posted, no cookies or localStorage set by its own code, and the OAuth access token is kept only in page memory. The installed app reads and writes the user's own training diary. Pro also sends that history to Gemini and imports from Strava. |
| **Payments** | None on the site. The template is sold through an external link to `payhip.com/b/UcQKA`. |

## Summary (sorted by severity)

| # | Severity | Area | Finding | Status |
|---|---|---|---|---|
| S1 | **High** | Security | Quick install deploys every user's app with `access: 'ANYONE'` + `executeAs: 'USER_DEPLOYING'`, so any signed-in Google user who has the link can read, edit and delete the owner's diary and spend their Gemini/Strava quota | Issue |
| L1 | **High** | Legal / trust | Claim "your data never leaves your own Google account" (plus 2 similar lines) is false for Pro: training history is sent to the Gemini API and Strava data is imported | Issue |
| L2 | **High** | Legal | No privacy policy anywhere | Issue |
| L3 | **High** | Legal | No business details / contact on a site that sells a product (via Payhip). Required by e-handelslagen 8 § and distansavtalslagen | Issue: TODO(owner) |
| S2 | **High** | Security / launch | Google OAuth consent screen looks like it is in *Testing* mode (the code tells users to be "listed as a test user"). Public Quick install needs Google verification, which needs a privacy policy, a homepage and possibly an owned domain | Needs verification |
| A1 | **High** | Accessibility | Core app actions can't be reached by keyboard: activity cards, planned cards, calendar days and the profile header are clickable `<div>`s with no tabindex, role or key handling | Issue |
| A2 | **High** | Accessibility | 20 icon-only buttons in the edit sheet (activity type + intensity) have no accessible name (axe `button-name`, critical) | Issue |
| P1 | **Medium** | Privacy | Google Fonts loaded from Google's servers on the site, the demo and the app (visitor IP goes to Google, which is contested under GDPR; LG München I 3 O 17493/20) | Issue |
| P2 | **Medium** | Privacy | Google Identity Services script (`accounts.google.com/gsi/client`) loads on every page view, before anyone clicks Quick install | Issue |
| L4 | **Medium** | Legal | No terms and no withdrawal/refund information for the paid template (digital content; the 14-day right is lost only with express consent + acknowledgement) | Issue: TODO(owner) |
| L5 | **Medium** | Legal / licensing | OpenStreetMap tiles used with no "© OpenStreetMap contributors" attribution (ODbL + OSM tile policy) | Issue |
| L6 | **Medium** | Legal / licensing | Strava-like logo glyph drawn on the "Get from Strava" button; Strava's brand guidelines require their official assets / "Powered by Strava" | Needs verification |
| S3 | **Medium** | Security | App uses `XFrameOptionsMode.ALLOWALL`: any website can frame the user's app (clickjacking) | Issue |
| A3 | **Medium** | Accessibility | Low contrast: `--text-3` / `--text-faint` `#6b6f67` gives 2.9–3.8:1 on the dark surfaces, used for small labels (needs 4.5:1) | Issue |
| A4 | **Medium** | Accessibility | App sheets and site modals aren't announced as dialogs (no `role="dialog"`/`aria-modal`), focus isn't moved into them or back, and app sheets don't close with Esc | Issue |
| A5 | **Medium** | Accessibility | App form fields use `<div class="fg-label">` rather than `<label for>`, so inputs have no programmatic label | Issue |
| A6 | **Medium** | Accessibility | On phones the app is drawn at 0.7×, so 10px labels render at ~7px. Pinch-zoom still works, but it's hard to read | Needs verification (design decision) |
| P3 | **Low** | Privacy | Part of the owner's personal sheet ID (`1yyyZOZodL4o…`) is printed in a wizard illustration | Issue |
| S4 | **Low** | Security | No Content-Security-Policy. GitHub Pages can't send CSP/X-Frame-Options headers; a `<meta>` CSP is possible (frame-ancestors isn't) | Issue |
| S5 | **Low** | Security | Leaflet loaded from cdnjs without Subresource Integrity | Issue |
| S6 | **Low** | Security | Two unescaped interpolations (swatch/type labels from server constants; avatar data URL in the profile preview). Only self-injection is possible | Issue |
| L7 | **Low** | Trust | Illustrative numbers in site mock-ups ("31 weeks in a row", "21,1 km", "42,7 km") aren't labelled as examples | Issue |
| L8 | **Low** | Trust | "Free forever" is an open-ended promise | Needs verification (owner intent) |
| A7 | **Low** | Accessibility | Month calendar scroller isn't keyboard-focusable (axe `scrollable-region-focusable`) | Issue |
| A8 | **Low** | Accessibility | No skip link on the site | Issue |
| L9 | **Low** | Legal | EU Accessibility Act (tillgänglighetslagen 2023:254) may apply if the template sale counts as an e-commerce service; the microenterprise exemption depends on the owner | Needs verification: TODO(owner) |
| S7 | OK | Security | No secrets in any of the 14 commits, including all decoded embedded files | OK |
| S8 | OK | Security | No `.env`, `.git`, backups or source maps reachable; HTTPS enforced with HSTS | OK |
| — | N/A | Security | Passwords, sessions and cookies, SQL/NoSQL, CSRF, rate limiting on login, DB least-privilege, npm/pip audit, depcheck: none exist in this project (no server, no DB, no package manager) | N/A |

## Details

### Security

**S7. Secret scan of the full history: OK**
- No gitleaks or trufflehog was installed, so I wrote a scanner (`git rev-list --all` → every file in every commit → also **base64-decodes every embedded `<script type="text/plain">` block**, since that's where `Code.gs` lives).
- Patterns checked: Google API keys (`AIza…`), OAuth secrets (`GOCSPX-…`), GitHub tokens, `sk-…`, private keys, Strava-style 40-hex secrets, generic `secret/password/token = "…"`, literal `SHEET_ID = '…'`, Drive-ID-like strings and emails.
- Result over 14 commits: only the Google OAuth **client ID** `956439119970-…apps.googleusercontent.com` (`index.html:1705`), which is public by design, and the placeholder `SHEET_ID = 'PASTE_YOUR_SHEET_ID_HERE'`.
- API keys (Gemini, Strava client ID/secret) are read from Script Properties server-side (`Code.gs` `getStravaService_`, `getTrainingRecommendation`). None are in client code.
- **Nothing to rotate.**
- Needs verification (owner): the OAuth client's *Authorized JavaScript origins* in Google Cloud should list only `https://maltehallman-debug.github.io`. That can't be checked from the repo.

**S1. The Quick-install deployment is open to any Google user: High**
- `index.html:1817`: `webapp: { access: 'ANYONE', executeAs: 'USER_DEPLOYING' }`.
- In Apps Script, `ANYONE` means any signed-in Google account. Because the app runs as the deployer, whoever has the URL gets the owner's full rights over the diary: `getData`, `saveActivity`, `deleteActivity`, `savePlan`. They can also trigger Gemini calls on the owner's key and Strava imports on the owner's token.
- None of the server functions check who is calling (single-user by design), so the deployment's access setting is the only access control.
- The manual wizard says "choose who has access" (`index.html:1071`, `:1293`) without recommending anything.
- **Fix:** change to `access: 'MYSELF'`, and in both wizards say "Who has access: **Only myself**".
- Needs verification before the fix: the app must still open when the owner is signed in on iPhone Safari. The `doGet` comment says sign-in works in a normal Safari tab.

**S2. OAuth app publishing status: Needs verification**
- `index.html:1884` tells users that failures may be because "this Google account is listed as a test user", which suggests the consent screen is in **Testing**.
- In Testing, only up to 100 listed test users can finish Quick install, and tokens expire after 7 days.
- Publishing with the `script.projects` / `script.deployments` scopes likely requires Google verification: a privacy policy URL (see L2), a homepage, and authorized domains the owner controls.
- `github.io` probably can't be used as an authorized domain, so a custom domain may be needed.
- **Owner action:** check Google Cloud → OAuth consent screen.

**S3. `XFrameOptionsMode.ALLOWALL`: Medium**
- Found in `Code.gs` `doGet` (free + Pro builds and the personal build).
- This lets any site embed the user's app in an iframe and overlay it, for example to trick a "Delete" click.
- Nothing in the project frames the Apps Script app (the site frames `demo.html` only).
- **Fix:** remove the call, which falls back to the SAMEORIGIN default.
- Needs verification: confirm the owner doesn't embed the app anywhere.

**S4. No CSP; headers: Low**
- Live response headers: `strict-transport-security: max-age=31556952` ✔, HTTP→HTTPS 301 ✔.
- No CSP, X-Frame-Options, Referrer-Policy or Permissions-Policy. `access-control-allow-origin: *` is harmless for a public static site.
- GitHub Pages doesn't allow custom headers.
- **Fix:** add a `<meta http-equiv="Content-Security-Policy">` allowing exactly `self`, `fonts.googleapis.com`, `fonts.gstatic.com`, `accounts.google.com`, `*.googleapis.com` (the Apps Script API) and `payhip.com` links. Also add `<meta name="referrer" content="strict-origin-when-cross-origin">`.
- Framing protection isn't possible on Pages without a proxy (e.g. Cloudflare) or another host.

**S5. No SRI on the Leaflet CDN: Low**
- The app loads `leaflet.min.js`/`.css` 1.9.4 from cdnjs without `integrity`.
- **Fix:** add the cdnjs SRI hashes. `accounts.google.com/gsi/client` is unversioned, so it can't be pinned, and Google recommends against SRI for it.

**S6. Unescaped interpolation: Low**
- Built `Index.html` (Pro): `' + d.label + '` and `' + t.label + '` in `swatchesHtml`/`typeButtonsHtml`. These are server constants.
- `'<img src="' + curImgSrc + '">'` in `openProfileSheet` uses the user's own avatar data URL.
- Only self-XSS is possible. All sheet text, Strava names and AI output are escaped (`escapeHtml`) or set with `textContent` (verified by grep; the chat uses `div.textContent`).
- **Fix:** wrap all three in `escapeHtml`.

**Other security items**
- **Debug / verbose errors:** server errors reach the UI as `err.message` (e.g. "Could not read the sheet. …"). These are Apps Script messages without stack traces. Low, acceptable.
- **Dependencies:** there's no package manager. Runtime third-party code is Leaflet 1.9.4 (cdnjs), GSI (Google) and the OAuth2 Apps Script library (`1B7FSrk5…`, googleworkspace/apps-script-oauth2, Apache-2.0). `npm audit`/`pip-audit`/depcheck: N/A.
- **Authentication:** handled entirely by Google (OAuth token client in the browser, Apps Script session in the app). The token is never stored (in memory only, `index.html` `tokenClient` callback). Requested scopes are minimal for installing: `script.projects script.deployments`. **OK.**
- **CORS / CSRF / SQL / passwords / rate limiting / DB RLS:** N/A (no server or DB of our own). Apps Script quotas are the only limiter; see S1 for the abuse path.
- **Exposed files (S8):** `/.git/config`, `/.env`, `/index.html.bak`, `/demo.html.map`, `/.DS_Store` all return 404 on the live site. **OK.**

### Privacy & data (GDPR / ePrivacy)

**Data inventory: what the code actually does**

| Where | Data | Sent to | Why |
|---|---|---|---|
| Site, every visit | IP, user agent | GitHub (hosting logs), Google Fonts, `accounts.google.com` (GSI script) | Hosting / fonts / Quick install button |
| Site, Quick install | Google OAuth access token (memory only); sheet link typed by the user | Google (Apps Script API) | Create and deploy the user's project |
| Demo | IP, user agent | GitHub, Google Fonts, cdnjs | Hosting / fonts / Leaflet |
| Installed app | Training diary (sessions, comments, times, distances, intensity), profile name/subtitle/photo, goal times | The user's own Sheet + Script Properties | Core feature |
| Installed app | IP | Google Fonts, cdnjs; OpenStreetMap tile servers when a Strava map is shown | Fonts / map |
| Pro | Last ~14 days of training, next 7 days of plans, 10 weeks of volume, goal times, chat questions | **Google Gemini API** (with a key the user supplies, possibly from another Google account) | AI tip / plan / chat |
| Pro | Strava activities (name, time, distance, route polyline); OAuth token stored in the deployer's User Properties | Strava API ↔ user's Sheet | Import |

- **Cookies set by our own code:** none. There's no `document.cookie`, `localStorage` or `sessionStorage` in the site, demo or app (grep: 0 hits).
- The GSI client may set Google cookies. Needs verification in a browser with a clean profile.
- **P1 Google Fonts: Medium.** Loaded from `fonts.googleapis.com`/`fonts.gstatic.com` in `index.html`, `demo.html` and the app. **Fix:** self-host Geist and Geist Mono (OFL licence). For the site, add the `.woff2` files to the repo. For the app, Apps Script can't serve binaries, so inline the fonts as base64 `@font-face` or fall back to system fonts.
- **P2 GSI script before any click: Medium.** `<script src="https://accounts.google.com/gsi/client" async defer>` at `index.html:1697`. **Fix:** load it only when the user clicks Quick install (the code already checks `isAutoSetupAvailable()`).
- **Consent banner:** not needed today, since there are no analytics, trackers or pixels (verified: no hosts other than those listed). After P1 and P2 are fixed there's nothing non-essential left to consent to.
- **Forms:** only the Quick install sheet-link field. No pre-ticked boxes. The fine print says "Nothing leaves your own Google account" (see L1). No link to a privacy policy (see L2).
- **Privacy policy (L2): High, missing.**
- **Cookie policy:** not strictly needed while no cookies are set. Mention it in the privacy policy instead.
- **P3 partial personal sheet ID in an illustration: Low.** `index.html:1000` and `:1147` (`1yyyZOZodL4o…`). A 12-character prefix isn't usable on its own, but it's the owner's real sheet. **Fix:** replace it with an obviously fake ID.

### Legal & trust

- **L1 misleading privacy claim: High.** Three lines:
  - Hero, `index.html:634`: "…so your data never leaves your own Google account."
  - FAQ, `index.html:909`: "nothing is copied anywhere else."
  - Auto-setup fine print, `index.html:1524`: "Nothing leaves your own Google account."

  Pro sends training history to the Gemini API, and depending on the key's account and tier, Google may use it to improve its products. Pro also imports from Strava. This is a false statement under marknadsföringslagen 10 § and a GDPR transparency (Art. 13) problem. **Fix:** reword to what's true, e.g. "Your diary stays in your own Google account. Pro's AI features send your recent training to Google's Gemini API using your own key."
- **L2 privacy policy: High.** Draft a page listing exactly the inventory above, with the controller identity as `TODO(owner)`. Mark it clearly as a draft that isn't legal advice.
- **L3 business details: High, TODO(owner).**
  - The site sells a product (template via Payhip) but shows no seller name, organisation number / personnummer-based firm, address or email.
  - Required by lagen om elektronisk handel (2002:562) 8 §, and distansavtalslagen 2 kap. 2 § for the sale itself.
  - I won't invent any of these. **Fix:** add a footer "Contact / Company" block of `TODO(owner):` placeholders.
- **L4 terms and withdrawal: Medium.**
  - A digital download sold to consumers has a 14-day right of withdrawal. It's lost only if the buyer expressly consents to immediate delivery and acknowledges losing the right (distansavtalslagen 2 kap. 11 § 13).
  - Needs verification: whether Payhip's checkout collects that consent and whether Payhip or the owner is the seller of record.
  - **Fix:** draft terms and withdrawal text as `TODO(owner)` drafts. No refund promises get invented.
- **L5 OpenStreetMap attribution: Medium.** App: `L.tileLayer('https://{s}.tile.openstreetmap.org/…', { maxZoom: 18 })` has no `attribution`. **Fix:** add `attribution: '© OpenStreetMap contributors'`. Also note the OSM tile usage policy (fine at this volume; set a proper `Referer`).
- **L6 Strava branding: Needs verification.**
  - The "Get from Strava" / "Connect to Strava" buttons use a CSS-masked chevron glyph resembling Strava's logo.
  - Strava's API brand guidelines require their official "Connect with Strava" button and "Powered by Strava" logo, and restrict look-alike marks.
  - **Fix:** use the official assets or remove the glyph.
- **L7 illustrative numbers: Low.** Site mock-ups show example figures ("42,7 km", "31 weeks in a row", "Longest run 21,1 km", "Fastest pace 4:34/km"). They're UI illustrations, not testimonials. **Fix:** add a small "Example data" caption to the feature grid.
- **Testimonials / reviews:** none found. **OK.**
- **Superlatives** ("best", "#1", "guaranteed"): none found. "Log in seconds" is mild puffery. **OK.**
- **L8 "Free forever": Needs verification.** Only keep it if the owner commits to it.
- **Images / licensing:** there are no raster images. All graphics are inline SVG drawn for this project, and the favicon is inline SVG. Fonts are Geist and Geist Mono (SIL OFL, fine). Leaflet is BSD-2 and needs its attribution kept (Leaflet's control shows it). OSM data: see L5. **Nothing of unknown licence.**
- **L9 EU Accessibility Act: Needs verification, TODO(owner).** It applies from 28 June 2025 to e-commerce services sold to consumers. Microenterprises (<10 staff and ≤ €2 M turnover) are exempt for services. The A-items below bring the site close to WCAG 2.1 AA either way.

### Accessibility (WCAG 2.1 AA)

Tool: axe-core 4.13.0 run through Playwright/Chromium (WCAG 2.0/2.1 A+AA rules) on eight states: site desktop, site phone, site wizard open, demo Home/Progress/Activities/AI, and the edit sheet.

- Some contrast hits were measured mid fade-in (e.g. `#383a36`) and are false positives.
- The real failures are the dim grey `#6b6f67` on dark surfaces (3.28–3.82:1). Replacing it with `#8b8f86` gives ≥ 4.54:1 on every surface used (computed).

| ID | Finding | Evidence | Fix |
|---|---|---|---|
| A1 High | Clickable `<div>`s, not keyboard-reachable | App `activityCardHtml`/`plannedCardHtml` (`<div class="activity-card" data-row=…>`), month cells `<div class="month-day has-activity">`, `#headerNameBtn` div | Render as `<button>` (or add `tabindex="0"` `role="button"` + Enter/Space handler) with a descriptive label |
| A2 High | 20 icon-only buttons with no name | axe `button-name` on `.type-btn` ×10, `.swatch` ×10 (edit sheet) | `aria-label="Running"` / `"Low intensity"` and `aria-pressed` for the selected one |
| A3 Medium | Contrast < 4.5:1 | axe: `.page-eyebrow` 3.82, `.card-meta` 3.54, `.kpi .l` 3.28, `.day-name` 3.54, site `.fact span` / `.mini-tile .l` 3.31 | `--text-faint` / `--text-3` → `#8b8f86` |
| A4 Medium | Dialogs not announced, focus not managed | `grep -c 'role="dialog"'` = 0 in both files; app has no Esc handler for sheets (only `aiChatInput` keydown) | `role="dialog" aria-modal="true" aria-labelledby`, move focus in, restore on close, Esc closes, trap Tab |
| A5 Medium | Inputs without labels | `<div class="fg-label">Name</div>` + `<input>` (profile, activity, plan, goal sheets) | `<label for>` or `aria-labelledby` |
| A6 Medium | 0.7× phone scale | `html.is-phone { --ui-scale: 0.7 }` | Owner decision: 0.8 would keep most text ≥ 8px, or raise the smallest label sizes |
| A7 Low | Month scroller not focusable | axe `scrollable-region-focusable` `#monthScroll` | `tabindex="0"` + `aria-label` |
| A8 Low | No skip link on the site | — | "Skip to content" link before the nav |
| OK | `lang="en"`, unique `<title>`s, iframe has `title`, heading order h1→h2→h3, visible `:focus-visible` on the site, `prefers-reduced-motion` respected, pinch-zoom not disabled | grep + axe (no heading/landmark/title/lang violations) | — |

## Proposed Phase 2 plan (waiting for approval)

**The branch matters because of how Pages is set up.**
- Fixes will be committed to `audit/pre-launch`, one commit per finding. Nothing goes live until it's merged into the branch Pages serves (currently `claude/friendly-wozniak-or7i5l`).
- The app fixes also have to go out to already-installed users. They re-paste `Code.gs`/`Index.html` or re-run Quick install. The owner's **personal build** needs the same `Code.gs` changes (S1 only applies there if it's deployed as "Anyone"; S3 applies).

**Order of work**
1. **High:**
   - S1: `MYSELF` + wizard wording.
   - L1: reword the three claims.
   - L2: draft `privacy.html`, marked DRAFT, controller = TODO(owner).
   - L3: footer contact block with TODO(owner).
   - A1 + A2: keyboard-operable cards and labelled buttons.
   - S2: owner action only; I'll write the checklist.
2. **Medium:**
   - P1: self-host fonts on the site; app falls back to system fonts or inline fonts (your choice).
   - P2: lazy-load GSI.
   - L4: draft terms + withdrawal text as TODO drafts.
   - L5: OSM attribution.
   - L6: remove the look-alike glyph until official Strava assets are added.
   - S3: drop ALLOWALL.
   - A3, A4, A5.
   - A6: only if you want it changed.
3. **Low:** S4 meta CSP + referrer, S5 SRI, S6 escaping, P3 fake ID in the illustration, L7 "Example data" caption, A7, A8.

**Verification after the fixes:** there's no build/lint/tests in the repo, so I'll use JS syntax checks of every inline script, a Playwright smoke test of the site and demo, re-run the secret scan and the axe scan, and diff the decoded embedded sources.

**Questions for you**
1. **S1:** OK to switch Quick install to "Only myself"? Is there any reason other people need access to a user's app?
2. **P1 in the app:** system fonts (simplest, no network) or inline base64 Geist (~120 KB extra per load)?
3. **A6:** keep 0.7× or go to 0.8×?
4. Business details for L3/L4: provide them now or leave TODO(owner) placeholders?
