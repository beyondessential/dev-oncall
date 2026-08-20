# Dev on-call page — what it is, what it's written in, and how it works

This note documents the "Which dev should I call?" website used by the Tamanu support team. It explains what the page is built from, the logic running inside it, and the general software concepts that apply to a site like this — so anyone on the team can understand, maintain, or extend it.

**Live site:** https://beyondessential.github.io/dev-oncall/
**Roster source of truth:** the "Tamanu on call roster" Google Sheet (Roster tab).
**Escalation policy source:** the decision tree agreed in #tamanu-support.

## What kind of website this is

This is a *static, single-file website*. The entire site is one file, `index.html`, and there is no server-side program behind it: no backend, no database, no login system. When someone opens the URL, GitHub Pages simply hands their browser that one file, and everything that happens after that — the clocks ticking, the roster loading, the escalation ladder being chosen — happens inside the visitor's own browser. This is the simplest and cheapest way to run a site, and it is the right fit here because the page only needs to *display and compute*, never to *store* anything.

The practical consequence of "static" is that the page itself cannot remember or save anything for other people. That is why the roster lives in the Google Sheet (which *is* a shared, editable data store) and the page only reads from it.

## The three languages inside the file

Although it is one file, it contains three languages, each doing a different job — this division of labour is universal to essentially every website.

**HTML** provides the structure and content: the headings, the tabs, the buttons with the developers' names, the table skeleton on the Roster tab. If you open the file, everything between tags like `<div>`, `<h2>` and `<table>` is HTML.

**CSS** (inside the `<style>` block near the top) provides the appearance: the navy header, the cyan highlights, the rounded cards, the amber warning boxes, how the layout reflows on a phone. Changing a colour or font means editing CSS only — no logic is affected.

**JavaScript** (inside the `<script>` block at the bottom) provides the behaviour — everything that changes or reacts. All of the logic described below is JavaScript, running in the visitor's browser.

## The logic this page actually runs

**Timezone conversion.** The hardest part of the whole problem is "what time is it for the primary dev right now?". The page uses the browser's built-in internationalisation library (`Intl.DateTimeFormat`) to convert the current moment into Auckland, Sydney and Caracas time. This is deliberate: daylight-saving rules are messy and change, and the browser ships with the official timezone database, so the page never does naive "add 12 hours" arithmetic and never needs updating for DST.

**A pure decision function.** The escalation rules are encoded in one function, `decide(topic, date)`, which takes who the channel topic names and the current moment, and returns the effective primary and which ladder (A, B or C) applies. Keeping the rules in one self-contained function with no side effects means it can be tested in isolation — it was verified against 16+ known date/time cases (boundaries at 9:00 and 17:30, night attribution, weekends, DST) before shipping. If the policy ever changes, this one function is where it changes.

**Fetching live data.** On every page load the JavaScript calls Google's spreadsheet-query endpoint (`gviz`) with the browser's `fetch` API, parses the response, finds the "Primary Dev" and "Secondary Dev" rows, and matches them column-by-column against the "Week Commencing" dates. This is a small example of the most common pattern on the web: a page calling an external API for its data instead of having the data baked in.

**Graceful degradation.** Network calls fail — the sheet may be unshared, Google unreachable, the visitor offline. The page therefore carries a snapshot of the roster inside the file and falls back to it whenever the live fetch fails, telling the user honestly which source it is showing via the badge in the header. A page used for 2am emergencies must never show a blank screen because a fetch failed.

**State and rendering.** The page keeps its current state (which name is selected, any manual secondary override, the live roster if loaded) in JavaScript variables, and re-draws the display from that state every second. This "state → render" loop is the core idea behind every modern web app, from this page up to React applications like Tamanu's own frontend.

**Precedence rules.** The logic encodes an explicit hierarchy of truth: the Slack channel topic beats the roster (with a visible warning when they disagree), a manually typed secondary beats the roster's secondary, and live sheet data beats the built-in snapshot. Making precedence explicit prevents the classic failure of two data sources silently disagreeing.

## Software concepts most sites need — and how this page handles each

| Concept | What it means in general | On this page |
|---|---|---|
| Hosting / deployment | Somewhere a server hands your files to browsers | GitHub Pages serves `index.html`; deploying = replacing the file in the repo |
| Frontend vs backend | Code in the browser vs code on a server | Frontend only — there is no backend at all |
| Data storage | A database or store that outlives the page | The Google Sheet plays this role; the page stores nothing |
| API calls | One system asking another for data over HTTP | The `fetch` to Google's gviz endpoint |
| CORS | Browsers block cross-site requests unless the other site allows them | Google allows it only for sheets shared "anyone with the link" — which is why that setting is required |
| Authentication | Knowing who the user is | None — the page is public and read-only, so none is needed (and none protects it) |
| Error handling | Deciding what happens when something fails | Snapshot fallback + honest status badge instead of a broken page |
| Caching | Reusing old data to be fast, at the risk of staleness | Live fetch uses `cache: no-store` so every load is fresh; the snapshot is the deliberate "stale but reliable" layer |
| Versioning | Tracking changes so you can review and roll back | Every change to the page is a git commit in the `dev-oncall` repo |
| Testing | Checking logic against known cases before release | The decision function and sheet parser have a test suite run before each release |

## Maintenance — what to change, where

Roster changes (swaps, leave, extending the rota) are made **in the Google Sheet only**; the site picks them up on the next page load. The page file needs editing only when the *rules or people* change: developers' names and timezones live in the `TZ` map at the top of the script, the escalation rules live in `decide()` and `ladderSteps()`, the fallback roster lives in the `SNAPSHOT` object, and the sheet's ID and tab number live in `SHEET_ID` / `ROSTER_GID`. To publish any page change, replace `index.html` in the GitHub repo (Add file → Upload files) — the URL never changes.

Two standing cautions: the published site and the link-shared sheet are readable by anyone who has the URL (first names and a rota only — keep phone numbers out of both), and the sheet's holiday-override tabs (e.g. "Summer Break") are *not* read by the page, so day-by-day holiday exceptions still need to be communicated another way.
