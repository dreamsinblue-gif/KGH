# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

운송기사 배송 입력 시스템 — a delivery-tracking app for drivers and admins. See [3_개발기획서.md](3_개발기획서.md) for the full plan (screens, data model, status). No backend server: Firebase Firestore handles realtime sync, and Google Apps Script backs up Firestore to a Google Sheet on a 5-minute trigger.

**Deployed instances:**
- Firebase project: `delivery-tracker-9b6d7`
- Driver app: https://delivery-tracker-9b6d7.web.app/driver (short alias; full path https://delivery-tracker-9b6d7.web.app/1_입력시스템(index).html still works — both serve the same file, see `hosting.rewrites` in `firebase.json`)
- Admin app: https://delivery-tracker-9b6d7.web.app/admin (short alias for .../2_관리자대시보드(admin).html) — admin login is `dreamsinblue@gmail.com` (Firebase Authentication user; password set in Firebase Console, not stored anywhere in this repo)
- Apps Script backup: source lives outside this repo at `../delivery-tracker-backup-script` (sibling folder, deployed via `clasp`), bound script project "배송기록 자동백업" — writes to Google Sheet https://docs.google.com/spreadsheets/d/1wUtlIpiU9eme7I1aPWtXKlQadPhOEj6W74VVBEgEZtQ/edit (tab `배송기록`). Runs on OAuth from the project-owner Google account, calling the Firestore REST API directly (bypasses `firestore.rules`, which only applies to Firebase client SDKs). Re-deploy with `clasp push` from that folder; re-run `setup()` in the Apps Script editor only if the 5-minute trigger needs recreating.
- Source control: https://github.com/dreamsinblue-gif/KGH (`master` branch). Push after committing so other machines can `git pull`; see "Setting up on a new machine" under Structure & running below.

## Structure & running

There is no build step, package manager, or test suite — this is two standalone static HTML files, each a self-contained vanilla-JS app (no framework, no bundler) with Firebase compat SDK loaded via `<script>` tags from a CDN:

- [1_입력시스템(index).html](1_입력시스템(index).html) — driver-facing app. Two-step flow: select region + stores (2단계 등록), then bulk-register a simultaneous "운송시작" (start) timestamp for all selected stores. Lists in-progress entries with a start→complete flow indicator; "운송완료" (complete, confirm-gated) writes an end timestamp and the card disappears. A "지방 N개 지점 허브 도착(일괄 운송완료)" button appears whenever 2+ 지방 entries are in progress, to complete a truckload that arrived at the shared hub together in one action instead of one tap per store.
- [2_관리자대시보드(admin).html](2_관리자대시보드(admin).html) — read-only admin app with two views toggled by local `view` state: `dashboard` (today's per-store status: 대기/운송중/완료) and `stats` (30-day lookback comparing each store's monthly average duration vs. a selected day's average).

To run either app, just open the `.html` file in a browser (or serve the directory statically) — no install/build/lint/test commands exist in this repo.

**Deploying after a change:**
```
firebase deploy --only hosting --project delivery-tracker-9b6d7          # HTML/CSS/JS changes
firebase deploy --only firestore:rules --project delivery-tracker-9b6d7  # firestore.rules changes
```
Both need `npx firebase-tools login` once per machine (browser OAuth) before they'll work non-interactively.

**Setting up on a new machine:** `git clone https://github.com/dreamsinblue-gif/KGH.git`, then `npx firebase-tools login` and `npx @google/clasp login` (both one-time browser OAuth per machine — credentials are account-bound, not part of the repo). The Apps Script backup source isn't in this repo; pull it separately with `npx @google/clasp clone 1ZXecsGwjtTCGQox1X0vBPbQsI3XyGz3DAOcxwVe-4AIjDaM05fdJR9PY`.

## Architecture notes

**Shared Firestore schema, no shared code.** Both apps independently read/write the same `deliveries` collection (`store`, `region`, `date`, `startAt`, `endAt`, `createdAt`), each with its own copy of `firebaseConfig`, `initFirebase`, timestamp/date helpers, and SVG icon functions. There is no shared module — a schema or helper change must be applied to both files by hand.

**Store list duplication.** The set of 36 stores (24 수도권 / 12 지방) is defined twice and must stay in sync and in the same display order:
- index.html: `STORE_DEFS` (array of `{name, region}`)
- admin.html: `METRO` + `RURAL` (flat name arrays, concatenated as `ALL_STORES`)

**Short URL aliases.** The real filenames are Korean and get percent-encoded in a browser's address bar, so `firebase.json`'s `hosting.rewrites` maps `/driver` → index.html and `/admin` → admin.html for sharing/typing. Both the short and original long paths serve the same files — this is an alias, not a redirect, so there's no need to update bookmarks that already use the long URLs.

**Firebase config is live, not a placeholder.** Both files' `firebaseConfig` point at the `delivery-tracker-9b6d7` project and are deployed (see 배포된 인스턴스 above). If this project is ever swapped for a different Firebase project, `firebaseConfig` must be updated identically in both HTML files.

**Admin access setup is done.** admin.html gates the dashboard behind `firebase.auth().signInWithEmailAndPassword()` using the `ADMIN_EMAIL` constant defined in admin.html. This required two manual steps in the Firebase Console, already completed: (1) creating that exact email/password user under Authentication → Users, and (2) publishing [`firestore.rules`](firestore.rules)'s contents to Firestore Database → Rules (also deployable via `firebase deploy --only firestore:rules` using `firebase.json`/`.firebaserc` in this repo root). `ADMIN_EMAIL` in admin.html and the email hardcoded in `firestore.rules` must stay in sync if the admin account ever changes.

**`request.auth.token.email` and `Timestamp` methods have sharp edges in `firestore.rules`.** Comparing a claim that doesn't exist on the token (e.g. `email` for the driver app's anonymous sessions) throws and denies the whole rule, even inside `||` — guard with `'email' in request.auth.token` first. Separately, `Timestamp.year()/.month()/.day()` take no arguments and always read UTC; there is no timezone-parameter overload despite what stale examples online suggest — Asia/Seoul is computed by shifting `request.time` by 9 hours before reading date parts. Both are explained inline in `firestore.rules`; both only surfaced once rules were actually deployed against a live anonymous session, so re-verify live (not just `compiled successfully`) after editing this file.

**Concurrent-driver duplicate guard.** Multiple drivers can use index.html at once against the same shared `deliveries` collection, and documents have no owner/driver field to reconcile duplicates after the fact. Step 1's store selection (`renderStep1`/`takenBy()`) disables any store that already has an entry for today (showing 등록됨/완료 instead of a selectable chip) so two drivers can't both start the same store — "전체 선택" for 지방 also skips already-taken stores. `takenBy()` reads the full `entries` array directly, independent of what the bottom list renders.

**Completed entries are hidden, with no undo.** `render()` filters `entries` down to `visibleEntries` (only `!e.endAt`) before rendering the bottom list, so a completed delivery disappears from index.html entirely — there is no 초기화/reset button anymore (`resetEntryFirestore` was removed). 운송완료 (and the bulk 지방 완료 button) confirm first and say so explicitly, since there's no self-service undo after — admin.html has no reset capability either, so fixing a wrong completion currently requires editing Firestore directly.

**Rural bulk-complete reuses the single-entry write path.** The "지방 N개 지점 허브 도착" button (`renderEntry`'s sibling block in `render()`, wired in `wire()`) just calls the existing `completeEntryFirestore(id)` once per in-progress 지방 entry via `Promise.all` — there's no batched/atomic write, so in principle a partial failure could complete some stores and not others (acceptable here: same failure mode as any other retry-by-reclicking action in this app, and the button reappears for whatever's still incomplete). `takenBy()` and the region badge already read the real `region` field per entry, so nothing else needed to change for this to be safe with duplicate-guard/admin display.

**`?test=1` sandbox mode (index.html only).** Skips `initFirebase()`/`signInAnonymously()` entirely, seeds `entries` with local sample data via `seedTestData()`, and makes `createEntryFirestore`/`completeEntryFirestore` branch to mutate that local array instead of calling Firestore — so every UI action works but nothing touches the network (verified: only the font stylesheet and the Firebase SDK script file load, zero Firestore API calls). A banner (`.test-banner`) makes the mode unmistakable; "다시 연결" re-seeds instead of reconnecting. Exists because testing on the live driver app otherwise meant creating real documents under real store names — which is blocked once a store is taken for the day, and pollutes production data otherwise. admin.html doesn't have this; it has no write path, so there's nothing unsafe to sandbox.

**Midnight rollover without a page reload.** `DATE` is computed once via `todayKey()` at load, so a device left open past midnight would otherwise keep reading/writing under the previous day's date. `checkDateRollover()` runs every 60s, and on a date change updates `DATE` and calls `subscribeToday()` again in place — no reload, so an in-progress store selection isn't interrupted.

**Kyobo Book Centre CI logo.** Both apps' topbar shows the CI (`Signature_좌우조합` — bird symbol + wordmark) to the left of the date/title block, inlined as a base64 `data:image/jpeg` constant (`LOGO_SRC`) rather than an external file, matching the single-file-app pattern. `mix-blend-mode:multiply` on the `<img>` makes its white JPG background disappear against `--paper` instead of showing a white box — this only works because `--paper` is light; don't reuse this trick if the page background is ever changed to something dark. `.brand-logo`'s height is hand-tuned per file to match that file's own adjacent text (measured via `getBoundingClientRect()`, not guessed): 35px in index.html (`.date-big`), 29px in admin.html (`.title`) — if either file's title/date font-size ever changes, re-measure and update `.brand-logo` to match rather than eyeballing it.

**Render pattern.** Both apps use a manual `render()` that rebuilds `#app` innerHTML from module-level state, followed by a `wire()` call to reattach event listeners — no virtual DOM/diffing. Any state mutation must be followed by calling `render()`.

**Realtime subscriptions.** Firestore `onSnapshot` listeners drive re-renders on remote changes (`subscribeToday` in index.html; `subscribeDashboard` and `subscribeStats` in admin.html). Both start together as soon as the admin logs in — the dashboard's per-store delay indicator and the stats page's monthly averages read from the same 30-day `statsEntries` dataset, fetched once (`statsSubscribing` guard prevents re-subscribing when the stats page is later opened). Date-comparison recomputation on the stats page still happens client-side on selection change, not by re-querying. A 60-second `dashboardTimer` re-renders the dashboard so in-progress elapsed-time displays keep advancing between Firestore updates.
